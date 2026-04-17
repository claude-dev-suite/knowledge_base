# LlamaIndex Architecture (0.12+)

## Overview

LlamaIndex is a Python framework for building LLM-powered applications over your data. Its core abstraction is a pipeline that moves data through five stages: loading, indexing, storing, querying, and evaluating. LlamaIndex 0.12+ (released 2025) restructured the codebase around modular components: `llama-index-core` provides the engine, and integrations (LLMs, vector stores, readers, embeddings) are installed as separate packages.

This guide covers the architectural primitives, how they connect, and the mental model for building applications.

---

## Core Abstractions

### Document

A Document is the top-level container for source data. It holds text content plus metadata:

```python
from llama_index.core import Document

doc = Document(
    text="PostgreSQL supports vector similarity search through pgvector...",
    metadata={
        "source": "pgvector-docs",
        "category": "database",
        "version": "0.8.0",
        "url": "https://github.com/pgvector/pgvector",
    },
    doc_id="pgvector-overview",  # optional, auto-generated if omitted
)
```

Documents are typically created by **Readers** (also called loaders) that ingest data from various sources:

```python
from llama_index.readers.file import PDFReader, MarkdownReader
from llama_index.readers.web import SimpleWebPageReader

# From PDF files
pdf_reader = PDFReader()
documents = pdf_reader.load_data(file_path="handbook.pdf")

# From web pages
web_reader = SimpleWebPageReader()
documents = web_reader.load_data(urls=["https://example.com/docs"])

# From a directory of files (auto-detects file types)
from llama_index.core import SimpleDirectoryReader
documents = SimpleDirectoryReader("./data").load_data()
```

### Node

A Node is a chunk of a Document. When you index documents, LlamaIndex splits them into Nodes -- the atomic units of retrieval:

```python
from llama_index.core.schema import TextNode

node = TextNode(
    text="pgvector supports HNSW and IVFFlat indexes...",
    metadata={
        "source": "pgvector-docs",
        "section": "indexes",
    },
    relationships={
        "parent": doc.doc_id,     # link back to source document
        "previous": "node_001",   # link to previous chunk
        "next": "node_003",       # link to next chunk
    },
)
```

Nodes maintain **relationships** to other nodes and their source documents. This enables:
- Retrieving the parent document for additional context
- Navigating to adjacent chunks (previous/next) for sentence-window retrieval
- Building hierarchical indexes (summary nodes -> detail nodes)

### Node Parsers (Splitters)

Node parsers control how Documents are split into Nodes:

```python
from llama_index.core.node_parser import (
    SentenceSplitter,
    SemanticSplitterNodeParser,
    HierarchicalNodeParser,
)
from llama_index.core import Settings

# Simple sentence-based splitting (most common)
splitter = SentenceSplitter(
    chunk_size=512,          # max tokens per chunk
    chunk_overlap=50,        # overlap between chunks
)
nodes = splitter.get_nodes_from_documents(documents)

# Semantic splitting (groups semantically similar sentences)
from llama_index.embeddings.openai import OpenAIEmbedding
semantic_splitter = SemanticSplitterNodeParser(
    embed_model=OpenAIEmbedding(model="text-embedding-3-small"),
    breakpoint_percentile_threshold=95,  # split when similarity drops
    buffer_size=1,                       # sentences to compare
)

# Hierarchical splitting (creates parent-child node relationships)
hierarchical_splitter = HierarchicalNodeParser.from_defaults(
    chunk_sizes=[2048, 512, 128],  # three levels
)
```

### Index

An Index organizes Nodes for retrieval. The most common is VectorStoreIndex:

```python
from llama_index.core import VectorStoreIndex, Settings
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.llms.openai import OpenAI

# Configure global settings
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")
Settings.llm = OpenAI(model="gpt-4o-mini")

# Build index from documents (handles chunking + embedding automatically)
index = VectorStoreIndex.from_documents(documents)

# Or from pre-built nodes
index = VectorStoreIndex(nodes=nodes)
```

**Index types:**

| Index | Use Case | How it Works |
|---|---|---|
| `VectorStoreIndex` | Standard RAG | Embeds nodes, stores in vector store, kNN retrieval |
| `SummaryIndex` | All-document queries | Stores all nodes, no filtering (feeds all to LLM) |
| `TreeIndex` | Hierarchical summarization | Builds tree of summaries, traverses at query time |
| `KeywordTableIndex` | Keyword-based routing | Extracts keywords per node, matches at query time |
| `KnowledgeGraphIndex` | Entity-relation queries | Builds knowledge graph from text |

---

## The Five Stages

### Stage 1: Loading

Convert external data into Documents:

```python
from llama_index.core import SimpleDirectoryReader

# Loads PDFs, Markdown, text, HTML, DOCX, etc.
documents = SimpleDirectoryReader(
    input_dir="./data",
    recursive=True,
    exclude=["*.tmp", "*.log"],
    filename_as_id=True,          # use filename as doc_id
    required_exts=[".md", ".pdf", ".txt"],
).load_data()

print(f"Loaded {len(documents)} documents")
```

For specialized sources, use LlamaHub readers:

```python
# Install: pip install llama-index-readers-database
from llama_index.readers.database import DatabaseReader

reader = DatabaseReader(uri="postgresql://user:pass@localhost/mydb")
documents = reader.load_data(
    query="SELECT title, content FROM articles WHERE published = true"
)
```

### Stage 2: Indexing

Transform Documents into Nodes and build an index structure:

```python
from llama_index.core import VectorStoreIndex
from llama_index.core.node_parser import SentenceSplitter

# Explicit node parsing + indexing
splitter = SentenceSplitter(chunk_size=512, chunk_overlap=50)
nodes = splitter.get_nodes_from_documents(documents)

# This embeds all nodes and stores them
index = VectorStoreIndex(nodes=nodes, show_progress=True)
```

For large datasets, use the IngestionPipeline (see `ingestion-deep.md`):

```python
from llama_index.core.ingestion import IngestionPipeline
from llama_index.core.node_parser import SentenceSplitter
from llama_index.embeddings.openai import OpenAIEmbedding

pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=512, chunk_overlap=50),
        OpenAIEmbedding(model="text-embedding-3-small"),
    ],
)

nodes = pipeline.run(documents=documents, show_progress=True)
index = VectorStoreIndex(nodes=nodes)
```

### Stage 3: Storing

Persist indexes to avoid re-indexing:

```python
# Local filesystem
index.storage_context.persist(persist_dir="./storage")

# Reload
from llama_index.core import StorageContext, load_index_from_storage

storage_context = StorageContext.from_defaults(persist_dir="./storage")
index = load_index_from_storage(storage_context)
```

For production, use a vector store backend:

```python
# pgvector
# Install: pip install llama-index-vector-stores-postgres
from llama_index.vector_stores.postgres import PGVectorStore

vector_store = PGVectorStore.from_params(
    database="vectordb",
    host="localhost",
    password="secret",
    port=5432,
    user="app",
    table_name="documents",
    embed_dim=1536,
    hnsw_kwargs={
        "hnsw_m": 16,
        "hnsw_ef_construction": 128,
        "hnsw_ef_search": 100,
        "hnsw_dist_method": "vector_cosine_ops",
    },
)

# Build index with external vector store
storage_context = StorageContext.from_defaults(vector_store=vector_store)
index = VectorStoreIndex.from_documents(documents, storage_context=storage_context)

# Reload (vectors are already in postgres, no re-embedding needed)
index = VectorStoreIndex.from_vector_store(vector_store)
```

### Stage 4: Querying

Convert the index into a query engine and ask questions:

```python
# Basic query engine
query_engine = index.as_query_engine(
    similarity_top_k=5,          # retrieve top 5 chunks
    response_mode="compact",     # how to synthesize the answer
)

response = query_engine.query("How do I create an HNSW index in pgvector?")
print(response.response)
print(f"Sources: {[n.node.metadata['source'] for n in response.source_nodes]}")
```

**Response modes:**

| Mode | Behavior | When to Use |
|---|---|---|
| `compact` | Stuff all chunks into one prompt | Default, good for most cases |
| `tree_summarize` | Hierarchically summarize chunks | Long responses, many chunks |
| `refine` | Iterate through chunks, refining answer | Precise answers from many sources |
| `simple_summarize` | Basic summarization | Quick, less accurate |
| `no_text` | Return retrieved nodes only | When you handle synthesis yourself |
| `accumulate` | Apply query to each node separately | When each chunk answers independently |

For chat-style interaction:

```python
chat_engine = index.as_chat_engine(
    chat_mode="condense_plus_context",  # condense chat history + retrieve
    similarity_top_k=5,
)

response = chat_engine.chat("What is pgvector?")
follow_up = chat_engine.chat("How do I install it?")  # remembers context
```

### Stage 5: Evaluating

Measure retrieval and response quality:

```python
from llama_index.core.evaluation import (
    FaithfulnessEvaluator,
    RelevancyEvaluator,
    CorrectnessEvaluator,
)

# Faithfulness: is the response grounded in the retrieved context?
faithfulness_eval = FaithfulnessEvaluator()
result = faithfulness_eval.evaluate_response(
    query="How to create an HNSW index?",
    response=response,
)
print(f"Faithfulness: {result.passing}")  # True/False

# Relevancy: are the retrieved nodes relevant to the query?
relevancy_eval = RelevancyEvaluator()
result = relevancy_eval.evaluate_response(
    query="How to create an HNSW index?",
    response=response,
)
print(f"Relevancy: {result.score}")  # 0.0-1.0

# Batch evaluation with a dataset
from llama_index.core.evaluation import BatchEvalRunner

runner = BatchEvalRunner(
    evaluators={
        "faithfulness": FaithfulnessEvaluator(),
        "relevancy": RelevancyEvaluator(),
    },
    workers=4,
)

eval_results = await runner.aevaluate_queries(
    query_engine=query_engine,
    queries=["How to create HNSW index?", "What distance functions exist?"],
)
```

---

## Settings and Configuration

LlamaIndex 0.12+ uses a global `Settings` object:

```python
from llama_index.core import Settings
from llama_index.llms.openai import OpenAI
from llama_index.llms.anthropic import Anthropic
from llama_index.embeddings.openai import OpenAIEmbedding

# LLM for query synthesis
Settings.llm = OpenAI(model="gpt-4o-mini", temperature=0)
# Or: Settings.llm = Anthropic(model="claude-sonnet-4-20250514")

# Embedding model
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")

# Chunking defaults
Settings.chunk_size = 512
Settings.chunk_overlap = 50

# Context window and output tokens
Settings.context_window = 128000
Settings.num_output = 4096
```

You can also override settings per-component:

```python
# Use a different LLM for a specific query engine
query_engine = index.as_query_engine(
    llm=Anthropic(model="claude-sonnet-4-20250514"),
    similarity_top_k=10,
)
```

---

## Package Structure

LlamaIndex 0.12+ is modular. Install only what you need:

```bash
# Core (always needed)
pip install llama-index-core

# LLM integrations
pip install llama-index-llms-openai
pip install llama-index-llms-anthropic

# Embedding integrations
pip install llama-index-embeddings-openai
pip install llama-index-embeddings-huggingface

# Vector store integrations
pip install llama-index-vector-stores-postgres
pip install llama-index-vector-stores-qdrant
pip install llama-index-vector-stores-chroma

# Reader integrations
pip install llama-index-readers-file
pip install llama-index-readers-web
pip install llama-index-readers-database

# Or install everything (not recommended for production)
pip install llama-index
```

---

## Minimal Complete Example

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader, Settings
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding

# Configure
Settings.llm = OpenAI(model="gpt-4o-mini")
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")

# Load, index, query
documents = SimpleDirectoryReader("./data").load_data()
index = VectorStoreIndex.from_documents(documents)
query_engine = index.as_query_engine(similarity_top_k=5)

response = query_engine.query("What are the main topics covered?")
print(response)
```

---

## Common Pitfalls

1. **Installing `llama-index` instead of specific packages**: the meta-package installs everything (~500 MB). Install `llama-index-core` plus the specific integrations you need.

2. **Not setting `Settings.embed_model`**: defaults to OpenAI, which requires an API key. For local models, explicitly set `Settings.embed_model = HuggingFaceEmbedding(model_name="BAAI/bge-small-en-v1.5")`.

3. **Using `from_documents` for large datasets**: this loads everything into memory and embeds sequentially. Use `IngestionPipeline` with batching and caching for datasets >10K documents.

4. **Ignoring chunk size tuning**: the default 1024-token chunks may be too large for precise retrieval or too small for context. Tune based on your documents and queries.

5. **Not persisting the index**: every restart re-embeds all documents. Always persist to a vector store or local storage.

6. **Mixing old and new API**: LlamaIndex 0.10 -> 0.12 had significant API changes. Ensure your code matches the installed version. Look for `llama_index.core` imports (new) vs `llama_index` imports (old).

---

## References

- LlamaIndex documentation: https://docs.llamaindex.ai/en/stable/
- LlamaIndex GitHub: https://github.com/run-llama/llama_index
- LlamaHub (integrations): https://llamahub.ai/
- LlamaIndex Discord: https://discord.gg/llamaindex
