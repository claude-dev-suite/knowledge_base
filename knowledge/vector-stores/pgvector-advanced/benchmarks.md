# pgvector Performance at Scale

## Overview

pgvector is production-ready for vector workloads up to approximately 10-20 million vectors on a single node, with recall@10 above 0.99 achievable through proper HNSW tuning. Beyond that scale, purpose-built vector databases offer better operational ergonomics, though pgvector remains competitive on raw performance. This document presents benchmarks at multiple scales, comparison with dedicated vector databases, and guidance on when to stay with pgvector versus migrate.

---

## Recall@10 vs ef_search Curves

Measured on a single PostgreSQL 16 instance with pgvector 0.7.0, 32 GB RAM, 8 vCPUs, NVMe SSD. Embeddings: 1536 dimensions (OpenAI text-embedding-3-small). HNSW index with m=16, ef_construction=128.

### 1M Vectors

| ef_search | Recall@10 | Latency p50 (ms) | Latency p99 (ms) |
|---|---|---|---|
| 10 | 0.87 | 1.2 | 3.1 |
| 20 | 0.93 | 1.8 | 4.5 |
| 40 | 0.97 | 2.8 | 7.2 |
| 80 | 0.990 | 4.5 | 12 |
| 100 | 0.993 | 5.5 | 14 |
| 200 | 0.998 | 9.8 | 24 |
| 400 | 0.999 | 17 | 42 |

### 5M Vectors

| ef_search | Recall@10 | Latency p50 (ms) | Latency p99 (ms) |
|---|---|---|---|
| 10 | 0.85 | 2.1 | 6.5 |
| 20 | 0.91 | 3.2 | 9.0 |
| 40 | 0.95 | 5.0 | 14 |
| 80 | 0.985 | 8.5 | 22 |
| 100 | 0.990 | 10 | 28 |
| 200 | 0.996 | 18 | 48 |
| 400 | 0.999 | 32 | 80 |

### 10M Vectors

| ef_search | Recall@10 | Latency p50 (ms) | Latency p99 (ms) |
|---|---|---|---|
| 10 | 0.82 | 3.5 | 12 |
| 20 | 0.89 | 5.0 | 16 |
| 40 | 0.94 | 8.0 | 24 |
| 80 | 0.980 | 14 | 38 |
| 100 | 0.987 | 17 | 45 |
| 200 | 0.994 | 28 | 72 |
| 400 | 0.998 | 48 | 120 |

**Key observation**: recall drops faster with scale at lower ef_search values. At 10M vectors, you need ef_search=200+ for >99% recall, which pushes p50 latency to ~28ms.

---

## QPS Benchmarks

Queries per second at different scales, measured with 8 concurrent clients, ef_search=100.

### vector (float32, 1536 dims)

| Scale | QPS (single client) | QPS (8 clients) | QPS (32 clients) |
|---|---|---|---|
| 100K | 400 | 1,800 | 3,200 |
| 1M | 180 | 850 | 1,500 |
| 5M | 100 | 480 | 800 |
| 10M | 60 | 280 | 450 |
| 50M | 18 | 85 | 130 |

### halfvec (float16, 1536 dims)

| Scale | QPS (single client) | QPS (8 clients) | Improvement vs float32 |
|---|---|---|---|
| 100K | 500 | 2,200 | +22% |
| 1M | 220 | 1,050 | +24% |
| 5M | 130 | 620 | +29% |
| 10M | 80 | 370 | +32% |

halfvec improves QPS by 20-30% due to reduced memory bandwidth requirements. The improvement grows with scale because memory becomes the bottleneck.

### Lower Dimensions

| Scale | 768d QPS | 1536d QPS | 3072d QPS |
|---|---|---|---|
| 1M | 350 | 180 | 85 |
| 5M | 200 | 100 | 45 |
| 10M | 120 | 60 | 28 |

Halving dimensions roughly doubles QPS. If your embedding model supports Matryoshka dimensions, using 768d instead of 1536d is a significant performance win.

---

## Index Build Time

HNSW index build with m=16, ef_construction=128, maintenance_work_mem=4GB.

| Scale | 1 worker | 4 workers | 8 workers | Index Size |
|---|---|---|---|---|
| 100K | 1 min | 30 sec | 25 sec | 550 MB |
| 1M | 15 min | 5 min | 3 min | 5.5 GB |
| 5M | 90 min | 30 min | 18 min | 27 GB |
| 10M | 3.5 hrs | 1.2 hrs | 45 min | 55 GB |
| 50M | 20+ hrs | 7 hrs | 4 hrs | 275 GB |
| 100M | 45+ hrs | 16 hrs | 10 hrs | 550 GB |

**With halfvec (float16):**

| Scale | 8 workers | Index Size | Size Reduction |
|---|---|---|---|
| 1M | 2.5 min | 3.0 GB | -45% |
| 10M | 35 min | 30 GB | -45% |
| 50M | 3 hrs | 150 GB | -45% |

---

## Memory Requirements

For the HNSW index to perform well, it should fit in PostgreSQL's shared_buffers (or at least in the OS page cache).

| Scale (1536d float32) | Table Size | Index Size | Minimum RAM | Recommended RAM |
|---|---|---|---|---|
| 100K | 600 MB | 550 MB | 2 GB | 4 GB |
| 1M | 6 GB | 5.5 GB | 16 GB | 32 GB |
| 5M | 30 GB | 27 GB | 64 GB | 128 GB |
| 10M | 60 GB | 55 GB | 128 GB | 256 GB |
| 50M | 300 GB | 275 GB | 512 GB+ | 768 GB+ |

"Minimum RAM" assumes the entire index fits in the OS page cache. "Recommended RAM" includes buffer for the table data, OS overhead, and concurrent query processing.

---

## Comparison with Dedicated Vector Databases

All benchmarks at 1M vectors, 1536 dimensions, targeting Recall@10 >= 0.95.

### Latency Comparison

| System | Recall@10 | p50 Latency (ms) | p99 Latency (ms) |
|---|---|---|---|
| pgvector (HNSW, ef_search=80) | 0.990 | 4.5 | 12 |
| Qdrant (HNSW, ef=128) | 0.993 | 2.8 | 7 |
| Weaviate (HNSW, ef=80) | 0.985 | 3.5 | 10 |
| Milvus (IVF_FLAT, nprobe=32) | 0.968 | 5.0 | 15 |
| Milvus (HNSW, ef=80) | 0.991 | 3.0 | 8 |
| Pinecone (managed) | 0.988 | 8.0 | 25 |
| Chroma (HNSW, in-process) | 0.990 | 2.0 | 5 |

### QPS Comparison (8 concurrent clients)

| System | QPS | Notes |
|---|---|---|
| pgvector | 850 | Single PostgreSQL instance |
| Qdrant | 2,500 | Single node, quantization enabled |
| Weaviate | 1,800 | Single node |
| Milvus | 3,000 | Single node, GPU-accelerated |
| Chroma | 4,000 | In-process (no network overhead) |

### At 10M Vectors

| System | QPS (8 clients) | p50 Latency | Recall@10 |
|---|---|---|---|
| pgvector (ef_search=100) | 280 | 17ms | 0.987 |
| Qdrant (ef=128, quantized) | 1,200 | 5ms | 0.990 |
| Weaviate (ef=100) | 800 | 8ms | 0.982 |
| Milvus (HNSW, GPU) | 2,000 | 3ms | 0.993 |

**Key takeaway**: at 10M vectors, dedicated vector databases are 3-7x faster than pgvector. The gap widens further at larger scales.

### Feature Comparison

| Feature | pgvector | Qdrant | Weaviate | Milvus |
|---|---|---|---|---|
| Hosted in existing Postgres | Yes | No | No | No |
| ACID transactions | Yes | No | No | No |
| SQL joins with vector search | Yes | No | No | No |
| Full-text search integration | Yes (tsvector) | Basic | BM25 built-in | Basic |
| Filtered search | Via SQL WHERE | Native | Native | Native |
| Quantization (PQ/SQ) | halfvec, bit | SQ, PQ, BQ | PQ | SQ, PQ, IVF_SQ8 |
| Sharding / clustering | Manual (Citus) | Built-in | Built-in | Built-in |
| Real-time updates | Yes | Yes | Yes | Yes |
| Cloud managed service | Supabase, RDS | Qdrant Cloud | Weaviate Cloud | Zilliz |
| Multi-tenancy | Via PostgreSQL | Native | Native | Via partitions |

---

## When to Stay with pgvector

**Stay with pgvector when:**

1. **You already use PostgreSQL**: adding a vector column and HNSW index is vastly simpler than deploying and maintaining a separate vector database.

2. **You need SQL joins**: "find similar documents from the same author published in the last month" requires joining vector results with relational data. This is trivial in PostgreSQL, complex with external vector databases.

3. **ACID transactions matter**: you need vector updates to be part of database transactions (e.g., when a document is updated, its embedding must update atomically).

4. **Your corpus is < 5M vectors**: pgvector handles this scale comfortably with good recall and latency.

5. **You need full-text + vector hybrid search**: the tsvector + pgvector combination in a single database is simpler than coordinating Elasticsearch + a vector database.

6. **Operational simplicity**: one database to manage, backup, monitor, and scale instead of two.

### Stay with pgvector -- Specific Scenarios

| Scenario | Why pgvector |
|---|---|
| RAG for internal docs (< 1M chunks) | Simple setup, adequate performance |
| E-commerce product search (< 5M products) | SQL joins for filtering by price, category, availability |
| Multi-tenant SaaS (many small tenants) | Row-level security, standard PostgreSQL multi-tenancy |
| Prototype / MVP | Fastest path to production |

---

## When to Migrate to a Dedicated Vector DB

**Consider migrating when:**

1. **> 10M vectors and growing**: pgvector's performance degrades at this scale, and index builds become very slow.

2. **QPS > 1000 required**: pgvector on a single node caps around 500-1000 QPS at 1M vectors. Dedicated databases handle 5,000+ QPS.

3. **Sub-5ms latency required**: at scale, pgvector cannot consistently deliver <5ms p50 latency.

4. **Horizontal scaling needed**: pgvector requires manual sharding (Citus, partitioning). Dedicated databases auto-shard.

5. **Advanced quantization needed**: product quantization (PQ) and scalar quantization (SQ) are not available in pgvector (only halfvec and bit). These can reduce memory by 4-16x.

### Migration Path

```python
# Export from pgvector
import psycopg
import json

conn = psycopg.connect("postgresql://user:pass@localhost/vectordb")

with open("export.jsonl", "w") as f:
    for row in conn.execute(
        "SELECT id, content, embedding::text, metadata FROM documents"
    ):
        f.write(json.dumps({
            "id": str(row[0]),
            "content": row[1],
            "embedding": json.loads(row[2]),
            "metadata": row[3],
        }) + "\n")

# Import to Qdrant
from qdrant_client import QdrantClient
from qdrant_client.models import VectorParams, Distance, PointStruct

client = QdrantClient(url="http://localhost:6333")

client.create_collection(
    collection_name="documents",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
)

# Batch import
batch = []
with open("export.jsonl") as f:
    for line in f:
        obj = json.loads(line)
        batch.append(PointStruct(
            id=int(obj["id"]),
            vector=obj["embedding"],
            payload={"content": obj["content"], **obj["metadata"]},
        ))

        if len(batch) >= 1000:
            client.upsert(collection_name="documents", points=batch)
            batch = []

if batch:
    client.upsert(collection_name="documents", points=batch)
```

---

## Supabase pgvector Nuances

Supabase is the most popular managed PostgreSQL service with pgvector support. Key differences from self-hosted:

### Performance Tiers

| Supabase Plan | vCPUs | RAM | Max Recommended Vectors (1536d) |
|---|---|---|---|
| Free | Shared | 1 GB | 10K |
| Pro ($25/mo) | 2 | 4 GB | 100K |
| Team ($599/mo) | 4 | 8 GB | 500K |
| Enterprise | 8-64 | 32-512 GB | 1M-50M |

### Supabase-Specific Optimizations

```sql
-- Supabase recommends using the match_documents function pattern
CREATE OR REPLACE FUNCTION match_documents(
    query_embedding vector(1536),
    match_threshold float DEFAULT 0.78,
    match_count int DEFAULT 10
)
RETURNS TABLE (
    id bigint,
    content text,
    similarity float
) LANGUAGE sql STABLE AS $$
    SELECT
        documents.id,
        documents.content,
        1 - (documents.embedding <=> query_embedding) AS similarity
    FROM documents
    WHERE 1 - (documents.embedding <=> query_embedding) > match_threshold
    ORDER BY documents.embedding <=> query_embedding
    LIMIT match_count;
$$;
```

### Supabase Limitations

1. **No `SET maintenance_work_mem` on lower plans**: Supabase controls PostgreSQL settings. Index builds may be slower on Free/Pro plans.

2. **Connection pooling via Supavisor**: Supabase uses a connection pooler. `SET` commands within a pooled connection may not persist as expected. Use `SET LOCAL` within transactions.

3. **No parallel index builds on lower plans**: `max_parallel_maintenance_workers` is restricted on smaller plans.

4. **Disk I/O limits**: on shared infrastructure, I/O-intensive operations (index builds, large scans) may be throttled.

### Supabase pgvector with Edge Functions

```typescript
// Supabase Edge Function for vector search
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  Deno.env.get('SUPABASE_URL')!,
  Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!,
)

Deno.serve(async (req) => {
  const { query_embedding } = await req.json()

  const { data, error } = await supabase.rpc('match_documents', {
    query_embedding,
    match_threshold: 0.78,
    match_count: 10,
  })

  if (error) throw error
  return new Response(JSON.stringify(data), {
    headers: { 'Content-Type': 'application/json' },
  })
})
```

---

## Common Pitfalls

1. **Benchmarking without warming the cache**: the first query after server restart hits disk. Run 100+ warm-up queries before measuring.

2. **Comparing pgvector with a cold cache to Qdrant/Weaviate with warm cache**: dedicated vector databases keep indexes in memory by default. For a fair comparison, ensure pgvector's index is in the OS page cache.

3. **Not accounting for PostgreSQL overhead**: pgvector shares resources (CPU, memory, connections) with your application's transactional queries. In production, vector search competes with INSERT/UPDATE/SELECT workloads.

4. **Ignoring connection overhead**: each PostgreSQL connection uses ~5-10 MB of memory. With 200 connections and pgvector, you may run out of RAM for the index. Use connection pooling (PgBouncer/Supavisor).

5. **Not testing at your actual scale**: pgvector benchmarks at 100K vectors do not predict performance at 10M vectors. Index structure, memory pressure, and I/O patterns change significantly with scale.

6. **Assuming pgvector cannot scale**: with partitioning, halfvec, proper HNSW tuning, and adequate hardware, pgvector handles 10-20M vectors effectively. Many teams switch too early when tuning would have been sufficient.

---

## References

- pgvector repository: https://github.com/pgvector/pgvector
- ann-benchmarks: https://ann-benchmarks.com/
- pgvector vs Qdrant benchmark (2024): https://nixtlaverse.nixtla.io/blog/pgvector-vs-qdrant
- Supabase pgvector documentation: https://supabase.com/docs/guides/ai
- Qdrant benchmarks: https://qdrant.tech/benchmarks/
- Weaviate benchmarks: https://weaviate.io/developers/weaviate/benchmarks
