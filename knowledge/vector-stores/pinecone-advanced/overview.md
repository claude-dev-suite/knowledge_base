# Pinecone Advanced Deep Guide

## Overview

Pinecone is a fully managed vector database available in two architectures: serverless and pod-based. It provides hybrid sparse-dense search, metadata filtering, namespaces for multi-tenancy, collections for snapshot/restore, an inference API for embedding generation, and an assistants API for RAG. This guide covers Pinecone's architecture, advanced features, and production patterns using the Python SDK v5+.

For basic quickstart, see the Pinecone documentation at <https://docs.pinecone.io/>.

---

## Architecture

### Serverless vs Pod-Based

Pinecone offers two deployment architectures with fundamentally different cost and performance characteristics.

**Serverless** (recommended for most workloads):
- Scales to zero when idle, scales up automatically under load
- Pay per operation (reads, writes, storage) rather than fixed compute
- Multi-region support
- Cold start latency on idle indexes
- Best for: variable workloads, cost optimization, starting new projects

**Pod-Based** (legacy, still available):
- Fixed compute allocation (p1, p2, s1 pod types)
- Predictable latency (no cold starts)
- Higher fixed cost but unlimited queries within capacity
- Best for: steady high-QPS workloads, latency-sensitive applications

```python
from pinecone import Pinecone, ServerlessSpec, PodSpec

pc = Pinecone(api_key="your-api-key")

# Serverless index
pc.create_index(
    name="documents-serverless",
    dimension=1536,
    metric="cosine",
    spec=ServerlessSpec(
        cloud="aws",
        region="us-east-1",
    ),
)

# Pod-based index
pc.create_index(
    name="documents-pod",
    dimension=1536,
    metric="cosine",
    spec=PodSpec(
        environment="us-east-1-aws",
        pod_type="p2.x1",
        pods=1,
        replicas=1,
    ),
)
```

### Metric Types

| Metric | Description | Use Case |
|--------|-------------|----------|
| cosine | Cosine similarity | Text embeddings (default) |
| dotproduct | Dot product | Pre-normalized embeddings |
| euclidean | L2 distance | Image/spatial embeddings |

---

## Index Operations

### Connecting to an Index

```python
from pinecone import Pinecone

pc = Pinecone(api_key="your-api-key")
index = pc.Index("documents-serverless")

# Check index stats
stats = index.describe_index_stats()
print(f"Total vectors: {stats.total_vector_count}")
print(f"Dimensions: {stats.dimension}")
print(f"Namespaces: {list(stats.namespaces.keys())}")
```

### Upsert Vectors

```python
import numpy as np

# Single upsert
index.upsert(
    vectors=[
        {
            "id": "doc-1",
            "values": np.random.rand(1536).tolist(),
            "metadata": {
                "title": "Introduction to Vector Databases",
                "category": "technology",
                "year": 2025,
                "tags": ["vectors", "databases", "search"],
            },
        },
    ],
)

# Batch upsert (recommended: 100-1000 vectors per call)
batch_size = 100
vectors_to_upsert = []
for i in range(10_000):
    vectors_to_upsert.append({
        "id": f"doc-{i}",
        "values": np.random.rand(1536).tolist(),
        "metadata": {
            "category": f"cat-{i % 10}",
            "year": 2020 + (i % 6),
        },
    })

    if len(vectors_to_upsert) >= batch_size:
        index.upsert(vectors=vectors_to_upsert)
        vectors_to_upsert = []

# Flush remaining
if vectors_to_upsert:
    index.upsert(vectors=vectors_to_upsert)
```

### Upsert with Async for Throughput

```python
import asyncio
from pinecone import Pinecone

pc = Pinecone(api_key="your-api-key")

async def async_upsert(index, batches):
    """Upsert multiple batches concurrently."""
    tasks = []
    for batch in batches:
        # Pinecone SDK v5 supports async operations
        tasks.append(asyncio.to_thread(index.upsert, vectors=batch))

    await asyncio.gather(*tasks)

# Prepare batches
all_vectors = [
    {"id": f"doc-{i}", "values": np.random.rand(1536).tolist(), "metadata": {"idx": i}}
    for i in range(100_000)
]

batches = [all_vectors[i:i+100] for i in range(0, len(all_vectors), 100)]
index = pc.Index("documents-serverless")

asyncio.run(async_upsert(index, batches))
```

---

## Hybrid Sparse-Dense Search

Pinecone supports hybrid search by combining dense vectors with sparse vectors in a single query.

### Upserting Sparse-Dense Vectors

```python
# Upsert with both dense and sparse vectors
index.upsert(
    vectors=[
        {
            "id": "doc-1",
            "values": [0.1, 0.2, ...],         # dense vector (1536-dim)
            "sparse_values": {
                "indices": [10, 47, 388, 2911, 15000],
                "values": [0.8, 0.3, 0.6, 0.9, 0.2],
            },
            "metadata": {"title": "Example document"},
        },
    ],
)
```

### Hybrid Query

```python
# Query with both dense and sparse vectors
results = index.query(
    vector=[0.1, 0.2, ...],         # dense query vector
    sparse_vector={
        "indices": [10, 47, 388],
        "values": [0.8, 0.3, 0.6],
    },
    top_k=10,
    include_metadata=True,
    alpha=0.7,    # weight: 0=sparse only, 1=dense only
)

for match in results.matches:
    print(f"ID: {match.id}, Score: {match.score:.4f}")
    print(f"  Metadata: {match.metadata}")
```

### Generating Sparse Vectors with SPLADE

```python
from transformers import AutoTokenizer, AutoModelForMaskedLM
import torch

# Load SPLADE model
tokenizer = AutoTokenizer.from_pretrained("naver/splade-cocondenser-ensembledistil")
model = AutoModelForMaskedLM.from_pretrained("naver/splade-cocondenser-ensembledistil")

def encode_sparse(text: str) -> dict:
    """Generate SPLADE sparse vector from text."""
    tokens = tokenizer(text, return_tensors="pt", truncation=True, max_length=512)
    with torch.no_grad():
        output = model(**tokens)

    logits = output.logits
    relu_log = torch.log1p(torch.relu(logits))
    weighted = torch.max(relu_log, dim=1).values.squeeze()

    # Extract non-zero indices and values
    nonzero = weighted.nonzero().squeeze()
    indices = nonzero.tolist()
    values = weighted[nonzero].tolist()

    if isinstance(indices, int):
        indices = [indices]
        values = [values]

    return {"indices": indices, "values": values}

# Example
sparse = encode_sparse("vector database performance optimization")
print(f"Non-zero tokens: {len(sparse['indices'])}")
```

---

## Metadata Filtering

### Filter Operators

| Operator | Type | Example |
|----------|------|---------|
| `$eq` | Equality | `{"category": {"$eq": "tech"}}` |
| `$ne` | Not equal | `{"status": {"$ne": "archived"}}` |
| `$gt` / `$gte` | Greater than | `{"year": {"$gte": 2023}}` |
| `$lt` / `$lte` | Less than | `{"score": {"$lt": 0.5}}` |
| `$in` | In list | `{"category": {"$in": ["tech", "science"]}}` |
| `$nin` | Not in list | `{"status": {"$nin": ["deleted", "draft"]}}` |
| `$exists` | Field exists | `{"tags": {"$exists": true}}` |
| `$and` | Logical AND | See below |
| `$or` | Logical OR | See below |

### Complex Filter Queries

```python
# Complex filter with AND/OR
results = index.query(
    vector=query_vector,
    top_k=10,
    filter={
        "$and": [
            {"category": {"$in": ["technology", "science"]}},
            {"year": {"$gte": 2023}},
            {
                "$or": [
                    {"priority": {"$eq": "high"}},
                    {"score": {"$gte": 0.8}},
                ]
            },
        ]
    },
    include_metadata=True,
)
```

### Metadata Filtering Performance

Metadata filtering in Pinecone is applied during the search, not as a pre-filter or post-filter. This means:
- Filters do not degrade recall (unlike pre-filtering in some engines)
- Highly selective filters may increase latency because more candidates must be evaluated
- Filters on low-cardinality fields (e.g., category with 10 values) are faster than high-cardinality (e.g., user_id with 1M values)

**Filter selectivity impact** (1M vectors, k=10):

| Selectivity | Latency Increase | Notes |
|-------------|-----------------|-------|
| 100% (no filter) | Baseline | |
| 50% | +10-20% | Minimal impact |
| 10% | +30-50% | Moderate impact |
| 1% | +100-200% | Significant; consider namespace instead |
| 0.1% | +300-500% | Use namespace for this level of isolation |

---

## Namespaces

Namespaces provide logical partitioning within an index. Each namespace has its own vector space and is isolated from other namespaces.

```python
# Upsert to a specific namespace
index.upsert(
    vectors=[
        {"id": "doc-1", "values": [0.1, 0.2, ...], "metadata": {"title": "Doc 1"}},
    ],
    namespace="tenant-alice",
)

# Query within a namespace
results = index.query(
    vector=query_vector,
    top_k=10,
    namespace="tenant-alice",
    include_metadata=True,
)

# Delete all vectors in a namespace
index.delete(delete_all=True, namespace="tenant-alice")

# List namespaces
stats = index.describe_index_stats()
for ns, ns_stats in stats.namespaces.items():
    print(f"Namespace '{ns}': {ns_stats.vector_count} vectors")
```

**Namespace strategy for multi-tenancy**:
- Use one namespace per tenant for strict data isolation
- Namespaces have zero overhead for empty namespaces
- Queries are scoped to a single namespace (no cross-namespace search)
- Maximum namespaces per index: no hard limit, but keep under 100K for best performance

---

## Collections (Snapshot/Restore)

Collections are static snapshots of an index, useful for backup and cloning.

```python
# Create a collection (snapshot of an index)
pc.create_collection(
    name="documents-backup-2025-04-15",
    source="documents-serverless",
)

# List collections
collections = pc.list_collections()
for col in collections:
    print(f"Collection: {col.name}, Status: {col.status}, Size: {col.size}")

# Create a new index from a collection (restore)
pc.create_index(
    name="documents-restored",
    dimension=1536,
    metric="cosine",
    spec=ServerlessSpec(cloud="aws", region="us-east-1"),
    source_collection="documents-backup-2025-04-15",
)

# Delete a collection
pc.delete_collection("documents-backup-2025-04-15")
```

**Collection limitations**:
- Collections are read-only (no upserts)
- Creating a collection can take minutes for large indexes
- Collections count toward storage billing
- Collections can only be used to create indexes with the same dimension and metric

---

## Inference API

Pinecone's inference API generates embeddings without external services.

```python
from pinecone import Pinecone

pc = Pinecone(api_key="your-api-key")

# Generate embeddings
embeddings = pc.inference.embed(
    model="multilingual-e5-large",
    inputs=["What is a vector database?", "How does similarity search work?"],
    parameters={"input_type": "query"},
)

# Use the embeddings for search
query_embedding = embeddings.data[0].values

index = pc.Index("documents-serverless")
results = index.query(
    vector=query_embedding,
    top_k=10,
    include_metadata=True,
)
```

### Supported Inference Models

| Model | Dimensions | Languages | Use Case |
|-------|-----------|-----------|----------|
| multilingual-e5-large | 1024 | 100+ | General purpose, multilingual |
| llama-text-embed-v2 | 1024 | English | High-quality English text |

---

## Assistants API

The Assistants API provides a managed RAG pipeline: upload files, and Pinecone handles chunking, embedding, indexing, and retrieval.

```python
# Create an assistant
assistant = pc.assistant.create_assistant(
    assistant_name="docs-assistant",
    instructions="You are a helpful technical documentation assistant.",
    model="gpt-4o",
)

# Upload a file
assistant.upload_file(
    file_path="/path/to/document.pdf",
    timeout=300,
)

# Chat with the assistant (RAG)
response = assistant.chat(
    messages=[
        {"role": "user", "content": "How do I configure vector search?"},
    ],
)

print(response.message.content)
for citation in response.citations:
    print(f"  Source: {citation.file_name}, Page: {citation.page}")
```

---

## Query Patterns

### Basic Similarity Search

```python
results = index.query(
    vector=query_vector,
    top_k=10,
    include_metadata=True,
    include_values=False,    # skip returning vector values for speed
)

for match in results.matches:
    print(f"ID: {match.id}, Score: {match.score:.4f}")
    if match.metadata:
        print(f"  Title: {match.metadata.get('title')}")
```

### Fetch by ID

```python
# Fetch specific vectors by ID
result = index.fetch(ids=["doc-1", "doc-2", "doc-3"])

for id, vector in result.vectors.items():
    print(f"ID: {id}, Metadata: {vector.metadata}")
```

### Update Metadata (Without Re-Embedding)

```python
# Update metadata without changing the vector
index.update(
    id="doc-1",
    set_metadata={"category": "updated-category", "reviewed": True},
)
```

### Delete Vectors

```python
# Delete by IDs
index.delete(ids=["doc-1", "doc-2"])

# Delete by metadata filter
index.delete(
    filter={"category": {"$eq": "deprecated"}},
)

# Delete all in a namespace
index.delete(delete_all=True, namespace="old-data")
```

---

## Error Handling and Retry

```python
from pinecone import Pinecone
from pinecone.exceptions import PineconeException
import time

pc = Pinecone(api_key="your-api-key")
index = pc.Index("documents-serverless")

def upsert_with_retry(vectors, namespace="", max_retries=3):
    """Upsert with exponential backoff retry."""
    for attempt in range(max_retries):
        try:
            index.upsert(vectors=vectors, namespace=namespace)
            return
        except PineconeException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Upsert failed (attempt {attempt + 1}): {e}. Retrying in {wait_time}s...")
            time.sleep(wait_time)

def query_with_retry(vector, top_k=10, filter=None, max_retries=3):
    """Query with exponential backoff retry."""
    for attempt in range(max_retries):
        try:
            return index.query(
                vector=vector,
                top_k=top_k,
                filter=filter,
                include_metadata=True,
            )
        except PineconeException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Query failed (attempt {attempt + 1}): {e}. Retrying in {wait_time}s...")
            time.sleep(wait_time)
```

---

## Common Pitfalls

1. **Not batching upserts**: single-vector upserts are 10-50x slower than batched upserts. Always batch 100-1000 vectors per call.

2. **Using metadata filters for high-cardinality tenant isolation**: filtering by `user_id` across millions of values is slow. Use namespaces for tenant isolation instead.

3. **Including vector values in query responses**: `include_values=True` increases response size significantly. Only include values when you need them for re-ranking or debugging.

4. **Ignoring cold start latency on serverless**: idle serverless indexes have cold start latency of 100-500ms on the first query. Keep indexes warm with periodic heartbeat queries if low latency is critical.

5. **Not setting `input_type` for inference API**: the embedding model produces different vectors for "query" vs "passage" input types. Using the wrong type degrades retrieval quality.

6. **Creating too many indexes instead of using namespaces**: each index has its own compute and storage overhead. Use namespaces within a single index for logical partitioning.

7. **Not monitoring index fullness**: pod-based indexes have fixed capacity. When an index is full, upserts fail silently or error. Monitor `total_vector_count` against your plan limits.

8. **Assuming immediate consistency**: Pinecone is eventually consistent. After an upsert, newly inserted vectors may not appear in query results for a few seconds.

---

## References

- Pinecone documentation: https://docs.pinecone.io/
- Pinecone Python SDK v5: https://github.com/pinecone-io/pinecone-python-client
- Pinecone hybrid search: https://docs.pinecone.io/guides/data/understanding-hybrid-search
- Pinecone namespaces: https://docs.pinecone.io/guides/indexes/use-namespaces
- Pinecone collections: https://docs.pinecone.io/guides/indexes/back-up-an-index
- Pinecone inference API: https://docs.pinecone.io/guides/inference/understanding-inference
- SPLADE paper: Formal et al. "SPLADE: Sparse Lexical and Expansion Model for First Stage Ranking." SIGIR 2021.
