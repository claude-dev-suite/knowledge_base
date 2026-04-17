# Recency Weighting -- Exponential Decay, Time-Range Filters, and Date Extraction from Queries

## TL;DR

Recency weighting modifies retrieval scoring so that newer documents are preferred over older ones, all else being equal. The most common approach is exponential decay: a document's recency score decreases exponentially with age, and this score is blended with the vector similarity score. This article covers decay function design (exponential, linear, step), parameter tuning (decay rate, time unit, blending weight), time-range filters for absolute temporal queries, and robust date extraction from natural language queries using both rule-based and LLM-based parsers.

---

## Decay Functions

### Exponential Decay

```python
import math
import time
from datetime import datetime
from dataclasses import dataclass


@dataclass
class DecayConfig:
    """Configuration for recency decay."""

    decay_type: str = "exponential"  # "exponential", "linear", "step"
    decay_rate: float = 0.01         # For exponential: controls speed of decay
    half_life_days: float | None = None  # Alternative: time for score to reach 0.5
    time_unit: str = "days"          # "hours", "days", "weeks"


class RecencyScorer:
    """Compute recency scores using configurable decay functions.

    The recency score is a value from 0 (infinitely old) to 1 (just created)
    that represents how "fresh" a document is.
    """

    def __init__(self, config: DecayConfig):
        self.config = config
        if config.half_life_days:
            # Convert half-life to decay rate
            # 0.5 = exp(-rate * half_life)
            # rate = ln(2) / half_life
            self.decay_rate = math.log(2) / config.half_life_days
        else:
            self.decay_rate = config.decay_rate

    def score(
        self, document_timestamp: float, now: float | None = None
    ) -> float:
        """Compute recency score for a document."""
        now = now or time.time()
        age_seconds = max(0, now - document_timestamp)

        # Convert to configured time unit
        if self.config.time_unit == "hours":
            age = age_seconds / 3600
        elif self.config.time_unit == "weeks":
            age = age_seconds / (86400 * 7)
        else:  # days
            age = age_seconds / 86400

        if self.config.decay_type == "exponential":
            return math.exp(-self.decay_rate * age)

        elif self.config.decay_type == "linear":
            # Linear decay from 1 to 0 over max_age
            max_age = 1.0 / self.decay_rate  # Age where score reaches 0
            return max(0.0, 1.0 - (age / max_age))

        elif self.config.decay_type == "step":
            # Binary: 1.0 if within threshold, 0.0 otherwise
            threshold = 1.0 / self.decay_rate
            return 1.0 if age <= threshold else 0.0

        return 0.0

    def score_from_iso(
        self, iso_timestamp: str, now: float | None = None
    ) -> float:
        """Compute recency score from an ISO 8601 timestamp string."""
        dt = datetime.fromisoformat(iso_timestamp.replace("Z", "+00:00"))
        return self.score(dt.timestamp(), now)
```

### Choosing Decay Parameters

| Content Type | Recommended Half-Life | Decay Rate | Rationale |
|-------------|----------------------|------------|-----------|
| Changelogs / release notes | 30 days | 0.023 | Only latest release matters |
| API documentation | 90 days | 0.0077 | APIs change quarterly |
| Tutorials / guides | 180 days | 0.0039 | Content stays relevant longer |
| Architecture docs | 365 days | 0.0019 | Fundamental docs age slowly |
| Legal / compliance | N/A (no decay) | 0.0 | Specific versions required |

### Content-Type-Aware Decay

```python
class ContentTypeDecayScorer:
    """Apply different decay rates based on document content type.

    Different document types age at different rates.
    A changelog from last month is old; a tutorial from last month
    is still fresh.
    """

    DECAY_CONFIGS = {
        "changelog": DecayConfig(half_life_days=30),
        "release_notes": DecayConfig(half_life_days=30),
        "api_docs": DecayConfig(half_life_days=90),
        "tutorial": DecayConfig(half_life_days=180),
        "guide": DecayConfig(half_life_days=180),
        "architecture": DecayConfig(half_life_days=365),
        "reference": DecayConfig(half_life_days=365),
        "default": DecayConfig(half_life_days=120),
    }

    def __init__(self):
        self.scorers = {
            content_type: RecencyScorer(config)
            for content_type, config in self.DECAY_CONFIGS.items()
        }

    def score(
        self,
        document_timestamp: float,
        content_type: str,
        now: float | None = None,
    ) -> float:
        """Score recency based on content type."""
        scorer = self.scorers.get(
            content_type,
            self.scorers["default"],
        )
        return scorer.score(document_timestamp, now)
```

---

## Score Blending

### Combining Similarity and Recency

```python
import numpy as np


class BlendedScorer:
    """Blend vector similarity and recency scores.

    final_score = similarity^p * recency^q

    Where p and q control the relative importance.
    Using multiplication (instead of weighted sum) ensures that
    a document with zero recency is always ranked lower, regardless
    of similarity.
    """

    def __init__(
        self,
        recency_scorer: RecencyScorer,
        similarity_weight: float = 0.7,
        recency_weight: float = 0.3,
        blend_mode: str = "weighted_sum",
    ):
        self.recency_scorer = recency_scorer
        self.sim_weight = similarity_weight
        self.rec_weight = recency_weight
        self.blend_mode = blend_mode

    def score(
        self,
        similarity: float,
        document_timestamp: float,
        content_type: str = "default",
    ) -> float:
        """Compute blended score."""
        recency = self.recency_scorer.score(document_timestamp)

        if self.blend_mode == "weighted_sum":
            return (
                similarity * self.sim_weight
                + recency * self.rec_weight
            )

        elif self.blend_mode == "multiplicative":
            return similarity * recency

        elif self.blend_mode == "geometric_mean":
            return (
                similarity ** self.sim_weight
                * recency ** self.rec_weight
            )

        return similarity

    def rerank(
        self,
        results: list[tuple],  # (document, similarity_score)
        k: int = 10,
    ) -> list:
        """Rerank retrieval results with recency blending."""
        scored = []
        for doc, sim_score in results:
            timestamp = doc.metadata.get("created_at")
            if isinstance(timestamp, str):
                ts = datetime.fromisoformat(timestamp).timestamp()
            elif isinstance(timestamp, (int, float)):
                ts = float(timestamp)
            else:
                ts = time.time() - 365 * 86400  # Default: 1 year ago

            final = self.score(sim_score, ts)
            doc.metadata["_blended_score"] = final
            scored.append((doc, final))

        scored.sort(key=lambda x: x[1], reverse=True)
        return [doc for doc, _ in scored[:k]]
```

---

## Time-Range Filters

### Hard Temporal Filtering

```python
class TemporalFilter:
    """Build vector store filters for time-range queries."""

    @staticmethod
    def build_range_filter(
        start: datetime | None,
        end: datetime | None,
        timestamp_field: str = "created_at",
    ) -> dict | None:
        """Build a metadata filter for a time range."""
        conditions = []

        if start:
            conditions.append({
                timestamp_field: {"$gte": start.isoformat()}
            })
        if end:
            conditions.append({
                timestamp_field: {"$lte": end.isoformat()}
            })

        if not conditions:
            return None
        if len(conditions) == 1:
            return conditions[0]
        return {"$and": conditions}

    @staticmethod
    def build_quarter_filter(
        quarter: int,
        year: int,
        timestamp_field: str = "created_at",
    ) -> dict:
        """Build filter for a specific calendar quarter."""
        from datetime import date

        start_month = (quarter - 1) * 3 + 1
        start = datetime(year, start_month, 1)

        if quarter == 4:
            end = datetime(year, 12, 31, 23, 59, 59)
        else:
            end_month = start_month + 3
            end = datetime(year, end_month, 1) - timedelta(seconds=1)

        return {
            "$and": [
                {timestamp_field: {"$gte": start.isoformat()}},
                {timestamp_field: {"$lte": end.isoformat()}},
            ],
        }

    @staticmethod
    def build_relative_filter(
        lookback_days: int,
        timestamp_field: str = "created_at",
    ) -> dict:
        """Build filter for 'last N days'."""
        cutoff = datetime.utcnow() - timedelta(days=lookback_days)
        return {
            timestamp_field: {"$gte": cutoff.isoformat()}
        }
```

---

## Date Extraction from Queries

### Rule-Based Date Extraction

```python
import re
from datetime import datetime, timedelta


class RuleDateExtractor:
    """Extract date references from natural language queries
    using regex patterns."""

    PATTERNS = [
        # Explicit dates: "January 2024", "March 15, 2024"
        (
            r"(january|february|march|april|may|june|july|august|"
            r"september|october|november|december)\s+(\d{4})",
            "_extract_month_year",
        ),
        # ISO dates: 2024-01-15
        (r"(\d{4})-(\d{2})-(\d{2})", "_extract_iso_date"),
        # Quarters: Q1 2024, Q3 2023
        (r"Q([1-4])\s+(\d{4})", "_extract_quarter"),
        # Relative: last N days/weeks/months
        (r"last\s+(\d+)\s+(days?|weeks?|months?)", "_extract_relative"),
        # Named relative: yesterday, last week, last month
        (r"\byesterday\b", "_extract_yesterday"),
        (r"\blast\s+week\b", "_extract_last_week"),
        (r"\blast\s+month\b", "_extract_last_month"),
        (r"\blast\s+quarter\b", "_extract_last_quarter"),
        (r"\blast\s+year\b", "_extract_last_year"),
        (r"\bthis\s+year\b", "_extract_this_year"),
        (r"\bthis\s+month\b", "_extract_this_month"),
    ]

    MONTH_MAP = {
        "january": 1, "february": 2, "march": 3, "april": 4,
        "may": 5, "june": 6, "july": 7, "august": 8,
        "september": 9, "october": 10, "november": 11, "december": 12,
    }

    def extract(self, query: str) -> TemporalReference:
        """Extract temporal reference from query."""
        query_lower = query.lower()

        for pattern, method_name in self.PATTERNS:
            match = re.search(pattern, query_lower)
            if match:
                method = getattr(self, method_name)
                return method(match)

        return TemporalReference(has_temporal=False, query_type="none")

    def _extract_month_year(self, match) -> TemporalReference:
        month = self.MONTH_MAP[match.group(1)]
        year = int(match.group(2))
        start = datetime(year, month, 1)
        if month == 12:
            end = datetime(year, 12, 31)
        else:
            end = datetime(year, month + 1, 1) - timedelta(days=1)

        return TemporalReference(
            has_temporal=True, query_type="absolute",
            start_date=start, end_date=end,
        )

    def _extract_quarter(self, match) -> TemporalReference:
        q = int(match.group(1))
        year = int(match.group(2))
        start_month = (q - 1) * 3 + 1
        start = datetime(year, start_month, 1)
        end_month = start_month + 2
        if end_month == 12:
            end = datetime(year, 12, 31)
        else:
            end = datetime(year, end_month + 1, 1) - timedelta(days=1)

        return TemporalReference(
            has_temporal=True, query_type="absolute",
            start_date=start, end_date=end,
        )

    def _extract_relative(self, match) -> TemporalReference:
        amount = int(match.group(1))
        unit = match.group(2).rstrip("s")
        now = datetime.utcnow()

        if unit == "day":
            delta = timedelta(days=amount)
        elif unit == "week":
            delta = timedelta(weeks=amount)
        elif unit == "month":
            delta = timedelta(days=amount * 30)
        else:
            delta = timedelta(days=amount)

        return TemporalReference(
            has_temporal=True, query_type="relative",
            start_date=now - delta, end_date=now,
        )

    def _extract_iso_date(self, match) -> TemporalReference:
        dt = datetime(
            int(match.group(1)), int(match.group(2)), int(match.group(3))
        )
        return TemporalReference(
            has_temporal=True, query_type="absolute",
            start_date=dt, end_date=dt + timedelta(days=1),
        )

    def _extract_yesterday(self, match) -> TemporalReference:
        yesterday = datetime.utcnow() - timedelta(days=1)
        return TemporalReference(
            has_temporal=True, query_type="relative",
            start_date=yesterday.replace(hour=0, minute=0),
            end_date=yesterday.replace(hour=23, minute=59),
        )

    def _extract_last_week(self, match) -> TemporalReference:
        now = datetime.utcnow()
        return TemporalReference(
            has_temporal=True, query_type="relative",
            start_date=now - timedelta(weeks=1), end_date=now,
        )

    def _extract_last_month(self, match) -> TemporalReference:
        now = datetime.utcnow()
        return TemporalReference(
            has_temporal=True, query_type="relative",
            start_date=now - timedelta(days=30), end_date=now,
        )

    def _extract_last_quarter(self, match) -> TemporalReference:
        now = datetime.utcnow()
        return TemporalReference(
            has_temporal=True, query_type="relative",
            start_date=now - timedelta(days=90), end_date=now,
        )

    def _extract_last_year(self, match) -> TemporalReference:
        now = datetime.utcnow()
        return TemporalReference(
            has_temporal=True, query_type="relative",
            start_date=now - timedelta(days=365), end_date=now,
        )

    def _extract_this_year(self, match) -> TemporalReference:
        now = datetime.utcnow()
        return TemporalReference(
            has_temporal=True, query_type="absolute",
            start_date=datetime(now.year, 1, 1), end_date=now,
        )

    def _extract_this_month(self, match) -> TemporalReference:
        now = datetime.utcnow()
        return TemporalReference(
            has_temporal=True, query_type="absolute",
            start_date=datetime(now.year, now.month, 1), end_date=now,
        )
```

---

## Common Pitfalls

1. **Using a single decay rate for all content.** Changelogs, tutorials, and reference docs age at different rates. A 30-day half-life is appropriate for changelogs but too aggressive for architectural documentation.
2. **Not handling missing timestamps.** If a document lacks a `created_at` field, recency scoring crashes or assigns an arbitrary score. Default to a moderate age (e.g., 6 months) for documents without timestamps.
3. **Applying recency to queries without temporal intent.** "What is a REST API?" does not benefit from recency. Only apply recency weighting when the temporal query parser detects temporal signals.
4. **Hard filtering with too narrow a range.** If "last month's incidents" returns zero results because all incidents are older, the user gets nothing. Fall back to recency-weighted search when hard filters return empty results.
5. **Not resolving relative dates at query time.** "Last quarter" means different things depending on when the query is made. Always resolve relative dates against the current timestamp, not a cached value.
6. **Ignoring document update timestamps.** A document created in 2020 but updated last week is still relevant. Track both `created_at` and `updated_at` and use the more recent one for recency scoring.

---

## References

- LangChain TimeWeightedVectorStoreRetriever: https://python.langchain.com/docs/how_to/time_weighted_vectorstore/
- Park et al. "Generative Agents" (2023) -- time-weighted memory retrieval
- dateutil parser: https://dateutil.readthedocs.io/
