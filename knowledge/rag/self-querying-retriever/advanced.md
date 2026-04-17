# Advanced Self-Querying -- Hybrid Combination, Filter Correctness Evaluation, and Fallback on Parse Failure

## TL;DR

The basic self-querying retriever has three weaknesses: it relies solely on the LLM parser (which can fail or produce incorrect filters), it does not combine well with other retrieval strategies (like BM25 keyword search), and there is no systematic way to evaluate whether the generated filters are correct. This article covers advanced patterns: combining self-querying with hybrid search (vector + BM25), evaluating filter correctness with ground-truth test sets, graceful fallback strategies when the parser fails, and filter caching to avoid repeated LLM parse calls for similar queries.

---

## Combining Self-Querying with Hybrid Search

### Architecture

```
User query: "Python error handling tutorials from 2024"
    |
    v
[LLM Parser]
    |
    +--> semantic_query: "error handling tutorials"
    +--> filters: {language: "python", year: 2024}
    |
    +----------+----------+
    |                     |
    v                     v
[Vector search]    [BM25 keyword search]
  (semantic)         (exact terms)
    |                     |
    +----------+----------+
               |
               v
         [Merge + Rerank]
               |
               v
         [Apply metadata filters]
               |
               v
         [Final results]
```

### Implementation

```python
from langchain_core.documents import Document
import numpy as np


class HybridSelfQueryRetriever:
    """Combine self-querying metadata filters with hybrid
    (vector + BM25) retrieval for maximum recall and precision.

    The LLM parser extracts filters, while both vector and
    keyword search contribute candidate documents.
    """

    def __init__(
        self,
        vector_store,
        bm25_retriever,
        parser,  # SelfQueryParser from overview
        vector_weight: float = 0.6,
        bm25_weight: float = 0.4,
    ):
        self.vector_store = vector_store
        self.bm25 = bm25_retriever
        self.parser = parser
        self.vector_weight = vector_weight
        self.bm25_weight = bm25_weight

    def invoke(self, query: str, k: int = 10) -> list:
        """Retrieve with hybrid search + self-querying filters."""
        # Step 1: Parse query
        structured = self.parser.parse(query)

        # Step 2: Vector search (uses semantic query)
        vector_results = self.vector_store.similarity_search_with_score(
            structured.semantic_query,
            k=k * 2,  # Over-retrieve before filtering
        )

        # Step 3: BM25 keyword search (uses semantic query)
        bm25_results = self.bm25.invoke(structured.semantic_query)

        # Step 4: Merge results using Reciprocal Rank Fusion
        merged = self._reciprocal_rank_fusion(
            vector_results, bm25_results, k=k * 2
        )

        # Step 5: Apply metadata filters
        filtered = self._apply_filters(merged, structured.filters)

        return filtered[:k]

    def _reciprocal_rank_fusion(
        self,
        vector_results: list[tuple],
        bm25_results: list,
        k: int = 60,
    ) -> list[Document]:
        """Merge results from vector and BM25 using RRF."""
        doc_scores: dict[str, float] = {}
        doc_map: dict[str, Document] = {}

        # Score vector results
        for rank, (doc, score) in enumerate(vector_results):
            doc_id = doc.metadata.get("doc_id", doc.page_content[:50])
            rrf_score = self.vector_weight / (k + rank + 1)
            doc_scores[doc_id] = doc_scores.get(doc_id, 0) + rrf_score
            doc_map[doc_id] = doc

        # Score BM25 results
        for rank, doc in enumerate(bm25_results):
            doc_id = doc.metadata.get("doc_id", doc.page_content[:50])
            rrf_score = self.bm25_weight / (k + rank + 1)
            doc_scores[doc_id] = doc_scores.get(doc_id, 0) + rrf_score
            if doc_id not in doc_map:
                doc_map[doc_id] = doc

        # Sort by combined score
        sorted_ids = sorted(
            doc_scores.keys(),
            key=lambda x: doc_scores[x],
            reverse=True,
        )

        return [doc_map[doc_id] for doc_id in sorted_ids]

    def _apply_filters(
        self, docs: list[Document], filters: dict
    ) -> list[Document]:
        """Apply parsed metadata filters to documents."""
        if not filters:
            return docs

        filtered = []
        for doc in docs:
            if self._matches_filters(doc.metadata, filters):
                filtered.append(doc)
        return filtered

    def _matches_filters(self, metadata: dict, filters: dict) -> bool:
        """Check if document metadata matches all filters."""
        for field, constraint in filters.items():
            value = metadata.get(field)
            if value is None:
                return False

            if isinstance(constraint, dict):
                for op, target in constraint.items():
                    if op == "$eq" and value != target:
                        return False
                    elif op == "$ne" and value == target:
                        return False
                    elif op == "$gt" and not (value > target):
                        return False
                    elif op == "$gte" and not (value >= target):
                        return False
                    elif op == "$lt" and not (value < target):
                        return False
                    elif op == "$lte" and not (value <= target):
                        return False
                    elif op == "$in" and value not in target:
                        return False
                    elif op == "$contains" and target not in str(value):
                        return False
            else:
                # Simple equality
                if value != constraint:
                    return False

        return True
```

---

## Evaluating Filter Correctness

### Ground-Truth Test Set

```python
from dataclasses import dataclass


@dataclass
class FilterTestCase:
    """A test case for evaluating filter generation accuracy."""

    query: str
    expected_semantic_query: str
    expected_filters: dict
    description: str


FILTER_TEST_CASES = [
    FilterTestCase(
        query="Python tutorials for beginners",
        expected_semantic_query="tutorials",
        expected_filters={
            "language": {"$eq": "python"},
            "difficulty": {"$eq": "beginner"},
        },
        description="Two exact-match filters",
    ),
    FilterTestCase(
        query="Advanced guides published after 2023",
        expected_semantic_query="guides",
        expected_filters={
            "difficulty": {"$eq": "advanced"},
            "year": {"$gt": 2023},
        },
        description="String filter + numeric comparison",
    ),
    FilterTestCase(
        query="What is recursion?",
        expected_semantic_query="recursion",
        expected_filters={},
        description="No filters -- pure semantic query",
    ),
    FilterTestCase(
        query="Show me the latest React or Vue tutorials",
        expected_semantic_query="tutorials",
        expected_filters={
            "framework": {"$in": ["react", "vue"]},
        },
        description="In-list filter with OR semantics",
    ),
    FilterTestCase(
        query="Short articles under 2000 words about databases",
        expected_semantic_query="databases",
        expected_filters={
            "word_count": {"$lt": 2000},
        },
        description="Numeric comparison filter",
    ),
]


class FilterCorrectnessEvaluator:
    """Evaluate the accuracy of self-query filter generation.

    Compares generated filters against ground-truth test cases
    to measure parser reliability.
    """

    def __init__(self, parser):
        self.parser = parser

    def evaluate(
        self, test_cases: list[FilterTestCase]
    ) -> dict:
        """Run all test cases and compute metrics."""
        results = []

        for tc in test_cases:
            parsed = self.parser.parse(tc.query)

            # Compare filters
            filter_match = self._compare_filters(
                parsed.filters, tc.expected_filters
            )

            # Compare semantic query (loose match -- word overlap)
            semantic_match = self._compare_semantic(
                parsed.semantic_query, tc.expected_semantic_query
            )

            results.append({
                "query": tc.query,
                "description": tc.description,
                "filter_correct": filter_match["exact_match"],
                "filter_partial": filter_match["partial_match"],
                "semantic_match": semantic_match,
                "parsed_query": parsed.semantic_query,
                "parsed_filters": parsed.filters,
                "expected_filters": tc.expected_filters,
                "filter_details": filter_match,
            })

        # Compute aggregate metrics
        n = len(results)
        return {
            "total_cases": n,
            "filter_exact_accuracy": (
                sum(1 for r in results if r["filter_correct"]) / n
            ),
            "filter_partial_accuracy": (
                sum(1 for r in results if r["filter_partial"]) / n
            ),
            "semantic_accuracy": (
                sum(1 for r in results if r["semantic_match"]) / n
            ),
            "details": results,
        }

    def _compare_filters(
        self, generated: dict, expected: dict
    ) -> dict:
        """Compare generated filters against expected."""
        if not generated and not expected:
            return {
                "exact_match": True,
                "partial_match": True,
                "missing_keys": [],
                "extra_keys": [],
                "wrong_values": [],
            }

        gen_keys = set(generated.keys()) if generated else set()
        exp_keys = set(expected.keys()) if expected else set()

        missing = exp_keys - gen_keys
        extra = gen_keys - exp_keys
        common = gen_keys & exp_keys

        wrong_values = []
        for key in common:
            if generated[key] != expected[key]:
                wrong_values.append({
                    "key": key,
                    "generated": generated[key],
                    "expected": expected[key],
                })

        exact_match = (
            not missing and not extra and not wrong_values
        )
        partial_match = len(common) > 0 and not wrong_values

        return {
            "exact_match": exact_match,
            "partial_match": partial_match,
            "missing_keys": list(missing),
            "extra_keys": list(extra),
            "wrong_values": wrong_values,
        }

    def _compare_semantic(
        self, generated: str, expected: str
    ) -> bool:
        """Compare semantic queries with word overlap."""
        gen_words = set(generated.lower().split())
        exp_words = set(expected.lower().split())
        if not exp_words:
            return True
        overlap = len(gen_words & exp_words) / len(exp_words)
        return overlap >= 0.5
```

---

## Fallback Strategies

### Graceful Degradation When Parser Fails

```python
import logging
import json

logger = logging.getLogger(__name__)


class ResilientSelfQueryRetriever:
    """Self-querying retriever with fallback on parse failure.

    Fallback cascade:
    1. Try self-querying (LLM parser + filtered search)
    2. On parse failure: fall back to standard vector search
    3. On empty results: retry with relaxed filters
    4. On still empty: search without any filters
    """

    def __init__(
        self,
        vector_store,
        parser,
        default_k: int = 10,
    ):
        self.store = vector_store
        self.parser = parser
        self.default_k = default_k

    def invoke(self, query: str, k: int | None = None) -> list:
        k = k or self.default_k

        # Attempt 1: Full self-querying
        try:
            structured = self.parser.parse(query)
            results = self._search_with_filters(
                structured.semantic_query,
                structured.filters,
                k,
            )
            if results:
                logger.info(
                    "Self-query succeeded: %d results with filters %s",
                    len(results),
                    structured.filters,
                )
                return results

            # Attempt 2: Relax filters (remove one at a time)
            results = self._search_with_relaxed_filters(
                structured.semantic_query,
                structured.filters,
                k,
            )
            if results:
                logger.info(
                    "Self-query succeeded with relaxed filters: %d results",
                    len(results),
                )
                return results

        except Exception as e:
            logger.warning(
                "Self-query parser failed: %s, falling back to standard search",
                str(e),
            )

        # Attempt 3: Standard vector search (no filters)
        logger.info("Falling back to standard vector search")
        return self.store.similarity_search(query, k=k)

    def _search_with_filters(
        self, query: str, filters: dict, k: int
    ) -> list:
        """Search with the parsed filters."""
        if not filters:
            return self.store.similarity_search(query, k=k)

        filter_dict = self._build_filter_dict(filters)
        return self.store.similarity_search(
            query, k=k, filter=filter_dict
        )

    def _search_with_relaxed_filters(
        self, query: str, filters: dict, k: int
    ) -> list:
        """Try removing filters one at a time until results are found."""
        filter_keys = list(filters.keys())

        for key_to_remove in filter_keys:
            relaxed = {
                k: v for k, v in filters.items()
                if k != key_to_remove
            }
            if not relaxed:
                continue

            results = self._search_with_filters(query, relaxed, k)
            if results:
                logger.info(
                    "Found results after removing filter: %s",
                    key_to_remove,
                )
                return results

        return []

    def _build_filter_dict(self, filters: dict) -> dict:
        """Convert parsed filters to vector store format."""
        conditions = []
        for field, constraint in filters.items():
            if isinstance(constraint, dict):
                conditions.append({field: constraint})
            else:
                conditions.append({field: {"$eq": constraint}})

        if len(conditions) == 1:
            return conditions[0]
        return {"$and": conditions}
```

### User Feedback on Filter Accuracy

```python
class FilterFeedbackCollector:
    """Collect user feedback on filter accuracy to improve the parser."""

    def __init__(self, storage):
        self.storage = storage

    def record_feedback(
        self,
        query: str,
        parsed_filters: dict,
        user_intended_filters: dict | None,
        was_helpful: bool,
    ) -> None:
        """Record whether the self-query filters were correct."""
        self.storage.append({
            "query": query,
            "parsed_filters": parsed_filters,
            "user_intended_filters": user_intended_filters,
            "was_helpful": was_helpful,
        })

    def compute_accuracy(self) -> dict:
        """Compute filter accuracy from collected feedback."""
        total = len(self.storage)
        if total == 0:
            return {"total": 0, "accuracy": 0}

        correct = sum(
            1 for f in self.storage if f["was_helpful"]
        )
        return {
            "total": total,
            "correct": correct,
            "accuracy": correct / total,
        }
```

---

## Filter Caching

### Avoiding Redundant LLM Parse Calls

```python
import hashlib
import json


class FilterCache:
    """Cache parsed query filters to avoid redundant LLM calls.

    Similar queries often produce the same filters. Caching
    the parse results saves the LLM call latency and cost.
    """

    def __init__(self, redis_client, ttl: int = 3600):
        self.redis = redis_client
        self.ttl = ttl
        self.prefix = "filter_cache:"

    def _cache_key(self, query: str) -> str:
        """Normalize and hash the query for cache lookup."""
        normalized = query.lower().strip()
        query_hash = hashlib.sha256(normalized.encode()).hexdigest()
        return f"{self.prefix}{query_hash}"

    def get(self, query: str) -> dict | None:
        """Look up cached filter parse result."""
        key = self._cache_key(query)
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)
        return None

    def put(self, query: str, parsed: dict) -> None:
        """Store parsed filter result."""
        key = self._cache_key(query)
        self.redis.setex(key, self.ttl, json.dumps(parsed))


class CachedSelfQueryRetriever:
    """Self-querying retriever with filter parse caching."""

    def __init__(self, retriever, cache: FilterCache):
        self.retriever = retriever
        self.cache = cache

    def invoke(self, query: str) -> list:
        # Check cache first
        cached_parse = self.cache.get(query)
        if cached_parse:
            # Use cached filters directly
            return self.retriever._search_with_filters(
                cached_parse["semantic_query"],
                cached_parse["filters"],
                k=10,
            )

        # Parse and cache
        structured = self.retriever.parser.parse(query)
        self.cache.put(query, {
            "semantic_query": structured.semantic_query,
            "filters": structured.filters,
        })

        return self.retriever.invoke(query)
```

---

## Common Pitfalls

1. **Not handling empty results after filtering.** Strict filters can eliminate all results. Always implement a fallback that relaxes or removes filters when zero results are returned.
2. **Caching parsed filters without expiration.** If metadata values change (new categories, renamed attributes), cached filters become invalid. Use TTL-based cache expiration.
3. **Not evaluating filter accuracy systematically.** Without a ground-truth test set, you do not know if the parser is generating correct filters. Build a test set of 50+ queries with expected filters and run it on every parser change.
4. **Combining hybrid search with server-side filters incorrectly.** If the BM25 retriever does not support metadata filtering, you must apply filters after merging. This means over-retrieving and filtering in application code.
5. **Not logging generated filters in production.** Without logs, you cannot debug why a specific query returned unexpected results. Log every parsed query and its generated filters.
6. **Ignoring filter generation latency.** The LLM parse call adds 200-500ms. For latency-sensitive applications, use filter caching or pre-compute filters for common query patterns.

---

## References

- LangChain SelfQueryRetriever: https://python.langchain.com/docs/how_to/self_query/
- Reciprocal Rank Fusion: Cormack et al. "Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods" (SIGIR 2009)
- Hybrid search patterns: https://weaviate.io/developers/weaviate/search/hybrid
