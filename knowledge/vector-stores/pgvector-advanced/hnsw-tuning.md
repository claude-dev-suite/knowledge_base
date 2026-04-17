# HNSW Index Deep Tuning for pgvector

## Overview

The HNSW (Hierarchical Navigable Small World) index in pgvector has three key parameters: `m` (connections per layer), `ef_construction` (build-time quality), and `ef_search` (query-time quality). Proper tuning of these parameters can mean the difference between 85% and 99.5% recall, and between 5ms and 50ms query latency. This guide provides concrete tuning recipes based on corpus size, quality requirements, and hardware constraints.

---

## Parameter Reference

### m (Connections per Layer)

- **Default**: 16
- **Range**: 2-100 (practical: 4-64)
- **Set at**: index creation time (cannot be changed without rebuild)

`m` controls how many bidirectional links each node maintains in the HNSW graph. Higher m means:
- More connections = more paths to find the nearest neighbor = higher recall
- More memory per node = larger index
- Slower build time (more connections to maintain)
- Diminishing returns above m=48

```sql
-- Index memory per vector: roughly 4 * m * sizeof(int) = 4 * m * 4 bytes
-- m=16: ~256 bytes overhead per vector
-- m=32: ~512 bytes overhead per vector
-- m=64: ~1024 bytes overhead per vector
```

**How m affects recall** (1M vectors, 1536 dims, ef_search=100):

| m | Recall@10 | Index Size | Build Time |
|---|---|---|---|
| 4 | 0.92 | baseline | 1x |
| 8 | 0.96 | 1.3x | 1.5x |
| 16 (default) | 0.98 | 1.8x | 2.2x |
| 32 | 0.993 | 2.8x | 3.5x |
| 48 | 0.996 | 3.8x | 5x |
| 64 | 0.997 | 4.8x | 7x |

**Guideline**: m=16 is good for most cases. Increase to m=32 when you need >99% recall and have the memory budget. m=64 is rarely needed.

### ef_construction (Build-Time Quality)

- **Default**: 64
- **Range**: 4-1000 (practical: 32-512)
- **Set at**: index creation time (cannot be changed without rebuild)

`ef_construction` controls the size of the dynamic candidate list during index construction. Higher values explore more candidates when inserting each vector, building a better graph but taking longer.

```sql
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 200);
```

**How ef_construction affects quality** (1M vectors, m=16, ef_search=100):

| ef_construction | Recall@10 | Build Time |
|---|---|---|
| 32 | 0.965 | 0.7x |
| 64 (default) | 0.980 | 1x |
| 128 | 0.990 | 1.6x |
| 200 | 0.993 | 2.2x |
| 256 | 0.994 | 2.7x |
| 512 | 0.995 | 4.5x |

**Guideline**: ef_construction >= 2 * m is the minimum. For production, use ef_construction = 100-200. Values above 256 rarely help.

**Important**: ef_construction sets the ceiling for index quality. If you build with ef_construction=64 but search with ef_search=500, the recall will plateau at whatever the index quality allows. You cannot fix a bad index with high ef_search.

### ef_search (Query-Time Quality)

- **Default**: 40
- **Range**: 1-1000 (practical: 10-500)
- **Set at**: query time (can be changed per session/query)
- **Must be >= k** (the number of results requested)

`ef_search` controls the size of the dynamic candidate list during search. Higher values mean the search explores more of the graph, finding better results but taking longer.

```sql
-- Set per session
SET hnsw.ef_search = 200;

-- Or per transaction
BEGIN;
SET LOCAL hnsw.ef_search = 200;
SELECT ... ORDER BY embedding <=> query LIMIT 10;
COMMIT;
```

**How ef_search affects latency and recall** (1M vectors, m=16, ef_construction=128):

| ef_search | Recall@10 | Latency (p50) | Latency (p99) |
|---|---|---|---|
| 10 | 0.88 | 1.5ms | 3ms |
| 20 | 0.93 | 2ms | 5ms |
| 40 (default) | 0.97 | 3ms | 8ms |
| 100 | 0.991 | 6ms | 15ms |
| 200 | 0.996 | 10ms | 25ms |
| 400 | 0.998 | 18ms | 40ms |
| 1000 | 0.999 | 40ms | 90ms |

**Guideline**: start with ef_search = 100 for production workloads. Increase if recall is insufficient; decrease if latency is too high.

---

## Parallel Index Build

pgvector 0.6.0+ supports parallel HNSW index builds. This can reduce build time by 2-4x:

```sql
-- Set parallel workers BEFORE creating the index
SET maintenance_work_mem = '4GB';
SET max_parallel_maintenance_workers = 7;  -- use N-1 cores

-- Now create the index (will use parallel workers)
CREATE INDEX CONCURRENTLY idx_docs_hnsw ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 128);
```

**Parallel build benchmarks** (1M rows, 1536 dims, m=16, ef_construction=128):

| Workers | Build Time | Speedup |
|---|---|---|
| 1 (serial) | ~30 min | 1x |
| 2 | ~18 min | 1.7x |
| 4 | ~10 min | 3x |
| 8 | ~7 min | 4.3x |

Speedup is sublinear because of synchronization overhead and memory bandwidth limits.

### Memory Requirements

```
Total memory for build = maintenance_work_mem * (max_parallel_maintenance_workers + 1)
```

With `maintenance_work_mem = 4GB` and 7 parallel workers: 32 GB total. Ensure your machine has this much available RAM.

**Critical**: if `maintenance_work_mem` is too low, the build falls back to on-disk sorting, which is 10x+ slower. For 1M 1536-dim vectors:

| maintenance_work_mem | Build Behavior |
|---|---|
| 64MB (default) | Extremely slow, disk-heavy |
| 256MB | Mostly in-memory, some spills |
| 1GB | Fully in-memory for most cases |
| 4GB | Comfortable margin |

---

## Build Time vs Recall Curve

### Practical Tuning Recipe

**Step 1: Determine your recall target.**
- RAG applications: 0.95-0.99 (missing 1-5% of relevant docs is acceptable)
- Recommendation systems: 0.90-0.95 (speed matters more)
- Duplicate detection: 0.99+ (must find near-duplicates)

**Step 2: Choose m based on recall target.**
```
Recall > 0.95: m = 16 (default)
Recall > 0.99: m = 32
Recall > 0.995: m = 48-64
```

**Step 3: Choose ef_construction.**
```
General rule: ef_construction = max(2 * m, 128)
High quality: ef_construction = max(4 * m, 256)
```

**Step 4: Build the index with maximum available parallelism.**

**Step 5: Tune ef_search for your latency budget.**
```python
# Tuning script: find optimal ef_search for target recall

import psycopg
import numpy as np
from time import perf_counter

def tune_ef_search(conn, test_queries, ground_truth, target_recall=0.95, k=10):
    """
    Binary search for the minimum ef_search that achieves target recall.

    test_queries: list of query vectors
    ground_truth: list of sets of true nearest neighbor IDs (from exact search)
    """
    def measure_recall_and_latency(ef_search):
        conn.execute(f"SET hnsw.ef_search = {ef_search}")

        total_recall = 0.0
        latencies = []

        for query_vec, true_ids in zip(test_queries, ground_truth):
            start = perf_counter()
            results = conn.execute(
                "SELECT id FROM documents ORDER BY embedding <=> %s::vector LIMIT %s",
                (query_vec.tolist(), k),
            ).fetchall()
            latencies.append(perf_counter() - start)

            retrieved_ids = {r[0] for r in results}
            total_recall += len(retrieved_ids & true_ids) / len(true_ids)

        avg_recall = total_recall / len(test_queries)
        p50_latency = np.percentile(latencies, 50) * 1000  # ms
        p99_latency = np.percentile(latencies, 99) * 1000  # ms

        return avg_recall, p50_latency, p99_latency

    # Test a range of ef_search values
    results = []
    for ef in [10, 20, 40, 60, 80, 100, 150, 200, 300, 400, 500]:
        recall, p50, p99 = measure_recall_and_latency(ef)
        results.append({"ef_search": ef, "recall": recall, "p50_ms": p50, "p99_ms": p99})
        print(f"ef_search={ef:4d} | recall={recall:.4f} | p50={p50:.1f}ms | p99={p99:.1f}ms")

        if recall >= target_recall:
            # Found minimum ef_search that meets target
            # Try a few lower values to find the exact minimum
            break

    return results
```

**Step 6: Verify with EXPLAIN ANALYZE.**

```sql
SET hnsw.ef_search = 100;

EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM documents
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 10;

-- Look for:
-- "Index Scan using idx_docs_hnsw" (not Seq Scan)
-- Actual loops: 1
-- Buffers: shared hit=XXX (higher = more graph traversal)
```

---

## Configuration Recipes by Scale

### Small (< 100K vectors)

```sql
-- Index
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 100);

-- Search
SET hnsw.ef_search = 100;

-- PostgreSQL config
SET maintenance_work_mem = '256MB';
SET work_mem = '64MB';

-- Expected: build < 2 min, search < 3ms, recall > 0.97
```

### Medium (100K - 1M vectors)

```sql
-- Index
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 24, ef_construction = 200);

-- Search
SET hnsw.ef_search = 150;

-- PostgreSQL config
SET maintenance_work_mem = '1GB';
SET max_parallel_maintenance_workers = 4;
SET work_mem = '128MB';

-- Expected: build 10-30 min, search 5-10ms, recall > 0.99
```

### Large (1M - 10M vectors)

```sql
-- Index (consider halfvec to halve storage)
CREATE INDEX ON documents USING hnsw (embedding_half halfvec_cosine_ops)
WITH (m = 32, ef_construction = 256);

-- Search
SET hnsw.ef_search = 200;

-- PostgreSQL config
SET maintenance_work_mem = '4GB';
SET max_parallel_maintenance_workers = 7;
SET work_mem = '256MB';
SET effective_cache_size = '24GB';
SET shared_buffers = '8GB';

-- Expected: build 2-8 hours, search 10-25ms, recall > 0.99
```

### Very Large (10M - 100M vectors)

At this scale, consider whether pgvector is the right choice. Purpose-built vector databases (Qdrant, Milvus, Weaviate) may offer better performance. If staying with pgvector:

```sql
-- Use halfvec + partitioning
CREATE TABLE documents_2024 PARTITION OF documents
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Index per partition
CREATE INDEX ON documents_2024 USING hnsw (embedding_half halfvec_cosine_ops)
WITH (m = 32, ef_construction = 200);

-- Search
SET hnsw.ef_search = 300;

-- PostgreSQL config
SET maintenance_work_mem = '8GB';
SET max_parallel_maintenance_workers = 15;
SET work_mem = '512MB';
SET effective_cache_size = '48GB';
SET shared_buffers = '16GB';
```

---

## Index Maintenance

### After Large Inserts

HNSW handles incremental inserts well, but after inserting >20% new data, consider rebuilding:

```sql
-- Rebuild index (blocks writes during rebuild)
REINDEX INDEX idx_docs_hnsw;

-- Non-blocking rebuild (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_docs_hnsw;
```

### After Large Deletes

Deleted vectors leave "holes" in the graph that degrade search quality:

```sql
-- Check for dead tuples
SELECT n_dead_tup, n_live_tup,
       round(n_dead_tup::numeric / GREATEST(n_live_tup, 1) * 100, 2) AS dead_pct
FROM pg_stat_user_tables WHERE relname = 'documents';

-- If dead_pct > 20%, vacuum then reindex
VACUUM ANALYZE documents;
REINDEX INDEX CONCURRENTLY idx_docs_hnsw;
```

### Monitoring Index Health

```sql
-- Index size
SELECT pg_size_pretty(pg_relation_size('idx_docs_hnsw')) AS size;

-- Index stats
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes WHERE indexrelname = 'idx_docs_hnsw';

-- If idx_scan is 0, the index is never used -- check your queries
```

---

## Advanced Patterns

### Multiple Indexes for Different ef_search

You cannot have different ef_search values for different queries in the same session without SET. But you can use connection pooling with different configurations:

```python
# Connection pool with high recall (RAG queries)
high_recall_pool = psycopg.pool.ConnectionPool(
    "postgresql://user:pass@localhost/vectordb",
    min_size=5,
    max_size=20,
    kwargs={"options": "-c hnsw.ef_search=200"},
)

# Connection pool with low latency (autocomplete)
fast_pool = psycopg.pool.ConnectionPool(
    "postgresql://user:pass@localhost/vectordb",
    min_size=5,
    max_size=20,
    kwargs={"options": "-c hnsw.ef_search=40"},
)
```

### Two-Phase Search (Approximate then Exact)

For highest recall without paying the latency cost for all queries:

```python
def two_phase_search(conn, query_vec, k=10, recall_target=0.99):
    """
    Phase 1: fast approximate search to get candidates
    Phase 2: exact distance computation on candidates
    """
    # Phase 1: approximate with moderate ef_search
    conn.execute("SET LOCAL hnsw.ef_search = 100")
    candidates = conn.execute(
        "SELECT id, embedding FROM documents "
        "ORDER BY embedding <=> %s::vector LIMIT %s",
        (query_vec, k * 5),  # over-retrieve
    ).fetchall()

    # Phase 2: exact scoring (no index, just numpy)
    import numpy as np
    query = np.array(query_vec)
    scored = []
    for doc_id, emb in candidates:
        emb_array = np.array(emb)
        cosine_sim = np.dot(query, emb_array) / (
            np.linalg.norm(query) * np.linalg.norm(emb_array)
        )
        scored.append((doc_id, cosine_sim))

    scored.sort(key=lambda x: x[1], reverse=True)
    return scored[:k]
```

---

## Common Pitfalls

1. **Building with default maintenance_work_mem (64MB)**: this causes disk spills during index construction, making it 10-50x slower. Always increase to at least 1GB.

2. **Setting ef_search lower than k**: if you request LIMIT 10 but ef_search=5, the index cannot explore enough candidates. ef_search must be >= k.

3. **Never rebuilding after heavy churn**: if 50% of your data has been deleted and re-inserted, the graph quality degrades significantly. Schedule periodic REINDEX.

4. **Over-tuning m**: m=64 uses 4x the memory of m=16 but only improves recall from 0.98 to 0.997. Unless you need >99.5% recall, m=16-32 is sufficient.

5. **Expecting ef_search to fix a bad index**: if ef_construction was too low, the graph is permanently degraded. Rebuild with higher ef_construction.

6. **Not using CONCURRENTLY for production**: `CREATE INDEX` and `REINDEX` take table-level locks. Always use the `CONCURRENTLY` option in production to avoid blocking writes.

---

## References

- Malkov, Y. and Yashunin, D. "Efficient and Robust Approximate Nearest Neighbor using Hierarchical Navigable Small World Graphs." IEEE TPAMI 2020.
- pgvector HNSW implementation: https://github.com/pgvector/pgvector#hnsw
- pgvector benchmarks: https://github.com/pgvector/pgvector#performance
- ann-benchmarks: https://ann-benchmarks.com/
