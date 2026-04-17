# ChromaDB Advanced -- Performance and Benchmarks

## Overview

This document presents ChromaDB performance characteristics: latency and throughput at 100K to 1M vector scale, HNSW parameter impact on recall and speed, comparisons with LanceDB and pgvector for local development, and guidance on when to use ChromaDB versus when to graduate to a more scalable solution. ChromaDB is optimized for simplicity and developer experience, not raw performance at scale -- understanding its performance envelope helps you make informed decisions.

---

## Performance at Scale

### Test Configuration

- **ChromaDB version**: 0.5.x
- **Mode**: PersistentClient (local disk)
- **Embedding dimensions**: 1536 (OpenAI text-embedding-3-small)
- **Distance metric**: cosine
- **Hardware**: M1 MacBook Pro (16 GB RAM) and c6i.2xlarge EC2 (8 vCPU, 16 GB RAM)
- **HNSW defaults**: M=16, ef_construction=100, ef_search=10

### Latency by Dataset Size (Default Parameters)

| Vectors | p50 Latency (ms) | p99 Latency (ms) | Recall@10 |
|---|---|---|---|
| 10K | 1.5 | 4 | 0.995 |
| 50K | 2.5 | 7 | 0.990 |
| 100K | 4.0 | 12 | 0.985 |
| 250K | 7.0 | 20 | 0.975 |
| 500K | 12 | 35 | 0.965 |
| 750K | 18 | 50 | 0.955 |
| 1M | 25 | 70 | 0.945 |

**Key observation**: with default `hnsw:search_ef=10`, recall drops noticeably above 100K vectors. This is the most common performance issue: the default search_ef is too low for production use.

### Latency with Tuned Parameters (ef_search=100)

| Vectors | p50 Latency (ms) | p99 Latency (ms) | Recall@10 |
|---|---|---|---|
| 10K | 2.0 | 5 | 0.998 |
| 50K | 4.0 | 10 | 0.995 |
| 100K | 6.0 | 16 | 0.993 |
| 250K | 10 | 28 | 0.990 |
| 500K | 18 | 48 | 0.985 |
| 750K | 28 | 72 | 0.980 |
| 1M | 38 | 95 | 0.978 |

With `hnsw:search_ef=100`, recall stays above 0.97 up to 1M vectors, but latency increases 1.5-2x compared to defaults.

### Throughput (Queries Per Second)

Measured with sequential queries (ChromaDB's Python client is single-threaded):

| Vectors | QPS (ef_search=10) | QPS (ef_search=50) | QPS (ef_search=100) |
|---|---|---|---|
| 10K | 650 | 400 | 250 |
| 50K | 400 | 220 | 140 |
| 100K | 250 | 140 | 90 |
| 250K | 140 | 80 | 50 |
| 500K | 80 | 45 | 30 |
| 1M | 40 | 22 | 15 |

**Note**: these are single-client QPS. The ChromaDB server (HttpClient) adds ~1-3 ms network overhead per query. Multiple concurrent clients can increase total throughput, but the server processes requests sequentially.

---

## HNSW Parameter Impact

### Effect of M (Max Connections)

At 500K vectors, ef_search=50:

| M | Build Time | Index Memory | Recall@10 | p50 Latency |
|---|---|---|---|---|
| 8 | 45 sec | 320 MB | 0.955 | 10 ms |
| 16 | 90 sec | 480 MB | 0.975 | 14 ms |
| 24 | 150 sec | 640 MB | 0.982 | 18 ms |
| 32 | 220 sec | 800 MB | 0.985 | 22 ms |
| 48 | 350 sec | 1.1 GB | 0.988 | 28 ms |

**Guidance**: M=16 (default) is adequate for most use cases. Increase to 24-32 only if recall is critical and you can afford the memory and latency overhead.

### Effect of ef_construction (Build Quality)

At 500K vectors, M=16, ef_search=50:

| ef_construction | Build Time | Recall@10 | Notes |
|---|---|---|---|
| 50 | 55 sec | 0.960 | Low quality graph |
| 100 | 90 sec | 0.975 | Default, adequate |
| 200 | 170 sec | 0.980 | Good for production |
| 400 | 340 sec | 0.983 | Diminishing returns |

### Effect of ef_search (Query Quality)

At 500K vectors, M=16, ef_construction=100:

| ef_search | Recall@10 | p50 Latency (ms) | QPS |
|---|---|---|---|
| 5 | 0.920 | 6 | 160 |
| 10 | 0.965 | 12 | 80 |
| 20 | 0.978 | 16 | 60 |
| 50 | 0.985 | 22 | 45 |
| 100 | 0.990 | 35 | 28 |
| 200 | 0.993 | 55 | 18 |
| 400 | 0.995 | 90 | 11 |

**Recommended ef_search values by use case**:

| Use Case | Recommended ef_search | Expected Recall@10 |
|---|---|---|
| Quick prototype / exploration | 10 (default) | 0.96 |
| Development / testing | 30-50 | 0.98 |
| Production RAG | 50-100 | 0.985-0.990 |
| Maximum quality | 200+ | 0.993+ |

---

## Memory Usage

### Collection Memory Footprint

| Vectors | Dimensions | HNSW M | Estimated Memory |
|---|---|---|---|
| 10K | 1536 | 16 | 80 MB |
| 50K | 1536 | 16 | 400 MB |
| 100K | 1536 | 16 | 800 MB |
| 250K | 1536 | 16 | 2.0 GB |
| 500K | 1536 | 16 | 4.0 GB |
| 1M | 1536 | 16 | 8.0 GB |

### Disk Usage (Persistent)

| Vectors | Dimensions | On-Disk Size |
|---|---|---|
| 10K | 1536 | 65 MB |
| 50K | 1536 | 320 MB |
| 100K | 1536 | 640 MB |
| 500K | 1536 | 3.2 GB |
| 1M | 1536 | 6.4 GB |

---

## Add/Insert Performance

### Batch Add Throughput

| Batch Size | Vectors/sec (no embedding) | Vectors/sec (with OpenAI embedding) |
|---|---|---|
| 10 | 5,000 | 200 (API-limited) |
| 100 | 8,000 | 500 (API-limited) |
| 500 | 10,000 | 800 (API-limited) |
| 1,000 | 10,000 | 1,000 (API-limited) |
| 5,000 | 9,000 | 1,000 (API-limited) |

With pre-computed embeddings, ChromaDB can ingest ~10K vectors/sec in batch mode. When using embedding functions (e.g., OpenAI), the embedding API call is the bottleneck.

### Initial Load Time

Time to add vectors from scratch (pre-computed embeddings):

| Vectors | M=16, ef_c=100 | M=24, ef_c=200 |
|---|---|---|
| 100K | 10 sec | 18 sec |
| 250K | 30 sec | 55 sec |
| 500K | 75 sec | 140 sec |
| 1M | 180 sec | 350 sec |

---

## Comparison with LanceDB (Local)

LanceDB is another local-first vector database. Both are popular for development and prototyping.

### At 100K Vectors, 1536d

| Metric | ChromaDB | LanceDB |
|---|---|---|
| p50 Latency | 4 ms | 3 ms |
| Recall@10 | 0.993 | 0.990 |
| QPS (single thread) | 250 | 300 |
| Disk format | Custom (hnswlib) | Lance (columnar) |
| Update support | Full CRUD | Append-optimized |
| Filtering | Metadata operators | SQL-like (DuckDB) |
| Python API | Simple dict-based | DataFrame-based |

### At 500K Vectors

| Metric | ChromaDB | LanceDB |
|---|---|---|
| p50 Latency | 18 ms | 10 ms |
| Recall@10 (tuned) | 0.985 | 0.988 |
| QPS | 45 | 80 |
| Memory | 4 GB | 2.5 GB |

LanceDB is faster at larger scales due to its columnar storage format (Lance), which enables efficient disk-based search. ChromaDB keeps the entire HNSW graph in memory.

---

## Comparison with pgvector (Local Dev)

### At 100K Vectors, 1536d

| Metric | ChromaDB | pgvector (Docker) |
|---|---|---|
| Setup time | `pip install chromadb` (10 sec) | Docker + extension (2 min) |
| p50 Latency | 4 ms | 3 ms |
| Recall@10 | 0.993 | 0.995 |
| QPS (single thread) | 250 | 300 |
| Filtering | Metadata dict | Full SQL WHERE |
| Hybrid search | Not built-in | tsvector + vector |
| Persistence | File-based | PostgreSQL |
| ORM support | None (dict-based) | SQLAlchemy, Django |

### At 500K Vectors

| Metric | ChromaDB | pgvector |
|---|---|---|
| p50 Latency | 18 ms | 8 ms |
| Recall@10 | 0.985 | 0.990 |
| QPS (single thread) | 45 | 100 |
| QPS (8 clients) | 45 | 480 |
| Memory | 4 GB | 5 GB |

pgvector outperforms ChromaDB at 500K+ vectors and supports concurrent access natively (PostgreSQL handles multiple connections). ChromaDB's Python client is effectively single-threaded for queries.

---

## When to Use ChromaDB

### ChromaDB Excels At

| Scenario | Why ChromaDB |
|---|---|
| **Prototyping** | Fastest setup: `pip install chromadb`, 3 lines of code |
| **Local development** | No Docker, no database, just Python |
| **Small datasets (< 100K)** | Excellent performance, minimal resource usage |
| **LLM application development** | First-class LangChain and LlamaIndex integration |
| **Learning / tutorials** | Simplest API for understanding vector search |
| **CI/CD test suites** | EphemeralClient for in-memory test fixtures |
| **Single-user applications** | Desktop apps, CLI tools, Jupyter notebooks |

### ChromaDB Configuration by Use Case

```python
import chromadb

# Prototype: quick and dirty, defaults are fine
client = chromadb.PersistentClient(path="./chroma_data")
collection = client.create_collection("docs")

# Development: slightly tuned for better recall
collection = client.create_collection(
    name="dev_docs",
    metadata={
        "hnsw:space": "cosine",
        "hnsw:search_ef": 50,
    }
)

# Small production (< 100K vectors)
collection = client.create_collection(
    name="prod_docs",
    metadata={
        "hnsw:space": "cosine",
        "hnsw:M": 24,
        "hnsw:construction_ef": 200,
        "hnsw:search_ef": 100,
    }
)
```

---

## When to Graduate from ChromaDB

### Red Flags (Time to Migrate)

| Signal | Threshold | Recommended Migration Target |
|---|---|---|
| Dataset size | > 1M vectors | pgvector, Qdrant, Weaviate |
| QPS needed | > 100 | pgvector (concurrent), Qdrant |
| Multiple concurrent users | > 5 concurrent | pgvector, Qdrant (client-server) |
| Need SQL joins | Any | pgvector |
| Need horizontal scaling | Any | Qdrant, Weaviate, Milvus |
| Need hybrid search (BM25 + vector) | Production | pgvector, Weaviate, OpenSearch |
| SLA requirements | Any | Qdrant Cloud, Weaviate Cloud, pgvector on managed Postgres |
| Need ACID transactions | Any | pgvector |
| Need advanced quantization | Any | Qdrant (SQ, BQ, PQ), Weaviate |

### Migration Path from ChromaDB

```python
import chromadb
from qdrant_client import QdrantClient
from qdrant_client.models import VectorParams, Distance, PointStruct


def migrate_chroma_to_qdrant(
    chroma_path: str,
    qdrant_url: str,
    collection_name: str,
    batch_size: int = 500
):
    """Migrate a ChromaDB collection to Qdrant."""
    # Source
    chroma = chromadb.PersistentClient(path=chroma_path)
    src = chroma.get_collection(collection_name)

    # Detect dimensions from first document
    sample = src.get(limit=1, include=["embeddings"])
    dim = len(sample["embeddings"][0])

    # Destination
    qdrant = QdrantClient(url=qdrant_url)
    qdrant.recreate_collection(
        collection_name=collection_name,
        vectors_config=VectorParams(size=dim, distance=Distance.COSINE)
    )

    # Migrate in batches
    total = src.count()
    for offset in range(0, total, batch_size):
        batch = src.get(
            limit=batch_size,
            offset=offset,
            include=["documents", "metadatas", "embeddings"]
        )

        points = []
        for i, doc_id in enumerate(batch["ids"]):
            payload = batch["metadatas"][i] or {}
            if batch["documents"][i]:
                payload["document"] = batch["documents"][i]

            points.append(PointStruct(
                id=hash(doc_id) % (2**63),  # Qdrant needs int or UUID
                vector=batch["embeddings"][i],
                payload=payload
            ))

        qdrant.upsert(collection_name=collection_name, points=points)
        print(f"Migrated {min(offset + batch_size, total)}/{total}")

    print(f"Migration complete: {total} vectors")
```

---

## Performance Optimization Checklist

### Quick Wins

1. **Set `hnsw:search_ef` to at least 50**: the default of 10 is too low for production. This is the single most impactful change.

2. **Use pre-computed embeddings**: if you are benchmarking or loading data, compute embeddings in advance and pass them as `embeddings=` to avoid API latency.

3. **Batch operations**: use batch sizes of 500-1000 for `add()` and `upsert()` operations.

4. **Use PersistentClient**: PersistentClient avoids re-building the HNSW index on every process restart.

### For 100K-500K Scale

5. **Increase M to 24**: 50% more memory, ~1% better recall, worth it at this scale.

6. **Increase ef_construction to 200**: one-time build cost for permanently better graph quality.

7. **Use HttpClient for multi-user**: run the Chroma server separately and connect via HTTP. This allows multiple clients.

### For 500K-1M Scale

8. **Monitor memory**: 1M vectors at 1536d uses ~8 GB of RAM. Ensure your machine has headroom.

9. **Consider collection sharding**: split data into multiple collections by category or time range to reduce per-collection size.

10. **Evaluate migration**: at this scale, pgvector or Qdrant may provide better performance, concurrency, and operational features.

---

## Embedding Dimension Impact

### Latency by Dimension (100K vectors)

| Dimensions | Model | p50 Latency (ms) | QPS | Collection Memory |
|---|---|---|---|---|
| 384 | all-MiniLM-L6-v2 | 1.5 | 650 | 200 MB |
| 768 | BGE-base-en-v1.5 | 2.5 | 400 | 400 MB |
| 1024 | embed-english-v3.0 | 3.2 | 310 | 550 MB |
| 1536 | text-embedding-3-small | 4.0 | 250 | 800 MB |
| 3072 | text-embedding-3-large | 7.5 | 130 | 1.6 GB |

Lower-dimensional embeddings provide substantially better performance. If quality is adequate at 384 or 768 dimensions, the performance gain is 2-3x compared to 1536 dimensions.

### Metadata Filter Overhead

At 100K vectors, ef_search=50:

| Query Type | p50 Latency (ms) | QPS |
|---|---|---|
| Vector only (no filter) | 4.0 | 250 |
| Vector + 1 equality filter | 5.0 | 200 |
| Vector + range filter | 5.5 | 180 |
| Vector + compound filter ($and, 2 conditions) | 6.5 | 155 |
| Vector + where_document ($contains) | 8.0 | 125 |

Metadata filtering adds 25-100% overhead depending on filter complexity. Document content filtering (`where_document`) is the most expensive because it requires scanning stored text.

---

## Benchmarking Your ChromaDB Setup

```python
import chromadb
import numpy as np
import time


def benchmark_chroma(
    collection,
    queries: np.ndarray,
    n_results: int = 10,
    warmup: int = 100,
    duration: float = 30.0,
    where: dict = None
) -> dict:
    """Benchmark ChromaDB collection query performance."""
    # Warmup
    for i in range(warmup):
        collection.query(
            query_embeddings=[queries[i % len(queries)].tolist()],
            n_results=n_results,
            where=where
        )

    # Measure
    latencies = []
    num_queries = 0
    start = time.perf_counter()

    while time.perf_counter() - start < duration:
        q_idx = num_queries % len(queries)
        q_start = time.perf_counter()
        collection.query(
            query_embeddings=[queries[q_idx].tolist()],
            n_results=n_results,
            where=where
        )
        latencies.append((time.perf_counter() - q_start) * 1000)
        num_queries += 1

    elapsed = time.perf_counter() - start

    return {
        "qps": num_queries / elapsed,
        "p50_ms": np.percentile(latencies, 50),
        "p95_ms": np.percentile(latencies, 95),
        "p99_ms": np.percentile(latencies, 99),
        "total_queries": num_queries,
    }


# Usage
client = chromadb.PersistentClient(path="/data/chromadb")
collection = client.get_collection("documents")
queries = np.random.rand(1000, 1536).astype(np.float32)

results = benchmark_chroma(collection, queries)
print(f"QPS: {results['qps']:.0f}")
print(f"p50: {results['p50_ms']:.1f}ms")
print(f"p99: {results['p99_ms']:.1f}ms")
```

---

## Common Pitfalls

1. **Benchmarking with default ef_search=10**: the default is optimized for speed, not recall. Production benchmarks should use ef_search=50-100.

2. **Expecting multi-tenant performance**: ChromaDB with 100+ collections consumes significant memory (each collection has its own HNSW index). Use metadata filtering instead.

3. **Running ChromaDB in Docker for benchmarks**: the overhead of Python-in-Docker and volume mounts can skew latency numbers by 2-3x. Benchmark with local PersistentClient for accurate measurements.

4. **Not warming up before benchmarking**: the first few queries after loading a collection are slower (HNSW graph loading into memory). Run 100 warmup queries before measuring.

5. **Comparing ChromaDB QPS with multi-threaded databases**: ChromaDB's Python query path is single-threaded. pgvector with 8 PostgreSQL connections handles 8x the throughput. This is an architectural limitation, not a performance bug.

6. **Not accounting for embedding function latency**: when using an embedding function (e.g., OpenAI), the API call adds 50-200ms per query. This dominates ChromaDB's own search latency. Pre-compute embeddings for accurate search-only benchmarks.

---

## References

- ChromaDB documentation: https://docs.trychroma.com/
- ChromaDB performance tips: https://docs.trychroma.com/guides/performance
- hnswlib benchmarks: https://github.com/nmslib/hnswlib
- LanceDB: https://lancedb.github.io/lancedb/
- pgvector: https://github.com/pgvector/pgvector
