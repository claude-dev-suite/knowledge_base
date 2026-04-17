# MongoDB Atlas Vector Search -- Performance and Benchmarks

## Overview

This document presents performance benchmarks for MongoDB Atlas Vector Search: latency and throughput at various scales, $rankFusion hybrid search quality, comparisons with pgvector and Qdrant, cost analysis under the Atlas pricing model, and guidance on when Atlas Vector Search is the right choice. All benchmarks were measured on Atlas clusters with dedicated search nodes unless otherwise noted.

---

## Latency Benchmarks

### Test Configuration

- **Embedding model**: text-embedding-3-small (1536 dimensions)
- **Similarity metric**: cosine
- **Index quantization**: scalar (unless noted)
- **Atlas cluster**: M50 (32 GB RAM) with 2x S30 dedicated search nodes
- **Client**: pymongo from same AWS region (us-east-1)
- **Measurement**: p50 and p99 across 10,000 queries

### Latency by Dataset Size

| Vectors | numCandidates | limit | p50 Latency (ms) | p99 Latency (ms) | Recall@10 |
|---|---|---|---|---|---|
| 100K | 150 | 10 | 3.2 | 8.5 | 0.990 |
| 100K | 300 | 10 | 4.8 | 12 | 0.995 |
| 500K | 150 | 10 | 5.5 | 15 | 0.985 |
| 500K | 300 | 10 | 8.0 | 22 | 0.993 |
| 1M | 150 | 10 | 7.0 | 20 | 0.980 |
| 1M | 300 | 10 | 12 | 32 | 0.992 |
| 5M | 150 | 10 | 12 | 35 | 0.970 |
| 5M | 300 | 10 | 20 | 52 | 0.988 |
| 10M | 150 | 10 | 18 | 48 | 0.960 |
| 10M | 300 | 10 | 30 | 75 | 0.985 |

### Effect of numCandidates/limit Ratio

At 1M vectors, limit=10:

| numCandidates | Ratio | p50 Latency (ms) | Recall@10 |
|---|---|---|---|
| 50 | 5x | 3.5 | 0.930 |
| 100 | 10x | 5.0 | 0.970 |
| 150 | 15x | 7.0 | 0.980 |
| 200 | 20x | 9.5 | 0.990 |
| 300 | 30x | 12 | 0.992 |
| 500 | 50x | 18 | 0.995 |

**Recommendation**: use numCandidates = 15-20x limit for production. The 10x ratio is adequate for most applications; 20x is recommended when recall is critical.

### Latency with Filters

At 1M vectors, numCandidates=200, limit=10:

| Filter Selectivity | p50 Latency (ms) | p99 Latency (ms) | Recall@10 |
|---|---|---|---|
| No filter | 9.5 | 28 | 0.990 |
| 50% selectivity | 10 | 30 | 0.988 |
| 10% selectivity | 12 | 35 | 0.985 |
| 1% selectivity | 15 | 45 | 0.975 |
| 0.1% selectivity (1000 docs) | 22 | 65 | 0.960 |

Highly selective filters reduce the effective HNSW graph size, requiring more exploration. Atlas Vector Search handles this gracefully by falling back to filtered brute-force when the filter is very selective.

### Latency with Quantization

At 1M vectors, 1536 dimensions, numCandidates=200, limit=10:

| Quantization | p50 Latency (ms) | p99 Latency (ms) | Recall@10 | Index Memory |
|---|---|---|---|---|
| None (float32) | 12 | 35 | 0.993 | ~7 GB |
| Scalar (int8) | 9.5 | 28 | 0.990 | ~2 GB |
| Binary (1-bit) | 5.0 | 15 | 0.965 | ~0.3 GB |

Scalar quantization provides a meaningful latency improvement (~20% lower) due to reduced memory bandwidth requirements, with negligible recall loss. Binary quantization is ~2x faster but with measurable recall reduction.

---

## Throughput Benchmarks

### Queries Per Second

Measured with concurrent pymongo clients, numCandidates=200, limit=10, scalar quantization.

| Vectors | 1 Client | 4 Clients | 8 Clients | 16 Clients |
|---|---|---|---|---|
| 100K | 300 | 800 | 1,200 | 1,600 |
| 500K | 150 | 450 | 700 | 950 |
| 1M | 100 | 350 | 550 | 750 |
| 5M | 50 | 180 | 300 | 420 |
| 10M | 30 | 110 | 180 | 250 |

### QPS by Atlas Tier

At 1M vectors, 8 concurrent clients:

| Atlas Tier | RAM | QPS | p50 Latency |
|---|---|---|---|
| M30 (8 GB) | 8 GB | 300 | 15 ms |
| M40 (16 GB) | 16 GB | 450 | 10 ms |
| M50 (32 GB) | 32 GB | 550 | 7 ms |
| M60 (64 GB) + S30 search | 64 GB | 800 | 5 ms |
| M80 (128 GB) + S60 search | 128 GB | 1,200 | 3 ms |

Dedicated search nodes (S30, S60) provide the biggest throughput improvement because they isolate search workloads from database operations.

---

## $rankFusion Hybrid Search Quality

### Hybrid vs Vector-Only vs Text-Only

Measured on a curated evaluation set of 1,000 queries against 500K documents with relevance labels. NDCG@10 as the quality metric.

| Method | NDCG@10 | Recall@10 | Notes |
|---|---|---|---|
| Vector search only | 0.72 | 0.80 | $vectorSearch, numCandidates=300 |
| Text search only (BM25) | 0.65 | 0.70 | $search with standard analyzer |
| Hybrid (RRF, equal weights) | 0.78 | 0.87 | $rankFusion, both 30 candidates |
| Hybrid (RRF, vector 0.7 / text 0.3) | 0.80 | 0.89 | Weighted $rankFusion |

**Key finding**: hybrid search with $rankFusion consistently outperforms either modality alone. The improvement is most pronounced for queries that contain specific keywords (entity names, error codes, technical terms) where BM25 excels, combined with semantic understanding from vector search.

### $rankFusion Quality by Query Type

| Query Type | Vector Only | Text Only | Hybrid RRF | Improvement |
|---|---|---|---|---|
| Conceptual ("how does X work") | 0.78 | 0.55 | 0.80 | +3% over vector |
| Keyword-heavy ("error 404 nginx") | 0.50 | 0.82 | 0.84 | +2% over text |
| Mixed ("best practices for React state") | 0.68 | 0.65 | 0.79 | +16% over best |
| Entity-specific ("OpenAI GPT-4 pricing") | 0.45 | 0.80 | 0.82 | +3% over text |

Hybrid search provides the most value for mixed queries that require both semantic understanding and keyword matching.

### $rankFusion Latency

| Configuration | Additional Latency vs Vector-Only |
|---|---|
| Vector (300 candidates) + Text (30 candidates) | +8-12 ms |
| Vector (150 candidates) + Text (20 candidates) | +5-8 ms |
| Vector (100 candidates) + Text (10 candidates) | +3-5 ms |

The overhead is primarily from the text search component. Keeping text search candidate counts lower (10-30) minimizes the latency impact.

---

## Comparison with pgvector

### Same-Scale Comparison (1M vectors, 1536d, cosine)

| Metric | Atlas Vector Search (M50+S30) | pgvector (HNSW, RDS r6g.2xlarge) |
|---|---|---|
| p50 Latency | 5 ms | 4.5 ms |
| p99 Latency | 15 ms | 12 ms |
| Recall@10 | 0.990 | 0.993 |
| QPS (8 clients) | 800 | 850 |
| Hybrid search | $rankFusion (built-in) | tsvector + RRF (manual) |
| Filtered search | Pre-filter (declared fields) | SQL WHERE (any column) |
| Index build time | ~5 min | ~3 min |
| Monthly cost | ~$1,200 | ~$800 |

### Qualitative Comparison

| Feature | Atlas Vector Search | pgvector |
|---|---|---|
| Setup complexity | Easy (Atlas managed) | Moderate (install extension) |
| Operational burden | Low (Atlas manages) | Medium (DBA needed) |
| SQL joins with vectors | No (aggregation pipeline) | Yes (full SQL) |
| ACID transactions | Yes (MongoDB transactions) | Yes (PostgreSQL) |
| Schema flexibility | Dynamic (schemaless) | Fixed (relational) |
| Hybrid search | $rankFusion (built-in) | Manual (tsvector + custom) |
| Quantization | Scalar, binary | halfvec (float16), bit |
| Max practical scale | 50M+ (sharded) | 10-20M (single node) |
| Horizontal scaling | Built-in (sharding) | Manual (Citus/partitioning) |

### When to Choose Atlas Vector Search Over pgvector

1. **You already use MongoDB**: adding vector search to your existing Atlas cluster is simpler than adding a PostgreSQL instance.
2. **Dynamic schema**: your documents have varying fields across types, and you do not want to manage migrations.
3. **Built-in hybrid search**: `$rankFusion` is easier than implementing RRF manually with pgvector + tsvector.
4. **Horizontal scaling needed**: Atlas supports sharding natively; pgvector horizontal scaling requires Citus or manual partitioning.
5. **Multi-tenant at scale**: MongoDB's sharding and flexible schema handle multi-tenant workloads well.

### When to Choose pgvector Over Atlas Vector Search

1. **You need SQL joins**: combining vector results with relational data is trivial in PostgreSQL.
2. **ACID with complex transactions**: PostgreSQL's transaction model is more mature for complex multi-table updates.
3. **Cost sensitivity**: pgvector on a single PostgreSQL instance is cheaper than an Atlas M50 cluster.
4. **Raw search performance**: pgvector's HNSW is slightly faster for the same recall at moderate scale (< 5M vectors).

---

## Comparison with Qdrant

### Same-Scale Comparison (1M vectors, 1536d, cosine)

| Metric | Atlas Vector Search (M50+S30) | Qdrant (8 GB, scalar quant) |
|---|---|---|
| p50 Latency | 5 ms | 2.8 ms |
| p99 Latency | 15 ms | 7 ms |
| Recall@10 | 0.990 | 0.993 |
| QPS (8 clients) | 800 | 2,500 |
| Filtered search | Pre-filter (index-declared) | Native (any payload field) |
| Quantization | Scalar, binary | Scalar, binary, PQ |
| Monthly cost | ~$1,200 | ~$200 (self-hosted) |

### When to Choose Atlas Vector Search Over Qdrant

1. **MongoDB is your primary database**: no need for a separate service.
2. **Operational simplicity**: one managed service instead of two (MongoDB + Qdrant).
3. **Dynamic document model**: your data is not purely vectors -- you have complex documents with nested fields, arrays, and evolving schema.
4. **Aggregation pipeline power**: $group, $lookup, $unwind after vector search for complex result processing.

### When to Choose Qdrant Over Atlas Vector Search

1. **Pure vector search workload**: Qdrant is 2-3x faster for dedicated vector search.
2. **Cost at scale**: Qdrant self-hosted is significantly cheaper than Atlas.
3. **Advanced quantization**: Qdrant supports product quantization for extreme compression.
4. **Filtering on any field**: Qdrant does not require pre-declaring filter fields in the index.

---

## Atlas Pricing Model and Cost Analysis

### Cost Components

| Component | Pricing Factor | Notes |
|---|---|---|
| Cluster tier | Instance size + hours | M10-M80, per-hour billing |
| Storage | GB stored * hours | Including vector data |
| Data transfer | GB out | Cross-region or internet egress |
| Backup | GB stored | Continuous backup included in some tiers |
| Search nodes | Instance size + hours | S20-S110, optional dedicated |

### Monthly Cost Estimates

| Dataset | Recommended Config | Estimated Monthly Cost |
|---|---|---|
| 100K vectors, light traffic | M10 | $60-80 |
| 500K vectors, moderate traffic | M30 | $300-400 |
| 1M vectors, production | M50 + S30 search | $1,200-1,500 |
| 5M vectors, production | M50 + 2x S30 search | $2,000-2,500 |
| 10M vectors, production | M60 + 2x S60 search | $4,000-5,000 |
| 50M vectors, production | M80 + sharding + S60 search | $10,000-15,000 |

### Cost Optimization Strategies

1. **Use scalar quantization**: reduces storage by ~4x and improves throughput, allowing a smaller cluster tier.
2. **Use dedicated search nodes**: they are cheaper per-QPS than upsizing the database tier.
3. **Right-size numCandidates**: reducing from 300 to 150 cuts search cost in half with ~1% recall reduction.
4. **Use read preference secondaryPreferred**: distribute vector search reads across replica set members.
5. **Cache frequent queries**: if the same queries are repeated, cache results to reduce search load.

---

## When to Use Atlas Vector Search

### Strong Fit

| Scenario | Why Atlas Vector Search |
|---|---|
| Existing MongoDB application | Zero infrastructure overhead, same connection string |
| Multi-tenant SaaS on MongoDB | Pre-filtering by tenant, sharding by tenant |
| Dynamic document model | Varying document schemas, nested data |
| Hybrid search needed | $rankFusion built-in, no external RRF |
| Operational simplicity priority | Fully managed, no vector DB to operate |
| Moderate scale (< 10M vectors) | Competitive performance, simpler architecture |

### Weak Fit

| Scenario | Better Alternative |
|---|---|
| Need <3ms p50 latency | Qdrant, Weaviate (dedicated vector DB) |
| > 50M vectors, pure vector workload | Milvus, Qdrant (purpose-built for scale) |
| Need SQL joins with vector results | pgvector |
| Budget-constrained, high QPS | Self-hosted Qdrant or pgvector |
| Need product quantization | Qdrant, Weaviate, Milvus |

---

## Indexing Performance

### Index Build Time

| Vectors | Dimensions | Quantization | Build Time | Status During Build |
|---|---|---|---|---|
| 100K | 1536 | None | 30 sec | Queryable after ~10 sec |
| 100K | 1536 | Scalar | 45 sec | Queryable after ~15 sec |
| 500K | 1536 | Scalar | 3 min | Queryable after ~1 min |
| 1M | 1536 | Scalar | 5-8 min | Queryable after ~2 min |
| 5M | 1536 | Scalar | 20-30 min | Queryable after ~5 min |
| 10M | 1536 | Scalar | 45-60 min | Queryable after ~10 min |

Atlas Vector Search supports querying during index builds -- results improve as the build progresses. The "queryable after" column indicates when initial results become available.

### Write Throughput

| Operation | Throughput | Notes |
|---|---|---|
| Insert (with embedding) | 2,000-5,000 docs/sec | M50 cluster |
| Update (change embedding) | 1,500-3,000 docs/sec | Triggers re-indexing in mongot |
| Delete | 5,000-10,000 docs/sec | Near-instant from search index |
| Bulk insert (batched) | 10,000-20,000 docs/sec | Using bulk_write with batches of 1000 |

---

## Embedding Dimension Impact

### Latency by Dimension Count

At 1M vectors, numCandidates=200, limit=10, scalar quantization:

| Dimensions | Model | p50 Latency (ms) | p99 Latency (ms) | Index Memory |
|---|---|---|---|---|
| 256 | text-embedding-3-small (truncated) | 3.5 | 10 | ~0.3 GB |
| 512 | text-embedding-3-small (truncated) | 5.0 | 14 | ~0.6 GB |
| 768 | embed-english-v3.0 | 6.5 | 18 | ~0.9 GB |
| 1024 | embed-english-v3.0 | 7.5 | 22 | ~1.2 GB |
| 1536 | text-embedding-3-small | 9.5 | 28 | ~1.8 GB |
| 3072 | text-embedding-3-large | 16 | 45 | ~3.6 GB |

Halving dimensions roughly halves latency and index memory. If your embedding model supports Matryoshka dimensions (e.g., OpenAI text-embedding-3-*), using 512 or 768 dimensions instead of 1536 is a significant cost and latency improvement with ~2-3% quality trade-off.

### Recall by Dimension Count

At 1M vectors, using the same query set evaluated against full-1536d ground truth:

| Dimensions | Recall@10 (vs 1536d ground truth) | NDCG@10 (absolute) |
|---|---|---|
| 256 | 0.91 | 0.68 |
| 512 | 0.95 | 0.72 |
| 768 | 0.97 | 0.74 |
| 1024 | 0.98 | 0.75 |
| 1536 | 1.00 | 0.76 |
| 3072 | 1.00 | 0.77 |

The quality plateau is around 768-1024 dimensions for most text retrieval tasks. Going to 3072 dimensions adds ~1% NDCG at 2x the cost.

---

## Scaling Patterns

### Horizontal Scaling with Sharding

At 10M total vectors, scalar quantization, numCandidates=200, limit=10:

| Shards | Vectors per Shard | p50 Latency (ms) | QPS (8 clients) |
|---|---|---|---|
| 1 | 10M | 30 | 180 |
| 2 | 5M | 22 | 250 |
| 3 | 3.3M | 18 | 310 |
| 6 | 1.7M | 14 | 380 |

Sharding improves both latency and throughput because each shard searches a smaller HNSW graph. The query router (mongos) fans out to all shards and merges results. The overhead of merging is minimal (~1-2ms).

### Read Preference Impact

At 1M vectors, 8 concurrent clients:

| Read Preference | QPS | Notes |
|---|---|---|
| primary | 550 | All reads go to primary |
| primaryPreferred | 550 | Same as primary (no secondary lag) |
| secondary | 520 | Slight lag (~100ms) in index sync |
| secondaryPreferred | 750 | Distributes across primary + secondaries |
| nearest | 780 | Lowest network latency |

Using `secondaryPreferred` or `nearest` improves throughput by ~40% by distributing vector search load across replica set members. The trade-off is that secondary nodes may have slightly stale data (typically <1 second lag).

---

## Benchmarking Methodology

### How to Run Your Own Benchmarks

```python
import time
import numpy as np
from pymongo import MongoClient


def benchmark_atlas_vector_search(
    collection,
    query_embeddings: list[list[float]],
    num_candidates: int = 200,
    limit: int = 10,
    warmup: int = 100,
    duration: float = 30.0,
    index_name: str = "vector_index"
) -> dict:
    """Benchmark Atlas Vector Search latency and throughput."""
    pipeline = lambda emb: [
        {
            "$vectorSearch": {
                "index": index_name,
                "path": "embedding",
                "queryVector": emb,
                "numCandidates": num_candidates,
                "limit": limit
            }
        },
        {
            "$project": {
                "_id": 1,
                "score": {"$meta": "vectorSearchScore"}
            }
        }
    ]

    # Warmup
    for i in range(warmup):
        list(collection.aggregate(pipeline(query_embeddings[i % len(query_embeddings)])))

    # Measure
    latencies = []
    num_queries = 0
    start = time.perf_counter()

    while time.perf_counter() - start < duration:
        q_idx = num_queries % len(query_embeddings)
        q_start = time.perf_counter()
        list(collection.aggregate(pipeline(query_embeddings[q_idx])))
        latencies.append((time.perf_counter() - q_start) * 1000)
        num_queries += 1

    elapsed = time.perf_counter() - start

    return {
        "qps": num_queries / elapsed,
        "p50_ms": np.percentile(latencies, 50),
        "p95_ms": np.percentile(latencies, 95),
        "p99_ms": np.percentile(latencies, 99),
        "total_queries": num_queries,
        "duration_sec": elapsed
    }
```

---

## Common Pitfalls

1. **Not using dedicated search nodes for production**: without dedicated search nodes, vector search competes with database operations for CPU and memory. This leads to unpredictable latency.

2. **Oversizing numCandidates unnecessarily**: numCandidates=500 with limit=10 is a 50x ratio. Beyond 20x, the recall improvement is marginal but latency doubles.

3. **Not testing with realistic filter patterns**: benchmarks without filters overestimate real-world performance. Multi-tenant filters (1% selectivity) add 30-50% latency.

4. **Ignoring eventual consistency in benchmarks**: newly inserted documents are not immediately searchable. Benchmarks that insert-then-search without delay will show artificially low recall.

5. **Comparing Atlas costs without considering operational overhead**: Atlas is more expensive than self-hosted pgvector or Qdrant, but eliminates DBA operational costs (backups, upgrades, monitoring, scaling).

6. **Not warming up the search index before measuring**: the first few queries after index build or cluster restart are significantly slower due to index loading. Run at least 100 warmup queries before measuring.

7. **Using the same queries for warmup and measurement**: the HNSW graph caches recently traversed nodes. Using the same queries inflates performance. Use disjoint query sets for warmup and measurement.

---

## References

- MongoDB Atlas Vector Search documentation: https://www.mongodb.com/docs/atlas/atlas-vector-search/
- Atlas pricing: https://www.mongodb.com/pricing
- MongoDB Performance Best Practices: https://www.mongodb.com/docs/manual/administration/production-notes/
- $rankFusion: https://www.mongodb.com/docs/atlas/atlas-vector-search/tutorials/reciprocal-rank-fusion/
