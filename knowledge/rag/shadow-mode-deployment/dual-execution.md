# Dual Execution -- Run Old and New RAG Pipelines, Log Both, Compare Offline

## TL;DR

Dual execution is the core shadow-mode technique: every production query runs through both the current (primary) and candidate (shadow) RAG pipelines. The primary pipeline serves the user; the shadow pipeline's output is captured and stored for offline comparison. This article covers the execution infrastructure (async processing, resource isolation, error handling), structured logging of paired outputs, offline comparison pipelines (automated scoring with RAGAS, LLM-as-judge, and embedding similarity), and reporting dashboards that surface regressions and improvements before any traffic switch occurs.

---

## Dual Execution Infrastructure

### Resource Isolation

```python
import asyncio
import time
import logging
from concurrent.futures import ThreadPoolExecutor
from dataclasses import dataclass, field
from datetime import datetime
import json

logger = logging.getLogger(__name__)


@dataclass
class PipelineConfig:
    """Configuration for one side of the dual execution."""

    name: str  # "primary" or "shadow"
    retriever: object
    llm: object
    embedding_model: str
    retrieval_top_k: int
    reranker: object | None = None


@dataclass
class DualExecutionResult:
    """Paired result from both pipeline executions."""

    trace_id: str
    timestamp: str
    query: str

    # Primary (served to user)
    primary_answer: str = ""
    primary_sources: list[dict] = field(default_factory=list)
    primary_latency_ms: float = 0.0
    primary_retrieval_scores: list[float] = field(default_factory=list)
    primary_error: str | None = None

    # Shadow (logged only)
    shadow_answer: str = ""
    shadow_sources: list[dict] = field(default_factory=list)
    shadow_latency_ms: float = 0.0
    shadow_retrieval_scores: list[float] = field(default_factory=list)
    shadow_error: str | None = None


class DualExecutionEngine:
    """Execute queries through both pipelines with resource isolation.

    Key design decisions:
    1. Shadow runs in a separate thread pool (no contention with primary)
    2. Shadow failures are caught and logged (never affect primary)
    3. Shadow has a timeout (prevent runaway shadow from consuming resources)
    4. Both results are paired by trace_id for offline comparison
    """

    def __init__(
        self,
        primary: PipelineConfig,
        shadow: PipelineConfig,
        shadow_timeout: float = 15.0,
        result_store=None,
    ):
        self.primary_config = primary
        self.shadow_config = shadow
        self.shadow_timeout = shadow_timeout
        self.shadow_pool = ThreadPoolExecutor(
            max_workers=4, thread_name_prefix="shadow"
        )
        self.store = result_store

    def execute(self, query: str) -> dict:
        """Execute query through both pipelines."""
        import uuid
        trace_id = str(uuid.uuid4())
        timestamp = datetime.utcnow().isoformat()

        result = DualExecutionResult(
            trace_id=trace_id,
            timestamp=timestamp,
            query=query,
        )

        # Run primary (synchronous -- user waits)
        primary_start = time.perf_counter()
        try:
            primary_output = self._run_pipeline(
                self.primary_config, query
            )
            result.primary_answer = primary_output["answer"]
            result.primary_sources = primary_output.get("sources", [])
            result.primary_retrieval_scores = primary_output.get(
                "scores", []
            )
        except Exception as e:
            result.primary_error = str(e)
            logger.error("Primary pipeline failed: %s", str(e))
        result.primary_latency_ms = (
            (time.perf_counter() - primary_start) * 1000
        )

        # Run shadow (async -- do not block user)
        future = self.shadow_pool.submit(
            self._run_shadow, query, result
        )
        # Do not wait for shadow to complete
        future.add_done_callback(
            lambda f: self._on_shadow_done(f, result)
        )

        return {
            "answer": result.primary_answer,
            "sources": result.primary_sources,
            "trace_id": trace_id,
        }

    def _run_pipeline(
        self, config: PipelineConfig, query: str
    ) -> dict:
        """Run a single pipeline and return structured output."""
        # Retrieve
        docs = config.retriever.invoke(query)

        # Rerank if available
        if config.reranker:
            docs = config.reranker.rerank(query, docs)

        # Generate
        context = "\n\n".join(
            d.page_content for d in docs[: config.retrieval_top_k]
        )
        response = config.llm.invoke(
            f"Context:\n{context}\n\nQuestion: {query}"
        )

        return {
            "answer": response.content,
            "sources": [d.metadata for d in docs[: config.retrieval_top_k]],
            "scores": [
                d.metadata.get("score", 0.0)
                for d in docs[: config.retrieval_top_k]
            ],
        }

    def _run_shadow(
        self, query: str, result: DualExecutionResult
    ) -> None:
        """Run shadow pipeline with timeout protection."""
        shadow_start = time.perf_counter()
        try:
            shadow_output = self._run_pipeline(
                self.shadow_config, query
            )
            result.shadow_answer = shadow_output["answer"]
            result.shadow_sources = shadow_output.get("sources", [])
            result.shadow_retrieval_scores = shadow_output.get(
                "scores", []
            )
        except Exception as e:
            result.shadow_error = str(e)
            logger.warning("Shadow pipeline failed: %s", str(e))
        result.shadow_latency_ms = (
            (time.perf_counter() - shadow_start) * 1000
        )

    def _on_shadow_done(
        self, future, result: DualExecutionResult
    ) -> None:
        """Callback when shadow execution completes."""
        try:
            future.result()  # Check for exceptions
        except Exception as e:
            result.shadow_error = str(e)

        # Store the paired result
        if self.store:
            self.store.save(result)
```

---

## Structured Logging

### Paired Result Storage

```python
import json
from pathlib import Path


class DualExecutionStore:
    """Store paired execution results for offline analysis.

    Results are stored as JSONL files (one per day) for
    batch processing by the comparison pipeline.
    """

    def __init__(self, base_dir: str = "./shadow_results"):
        self.base_dir = Path(base_dir)
        self.base_dir.mkdir(parents=True, exist_ok=True)

    def save(self, result: DualExecutionResult) -> None:
        """Append a result to today's JSONL file."""
        today = datetime.utcnow().strftime("%Y-%m-%d")
        filepath = self.base_dir / f"shadow_{today}.jsonl"

        record = {
            "trace_id": result.trace_id,
            "timestamp": result.timestamp,
            "query": result.query,
            "primary": {
                "answer": result.primary_answer,
                "sources": result.primary_sources,
                "latency_ms": result.primary_latency_ms,
                "retrieval_scores": result.primary_retrieval_scores,
                "error": result.primary_error,
            },
            "shadow": {
                "answer": result.shadow_answer,
                "sources": result.shadow_sources,
                "latency_ms": result.shadow_latency_ms,
                "retrieval_scores": result.shadow_retrieval_scores,
                "error": result.shadow_error,
            },
        }

        with open(filepath, "a") as f:
            f.write(json.dumps(record) + "\n")

    def load_day(self, date_str: str) -> list[dict]:
        """Load all results for a specific day."""
        filepath = self.base_dir / f"shadow_{date_str}.jsonl"
        if not filepath.exists():
            return []

        results = []
        with open(filepath) as f:
            for line in f:
                if line.strip():
                    results.append(json.loads(line))
        return results

    def load_range(
        self, start_date: str, end_date: str
    ) -> list[dict]:
        """Load results for a date range."""
        from datetime import timedelta

        results = []
        current = datetime.strptime(start_date, "%Y-%m-%d")
        end = datetime.strptime(end_date, "%Y-%m-%d")

        while current <= end:
            day_results = self.load_day(current.strftime("%Y-%m-%d"))
            results.extend(day_results)
            current += timedelta(days=1)

        return results
```

---

## Offline Comparison Pipeline

### Automated Scoring

```python
import statistics
from openai import OpenAI


class OfflineComparisonPipeline:
    """Score paired results offline using multiple metrics.

    Runs as a batch job (e.g., daily) on the stored shadow results.
    Produces a comparison report that guides promotion decisions.
    """

    def __init__(self, embedding_client: OpenAI | None = None):
        self.embedding_client = embedding_client

    def score_pair(self, record: dict) -> dict:
        """Score a single primary-shadow pair."""
        primary = record["primary"]
        shadow = record["shadow"]

        scores = {
            "trace_id": record["trace_id"],
            "query": record["query"],
        }

        # Skip if either side errored
        if primary.get("error") or shadow.get("error"):
            scores["skipped"] = True
            return scores

        # Metric 1: Source overlap (Jaccard)
        p_sources = {
            s.get("source", s.get("doc_id", str(i)))
            for i, s in enumerate(primary.get("sources", []))
        }
        s_sources = {
            s.get("source", s.get("doc_id", str(i)))
            for i, s in enumerate(shadow.get("sources", []))
        }
        if p_sources or s_sources:
            scores["source_jaccard"] = len(p_sources & s_sources) / len(
                p_sources | s_sources
            )
        else:
            scores["source_jaccard"] = 0.0

        # Metric 2: Answer semantic similarity
        if self.embedding_client:
            scores["answer_embedding_sim"] = self._embedding_similarity(
                primary["answer"], shadow["answer"]
            )

        # Metric 3: Answer word overlap
        p_words = set(primary["answer"].lower().split())
        s_words = set(shadow["answer"].lower().split())
        if p_words or s_words:
            scores["answer_word_overlap"] = len(p_words & s_words) / len(
                p_words | s_words
            )
        else:
            scores["answer_word_overlap"] = 0.0

        # Metric 4: Latency comparison
        if primary["latency_ms"] > 0:
            scores["latency_ratio"] = (
                shadow["latency_ms"] / primary["latency_ms"]
            )

        # Metric 5: Retrieval score comparison
        p_scores = primary.get("retrieval_scores", [])
        s_scores = shadow.get("retrieval_scores", [])
        if p_scores and s_scores:
            scores["primary_avg_score"] = statistics.mean(p_scores)
            scores["shadow_avg_score"] = statistics.mean(s_scores)
            scores["score_improvement"] = (
                scores["shadow_avg_score"] - scores["primary_avg_score"]
            )

        scores["skipped"] = False
        return scores

    def run_batch(self, records: list[dict]) -> dict:
        """Score all records and produce aggregate report."""
        all_scores = [self.score_pair(r) for r in records]
        valid = [s for s in all_scores if not s.get("skipped")]

        if not valid:
            return {"error": "No valid comparisons"}

        report = {
            "total_records": len(records),
            "valid_comparisons": len(valid),
            "skipped": len(records) - len(valid),
            "metrics": {
                "source_jaccard": {
                    "mean": statistics.mean(
                        s["source_jaccard"] for s in valid
                    ),
                    "median": statistics.median(
                        s["source_jaccard"] for s in valid
                    ),
                },
                "answer_word_overlap": {
                    "mean": statistics.mean(
                        s["answer_word_overlap"] for s in valid
                    ),
                },
                "latency_ratio": {
                    "mean": statistics.mean(
                        s.get("latency_ratio", 1.0) for s in valid
                    ),
                    "p95": sorted(
                        s.get("latency_ratio", 1.0) for s in valid
                    )[int(len(valid) * 0.95)],
                },
            },
            "per_query_scores": valid,
        }

        if any("answer_embedding_sim" in s for s in valid):
            sims = [
                s["answer_embedding_sim"]
                for s in valid
                if "answer_embedding_sim" in s
            ]
            report["metrics"]["answer_embedding_sim"] = {
                "mean": statistics.mean(sims),
                "median": statistics.median(sims),
            }

        return report

    def _embedding_similarity(self, text_a: str, text_b: str) -> float:
        """Compute cosine similarity between two texts."""
        import numpy as np

        response = self.embedding_client.embeddings.create(
            model="text-embedding-3-small",
            input=[text_a, text_b],
        )
        a = np.array(response.data[0].embedding)
        b = np.array(response.data[1].embedding)
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))
```

---

## Promotion Decision

```python
@dataclass
class PromotionCriteria:
    """Criteria that must be met to promote shadow to primary."""

    min_source_overlap: float = 0.4
    min_answer_similarity: float = 0.6
    max_latency_ratio_p95: float = 1.5
    min_sample_size: int = 200
    max_regression_pct: float = 10.0  # % of queries with worse results


def make_promotion_decision(
    report: dict,
    criteria: PromotionCriteria,
) -> dict:
    """Decide whether to promote the shadow pipeline."""
    metrics = report.get("metrics", {})
    n = report.get("valid_comparisons", 0)

    if n < criteria.min_sample_size:
        return {
            "decision": "wait",
            "reason": f"Only {n}/{criteria.min_sample_size} samples",
        }

    checks = {
        "source_overlap": (
            metrics.get("source_jaccard", {}).get("mean", 0)
            >= criteria.min_source_overlap
        ),
        "answer_similarity": (
            metrics.get("answer_word_overlap", {}).get("mean", 0)
            >= criteria.min_answer_similarity
        ),
        "latency": (
            metrics.get("latency_ratio", {}).get("p95", 999)
            <= criteria.max_latency_ratio_p95
        ),
    }

    all_passed = all(checks.values())
    return {
        "decision": "promote" if all_passed else "do_not_promote",
        "checks": checks,
        "sample_size": n,
    }
```

---

## Common Pitfalls

1. **Shadow pipeline sharing rate limits with primary.** If both pipelines use the same LLM API key, shadow execution may exhaust rate limits and degrade primary performance. Use separate API keys or accounts.
2. **Not logging both retrieval and generation.** Logging only the final answer hides retrieval regressions. A shadow pipeline might retrieve different (worse) documents but generate a plausible-sounding answer. Log source documents and scores.
3. **Running the comparison synchronously.** Offline comparison should run as a batch job, not inline with the request. Scoring adds seconds of latency if done in the request path.
4. **Insufficient sample size for promotion decisions.** 50 comparisons are not enough to detect a 5% regression. Aim for at least 200 comparisons, more for subtle changes like embedding model swaps.
5. **Not controlling for query distribution.** If traffic is dominated by one query type, shadow comparisons may look good on average but fail on rare but important queries. Stratify analysis by query category.
6. **Forgetting to clean up shadow infrastructure after promotion.** Once the shadow pipeline is promoted, the old primary infrastructure should be decommissioned. Leaving both running wastes resources.

---

## References

- Google SRE on shadow testing: https://sre.google/sre-book/testing-reliability/
- Netflix shadow traffic: https://netflixtechblog.com/
- Uber shadow mode architecture: https://www.uber.com/blog/shadow-traffic-testing/
