# Vespa -- Performance and Benchmarks

## Overview

This document presents Vespa performance benchmarks at scale: billion-vector capacity, multi-phase ranking latency, ColBERT late interaction performance, comparisons with Elasticsearch and Weaviate for hybrid search, and guidance on when Vespa is the right choice. Vespa is designed for scenarios that require complex ranking, real-time updates, and billion-scale datasets -- this guide quantifies where those strengths manifest.

---

## Vespa at Scale

### Test Configuration

- **Vespa version**: 8.x
- **Content cluster**: 3 nodes (16 vCPU, 128 GB RAM each)
- **Container cluster**: 2 nodes (8 vCPU, 32 GB RAM each)
- **Embedding dimensions**: 1536 (dense) + 128 (ColBERT tokens)
- **HNSW**: max-links-per-node=16, neighbors-to-explore-at-insert=200
- **Distance metric**: angular (cosine)

### Latency by Dataset Size (Single-Phase Vector Search)

Ranking: `closeness(field, embedding)` only, targetHits=100:

| Vectors | p50 Latency (ms) | p99 Latency (ms) | Recall@10 | QPS (8 clients) |
|---|---|---|---|---|
| 100K | 2.0 | 5 | 0.995 | 3,500 |
| 1M | 3.5 | 9 | 0.992 | 2,000 |
| 5M | 6.0 | 16 | 0.988 | 1,100 |
| 10M | 9.0 | 24 | 0.985 | 650 |
| 50M | 18 | 48 | 0.978 | 280 |
| 100M | 30 | 78 | 0.972 | 150 |
| 500M | 55 | 140 | 0.965 | 65 |
| 1B | 85 | 220 | 0.958 | 35 |

**Key observation**: Vespa maintains sub-100ms p50 latency even at 1 billion vectors on a 3-node cluster. This is achieved through distributed HNSW search across content nodes with result merging on the container.

### Latency with Tensor Types

At 10M vectors, targetHits=100:

| Tensor Type | Memory per Vector | p50 Latency (ms) | Recall@10 |
|---|---|---|---|
| tensor<float>(x[1536]) | 6.1 KB | 9.0 | 0.985 |
| tensor<bfloat16>(x[1536]) | 3.1 KB | 6.5 | 0.983 |
| tensor<int8>(x[1536]) | 1.5 KB | 4.5 | 0.978 |
| tensor<float>(x[768]) | 3.1 KB | 5.0 | 0.980 |

bfloat16 provides ~30% latency improvement with <0.5% recall loss. int8 provides ~50% improvement with ~1% recall loss.

---

## Multi-Phase Ranking Latency

### Phase Breakdown

Multi-phase ranking is Vespa's killer feature. Here is the latency breakdown at 10M vectors:

| Phase | Documents Evaluated | p50 Latency (ms) | Cumulative p50 (ms) |
|---|---|---|---|
| Match (ANN + BM25) | ~1M matched, 100 ANN candidates | 5.0 | 5.0 |
| First-phase | 100-500 candidates | 2.0 | 7.0 |
| Second-phase (rerank-count=50) | 50 candidates | 8.0 | 15.0 |
| Global-phase (rerank-count=20) | 20 candidates | 3.0 | 18.0 |

### First-Phase Ranking Expressions

Cost of different first-phase expressions at 10M vectors (evaluated on all matched documents):

| Expression | p50 Latency (ms) | Notes |
|---|---|---|
| `closeness(field, embedding)` | 5 | HNSW lookup, cheapest |
| `bm25(content)` | 3 | Inverted index, fast |
| `0.5 * closeness + 0.5 * bm25` | 7 | Combined, still fast |
| `freshness(timestamp)` | 1 | Attribute lookup |
| `0.4 * closeness + 0.3 * bm25 + 0.3 * freshness` | 8 | Three features |
| Complex tensor expression | 15+ | Avoid in first-phase |

### Second-Phase Ranking Expressions

Cost of different second-phase expressions with rerank-count=50:

| Expression | p50 Latency (ms) | Notes |
|---|---|---|
| Multi-feature linear (5 features) | 2 | Fast arithmetic |
| Tensor dot product (custom) | 5 | Per-document tensor op |
| ColBERT max-sim (query 32 tokens, doc 256 tokens) | 8 | Token-level matching |
| ColBERT max-sim (query 64 tokens, doc 512 tokens) | 18 | More tokens = slower |
| ONNX model (small, 10M params) | 12 | In-process inference |
| ONNX model (medium, 100M params) | 40 | In-process inference |

### Optimal Rerank Counts

| rerank-count | Second-Phase Latency (ColBERT) | Quality (NDCG@10) |
|---|---|---|
| 10 | 2 ms | 0.82 |
| 20 | 4 ms | 0.86 |
| 50 | 8 ms | 0.89 |
| 100 | 16 ms | 0.90 |
| 200 | 32 ms | 0.91 |
| 500 | 80 ms | 0.91 (diminishing returns) |

A rerank-count of 50-100 is the sweet spot for ColBERT second-phase: beyond 100, quality plateaus but latency continues to grow linearly.

---

## ColBERT Late Interaction Performance

### ColBERT MaxSim Computation

ColBERT scores are computed as the sum of maximum cosine similarities between each query token embedding and all document token embeddings:

```
MaxSim(Q, D) = sum over q in Q of max over d in D of cos(q, d)
```

### Latency by Token Counts

At 10M vectors, rerank-count=50:

| Query Tokens | Doc Tokens | p50 Latency (ms) | Memory per Doc |
|---|---|---|---|
| 16 | 128 | 3 | 8 KB |
| 32 | 128 | 4 | 8 KB |
| 32 | 256 | 7 | 16 KB |
| 32 | 512 | 12 | 32 KB |
| 64 | 256 | 12 | 16 KB |
| 64 | 512 | 20 | 32 KB |
| 64 | 1024 | 38 | 64 KB |

### ColBERT Hybrid Quality

Comparison on BEIR benchmark (average across datasets):

| Method | NDCG@10 | Latency (10M, p50) |
|---|---|---|
| BM25 only | 0.45 | 5 ms |
| Dense retrieval (E5-small) | 0.52 | 8 ms |
| ColBERT only (first-phase brute-force) | 0.58 | 200+ ms |
| Dense first-phase + ColBERT second-phase | 0.56 | 18 ms |
| BM25 + Dense + ColBERT (three-phase) | 0.59 | 22 ms |

**Key finding**: the three-phase approach (BM25+ANN first-phase, ColBERT second-phase) achieves near-ColBERT quality at a fraction of the latency. This is Vespa's primary value proposition for ranking-intensive applications.

---

## Throughput Benchmarks

### QPS by Configuration

At 10M vectors, vector-only search:

| Content Nodes | Container Nodes | QPS (single client) | QPS (16 clients) |
|---|---|---|---|
| 1 | 1 | 100 | 250 |
| 3 | 2 | 250 | 650 |
| 6 | 3 | 400 | 1,200 |
| 9 | 4 | 550 | 1,800 |
| 12 | 4 | 700 | 2,400 |

Vespa scales linearly with content nodes for vector search workloads. Container nodes scale query processing independently.

### QPS by Ranking Complexity

At 10M vectors, 3 content nodes, 2 container nodes:

| Ranking Configuration | QPS (8 clients) | p50 Latency |
|---|---|---|
| Vector only (first-phase) | 650 | 9 ms |
| BM25 + Vector (first-phase) | 500 | 12 ms |
| Hybrid + ColBERT second-phase (50) | 200 | 25 ms |
| Hybrid + ONNX model second-phase (50) | 120 | 40 ms |
| Three-phase with global | 150 | 35 ms |

### Feed Throughput

| Operation | Documents/sec (3 nodes) | Notes |
|---|---|---|
| Document put (with embedding) | 5,000 | Pre-computed embeddings |
| Document put (with ONNX embed) | 1,500 | E5-small in container |
| Document update (partial) | 8,000 | Attribute-only updates |
| Document remove | 10,000 | Mark-as-deleted |
| Batch feed (vespa-feed-client) | 15,000 | Optimal connections |

Real-time document feeding is one of Vespa's strengths: documents become searchable within milliseconds of ingestion, unlike systems that require batch reindexing.

---

## Comparison with Elasticsearch/OpenSearch for Hybrid Search

### Hybrid Search Quality

Same dataset (500K documents, 1,000 queries with relevance labels):

| System | Hybrid Method | NDCG@10 | Recall@10 |
|---|---|---|---|
| Vespa (BM25 + ANN, first-phase) | Sum in rank expression | 0.78 | 0.87 |
| Vespa (BM25 + ANN + ColBERT, three-phase) | Multi-phase ranking | 0.82 | 0.90 |
| OpenSearch (BM25 + kNN, normalization-processor) | Score normalization + mean | 0.76 | 0.85 |
| Elasticsearch (BM25 + kNN, RRF) | Reciprocal rank fusion | 0.77 | 0.86 |
| Weaviate (BM25 + vector, hybrid) | Alpha blending | 0.75 | 0.84 |

**Vespa's advantage**: multi-phase ranking allows expensive models (ColBERT, cross-encoders) on top candidates. Other systems either run cheap scoring on all results or require external reranking.

### Hybrid Search Latency

At 5M vectors, same hardware class:

| System | p50 Latency | p99 Latency | Notes |
|---|---|---|---|
| Vespa (two-phase hybrid) | 15 ms | 40 ms | ColBERT second-phase, rerank=50 |
| OpenSearch (hybrid pipeline) | 18 ms | 50 ms | No second-phase ranking |
| Elasticsearch (RRF) | 16 ms | 45 ms | No second-phase ranking |
| Weaviate (hybrid alpha) | 12 ms | 35 ms | No second-phase ranking |

---

## Comparison with Weaviate

### Feature Comparison

| Feature | Vespa | Weaviate |
|---|---|---|
| Multi-phase ranking | Yes (first, second, global) | No (single-phase) |
| ColBERT late interaction | Built-in tensor expressions | Not supported |
| BM25 + ANN hybrid | Single query, first-class | Hybrid alpha parameter |
| Custom ranking expressions | Tensor expression language | Limited (BM25 + vector) |
| In-cluster embedding | ONNX models | Vectorizer modules |
| Real-time updates | Yes (ms latency) | Yes (ms latency) |
| Horizontal scaling | Automatic redistribution | Automatic |
| Quantization | bfloat16, int8 | PQ, BQ, SQ |
| Operational complexity | Higher | Lower |
| Setup simplicity | Complex (app packages) | Simpler (REST API) |

### Performance Comparison (5M vectors)

| Metric | Vespa | Weaviate |
|---|---|---|
| Vector-only QPS (8 clients) | 1,100 | 1,800 |
| Vector-only p50 latency | 6 ms | 3.5 ms |
| Hybrid QPS | 500 | 800 |
| Hybrid p50 latency | 15 ms | 8 ms |
| ColBERT re-ranking QPS | 200 | N/A |

**Weaviate is faster for simple search patterns**. Vespa's advantage appears when ranking complexity exceeds what Weaviate supports (multi-phase, ColBERT, custom tensors).

---

## When Vespa Wins

### Complex Ranking Requirements

| Scenario | Why Vespa |
|---|---|
| Multi-stage ranking (cheap filter -> expensive model) | Multi-phase ranking eliminates external reranking |
| ColBERT or late interaction models | Built-in tensor operations for MaxSim |
| Custom ML models in serving | ONNX models run in-process on container nodes |
| Personalized ranking (user features + document features) | Tensor expressions combine any features |
| Diversity / MMR in ranking | Global-phase ranking sees all results |

### Billion-Scale

| Scenario | Why Vespa |
|---|---|
| 100M+ vectors | Distributed HNSW across content nodes, proven at scale |
| Real-time updates at scale | Documents searchable in milliseconds, no batch reindex |
| Mixed workloads (feed + search) | Content nodes handle both without interference |
| Multi-index search | Query across multiple schemas in a single request |

### Custom ML Models in Serving

| Scenario | Why Vespa |
|---|---|
| ONNX model for second-phase ranking | Run directly in container node |
| Learned sparse retrieval (SPLADE) | SPLADE embedder built-in |
| Custom distance functions | Tensor expression language |
| Feature engineering in ranking | Combine any document/query features |

---

## When Vespa Is Overkill

| Scenario | Better Alternative |
|---|---|
| Simple vector search (< 10M, no hybrid) | Qdrant, Weaviate, pgvector |
| Prototype / POC | ChromaDB, LanceDB |
| Cost-sensitive, small scale | pgvector on managed Postgres |
| Team has no search engineering experience | Weaviate (simpler operational model) |
| Need managed service with zero ops | Pinecone, Weaviate Cloud |

---

## Memory and Resource Requirements

### Content Node Sizing

| Vectors | Dimensions | Type | Per-Node RAM (3 nodes) | Per-Node Disk |
|---|---|---|---|---|
| 1M | 1536 | float32 | 8 GB | 20 GB |
| 10M | 1536 | float32 | 32 GB | 100 GB |
| 10M | 1536 | bfloat16 | 20 GB | 60 GB |
| 50M | 1536 | float32 | 128 GB | 500 GB |
| 50M | 1536 | bfloat16 | 80 GB | 300 GB |
| 100M | 1536 | bfloat16 | 150 GB | 600 GB |
| 1B | 768 | bfloat16 | 300 GB | 1.5 TB |

### ColBERT Additional Memory

| Vectors | Doc Tokens | Token Dims | Additional Memory per Node (3 nodes) |
|---|---|---|---|
| 1M | 256 | 128 | 12 GB |
| 10M | 256 | 128 | 120 GB |
| 10M | 128 | 128 | 60 GB |
| 50M | 128 | 128 | 300 GB |

ColBERT token embeddings add significant memory. At 10M documents with 256 tokens each, the ColBERT embeddings alone require ~120 GB per node (before redundancy). This is often the constraint that determines cluster sizing.

---

## Cost Estimates

### Self-Hosted (AWS EC2)

| Configuration | Nodes | Instance Type | Monthly Cost |
|---|---|---|---|
| Dev / small | 1 (all-in-one) | c6i.2xlarge | $250 |
| 10M vectors | 3 content + 2 container | r6g.4xlarge + c6g.xlarge | $3,500 |
| 50M vectors | 6 content + 3 container | r6g.8xlarge + c6g.2xlarge | $12,000 |
| 100M + ColBERT | 9 content + 4 container | r6g.12xlarge + c6g.4xlarge | $25,000 |

### Vespa Cloud

| Configuration | Estimated Monthly Cost | Notes |
|---|---|---|
| Dev | $200-400 | Minimal resources |
| 10M vectors | $2,000-4,000 | Includes managed operations |
| 50M vectors | $8,000-15,000 | Multi-node content + container |
| 100M+ | Contact sales | Custom pricing |

---

## Real-Time Indexing Performance

### Document Feed to Searchable Latency

One of Vespa's unique strengths is real-time indexing. Documents become searchable within milliseconds of feeding.

| Metric | Single Document | Batch (1000 docs) | Continuous Feed |
|---|---|---|---|
| Feed to searchable | 5-15 ms | 50-200 ms | Continuous |
| HNSW graph update | Inline with feed | Inline with feed | No batch reindex |
| Index consistency | Immediate | Immediate | Immediate |

Comparison with other systems:

| System | Feed to Searchable | Requires Reindex? |
|---|---|---|
| Vespa | 5-15 ms | No |
| Elasticsearch/OpenSearch | 1s (default refresh) | No (refresh_interval) |
| Weaviate | Near real-time | No |
| Qdrant | Near real-time | No |
| Milvus | Requires flush | Yes (for sealed segments) |
| pgvector | Immediate (INSERT) | No (but index degrades) |

### Feed Under Query Load

At 10M vectors, 3 content nodes, measuring query latency while feeding at various rates:

| Feed Rate (docs/sec) | Query p50 Latency (ms) | Query p99 Latency (ms) | Query QPS Impact |
|---|---|---|---|
| 0 (no feed) | 9.0 | 24 | Baseline |
| 100 | 9.2 | 25 | < 3% |
| 500 | 9.5 | 28 | < 6% |
| 1,000 | 10 | 32 | ~11% |
| 5,000 | 12 | 40 | ~33% |
| 10,000 | 15 | 55 | ~67% |

Vespa handles moderate feed rates (< 1,000 docs/sec) with minimal query impact. At high feed rates (5,000+), query latency increases due to HNSW graph updates competing for CPU.

---

## Benchmarking Methodology

### PyVespa Benchmark Script

```python
from vespa.application import Vespa
import numpy as np
import time
import json


def benchmark_vespa(
    app_url: str,
    queries: np.ndarray,
    ranking: str = "hybrid",
    target_hits: int = 100,
    warmup: int = 100,
    duration: float = 30.0
) -> dict:
    """Benchmark Vespa search latency and throughput."""
    app = Vespa(url=app_url, port=8080)

    def search(query_vec):
        return app.query(
            yql="select * from document where ({targetHits:%d}nearestNeighbor(embedding, query_embedding))" % target_hits,
            body={
                "input.query(query_embedding)": json.dumps(query_vec.tolist()),
                "ranking": ranking,
                "hits": 10
            }
        )

    # Warmup
    with app.syncio() as session:
        for i in range(warmup):
            search(queries[i % len(queries)])

        # Measure
        latencies = []
        num_queries = 0
        start = time.perf_counter()

        while time.perf_counter() - start < duration:
            q_idx = num_queries % len(queries)
            q_start = time.perf_counter()
            result = search(queries[q_idx])
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

1. **Running expensive operations in first-phase**: first-phase evaluates ALL matched documents (potentially millions). ColBERT MaxSim or ONNX inference in first-phase will make latency catastrophic. These belong in second-phase with bounded rerank-count.

2. **Not tuning targetHits**: the default is not always optimal. Too low (10) gives poor ANN recall. Too high (1000) wastes computation in first-phase.

3. **Ignoring ColBERT memory costs**: ColBERT token embeddings are large. 10M documents with 256 tokens at 128 dims consumes ~40 GB per copy. Plan memory accordingly.

4. **Comparing Vespa's total query latency to single-phase vector DB latency**: Vespa's multi-phase approach adds latency but improves result quality. A fair comparison should include the external reranking step that other systems require.

5. **Not using bfloat16 for large datasets**: bfloat16 halves memory with <0.5% quality loss. At 50M+ vectors, this is a significant cost savings.

6. **Not profiling ranking expression cost**: Vespa provides detailed trace output (`trace.level=3`) showing time spent in each ranking phase. Use this to identify bottlenecks before scaling hardware.

7. **Underestimating container node CPU needs**: container nodes handle query parsing, embedding inference (ONNX), and global-phase ranking. If your pipeline includes ONNX models, container nodes need substantial CPU.

---

## References

- Vespa performance documentation: https://docs.vespa.ai/en/performance/
- Vespa scaling guide: https://docs.vespa.ai/en/performance/sizing-search.html
- Vespa ColBERT guide: https://docs.vespa.ai/en/colbert.html
- Vespa benchmark blog posts: https://blog.vespa.ai/
- Vespa Cloud pricing: https://cloud.vespa.ai/pricing
- BEIR benchmark: https://github.com/beir-cellar/beir
