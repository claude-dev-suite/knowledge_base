# ChromaDB Advanced Deep Guide

## Overview

ChromaDB is an open-source embedding database designed for simplicity and developer experience. Starting with version 0.5+, ChromaDB introduced a significant architectural overhaul with a Rust-based storage engine, persistent-by-default mode, and improved multi-tenancy. ChromaDB uses HNSW (via hnswlib) as its core index and supports pluggable embedding functions, rich metadata filtering, and multiple distance metrics. This guide covers the 0.5+ architecture, collection configuration, embedding function plugins, metadata filtering, and advanced patterns.

For basic ChromaDB usage (create collection, add, query), see the companion introductory guide. This document focuses on advanced configuration and production patterns.

---

## Architecture (0.5+)

### Core Components

```
ChromaDB 0.5+ Architecture
===========================

Client (Python/JS/REST)
    |
    v
Chroma Server (optional, for client/server mode)
    |
    v
+-----------------------------------+
|  Chroma Core (Rust)               |
|  - Segment Manager                |
|  - Write-Ahead Log (WAL)         |
|  - Compaction Engine              |
+-----------------------------------+
    |              |
    v              v
HNSW Index     Metadata Index
(hnswlib)      (blockfile-based)
    |              |
    v              v
Persistent Storage (local disk or cloud)
```

### Persistent vs Ephemeral Mode

In 0.5+, persistent mode is the default. Ephemeral mode is opt-in for testing.

```python
import chromadb

# Persistent (default) -- data survives process restart
client = chromadb.PersistentClient(path="/data/chromadb")

# Ephemeral -- in-memory only, data lost on process exit
client = chromadb.EphemeralClient()

# Client/Server -- connects to a running Chroma server
client = chromadb.HttpClient(host="localhost", port=8000)

# Cloud -- connects to Chroma Cloud (managed)
client = chromadb.CloudClient(
    tenant="my-tenant",
    database="my-database",
    api_key="your-api-key"
)
```

### Storage Layout

```
/data/chromadb/
  chroma.sqlite3          # Metadata: collections, tenants, databases
  {collection_id}/
    data_level0.bin       # HNSW graph layer 0
    header.bin            # HNSW index metadata
    length.bin            # Vector count
    link_lists.bin        # HNSW upper layer links
    blockfile/            # Metadata and document storage
```

---

## Collection Configuration

### HNSW Parameters

ChromaDB exposes hnswlib's configuration parameters through collection metadata.

```python
collection = client.create_collection(
    name="documents",
    metadata={
        "hnsw:space": "cosine",            # Distance metric
        "hnsw:M": 32,                       # Max connections per node (default: 16)
        "hnsw:construction_ef": 200,         # Build-time beam width (default: 100)
        "hnsw:search_ef": 100,               # Query-time beam width (default: 10)
        "hnsw:num_threads": 4,               # Threads for index operations
        "hnsw:batch_size": 1000,             # Internal batching for adds
        "hnsw:sync_threshold": 5000,         # Flush to disk threshold
    }
)
```

### Parameter Effects

| Parameter | Default | Effect of Increasing | Tradeoff |
|---|---|---|---|
| `hnsw:M` | 16 | Higher recall, more memory | Memory vs recall |
| `hnsw:construction_ef` | 100 | Better index quality, slower builds | Build time vs recall |
| `hnsw:search_ef` | 10 | Higher recall, slower queries | Query speed vs recall |
| `hnsw:num_threads` | 4 | Faster parallel operations | CPU usage |
| `hnsw:batch_size` | 1000 | Fewer disk flushes | Memory spike vs I/O |
| `hnsw:sync_threshold` | 5000 | Fewer disk syncs | Durability vs speed |

**Tuning guidance**:

```python
# For prototyping / small datasets (< 50K vectors)
# Use defaults -- they are optimized for this range
collection = client.create_collection(name="prototype")

# For medium datasets (50K - 500K vectors)
collection = client.create_collection(
    name="medium_dataset",
    metadata={
        "hnsw:space": "cosine",
        "hnsw:M": 24,
        "hnsw:construction_ef": 150,
        "hnsw:search_ef": 50,
    }
)

# For larger datasets (500K - 1M vectors)
collection = client.create_collection(
    name="large_dataset",
    metadata={
        "hnsw:space": "cosine",
        "hnsw:M": 32,
        "hnsw:construction_ef": 200,
        "hnsw:search_ef": 100,
        "hnsw:num_threads": 8,
    }
)
```

### Distance Metrics

| Space | Metric | Range | Lower = More Similar |
|---|---|---|---|
| `cosine` | Cosine distance | [0, 2] | Yes |
| `l2` | Squared L2 distance | [0, inf) | Yes |
| `ip` | Negative inner product | (-inf, inf) | Yes (more negative = more similar) |

```python
# Cosine distance (default, most common for text embeddings)
collection = client.create_collection(
    name="cosine_collection",
    metadata={"hnsw:space": "cosine"}
)

# Inner product (use with pre-normalized embeddings)
collection = client.create_collection(
    name="ip_collection",
    metadata={"hnsw:space": "ip"}
)

# L2 / Euclidean distance
collection = client.create_collection(
    name="l2_collection",
    metadata={"hnsw:space": "l2"}
)
```

---

## Embedding Function Plugins

ChromaDB supports pluggable embedding functions that automatically embed documents on add and queries on search.

### OpenAI

```python
from chromadb.utils.embedding_functions import OpenAIEmbeddingFunction

openai_ef = OpenAIEmbeddingFunction(
    api_key="sk-...",
    model_name="text-embedding-3-small",
    dimensions=1536  # Or 512/256 for Matryoshka
)

collection = client.create_collection(
    name="openai_docs",
    embedding_function=openai_ef
)

# Add documents -- embeddings generated automatically
collection.add(
    ids=["doc1", "doc2"],
    documents=["Hello world", "Machine learning overview"],
    metadatas=[{"source": "test"}, {"source": "wiki"}]
)

# Query -- query text embedded automatically
results = collection.query(
    query_texts=["artificial intelligence"],
    n_results=5
)
```

### Cohere

```python
from chromadb.utils.embedding_functions import CohereEmbeddingFunction

cohere_ef = CohereEmbeddingFunction(
    api_key="...",
    model_name="embed-english-v3.0",
    input_type="search_document"  # "search_query" for queries
)

collection = client.create_collection(
    name="cohere_docs",
    embedding_function=cohere_ef
)
```

### HuggingFace (Local)

```python
from chromadb.utils.embedding_functions import HuggingFaceEmbeddingFunction

hf_ef = HuggingFaceEmbeddingFunction(
    api_key="hf_...",
    model_name="BAAI/bge-small-en-v1.5"
)

# Or use SentenceTransformer locally (no API key needed)
from chromadb.utils.embedding_functions import SentenceTransformerEmbeddingFunction

st_ef = SentenceTransformerEmbeddingFunction(
    model_name="all-MiniLM-L6-v2",
    device="cuda"  # "cpu" or "cuda"
)

collection = client.create_collection(
    name="local_docs",
    embedding_function=st_ef
)
```

### Voyage AI

```python
from chromadb.utils.embedding_functions import VoyageAIEmbeddingFunction

voyage_ef = VoyageAIEmbeddingFunction(
    api_key="pa-...",
    model_name="voyage-3"
)

collection = client.create_collection(
    name="voyage_docs",
    embedding_function=voyage_ef
)
```

### Custom Embedding Function

```python
from chromadb import Documents, EmbeddingFunction, Embeddings
import numpy as np


class CustomEmbeddingFunction(EmbeddingFunction[Documents]):
    """Custom embedding function that wraps any model."""

    def __init__(self, model_name: str):
        self.model_name = model_name
        # Initialize your model here

    def __call__(self, input: Documents) -> Embeddings:
        # input is a list of strings
        embeddings = []
        for text in input:
            # Replace with your actual embedding logic
            embedding = np.random.rand(384).tolist()
            embeddings.append(embedding)
        return embeddings


custom_ef = CustomEmbeddingFunction(model_name="my-model")
collection = client.create_collection(
    name="custom_docs",
    embedding_function=custom_ef
)
```

---

## Metadata Filtering

ChromaDB supports rich metadata filtering using a `where` clause that mirrors MongoDB-style query operators.

### Filter Operators

| Operator | Description | Example |
|---|---|---|
| `$eq` | Equal (default) | `{"field": {"$eq": "value"}}` or `{"field": "value"}` |
| `$ne` | Not equal | `{"field": {"$ne": "value"}}` |
| `$gt` | Greater than | `{"field": {"$gt": 10}}` |
| `$gte` | Greater than or equal | `{"field": {"$gte": 10}}` |
| `$lt` | Less than | `{"field": {"$lt": 100}}` |
| `$lte` | Less than or equal | `{"field": {"$lte": 100}}` |
| `$in` | In list | `{"field": {"$in": ["a", "b"]}}` |
| `$nin` | Not in list | `{"field": {"$nin": ["x"]}}` |
| `$and` | Logical AND | `{"$and": [{...}, {...}]}` |
| `$or` | Logical OR | `{"$or": [{...}, {...}]}` |

### Filtering Examples

```python
# Simple equality filter
results = collection.query(
    query_embeddings=[query_vec],
    n_results=10,
    where={"source": "arxiv"}
)

# Range filter
results = collection.query(
    query_embeddings=[query_vec],
    n_results=10,
    where={
        "$and": [
            {"year": {"$gte": 2023}},
            {"year": {"$lte": 2025}}
        ]
    }
)

# Compound filter
results = collection.query(
    query_embeddings=[query_vec],
    n_results=10,
    where={
        "$and": [
            {"category": {"$in": ["ml", "ai", "nlp"]}},
            {"is_published": {"$eq": True}},
            {"citation_count": {"$gt": 50}}
        ]
    }
)

# Document content filter (search within stored document text)
results = collection.query(
    query_embeddings=[query_vec],
    n_results=10,
    where_document={"$contains": "transformer"}
)

# Combined metadata + document filter
results = collection.query(
    query_embeddings=[query_vec],
    n_results=10,
    where={"year": {"$gte": 2024}},
    where_document={"$contains": "attention mechanism"}
)
```

### Metadata Types

ChromaDB metadata values must be one of: `str`, `int`, `float`, `bool`. Lists and nested dicts are not supported as metadata values.

```python
# Valid metadata
collection.add(
    ids=["doc1"],
    documents=["Some text"],
    metadatas=[{
        "source": "arxiv",        # str
        "year": 2024,              # int
        "score": 0.95,             # float
        "is_published": True       # bool
    }]
)

# INVALID -- will raise an error
collection.add(
    ids=["doc2"],
    documents=["Some text"],
    metadatas=[{
        "tags": ["ml", "ai"],     # Lists not supported
        "nested": {"key": "val"}  # Nested dicts not supported
    }]
)

# Workaround for lists: serialize as comma-separated string
collection.add(
    ids=["doc3"],
    documents=["Some text"],
    metadatas=[{
        "tags": "ml,ai,nlp",  # Store as string, parse on retrieval
    }]
)
```

---

## CRUD Operations

### Add Documents

```python
# Add with automatic embedding (requires embedding function)
collection.add(
    ids=["id1", "id2", "id3"],
    documents=["First document", "Second document", "Third document"],
    metadatas=[
        {"source": "web", "year": 2024},
        {"source": "pdf", "year": 2023},
        {"source": "api", "year": 2024}
    ]
)

# Add with pre-computed embeddings
collection.add(
    ids=["id4", "id5"],
    embeddings=[[0.1, 0.2, ...], [0.3, 0.4, ...]],
    documents=["Fourth doc", "Fifth doc"],
    metadatas=[{"source": "custom"}, {"source": "custom"}]
)
```

### Update Documents

```python
# Update replaces the entire document, embedding, and metadata
collection.update(
    ids=["id1"],
    documents=["Updated first document"],
    metadatas=[{"source": "web", "year": 2025, "updated": True}]
)
```

### Upsert (Insert or Update)

```python
# Upsert: insert if ID does not exist, update if it does
collection.upsert(
    ids=["id1", "id6"],
    documents=["Updated first", "Brand new sixth"],
    metadatas=[
        {"source": "web", "year": 2025},
        {"source": "new", "year": 2025}
    ]
)
```

### Delete Documents

```python
# Delete by IDs
collection.delete(ids=["id1", "id2"])

# Delete by metadata filter
collection.delete(where={"year": {"$lt": 2023}})

# Delete by document content
collection.delete(where_document={"$contains": "deprecated"})
```

### Get (Non-Vector Retrieval)

```python
# Get by IDs
docs = collection.get(ids=["id1", "id2"])

# Get with metadata filter (no vector search)
docs = collection.get(
    where={"source": "arxiv"},
    limit=100,
    include=["documents", "metadatas", "embeddings"]
)

# Count documents
count = collection.count()
```

---

## Multi-Collection Patterns

### Namespace Isolation

```python
# Use separate collections for different data types or tenants
code_collection = client.create_collection(
    name="code_snippets",
    metadata={"hnsw:space": "cosine"},
    embedding_function=code_ef
)

docs_collection = client.create_collection(
    name="documentation",
    metadata={"hnsw:space": "cosine"},
    embedding_function=text_ef
)

# Search across multiple collections
def multi_collection_search(query: str, n_results: int = 5):
    code_results = code_collection.query(
        query_texts=[query],
        n_results=n_results
    )
    doc_results = docs_collection.query(
        query_texts=[query],
        n_results=n_results
    )
    # Merge and re-rank
    return merge_results(code_results, doc_results)
```

### Collection Management

```python
# List all collections
collections = client.list_collections()

# Get existing collection (must specify same embedding function)
collection = client.get_collection(
    name="documents",
    embedding_function=openai_ef
)

# Get or create (idempotent)
collection = client.get_or_create_collection(
    name="documents",
    embedding_function=openai_ef,
    metadata={"hnsw:space": "cosine"}
)

# Delete collection
client.delete_collection(name="old_collection")
```

---

## LangChain and LlamaIndex Integration

### LangChain

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# Create from documents
vectorstore = Chroma.from_documents(
    documents=docs,
    embedding=embeddings,
    collection_name="langchain_docs",
    persist_directory="/data/chromadb",
    collection_metadata={"hnsw:space": "cosine"}
)

# Load existing
vectorstore = Chroma(
    collection_name="langchain_docs",
    embedding_function=embeddings,
    persist_directory="/data/chromadb"
)

# Search
results = vectorstore.similarity_search_with_score(
    query="machine learning",
    k=5,
    filter={"source": "arxiv"}
)
```

### LlamaIndex

```python
from llama_index.vector_stores.chroma import ChromaVectorStore
from llama_index.core import VectorStoreIndex, StorageContext
import chromadb

chroma_client = chromadb.PersistentClient(path="/data/chromadb")
chroma_collection = chroma_client.get_or_create_collection("llamaindex_docs")

vector_store = ChromaVectorStore(chroma_collection=chroma_collection)
storage_context = StorageContext.from_defaults(vector_store=vector_store)

# Build index from documents
index = VectorStoreIndex.from_documents(
    documents,
    storage_context=storage_context
)

# Query
query_engine = index.as_query_engine(similarity_top_k=5)
response = query_engine.query("What is retrieval augmented generation?")
```

---

## Common Pitfalls

1. **Not setting `hnsw:search_ef` high enough**: the default of 10 gives fast but low-recall results. For production RAG, set it to at least 50-100.

2. **Using the wrong embedding function on get_collection**: ChromaDB does not store the embedding function with the collection. If you retrieve a collection with a different embedding function than was used to add documents, query results will be meaningless.

3. **Expecting nested metadata**: ChromaDB metadata values must be scalar types (str, int, float, bool). Lists and nested objects will raise errors.

4. **Not handling ID uniqueness**: if you `add` a document with an ID that already exists, ChromaDB will raise an error in 0.5+. Use `upsert` instead for idempotent operations.

5. **Assuming ChromaDB scales to millions**: ChromaDB is optimized for up to ~1M vectors on a single node. Beyond that, consider Qdrant, Weaviate, or pgvector with proper tuning.

6. **Mixing distance metrics across adds**: once a collection is created with a specific `hnsw:space`, all vectors must use that metric. You cannot mix cosine and L2 vectors in the same collection.

7. **Ignoring persistence path on PersistentClient**: if you create a PersistentClient with a different `path` than before, you get a new, empty database. The path is the identity of the database.

---

## References

- ChromaDB documentation: https://docs.trychroma.com/
- ChromaDB GitHub: https://github.com/chroma-core/chroma
- ChromaDB 0.5 migration guide: https://docs.trychroma.com/migration
- hnswlib: https://github.com/nmslib/hnswlib
- LangChain Chroma integration: https://python.langchain.com/docs/integrations/vectorstores/chroma
- LlamaIndex Chroma integration: https://docs.llamaindex.ai/en/stable/examples/vector_stores/ChromaIndexDemo/
