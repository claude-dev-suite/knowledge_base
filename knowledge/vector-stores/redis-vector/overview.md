# Redis Vector Search Deep Guide

## Overview

Redis Stack provides vector similarity search via the RediSearch module, supporting both FLAT (brute-force) and HNSW index algorithms. Vectors are stored as fields within Redis Hash or JSON data structures, enabling hybrid queries that combine vector similarity with TAG, TEXT, NUMERIC, and GEO filters. The RedisVL Python library provides a high-level abstraction. This guide covers Redis vector search architecture, index creation, hybrid queries, JSON + vectors patterns, and TTL for ephemeral vectors.

For basic Redis setup, see the Redis Stack documentation at <https://redis.io/docs/latest/develop/interact/search-and-query/query/vector-search/>.

---

## Architecture

### How Redis Vector Search Works

```
Client  -->  [Redis Server + RediSearch Module]
                     |
                     +-- [Vector Index] (FLAT or HNSW)
                     +-- [Inverted Index] (TAG, TEXT, NUMERIC)
                     +-- [Data Store] (Hash or JSON)

Search Flow:
1. Client sends FT.SEARCH with vector query + optional filters
2. RediSearch applies pre-filters using inverted index
3. Filtered candidates are scored by vector similarity
4. Results are returned sorted by combined score
```

### Key Concepts

- **Index**: a RediSearch index definition that specifies which fields to index and how
- **Document**: a Redis Hash or JSON object that is indexed
- **Vector field**: a binary blob stored as a Hash field or JSON property, indexed for similarity search
- **Schema**: the field definitions (VECTOR, TAG, TEXT, NUMERIC, GEO) that control indexing

### Prerequisites

Redis Stack includes RediSearch (required for vector search). Standard Redis without modules does not support vector search.

```bash
# Check if RediSearch is loaded
redis-cli MODULE LIST
# Should include: search (version >= 2.8.0)
```

---

## Index Creation

### FT.CREATE with VECTOR Field

```bash
# Create index on Hash documents with HNSW vector field
FT.CREATE idx:documents ON HASH PREFIX 1 doc:
  SCHEMA
    title TEXT WEIGHT 2.0
    content TEXT
    category TAG SORTABLE
    year NUMERIC SORTABLE
    embedding VECTOR HNSW 10
      TYPE FLOAT32
      DIM 1536
      DISTANCE_METRIC COSINE
      M 16
      EF_CONSTRUCTION 200
      EF_RUNTIME 100
      EPSILON 0.01
```

### FLAT vs HNSW

```bash
# FLAT index (brute-force, exact results)
FT.CREATE idx:exact ON HASH PREFIX 1 doc:
  SCHEMA
    embedding VECTOR FLAT 6
      TYPE FLOAT32
      DIM 1536
      DISTANCE_METRIC COSINE
      INITIAL_CAP 100000
      BLOCK_SIZE 1024

# HNSW index (approximate, fast)
FT.CREATE idx:approx ON HASH PREFIX 1 doc:
  SCHEMA
    embedding VECTOR HNSW 10
      TYPE FLOAT32
      DIM 1536
      DISTANCE_METRIC COSINE
      M 16
      EF_CONSTRUCTION 200
      EF_RUNTIME 100
      EPSILON 0.01
      INITIAL_CAP 100000
```

| Algorithm | Recall | Speed | Memory | Use Case |
|-----------|--------|-------|--------|----------|
| FLAT | 100% | Slow (linear scan) | Low | < 100K vectors, exact results required |
| HNSW | ~95-99% | Fast (sublinear) | Higher | > 100K vectors, latency-critical |

### Distance Metrics

| Metric | Description | Score Range |
|--------|-------------|-------------|
| COSINE | Cosine distance | [0, 2] (0 = identical) |
| L2 | Euclidean distance | [0, inf) |
| IP | Inner product (negative) | (-inf, 0] (more negative = more similar) |

---

## Python Client Operations

### Using redis-py

```python
import redis
import numpy as np
from redis.commands.search.field import (
    TagField,
    TextField,
    NumericField,
    VectorField,
)
from redis.commands.search.indexDefinition import IndexDefinition, IndexType
from redis.commands.search.query import Query

# Connect to Redis Stack
r = redis.Redis(host="localhost", port=6379, decode_responses=False)

# Create index
schema = [
    TextField("title", weight=2.0),
    TextField("content"),
    TagField("category", sortable=True),
    NumericField("year", sortable=True),
    VectorField(
        "embedding",
        "HNSW",
        {
            "TYPE": "FLOAT32",
            "DIM": 1536,
            "DISTANCE_METRIC": "COSINE",
            "M": 16,
            "EF_CONSTRUCTION": 200,
            "EF_RUNTIME": 100,
            "INITIAL_CAP": 100000,
        },
    ),
]

r.ft("idx:documents").create_index(
    schema,
    definition=IndexDefinition(prefix=["doc:"], index_type=IndexType.HASH),
)
```

### Inserting Vectors

```python
import struct

def vector_to_bytes(vector: list[float]) -> bytes:
    """Convert float list to bytes for Redis storage."""
    return struct.pack(f"{len(vector)}f", *vector)

# Insert a document with vector
embedding = np.random.rand(1536).astype(np.float32)
r.hset(
    "doc:1",
    mapping={
        "title": "Introduction to Vector Search",
        "content": "Redis Stack supports vector similarity search...",
        "category": "technology",
        "year": "2025",
        "embedding": vector_to_bytes(embedding.tolist()),
    },
)

# Batch insert
pipeline = r.pipeline()
for i in range(10_000):
    emb = np.random.rand(1536).astype(np.float32)
    pipeline.hset(
        f"doc:{i}",
        mapping={
            "title": f"Document {i}",
            "category": f"cat-{i % 10}",
            "year": str(2020 + i % 6),
            "embedding": vector_to_bytes(emb.tolist()),
        },
    )

    if i % 500 == 0:
        pipeline.execute()
        pipeline = r.pipeline()

pipeline.execute()
```

### Vector Similarity Search

```python
# KNN search (top-k nearest neighbors)
query_vector = np.random.rand(1536).astype(np.float32)
query_bytes = vector_to_bytes(query_vector.tolist())

q = (
    Query("*=>[KNN 10 @embedding $vec AS score]")
    .sort_by("score")
    .return_fields("title", "category", "year", "score")
    .dialect(2)
)

results = r.ft("idx:documents").search(
    q,
    query_params={"vec": query_bytes},
)

for doc in results.docs:
    print(f"ID: {doc.id}, Score: {doc.score}, Title: {doc.title}")
```

---

## Hybrid Queries (Vector + Filters)

### Vector + TAG Filter

```python
# Search within a specific category
q = (
    Query("@category:{technology}=>[KNN 10 @embedding $vec AS score]")
    .sort_by("score")
    .return_fields("title", "category", "score")
    .dialect(2)
)

results = r.ft("idx:documents").search(
    q,
    query_params={"vec": query_bytes},
)
```

### Vector + NUMERIC Range

```python
# Search with year filter
q = (
    Query("@year:[2023 2025]=>[KNN 10 @embedding $vec AS score]")
    .sort_by("score")
    .return_fields("title", "year", "score")
    .dialect(2)
)

results = r.ft("idx:documents").search(
    q,
    query_params={"vec": query_bytes},
)
```

### Vector + TEXT Match

```python
# Search with full-text filter
q = (
    Query("@content:(vector database)=>[KNN 10 @embedding $vec AS score]")
    .sort_by("score")
    .return_fields("title", "score")
    .dialect(2)
)

results = r.ft("idx:documents").search(
    q,
    query_params={"vec": query_bytes},
)
```

### Combined Filters

```python
# Complex filter: category=tech AND year>=2023 AND text match
q = (
    Query(
        "(@category:{technology} @year:[2023 +inf] @content:(optimization))=>[KNN 10 @embedding $vec AS score]"
    )
    .sort_by("score")
    .return_fields("title", "category", "year", "score")
    .dialect(2)
)

results = r.ft("idx:documents").search(
    q,
    query_params={"vec": query_bytes},
)
```

---

## RedisVL Python Library

RedisVL provides a higher-level, more Pythonic interface for Redis vector search.

```python
from redisvl.index import SearchIndex
from redisvl.schema import IndexSchema
from redisvl.query import VectorQuery, FilterQuery
from redisvl.query.filter import Tag, Num, Text
import numpy as np

# Define schema
schema = IndexSchema.from_dict({
    "index": {
        "name": "idx:documents",
        "prefix": "doc",
        "storage_type": "hash",
    },
    "fields": [
        {"name": "title", "type": "text", "attrs": {"weight": 2.0}},
        {"name": "content", "type": "text"},
        {"name": "category", "type": "tag", "attrs": {"sortable": True}},
        {"name": "year", "type": "numeric", "attrs": {"sortable": True}},
        {
            "name": "embedding",
            "type": "vector",
            "attrs": {
                "algorithm": "hnsw",
                "dims": 1536,
                "distance_metric": "cosine",
                "datatype": "float32",
                "m": 16,
                "ef_construction": 200,
                "ef_runtime": 100,
            },
        },
    ],
})

# Create index
index = SearchIndex(schema, redis_url="redis://localhost:6379")
index.create(overwrite=True)

# Insert data
data = [
    {
        "title": "Vector Search Guide",
        "content": "Redis supports vector similarity search...",
        "category": "technology",
        "year": 2025,
        "embedding": np.random.rand(1536).astype(np.float32).tobytes(),
    },
]
index.load(data, id_field=None)

# Vector search
query = VectorQuery(
    vector=np.random.rand(1536).astype(np.float32).tobytes(),
    vector_field_name="embedding",
    num_results=10,
    return_fields=["title", "category", "year"],
)
results = index.query(query)

# Filtered vector search
filter_expr = (Tag("category") == "technology") & (Num("year") >= 2023)
query = VectorQuery(
    vector=np.random.rand(1536).astype(np.float32).tobytes(),
    vector_field_name="embedding",
    num_results=10,
    return_fields=["title", "category"],
    filter_expression=filter_expr,
)
results = index.query(query)
```

---

## JSON + Vectors Pattern

Redis JSON (RedisJSON module) allows storing structured documents with vectors.

```bash
# Create index on JSON documents
FT.CREATE idx:json_docs ON JSON PREFIX 1 jdoc:
  SCHEMA
    $.title AS title TEXT
    $.metadata.category AS category TAG
    $.metadata.year AS year NUMERIC
    $.embedding AS embedding VECTOR HNSW 10
      TYPE FLOAT32
      DIM 1536
      DISTANCE_METRIC COSINE
      M 16
      EF_CONSTRUCTION 200
      EF_RUNTIME 100
```

```python
import json

# Store JSON document with vector
doc = {
    "title": "JSON Vector Document",
    "metadata": {
        "category": "technology",
        "year": 2025,
        "author": "alice",
        "tags": ["vectors", "redis"],
    },
    "content": "Full document content here...",
    "embedding": np.random.rand(1536).astype(np.float32).tolist(),
}

r.json().set("jdoc:1", "$", doc)

# Search JSON documents
q = (
    Query("@category:{technology}=>[KNN 10 @embedding $vec AS score]")
    .sort_by("score")
    .return_fields("title", "category", "score")
    .dialect(2)
)

results = r.ft("idx:json_docs").search(
    q,
    query_params={"vec": query_bytes},
)
```

**JSON vs Hash for vectors**:

| Aspect | Hash | JSON |
|--------|------|------|
| Storage overhead | Lower | Higher (~20% more) |
| Nested data | Not supported | Full JSON support |
| Partial updates | HSET individual fields | JSON.SET on paths |
| Vector storage | Binary blob | Float array |
| Read speed | Faster | Slightly slower |

---

## TTL for Ephemeral Vectors

Redis TTL (time-to-live) enables automatic expiration of vectors, useful for caching, session-based search, and real-time recommendation systems.

```python
# Insert vector with TTL (expires in 1 hour)
pipeline = r.pipeline()
pipeline.hset(
    "doc:session-42",
    mapping={
        "title": "Session query context",
        "embedding": vector_to_bytes(embedding.tolist()),
    },
)
pipeline.expire("doc:session-42", 3600)  # 1 hour TTL
pipeline.execute()

# Batch insert with TTL
def insert_with_ttl(r, key, data, ttl_seconds):
    pipe = r.pipeline()
    pipe.hset(key, mapping=data)
    pipe.expire(key, ttl_seconds)
    pipe.execute()

# Pattern: cache embeddings with 2-hour TTL
for doc_id, embedding in compute_embeddings(new_documents):
    insert_with_ttl(
        r,
        f"cache:{doc_id}",
        {
            "embedding": vector_to_bytes(embedding),
            "cached_at": str(int(time.time())),
        },
        ttl_seconds=7200,
    )
```

**TTL and index interaction**: when a key expires, RediSearch automatically removes it from the index. No manual cleanup is needed. However, mass expiration can cause index rebuilding overhead.

### Use Cases for Ephemeral Vectors

1. **Search session context**: store user's recent queries as vectors with 30-min TTL for contextual search
2. **Embedding cache**: cache expensive embedding API results with 2-hour TTL
3. **Real-time recommendations**: store user activity vectors that expire after inactivity
4. **A/B testing**: store experimental embeddings with short TTL for evaluation

---

## Aggregation Queries

```python
# Aggregation with vector search
from redis.commands.search.aggregation import AggregateRequest, Reducer

agg = (
    AggregateRequest("*=>[KNN 100 @embedding $vec AS score]")
    .group_by("@category", Reducer.count().alias("count"))
    .sort_by("@count", asc=False)
    .limit(0, 10)
    .dialect(2)
)

results = r.ft("idx:documents").aggregate(
    agg,
    query_params={"vec": query_bytes},
)

for row in results.rows:
    print(f"Category: {row[1]}, Count: {row[3]}")
```

---

## Range Queries (Radius Search)

```python
# Find all vectors within a distance threshold
q = (
    Query("@embedding:[VECTOR_RANGE 0.3 $vec]")  # cosine distance < 0.3
    .sort_by("__embedding_score")
    .return_fields("title", "__embedding_score")
    .dialect(2)
)

results = r.ft("idx:documents").search(
    q,
    query_params={"vec": query_bytes},
)

print(f"Found {results.total} documents within distance 0.3")
```

---

## Common Pitfalls

1. **Using standard Redis without RediSearch module**: vector search requires Redis Stack or the RediSearch module. Standard Redis does not support FT.CREATE or FT.SEARCH.

2. **Not converting vectors to bytes for Hash storage**: Redis Hash fields store binary data. You must pack float arrays into bytes using `struct.pack` or numpy's `.tobytes()`.

3. **Forgetting `dialect(2)` in queries**: vector search requires RediSearch query dialect 2. Without it, KNN queries fail or return unexpected results.

4. **Using FLAT index for large datasets**: FLAT performs brute-force scan. Above 100K vectors, HNSW is dramatically faster with minimal recall loss.

5. **Not setting EF_RUNTIME for HNSW queries**: the default ef_runtime may be too low for high-recall requirements. Set it to 100-200 for production workloads.

6. **Mass-expiring keys simultaneously**: if thousands of keys expire at the same second, the index rebuilding can cause latency spikes. Spread TTLs across a time range.

7. **Storing vectors in JSON as float arrays without index type specification**: JSON vector fields must be indexed with the correct TYPE (FLOAT32). Mismatch causes search failures.

8. **Not using pipeline for batch inserts**: individual HSET commands have per-command overhead. Use pipelines for 10-50x faster batch inserts.

---

## References

- Redis vector search documentation: https://redis.io/docs/latest/develop/interact/search-and-query/query/vector-search/
- Redis Stack: https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack/
- RedisVL: https://github.com/redis/redis-vl-python
- RediSearch: https://redis.io/docs/latest/develop/interact/search-and-query/
- Redis TTL documentation: https://redis.io/docs/latest/commands/expire/
