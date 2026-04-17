# Time-Aware Retrieval Implementation -- Metadata Timestamps, Score Blending, and Event-Based Retrieval

## TL;DR

This article covers the implementation details of time-aware retrieval: storing temporal metadata during document ingestion (timestamps, version numbers, event dates), building a production retrieval pipeline that blends similarity and recency scores, implementing event-based retrieval for incident logs and changelogs where temporal ordering is essential, and testing temporal retrieval quality with time-specific evaluation datasets.

---

## Temporal Metadata in Document Ingestion

### Enriching Documents with Timestamps

```python
from datetime import datetime
from dataclasses import dataclass, field
import re


@dataclass
class TemporalMetadata:
    """Temporal metadata extracted from and attached to a document."""

    created_at: str          # ISO 8601 timestamp
    updated_at: str          # ISO 8601 timestamp
    content_date: str | None = None  # Date the content refers to
    version: str | None = None
    event_date: str | None = None    # For incident/changelog entries
    expiry_date: str | None = None   # When the content becomes stale
    content_type: str = "default"


class TemporalMetadataExtractor:
    """Extract temporal metadata from document content and source metadata.

    Enriches documents with timestamps for time-aware retrieval.
    """

    DATE_PATTERNS = [
        # ISO date: 2024-01-15
        (re.compile(r"\b(\d{4})-(\d{2})-(\d{2})\b"), "iso"),
        # Date header: Date: January 15, 2024
        (re.compile(
            r"(?:date|published|updated|created):\s*"
            r"(\w+\s+\d{1,2},?\s+\d{4})",
            re.IGNORECASE,
        ), "header"),
        # Version: v3.2.1, Version 3.2
        (re.compile(r"(?:v|version\s*)(\d+\.\d+(?:\.\d+)?)", re.IGNORECASE), "version"),
    ]

    def extract(
        self,
        content: str,
        source_metadata: dict | None = None,
    ) -> TemporalMetadata:
        """Extract temporal metadata from content and source."""
        now = datetime.utcnow().isoformat()
        meta = source_metadata or {}

        # Use source metadata if available
        created_at = meta.get("created_at") or meta.get("date") or now
        updated_at = meta.get("updated_at") or meta.get("modified") or created_at

        # Try to extract content date from the text
        content_date = self._extract_content_date(content)

        # Extract version
        version = self._extract_version(content)

        # Determine content type
        content_type = meta.get("type", self._infer_content_type(content))

        return TemporalMetadata(
            created_at=str(created_at),
            updated_at=str(updated_at),
            content_date=content_date,
            version=version,
            content_type=content_type,
        )

    def _extract_content_date(self, content: str) -> str | None:
        """Extract the most prominent date from content."""
        for pattern, fmt in self.DATE_PATTERNS:
            match = pattern.search(content[:500])  # Check first 500 chars
            if match and fmt == "iso":
                return f"{match.group(1)}-{match.group(2)}-{match.group(3)}"
        return None

    def _extract_version(self, content: str) -> str | None:
        for pattern, fmt in self.DATE_PATTERNS:
            if fmt == "version":
                match = pattern.search(content[:500])
                if match:
                    return match.group(1)
        return None

    def _infer_content_type(self, content: str) -> str:
        content_lower = content[:200].lower()
        if "changelog" in content_lower or "release notes" in content_lower:
            return "changelog"
        elif "api" in content_lower and "endpoint" in content_lower:
            return "api_docs"
        elif "tutorial" in content_lower or "getting started" in content_lower:
            return "tutorial"
        elif "incident" in content_lower or "postmortem" in content_lower:
            return "incident"
        return "default"


class TemporalIndexer:
    """Index documents with temporal metadata for time-aware retrieval."""

    def __init__(self, vector_store, embedding_fn):
        self.store = vector_store
        self.embed = embedding_fn
        self.extractor = TemporalMetadataExtractor()

    def index_documents(
        self, documents: list[dict]
    ) -> dict:
        """Index documents with temporal metadata.

        document format: {"content": str, "metadata": dict}
        """
        texts = []
        metadatas = []

        for doc in documents:
            temporal = self.extractor.extract(
                doc["content"], doc.get("metadata")
            )

            meta = doc.get("metadata", {}).copy()
            meta.update({
                "created_at": temporal.created_at,
                "updated_at": temporal.updated_at,
                "content_date": temporal.content_date,
                "version": temporal.version,
                "content_type": temporal.content_type,
            })

            texts.append(doc["content"])
            metadatas.append(meta)

        self.store.add_texts(texts, metadatas=metadatas)

        return {
            "indexed": len(texts),
            "content_types": {
                m.get("content_type", "default")
                for m in metadatas
            },
        }
```

---

## Production Time-Aware Retriever

### Full Implementation

```python
import math
import time
import logging
from datetime import datetime, timedelta

logger = logging.getLogger(__name__)


class ProductionTimeAwareRetriever:
    """Production-grade time-aware retriever with:
    - Temporal query parsing
    - Content-type-aware decay
    - Score blending
    - Fallback on empty results
    - Metrics logging
    """

    DECAY_HALF_LIVES = {
        "changelog": 30,
        "release_notes": 30,
        "api_docs": 90,
        "incident": 60,
        "tutorial": 180,
        "guide": 180,
        "reference": 365,
        "default": 120,
    }

    def __init__(
        self,
        vector_store,
        temporal_parser,
        default_recency_weight: float = 0.2,
        over_retrieve_factor: int = 3,
    ):
        self.store = vector_store
        self.parser = temporal_parser
        self.recency_weight = default_recency_weight
        self.over_retrieve = over_retrieve_factor

    def invoke(
        self, query: str, k: int = 10, **kwargs
    ) -> list:
        """Time-aware retrieval."""
        temporal = self.parser.parse(query)

        # Strategy selection
        if temporal.query_type == "absolute":
            results = self._absolute_search(
                query, temporal, k
            )
        elif temporal.query_type == "relative":
            results = self._relative_search(
                query, temporal, k
            )
        elif temporal.query_type == "recency":
            results = self._recency_search(
                query, temporal.recency_bias or self.recency_weight, k
            )
        else:
            results = self.store.similarity_search(query, k=k)

        logger.info(
            "Time-aware retrieval: query_type=%s, results=%d, k=%d",
            temporal.query_type,
            len(results),
            k,
        )

        return results

    def _absolute_search(
        self, query: str, temporal, k: int
    ) -> list:
        """Search within an absolute date range."""
        time_filter = {
            "$and": [
                {"created_at": {"$gte": temporal.start_date.isoformat()}},
                {"created_at": {"$lte": temporal.end_date.isoformat()}},
            ]
        }

        try:
            results = self.store.similarity_search(
                query, k=k, filter=time_filter
            )
        except Exception as e:
            logger.warning("Filtered search failed: %s", str(e))
            results = []

        if not results:
            logger.info("No results in date range, falling back to recency")
            return self._recency_search(query, 0.4, k)

        return results

    def _relative_search(
        self, query: str, temporal, k: int
    ) -> list:
        """Search with relative time filter and recency weighting."""
        time_filter = {
            "created_at": {"$gte": temporal.start_date.isoformat()}
        }

        try:
            results_with_scores = self.store.similarity_search_with_score(
                query, k=k * self.over_retrieve, filter=time_filter
            )
        except Exception:
            results_with_scores = self.store.similarity_search_with_score(
                query, k=k * self.over_retrieve
            )

        return self._apply_recency_reranking(
            results_with_scores, 0.3, k
        )

    def _recency_search(
        self, query: str, recency_weight: float, k: int
    ) -> list:
        """Search with recency-weighted scoring."""
        results_with_scores = self.store.similarity_search_with_score(
            query, k=k * self.over_retrieve
        )
        return self._apply_recency_reranking(
            results_with_scores, recency_weight, k
        )

    def _apply_recency_reranking(
        self,
        results_with_scores: list[tuple],
        recency_weight: float,
        k: int,
    ) -> list:
        """Rerank results by blending similarity and recency."""
        now = time.time()
        scored = []

        for doc, sim_score in results_with_scores:
            # Get timestamp
            ts = self._get_timestamp(doc.metadata)
            age_days = (now - ts) / 86400

            # Content-type-aware decay
            content_type = doc.metadata.get("content_type", "default")
            half_life = self.DECAY_HALF_LIVES.get(
                content_type, self.DECAY_HALF_LIVES["default"]
            )
            decay_rate = math.log(2) / half_life
            recency_score = math.exp(-decay_rate * age_days)

            # Blend scores
            final_score = (
                sim_score * (1 - recency_weight)
                + recency_score * recency_weight
            )

            doc.metadata["_recency_score"] = round(recency_score, 4)
            doc.metadata["_final_score"] = round(final_score, 4)
            doc.metadata["_age_days"] = round(age_days, 1)

            scored.append((doc, final_score))

        scored.sort(key=lambda x: x[1], reverse=True)
        return [doc for doc, _ in scored[:k]]

    def _get_timestamp(self, metadata: dict) -> float:
        """Extract timestamp from metadata, handling various formats."""
        for key in ("updated_at", "created_at", "date", "timestamp"):
            value = metadata.get(key)
            if value is None:
                continue
            if isinstance(value, (int, float)):
                return float(value)
            if isinstance(value, str):
                try:
                    dt = datetime.fromisoformat(
                        value.replace("Z", "+00:00")
                    )
                    return dt.timestamp()
                except ValueError:
                    continue

        # Default: assume 6 months ago
        return time.time() - 180 * 86400
```

---

## Event-Based Retrieval

### For Incident Logs and Changelogs

```python
class EventBasedRetriever:
    """Specialized retriever for time-ordered events.

    For incident logs, changelogs, and audit trails where
    temporal ordering is the primary organization principle.
    """

    def __init__(self, vector_store):
        self.store = vector_store

    def get_latest_events(
        self, query: str, k: int = 5, days_back: int = 30
    ) -> list:
        """Retrieve the most recent events matching a query."""
        cutoff = datetime.utcnow() - timedelta(days=days_back)

        results = self.store.similarity_search(
            query,
            k=k * 3,
            filter={
                "$and": [
                    {"content_type": {"$in": ["incident", "changelog", "event"]}},
                    {"created_at": {"$gte": cutoff.isoformat()}},
                ]
            },
        )

        # Sort by date (most recent first)
        results.sort(
            key=lambda d: d.metadata.get("created_at", ""),
            reverse=True,
        )

        return results[:k]

    def get_events_in_range(
        self,
        query: str,
        start: datetime,
        end: datetime,
        k: int = 20,
    ) -> list:
        """Retrieve events within a specific date range."""
        results = self.store.similarity_search(
            query,
            k=k,
            filter={
                "$and": [
                    {"event_date": {"$gte": start.isoformat()}},
                    {"event_date": {"$lte": end.isoformat()}},
                ]
            },
        )

        results.sort(
            key=lambda d: d.metadata.get("event_date", ""),
        )

        return results

    def get_event_timeline(
        self, topic: str, k: int = 10
    ) -> list:
        """Build a timeline of events related to a topic.

        Retrieves semantically relevant events sorted chronologically.
        """
        all_events = self.store.similarity_search(
            topic, k=k * 2
        )

        # Filter to event types only
        events = [
            d for d in all_events
            if d.metadata.get("content_type") in (
                "incident", "changelog", "event", "release_notes"
            )
        ]

        # Sort chronologically
        events.sort(
            key=lambda d: d.metadata.get(
                "event_date",
                d.metadata.get("created_at", ""),
            ),
        )

        return events[:k]
```

---

## Testing Time-Aware Retrieval

### Temporal Evaluation Dataset

```python
from dataclasses import dataclass


@dataclass
class TemporalTestCase:
    query: str
    expected_recency: str  # "newest_first", "date_range", "any"
    expected_date_range: tuple | None = None
    expected_content_types: list[str] | None = None
    description: str = ""


TEMPORAL_TEST_CASES = [
    TemporalTestCase(
        query="What are the current pricing plans?",
        expected_recency="newest_first",
        description="Should prefer most recent pricing docs",
    ),
    TemporalTestCase(
        query="What changed in Q3 2024?",
        expected_recency="date_range",
        expected_date_range=("2024-07-01", "2024-09-30"),
        expected_content_types=["changelog", "release_notes"],
        description="Should filter to Q3 2024",
    ),
    TemporalTestCase(
        query="What is REST?",
        expected_recency="any",
        description="No temporal preference -- pure semantic relevance",
    ),
    TemporalTestCase(
        query="Recent security incidents",
        expected_recency="newest_first",
        expected_content_types=["incident"],
        description="Should prefer recent incidents",
    ),
]


class TemporalRetrievalEvaluator:
    """Evaluate whether temporal retrieval returns results
    with appropriate temporal characteristics."""

    def evaluate(
        self,
        retriever,
        test_cases: list[TemporalTestCase],
    ) -> dict:
        results = []

        for tc in test_cases:
            docs = retriever.invoke(tc.query, k=5)

            # Check temporal ordering
            dates = [
                d.metadata.get("created_at", "")
                for d in docs
                if d.metadata.get("created_at")
            ]

            if tc.expected_recency == "newest_first":
                is_ordered = dates == sorted(dates, reverse=True)
                results.append({
                    "query": tc.query,
                    "passed": is_ordered,
                    "check": "newest_first",
                    "dates": dates[:3],
                })

            elif tc.expected_recency == "date_range" and tc.expected_date_range:
                start, end = tc.expected_date_range
                in_range = all(
                    start <= d[:10] <= end
                    for d in dates
                    if d
                )
                results.append({
                    "query": tc.query,
                    "passed": in_range or len(dates) == 0,
                    "check": "date_range",
                    "dates": dates[:3],
                    "expected_range": tc.expected_date_range,
                })

            else:
                results.append({
                    "query": tc.query,
                    "passed": len(docs) > 0,
                    "check": "has_results",
                })

        passed = sum(1 for r in results if r["passed"])
        return {
            "total": len(results),
            "passed": passed,
            "accuracy": passed / len(results) if results else 0,
            "details": results,
        }
```

---

## Common Pitfalls

1. **Storing timestamps as strings without a consistent format.** "Jan 15 2024", "2024-01-15", and "1705276800" in the same index break filtering. Normalize all timestamps to ISO 8601 during ingestion.
2. **Not updating the `updated_at` field when documents change.** If a document is edited but the timestamp stays the same, the recency scorer treats it as old. Always update `updated_at` on every document modification.
3. **Applying the same recency weight to all queries.** Use the temporal query parser to detect whether recency matters and adjust the weight dynamically.
4. **Not testing with temporal-specific evaluation datasets.** Standard RAG evaluation datasets (like RAGAS) do not test temporal relevance. Build a custom dataset with time-sensitive queries and expected temporal ordering.
5. **Over-retrieving without a bound.** Setting `over_retrieve_factor` too high (e.g., 10x) retrieves hundreds of documents for re-ranking, adding latency. Keep it at 2-3x for production.
6. **Ignoring timezone differences.** If documents are indexed with UTC timestamps but queries use local time ("yesterday"), the date extraction must account for the user's timezone.

---

## References

- LangChain TimeWeightedVectorStoreRetriever: https://python.langchain.com/docs/how_to/time_weighted_vectorstore/
- Elasticsearch range queries: https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-range-query.html
- Python dateutil: https://dateutil.readthedocs.io/
