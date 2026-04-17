# Hybrid Search in PostgreSQL: pgvector + pg_trgm + tsvector

## Overview

PostgreSQL can serve as a complete hybrid search engine by combining three extensions: pgvector for semantic vector search, tsvector (built-in) for full-text search with ranking, and pg_trgm for fuzzy trigram similarity matching. This eliminates the need for separate search infrastructure (Elasticsearch + vector DB) for many use cases. This guide covers practical implementation of hybrid search using RRF fusion, custom scoring functions, and materialized views for performance.

---

## Architecture

```
Query: "database connection pooling best practices"
    |
    +-- vector embedding --> pgvector kNN (semantic similarity)
    |
    +-- text query --> tsvector full-text search (BM25-ish ranking)
    |
    +-- trigrams --> pg_trgm similarity (fuzzy matching)
    |
    v
Fusion (RRF or weighted combination in SQL)
    |
    v
Top-K results
```

### When to Use Each Component

| Component | Matches | Example |
|---|---|---|
| pgvector | Semantic similarity, paraphrases | "car repair" matches "automobile maintenance" |
| tsvector | Keyword matching with stemming and ranking | "connecting" matches "connection" |
| pg_trgm | Fuzzy string matching, typos | "databse" matches "database" |

---

## Setup

### Extensions

```sql
CREATE EXTENSION IF NOT EXISTS vector;       -- pgvector
CREATE EXTENSION IF NOT EXISTS pg_trgm;      -- trigram similarity

-- tsvector is built-in, no extension needed
```

### Table Schema

```sql
CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    title TEXT,
    embedding vector(1536),
    content_tsv tsvector GENERATED ALWAYS AS (
        setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
        setweight(to_tsvector('english', content), 'B')
    ) STORED,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    metadata JSONB DEFAULT '{}'
);

-- Indexes
CREATE INDEX idx_docs_hnsw ON documents
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 128);

CREATE INDEX idx_docs_tsv ON documents USING gin(content_tsv);

CREATE INDEX idx_docs_trgm ON documents USING gin(content gin_trgm_ops);
CREATE INDEX idx_docs_title_trgm ON documents USING gin(title gin_trgm_ops);
```

### Weight Configuration

The tsvector column uses **weighted sections**:
- Weight A (title): highest priority -- title matches are most relevant
- Weight B (body): standard priority

Weights affect `ts_rank` and `ts_rank_cd` scoring:

```sql
-- Default weights: {D=0.1, C=0.2, B=0.4, A=1.0}
-- Custom weights: boost title matches even more
SELECT ts_rank(content_tsv, query, 32) -- 32 = normalize by rank / (rank + 1)
FROM documents, plainto_tsquery('english', 'database connection') query
WHERE content_tsv @@ query;
```

---

## Full-Text Search (tsvector) Deep Dive

### Ranking Functions

PostgreSQL provides two built-in ranking functions:

```sql
-- ts_rank: standard frequency-based ranking
-- Considers: term frequency, inverse document frequency (approximate), weight sections
SELECT id, title,
       ts_rank(content_tsv, query) AS rank
FROM documents, plainto_tsquery('english', 'database connection') query
WHERE content_tsv @@ query
ORDER BY rank DESC
LIMIT 20;

-- ts_rank_cd: cover density ranking
-- Also considers: proximity of matching terms (closer = higher score)
-- Generally better for natural language queries
SELECT id, title,
       ts_rank_cd(content_tsv, query, 32) AS rank
FROM documents, plainto_tsquery('english', 'database connection') query
WHERE content_tsv @@ query
ORDER BY rank DESC
LIMIT 20;
```

### BM25-ish Scoring in PostgreSQL

PostgreSQL's `ts_rank` is not exactly BM25, but you can approximate BM25 with a custom function:

```sql
CREATE OR REPLACE FUNCTION bm25_score(
    doc_tsv tsvector,
    query tsquery,
    k1 float DEFAULT 1.2,
    b float DEFAULT 0.75,
    avgdl float DEFAULT 500.0
)
RETURNS float AS $$
DECLARE
    score float := 0;
    term text;
    tf int;
    df bigint;
    n bigint;
    idf float;
    dl int;
    tf_norm float;
BEGIN
    -- Document length (number of lexemes)
    dl := length(doc_tsv);

    -- Total documents (approximate -- cache this for production)
    SELECT reltuples::bigint INTO n FROM pg_class WHERE relname = 'documents';
    IF n = 0 THEN n := 1; END IF;

    -- For each term in the query
    FOR term IN SELECT unnest(tsvector_to_array(querytree(query)::tsvector))
    LOOP
        -- Term frequency in this document
        SELECT COALESCE(
            (SELECT (word_entry).npos
             FROM unnest(doc_tsv) AS word_entry
             WHERE (word_entry).lexeme = term),
            0
        ) INTO tf;

        IF tf = 0 THEN CONTINUE; END IF;

        -- Document frequency
        SELECT count(*) INTO df FROM documents
        WHERE content_tsv @@ to_tsquery('english', term);
        IF df = 0 THEN df := 1; END IF;

        -- IDF
        idf := ln(1 + (n - df + 0.5) / (df + 0.5));

        -- BM25 TF component
        tf_norm := (tf * (k1 + 1)) / (tf + k1 * (1 - b + b * dl / avgdl));

        score := score + idf * tf_norm;
    END LOOP;

    RETURN score;
END;
$$ LANGUAGE plpgsql STABLE;
```

**Warning**: this custom function is for educational purposes. For production, use `ts_rank_cd` with normalization -- it is much faster because it does not require per-term subqueries.

---

## pg_trgm Fuzzy Matching

### How Trigrams Work

pg_trgm splits text into 3-character sequences:

```sql
SELECT show_trgm('database');
-- {"  d"," da","ase","ata","bas","dat","se ","tab"}

-- Similarity: proportion of shared trigrams
SELECT similarity('database', 'databse');  -- 0.625 (typo still matches)
SELECT similarity('database', 'databank'); -- 0.4

-- Word similarity (matches substring)
SELECT word_similarity('base', 'database'); -- 1.0 (base is a substring)
```

### Using pg_trgm in Search

```sql
-- Fuzzy search with similarity threshold
SELECT id, content,
       similarity(content, 'databse conection') AS sim
FROM documents
WHERE content % 'databse conection'  -- % operator uses similarity threshold
ORDER BY sim DESC
LIMIT 10;

-- Adjust threshold (default: 0.3)
SET pg_trgm.similarity_threshold = 0.2;  -- more permissive

-- Distance operator (for ORDER BY)
SELECT id, content,
       content <-> 'database connection' AS distance
FROM documents
ORDER BY content <-> 'database connection'
LIMIT 10;
```

---

## Hybrid Search Function

### RRF-Based Hybrid Search

```sql
CREATE OR REPLACE FUNCTION hybrid_search(
    query_text text,
    query_embedding vector,
    match_count int DEFAULT 10,
    rrf_k int DEFAULT 60,
    vector_weight float DEFAULT 1.0,
    fulltext_weight float DEFAULT 1.0,
    trgm_weight float DEFAULT 0.5
)
RETURNS TABLE (
    id bigint,
    content text,
    title text,
    rrf_score float,
    vector_rank int,
    fulltext_rank int,
    trgm_rank int
) AS $$
DECLARE
    candidate_count int := match_count * 5;
BEGIN
    RETURN QUERY
    WITH vector_results AS (
        SELECT d.id, d.content, d.title,
               ROW_NUMBER() OVER (
                   ORDER BY d.embedding <=> query_embedding
               )::int AS rank
        FROM documents d
        ORDER BY d.embedding <=> query_embedding
        LIMIT candidate_count
    ),
    fulltext_results AS (
        SELECT d.id, d.content, d.title,
               ROW_NUMBER() OVER (
                   ORDER BY ts_rank_cd(d.content_tsv, plainto_tsquery('english', query_text), 32) DESC
               )::int AS rank
        FROM documents d
        WHERE d.content_tsv @@ plainto_tsquery('english', query_text)
        ORDER BY ts_rank_cd(d.content_tsv, plainto_tsquery('english', query_text), 32) DESC
        LIMIT candidate_count
    ),
    trgm_results AS (
        SELECT d.id, d.content, d.title,
               ROW_NUMBER() OVER (
                   ORDER BY d.content <-> query_text
               )::int AS rank
        FROM documents d
        WHERE d.content % query_text  -- similarity threshold filter
        ORDER BY d.content <-> query_text
        LIMIT candidate_count
    ),
    all_ids AS (
        SELECT id FROM vector_results
        UNION
        SELECT id FROM fulltext_results
        UNION
        SELECT id FROM trgm_results
    ),
    fused AS (
        SELECT
            a.id,
            COALESCE(vector_weight / (rrf_k + v.rank), 0.0) +
            COALESCE(fulltext_weight / (rrf_k + f.rank), 0.0) +
            COALESCE(trgm_weight / (rrf_k + t.rank), 0.0) AS score,
            COALESCE(v.rank, 0) AS v_rank,
            COALESCE(f.rank, 0) AS f_rank,
            COALESCE(t.rank, 0) AS t_rank
        FROM all_ids a
        LEFT JOIN vector_results v ON a.id = v.id
        LEFT JOIN fulltext_results f ON a.id = f.id
        LEFT JOIN trgm_results t ON a.id = t.id
    )
    SELECT f.id, d.content, d.title,
           f.score AS rrf_score,
           f.v_rank AS vector_rank,
           f.f_rank AS fulltext_rank,
           f.t_rank AS trgm_rank
    FROM fused f
    JOIN documents d ON d.id = f.id
    ORDER BY f.score DESC
    LIMIT match_count;
END;
$$ LANGUAGE plpgsql STABLE;
```

### Usage

```sql
-- Search with all three signals
SELECT * FROM hybrid_search(
    'database connection pooling',
    '[0.1, 0.2, ...]'::vector,
    10,             -- match_count
    60,             -- rrf_k
    1.0,            -- vector_weight
    1.0,            -- fulltext_weight
    0.5             -- trgm_weight (lower because it is less precise)
);
```

### Python Wrapper

```python
import psycopg
from pgvector.psycopg import register_vector
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("sentence-transformers/all-MiniLM-L6-v2")

def hybrid_search(conn, query: str, k: int = 10):
    """Execute hybrid search combining vector, full-text, and trigram."""
    query_embedding = model.encode(query).tolist()

    results = conn.execute(
        "SELECT * FROM hybrid_search(%s, %s::vector, %s)",
        (query, query_embedding, k),
    ).fetchall()

    return [
        {
            "id": r[0],
            "content": r[1],
            "title": r[2],
            "rrf_score": r[3],
            "vector_rank": r[4],
            "fulltext_rank": r[5],
            "trgm_rank": r[6],
        }
        for r in results
    ]

# Usage
conn = psycopg.connect("postgresql://user:pass@localhost/vectordb")
register_vector(conn)

results = hybrid_search(conn, "how to optimize database connections")
for r in results:
    print(f"Score: {r['rrf_score']:.4f} | V:{r['vector_rank']} F:{r['fulltext_rank']} T:{r['trgm_rank']}")
    print(f"  {r['title']}: {r['content'][:80]}")
```

---

## Materialized Views for Performance

For large tables, precompute full-text search scores:

```sql
-- Materialized view with precomputed rankings
CREATE MATERIALIZED VIEW documents_search_view AS
SELECT
    d.id,
    d.content,
    d.title,
    d.embedding,
    d.content_tsv,
    length(d.content_tsv) AS doc_length,
    d.metadata
FROM documents d
WHERE d.content IS NOT NULL AND d.embedding IS NOT NULL;

CREATE INDEX ON documents_search_view USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 128);
CREATE INDEX ON documents_search_view USING gin(content_tsv);
CREATE INDEX ON documents_search_view USING gin(content gin_trgm_ops);

-- Refresh periodically
REFRESH MATERIALIZED VIEW CONCURRENTLY documents_search_view;
```

### Incremental Refresh Pattern

For near-real-time updates without full materialization:

```sql
-- Track last refresh
CREATE TABLE search_view_metadata (
    last_refreshed TIMESTAMPTZ DEFAULT '1970-01-01'
);

-- Function to check if refresh is needed
CREATE OR REPLACE FUNCTION should_refresh_search_view()
RETURNS boolean AS $$
    SELECT EXISTS (
        SELECT 1 FROM documents
        WHERE created_at > (SELECT last_refreshed FROM search_view_metadata LIMIT 1)
        OR updated_at > (SELECT last_refreshed FROM search_view_metadata LIMIT 1)
        LIMIT 1
    );
$$ LANGUAGE sql;
```

---

## Performance Considerations

### Query Plan Analysis

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM hybrid_search(
    'database connection',
    '[0.1, 0.2, ...]'::vector,
    10
);
```

### Expected Performance

| Corpus Size | Hybrid Query (p50) | Hybrid Query (p99) | Notes |
|---|---|---|---|
| 10K docs | 5ms | 15ms | All indexes fit in memory |
| 100K docs | 15ms | 40ms | Most indexes in memory |
| 1M docs | 30ms | 80ms | May need larger shared_buffers |
| 10M docs | 60ms | 200ms | Consider partitioning |

### Optimization Checklist

```sql
-- 1. Ensure indexes exist and are used
EXPLAIN ANALYZE SELECT id FROM documents ORDER BY embedding <=> '[...]'::vector LIMIT 10;

-- 2. Check shared_buffers hit ratio
SELECT
    sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit + heap_blks_read), 0) AS cache_hit_ratio
FROM pg_statio_user_tables
WHERE relname = 'documents';
-- Should be > 0.95

-- 3. Check index hit ratio
SELECT
    sum(idx_blks_hit) / NULLIF(sum(idx_blks_hit + idx_blks_read), 0) AS idx_cache_hit_ratio
FROM pg_statio_user_indexes
WHERE relname = 'documents';
-- Should be > 0.99

-- 4. Set ef_search appropriately
SET hnsw.ef_search = 100;  -- balance recall vs latency

-- 5. Adjust trgm threshold for your data
SET pg_trgm.similarity_threshold = 0.3;
```

---

## Two-Signal vs Three-Signal Hybrid

In practice, most teams use two signals (vector + full-text) rather than three:

| Configuration | When to Use |
|---|---|
| Vector + tsvector | Standard hybrid search -- best for most cases |
| Vector + pg_trgm | When typo tolerance is critical (search bars, user-facing) |
| Vector + tsvector + pg_trgm | Maximum coverage but more complex tuning |
| tsvector + pg_trgm (no vector) | When you have no embedding model or semantic search is not needed |

### Simplified Two-Signal Version

```sql
CREATE OR REPLACE FUNCTION hybrid_search_v2(
    query_text text,
    query_embedding vector,
    match_count int DEFAULT 10,
    rrf_k int DEFAULT 60,
    alpha float DEFAULT 0.5  -- 0=pure BM25, 1=pure vector
)
RETURNS TABLE (id bigint, content text, score float) AS $$
BEGIN
    RETURN QUERY
    WITH vector_cte AS (
        SELECT d.id, d.content,
               ROW_NUMBER() OVER (ORDER BY d.embedding <=> query_embedding)::int AS rank
        FROM documents d
        ORDER BY d.embedding <=> query_embedding
        LIMIT match_count * 5
    ),
    text_cte AS (
        SELECT d.id, d.content,
               ROW_NUMBER() OVER (
                   ORDER BY ts_rank_cd(d.content_tsv, plainto_tsquery('english', query_text)) DESC
               )::int AS rank
        FROM documents d
        WHERE d.content_tsv @@ plainto_tsquery('english', query_text)
        LIMIT match_count * 5
    ),
    combined AS (
        SELECT COALESCE(v.id, t.id) AS id,
               COALESCE(v.content, t.content) AS content,
               COALESCE(alpha / (rrf_k + v.rank), 0.0) +
               COALESCE((1 - alpha) / (rrf_k + t.rank), 0.0) AS score
        FROM vector_cte v
        FULL OUTER JOIN text_cte t ON v.id = t.id
    )
    SELECT c.id, c.content, c.score
    FROM combined c
    ORDER BY c.score DESC
    LIMIT match_count;
END;
$$ LANGUAGE plpgsql STABLE;
```

---

## Common Pitfalls

1. **Not creating GIN index for tsvector**: without the GIN index, full-text search does a sequential scan. This is the most common performance mistake.

2. **Using `to_tsvector` without specifying language**: `to_tsvector(content)` uses the default configuration (often `simple`, which does no stemming). Always specify: `to_tsvector('english', content)`.

3. **Forgetting to set `hnsw.ef_search`**: the default (40) may give poor recall. Set it before running hybrid queries.

4. **Not normalizing weights in RRF**: if vector_weight is 1.0 and fulltext_weight is 5.0, full-text results dominate regardless of relevance. Keep weights in a similar range.

5. **Over-retrieving candidates**: setting candidate_count too high (e.g., 10,000) makes the FULL OUTER JOIN expensive. 5-10x the final match_count is usually sufficient.

6. **Not handling empty query results**: if the full-text query matches nothing (no documents contain the query terms), the tsvector CTE returns zero rows. The FULL OUTER JOIN handles this correctly, but your application code should handle the case where all results come from only one signal.

---

## References

- pgvector: https://github.com/pgvector/pgvector
- PostgreSQL Full-Text Search: https://www.postgresql.org/docs/current/textsearch.html
- pg_trgm: https://www.postgresql.org/docs/current/pgtrgm.html
- Supabase hybrid search guide: https://supabase.com/docs/guides/ai/hybrid-search
