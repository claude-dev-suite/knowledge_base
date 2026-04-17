# Shadow-Mode Deployment -- Canary and Shadow Patterns for RAG Changes

## TL;DR

Deploying changes to a RAG pipeline (new embedding model, different chunking strategy, updated retrieval parameters, new LLM) is risky because quality regressions are hard to detect with unit tests alone. Shadow-mode deployment runs the new pipeline alongside the existing one, comparing outputs without exposing users to the new version. Canary deployment routes a small percentage of real traffic to the new version while monitoring for regressions. Together, these patterns provide a safety net: validate changes on real traffic before full rollout, detect regressions early, and enable instant rollback. This overview covers the deployment spectrum, when to use each strategy, the comparison framework, and decision criteria for promoting or rolling back. Subsequent articles cover dual-execution implementation and gradual rollout with feature flags.

---

## The Deployment Risk Problem

### What Can Go Wrong

| Change | Risk | Detection Difficulty |
|--------|------|---------------------|
| New embedding model | Retrieval quality regression | Hard (gradual degradation) |
| New chunking strategy | Information loss in chunks | Hard (specific queries fail) |
| Retrieval parameter change (top_k) | Over/under retrieval | Medium |
| New LLM model | Answer quality change | Hard (subjective quality) |
| System prompt change | Behavior shift | Medium |
| Reranker change | Document ordering regression | Hard |
| Index rebuild | Missing or duplicate documents | Medium |

### Why Standard Testing Is Insufficient

```
Unit tests:  Pass (they test known inputs/outputs)
Integration: Pass (the pipeline runs end-to-end)
Staging:     Pass (staging data is clean and small)
Production:  FAIL (real queries are diverse, messy, and edge-case-heavy)

The gap between staging and production is where shadow/canary helps.
```

---

## Deployment Strategies

### Strategy Spectrum

```
Risk Level:    LOW                                           HIGH
               |                                              |
Strategy:      Full rollout -> Canary -> Shadow -> A/B test
               |                                              |
Speed:         FAST                                          SLOW
```

| Strategy | Traffic Exposure | Comparison | Rollback Speed |
|----------|-----------------|------------|----------------|
| Shadow mode | 0% (users see old only) | Offline comparison | N/A (no exposure) |
| Canary | 1-10% see new | Real-time metrics | Instant |
| Blue-green | 100% switchover | Pre-switch validation | Instant (swap back) |
| A/B test | 50/50 split | Statistical significance | Moderate |

---

## Shadow Mode Architecture

### Dual-Execution Pipeline

```python
import asyncio
import time
import logging
from dataclasses import dataclass, field
from typing import Any

logger = logging.getLogger(__name__)


@dataclass
class ShadowResult:
    """Result from both primary and shadow pipelines."""

    query: str
    primary_answer: str
    shadow_answer: str
    primary_sources: list[dict]
    shadow_sources: list[dict]
    primary_latency_ms: float
    shadow_latency_ms: float
    comparison: dict = field(default_factory=dict)


class ShadowModeRAG:
    """Run two RAG pipelines simultaneously: primary serves the user,
    shadow runs in the background for comparison.

    The shadow pipeline's output is never shown to the user.
    Results are logged for offline analysis.
    """

    def __init__(
        self,
        primary_pipeline,
        shadow_pipeline,
        comparison_fn=None,
        log_fn=None,
    ):
        self.primary = primary_pipeline
        self.shadow = shadow_pipeline
        self.compare = comparison_fn or self._default_compare
        self.log = log_fn or self._default_log

    def query(self, user_query: str) -> dict:
        """Execute query through both pipelines.

        The primary result is returned to the user.
        The shadow result is logged for comparison.
        """
        # Run primary (blocking -- user waits for this)
        primary_start = time.perf_counter()
        primary_result = self.primary.query(user_query)
        primary_latency = (time.perf_counter() - primary_start) * 1000

        # Run shadow (async -- do not block the user)
        shadow_result = None
        shadow_latency = 0.0
        try:
            shadow_start = time.perf_counter()
            shadow_result = self.shadow.query(user_query)
            shadow_latency = (time.perf_counter() - shadow_start) * 1000
        except Exception as e:
            logger.warning("Shadow pipeline failed: %s", str(e))

        # Compare if shadow succeeded
        if shadow_result:
            comparison = self.compare(
                primary_result, shadow_result, user_query
            )

            shadow_log = ShadowResult(
                query=user_query,
                primary_answer=primary_result.get("answer", ""),
                shadow_answer=shadow_result.get("answer", ""),
                primary_sources=primary_result.get("sources", []),
                shadow_sources=shadow_result.get("sources", []),
                primary_latency_ms=primary_latency,
                shadow_latency_ms=shadow_latency,
                comparison=comparison,
            )
            self.log(shadow_log)

        # Always return primary result to user
        return primary_result

    def _default_compare(
        self,
        primary: dict,
        shadow: dict,
        query: str,
    ) -> dict:
        """Default comparison: source overlap and answer similarity."""
        primary_source_ids = {
            s.get("source", s.get("doc_id", ""))
            for s in primary.get("sources", [])
        }
        shadow_source_ids = {
            s.get("source", s.get("doc_id", ""))
            for s in shadow.get("sources", [])
        }

        if primary_source_ids or shadow_source_ids:
            overlap = len(primary_source_ids & shadow_source_ids)
            total = len(primary_source_ids | shadow_source_ids)
            source_jaccard = overlap / total if total > 0 else 0
        else:
            source_jaccard = 0

        # Simple answer similarity (word overlap)
        p_words = set(primary.get("answer", "").lower().split())
        s_words = set(shadow.get("answer", "").lower().split())
        if p_words or s_words:
            answer_jaccard = (
                len(p_words & s_words)
                / len(p_words | s_words)
            )
        else:
            answer_jaccard = 0

        return {
            "source_overlap": source_jaccard,
            "answer_similarity": answer_jaccard,
            "source_count_diff": (
                len(shadow_source_ids) - len(primary_source_ids)
            ),
        }

    def _default_log(self, result: ShadowResult) -> None:
        logger.info(
            "Shadow comparison -- Query: %s, "
            "Source overlap: %.2f, Answer similarity: %.2f, "
            "Primary: %.0fms, Shadow: %.0fms",
            result.query[:50],
            result.comparison.get("source_overlap", 0),
            result.comparison.get("answer_similarity", 0),
            result.primary_latency_ms,
            result.shadow_latency_ms,
        )
```

### Async Shadow Execution

```python
class AsyncShadowModeRAG:
    """Shadow mode with async execution to minimize latency impact."""

    def __init__(self, primary, shadow, comparison_store):
        self.primary = primary
        self.shadow = shadow
        self.store = comparison_store

    async def query(self, user_query: str) -> dict:
        """Run primary and shadow concurrently."""
        primary_task = asyncio.create_task(
            self._run_primary(user_query)
        )
        shadow_task = asyncio.create_task(
            self._run_shadow(user_query)
        )

        # Wait for primary (this is what the user sees)
        primary_result = await primary_task

        # Do not wait for shadow -- let it complete in background
        shadow_task.add_done_callback(
            lambda t: self._on_shadow_complete(
                user_query, primary_result, t
            )
        )

        return primary_result

    async def _run_primary(self, query: str) -> dict:
        return await asyncio.to_thread(self.primary.query, query)

    async def _run_shadow(self, query: str) -> dict:
        return await asyncio.to_thread(self.shadow.query, query)

    def _on_shadow_complete(
        self, query: str, primary: dict, task
    ) -> None:
        try:
            shadow = task.result()
            self.store.save_comparison(query, primary, shadow)
        except Exception as e:
            logger.warning("Shadow task failed: %s", str(e))
```

---

## Comparison Metrics

### What to Compare Between Primary and Shadow

```python
@dataclass
class ComparisonMetrics:
    """Metrics comparing primary and shadow pipeline outputs."""

    # Retrieval comparison
    source_overlap_jaccard: float  # Jaccard similarity of retrieved doc sets
    source_rank_correlation: float  # Spearman correlation of doc rankings
    retrieval_recall_at_k: float  # How many primary docs appear in shadow top-k

    # Answer comparison
    answer_semantic_similarity: float  # Embedding cosine similarity
    answer_length_ratio: float  # shadow_len / primary_len
    answer_factual_agreement: float  # Do both answers agree on facts?

    # Latency comparison
    latency_ratio: float  # shadow_latency / primary_latency

    # Overall
    should_promote: bool  # Based on all metrics


class ComparisonAnalyzer:
    """Analyze shadow comparison results to decide on promotion."""

    def __init__(
        self,
        min_source_overlap: float = 0.5,
        min_answer_similarity: float = 0.7,
        max_latency_ratio: float = 1.5,
        min_sample_size: int = 100,
    ):
        self.min_source_overlap = min_source_overlap
        self.min_answer_similarity = min_answer_similarity
        self.max_latency_ratio = max_latency_ratio
        self.min_sample_size = min_sample_size

    def analyze(
        self, comparisons: list[dict]
    ) -> dict:
        """Analyze a batch of shadow comparisons."""
        if len(comparisons) < self.min_sample_size:
            return {
                "verdict": "insufficient_data",
                "sample_size": len(comparisons),
                "min_required": self.min_sample_size,
            }

        import statistics

        source_overlaps = [
            c["source_overlap"] for c in comparisons
        ]
        answer_sims = [
            c["answer_similarity"] for c in comparisons
        ]
        latency_ratios = [
            c.get("shadow_latency_ms", 1)
            / max(c.get("primary_latency_ms", 1), 1)
            for c in comparisons
        ]

        metrics = {
            "sample_size": len(comparisons),
            "source_overlap": {
                "mean": statistics.mean(source_overlaps),
                "p25": sorted(source_overlaps)[len(source_overlaps) // 4],
                "median": statistics.median(source_overlaps),
            },
            "answer_similarity": {
                "mean": statistics.mean(answer_sims),
                "median": statistics.median(answer_sims),
            },
            "latency_ratio": {
                "mean": statistics.mean(latency_ratios),
                "p95": sorted(latency_ratios)[
                    int(len(latency_ratios) * 0.95)
                ],
            },
        }

        # Decision logic
        source_ok = (
            metrics["source_overlap"]["mean"]
            >= self.min_source_overlap
        )
        answer_ok = (
            metrics["answer_similarity"]["mean"]
            >= self.min_answer_similarity
        )
        latency_ok = (
            metrics["latency_ratio"]["p95"]
            <= self.max_latency_ratio
        )

        if source_ok and answer_ok and latency_ok:
            verdict = "promote"
        elif source_ok and answer_ok:
            verdict = "promote_with_latency_warning"
        else:
            verdict = "do_not_promote"

        metrics["verdict"] = verdict
        metrics["checks"] = {
            "source_overlap": source_ok,
            "answer_similarity": answer_ok,
            "latency": latency_ok,
        }

        return metrics
```

---

## Canary Deployment

### Traffic Splitting

```python
import random


class CanaryRouter:
    """Route traffic between stable and canary RAG pipelines.

    A configurable percentage of traffic goes to the canary.
    The rest goes to the stable pipeline.
    """

    def __init__(
        self,
        stable_pipeline,
        canary_pipeline,
        canary_percentage: float = 5.0,
    ):
        self.stable = stable_pipeline
        self.canary = canary_pipeline
        self.canary_pct = canary_percentage

    def query(self, user_query: str) -> dict:
        if random.random() * 100 < self.canary_pct:
            result = self.canary.query(user_query)
            result["_pipeline"] = "canary"
        else:
            result = self.stable.query(user_query)
            result["_pipeline"] = "stable"
        return result

    def set_canary_percentage(self, pct: float) -> None:
        """Adjust canary traffic percentage (0-100)."""
        self.canary_pct = max(0, min(100, pct))
```

---

## Common Pitfalls

1. **Running shadow mode without enough traffic.** Shadow comparisons need statistical significance. With 10 queries per day, you need weeks to accumulate enough data. Ensure sufficient traffic volume before relying on shadow results.
2. **Comparing only answers, not retrieval.** Two pipelines can produce similar answers from completely different documents. Compare both retrieved sources and generated answers for a complete picture.
3. **Shadow pipeline affecting production latency.** If the shadow pipeline runs synchronously and shares resources with production (same embedding API, same vector store), it can degrade production performance. Run shadow asynchronously with separate resources.
4. **Not defining promotion criteria before starting shadow.** Without pre-defined thresholds (source overlap > 0.5, answer similarity > 0.7), the decision to promote becomes subjective. Define criteria before running the shadow experiment.
5. **Canary without automatic rollback.** If the canary degrades quality for 5% of users and nobody notices for hours, that is hundreds of bad answers. Automate rollback when canary metrics drop below thresholds.
6. **Testing only common queries.** Shadow mode naturally tests whatever queries production receives. If your traffic is skewed toward common queries, rare edge cases remain untested. Supplement with a curated evaluation dataset.

---

## References

- Google SRE on canary deployments: https://sre.google/workbook/canarying-releases/
- LaunchDarkly feature flags: https://docs.launchdarkly.com/
- Netflix canary analysis: https://netflixtechblog.com/automated-canary-analysis-at-netflix-with-kayenta-3260bc7acc69
