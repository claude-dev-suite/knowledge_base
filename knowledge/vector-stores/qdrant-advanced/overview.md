# Qdrant Advanced Deep Guide

## Overview

Qdrant is a high-performance vector similarity search engine written in Rust. It provides a production-ready service with a convenient API for storing, searching, and managing vectors with additional payload. Qdrant supports named vectors (multiple vector spaces per point), rich payload indexing, advanced filtering, quantization (scalar, binary, product), sparse vectors for hybrid search, and a recommendations API. This guide covers architecture, advanced features, and production patterns.

For basic quickstart, see the Qdrant documentation at <https://qdrant.tech/documentation/>.

---

## Architecture

Qdrant uses a segment-based architecture where each collection is divided into shards, and each shard contains segments. Segments are the atomic unit of data storage and indexing.

```
Collection
  +-- Shard 0
  |     +-- Segment A (indexed, immutable)
  |     +-- Segment B (indexed, immutable)
  |     +-- Segment C (write-ahead, mutable)
  +-- Shard 1
        +-- Segment D ...
```

### Key Concepts

- **Collection**: a named set of points (vectors + payloads) with a defined distance metric
- **Point**: a record consisting of an ID (u64 or UUID), one or more named vectors, and an optional JSON payload
- **Shard**: a horizontal partition of a collection, distributable across nodes
- **Segment**: an internal storage unit within a shard; immutable segments are optimized for search

### Distance Metrics

| Metric | Formula | Use Case |
|--------|---------|----------|
| Cosine | 1 - cos(a, b) | Text embeddings (most common) |
| Euclid | L2 distance | Image embeddings, spatial data |
| Dot | -dot(a, b) | Pre-normalized embeddings |
| Manhattan | L1 distance | Sparse feature spaces |

---

## Collection Management

### Creating Collections

```python
from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance,
    VectorParams,
    HnswConfigDiff,
    OptimizersConfigDiff,
    QuantizationConfig,
    ScalarQuantization,
    ScalarQuantizationConfig,
    ScalarType,
)

client = QdrantClient(host="localhost", port=6333)

# Basic collection with single vector
client.create_collection(
    collection_name="documents",
    vectors_config=VectorParams(
        size=1536,
        distance=Distance.COSINE,
        on_disk=False,          # keep vectors in RAM for speed
    ),
    hnsw_config=HnswConfigDiff(
        m=16,
        ef_construct=128,
        full_scan_threshold=10000,
    ),
    optimizers_config=OptimizersConfigDiff(
        default_segment_number=4,       # parallelism for search
        indexing_threshold=20000,        # build HNSW after this many points
        memmap_threshold=50000,         # switch to mmap after this many
    ),
)
```

### Named Vectors (Multiple Vector Spaces)

Named vectors allow storing multiple embeddings per point -- for example, a title embedding and a content embedding, or embeddings from different models.

```python
from qdrant_client.models import VectorParams

client.create_collection(
    collection_name="articles",
    vectors_config={
        "title": VectorParams(size=384, distance=Distance.COSINE),
        "content": VectorParams(size=1536, distance=Distance.COSINE),
        "image": VectorParams(size=512, distance=Distance.COSINE),
    },
)

# Upsert with named vectors
from qdrant_client.models import PointStruct

client.upsert(
    collection_name="articles",
    points=[
        PointStruct(
            id=1,
            vector={
                "title": [0.1, 0.2, ...],    # 384-dim
                "content": [0.3, 0.4, ...],  # 1536-dim
                "image": [0.5, 0.6, ...],    # 512-dim
            },
            payload={"category": "tech", "author": "alice"},
        ),
    ],
)

# Search using a specific named vector
results = client.query_points(
    collection_name="articles",
    query=[0.1, 0.2, ...],       # query vector
    using="content",              # which named vector to search
    limit=10,
)
```

### Sparse Vectors

Qdrant supports sparse vectors natively, enabling hybrid dense+sparse search (like SPLADE + dense embedding).

```python
from qdrant_client.models import (
    SparseVectorParams,
    SparseIndexParams,
    PointStruct,
    SparseVector,
)

# Create collection with both dense and sparse vectors
client.create_collection(
    collection_name="hybrid_docs",
    vectors_config={
        "dense": VectorParams(size=1536, distance=Distance.COSINE),
    },
    sparse_vectors_config={
        "sparse": SparseVectorParams(
            index=SparseIndexParams(on_disk=False),
        ),
    },
)

# Upsert with sparse vector
client.upsert(
    collection_name="hybrid_docs",
    points=[
        PointStruct(
            id=1,
            vector={
                "dense": [0.1, 0.2, ...],
                "sparse": SparseVector(
                    indices=[10, 47, 388, 2911],
                    values=[0.8, 0.3, 0.6, 0.9],
                ),
            },
            payload={"text": "example document"},
        ),
    ],
)
```

---

## Payload Indexing and Filtering

Qdrant can index payload fields for fast pre-filtering. This is critical for production workloads where most queries include metadata filters.

### Index Types

| Field Type | Index Type | Operators |
|-----------|-----------|-----------|
| keyword | Keyword index | match (exact), match (any) |
| integer | Integer index | range, match |
| float | Float index | range |
| bool | Bool index | match |
| geo | Geo index | geo_bounding_box, geo_radius |
| datetime | Datetime index | range |
| text | Full-text index | text match |

### Creating Payload Indexes

```python
from qdrant_client.models import PayloadSchemaType, TextIndexParams, TokenizerType

# Keyword index for exact matching
client.create_payload_index(
    collection_name="documents",
    field_name="category",
    field_schema=PayloadSchemaType.KEYWORD,
)

# Integer index for range queries
client.create_payload_index(
    collection_name="documents",
    field_name="year",
    field_schema=PayloadSchemaType.INTEGER,
)

# Full-text index for text search within payloads
client.create_payload_index(
    collection_name="documents",
    field_name="content",
    field_schema=TextIndexParams(
        type="text",
        tokenizer=TokenizerType.WORD,
        min_token_len=2,
        max_token_len=20,
        lowercase=True,
    ),
)
```

### Filtering Queries

```python
from qdrant_client.models import Filter, FieldCondition, MatchValue, Range

# Combined vector search + payload filter
results = client.query_points(
    collection_name="documents",
    query=[0.1, 0.2, ...],
    query_filter=Filter(
        must=[
            FieldCondition(
                key="category",
                match=MatchValue(value="engineering"),
            ),
            FieldCondition(
                key="year",
                range=Range(gte=2023),
            ),
        ],
        must_not=[
            FieldCondition(
                key="status",
                match=MatchValue(value="archived"),
            ),
        ],
    ),
    limit=10,
)
```

**Performance note**: payload indexes dramatically reduce search time for filtered queries. Without an index, Qdrant must scan all payloads. With an index, it pre-filters the candidate set before vector comparison. For collections over 100K points with filters, always create payload indexes on filtered fields.

---

## Quantization

Quantization reduces memory usage by compressing vector representations. Qdrant supports three types.

### Scalar Quantization (SQ)

Converts float32 to int8. Reduces memory by ~4x with <1% recall loss.

```python
from qdrant_client.models import (
    QuantizationConfig,
    ScalarQuantization,
    ScalarQuantizationConfig,
    ScalarType,
)

client.update_collection(
    collection_name="documents",
    quantization_config=ScalarQuantization(
        scalar=ScalarQuantizationConfig(
            type=ScalarType.INT8,
            quantile=0.99,        # clip outliers above 99th percentile
            always_ram=True,      # keep quantized vectors in RAM
        ),
    ),
)
```

### Binary Quantization (BQ)

Converts float32 to single bits. Reduces memory by ~32x but has higher recall loss. Best for high-dimensional embeddings (1536+) where the sign of each dimension carries most of the information.

```python
from qdrant_client.models import BinaryQuantization, BinaryQuantizationConfig

client.update_collection(
    collection_name="documents",
    quantization_config=BinaryQuantization(
        binary=BinaryQuantizationConfig(
            always_ram=True,
        ),
    ),
)

# Use oversampling + rescoring to recover recall
results = client.query_points(
    collection_name="documents",
    query=[0.1, 0.2, ...],
    search_params=SearchParams(
        quantization=QuantizationSearchParams(
            rescore=True,         # rescore with original vectors
            oversampling=2.0,     # retrieve 2x candidates before rescoring
        ),
    ),
    limit=10,
)
```

**Binary quantization recall by dimension** (1M vectors, rescore=True, oversampling=2.0):

| Dimensions | Recall@10 | Memory Reduction |
|-----------|-----------|-----------------|
| 384 | 0.89 | 32x |
| 768 | 0.94 | 32x |
| 1536 | 0.97 | 32x |
| 3072 | 0.98 | 32x |

### Product Quantization (PQ)

Divides vectors into sub-vectors and quantizes each independently. Offers tunable compression between SQ and BQ.

```python
from qdrant_client.models import ProductQuantization, ProductQuantizationConfig

client.update_collection(
    collection_name="documents",
    quantization_config=ProductQuantization(
        product=ProductQuantizationConfig(
            compression=CompressionRatio.X16,   # 16x compression
            always_ram=True,
        ),
    ),
)
```

**Quantization comparison** (1M vectors, 1536 dims):

| Type | Memory per Vector | Recall@10 | Search Speed |
|------|------------------|-----------|-------------|
| None (float32) | 6,144 bytes | 1.000 | Baseline |
| Scalar (int8) | 1,536 bytes | 0.995 | 1.2x faster |
| Product (x16) | 384 bytes | 0.975 | 2.5x faster |
| Binary | 192 bytes | 0.970 | 4x faster |

---

## Recommendations API

Qdrant provides a recommendations API that finds points similar to positive examples and dissimilar to negative examples, without requiring a query vector.

```python
# Find articles similar to IDs 42 and 87, but NOT like ID 13
results = client.recommend(
    collection_name="documents",
    positive=[42, 87],
    negative=[13],
    strategy="average_vector",   # or "best_score"
    limit=10,
    query_filter=Filter(
        must=[
            FieldCondition(key="category", match=MatchValue(value="tech")),
        ],
    ),
)
```

**Strategies**:
- `average_vector`: averages positive vectors, subtracts negative vectors, searches for the result
- `best_score`: scores each candidate against all positive/negative examples individually, combines scores

---

## gRPC vs REST

Qdrant exposes both REST (port 6333) and gRPC (port 6334) interfaces.

| Aspect | REST | gRPC |
|--------|------|------|
| Latency | Higher (JSON serialization) | Lower (protobuf) |
| Throughput | Good | ~2-3x higher |
| Streaming | No | Yes |
| Browser support | Yes | Limited |
| Debugging | Easy (curl, Postman) | Harder |

```python
# REST client (default)
client = QdrantClient(host="localhost", port=6333)

# gRPC client (recommended for production)
client = QdrantClient(host="localhost", port=6334, prefer_grpc=True)
```

**Recommendation**: use gRPC for production services where throughput matters. Use REST for development, debugging, and browser-based clients.

---

## Batch Operations

### Bulk Upsert

```python
import numpy as np
from qdrant_client.models import PointStruct, Batch

# Generate or load your data
num_points = 100_000
dim = 1536
vectors = np.random.rand(num_points, dim).astype(np.float32)
payloads = [{"category": f"cat_{i % 10}", "idx": i} for i in range(num_points)]

# Batch upsert (recommended batch size: 100-500 points)
BATCH_SIZE = 256

for i in range(0, num_points, BATCH_SIZE):
    end = min(i + BATCH_SIZE, num_points)
    points = [
        PointStruct(
            id=j,
            vector=vectors[j].tolist(),
            payload=payloads[j],
        )
        for j in range(i, end)
    ]
    client.upsert(
        collection_name="documents",
        points=points,
        wait=False,    # async upsert for throughput
    )

# Wait for indexing to complete
client.update_collection(
    collection_name="documents",
    optimizer_config=OptimizersConfigDiff(
        indexing_threshold=20000,
    ),
)
```

### Batch Search

```python
from qdrant_client.models import SearchRequest

# Multiple queries in a single request
queries = [
    SearchRequest(vector=[0.1, 0.2, ...], limit=10),
    SearchRequest(vector=[0.3, 0.4, ...], limit=10),
    SearchRequest(vector=[0.5, 0.6, ...], limit=10, filter=Filter(
        must=[FieldCondition(key="category", match=MatchValue(value="tech"))],
    )),
]

batch_results = client.search_batch(
    collection_name="documents",
    requests=queries,
)

for i, results in enumerate(batch_results):
    print(f"Query {i}: {len(results)} results")
```

---

## When to Pick Qdrant

**Choose Qdrant when:**
- You need a purpose-built vector database with rich filtering
- Named vectors (multi-model search) are a requirement
- You want built-in quantization without external tooling
- Sparse vector support for hybrid search is needed
- You need the recommendations API for discovery use cases
- You want a Rust-based engine with low memory overhead
- gRPC throughput for high-QPS workloads matters

**Consider alternatives when:**
- You need tight SQL integration (pgvector)
- You already have Elasticsearch and want to avoid a new service
- You need serverless with zero ops (Pinecone)
- You need billion-scale with GPU acceleration (Milvus)
- You need embedded/local-only with zero server (LanceDB)

---

## Search Patterns

### Hybrid Dense + Sparse Search

```python
from qdrant_client.models import Prefetch, FusionQuery, Fusion

# Two-phase: prefetch dense and sparse, then fuse with RRF
results = client.query_points(
    collection_name="hybrid_docs",
    prefetch=[
        Prefetch(
            query=[0.1, 0.2, ...],     # dense query
            using="dense",
            limit=50,
        ),
        Prefetch(
            query=SparseVector(
                indices=[10, 47, 388],
                values=[0.8, 0.3, 0.6],
            ),
            using="sparse",
            limit=50,
        ),
    ],
    query=FusionQuery(fusion=Fusion.RRF),   # Reciprocal Rank Fusion
    limit=10,
)
```

### Multi-Stage Search with Re-Ranking

```python
# Stage 1: fast search with quantized vectors
# Stage 2: rescore top candidates with full-precision vectors
results = client.query_points(
    collection_name="documents",
    query=[0.1, 0.2, ...],
    search_params=SearchParams(
        quantization=QuantizationSearchParams(
            rescore=True,
            oversampling=3.0,    # 3x candidates for rescoring
        ),
        hnsw_ef=128,             # search-time ef parameter
    ),
    limit=10,
)
```

### Scroll (Iterate All Points)

```python
# Iterate through all points in a collection
offset = None
while True:
    points, offset = client.scroll(
        collection_name="documents",
        scroll_filter=Filter(
            must=[FieldCondition(key="category", match=MatchValue(value="tech"))],
        ),
        limit=100,
        offset=offset,
        with_payload=True,
        with_vectors=False,    # skip vectors for faster scroll
    )

    for point in points:
        process(point)

    if offset is None:
        break
```

---

## Collection Configuration Tuning

### Optimizer Configuration

```python
client.update_collection(
    collection_name="documents",
    optimizers_config=OptimizersConfigDiff(
        # Number of segments to maintain (more = more parallel search)
        default_segment_number=4,

        # Minimum number of vectors before HNSW index is built
        indexing_threshold=20000,

        # Flush interval in seconds
        flush_interval_sec=5,

        # Maximum segment size (bytes) before splitting
        max_segment_size=200_000,

        # Number of vectors to trigger memory-mapping
        memmap_threshold=50000,

        # Maximum optimization threads
        max_optimization_threads=4,
    ),
)
```

### HNSW Configuration

```python
client.update_collection(
    collection_name="documents",
    hnsw_config=HnswConfigDiff(
        m=16,                           # connections per node (4-64)
        ef_construct=128,               # build quality (64-512)
        full_scan_threshold=10000,      # below this, skip index
        max_indexing_threads=0,         # 0 = auto (all available)
        on_disk=False,                  # keep graph in RAM
    ),
)
```

### Write-Ahead Log (WAL) Configuration

```python
from qdrant_client.models import WalConfigDiff

client.update_collection(
    collection_name="documents",
    wal_config=WalConfigDiff(
        wal_capacity_mb=32,             # WAL segment size
        wal_segments_ahead=0,           # pre-allocated segments
    ),
)
```

---

## Common Pitfalls

1. **Not creating payload indexes for filtered queries**: without indexes, every search with a filter becomes a full scan of payloads. For collections over 10K points, always index fields used in filters.

2. **Using REST for high-throughput pipelines**: the JSON serialization overhead is significant at scale. Switch to gRPC (`prefer_grpc=True`) for 2-3x throughput improvement.

3. **Binary quantization on low-dimensional vectors**: BQ works well for 1536+ dimensions but degrades significantly below 512 dimensions. Use scalar quantization for lower dimensions.

4. **Setting `wait=True` on every upsert**: synchronous writes wait for the WAL flush and segment optimization. Use `wait=False` for bulk ingestion, then verify with a final synchronous call.

5. **Not tuning `ef_construct` for large collections**: the default (128) is reasonable, but for 10M+ collections, increasing to 200-256 improves recall at the cost of longer index build time.

6. **Ignoring segment configuration**: too few segments limit search parallelism; too many increase overhead. A good rule is `default_segment_number` = number of CPU cores / 2.

7. **Using oversampling without rescoring**: oversampling retrieves extra candidates from quantized search but without `rescore=True`, those candidates are not re-ranked with original vectors, defeating the purpose.

8. **Forgetting to set `on_disk=True` when RAM is limited**: by default, vectors and HNSW graph are stored in RAM. For collections that exceed available RAM, enable `on_disk` for vectors and/or the HNSW graph.

---

## References

- Qdrant documentation: https://qdrant.tech/documentation/
- Qdrant Python client: https://github.com/qdrant/qdrant-client
- Qdrant quantization guide: https://qdrant.tech/documentation/guides/quantization/
- Qdrant filtering guide: https://qdrant.tech/documentation/concepts/filtering/
- Qdrant hybrid search: https://qdrant.tech/documentation/concepts/hybrid-queries/
- HNSW paper: Malkov, Y. and Yashunin, D. "Efficient and Robust Approximate Nearest Neighbor using Hierarchical Navigable Small World Graphs." IEEE TPAMI 2020.
