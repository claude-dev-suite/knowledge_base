# pgvector Deep Guide

## Overview

pgvector is a PostgreSQL extension that adds vector similarity search to Postgres. It supports multiple data types (vector, halfvec, sparsevec, bit), multiple distance functions (cosine, L2, inner product, L1, Hamming, Jaccard), and two index types (IVFFlat and HNSW). This guide covers advanced usage beyond the basics: data type selection, distance operator details, production configuration, and patterns for combining vector search with PostgreSQL's existing capabilities.

For basic installation, table creation, and simple queries, see the companion document at `vector-databases/pgvector.md`.

---

## Installation

### Extension Setup

```sql
-- Requires pgvector to be installed on the system first
CREATE EXTENSION IF NOT EXISTS vector;

-- Verify version (0.7.0+ recommended for halfvec, sparsevec, bit)
SELECT extversion FROM pg_extension WHERE extname = 'vector';
```

### System Installation

```bash
# Ubuntu/Debian (PostgreSQL 16)
sudo apt install postgresql-16-pgvector

# macOS
brew install pgvector

# From source (latest)
cd /tmp
git clone --branch v0.8.0 https://github.com/pgvector/pgvector.git
cd pgvector
make
sudo make install
```

### Docker (Recommended for Development)

```yaml
# docker-compose.yml
services:
  postgres:
    image: pgvector/pgvector:0.8.0-pg16
    environment:
      POSTGRES_DB: vectordb
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    command: >
      postgres
        -c shared_buffers=512MB
        -c effective_cache_size=1536MB
        -c maintenance_work_mem=256MB
        -c max_parallel_maintenance_workers=4
volumes:
  pgdata:
```

---

## Data Types

### vector (float32)

The primary type. Stores fixed-dimension float32 vectors. Maximum 16,000 dimensions.

```sql
CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    embedding vector(1536),     -- OpenAI text-embedding-3-small
    metadata JSONB DEFAULT '{}'
);

-- Insert
INSERT INTO documents (content, embedding)
VALUES ('Hello world', '[0.1, 0.2, 0.3, ...]'::vector);
```

**Storage**: 4 bytes per dimension + 8 bytes header.
- 768-dim vector: 3,080 bytes
- 1536-dim vector: 6,152 bytes
- 3072-dim vector: 12,296 bytes

### halfvec (float16)

Half-precision floating point. Half the storage of vector with minimal quality loss for most embedding models.

```sql
ALTER TABLE documents ADD COLUMN embedding_half halfvec(1536);

-- Convert from float32 to float16
UPDATE documents SET embedding_half = embedding::halfvec;

-- Index on halfvec
CREATE INDEX ON documents USING hnsw (embedding_half halfvec_cosine_ops);
```

**Storage**: 2 bytes per dimension + 8 bytes header.
- 1536-dim halfvec: 3,080 bytes (50% of float32)

**When to use**: when storage or memory is a constraint and you can accept <0.5% recall loss. Most embedding models output float32 but the precision beyond float16 adds negligible retrieval quality.

### sparsevec

For high-dimensional sparse vectors (mostly zeros). Stores only non-zero values.

```sql
ALTER TABLE documents ADD COLUMN sparse_emb sparsevec(30000);

-- Insert sparse vector: {index:value, index:value, ...}/total_dimensions
INSERT INTO documents (sparse_emb)
VALUES ('{1:0.5, 100:0.3, 999:0.8, 15000:0.2}/30000'::sparsevec);
```

**Storage**: 12 bytes per non-zero element + 8 bytes header.

**Use cases**:
- SPLADE embeddings (learned sparse representations, typically 30K dimensions with ~100 non-zeros)
- TF-IDF or BM25 score vectors
- One-hot encoded features

### bit

Binary vectors for Hamming distance search. Each dimension is a single bit.

```sql
ALTER TABLE documents ADD COLUMN binary_emb bit(1024);

-- Insert binary vector
INSERT INTO documents (binary_emb) VALUES (B'10110001...');

-- Hamming distance search
SELECT id, binary_emb <~> B'10110001...' AS hamming_distance
FROM documents
ORDER BY binary_emb <~> B'10110001...'
LIMIT 10;
```

**Storage**: 1 bit per dimension (128 bytes for 1024-dim).

**Use cases**:
- Binary quantization of embeddings (massive storage reduction)
- Locality-sensitive hashing
- Product quantization codes

---

## Distance Operators

| Operator | Function | Formula | Index Ops Class |
|---|---|---|---|
| `<->` | L2 (Euclidean) distance | sqrt(sum((a_i - b_i)^2)) | `vector_l2_ops` |
| `<=>` | Cosine distance | 1 - (a . b) / (norm(a) * norm(b)) | `vector_cosine_ops` |
| `<#>` | Negative inner product | -(a . b) | `vector_ip_ops` |
| `<+>` | L1 (Manhattan) distance | sum(abs(a_i - b_i)) | `vector_l1_ops` |
| `<~>` | Hamming distance (bit) | popcount(a XOR b) | `bit_hamming_ops` |
| `<%>` | Jaccard distance (bit) | 1 - popcount(a AND b)/popcount(a OR b) | `bit_jaccard_ops` |

### Choosing a Distance Function

```sql
-- Cosine distance (most common for text embeddings)
-- Range: [0, 2], lower = more similar
-- Use when: embeddings are NOT pre-normalized
SELECT id, 1 - (embedding <=> query_vec) AS cosine_similarity
FROM documents
ORDER BY embedding <=> query_vec
LIMIT 10;

-- Inner product (use with pre-normalized embeddings)
-- The operator returns NEGATIVE inner product (for ORDER BY ASC)
-- Use when: embeddings ARE pre-normalized (then cosine = inner product)
SELECT id, (embedding <#> query_vec) * -1 AS inner_product
FROM documents
ORDER BY embedding <#> query_vec
LIMIT 10;

-- L2 distance (for absolute position in embedding space)
-- Range: [0, inf), lower = more similar
-- Use when: magnitude matters (rare for text embeddings)
SELECT id, embedding <-> query_vec AS l2_distance
FROM documents
ORDER BY embedding <-> query_vec
LIMIT 10;
```

**Performance note**: cosine distance internally normalizes vectors at query time. If your embeddings are already L2-normalized (most modern embedding models output normalized vectors), use inner product (`<#>`) instead -- it avoids the normalization step and is ~5-10% faster.

---

## Index Types

### HNSW (Recommended)

Hierarchical Navigable Small World graph. Best recall-latency tradeoff for most use cases.

```sql
CREATE INDEX idx_docs_embedding_hnsw ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Set search-time quality parameter
SET hnsw.ef_search = 100;
```

See `hnsw-tuning.md` for deep tuning guidance.

**Key properties:**
- Can be built on an empty table (unlike IVFFlat)
- Handles incremental inserts well
- Higher memory usage (graph structure)
- Build time: O(N * log(N))

### IVFFlat

Inverted File with flat quantization. Faster to build, lower memory, but requires data before index creation.

```sql
-- MUST have data in the table before creating IVFFlat index
CREATE INDEX idx_docs_embedding_ivf ON documents
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- Set search-time quality parameter
SET ivfflat.probes = 10;
```

**Choosing the number of lists:**
- Up to 1M rows: lists = rows / 1000
- 1M to 10M rows: lists = sqrt(rows)
- Over 10M rows: lists = sqrt(rows) (cap at ~10,000)

**Key properties:**
- Must have data before index creation (k-means clustering)
- Better for append-heavy workloads (no graph rebalancing)
- Lower memory than HNSW
- Needs periodic `REINDEX` after large inserts (cluster centroids become stale)

### Comparison

| Property | HNSW | IVFFlat |
|---|---|---|
| Build on empty table | Yes | No |
| Build time (1M rows) | ~30 min | ~5 min |
| Search quality | Excellent | Good (with tuning) |
| Incremental inserts | Good | Degrades (needs reindex) |
| Memory overhead | Higher | Lower |
| Search latency | Lower | Higher (at same recall) |
| Parallel build | Yes (0.8.0+) | Yes |

---

## Query Patterns

### Basic Similarity Search

```sql
-- Top-10 nearest neighbors by cosine similarity
SELECT id, content, 1 - (embedding <=> $1::vector) AS similarity
FROM documents
ORDER BY embedding <=> $1::vector
LIMIT 10;
```

### Filtered Search

```sql
-- Filter by metadata before vector search
-- IMPORTANT: for the index to be used, the filter must be selective enough
SELECT id, content, 1 - (embedding <=> $1::vector) AS similarity
FROM documents
WHERE metadata->>'category' = 'engineering'
  AND created_at > NOW() - INTERVAL '30 days'
ORDER BY embedding <=> $1::vector
LIMIT 10;
```

### Partial Index (Filtered Index)

If you frequently filter by the same condition, create a partial index:

```sql
-- Index only active documents
CREATE INDEX idx_active_docs_hnsw ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64)
WHERE is_active = true;

-- This query will use the partial index
SELECT id, content
FROM documents
WHERE is_active = true
ORDER BY embedding <=> $1::vector
LIMIT 10;
```

### Distance Threshold

```sql
-- Only return results within a similarity threshold
SELECT id, content, 1 - (embedding <=> $1::vector) AS similarity
FROM documents
WHERE (embedding <=> $1::vector) < 0.3  -- cosine distance < 0.3 means similarity > 0.7
ORDER BY embedding <=> $1::vector
LIMIT 10;
```

### Batch Queries

```python
import psycopg2
import numpy as np

def batch_similarity_search(conn, query_embeddings: list[list[float]], k: int = 10):
    """Search for multiple query vectors in a single round-trip."""
    results = []
    with conn.cursor() as cur:
        for emb in query_embeddings:
            cur.execute(
                """
                SELECT id, content, 1 - (embedding <=> %s::vector) AS similarity
                FROM documents
                ORDER BY embedding <=> %s::vector
                LIMIT %s
                """,
                (str(emb), str(emb), k)
            )
            results.append(cur.fetchall())
    return results
```

---

## Performance Configuration

### PostgreSQL Settings for Vector Workloads

```sql
-- Memory settings (adjust based on available RAM)
SET shared_buffers = '4GB';           -- 25% of total RAM
SET effective_cache_size = '12GB';    -- 75% of total RAM
SET work_mem = '256MB';               -- per-operation memory (important for sorting)
SET maintenance_work_mem = '2GB';     -- for index builds

-- Parallel workers
SET max_parallel_maintenance_workers = 4;   -- for parallel index builds
SET max_parallel_workers_per_gather = 4;    -- for parallel queries

-- WAL settings (if inserting large batches)
SET max_wal_size = '4GB';
SET checkpoint_completion_target = 0.9;
```

### Verifying Index Usage

```sql
-- Always check that the index is being used
EXPLAIN ANALYZE
SELECT id FROM documents
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 10;

-- Expected output should show: Index Scan using idx_docs_embedding_hnsw
-- If you see Seq Scan, the planner decided the index is not beneficial
-- (common for very small tables or unselective filters)
```

### Monitoring Index Performance

```sql
-- Index size
SELECT pg_size_pretty(pg_relation_size('idx_docs_embedding_hnsw')) AS index_size;

-- Table size vs index size
SELECT
    pg_size_pretty(pg_relation_size('documents')) AS table_size,
    pg_size_pretty(pg_relation_size('idx_docs_embedding_hnsw')) AS index_size,
    pg_size_pretty(pg_total_relation_size('documents')) AS total_size;
```

---

## ORM Integration

### SQLAlchemy

```python
from sqlalchemy import create_engine, Column, BigInteger, Text
from sqlalchemy.orm import declarative_base, Session
from pgvector.sqlalchemy import Vector, HalfVector

Base = declarative_base()

class Document(Base):
    __tablename__ = "documents"
    id = Column(BigInteger, primary_key=True, autoincrement=True)
    content = Column(Text, nullable=False)
    embedding = Column(Vector(1536))

engine = create_engine("postgresql+psycopg://user:pass@localhost/vectordb")

# Similarity search
with Session(engine) as session:
    query_vec = [0.1, 0.2, ...]  # 1536-dim

    results = (
        session.query(
            Document,
            (1 - Document.embedding.cosine_distance(query_vec)).label("similarity"),
        )
        .order_by(Document.embedding.cosine_distance(query_vec))
        .limit(10)
        .all()
    )

    for doc, similarity in results:
        print(f"{doc.id}: {similarity:.4f} - {doc.content[:50]}")
```

### psycopg3 (Native)

```python
import psycopg
from pgvector.psycopg import register_vector

conn = psycopg.connect("postgresql://user:pass@localhost/vectordb")
register_vector(conn)

# Insert
embedding = [0.1, 0.2, 0.3]  # Your embedding
conn.execute(
    "INSERT INTO documents (content, embedding) VALUES (%s, %s)",
    ("Hello world", embedding),
)
conn.commit()

# Search
results = conn.execute(
    "SELECT id, content, 1 - (embedding <=> %s::vector) AS similarity "
    "FROM documents ORDER BY embedding <=> %s::vector LIMIT 10",
    (embedding, embedding),
).fetchall()
```

---

## Common Pitfalls

1. **Not setting `ef_search` for HNSW**: the default (40) prioritizes speed over recall. For RAG applications, set it to 100-200 for better recall.

2. **Creating IVFFlat on empty table**: IVFFlat requires data to compute cluster centroids. Use HNSW if you need an index before data is loaded.

3. **Using cosine distance with pre-normalized vectors**: if your embedding model already L2-normalizes output, use inner product (`<#>`) instead of cosine (`<=>`) for a ~5-10% speed improvement.

4. **Not tuning `maintenance_work_mem` for builds**: HNSW index builds are memory-intensive. The default 64MB is far too low for large tables. Set to at least 1GB.

5. **Storing embeddings as JSONB arrays**: use the `vector` type. JSONB arrays cannot be indexed for similarity search.

6. **Forgetting to VACUUM**: after large DELETE/UPDATE operations, dead tuples slow down both sequential and index scans. Schedule regular VACUUM ANALYZE.

7. **Excessive dimensions**: 3072-dim vectors use 2x the storage and are ~2x slower to search than 1536-dim. If your model supports Matryoshka dimensions, use a lower dimension (512-1024) with minimal quality loss.

---

## References

- pgvector repository: https://github.com/pgvector/pgvector
- pgvector 0.8.0 release notes: https://github.com/pgvector/pgvector/blob/master/CHANGELOG.md
- pgvector Python library: https://github.com/pgvector/pgvector-python
- PostgreSQL performance tuning: https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server
- Supabase pgvector guide: https://supabase.com/docs/guides/ai/vector-columns
