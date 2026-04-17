# Hybrid Search for RAG -- Comprehensive Guide

## TL;DR

Hybrid search fuses dense vector retrieval (semantic) with sparse keyword retrieval (BM25/full-text) to compensate for the blind spots of each. Dense search handles synonyms and conceptual queries; sparse search handles exact identifiers, rare tokens, and negation. Combining them with rank fusion (typically Reciprocal Rank Fusion) produces recall and precision that neither achieves alone. Every production RAG system handling mixed query types should use hybrid search.

---

## Why Neither Retrieval Mode Is Sufficient Alone

### Dense Vector Search Limitations

Dense retrieval encodes query and documents into fixed-dimensional embeddings (768-3072 dims) and scores by cosine/dot-product similarity. This works well for paraphrase and conceptual matching, but fails predictably in several cases:

1. **Exact identifiers** -- error codes (`ECONNREFUSED`), version strings (`v4.2.1`), model names (`claude-3-5-sonnet`), UUIDs. Embedding models compress these into the same neighborhood as semantically similar but literally different strings.
2. **Rare domain terms** -- abbreviations (`RLS`, `RBAC`, `CTE`) or niche jargon not well-represented in the embedding model's pretraining data.
3. **Negation and boolean logic** -- "authentication WITHOUT OAuth" and "authentication WITH OAuth" produce nearly identical embeddings because the negation word contributes minimal signal.
4. **Out-of-vocabulary tokens** -- new library names, internal product names, custom enums.
5. **Numeric precision** -- "error code 429" vs "error code 500" embed similarly despite being semantically distinct.

### Sparse Keyword Search (BM25) Limitations

BM25 (Best Matching 25) scores documents by term frequency, inverse document frequency, and document length normalization. It excels at exact-match queries but struggles with:

1. **Synonyms** -- "authentication" vs "login" vs "sign-in" vs "auth".
2. **Conceptual queries** -- "how to make my database faster" should match indexing, query optimization, and caching docs.
3. **Paraphrasing** -- different phrasings of identical questions.
4. **Morphological variation** -- stemming helps but does not fully resolve "containerize" vs "run in Docker".
5. **Cross-lingual queries** -- a query in one language should ideally match content in another.

### Empirical Evidence

Multiple studies confirm hybrid superiority:

| Benchmark | BM25 NDCG@10 | Dense NDCG@10 | Hybrid NDCG@10 | Source |
|-----------|-------------|--------------|----------------|--------|
| BEIR (avg 18 datasets) | 0.428 | 0.443 | 0.491 | Thakur et al. 2021 |
| MS MARCO Dev | 0.228 | 0.334 | 0.367 | MTEB leaderboard |
| NQ (Natural Questions) | 0.329 | 0.468 | 0.504 | Weaviate benchmark |
| FiQA (finance) | 0.236 | 0.295 | 0.341 | BEIR subset |
| SciFact | 0.665 | 0.637 | 0.712 | BEIR subset |

Key insight: the datasets where BM25 wins over dense (SciFact, FiQA) still benefit from hybrid because each method retrieves different relevant documents.

---

## BM25 Internals for RAG Engineers

Understanding BM25 scoring helps tune hybrid systems. The formula:

```
score(q, d) = SUM over t in q:
    IDF(t) * (tf(t,d) * (k1 + 1)) / (tf(t,d) + k1 * (1 - b + b * |d| / avgdl))
```

Where:
- `tf(t,d)` = frequency of term t in document d
- `IDF(t) = log((N - n(t) + 0.5) / (n(t) + 0.5) + 1)` (N = total docs, n(t) = docs containing t)
- `k1` = term frequency saturation (default 1.2-2.0)
- `b` = document length normalization (default 0.75)
- `avgdl` = average document length

**For chunked RAG corpora**: set `b` lower (0.3-0.5) because chunks are already roughly equal length. High `b` penalizes longer chunks too aggressively.

### BM25 in Python with rank-bm25

```python
from rank_bm25 import BM25Okapi
import re

def tokenize(text: str) -> list[str]:
    """Simple whitespace + lowercasing tokenizer."""
    return re.findall(r'\w+', text.lower())

corpus = [
    "PostgreSQL row level security policies",
    "How to configure RLS in PostgreSQL databases",
    "MySQL user privilege management",
    "Database access control with RBAC patterns",
]

tokenized_corpus = [tokenize(doc) for doc in corpus]
bm25 = BM25Okapi(tokenized_corpus, k1=1.5, b=0.4)

query = "PostgreSQL RLS setup"
scores = bm25.get_scores(tokenize(query))
# Returns: array of BM25 scores, one per document
```

---

## Score Fusion Strategies

The core challenge of hybrid search: how to combine ranked lists from different retrieval systems whose scores are on incomparable scales.

### Strategy 1: Reciprocal Rank Fusion (RRF)

RRF is rank-based -- it ignores raw scores entirely and operates only on the rank position. This makes it the most robust and widely-used fusion strategy.

```
RRF_score(d) = SUM over each retriever i: 1 / (k + rank_i(d))
```

Where `k` is a damping constant (default 60). See the companion article `rrf-deep.md` for full mathematical analysis and tuning.

```python
def reciprocal_rank_fusion(
    result_lists: list[list[str]],
    k: int = 60,
) -> list[tuple[str, float]]:
    """Fuse multiple ranked lists using RRF."""
    scores: dict[str, float] = {}
    for results in result_lists:
        for rank, doc_id in enumerate(results, start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

### Strategy 2: Weighted Linear Combination (Normalized Scores)

Requires normalizing scores from each retriever to [0, 1] before combining:

```python
def min_max_normalize(scores: list[float]) -> list[float]:
    mn, mx = min(scores), max(scores)
    if mx == mn:
        return [1.0] * len(scores)
    return [(s - mn) / (mx - mn) for s in scores]


def weighted_linear_fusion(
    vector_results: list[tuple[str, float]],
    keyword_results: list[tuple[str, float]],
    alpha: float = 0.5,
) -> list[tuple[str, float]]:
    """
    Fuse using normalized score combination.
    alpha = vector weight, (1-alpha) = keyword weight.
    """
    # Normalize each list
    v_ids = [r[0] for r in vector_results]
    v_scores = min_max_normalize([r[1] for r in vector_results])
    k_ids = [r[0] for r in keyword_results]
    k_scores = min_max_normalize([r[1] for r in keyword_results])

    combined: dict[str, float] = {}
    for doc_id, score in zip(v_ids, v_scores):
        combined[doc_id] = combined.get(doc_id, 0.0) + alpha * score
    for doc_id, score in zip(k_ids, k_scores):
        combined[doc_id] = combined.get(doc_id, 0.0) + (1 - alpha) * score

    return sorted(combined.items(), key=lambda x: x[1], reverse=True)
```

**When to use linear combination over RRF**: when you have well-calibrated scores and want finer control over the contribution of each retriever. In practice, RRF is safer.

### Strategy 3: Distribution-Based Score Fusion (DBSF)

Used by Weaviate's `RELATIVE_SCORE` fusion. Normalizes each retriever's scores based on their statistical distribution (z-score normalization) rather than min-max. More robust when score distributions are skewed.

```python
import numpy as np

def dbsf_normalize(scores: list[float]) -> list[float]:
    """Normalize scores using mean and std (z-score to [0,1])."""
    arr = np.array(scores)
    mean, std = arr.mean(), arr.std()
    if std == 0:
        return [0.5] * len(scores)
    z = (arr - mean) / std
    # Map z-scores to [0, 1] using sigmoid-like clamping
    normalized = (z - z.min()) / (z.max() - z.min())
    return normalized.tolist()
```

---

## Implementation with Major Vector Databases

### Weaviate

Weaviate supports hybrid search natively via its `hybrid` query endpoint.

```python
import weaviate
from weaviate.classes.query import HybridFusion

client = weaviate.connect_to_local()  # or connect_to_weaviate_cloud(...)
collection = client.collections.get("Documents")

# Hybrid search
results = collection.query.hybrid(
    query="PostgreSQL row level security configuration",
    alpha=0.5,          # 0.0 = pure BM25, 1.0 = pure vector
    limit=10,
    fusion_type=HybridFusion.RELATIVE_SCORE,  # DBSF-based
    # fusion_type=HybridFusion.RANKED,        # RRF-based
    return_metadata=["score", "explain_score"],
)

for obj in results.objects:
    print(f"{obj.metadata.score:.4f} | {obj.properties['title']}")
```

**Weaviate alpha semantics**: `alpha=0` is pure keyword (BM25), `alpha=1` is pure vector. This is the inverse of Pinecone.

### Qdrant

Qdrant supports hybrid search via its `query` API with prefetch + fusion.

```python
from qdrant_client import QdrantClient, models

client = QdrantClient("localhost", port=6333)

# Hybrid: prefetch from both dense and sparse, then fuse
results = client.query_points(
    collection_name="documents",
    prefetch=[
        # Dense vector search
        models.Prefetch(
            query=dense_embedding,  # list[float]
            using="dense",
            limit=20,
        ),
        # Sparse vector search (BM25 via SPLADE or manual)
        models.Prefetch(
            query=models.SparseVector(
                indices=sparse_indices,
                values=sparse_values,
            ),
            using="sparse",
            limit=20,
        ),
    ],
    query=models.FusionQuery(fusion=models.Fusion.RRF),
    limit=10,
)
```

### Pinecone

```python
from pinecone import Pinecone
from pinecone_text.sparse import BM25Encoder

pc = Pinecone(api_key="your-key")
index = pc.Index("my-index")

bm25 = BM25Encoder()
bm25.fit(corpus_texts)

dense_query = embedding_model.embed(query)
sparse_query = bm25.encode_queries(query)

results = index.query(
    vector=dense_query,
    sparse_vector=sparse_query,
    top_k=10,
    include_metadata=True,
)
```

### PostgreSQL (pgvector + tsvector)

Native hybrid search within a single database. See the `rag-patterns/hybrid-search.md` document for the complete SQL implementation with the `hybrid_search()` stored function.

```python
import psycopg

async def hybrid_search(
    conn: psycopg.AsyncConnection,
    query_embedding: list[float],
    query_text: str,
    k: int = 10,
    vector_weight: float = 0.5,
) -> list[dict]:
    rows = await conn.execute(
        """
        SELECT * FROM hybrid_search(
            $1::vector, $2, $3, $4, $5
        )
        """,
        [query_embedding, query_text, k, vector_weight, 1 - vector_weight],
    )
    return [dict(r) for r in await rows.fetchall()]
```

---

## LangChain EnsembleRetriever

LangChain provides a high-level abstraction that fuses any two retrievers using RRF:

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

# Build retrievers
vectorstore = Chroma.from_documents(documents, OpenAIEmbeddings())
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})

bm25_retriever = BM25Retriever.from_documents(documents, k=20)

# Ensemble
hybrid_retriever = EnsembleRetriever(
    retrievers=[vector_retriever, bm25_retriever],
    weights=[0.5, 0.5],
)

docs = hybrid_retriever.invoke("How to configure RLS in PostgreSQL?")
```

### LlamaIndex QueryFusionRetriever

```python
from llama_index.core.retrievers import QueryFusionRetriever
from llama_index.retrievers.bm25 import BM25Retriever as LlamaBM25

vector_retriever = index.as_retriever(similarity_top_k=20)
bm25_retriever = LlamaBM25.from_defaults(
    index=index, similarity_top_k=20
)

fusion_retriever = QueryFusionRetriever(
    retrievers=[vector_retriever, bm25_retriever],
    similarity_top_k=10,
    num_queries=1,         # Set to 1 to disable query expansion
    mode="reciprocal_rerank",  # Uses RRF
)

nodes = fusion_retriever.retrieve("PostgreSQL RLS setup")
```

---

## When Hybrid Wins vs. When It Does Not

### Hybrid search clearly wins:

- Technical documentation with API names, config keys, error codes
- Legal/compliance content with specific clause identifiers
- Medical/scientific text with abbreviations and formal nomenclature
- Mixed query types (some users type keywords, others ask questions)
- Multilingual corpora where BM25 covers exact-match in different scripts

### Hybrid search may not be necessary:

- Pure conceptual/semantic corpus (e.g., literary analysis, philosophical texts)
- Very small corpus (<500 chunks) where brute-force works fine
- When all queries are natural-language questions with no exact-match component
- Cost-constrained environments where maintaining two indexes is prohibitive

### Decision framework:

```python
def should_use_hybrid(
    corpus_has_identifiers: bool,
    query_types_are_mixed: bool,
    corpus_size: int,
    budget_allows_two_indexes: bool,
) -> bool:
    if not budget_allows_two_indexes:
        return False
    if corpus_size < 500:
        return False  # Not worth the complexity
    if corpus_has_identifiers or query_types_are_mixed:
        return True
    return False  # Pure vector likely sufficient
```

---

## Adaptive Alpha Selection

Rather than a static alpha, classify queries at runtime and adjust the BM25-vs-vector weight dynamically. See the companion article `alpha-tuning.md` for the full methodology.

Quick heuristic:

```python
import re

def adaptive_alpha(query: str) -> float:
    """Return alpha (vector weight) based on query characteristics."""
    # Exact identifiers -> boost keyword
    if re.search(r'[A-Z_]{3,}|v\d+\.\d+|\b\d{3}\b|error.?code', query):
        return 0.25  # Heavy BM25

    # Conceptual / how-to -> boost vector
    if re.search(r'\b(how|what|why|explain|best.?practice|overview)\b', query, re.I):
        return 0.75  # Heavy vector

    # Code-like queries
    if re.search(r'[(){}\[\]<>]|::|->|\.\w+\(', query):
        return 0.35

    return 0.50  # Balanced default
```

---

## Performance Optimization

### Parallel Retrieval

Run BM25 and vector search concurrently to avoid sequential latency:

```python
import asyncio

async def parallel_hybrid_search(
    query: str,
    query_embedding: list[float],
    k: int = 20,
    alpha: float = 0.5,
) -> list[dict]:
    vector_task = asyncio.create_task(
        vector_store.asimilarity_search_with_score(query_embedding, k=k)
    )
    bm25_task = asyncio.create_task(
        bm25_index.asearch(query, k=k)
    )
    vector_results, bm25_results = await asyncio.gather(vector_task, bm25_task)
    return fuse_results(vector_results, bm25_results, alpha=alpha)
```

### Caching BM25 Index

For static or slowly-changing corpora, serialize the BM25 index to avoid rebuilding:

```python
import pickle
from rank_bm25 import BM25Okapi

# Build and save
bm25 = BM25Okapi(tokenized_corpus)
with open("bm25_index.pkl", "wb") as f:
    pickle.dump(bm25, f)

# Load
with open("bm25_index.pkl", "rb") as f:
    bm25 = pickle.load(f)
```

### Pre-filtering

Apply metadata filters before fusion to reduce the candidate set:

```python
# Filter by date range + document type, then hybrid search
results = collection.query.hybrid(
    query="RLS configuration",
    alpha=0.5,
    limit=10,
    filters=weaviate.classes.query.Filter.by_property("doc_type").equal("api-reference")
    & weaviate.classes.query.Filter.by_property("updated_at").greater_than("2024-01-01"),
)
```

---

## Common Pitfalls

1. **Score-based fusion without normalization.** BM25 scores range 0 to ~30; cosine similarity ranges 0 to 1. Combining raw scores is meaningless. Use rank-based fusion (RRF) or normalize first.
2. **Confusing alpha direction.** Weaviate: alpha=0 is pure BM25. Pinecone: alpha=0 is pure sparse. Always verify in the vendor docs.
3. **Not tokenizing domain terms for BM25.** If your corpus uses `row_level_security` but users query "row level security", the BM25 index must tokenize underscored compound words.
4. **Static alpha for diverse query types.** A fixed 0.5 alpha is a decent default but leaves 5-15% recall on the table compared to adaptive alpha. See `alpha-tuning.md`.
5. **Ignoring the full-text analyzer configuration.** Default English analyzers remove stop words and apply Porter stemming. For code-heavy corpora, use a custom analyzer that preserves camelCase tokens.
6. **Over-fetching from one retriever.** If you fetch top-5 from BM25 but top-50 from vector, the RRF fusion is biased. Fetch the same number from each retriever (typically 2-3x the final k).
7. **Not benchmarking hybrid against pure vector.** For some corpora (pure Q&A, no identifiers), hybrid adds cost with minimal gain. Always measure with your actual queries.

---

## Production Checklist

- [ ] Hybrid search uses RRF or DBSF fusion (not raw score addition)
- [ ] Alpha/weight is tuned per query type or adaptively classified
- [ ] Both retrievers fetch the same candidate pool size (2-3x final k)
- [ ] BM25 tokenizer handles domain-specific terms (underscores, camelCase, abbreviations)
- [ ] Full-text index uses appropriate language analyzer with correct stemming
- [ ] Retrieval latency is measured end-to-end (both retrievers + fusion)
- [ ] Vector and keyword retrievers run in parallel (not sequential)
- [ ] Metadata pre-filtering is applied before retrieval when applicable
- [ ] Evaluation compares pure vector, pure BM25, and hybrid on your actual eval set
- [ ] Fallback to pure vector when BM25 returns zero results (empty keyword match)
- [ ] BM25 index is rebuilt/updated when corpus changes significantly
- [ ] Alpha direction is verified for the specific vector database in use

---

## References

- Thakur et al. "BEIR: A Heterogeneous Benchmark for Zero-shot Evaluation of Information Retrieval Models" (NeurIPS 2021)
- Cormack, Clarke, Buettcher. "Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods" (SIGIR 2009)
- Robertson & Zaragoza. "The Probabilistic Relevance Framework: BM25 and Beyond" (Foundations and Trends in IR, 2009)
- Weaviate hybrid search docs: https://weaviate.io/developers/weaviate/search/hybrid
- Qdrant hybrid search: https://qdrant.tech/documentation/concepts/hybrid-queries/
- Pinecone hybrid search: https://docs.pinecone.io/guides/data/understanding-hybrid-search
- LangChain EnsembleRetriever: https://python.langchain.com/docs/how_to/ensemble_retriever/
