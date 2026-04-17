# Time-Aware Retrieval -- Temporal Awareness in RAG

## TL;DR

Standard vector search treats all documents as equally relevant regardless of when they were created. This fails for time-sensitive queries: "What is our current pricing?" should prefer recent pricing docs over outdated ones, "What happened last quarter?" must filter to a specific date range, and "What is the latest update on the migration?" should rank recent documents higher than old ones. Time-aware retrieval adds temporal intelligence to RAG by combining vector similarity with recency signals: exponential decay weighting (newer documents score higher), time-range filtering (restrict results to specific periods), and temporal query understanding (extracting date references from natural language). This overview covers the temporal retrieval taxonomy, when recency matters, and integration patterns. Subsequent articles cover recency-weighting algorithms and implementation with metadata timestamps.

---

## The Temporal Problem in RAG

### Time-Insensitive Retrieval Failures

| Query | Correct Answer | Standard RAG Answer | Problem |
|-------|---------------|--------------------| --------|
| "Current pricing plans" | 2024 pricing page | 2022 pricing (higher similarity to "plans") | Outdated info |
| "What changed in v3.0?" | v3.0 changelog | v2.0 changelog (similar keywords) | Wrong version |
| "Recent security incidents" | Last month's incident | Major incident from 2 years ago | Not recent |
| "Q3 2024 results" | Q3 2024 report | Q3 2023 report (same keywords) | Wrong period |

### When Recency Matters

```
Always prefer recent:
  - Product documentation (features change)
  - Pricing and policies (updated frequently)
  - Security advisories (latest matters most)
  - API documentation (endpoints change)

Recency does not matter:
  - Foundational concepts ("What is TCP/IP?")
  - Historical analysis ("What was our first product?")
  - Legal documents (specific version matters, not recency)
  - Research papers (quality over recency)
```

---

## Temporal Retrieval Taxonomy

### Three Approaches

```python
from enum import Enum


class TemporalStrategy(Enum):
    RECENCY_WEIGHTED = "recency_weighted"  # Blend similarity + recency
    TIME_FILTERED = "time_filtered"        # Hard filter on date range
    HYBRID = "hybrid"                      # Filter first, then weight
```

### Approach 1: Recency-Weighted Scoring

```
final_score = similarity_score * (1 - alpha) + recency_score * alpha

Where:
  similarity_score = cosine similarity (0 to 1)
  recency_score = exponential_decay(document_age)
  alpha = recency weight (0.0 = pure similarity, 1.0 = pure recency)
```

### Approach 2: Time-Range Filtering

```
User: "What were Q3 2024 results?"
Filter: document_date >= 2024-07-01 AND document_date <= 2024-09-30
Then: standard similarity search within the filtered set
```

### Approach 3: Hybrid (Recommended)

```
1. Extract temporal references from query
2. If explicit time range -> apply hard filter
3. If implicit recency ("recent", "latest", "current") -> apply recency weighting
4. If no temporal reference -> standard similarity
```

---

## Temporal Query Understanding

### Extracting Time References from Queries

```python
import re
from datetime import datetime, timedelta
from dataclasses import dataclass


@dataclass
class TemporalReference:
    """Extracted temporal information from a query."""

    has_temporal: bool
    query_type: str  # "absolute", "relative", "recency", "none"
    start_date: datetime | None = None
    end_date: datetime | None = None
    recency_bias: float = 0.0  # 0 to 1


class TemporalQueryParser:
    """Parse temporal references from natural language queries."""

    RECENCY_KEYWORDS = [
        "latest", "recent", "newest", "current", "today",
        "now", "updated", "last", "this week", "this month",
    ]

    RELATIVE_PATTERNS = {
        r"last\s+(\d+)\s+days?": lambda m: timedelta(days=int(m.group(1))),
        r"last\s+(\d+)\s+weeks?": lambda m: timedelta(weeks=int(m.group(1))),
        r"last\s+(\d+)\s+months?": lambda m: timedelta(days=int(m.group(1)) * 30),
        r"last\s+week": lambda m: timedelta(weeks=1),
        r"last\s+month": lambda m: timedelta(days=30),
        r"last\s+quarter": lambda m: timedelta(days=90),
        r"last\s+year": lambda m: timedelta(days=365),
        r"past\s+(\d+)\s+days?": lambda m: timedelta(days=int(m.group(1))),
        r"yesterday": lambda m: timedelta(days=1),
    }

    QUARTER_PATTERN = re.compile(
        r"Q([1-4])\s+(\d{4})"
    )

    def parse(self, query: str) -> TemporalReference:
        """Extract temporal references from a query."""
        now = datetime.utcnow()
        query_lower = query.lower()

        # Check for explicit quarter references
        quarter_match = self.QUARTER_PATTERN.search(query)
        if quarter_match:
            q = int(quarter_match.group(1))
            year = int(quarter_match.group(2))
            start_month = (q - 1) * 3 + 1
            start = datetime(year, start_month, 1)
            end_month = start_month + 2
            if end_month == 12:
                end = datetime(year, 12, 31)
            else:
                end = datetime(year, end_month + 1, 1) - timedelta(days=1)

            return TemporalReference(
                has_temporal=True,
                query_type="absolute",
                start_date=start,
                end_date=end,
            )

        # Check for relative time patterns
        for pattern, delta_fn in self.RELATIVE_PATTERNS.items():
            match = re.search(pattern, query_lower)
            if match:
                delta = delta_fn(match)
                return TemporalReference(
                    has_temporal=True,
                    query_type="relative",
                    start_date=now - delta,
                    end_date=now,
                )

        # Check for recency keywords
        for keyword in self.RECENCY_KEYWORDS:
            if keyword in query_lower:
                return TemporalReference(
                    has_temporal=True,
                    query_type="recency",
                    recency_bias=0.3,  # Default recency weight
                )

        return TemporalReference(
            has_temporal=False,
            query_type="none",
        )
```

---

## Time-Aware Retriever

```python
import math
import time
from datetime import datetime


class TimeAwareRetriever:
    """Retriever that incorporates temporal awareness.

    Combines vector similarity with recency scoring and
    temporal filtering based on query analysis.
    """

    def __init__(
        self,
        vector_store,
        temporal_parser: TemporalQueryParser,
        default_recency_weight: float = 0.2,
        decay_rate: float = 0.01,
    ):
        self.store = vector_store
        self.parser = temporal_parser
        self.default_recency_weight = default_recency_weight
        self.decay_rate = decay_rate

    def invoke(self, query: str, k: int = 10) -> list:
        """Retrieve with temporal awareness."""
        temporal = self.parser.parse(query)

        if temporal.query_type == "absolute":
            # Hard filter on date range
            return self._filtered_search(
                query, temporal.start_date, temporal.end_date, k
            )
        elif temporal.query_type == "relative":
            return self._filtered_search(
                query, temporal.start_date, temporal.end_date, k
            )
        elif temporal.query_type == "recency":
            return self._recency_weighted_search(
                query, temporal.recency_bias, k
            )
        else:
            # No temporal reference -- standard search
            return self.store.similarity_search(query, k=k)

    def _filtered_search(
        self,
        query: str,
        start_date: datetime,
        end_date: datetime,
        k: int,
    ) -> list:
        """Search within a specific time range."""
        time_filter = {
            "$and": [
                {"created_at": {"$gte": start_date.isoformat()}},
                {"created_at": {"$lte": end_date.isoformat()}},
            ]
        }
        results = self.store.similarity_search(
            query, k=k, filter=time_filter
        )

        # If no results in range, fall back to recency-weighted
        if not results:
            return self._recency_weighted_search(query, 0.3, k)

        return results

    def _recency_weighted_search(
        self,
        query: str,
        recency_weight: float,
        k: int,
    ) -> list:
        """Search with recency weighting on similarity scores."""
        # Over-retrieve to allow re-ranking
        results_with_scores = self.store.similarity_search_with_score(
            query, k=k * 3
        )

        now = time.time()
        scored = []

        for doc, sim_score in results_with_scores:
            # Get document timestamp
            created_at = doc.metadata.get("created_at")
            if created_at:
                if isinstance(created_at, str):
                    doc_time = datetime.fromisoformat(
                        created_at
                    ).timestamp()
                else:
                    doc_time = float(created_at)
                age_days = (now - doc_time) / 86400
            else:
                age_days = 365  # Default: assume 1 year old

            # Exponential decay recency score
            recency_score = math.exp(-self.decay_rate * age_days)

            # Combined score
            final_score = (
                sim_score * (1 - recency_weight)
                + recency_score * recency_weight
            )

            doc.metadata["_recency_score"] = recency_score
            doc.metadata["_final_score"] = final_score
            doc.metadata["_age_days"] = age_days
            scored.append((doc, final_score))

        # Sort by combined score (descending)
        scored.sort(key=lambda x: x[1], reverse=True)

        return [doc for doc, _ in scored[:k]]
```

---

## Common Pitfalls

1. **Applying recency bias to all queries.** Questions about foundational concepts ("What is REST?") do not benefit from recency. Only apply recency weighting when the query has temporal signals.
2. **Not storing timestamps in document metadata.** Without a `created_at` or `updated_at` field in document metadata, time-aware retrieval is impossible. Always include timestamps during ingestion.
3. **Using a single decay rate for all content types.** API docs expire faster than architectural guides. Use content-type-specific decay rates: fast decay for changelogs, slow decay for tutorials.
4. **Hard filtering with no fallback.** If "Q3 2024 results" returns zero results (no documents from that period), the user gets nothing. Always fall back to recency-weighted search when hard filters return empty results.
5. **Not normalizing timestamp formats.** If some documents have Unix timestamps and others have ISO strings, comparisons fail. Normalize all timestamps to ISO 8601 during ingestion.
6. **Ignoring query ambiguity.** "Last quarter" is ambiguous -- it depends on when the query is made. Always resolve relative references against the current date at query time.

---

## References

- LangChain TimeWeightedVectorStoreRetriever: https://python.langchain.com/docs/how_to/time_weighted_vectorstore/
- Park et al. "Generative Agents: Interactive Simulacra of Human Behavior" (2023) -- introduced time-weighted retrieval
- Elasticsearch time-based queries: https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-range-query.html
