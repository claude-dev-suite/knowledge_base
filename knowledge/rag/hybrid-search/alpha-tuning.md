# Alpha Tuning for Hybrid Search

## TL;DR

The alpha parameter controls the balance between dense vector retrieval and sparse keyword retrieval in hybrid search. Alpha=1.0 means pure vector; alpha=0.0 means pure BM25 (in Weaviate convention -- verify your database's direction). A static alpha=0.5 is a reasonable starting point but leaves 5-15% retrieval quality on the table. This article covers systematic approaches to tuning alpha: grid search with an eval set, per-query-type alpha selection, dynamic alpha at inference time, and practical implementation patterns.

---

## Alpha Semantics Across Databases

**Critical**: the meaning of alpha differs by vendor.

| Database | alpha=0 | alpha=1 | Default |
|----------|---------|---------|---------|
| Weaviate | Pure BM25 | Pure vector | 0.75 |
| Pinecone | Pure sparse | Pure dense | 0.5 |
| Qdrant | N/A (uses fusion query, not alpha) | N/A | RRF fusion |
| Custom (EnsembleRetriever) | weights=[0, 1] = pure retriever B | weights=[1, 0] = pure retriever A | [0.5, 0.5] |

Throughout this document, we use the Weaviate convention: **alpha = vector weight, (1 - alpha) = keyword weight**. Translate accordingly for your database.

---

## Why Static Alpha Is Suboptimal

Different query types have fundamentally different retrieval characteristics:

| Query Type | Example | Optimal Alpha | Why |
|-----------|---------|--------------|-----|
| Exact identifier | "ERR_CONNECTION_REFUSED fix" | 0.15-0.25 | Exact token match is critical |
| Code lookup | "useEffect cleanup return" | 0.25-0.40 | API names are literal tokens |
| Conceptual | "how to make my database faster" | 0.70-0.85 | Semantic understanding needed |
| Mixed | "configure OAuth2 in Spring Boot 3" | 0.45-0.55 | Both meaning and terms matter |
| Factual | "what is the default port for PostgreSQL" | 0.50-0.65 | Moderate semantic + keyword |
| Comparison | "difference between RLS and RBAC" | 0.60-0.75 | Conceptual but with specific terms |

A fixed alpha=0.5 handles "Mixed" queries well but under-serves the extremes. An adaptive approach selects alpha per query, recovering 5-15% recall improvement on the tails.

---

## Method 1: Grid Search with an Eval Set

The most rigorous approach. Requires a golden evaluation dataset with queries and known-relevant documents.

### Step 1: Build the Evaluation Dataset

You need at minimum:
- 50-200 queries representative of your actual traffic
- For each query, the set of document IDs that are relevant (ground truth)
- Categorize queries by type (exact, conceptual, mixed) for analysis

```python
# eval_dataset.json structure
eval_queries = [
    {
        "query": "ERR_CONN_REFUSED PostgreSQL",
        "relevant_doc_ids": ["pg_errors_001", "pg_troubleshoot_012"],
        "query_type": "exact",
    },
    {
        "query": "how to optimize database read performance",
        "relevant_doc_ids": ["db_indexing_003", "caching_007", "read_replicas_001"],
        "query_type": "conceptual",
    },
    # ... 50-200 queries
]
```

### Step 2: Run Grid Search

```python
import json
import numpy as np
from dataclasses import dataclass


@dataclass
class AlphaEvalResult:
    alpha: float
    precision_at_10: float
    recall_at_10: float
    ndcg_at_10: float
    mrr: float


def precision_at_k(retrieved: list[str], relevant: set[str], k: int) -> float:
    retrieved_k = retrieved[:k]
    return len(set(retrieved_k) & relevant) / k


def recall_at_k(retrieved: list[str], relevant: set[str], k: int) -> float:
    if not relevant:
        return 0.0
    retrieved_k = retrieved[:k]
    return len(set(retrieved_k) & relevant) / len(relevant)


def ndcg_at_k(retrieved: list[str], relevant: set[str], k: int) -> float:
    dcg = 0.0
    for i, doc_id in enumerate(retrieved[:k]):
        if doc_id in relevant:
            dcg += 1.0 / np.log2(i + 2)  # i+2 because rank is 1-based and log2(1)=0

    # Ideal DCG: all relevant docs at top
    ideal_dcg = sum(1.0 / np.log2(i + 2) for i in range(min(len(relevant), k)))
    return dcg / ideal_dcg if ideal_dcg > 0 else 0.0


def mrr(retrieved: list[str], relevant: set[str]) -> float:
    for i, doc_id in enumerate(retrieved):
        if doc_id in relevant:
            return 1.0 / (i + 1)
    return 0.0


def grid_search_alpha(
    eval_queries: list[dict],
    search_fn,  # Callable: (query, alpha, k) -> list[doc_ids]
    alpha_values: list[float] | None = None,
    k: int = 10,
) -> list[AlphaEvalResult]:
    """
    Run grid search over alpha values.

    Args:
        eval_queries: List of {"query": str, "relevant_doc_ids": list[str]}
        search_fn: Function that takes (query, alpha, k) and returns ranked doc IDs
        alpha_values: Alpha values to test
        k: Cutoff for metrics
    """
    if alpha_values is None:
        alpha_values = [round(a * 0.05, 2) for a in range(21)]  # 0.00 to 1.00 step 0.05

    results = []
    for alpha in alpha_values:
        all_p = []
        all_r = []
        all_ndcg = []
        all_mrr = []

        for eq in eval_queries:
            relevant = set(eq["relevant_doc_ids"])
            retrieved = search_fn(eq["query"], alpha, k)

            all_p.append(precision_at_k(retrieved, relevant, k))
            all_r.append(recall_at_k(retrieved, relevant, k))
            all_ndcg.append(ndcg_at_k(retrieved, relevant, k))
            all_mrr.append(mrr(retrieved, relevant))

        result = AlphaEvalResult(
            alpha=alpha,
            precision_at_10=np.mean(all_p),
            recall_at_10=np.mean(all_r),
            ndcg_at_10=np.mean(all_ndcg),
            mrr=np.mean(all_mrr),
        )
        results.append(result)
        print(
            f"alpha={alpha:.2f}: "
            f"P@{k}={result.precision_at_10:.4f} "
            f"R@{k}={result.recall_at_10:.4f} "
            f"NDCG@{k}={result.ndcg_at_10:.4f} "
            f"MRR={result.mrr:.4f}"
        )

    return results


# Weaviate search function example
import weaviate

client = weaviate.connect_to_local()
collection = client.collections.get("Documents")


def weaviate_hybrid(query: str, alpha: float, k: int) -> list[str]:
    results = collection.query.hybrid(
        query=query,
        alpha=alpha,
        limit=k,
    )
    return [obj.properties["doc_id"] for obj in results.objects]


# Run the grid search
alpha_results = grid_search_alpha(eval_queries, weaviate_hybrid)

# Find optimal
best = max(alpha_results, key=lambda r: r.ndcg_at_10)
print(f"\nBest alpha: {best.alpha} (NDCG@10 = {best.ndcg_at_10:.4f})")
```

### Step 3: Analyze by Query Type

A single global alpha is a compromise. Segment results by query type to see if per-type alpha would help:

```python
def grid_search_by_type(
    eval_queries: list[dict],
    search_fn,
    alpha_values: list[float] | None = None,
    k: int = 10,
) -> dict[str, float]:
    """Find optimal alpha per query type."""
    if alpha_values is None:
        alpha_values = [round(a * 0.05, 2) for a in range(21)]

    # Group queries by type
    by_type: dict[str, list[dict]] = {}
    for eq in eval_queries:
        qtype = eq.get("query_type", "unknown")
        by_type.setdefault(qtype, []).append(eq)

    optimal_alphas = {}
    for qtype, queries in by_type.items():
        results = grid_search_alpha(queries, search_fn, alpha_values, k)
        best = max(results, key=lambda r: r.ndcg_at_10)
        optimal_alphas[qtype] = best.alpha
        print(f"Query type '{qtype}': optimal alpha = {best.alpha}")

    return optimal_alphas

# Typical output:
# Query type 'exact': optimal alpha = 0.20
# Query type 'conceptual': optimal alpha = 0.75
# Query type 'mixed': optimal alpha = 0.50
# Query type 'code': optimal alpha = 0.35
```

---

## Method 2: Per-Query-Type Alpha (Static Classification)

Use a rule-based classifier to select alpha at query time based on query characteristics.

```python
import re
from enum import Enum


class QueryType(Enum):
    EXACT = "exact"
    CODE = "code"
    CONCEPTUAL = "conceptual"
    FACTUAL = "factual"
    COMPARISON = "comparison"
    MIXED = "mixed"


# Map query types to optimal alphas (tuned via grid search on your eval set)
ALPHA_MAP: dict[QueryType, float] = {
    QueryType.EXACT: 0.20,
    QueryType.CODE: 0.35,
    QueryType.CONCEPTUAL: 0.75,
    QueryType.FACTUAL: 0.55,
    QueryType.COMPARISON: 0.65,
    QueryType.MIXED: 0.50,
}


def classify_query(query: str) -> QueryType:
    """Rule-based query type classifier."""
    query_lower = query.lower().strip()

    # Exact identifiers: error codes, versions, constants, UUIDs
    if re.search(
        r'[A-Z_]{3,}|v\d+\.\d+|0x[0-9a-f]+|error.?code|status\s*\d{3}|'
        r'[0-9a-f]{8}-[0-9a-f]{4}|port\s*\d{2,5}',
        query,
    ):
        return QueryType.EXACT

    # Code patterns: function calls, operators, brackets
    if re.search(r'[(){}\[\]]|::|->|\.\w+\(|import\s|from\s.*import|def\s|class\s', query):
        return QueryType.CODE

    # Comparison queries
    if re.search(r'\b(difference|vs\.?|versus|compare|between)\b', query_lower):
        return QueryType.COMPARISON

    # Conceptual/how-to queries
    if re.search(
        r'\b(how|what|why|when|explain|overview|best.?practice|guide|tutorial|'
        r'approach|strategy|pattern|recommend)\b',
        query_lower,
    ):
        return QueryType.CONCEPTUAL

    # Factual queries (short, specific)
    if re.search(r'\b(what is|default|maximum|minimum|limit|size|version)\b', query_lower):
        return QueryType.FACTUAL

    return QueryType.MIXED


def get_alpha(query: str) -> float:
    """Get the optimal alpha for a query."""
    query_type = classify_query(query)
    return ALPHA_MAP[query_type]


# Examples
assert classify_query("ERR_CONN_REFUSED fix PostgreSQL") == QueryType.EXACT
assert classify_query("useEffect(() => { cleanup(); })") == QueryType.CODE
assert classify_query("how to optimize database queries") == QueryType.CONCEPTUAL
assert classify_query("difference between RLS and RBAC") == QueryType.COMPARISON
assert classify_query("configure OAuth2 Spring Boot") == QueryType.MIXED
```

### Integration with Weaviate

```python
def adaptive_hybrid_search(
    collection,
    query: str,
    k: int = 10,
) -> list:
    alpha = get_alpha(query)
    results = collection.query.hybrid(
        query=query,
        alpha=alpha,
        limit=k,
    )
    return results.objects
```

---

## Method 3: Dynamic Alpha with LLM Classification

For higher accuracy than regex rules, use a fast LLM to classify query type:

```python
import anthropic

client = anthropic.Anthropic()

CLASSIFICATION_PROMPT = """\
Classify this search query into exactly one category. Respond with ONLY the category name.

Categories:
- exact: Contains specific identifiers, error codes, version numbers, config keys
- code: Contains code syntax, function names, imports
- conceptual: Asks how/why/when, seeks understanding or best practices
- factual: Asks for specific facts (defaults, limits, versions)
- comparison: Compares two or more concepts
- mixed: Combination that does not fit a single category

Query: {query}
Category:"""


ALPHA_MAP_LLM = {
    "exact": 0.20,
    "code": 0.35,
    "conceptual": 0.75,
    "factual": 0.55,
    "comparison": 0.65,
    "mixed": 0.50,
}


def classify_with_llm(query: str) -> tuple[str, float]:
    """Classify query type using Claude Haiku for speed."""
    response = client.messages.create(
        model="claude-3-5-haiku-20241022",
        max_tokens=10,
        messages=[{
            "role": "user",
            "content": CLASSIFICATION_PROMPT.format(query=query),
        }],
    )
    category = response.content[0].text.strip().lower()
    alpha = ALPHA_MAP_LLM.get(category, 0.50)
    return category, alpha
```

**Cost-performance tradeoff**: Haiku classification adds ~50ms and ~$0.0001 per query. For high-throughput systems, cache classifications for repeated query patterns:

```python
from functools import lru_cache

@lru_cache(maxsize=10000)
def cached_classify(query: str) -> float:
    """Cache LLM classifications to avoid repeated calls."""
    _, alpha = classify_with_llm(query)
    return alpha
```

---

## Method 4: Learned Alpha (Regression Model)

Train a lightweight model to predict optimal alpha directly from query features:

```python
from sklearn.ensemble import GradientBoostingRegressor
from sklearn.feature_extraction.text import TfidfVectorizer
import numpy as np
import pickle


def build_query_features(query: str) -> dict:
    """Extract features for alpha prediction."""
    tokens = query.split()
    return {
        "length": len(tokens),
        "has_uppercase": int(bool(re.search(r'[A-Z]{2,}', query))),
        "has_code_chars": int(bool(re.search(r'[(){}\[\]<>]', query))),
        "has_question_word": int(bool(re.search(r'\b(how|what|why|when)\b', query.lower()))),
        "has_version": int(bool(re.search(r'v?\d+\.\d+', query))),
        "has_error_pattern": int(bool(re.search(r'error|exception|fail|crash', query.lower()))),
        "avg_token_length": np.mean([len(t) for t in tokens]) if tokens else 0,
        "num_digits": sum(c.isdigit() for c in query),
        "special_char_ratio": sum(not c.isalnum() and not c.isspace() for c in query) / max(len(query), 1),
    }


# Training: requires labeled data (query, optimal_alpha) from grid search
# For each eval query, the optimal alpha is the one that maximizes NDCG@10 for that query
def train_alpha_predictor(
    queries: list[str],
    optimal_alphas: list[float],
) -> GradientBoostingRegressor:
    """Train a regression model to predict optimal alpha."""
    features = [list(build_query_features(q).values()) for q in queries]
    X = np.array(features)
    y = np.array(optimal_alphas)

    model = GradientBoostingRegressor(
        n_estimators=100,
        max_depth=4,
        learning_rate=0.1,
    )
    model.fit(X, y)
    return model


def predict_alpha(model: GradientBoostingRegressor, query: str) -> float:
    """Predict optimal alpha for a new query."""
    features = list(build_query_features(query).values())
    alpha = model.predict([features])[0]
    return float(np.clip(alpha, 0.0, 1.0))
```

This approach requires a labeled training set (generated from per-query grid search) but once trained, inference is sub-millisecond with no API calls.

---

## Evaluation: Measuring Alpha Impact

### A/B Testing Alpha Strategies

```python
def compare_alpha_strategies(
    eval_queries: list[dict],
    search_fn,
    strategies: dict[str, callable],  # name -> fn(query) -> alpha
    k: int = 10,
) -> dict[str, dict[str, float]]:
    """Compare alpha selection strategies on the eval set."""
    results = {}

    for name, alpha_fn in strategies.items():
        all_ndcg = []
        all_mrr = []
        for eq in eval_queries:
            alpha = alpha_fn(eq["query"])
            retrieved = search_fn(eq["query"], alpha, k)
            relevant = set(eq["relevant_doc_ids"])
            all_ndcg.append(ndcg_at_k(retrieved, relevant, k))
            all_mrr.append(mrr(retrieved, relevant))

        results[name] = {
            "NDCG@10": np.mean(all_ndcg),
            "MRR": np.mean(all_mrr),
        }
        print(f"{name:25s}: NDCG@10={results[name]['NDCG@10']:.4f}  MRR={results[name]['MRR']:.4f}")

    return results


# Compare strategies
strategies = {
    "static_0.50": lambda q: 0.50,
    "static_0.60": lambda q: 0.60,
    "rule_based": lambda q: get_alpha(q),
    "llm_classified": lambda q: cached_classify(q),
}

compare_alpha_strategies(eval_queries, weaviate_hybrid, strategies)

# Typical output:
# static_0.50              : NDCG@10=0.5823  MRR=0.6145
# static_0.60              : NDCG@10=0.5912  MRR=0.6203
# rule_based               : NDCG@10=0.6187  MRR=0.6489
# llm_classified           : NDCG@10=0.6341  MRR=0.6622
```

---

## Production Architecture

### Recommended Pipeline

```
Query
  |
  v
[Query Classifier] --> alpha value
  |
  v
[Parallel Retrieval]
  |                 |
  v                 v
[Vector Search]  [BM25 Search]
  |                 |
  v                 v
[Fusion (RRF or weighted linear with alpha)]
  |
  v
[Reranker (optional)]
  |
  v
[Top-K results]
```

### Implementation with FastAPI

```python
from fastapi import FastAPI
from pydantic import BaseModel
import weaviate

app = FastAPI()
weaviate_client = weaviate.connect_to_local()
collection = weaviate_client.collections.get("Documents")


class SearchRequest(BaseModel):
    query: str
    k: int = 10
    alpha: float | None = None  # None = auto-select


class SearchResult(BaseModel):
    doc_id: str
    content: str
    score: float
    alpha_used: float


@app.post("/search", response_model=list[SearchResult])
async def search(req: SearchRequest):
    # Auto-select alpha if not provided
    alpha = req.alpha if req.alpha is not None else get_alpha(req.query)

    results = collection.query.hybrid(
        query=req.query,
        alpha=alpha,
        limit=req.k,
        return_metadata=["score"],
    )

    return [
        SearchResult(
            doc_id=obj.properties.get("doc_id", ""),
            content=obj.properties.get("content", ""),
            score=obj.metadata.score or 0.0,
            alpha_used=alpha,
        )
        for obj in results.objects
    ]
```

### Logging Alpha for Analysis

Track which alpha values are selected and their retrieval outcomes:

```python
import logging
import json

logger = logging.getLogger("hybrid_search")


def search_with_logging(query: str, k: int = 10) -> list:
    query_type = classify_query(query)
    alpha = ALPHA_MAP[query_type]

    results = collection.query.hybrid(query=query, alpha=alpha, limit=k)

    logger.info(json.dumps({
        "query": query,
        "query_type": query_type.value,
        "alpha": alpha,
        "num_results": len(results.objects),
        "top_score": results.objects[0].metadata.score if results.objects else None,
    }))

    return results.objects
```

---

## Common Pitfalls

1. **Tuning alpha without an eval set.** Intuition-based alpha is rarely optimal. Even a small eval set (50 queries) dramatically improves tuning quality.
2. **Optimizing for a single metric.** Optimize NDCG@10 (considers ranking quality) rather than just recall. High recall with poor ranking wastes the LLM's context window.
3. **Forgetting to re-tune after corpus changes.** When you add new document types or the query distribution shifts, optimal alpha may change. Re-run grid search quarterly.
4. **Over-engineering dynamic alpha.** If your query types are homogeneous (e.g., all conceptual questions), a single tuned alpha is sufficient. Dynamic selection adds latency and complexity that is only justified for diverse query distributions.
5. **Confusing alpha direction across databases.** Weaviate: alpha=0 is BM25. Pinecone: alpha=0 is sparse (similar meaning but check docs). Always verify with a sanity test: search for a unique exact string and confirm alpha=0 returns it.
6. **Not accounting for the reranker.** If you use a cross-encoder reranker after hybrid search, the alpha matters less because the reranker re-scores everything. In this case, optimize for recall@K_initial rather than precision, and use a moderate alpha.

---

## References

- Weaviate hybrid search alpha: https://weaviate.io/developers/weaviate/search/hybrid
- Ma et al. "A Systematic Investigation of Hybrid Fusion Methods for Passage Retrieval" (ACL 2024)
- Pinecone hybrid search: https://docs.pinecone.io/guides/data/understanding-hybrid-search
- Chen et al. "Out-of-Domain Semantics to the Rescue! Zero-Shot Hybrid Retrieval Models" (ECIR 2023)
