# Milvus Deep Guide

## Overview

Milvus is an open-source vector database designed for billion-scale similarity search. Built on a cloud-native, disaggregated architecture (separating compute, storage, and coordination), Milvus supports multiple index types (IVF_FLAT, HNSW, DiskANN, GPU_IVF_PQ), scalar field indexing, hybrid search (sparse + dense), dynamic schema, upsert, and time travel. This guide covers Milvus 2.4+ architecture, advanced features, and production patterns.

For basic quickstart, see the Milvus documentation at <https://milvus.io/docs>.

---

## Architecture

### Milvus Cluster Components

```
                       +-- [Query Node 1]
Client --> [Proxy] --> +-- [Query Node 2]   (search/query)
              |        +-- [Query Node N]
              |
              +------> [Data Node 1]         (insert/delete)
              |        [Data Node 2]
              |
              +------> [Index Node 1]        (index building)
                       [Index Node 2]

Coordination:
  [Root Coord] -- manages topology
  [Data Coord] -- manages data nodes and segments
  [Query Coord] -- manages query nodes and channels
  [Index Coord] -- manages index building

Storage:
  [etcd]  -- metadata storage
  [MinIO / S3]  -- object storage (segments, indexes)
  [Pulsar / Kafka]  -- message queue (WAL, change streams)
```

### Key Concepts

- **Collection**: a logical table containing fields (columns) and vectors
- **Partition**: a sub-division of a collection for data isolation and targeted search
- **Segment**: the physical storage unit; immutable once sealed
- **Index**: a search-acceleration structure built on vector fields
- **Channel**: a message queue partition for write-ahead log

### Standalone vs Cluster

| Aspect | Standalone | Cluster |
|--------|-----------|---------|
| Deployment | Single process | Multiple nodes |
| Dependencies | etcd, MinIO | etcd, MinIO, Pulsar/Kafka |
| Scale | Millions of vectors | Billions of vectors |
| HA | No | Yes (node redundancy) |
| Use case | Dev, testing, small prod | Large-scale production |

---

## Collection Management

### Creating Collections

```python
from pymilvus import (
    connections,
    Collection,
    CollectionSchema,
    FieldSchema,
    DataType,
    utility,
)

# Connect to Milvus
connections.connect(host="localhost", port="19530")

# Define schema
fields = [
    FieldSchema(name="id", dtype=DataType.INT64, is_primary=True, auto_id=True),
    FieldSchema(name="title", dtype=DataType.VARCHAR, max_length=512),
    FieldSchema(name="category", dtype=DataType.VARCHAR, max_length=64),
    FieldSchema(name="year", dtype=DataType.INT32),
    FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=1536),
]

schema = CollectionSchema(
    fields=fields,
    description="Document embeddings collection",
    enable_dynamic_field=True,   # allow arbitrary JSON fields
)

# Create collection
collection = Collection(
    name="documents",
    schema=schema,
    consistency_level="Bounded",  # Strong, Bounded, Session, Eventually
)

print(f"Collection created: {collection.name}")
```

### Dynamic Schema (JSON Fields)

Milvus 2.4+ supports dynamic schema, allowing arbitrary fields without predefined columns.

```python
# With enable_dynamic_field=True, you can insert any extra fields
data = [
    {
        "title": "Vector Database Guide",
        "category": "technology",
        "year": 2025,
        "embedding": [0.1, 0.2, ...],
        # Dynamic fields (not in schema)
        "tags": ["vectors", "search"],
        "author": "alice",
        "rating": 4.5,
    },
]

collection.insert(data)
```

### Partitions

```python
# Create partitions for data isolation
collection.create_partition("tech")
collection.create_partition("science")
collection.create_partition("business")

# Insert into a specific partition
collection.insert(data, partition_name="tech")

# Search within specific partitions
results = collection.search(
    data=[query_vector],
    anns_field="embedding",
    param={"metric_type": "COSINE", "params": {"ef": 100}},
    limit=10,
    partition_names=["tech", "science"],   # search only these partitions
)
```

---

## Index Types

### HNSW (Recommended for < 10M vectors)

```python
index_params = {
    "metric_type": "COSINE",
    "index_type": "HNSW",
    "params": {
        "M": 16,                  # connections per node
        "efConstruction": 200,    # build-time quality
    },
}

collection.create_index(
    field_name="embedding",
    index_params=index_params,
)

# Search with HNSW
search_params = {"metric_type": "COSINE", "params": {"ef": 100}}
```

### IVF_FLAT (Good for 1M-100M vectors)

```python
index_params = {
    "metric_type": "COSINE",
    "index_type": "IVF_FLAT",
    "params": {
        "nlist": 1024,    # number of clusters
    },
}

collection.create_index(field_name="embedding", index_params=index_params)

# Search with IVF_FLAT
search_params = {"metric_type": "COSINE", "params": {"nprobe": 32}}
```

### IVF_PQ (Memory-Optimized)

```python
index_params = {
    "metric_type": "COSINE",
    "index_type": "IVF_PQ",
    "params": {
        "nlist": 1024,
        "m": 48,            # number of sub-quantizers
        "nbits": 8,         # bits per sub-quantizer
    },
}

collection.create_index(field_name="embedding", index_params=index_params)

# Search with IVF_PQ
search_params = {"metric_type": "COSINE", "params": {"nprobe": 32}}
```

### DiskANN (For Billion-Scale, Disk-Based)

```python
index_params = {
    "metric_type": "COSINE",
    "index_type": "DISKANN",
    "params": {
        "search_list": 128,   # controls build quality
    },
}

collection.create_index(field_name="embedding", index_params=index_params)

# Search with DiskANN
search_params = {"metric_type": "COSINE", "params": {"search_list": 64}}
```

### GPU_IVF_PQ (GPU-Accelerated)

```python
index_params = {
    "metric_type": "COSINE",
    "index_type": "GPU_IVF_PQ",
    "params": {
        "nlist": 2048,
        "m": 48,
        "nbits": 8,
    },
}

collection.create_index(field_name="embedding", index_params=index_params)

# Search with GPU_IVF_PQ
search_params = {"metric_type": "COSINE", "params": {"nprobe": 64}}
```

### Index Type Comparison

| Index | Memory | Build Time | Search Speed | Recall | Best For |
|-------|--------|-----------|-------------|--------|----------|
| HNSW | High | Slow | Fast | High | < 10M, latency-critical |
| IVF_FLAT | Medium | Medium | Medium | High | 1M-100M, balanced |
| IVF_PQ | Low | Medium | Fast | Medium | Large scale, memory-constrained |
| DiskANN | Low (disk) | Slow | Medium | High | Billion-scale |
| GPU_IVF_PQ | Low | Fast | Very fast | Medium | GPU available, high throughput |

---

## Scalar Field Indexing

Milvus can index scalar fields for fast filtering.

```python
# Create scalar indexes
from pymilvus import Collection

collection = Collection("documents")

# Inverted index for VARCHAR fields
collection.create_index(
    field_name="category",
    index_params={"index_type": "INVERTED"},
)

# Inverted index for integer fields (supports range queries)
collection.create_index(
    field_name="year",
    index_params={"index_type": "INVERTED"},
)
```

---

## Hybrid Search (Sparse + Dense)

Milvus 2.4+ supports sparse vectors natively for hybrid search.

### Schema with Sparse Vectors

```python
from pymilvus import FieldSchema, DataType

fields = [
    FieldSchema(name="id", dtype=DataType.INT64, is_primary=True, auto_id=True),
    FieldSchema(name="content", dtype=DataType.VARCHAR, max_length=4096),
    FieldSchema(name="dense_embedding", dtype=DataType.FLOAT_VECTOR, dim=1536),
    FieldSchema(name="sparse_embedding", dtype=DataType.SPARSE_FLOAT_VECTOR),
]

schema = CollectionSchema(fields=fields, enable_dynamic_field=True)
collection = Collection("hybrid_docs", schema=schema)
```

### Inserting Sparse Vectors

```python
from scipy.sparse import csr_array

# Dense vector
dense_vec = [0.1, 0.2, ...]  # 1536-dim

# Sparse vector (e.g., from SPLADE)
sparse_vec = {10: 0.8, 47: 0.3, 388: 0.6, 2911: 0.9}

collection.insert([
    {
        "content": "Example document",
        "dense_embedding": dense_vec,
        "sparse_embedding": sparse_vec,
    }
])
```

### Hybrid Search with Ranker

```python
from pymilvus import AnnSearchRequest, RRFRanker, WeightedRanker

# Dense search request
dense_req = AnnSearchRequest(
    data=[query_dense_vector],
    anns_field="dense_embedding",
    param={"metric_type": "COSINE", "params": {"ef": 100}},
    limit=50,
)

# Sparse search request
sparse_req = AnnSearchRequest(
    data=[query_sparse_vector],
    anns_field="sparse_embedding",
    param={"metric_type": "IP", "params": {}},
    limit=50,
)

# Hybrid search with RRF fusion
results = collection.hybrid_search(
    reqs=[dense_req, sparse_req],
    ranker=RRFRanker(k=60),     # RRF with k=60
    limit=10,
    output_fields=["content"],
)

# Alternative: weighted fusion
results = collection.hybrid_search(
    reqs=[dense_req, sparse_req],
    ranker=WeightedRanker(0.7, 0.3),   # 70% dense, 30% sparse
    limit=10,
    output_fields=["content"],
)

for hits in results:
    for hit in hits:
        print(f"ID: {hit.id}, Score: {hit.score:.4f}")
        print(f"  Content: {hit.entity.get('content')[:100]}")
```

---

## Search Patterns

### Basic Search

```python
# Load collection into memory
collection.load()

# Search
results = collection.search(
    data=[query_vector],
    anns_field="embedding",
    param={"metric_type": "COSINE", "params": {"ef": 100}},
    limit=10,
    output_fields=["title", "category", "year"],
)

for hits in results:
    for hit in hits:
        print(f"ID: {hit.id}, Distance: {hit.distance:.4f}")
        print(f"  Title: {hit.entity.get('title')}")
```

### Filtered Search

```python
# Search with scalar filter
results = collection.search(
    data=[query_vector],
    anns_field="embedding",
    param={"metric_type": "COSINE", "params": {"ef": 100}},
    limit=10,
    expr='category == "technology" and year >= 2023',
    output_fields=["title", "category", "year"],
)
```

### Batch Search

```python
# Multiple query vectors in one call
query_vectors = [vec1, vec2, vec3, vec4, vec5]

results = collection.search(
    data=query_vectors,
    anns_field="embedding",
    param={"metric_type": "COSINE", "params": {"ef": 100}},
    limit=10,
    output_fields=["title"],
)

for i, hits in enumerate(results):
    print(f"Query {i}: {len(hits)} results")
    for hit in hits:
        print(f"  {hit.id}: {hit.distance:.4f} - {hit.entity.get('title')}")
```

### Upsert

```python
# Upsert: insert or update based on primary key
data = [
    {"id": 42, "title": "Updated title", "embedding": [0.1, 0.2, ...]},
]

collection.upsert(data)
```

---

## Time Travel (Consistency)

Milvus supports time travel queries that search data as it existed at a specific point in time.

```python
import datetime

# Search with a specific timestamp
timestamp = int(datetime.datetime(2025, 4, 1).timestamp())

results = collection.search(
    data=[query_vector],
    anns_field="embedding",
    param={"metric_type": "COSINE", "params": {"ef": 100}},
    limit=10,
    travel_timestamp=timestamp,
    output_fields=["title"],
)
```

**Note**: time travel is limited by the garbage collection window (default: 432,000 seconds / 5 days). Data older than this window may be compacted away.

---

## Bulk Insert

```python
import numpy as np

# Prepare data in batches
NUM_VECTORS = 1_000_000
BATCH_SIZE = 10_000
DIM = 1536

for i in range(0, NUM_VECTORS, BATCH_SIZE):
    end = min(i + BATCH_SIZE, NUM_VECTORS)
    batch_size = end - i

    data = [
        [f"Document {j}" for j in range(i, end)],                    # title
        [f"cat-{j % 10}" for j in range(i, end)],                   # category
        [2020 + (j % 6) for j in range(i, end)],                    # year
        np.random.rand(batch_size, DIM).astype(np.float32).tolist(), # embedding
    ]

    collection.insert(data)

    if (i + BATCH_SIZE) % 100_000 == 0:
        print(f"Inserted {i + BATCH_SIZE:,} vectors")

# Flush to persist
collection.flush()
print(f"Total vectors: {collection.num_entities}")
```

### BulkInsert from Files (Large Scale)

```python
from pymilvus import utility, BulkInsertState

# Prepare data as Parquet or JSON files, upload to MinIO
# Then trigger bulk insert
task_id = utility.do_bulk_insert(
    collection_name="documents",
    files=["data/batch_001.parquet", "data/batch_002.parquet"],
)

# Check status
state = utility.get_bulk_insert_state(task_id)
print(f"State: {state.state_name}, Progress: {state.progress}%")

# Wait for completion
while state.state != BulkInsertState.ImportCompleted:
    time.sleep(5)
    state = utility.get_bulk_insert_state(task_id)
```

---

## Connection Patterns

### pymilvus v2 Connection

```python
from pymilvus import connections, MilvusClient

# Classic connection
connections.connect(
    alias="default",
    host="localhost",
    port="19530",
    user="root",
    password="Milvus",
    secure=False,
)

# MilvusClient (simplified API, Milvus 2.4+)
client = MilvusClient(
    uri="http://localhost:19530",
    token="root:Milvus",
)

# Create collection via MilvusClient
client.create_collection(
    collection_name="documents",
    dimension=1536,
    metric_type="COSINE",
)

# Search via MilvusClient
results = client.search(
    collection_name="documents",
    data=[query_vector],
    limit=10,
    output_fields=["title"],
)
```

### Connection Pooling

```python
from pymilvus import connections
import threading

# Thread-local connections
local = threading.local()

def get_connection():
    if not hasattr(local, "conn"):
        alias = f"thread-{threading.current_thread().ident}"
        connections.connect(
            alias=alias,
            host="localhost",
            port="19530",
        )
        local.conn = alias
    return local.conn
```

---

## Common Pitfalls

1. **Not calling `collection.load()` before search**: Milvus requires explicitly loading collections into memory. Without `load()`, search returns errors.

2. **Forgetting to flush after inserts**: data is buffered in the message queue until flushed. Call `collection.flush()` after bulk inserts to ensure persistence.

3. **Using HNSW for billion-scale data**: HNSW requires all vectors in memory. For billion-scale, use DiskANN or IVF_PQ.

4. **Not creating scalar indexes for filtered queries**: without scalar indexes, filters require full scans. Create inverted indexes on fields used in `expr` filters.

5. **Setting `nprobe` too low for IVF indexes**: the default `nprobe=1` searches only one cluster. For recall >95%, use `nprobe` = 5-10% of `nlist`.

6. **Not tuning `consistency_level`**: `Strong` consistency is the safest but slowest. For read-heavy workloads, `Bounded` or `Eventually` provides better performance.

7. **Ignoring the garbage collection window for time travel**: data older than `common.retentionDuration` (default 5 days) cannot be queried via time travel.

8. **Using auto_id with upsert**: if `auto_id=True`, you cannot upsert because you do not control the primary key. Set `auto_id=False` for upsert use cases.

---

## References

- Milvus documentation: https://milvus.io/docs
- pymilvus: https://github.com/milvus-io/pymilvus
- Milvus architecture: https://milvus.io/docs/architecture_overview.md
- Milvus hybrid search: https://milvus.io/docs/multi-vector-search.md
- DiskANN paper: Jayaram Subramanya et al. "DiskANN: Fast Accurate Billion-point Nearest Neighbor Search on a Single Node." NeurIPS 2019.
- Milvus index types: https://milvus.io/docs/index.md
