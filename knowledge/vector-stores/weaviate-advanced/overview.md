# Weaviate Advanced Deep Guide

## Overview

Weaviate is an open-source vector database that provides native hybrid search (BM25 + vector), pluggable vectorizer modules, generative search, multi-tenancy, and built-in compression (PQ, BQ, SQ). Built in Go, it is designed for production workloads with a schema-driven approach. This guide covers Weaviate's architecture, module system, hybrid search internals, multi-tenancy, compression options, and production patterns.

For basic quickstart, see the Weaviate documentation at <https://weaviate.io/developers/weaviate>.

---

## Architecture

### Core Components

```
Client Request
    |
    v
  [API Gateway]  --  REST (8080) / gRPC (50051) / GraphQL
    |
    v
  [Schema Manager]  --  Collection definitions, properties, vectorizer config
    |
    v
  [Shard Manager]  --  Routes to correct shard/tenant
    |
    v
  [HNSW Index]  +  [Inverted Index]  +  [Object Store (LSM)]
    |                   |                      |
    v                   v                      v
  Vector search    BM25 / filter         Object retrieval
```

### Key Concepts

- **Collection** (formerly "Class"): a named group of objects with a defined schema. Each collection has its own vector index and inverted index.
- **Property**: a typed field on objects within a collection. Properties can be indexed for filtering.
- **Object**: a single record with properties, an optional vector, and a UUID.
- **Shard**: a horizontal partition. Single-node Weaviate typically uses one shard per collection.
- **Tenant**: an isolated partition within a multi-tenant collection (each tenant has its own vector index).

---

## Collection Schema

### Creating a Collection

```python
import weaviate
from weaviate.classes.config import (
    Configure,
    Property,
    DataType,
    VectorDistances,
)

client = weaviate.connect_to_local()

# Create a collection with explicit schema
collection = client.collections.create(
    name="Document",
    vectorizer_config=Configure.Vectorizer.text2vec_openai(
        model="text-embedding-3-small",
    ),
    vector_index_config=Configure.VectorIndex.hnsw(
        distance_metric=VectorDistances.COSINE,
        ef_construction=128,
        max_connections=32,       # m parameter
        ef=100,                   # ef_search default
        dynamic_ef_min=100,
        dynamic_ef_max=500,
        dynamic_ef_factor=8,
    ),
    properties=[
        Property(name="title", data_type=DataType.TEXT),
        Property(name="content", data_type=DataType.TEXT),
        Property(name="category", data_type=DataType.TEXT,
                 index_filterable=True, index_searchable=True),
        Property(name="year", data_type=DataType.INT,
                 index_filterable=True),
        Property(name="tags", data_type=DataType.TEXT_ARRAY,
                 index_filterable=True),
    ],
)
```

### Named Vectors (Multiple Vector Spaces)

```python
from weaviate.classes.config import Configure, Property, DataType

collection = client.collections.create(
    name="Article",
    vectorizer_config=[
        Configure.NamedVectors.text2vec_openai(
            name="title_vector",
            source_properties=["title"],
            model="text-embedding-3-small",
        ),
        Configure.NamedVectors.text2vec_openai(
            name="content_vector",
            source_properties=["content"],
            model="text-embedding-3-large",
        ),
    ],
    properties=[
        Property(name="title", data_type=DataType.TEXT),
        Property(name="content", data_type=DataType.TEXT),
        Property(name="category", data_type=DataType.TEXT),
    ],
)
```

---

## Vectorizer Modules

Weaviate supports pluggable vectorizer modules that automatically generate embeddings on insert.

### Available Vectorizers

| Module | Provider | Notes |
|--------|----------|-------|
| text2vec-openai | OpenAI | text-embedding-3-small/large |
| text2vec-cohere | Cohere | embed-english-v3.0, embed-multilingual-v3.0 |
| text2vec-huggingface | Hugging Face Inference | Any HF model |
| text2vec-transformers | Local | Self-hosted transformer model |
| text2vec-ollama | Ollama | Local LLM embeddings |
| multi2vec-clip | OpenAI CLIP | Image + text embeddings |
| multi2vec-bind | ImageBind | Multi-modal (image, audio, text) |
| img2vec-neural | ResNet | Image embeddings |

### Using Vectorizers

```python
# With vectorizer: no need to provide vectors manually
collection = client.collections.get("Document")

# Insert -- Weaviate auto-vectorizes the text properties
collection.data.insert(
    properties={
        "title": "Introduction to Vector Databases",
        "content": "Vector databases store and search high-dimensional embeddings...",
        "category": "technology",
        "year": 2025,
    }
)

# Search -- Weaviate auto-vectorizes the query
results = collection.query.near_text(
    query="how do vector databases work",
    limit=10,
    return_metadata=weaviate.classes.query.MetadataQuery(distance=True),
)

for obj in results.objects:
    print(f"{obj.properties['title']}: distance={obj.metadata.distance:.4f}")
```

### Bring Your Own Vectors

```python
import numpy as np

collection = client.collections.get("Document")

# Insert with explicit vector
embedding = np.random.rand(1536).tolist()
collection.data.insert(
    properties={
        "title": "Custom embedded document",
        "content": "...",
    },
    vector=embedding,
)

# Search with explicit vector
query_vec = np.random.rand(1536).tolist()
results = collection.query.near_vector(
    near_vector=query_vec,
    limit=10,
)
```

---

## Hybrid Search (BM25 + Vector)

Weaviate provides native hybrid search that combines BM25 keyword scores with vector similarity scores using a fusion algorithm.

### How It Works

1. **BM25 search**: runs an inverted index query on tokenized text properties
2. **Vector search**: runs HNSW nearest neighbor search
3. **Fusion**: combines scores using either Ranked Fusion or Relative Score Fusion

### Hybrid Query

```python
collection = client.collections.get("Document")

# Hybrid search (default alpha=0.75 -- 75% vector, 25% BM25)
results = collection.query.hybrid(
    query="vector database performance optimization",
    alpha=0.75,           # 0.0 = pure BM25, 1.0 = pure vector
    limit=10,
    return_metadata=weaviate.classes.query.MetadataQuery(
        score=True, explain_score=True
    ),
)

for obj in results.objects:
    print(f"{obj.properties['title']}: score={obj.metadata.score:.4f}")
    print(f"  Explain: {obj.metadata.explain_score}")
```

### Fusion Algorithms

```python
from weaviate.classes.query import HybridFusion

# Ranked Fusion (default) -- combines ranks, not scores
results = collection.query.hybrid(
    query="vector database",
    fusion_type=HybridFusion.RANKED,
    alpha=0.75,
    limit=10,
)

# Relative Score Fusion -- normalizes scores to [0,1] then combines
results = collection.query.hybrid(
    query="vector database",
    fusion_type=HybridFusion.RELATIVE_SCORE,
    alpha=0.75,
    limit=10,
)
```

**Ranked Fusion** uses reciprocal rank fusion (RRF): `score = alpha * (1 / (rank_vector + 60)) + (1 - alpha) * (1 / (rank_bm25 + 60))`. It is robust and works well without score calibration.

**Relative Score Fusion** normalizes both score sets to [0,1] and computes a weighted average: `score = alpha * norm_vector + (1 - alpha) * norm_bm25`. It can be more precise when score distributions are comparable.

### Hybrid with Filters

```python
from weaviate.classes.query import Filter

results = collection.query.hybrid(
    query="database performance",
    alpha=0.75,
    filters=Filter.by_property("category").equal("technology")
    & Filter.by_property("year").greater_or_equal(2023),
    limit=10,
)
```

### Alpha Tuning Guide

| alpha | Behavior | Best For |
|-------|----------|----------|
| 0.0 | Pure BM25 | Exact keyword matching, legal search |
| 0.25 | BM25-heavy | When exact terms matter more than semantics |
| 0.50 | Balanced | General-purpose |
| 0.75 | Vector-heavy (default) | Semantic search with keyword boost |
| 1.0 | Pure vector | When synonyms/paraphrases matter most |

---

## Generative Modules

Weaviate can run LLM generation on search results directly in the query pipeline.

```python
from weaviate.classes.config import Configure

# Collection with generative module
collection = client.collections.create(
    name="KnowledgeBase",
    vectorizer_config=Configure.Vectorizer.text2vec_openai(),
    generative_config=Configure.Generative.openai(
        model="gpt-4o-mini",
    ),
    properties=[
        Property(name="title", data_type=DataType.TEXT),
        Property(name="content", data_type=DataType.TEXT),
    ],
)

# RAG query: search + generate
results = collection.generate.near_text(
    query="how to optimize vector search",
    limit=5,
    grouped_task="Summarize these documents into a concise answer about vector search optimization.",
)

# Generated answer from all retrieved documents
print(results.generated)

# Individual object generation
results = collection.generate.near_text(
    query="vector search",
    limit=5,
    single_prompt="Explain this document in one sentence: {content}",
)

for obj in results.objects:
    print(f"Original: {obj.properties['title']}")
    print(f"Generated: {obj.generated}")
```

---

## Multi-Tenancy

Multi-tenancy provides data isolation at the collection level. Each tenant has its own vector index, inverted index, and object store. Tenants can be independently activated/deactivated to manage memory.

### Setup

```python
from weaviate.classes.config import Configure
from weaviate.classes.tenants import Tenant, TenantActivityStatus

# Create a multi-tenant collection
collection = client.collections.create(
    name="UserDocuments",
    multi_tenancy_config=Configure.multi_tenancy(
        enabled=True,
        auto_tenant_creation=False,
        auto_tenant_activation=True,
    ),
    vectorizer_config=Configure.Vectorizer.text2vec_openai(),
    properties=[
        Property(name="content", data_type=DataType.TEXT),
    ],
)

# Add tenants
collection.tenants.create([
    Tenant(name="tenant_alice"),
    Tenant(name="tenant_bob"),
    Tenant(name="tenant_charlie"),
])
```

### Tenant Operations

```python
# Insert data for a specific tenant
tenant_collection = collection.with_tenant("tenant_alice")

tenant_collection.data.insert(
    properties={"content": "Alice's private document"},
)

# Search within a tenant (isolated from other tenants)
results = tenant_collection.query.near_text(
    query="private document",
    limit=10,
)

# Deactivate a tenant (offloads from memory)
collection.tenants.update([
    Tenant(name="tenant_charlie", activity_status=TenantActivityStatus.INACTIVE),
])

# Reactivate
collection.tenants.update([
    Tenant(name="tenant_charlie", activity_status=TenantActivityStatus.ACTIVE),
])

# Offload to cold storage (S3/GCS)
collection.tenants.update([
    Tenant(name="tenant_charlie", activity_status=TenantActivityStatus.OFFLOADED),
])
```

**Multi-tenancy best practices**:
- Use tenant deactivation to manage memory for inactive users
- Offloading reduces RAM to near-zero but reactivation takes seconds
- Each tenant has independent HNSW indexes, so search performance is unaffected by other tenants' data size
- Maximum recommended tenants per collection: ~50,000 (above this, consider sharding)

---

## Compression (PQ, BQ, SQ)

### Product Quantization (PQ)

```python
from weaviate.classes.config import Configure

collection = client.collections.create(
    name="CompressedDocs",
    vectorizer_config=Configure.Vectorizer.text2vec_openai(),
    vector_index_config=Configure.VectorIndex.hnsw(
        quantizer=Configure.VectorIndex.Quantizer.pq(
            segments=384,         # number of sub-quantizers (dimension / segments = codes per segment)
            centroids=256,        # number of centroids per segment (typically 256)
            training_limit=100000, # vectors used for training PQ codebook
            encoder_type="kmeans",
            encoder_distribution="log-normal",
        ),
    ),
    properties=[
        Property(name="content", data_type=DataType.TEXT),
    ],
)
```

### Binary Quantization (BQ)

```python
collection = client.collections.create(
    name="BinaryCompressed",
    vectorizer_config=Configure.Vectorizer.text2vec_openai(),
    vector_index_config=Configure.VectorIndex.hnsw(
        quantizer=Configure.VectorIndex.Quantizer.bq(
            rescore_limit=200,    # re-rank top 200 with full vectors
            cache=True,           # cache rescored results
        ),
    ),
    properties=[
        Property(name="content", data_type=DataType.TEXT),
    ],
)
```

### Scalar Quantization (SQ)

```python
collection = client.collections.create(
    name="ScalarCompressed",
    vectorizer_config=Configure.Vectorizer.text2vec_openai(),
    vector_index_config=Configure.VectorIndex.hnsw(
        quantizer=Configure.VectorIndex.Quantizer.sq(
            rescore_limit=200,
            training_limit=100000,
            cache=True,
        ),
    ),
    properties=[
        Property(name="content", data_type=DataType.TEXT),
    ],
)
```

### Compression Comparison (1M vectors, 1536 dims)

| Method | Memory per Vector | Total RAM | Recall@10 (rescored) |
|--------|------------------|-----------|---------------------|
| None (float32) | 6,144 B | ~9.2 GB | 1.000 |
| SQ (int8) | 1,536 B | ~4.8 GB | 0.995 |
| PQ (384 segments) | ~384 B | ~3.2 GB | 0.980 |
| BQ | 192 B | ~2.8 GB | 0.965 |

---

## Async Indexing

Weaviate 1.22+ supports async indexing, which decouples object insertion from vector indexing. Objects become available for filtered/BM25 search immediately, while vector indexing happens in the background.

```python
from weaviate.classes.config import Configure

collection = client.collections.create(
    name="AsyncDocs",
    vectorizer_config=Configure.Vectorizer.text2vec_openai(),
    vector_index_config=Configure.VectorIndex.hnsw(
        skip=False,
    ),
    properties=[
        Property(name="content", data_type=DataType.TEXT),
    ],
)

# Batch insert -- objects are searchable via BM25/filters immediately
# Vector index is built asynchronously
with collection.batch.dynamic() as batch:
    for i in range(100_000):
        batch.add_object(
            properties={"content": f"Document {i} content..."},
        )
```

### Checking Indexing Status

```python
# Check if async indexing is complete
status = collection.aggregate.over_all()
print(f"Total objects: {status.total_count}")

# Check shard status
shard_status = client.collections.get("AsyncDocs").config.get()
print(f"Vector index status: {shard_status}")
```

---

## Replication

```python
from weaviate.classes.config import Configure

collection = client.collections.create(
    name="ReplicatedDocs",
    replication_config=Configure.replication(
        factor=3,                 # 3 copies across nodes
        async_enabled=True,       # async replication for write throughput
    ),
    vectorizer_config=Configure.Vectorizer.text2vec_openai(),
    properties=[
        Property(name="content", data_type=DataType.TEXT),
    ],
)
```

### Consistency Levels

```python
from weaviate.classes.config import ConsistencyLevel

# Strong consistency (all replicas must acknowledge)
results = collection.query.near_text(
    query="important query",
    limit=10,
    consistency_level=ConsistencyLevel.ALL,
)

# Quorum (majority of replicas)
results = collection.query.near_text(
    query="normal query",
    limit=10,
    consistency_level=ConsistencyLevel.QUORUM,
)

# One (any single replica -- fastest)
results = collection.query.near_text(
    query="speed-critical query",
    limit=10,
    consistency_level=ConsistencyLevel.ONE,
)
```

---

## Batch Operations

### Efficient Bulk Insert

```python
collection = client.collections.get("Document")

# Dynamic batching (auto-tunes batch size)
with collection.batch.dynamic() as batch:
    for item in data_items:
        batch.add_object(
            properties={
                "title": item["title"],
                "content": item["content"],
                "category": item["category"],
            },
            # vector=item.get("vector"),  # optional if using vectorizer
        )

# Check for errors
if batch.number_errors > 0:
    print(f"Batch insert had {batch.number_errors} errors")
    for err in collection.batch.failed_objects:
        print(f"  Error: {err.message}")

# Fixed-size batching
with collection.batch.fixed_size(batch_size=200, concurrent_requests=4) as batch:
    for item in data_items:
        batch.add_object(properties=item)
```

### Batch Delete

```python
from weaviate.classes.query import Filter

# Delete all objects matching a filter
result = collection.data.delete_many(
    where=Filter.by_property("category").equal("deprecated"),
)
print(f"Deleted {result.successful} objects")
```

---

## Common Pitfalls

1. **Not configuring a vectorizer module**: without a vectorizer, `near_text` queries will fail. Either configure a vectorizer module or use `near_vector` with your own embeddings.

2. **Setting alpha=1.0 for hybrid and expecting keyword matching**: alpha=1.0 is pure vector search. For keyword-sensitive use cases, use alpha=0.5-0.75 or lower.

3. **Ignoring BM25 tokenization for non-English text**: the default tokenizer is word-based and optimized for English. For CJK or other languages, configure appropriate tokenization.

4. **Creating too many properties without indexing control**: every indexed property consumes memory. Set `index_filterable=False` and `index_searchable=False` on properties you never filter or search on.

5. **Not using multi-tenancy for user-isolated data**: without multi-tenancy, all users share the same vector index. Filtering by user_id is slower and less secure than proper tenant isolation.

6. **Ignoring PQ training requirements**: PQ needs a minimum number of vectors (typically 10,000+) to train a good codebook. Enabling PQ on small collections degrades recall.

7. **Using sync replication for write-heavy workloads**: synchronous replication waits for all replicas, significantly reducing write throughput. Use `async_enabled=True` unless strong consistency is required.

8. **Not setting resource limits in Docker/Kubernetes**: Weaviate can consume all available RAM for vector indexes. Always set memory limits and monitor usage.

---

## References

- Weaviate documentation: https://weaviate.io/developers/weaviate
- Weaviate Python client v4: https://weaviate.io/developers/weaviate/client-libraries/python
- Weaviate hybrid search: https://weaviate.io/developers/weaviate/search/hybrid
- Weaviate multi-tenancy: https://weaviate.io/developers/weaviate/concepts/data#multi-tenancy
- Weaviate compression: https://weaviate.io/developers/weaviate/configuration/compression
- Weaviate modules: https://weaviate.io/developers/weaviate/modules
