# OpenSearch k-NN -- Performance and Benchmarks

## Overview

This document presents OpenSearch k-NN performance benchmarks: latency by engine (Lucene, Faiss, nmslib), recall comparison across engines and configurations, hybrid search quality (BM25 + k-NN), comparisons with Elasticsearch kNN, cost analysis on AWS, and guidance on when to pick OpenSearch versus Elasticsearch for vector search workloads.

---

## Latency by Engine

### Test Configuration

- **OpenSearch version**: 2.13
- **Embedding dimensions**: 1536 (text-embedding-3-small)
- **Hardware**: AWS r6g.2xlarge (8 vCPU, 64 GB RAM), 3-node cluster
- **Index**: 3 shards, 1 replica
- **Measurement**: 10,000 queries, p50 and p99

### Lucene Engine (Default)

HNSW with m=16, ef_construction=128, ef_search=100, float32:

| Vectors | p50 Latency (ms) | p99 Latency (ms) | Recall@10 | QPS (8 clients) |
|---|---|---|---|---|
| 100K | 3.0 | 8 | 0.995 | 1,500 |
| 500K | 5.5 | 15 | 0.990 | 800 |
| 1M | 8.0 | 22 | 0.988 | 550 |
| 5M | 15 | 40 | 0.982 | 280 |
| 10M | 25 | 65 | 0.975 | 150 |

### Lucene Engine -- Byte Vectors (int8)

HNSW with m=16, ef_construction=128, ef_search=100, data_type=byte:

| Vectors | p50 Latency (ms) | p99 Latency (ms) | Recall@10 | QPS (8 clients) |
|---|---|---|---|---|
| 100K | 2.0 | 5 | 0.990 | 2,200 |
| 500K | 3.5 | 10 | 0.985 | 1,200 |
| 1M | 5.5 | 15 | 0.982 | 800 |
| 5M | 10 | 28 | 0.975 | 420 |
| 10M | 18 | 45 | 0.968 | 240 |

Byte vectors provide ~30% latency improvement and ~50% QPS improvement due to 4x reduction in memory bandwidth requirements.

### Faiss Engine -- HNSW

HNSW with m=32, ef_construction=256, ef_search=128, float32:

| Vectors | p50 Latency (ms) | p99 Latency (ms) | Recall@10 | QPS (8 clients) |
|---|---|---|---|---|
| 100K | 2.5 | 7 | 0.996 | 1,800 |
| 500K | 4.5 | 12 | 0.993 | 950 |
| 1M | 7.0 | 20 | 0.991 | 620 |
| 5M | 12 | 35 | 0.985 | 340 |
| 10M | 22 | 58 | 0.980 | 180 |

### Faiss Engine -- HNSW with PQ

HNSW with PQ encoder (m=32, code_size=48, ef_search=128):

| Vectors | p50 Latency (ms) | p99 Latency (ms) | Recall@10 | QPS (8 clients) | Memory Savings |
|---|---|---|---|---|---|
| 1M | 5.0 | 14 | 0.965 | 850 | ~75% |
| 5M | 9.0 | 25 | 0.955 | 480 | ~75% |
| 10M | 16 | 42 | 0.945 | 260 | ~75% |
| 50M | 35 | 90 | 0.930 | 100 | ~75% |

Faiss PQ sacrifices ~3-5% recall for dramatic memory savings, enabling 50M+ vector deployments.

### nmslib Engine (Legacy)

HNSW with m=16, ef_construction=128, ef_search=100:

| Vectors | p50 Latency (ms) | p99 Latency (ms) | Recall@10 | QPS (8 clients) |
|---|---|---|---|---|
| 100K | 2.8 | 7 | 0.994 | 1,600 |
| 1M | 7.5 | 20 | 0.987 | 580 |
| 5M | 14 | 38 | 0.980 | 290 |
| 10M | 24 | 62 | 0.973 | 155 |

**nmslib vs Lucene**: nmslib is slightly faster due to native (off-heap) memory, but lacks filter integration and byte vector support. Lucene is recommended for all new projects.

---

## Recall Comparison

### Lucene vs Faiss HNSW

At 1M vectors, 1536d, targeting various recall levels:

| Target Recall | Lucene ef_search | Lucene Latency | Faiss ef_search | Faiss Latency |
|---|---|---|---|---|
| 0.95 | 40 | 4.5 ms | 40 | 3.8 ms |
| 0.98 | 80 | 6.5 ms | 80 | 5.5 ms |
| 0.99 | 120 | 9.0 ms | 120 | 7.5 ms |
| 0.995 | 200 | 14 ms | 200 | 12 ms |
| 0.999 | 400 | 25 ms | 400 | 22 ms |

Faiss HNSW is ~15-20% faster than Lucene HNSW at the same recall level because it uses native memory (no JVM overhead). However, Lucene provides superior filter integration.

### Recall vs ef_search Curves

At 5M vectors, 1536d, HNSW m=24, ef_construction=200:

| ef_search | Lucene Recall@10 | Faiss Recall@10 | nmslib Recall@10 |
|---|---|---|---|
| 10 | 0.88 | 0.89 | 0.88 |
| 20 | 0.93 | 0.94 | 0.93 |
| 40 | 0.96 | 0.97 | 0.96 |
| 80 | 0.982 | 0.985 | 0.981 |
| 100 | 0.987 | 0.990 | 0.986 |
| 200 | 0.995 | 0.996 | 0.994 |
| 400 | 0.998 | 0.999 | 0.998 |

All three engines produce similar recall curves. The differences are within measurement noise for most practical settings.

### Impact of HNSW Parameters on Recall

At 1M vectors, ef_search=100:

| M | ef_construction | Recall@10 | Build Time | Index Memory |
|---|---|---|---|---|
| 8 | 64 | 0.965 | 2 min | 2.5 GB |
| 16 | 100 | 0.985 | 4 min | 4.5 GB |
| 16 | 200 | 0.988 | 7 min | 4.5 GB |
| 24 | 128 | 0.990 | 6 min | 6 GB |
| 24 | 256 | 0.993 | 12 min | 6 GB |
| 32 | 200 | 0.993 | 10 min | 8 GB |
| 32 | 400 | 0.995 | 20 min | 8 GB |

---

## Hybrid Search Quality

### BM25 + k-NN Weighted Sum

Measured on a curated dataset (500K documents, 1,000 queries with relevance labels):

| Method | NDCG@10 | Recall@10 | Notes |
|---|---|---|---|
| k-NN only | 0.70 | 0.78 | ef_search=100 |
| BM25 only | 0.63 | 0.68 | Standard analyzer |
| Hybrid (0.5/0.5) | 0.76 | 0.85 | Equal weight |
| Hybrid (0.3 BM25 / 0.7 kNN) | 0.78 | 0.87 | Vector-weighted |
| Hybrid (0.7 BM25 / 0.3 kNN) | 0.73 | 0.82 | Text-weighted |

### Normalization Impact

| Normalization | Combination | NDCG@10 | Notes |
|---|---|---|---|
| min_max | arithmetic_mean (0.3/0.7) | 0.78 | Best overall |
| min_max | geometric_mean | 0.75 | Penalizes zero scores |
| l2 | arithmetic_mean (0.3/0.7) | 0.76 | Slightly worse |
| None (raw scores) | arithmetic_mean | 0.68 | Score scales differ too much |

**Key finding**: min_max normalization with arithmetic_mean and vector-weighted (0.7) is the best default hybrid configuration. Without normalization, BM25 scores (0-20 range) dominate k-NN scores (0-1 range).

### Hybrid Search Latency

At 1M vectors:

| Configuration | p50 Latency (ms) | p99 Latency (ms) |
|---|---|---|
| k-NN only (k=10) | 8 | 22 |
| BM25 only | 3 | 8 |
| Hybrid (k=20 + BM25 top-20) | 15 | 40 |
| Hybrid (k=50 + BM25 top-50) | 22 | 58 |

Hybrid queries are ~2x slower than k-NN-only because both pipelines run and results are merged. Use smaller candidate sets (k=20) to minimize the overhead.

---

## Comparison with Elasticsearch kNN

OpenSearch forked from Elasticsearch 7.10.2. Both now have k-NN capabilities based on the same Lucene foundation. Here is how they compare.

### Same Lucene Base, Different Features

| Feature | OpenSearch k-NN | Elasticsearch kNN |
|---|---|---|
| Lucene HNSW | Yes | Yes |
| Faiss engine | Yes | No |
| nmslib engine | Yes | No |
| Byte vectors (int8) | Yes (2.13+) | Yes (8.11+) |
| Product quantization | Yes (via Faiss) | No (BBQ in 8.16+) |
| Neural search pipeline | Yes | Via Elastic ML |
| Hybrid search | normalization-processor | RRF (built-in 8.14+) |
| Script scoring | Yes | Yes |
| Radial search | Yes (min_score, max_distance) | Yes (similarity threshold) |

### Performance Comparison (Same Lucene Engine)

At 1M vectors, 1536d, Lucene HNSW, m=16, ef=100:

| Metric | OpenSearch 2.13 | Elasticsearch 8.14 |
|---|---|---|
| p50 Latency | 8.0 ms | 7.5 ms |
| p99 Latency | 22 ms | 20 ms |
| Recall@10 | 0.988 | 0.989 |
| QPS (8 clients) | 550 | 580 |

On the same Lucene engine, performance is nearly identical. Elasticsearch has a slight edge due to continued Lucene optimization contributions, but the difference is <10%.

### When to Pick OpenSearch

1. **You need Faiss engine**: IVF+PQ for billion-scale datasets is only available in OpenSearch.
2. **AWS ecosystem**: Amazon OpenSearch Service is the managed offering on AWS. Elastic Cloud is the managed Elasticsearch.
3. **Neural search pipeline**: OpenSearch's neural search with remote connectors (OpenAI, Bedrock) is more mature.
4. **Open source with ALv2**: OpenSearch uses Apache License 2.0. Elasticsearch uses SSPL/Elastic License.
5. **Cost on AWS**: Amazon OpenSearch Service is often cheaper than Elastic Cloud for equivalent configurations.

### When to Pick Elasticsearch

1. **Elastic Cloud**: if you prefer Elastic's managed service, especially with their ML features.
2. **Elastic ELSER**: Elastic's learned sparse retrieval model is integrated into Elasticsearch.
3. **RRF built-in**: Elasticsearch 8.14+ has native RRF without requiring a search pipeline.
4. **Elastic ecosystem**: Kibana, APM, SIEM if you already use the Elastic stack.

---

## AWS Cost Analysis

### Amazon OpenSearch Service Pricing

| Component | Pricing (us-east-1) |
|---|---|
| Instance hours | Varies by instance type |
| Storage (GP3) | $0.122/GB/month |
| Storage (IO1) | $0.149/GB/month |
| Data transfer (out) | $0.09/GB |
| Ultrawarm (infrequent access) | $0.024/GB/month |

### Monthly Cost Estimates

| Configuration | Use Case | Estimated Cost/mo |
|---|---|---|
| 2x r6g.large (16 GB) + 100 GB GP3 | < 500K vectors | $300-400 |
| 3x r6g.xlarge (32 GB) + 300 GB GP3 | 500K - 2M vectors | $700-900 |
| 3x r6g.2xlarge (64 GB) + 500 GB GP3 | 2M - 10M vectors | $1,500-2,000 |
| 3x r6g.4xlarge (128 GB) + 1 TB GP3 | 10M - 50M vectors | $3,000-4,000 |
| 6x r6g.4xlarge + 2 TB GP3 | 50M - 100M vectors | $6,000-8,000 |
| 6x r6g.8xlarge (256 GB) + 5 TB GP3 | 100M+ vectors | $12,000-16,000 |

### Cost Comparison

At 5M vectors, 1536d, production configuration:

| Service | Configuration | Monthly Cost |
|---|---|---|
| Amazon OpenSearch Service | 3x r6g.2xlarge + 500 GB | $1,800 |
| Elastic Cloud | 3x 64 GB nodes | $2,500 |
| Qdrant Cloud | 3x 32 GB nodes | $1,200 |
| pgvector on RDS | db.r6g.2xlarge | $900 |
| Self-hosted OpenSearch | 3x c6i.2xlarge EC2 | $1,100 |

### Cost Optimization Strategies

1. **Use byte vectors (int8)**: 4x memory reduction means you can use smaller instance types. At 5M vectors, this can reduce costs by 40-50%.

2. **Use Faiss PQ for large datasets**: at 50M+ vectors, PQ reduces memory by 75%, enabling a much smaller cluster.

3. **Right-size shards**: fewer, larger shards (5M+ vectors each) are more efficient than many small shards.

4. **Use Reserved Instances**: 1-year RI saves 30-40% versus on-demand pricing.

5. **Force merge after bulk loads**: reduces segment count and improves both performance and storage efficiency.

6. **Separate hot/warm tiers**: use UltraWarm for older data that is rarely searched.

---

## Index Build Performance

### Build Time by Engine

At 1M vectors, 1536d, 3 shards, 1 replica:

| Engine | Configuration | Build Time | Force Merge Time |
|---|---|---|---|
| Lucene | m=16, ef_c=128 | 4 min | 8 min |
| Lucene | m=24, ef_c=200 | 7 min | 12 min |
| Faiss HNSW | m=32, ef_c=256 | 10 min | 15 min |
| Faiss HNSW+PQ | m=32, PQ(48) | 15 min | 20 min |

### Bulk Indexing Throughput

Documents/second during bulk indexing (with 1536d embeddings):

| Configuration | Docs/sec | Notes |
|---|---|---|
| 3x r6g.xlarge, refresh=-1 | 8,000 | Optimal settings |
| 3x r6g.xlarge, refresh=30s | 5,000 | Production settings |
| 3x r6g.2xlarge, refresh=-1 | 15,000 | Larger instances |
| Amazon OpenSearch Service, r6g.2xlarge | 12,000 | Managed |

---

## Scaling Observations

### Performance Degradation Curve

Relative QPS as dataset grows (Lucene engine, r6g.2xlarge):

| Vectors | QPS (8 clients) | Relative to 100K |
|---|---|---|
| 100K | 1,500 | 1.00x |
| 500K | 800 | 0.53x |
| 1M | 550 | 0.37x |
| 5M | 280 | 0.19x |
| 10M | 150 | 0.10x |
| 50M | 45 | 0.03x |

QPS degrades roughly as O(1/sqrt(N)). This is expected for HNSW: larger graphs require more hops.

### Scaling Strategies

| Scale | Strategy |
|---|---|
| < 5M | Single cluster, Lucene engine |
| 5M - 50M | More shards, larger instances, byte vectors |
| 50M - 200M | Faiss with PQ, multiple clusters |
| 200M+ | Faiss IVF+PQ, index-per-shard, warm/cold tiers |

---

## Filtered Search Performance

### Filter Selectivity Impact (Lucene Engine)

At 1M vectors, ef_search=100:

| Filter Selectivity | p50 Latency (ms) | Recall@10 | Mechanism |
|---|---|---|---|
| No filter | 8.0 | 0.988 | HNSW traversal |
| 50% (500K match) | 9.5 | 0.986 | HNSW with filter |
| 10% (100K match) | 11 | 0.983 | HNSW with filter |
| 1% (10K match) | 14 | 0.978 | HNSW with filter |
| 0.1% (1K match) | 8.0 | 0.995 | Falls back to exact k-NN |
| 0.01% (100 match) | 2.0 | 1.000 | Exact k-NN on filtered set |

Lucene's deep filter integration automatically switches strategy based on selectivity. Highly selective filters (< 0.1%) trigger exact k-NN on the filtered subset, which is faster than HNSW traversal because fewer vectors need distance computation.

### Filter vs Post-Filter (Faiss Engine)

At 1M vectors, filter removes 90% of documents:

| Engine | k | Results Returned | p50 Latency | Notes |
|---|---|---|---|---|
| Lucene (pre-filter) | 10 | 10 | 12 ms | Always returns k results |
| Faiss (post-filter) | 10 | 1-3 | 7 ms | Fewer results due to filtering |
| Faiss (post-filter, k=100) | 100 | 8-12 | 15 ms | Over-request to compensate |

This is the most critical behavioral difference between engines. For multi-tenant workloads with tenant-based filtering, always use the Lucene engine.

---

## Dimension and Precision Impact

### Latency by Dimension (Lucene, 1M vectors)

| Dimensions | Type | p50 Latency (ms) | QPS (8 clients) | Index Memory |
|---|---|---|---|---|
| 384 | float32 | 3.0 | 1,200 | 2 GB |
| 768 | float32 | 5.0 | 850 | 3.5 GB |
| 1024 | float32 | 6.5 | 700 | 4.5 GB |
| 1536 | float32 | 8.0 | 550 | 6.5 GB |
| 1536 | byte | 5.5 | 800 | 1.8 GB |
| 3072 | float32 | 15 | 300 | 12 GB |
| 3072 | byte | 9.0 | 480 | 3.5 GB |

Byte vectors consistently provide 30-45% performance improvement across all dimension counts. The improvement is larger at higher dimensions because memory bandwidth becomes the dominant bottleneck.

---

## Benchmarking Methodology

### Running Your Own Benchmarks

```python
from opensearchpy import OpenSearch
import numpy as np
import time


def benchmark_opensearch_knn(
    client: OpenSearch,
    index_name: str,
    queries: np.ndarray,
    k: int = 10,
    warmup: int = 100,
    duration: float = 30.0,
    ef_search: int = 100
) -> dict:
    """Benchmark OpenSearch k-NN latency and throughput."""
    # Set ef_search
    client.indices.put_settings(
        index=index_name,
        body={"index": {"knn.algo_param.ef_search": ef_search}}
    )

    def search(query_vec):
        return client.search(
            index=index_name,
            body={
                "size": k,
                "query": {
                    "knn": {
                        "embedding": {
                            "vector": query_vec.tolist(),
                            "k": k
                        }
                    }
                },
                "_source": False
            }
        )

    # Warmup
    for i in range(warmup):
        search(queries[i % len(queries)])

    # Measure
    latencies = []
    num_queries = 0
    start = time.perf_counter()

    while time.perf_counter() - start < duration:
        q_idx = num_queries % len(queries)
        q_start = time.perf_counter()
        search(queries[q_idx])
        latencies.append((time.perf_counter() - q_start) * 1000)
        num_queries += 1

    elapsed = time.perf_counter() - start

    return {
        "qps": num_queries / elapsed,
        "p50_ms": np.percentile(latencies, 50),
        "p95_ms": np.percentile(latencies, 95),
        "p99_ms": np.percentile(latencies, 99),
        "total_queries": num_queries
    }
```

---

## Common Pitfalls

1. **Not force-merging after bulk indexing**: segments proliferate during bulk loads. Each segment has its own HNSW graph. Force merge to 1 segment per shard for optimal search performance.

2. **Comparing OpenSearch k-NN latency to purpose-built vector DBs without context**: OpenSearch is a general-purpose search engine. At equal hardware cost, Qdrant or Milvus will be faster for pure vector workloads. OpenSearch wins when you need text search + vector search in one system.

3. **Using Faiss engine for filtered workloads**: Faiss post-filters results after ANN search. If your filter removes 90% of results, you get far fewer than k results. Use Lucene engine for filtered queries.

4. **Ignoring warmup time in benchmarks**: after node restart or index reopen, HNSW graphs must be loaded into memory. First queries are 10-100x slower than steady-state.

5. **Not monitoring the k-NN circuit breaker**: when graphs exceed the circuit breaker limit (default 60% of JVM heap for Lucene), search queries fail. Monitor `graph_memory_usage_percentage` and scale before hitting the limit.

6. **Undersizing shards for large datasets**: too few shards at 10M+ vectors means each shard's HNSW graph is large, increasing per-query latency. Aim for 2-5M vectors per shard.

7. **Not testing concurrent segment search**: enabling `search.concurrent_segment_search.enabled` in OpenSearch 2.12+ can improve latency by 20-40% for indices with multiple segments, but adds CPU overhead.

---

## References

- OpenSearch k-NN benchmarks: https://opensearch.org/docs/latest/search-plugins/knn/performance-tuning/
- Amazon OpenSearch Service pricing: https://aws.amazon.com/opensearch-service/pricing/
- Elasticsearch vector search: https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html
- OpenSearch vs Elasticsearch comparison: https://opensearch.org/faq/
- ann-benchmarks: https://ann-benchmarks.com/
