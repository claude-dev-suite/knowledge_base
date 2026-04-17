# Hybrid Search: Combining BM25 with Vector Search

## Overview

Hybrid search runs BM25 (lexical) and vector (semantic) retrieval in parallel, then fuses the results into a single ranked list. This approach consistently outperforms either method alone because the two signals are complementary: BM25 excels at exact term matching (names, codes, IDs) while vector search captures semantic similarity (paraphrases, synonyms, conceptual relatedness). This document covers the core fusion algorithms, implementations in Elasticsearch, Weaviate, and PostgreSQL, and guidelines for when each signal dominates.

---

## Why Hybrid Beats Either Alone

### Failure Modes of Each Method

**BM25 fails when:**
- The query uses different words than the document ("automobile" vs "car")
- The query is conceptual ("how to prevent memory issues in long-running services" -- the document says "garbage collection optimization")
- The query is in a different language than the document

**Vector search fails when:**
- The query contains specific identifiers ("CVE-2024-3094", "error code E_TIMEOUT")
- The query requires exact term matching ("kubectl get pods" should not match "docker ps")
- The embedding model was not trained on your domain vocabulary
- Short, precise queries where BM25 already works perfectly

**Hybrid catches both:**
- BM25 retrieves the exact matches that vector search misses
- Vector search retrieves the semantic matches that BM25 misses
- Fusion ensures both types of relevant documents appear in the final results

### Empirical Evidence

On BEIR benchmark (average across 13 datasets):

| Method | NDCG@10 |
|---|---|
| BM25 | 0.440 |
| E5-large (vector only) | 0.486 |
| ColBERTv2 (late interaction) | 0.497 |
| **BM25 + E5-large (hybrid)** | **0.521** |
| **BM25 + ColBERTv2 (hybrid)** | **0.532** |

Hybrid consistently adds 3-7% over the best single method.

---

## Fusion Algorithms

### Reciprocal Rank Fusion (RRF)

RRF is the most widely used fusion algorithm. It is parameter-free (except for k) and works well in practice:

```
RRF_score(d) = SUM_{r in retrievers} 1 / (k + rank_r(d))
```

Where:
- `rank_r(d)` = rank of document d in retriever r's result list (1-indexed)
- `k` = smoothing constant (default: 60)

```python
def reciprocal_rank_fusion(
    result_lists: list[list[str]],
    k: int = 60,
) -> list[tuple[str, float]]:
    """
    Fuse multiple ranked lists using RRF.

    result_lists: list of ranked lists, each containing document IDs
                  ordered by relevance (most relevant first)
    k: smoothing constant (higher = less weight on top ranks)
    """
    scores = {}

    for result_list in result_lists:
        for rank, doc_id in enumerate(result_list, start=1):
            if doc_id not in scores:
                scores[doc_id] = 0.0
            scores[doc_id] += 1.0 / (k + rank)

    ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)
    return ranked
```

**Why k=60?** The original RRF paper (Cormack et al., 2009) tested values from 1 to 1000 and found 60 worked well across multiple benchmarks. Lower k values amplify the difference between rank 1 and rank 2; higher k values treat all top results more equally.

**Example:**

| Document | BM25 Rank | Vector Rank | RRF Score (k=60) |
|---|---|---|---|
| doc_A | 1 | 5 | 1/61 + 1/65 = 0.0318 |
| doc_B | 3 | 1 | 1/63 + 1/61 = 0.0323 |
| doc_C | 2 | 8 | 1/62 + 1/68 = 0.0308 |
| doc_D | -- | 2 | 0 + 1/62 = 0.0161 |
| doc_E | 4 | -- | 1/64 + 0 = 0.0156 |

Result: doc_B > doc_A > doc_C > doc_D > doc_E

doc_B wins because it ranks highly in both lists. doc_D and doc_E, which appear in only one list, score lower.

### Weighted Score Fusion

Normalize scores from each retriever to [0, 1] and combine with weights:

```python
def weighted_score_fusion(
    bm25_results: list[tuple[str, float]],
    vector_results: list[tuple[str, float]],
    alpha: float = 0.5,
) -> list[tuple[str, float]]:
    """
    alpha: weight for vector scores (1-alpha for BM25)
    """
    def normalize(results):
        if not results:
            return {}
        scores = {doc_id: score for doc_id, score in results}
        min_s = min(scores.values())
        max_s = max(scores.values())
        if max_s == min_s:
            return {k: 1.0 for k in scores}
        return {k: (v - min_s) / (max_s - min_s) for k, v in scores.items()}

    bm25_norm = normalize(bm25_results)
    vector_norm = normalize(vector_results)

    all_docs = set(bm25_norm) | set(vector_norm)
    combined = {}
    for doc_id in all_docs:
        bm25_score = bm25_norm.get(doc_id, 0.0)
        vector_score = vector_norm.get(doc_id, 0.0)
        combined[doc_id] = (1 - alpha) * bm25_score + alpha * vector_score

    return sorted(combined.items(), key=lambda x: x[1], reverse=True)
```

**Tuning alpha**: the optimal weight depends on the query type. See "Optimal Ratio per Query Type" section below.

### Convex Combination (Weaviate-Style)

Weaviate's hybrid search uses a simpler approach:

```
hybrid_score(d) = alpha * vector_score(d) + (1 - alpha) * bm25_score(d)
```

With both scores normalized to [0, 1]. alpha=0.5 is the default.

---

## Implementation: Elasticsearch Hybrid Search

### Using kNN + BM25 in a Single Query

Elasticsearch 8.x supports combining kNN (vector) and BM25 in one query:

```json
POST /my_index/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "content": {
              "query": "database connection pooling",
              "boost": 1.0
            }
          }
        }
      ]
    }
  },
  "knn": {
    "field": "embedding",
    "query_vector": [0.12, -0.34, 0.56, ...],
    "k": 20,
    "num_candidates": 100,
    "boost": 1.0
  },
  "size": 10
}
```

Elasticsearch combines the BM25 and kNN scores using a simple sum. The `boost` parameters control relative weight.

### RRF in Elasticsearch (8.8+)

```json
POST /my_index/_search
{
  "retriever": {
    "rrf": {
      "retrievers": [
        {
          "standard": {
            "query": {
              "match": {
                "content": "database connection pooling"
              }
            }
          }
        },
        {
          "knn": {
            "field": "embedding",
            "query_vector": [0.12, -0.34, 0.56, ...],
            "k": 20,
            "num_candidates": 100
          }
        }
      ],
      "rank_window_size": 50,
      "rank_constant": 60
    }
  },
  "size": 10
}
```

### Python Implementation

```python
from elasticsearch import Elasticsearch
from sentence_transformers import SentenceTransformer

es = Elasticsearch("http://localhost:9200")
model = SentenceTransformer("sentence-transformers/all-MiniLM-L6-v2")

def hybrid_search_es(query: str, index: str, k: int = 10):
    query_vector = model.encode(query).tolist()

    response = es.search(
        index=index,
        body={
            "retriever": {
                "rrf": {
                    "retrievers": [
                        {
                            "standard": {
                                "query": {
                                    "match": {
                                        "content": query
                                    }
                                }
                            }
                        },
                        {
                            "knn": {
                                "field": "embedding",
                                "query_vector": query_vector,
                                "k": k * 3,
                                "num_candidates": k * 10,
                            }
                        },
                    ],
                    "rank_window_size": k * 5,
                    "rank_constant": 60,
                }
            },
            "size": k,
        },
    )

    return [
        {
            "id": hit["_id"],
            "content": hit["_source"]["content"],
            "score": hit["_score"],
        }
        for hit in response["hits"]["hits"]
    ]
```

---

## Implementation: Weaviate Hybrid Search

Weaviate has native hybrid search support:

```python
import weaviate
from weaviate.classes.query import HybridFusion

client = weaviate.connect_to_local()

collection = client.collections.get("Documents")

# Hybrid search with alpha (vector weight)
results = collection.query.hybrid(
    query="database connection pooling",
    alpha=0.5,                    # 0 = pure BM25, 1 = pure vector
    fusion_type=HybridFusion.RELATIVE_SCORE,  # or RANKED (RRF)
    limit=10,
    return_metadata=["score", "explain_score"],
)

for obj in results.objects:
    print(f"Score: {obj.metadata.score:.4f}")
    print(f"Content: {obj.properties['content'][:80]}")
    print(f"Explain: {obj.metadata.explain_score}")
    print("---")
```

### Weaviate Fusion Types

| Fusion | Method | When to Use |
|---|---|---|
| `RELATIVE_SCORE` | Normalize scores, weighted sum | When score magnitudes matter |
| `RANKED` | RRF on rank positions | When score distributions differ greatly |

```python
# RRF fusion (generally more robust)
results = collection.query.hybrid(
    query="database connection pooling",
    alpha=0.5,
    fusion_type=HybridFusion.RANKED,
    limit=10,
)
```

---

## Implementation: PostgreSQL Hybrid (pgvector + tsvector)

See also `vector-stores/pgvector-advanced/hybrid-pg-trgm.md` for deeper coverage.

```sql
-- Setup: table with both tsvector and vector columns
CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    content_tsv tsvector GENERATED ALWAYS AS (to_tsvector('english', content)) STORED,
    embedding vector(1536)
);

CREATE INDEX ON documents USING gin(content_tsv);
CREATE INDEX ON documents USING hnsw(embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);

-- Hybrid search with RRF in SQL
WITH bm25_results AS (
    SELECT id, content,
           ROW_NUMBER() OVER (ORDER BY ts_rank_cd(content_tsv, plainto_tsquery('english', $1)) DESC) AS rank
    FROM documents
    WHERE content_tsv @@ plainto_tsquery('english', $1)
    LIMIT 50
),
vector_results AS (
    SELECT id, content,
           ROW_NUMBER() OVER (ORDER BY embedding <=> $2::vector) AS rank
    FROM documents
    ORDER BY embedding <=> $2::vector
    LIMIT 50
),
rrf AS (
    SELECT
        COALESCE(b.id, v.id) AS id,
        COALESCE(b.content, v.content) AS content,
        COALESCE(1.0 / (60 + b.rank), 0) + COALESCE(1.0 / (60 + v.rank), 0) AS rrf_score
    FROM bm25_results b
    FULL OUTER JOIN vector_results v ON b.id = v.id
)
SELECT id, content, rrf_score
FROM rrf
ORDER BY rrf_score DESC
LIMIT 10;
```

### Python Function

```python
import psycopg2
import numpy as np
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("sentence-transformers/all-MiniLM-L6-v2")

def hybrid_search_pg(conn, query: str, k: int = 10):
    query_embedding = model.encode(query).tolist()

    sql = """
    WITH bm25 AS (
        SELECT id, content,
               ROW_NUMBER() OVER (
                   ORDER BY ts_rank_cd(content_tsv, plainto_tsquery('english', %s)) DESC
               ) AS rank
        FROM documents
        WHERE content_tsv @@ plainto_tsquery('english', %s)
        LIMIT %s
    ),
    vector AS (
        SELECT id, content,
               ROW_NUMBER() OVER (ORDER BY embedding <=> %s::vector) AS rank
        FROM documents
        ORDER BY embedding <=> %s::vector
        LIMIT %s
    ),
    fused AS (
        SELECT
            COALESCE(b.id, v.id) AS id,
            COALESCE(b.content, v.content) AS content,
            COALESCE(1.0 / (60 + b.rank), 0) + COALESCE(1.0 / (60 + v.rank), 0) AS score
        FROM bm25 b
        FULL OUTER JOIN vector v ON b.id = v.id
    )
    SELECT id, content, score FROM fused ORDER BY score DESC LIMIT %s
    """

    with conn.cursor() as cur:
        candidate_k = k * 5
        cur.execute(sql, (query, query, candidate_k,
                          str(query_embedding), str(query_embedding), candidate_k,
                          k))
        return cur.fetchall()
```

---

## When BM25-Only Outperforms

BM25 alone is better in these scenarios -- adding vector search would not help or would hurt:

| Scenario | Why BM25 Wins | Example Query |
|---|---|---|
| Exact identifiers | Embedding models average away specific tokens | "CVE-2024-3094" |
| Error codes | Codes are out-of-vocabulary for most models | "ORA-01017" |
| Function names | Code tokens are poorly embedded | "torch.nn.functional.cross_entropy" |
| Product SKUs | Arbitrary alphanumeric strings | "MBP-2024-M3-16GB" |
| Version numbers | Numerics are poorly differentiated in embedding space | "Python 3.11.4" |
| Boolean queries | AND/OR/NOT logic is not captured by embeddings | "PostgreSQL AND NOT MySQL" |
| Exact phrase matching | Embeddings lose word order | "machine learning" vs "learning machine" |

---

## When Vector-Only Wins

| Scenario | Why Vector Wins | Example |
|---|---|---|
| Paraphrasing | Different words, same meaning | "automobile repair" vs "car fixing" |
| Cross-lingual | Query in one language, docs in another | English query, French documents |
| Conceptual queries | Abstract concepts, no specific terms | "how to make my app faster" |
| Typo tolerance | Misspelled words still embed similarly | "datbase conection" |
| Synonym matching | BM25 requires exact lexical match | "big" vs "large" |

---

## Optimal Ratio per Query Type

The optimal alpha (vector weight) depends on the query:

| Query Type | Optimal alpha | BM25 Weight | Vector Weight |
|---|---|---|---|
| Exact term lookup | 0.0-0.2 | 80-100% | 0-20% |
| Technical question | 0.3-0.5 | 50-70% | 30-50% |
| Conceptual/vague question | 0.6-0.8 | 20-40% | 60-80% |
| Cross-lingual query | 0.9-1.0 | 0-10% | 90-100% |

### Dynamic Alpha Selection

Use an LLM or classifier to select alpha per query:

```python
from openai import OpenAI

client = OpenAI()

def classify_query_type(query: str) -> float:
    """Return optimal alpha (vector weight) for this query."""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": (
                    "Classify the search query into one of these types and return ONLY the number:\n"
                    "1 = exact term/code lookup (return 0.1)\n"
                    "2 = technical question with specific terms (return 0.4)\n"
                    "3 = conceptual/vague question (return 0.7)\n"
                    "4 = cross-lingual or very abstract (return 0.9)\n"
                    "Return only the float number."
                ),
            },
            {"role": "user", "content": query},
        ],
        temperature=0,
        max_tokens=5,
    )
    try:
        return float(response.choices[0].message.content.strip())
    except ValueError:
        return 0.5  # default

# Usage
alpha = classify_query_type("CVE-2024-3094 mitigation")  # -> 0.1
alpha = classify_query_type("how to speed up database queries")  # -> 0.7
```

---

## Production Architecture

```
User Query
    |
    v
[Query Analyzer]
    |
    +-- query_text --> [BM25 Search] --> ranked list 1
    |
    +-- query_embedding --> [Vector Search] --> ranked list 2
    |
    v
[Fusion (RRF or Weighted)]
    |
    v
[Top-K Results]
    |
    v (optional)
[Cross-Encoder Reranker]
    |
    v
[Final Results --> LLM Context]
```

Adding a cross-encoder reranker after fusion is the highest-quality configuration, adding 3-5% NDCG@10 on top of hybrid search alone.

---

## Common Pitfalls

1. **Using linear score combination without normalization**: BM25 scores (typically 5-30) and cosine similarity (0-1) are on completely different scales. Always normalize before combining, or use RRF (which is rank-based and scale-independent).

2. **Not retrieving enough candidates from each retriever**: if you want top-10 final results, retrieve at least top-50 from each retriever before fusion. Documents that rank 30th in BM25 but 2nd in vector search may end up in the top 5 after fusion.

3. **Fixed alpha for all queries**: as shown above, the optimal ratio varies by query type. Even a simple heuristic (shorter queries = more BM25 weight) helps.

4. **Ignoring BM25 in a "modern" stack**: teams that go all-in on vector search often find that adding BM25 back improves quality by 5-10%, especially for exact matches.

5. **Not evaluating fusion independently**: measure the quality of each retriever separately, then the fused result. This isolates whether the fusion algorithm or one of the retrievers is the bottleneck.

---

## References

- Cormack, G. et al. "Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods." SIGIR 2009.
- Elasticsearch hybrid search: https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html
- Weaviate hybrid search: https://weaviate.io/developers/weaviate/search/hybrid
- Ma, X. et al. "A Simple yet Effective Approach to Combining BM25 and Dense Retrievers." arXiv 2023.
- Karpukhin, V. et al. "Dense Passage Retrieval for Open-Domain Question Answering." EMNLP 2020.
