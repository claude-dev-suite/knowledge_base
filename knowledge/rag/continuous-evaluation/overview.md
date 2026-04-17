# Continuous Evaluation for RAG Pipelines -- Overview

## TL;DR

Continuous evaluation applies CI/CD principles to RAG quality: every pipeline change is automatically evaluated against a golden dataset, metrics are tracked over time, regressions are caught before they reach production, and production quality is monitored through scheduled evaluation runs. This guide covers the architecture of a continuous evaluation system, what to measure and when, how to build golden datasets that grow over time, regression detection strategies, and the integration points between evaluation and your deployment pipeline. The goal is to make RAG quality a first-class metric that is tested as rigorously as code correctness.

---

## Why Continuous Evaluation Matters

### The RAG Quality Decay Problem

RAG pipelines degrade silently. Unlike application bugs that cause visible errors, quality regressions in retrieval and generation produce answers that look plausible but are subtly wrong. The causes:

| Decay Source | Mechanism | Detection Difficulty |
|---|---|---|
| Corpus drift | New documents change retrieval distribution | Hard -- no errors, just different results |
| Embedding model updates | Provider silently updates embeddings | Very hard -- old vectors incompatible |
| LLM version changes | OpenAI/Anthropic updates model behavior | Hard -- output quality shifts gradually |
| Prompt regression | Well-intentioned prompt edit hurts edge cases | Medium -- only visible on affected queries |
| Index corruption | Vector store issues after scaling/migration | Medium -- retrieval quality drops uniformly |
| Chunk overlap changes | Re-indexing with different chunk parameters | Hard -- precision/recall balance shifts |

Without continuous evaluation, these regressions go undetected for days or weeks until users notice degraded answers.

### The Evaluation Maturity Model

```
Level 0: No evaluation
  "We tried it and it seemed to work"
  --> Ship and pray

Level 1: Manual evaluation
  "We tested 20 queries before the demo"
  --> Works for prototypes, does not scale

Level 2: Periodic evaluation
  "We run RAGAS monthly on 100 queries"
  --> Catches large regressions, misses gradual drift

Level 3: Continuous evaluation (this guide)
  "Every PR runs evaluation, production is monitored daily"
  --> Catches regressions before they ship, detects drift

Level 4: Adaptive evaluation
  "Evaluation dataset grows from production failures"
  --> Self-improving quality system
```

This guide covers Levels 3 and 4.

---

## Architecture of a Continuous Evaluation System

### Components

```
+-------------------+     +------------------+     +-------------------+
| Golden Dataset    |     | Evaluation       |     | Metrics Store     |
| (version-        |---->| Runner           |---->| (time-series DB   |
|  controlled)      |     | (pytest/CI)      |     |  or JSON files)   |
+-------------------+     +------------------+     +-------------------+
                               |                         |
                               v                         v
                     +------------------+     +-------------------+
                     | Regression       |     | Dashboard /       |
                     | Detector         |     | Alerting          |
                     | (threshold +     |     | (Grafana, Slack,  |
                     |  statistical)    |     |  PagerDuty)       |
                     +------------------+     +-------------------+
```

### The Golden Dataset

The golden dataset is the foundation of continuous evaluation. It contains query-answer pairs that represent your pipeline's expected behavior.

```python
"""
Golden dataset management for continuous RAG evaluation.
"""
import hashlib
import json
from dataclasses import asdict, dataclass, field
from datetime import datetime
from pathlib import Path


@dataclass
class GoldenExample:
    """A single golden evaluation example."""
    query: str
    expected_answer: str
    expected_contexts: list[str]
    category: str  # e.g., "factual", "reasoning", "comparison"
    difficulty: str  # "easy", "medium", "hard"
    source: str  # "manual", "production_failure", "adversarial"
    added_date: str = ""
    query_id: str = ""
    tags: list[str] = field(default_factory=list)

    def __post_init__(self):
        if not self.query_id:
            self.query_id = hashlib.md5(self.query.encode()).hexdigest()[:12]
        if not self.added_date:
            self.added_date = datetime.now().strftime("%Y-%m-%d")


class GoldenDatasetManager:
    """
    Manage the golden dataset used for continuous evaluation.

    The golden dataset is version-controlled alongside the RAG pipeline
    code. It grows over time as production failures and edge cases
    are added.
    """

    def __init__(self, dataset_path: str):
        self.path = Path(dataset_path)
        self.examples = self._load()

    def _load(self) -> list[GoldenExample]:
        if not self.path.exists():
            return []
        with open(self.path) as f:
            data = json.load(f)
        return [GoldenExample(**item) for item in data]

    def save(self) -> None:
        with open(self.path, "w") as f:
            json.dump([asdict(ex) for ex in self.examples], f, indent=2)

    def add_example(self, example: GoldenExample) -> None:
        """Add a new example, checking for duplicates."""
        existing_ids = {ex.query_id for ex in self.examples}
        if example.query_id in existing_ids:
            print(f"Duplicate query_id {example.query_id}, skipping")
            return
        self.examples.append(example)
        self.save()
        print(f"Added example {example.query_id}: {example.query[:60]}...")

    def add_from_production_failure(
        self,
        query: str,
        wrong_answer: str,
        correct_answer: str,
        relevant_contexts: list[str],
        tags: list[str] | None = None,
    ) -> None:
        """
        Add a new golden example from a production failure.

        This is the primary mechanism for growing the dataset over time.
        When a user reports a bad answer, capture the query and correct
        answer and add it to the golden set.
        """
        example = GoldenExample(
            query=query,
            expected_answer=correct_answer,
            expected_contexts=relevant_contexts,
            category="production_failure",
            difficulty="hard",
            source="production_failure",
            tags=tags or ["regression"],
        )
        self.add_example(example)

    def get_by_category(self, category: str) -> list[GoldenExample]:
        return [ex for ex in self.examples if ex.category == category]

    def get_by_difficulty(self, difficulty: str) -> list[GoldenExample]:
        return [ex for ex in self.examples if ex.difficulty == difficulty]

    def statistics(self) -> dict:
        """Report dataset composition."""
        stats = {
            "total": len(self.examples),
            "by_category": {},
            "by_difficulty": {},
            "by_source": {},
        }
        for ex in self.examples:
            stats["by_category"][ex.category] = (
                stats["by_category"].get(ex.category, 0) + 1
            )
            stats["by_difficulty"][ex.difficulty] = (
                stats["by_difficulty"].get(ex.difficulty, 0) + 1
            )
            stats["by_source"][ex.source] = (
                stats["by_source"].get(ex.source, 0) + 1
            )
        return stats
```

### Building the Initial Golden Dataset

Start with 50-100 examples covering your query distribution:

| Category | Count | Source | Purpose |
|---|---|---|---|
| Factual lookup | 20-30 | Manual curation | Basic retrieval accuracy |
| Multi-hop reasoning | 10-15 | Manual curation | Complex query handling |
| Comparison queries | 10-15 | Manual curation | Multi-document synthesis |
| Edge cases | 10-15 | Known failure modes | Boundary conditions |
| Adversarial queries | 5-10 | Red-teaming | Robustness |
| Out-of-scope queries | 5-10 | Manual curation | Appropriate abstention |

```python
"""
Bootstrap a golden dataset from existing documents.
"""
import anthropic


def generate_golden_examples(
    corpus_passages: list[str],
    num_examples: int = 50,
    model: str = "claude-sonnet-4-20250514",
) -> list[dict]:
    """
    Generate initial golden dataset by having an LLM create
    query-answer pairs from corpus passages.

    These should be reviewed and edited by a human before use.
    """
    client = anthropic.Anthropic()
    examples = []

    for passage in corpus_passages[:num_examples]:
        response = client.messages.create(
            model=model,
            max_tokens=500,
            messages=[{
                "role": "user",
                "content": (
                    "Given this passage from a knowledge base, generate:\n"
                    "1. A natural question a user might ask\n"
                    "2. The correct answer based on the passage\n"
                    "3. A category (factual, reasoning, comparison, or procedural)\n"
                    "4. A difficulty (easy, medium, hard)\n\n"
                    f"Passage:\n{passage[:2000]}\n\n"
                    "Respond in this exact JSON format:\n"
                    '{"query": "...", "answer": "...", '
                    '"category": "...", "difficulty": "..."}'
                ),
            }],
        )

        try:
            import json
            data = json.loads(response.content[0].text)
            data["expected_contexts"] = [passage]
            data["expected_answer"] = data.pop("answer")
            data["source"] = "llm_generated"
            examples.append(data)
        except (json.JSONDecodeError, KeyError):
            continue

    return examples
```

---

## Regression Detection Strategies

### Strategy 1: Threshold-Based

The simplest approach: fail if any metric drops below a fixed threshold.

```python
"""
Threshold-based regression detection.
"""
from dataclasses import dataclass


@dataclass
class QualityThresholds:
    """Minimum acceptable quality thresholds."""
    faithfulness: float = 0.75
    answer_relevancy: float = 0.70
    context_precision: float = 0.60
    context_recall: float = 0.65


def check_thresholds(
    metrics: dict[str, float],
    thresholds: QualityThresholds,
) -> tuple[bool, list[str]]:
    """
    Check if all metrics meet minimum thresholds.

    Returns (passed, list_of_failures).
    """
    failures = []
    threshold_dict = {
        "faithfulness": thresholds.faithfulness,
        "answer_relevancy": thresholds.answer_relevancy,
        "context_precision": thresholds.context_precision,
        "context_recall": thresholds.context_recall,
    }

    for metric_name, threshold in threshold_dict.items():
        if metric_name in metrics:
            if metrics[metric_name] < threshold:
                failures.append(
                    f"{metric_name}: {metrics[metric_name]:.4f} < {threshold:.4f}"
                )

    passed = len(failures) == 0
    return passed, failures
```

**Limitation**: does not catch gradual drift. A metric can drop from 0.90 to 0.76 over 10 releases without triggering a 0.75 threshold.

### Strategy 2: Delta-Based (Relative Regression)

Fail if any metric drops more than X% compared to the previous baseline.

```python
"""
Delta-based regression detection: catch relative drops.
"""
import json
from pathlib import Path


class DeltaRegressionDetector:
    """
    Detect regressions by comparing against the previous baseline.

    Catches gradual drift that threshold-based detection misses.
    """

    def __init__(
        self,
        baseline_path: str,
        max_regression: float = 0.03,  # 3% max drop
    ):
        self.baseline_path = Path(baseline_path)
        self.max_regression = max_regression
        self.baseline = self._load_baseline()

    def _load_baseline(self) -> dict:
        if self.baseline_path.exists():
            with open(self.baseline_path) as f:
                return json.load(f)
        return {}

    def check(
        self,
        current_metrics: dict[str, float],
    ) -> tuple[bool, list[str]]:
        """
        Check for regressions against the baseline.

        Returns (passed, list_of_regressions).
        """
        if not self.baseline:
            # No baseline yet -- save current as baseline
            self._save_baseline(current_metrics)
            return True, []

        regressions = []
        for metric, current_value in current_metrics.items():
            if metric in self.baseline:
                baseline_value = self.baseline[metric]
                drop = baseline_value - current_value

                if drop > self.max_regression:
                    regressions.append(
                        f"{metric}: {baseline_value:.4f} -> {current_value:.4f} "
                        f"(dropped {drop:.4f}, max allowed {self.max_regression})"
                    )

        passed = len(regressions) == 0

        if passed:
            # Update baseline with current values (ratchet forward)
            self._save_baseline(current_metrics)

        return passed, regressions

    def _save_baseline(self, metrics: dict) -> None:
        with open(self.baseline_path, "w") as f:
            json.dump(metrics, f, indent=2)
```

### Strategy 3: Statistical Regression (Best for Production)

Use statistical tests to determine if a metric change is significant or just noise.

```python
"""
Statistical regression detection using bootstrap confidence intervals.
"""
import numpy as np
from scipy import stats


class StatisticalRegressionDetector:
    """
    Use bootstrap confidence intervals to detect statistically
    significant regressions.

    This avoids false alarms from evaluation noise while catching
    real regressions.
    """

    def __init__(
        self,
        significance_level: float = 0.05,
        n_bootstrap: int = 1000,
    ):
        self.significance_level = significance_level
        self.n_bootstrap = n_bootstrap

    def is_regression(
        self,
        baseline_scores: np.ndarray,
        current_scores: np.ndarray,
    ) -> dict:
        """
        Test whether current scores represent a significant regression
        from baseline scores.

        Args:
            baseline_scores: per-query scores from the baseline run
            current_scores: per-query scores from the current run

        Returns:
            dict with test results
        """
        # Paired difference (if same queries)
        if len(baseline_scores) == len(current_scores):
            diffs = current_scores - baseline_scores
            mean_diff = np.mean(diffs)

            # Bootstrap confidence interval for the mean difference
            bootstrap_means = []
            for _ in range(self.n_bootstrap):
                sample = np.random.choice(diffs, size=len(diffs), replace=True)
                bootstrap_means.append(np.mean(sample))

            ci_lower = np.percentile(bootstrap_means, 100 * self.significance_level / 2)
            ci_upper = np.percentile(bootstrap_means, 100 * (1 - self.significance_level / 2))

            # Regression if the entire CI is below zero
            is_significant_regression = ci_upper < 0

            return {
                "mean_difference": float(mean_diff),
                "ci_lower": float(ci_lower),
                "ci_upper": float(ci_upper),
                "is_regression": is_significant_regression,
                "p_value": float(np.mean(np.array(bootstrap_means) < 0)),
            }

        # Unpaired test (different query sets)
        else:
            t_stat, p_value = stats.ttest_ind(
                current_scores, baseline_scores, alternative="less"
            )
            mean_diff = np.mean(current_scores) - np.mean(baseline_scores)

            return {
                "mean_difference": float(mean_diff),
                "t_statistic": float(t_stat),
                "p_value": float(p_value),
                "is_regression": p_value < self.significance_level,
            }

    def check_all_metrics(
        self,
        baseline_per_query: dict[str, np.ndarray],
        current_per_query: dict[str, np.ndarray],
    ) -> dict:
        """Check all metrics for regression."""
        results = {}
        any_regression = False

        for metric in baseline_per_query:
            if metric in current_per_query:
                result = self.is_regression(
                    baseline_per_query[metric],
                    current_per_query[metric],
                )
                results[metric] = result
                if result["is_regression"]:
                    any_regression = True

                status = "REGRESSION" if result["is_regression"] else "OK"
                print(
                    f"  {metric}: diff={result['mean_difference']:+.4f}, "
                    f"p={result.get('p_value', 0):.4f} [{status}]"
                )

        results["any_regression"] = any_regression
        return results
```

---

## Evaluation Scheduling

### When to Evaluate

| Trigger | What to Run | Why |
|---|---|---|
| Every PR | Smoke test (20 queries) | Catch obvious regressions quickly |
| Every merge to main | Full eval (100+ queries) | Gate releases on quality |
| Daily (scheduled) | Production sample eval | Detect corpus drift and model changes |
| Weekly (scheduled) | Full eval + statistical comparison | Comprehensive quality tracking |
| After re-indexing | Full eval + retrieval focus | Catch chunking/embedding issues |
| After LLM provider change | Full eval | Detect generation quality shifts |

### The Evaluation Pyramid

```
                    /\
                   /  \   Weekly: full statistical evaluation
                  /    \  (200+ queries, confidence intervals)
                 /------\
                /        \  Daily: production sample monitoring
               /          \ (50-100 production queries scored)
              /------------\
             /              \  Per-merge: full golden dataset
            /                \ (100+ queries, all metrics)
           /------------------\
          /                    \  Per-PR: smoke test
         /                      \ (20 high-priority queries, fast)
        /------------------------\
```

---

## Metric Tracking and Storage

### Time-Series Metrics Store

```python
"""
Track evaluation metrics over time for trend analysis.
"""
import json
from datetime import datetime
from pathlib import Path


class MetricsStore:
    """
    Simple file-based metrics store for evaluation results.

    For production, consider InfluxDB, Prometheus, or a time-series database.
    """

    def __init__(self, store_dir: str = "./eval_metrics"):
        self.store_dir = Path(store_dir)
        self.store_dir.mkdir(parents=True, exist_ok=True)

    def record(
        self,
        metrics: dict[str, float],
        pipeline_version: str,
        eval_type: str,  # "pr", "merge", "daily", "weekly"
        metadata: dict | None = None,
    ) -> str:
        """Record an evaluation run."""
        timestamp = datetime.now().isoformat()
        record = {
            "timestamp": timestamp,
            "pipeline_version": pipeline_version,
            "eval_type": eval_type,
            "metrics": metrics,
            "metadata": metadata or {},
        }

        # Append to daily log
        date_str = datetime.now().strftime("%Y-%m-%d")
        log_path = self.store_dir / f"{date_str}.jsonl"
        with open(log_path, "a") as f:
            f.write(json.dumps(record) + "\n")

        return timestamp

    def get_history(
        self,
        metric_name: str,
        days: int = 30,
    ) -> list[tuple[str, float]]:
        """Get metric values over the last N days."""
        history = []

        for log_file in sorted(self.store_dir.glob("*.jsonl")):
            with open(log_file) as f:
                for line in f:
                    record = json.loads(line)
                    if metric_name in record["metrics"]:
                        history.append((
                            record["timestamp"],
                            record["metrics"][metric_name],
                        ))

        return history[-days * 10:]  # Rough limit

    def detect_trend(
        self,
        metric_name: str,
        window: int = 10,
    ) -> dict:
        """
        Detect if a metric is trending downward.

        Returns trend analysis over the last `window` data points.
        """
        import numpy as np

        history = self.get_history(metric_name)
        if len(history) < window:
            return {"trend": "insufficient_data", "data_points": len(history)}

        recent = [v for _, v in history[-window:]]
        values = np.array(recent)

        # Linear regression
        x = np.arange(len(values))
        slope, intercept, r_value, p_value, std_err = (
            __import__("scipy").stats.linregress(x, values)
        )

        trend = "declining" if slope < -0.005 else "stable" if abs(slope) < 0.005 else "improving"

        return {
            "trend": trend,
            "slope": float(slope),
            "r_squared": float(r_value ** 2),
            "p_value": float(p_value),
            "current_value": float(values[-1]),
            "mean_value": float(np.mean(values)),
            "data_points": len(values),
        }
```

---

## Growing the Dataset Over Time

### Feedback Loop: Production Failures to Golden Dataset

```python
"""
Capture production failures and add them to the golden dataset.
"""
from datetime import datetime


class ProductionFeedbackLoop:
    """
    Capture production failures and convert them into golden
    dataset examples.

    This creates a virtuous cycle: production failures improve
    evaluation, which catches similar failures earlier.
    """

    def __init__(
        self,
        golden_dataset_manager,
        feedback_log_path: str = "./feedback_log.jsonl",
    ):
        self.golden = golden_dataset_manager
        self.feedback_log = feedback_log_path

    def report_bad_answer(
        self,
        query: str,
        bad_answer: str,
        correct_answer: str,
        retrieved_contexts: list[str],
        reporter: str = "user",
        severity: str = "medium",
    ) -> str:
        """
        Report a bad answer from production.

        This will:
        1. Log the failure for analysis
        2. Add a golden example to catch this failure in CI
        """
        import json

        # Log the failure
        feedback = {
            "timestamp": datetime.now().isoformat(),
            "query": query,
            "bad_answer": bad_answer,
            "correct_answer": correct_answer,
            "contexts": retrieved_contexts,
            "reporter": reporter,
            "severity": severity,
        }

        with open(self.feedback_log, "a") as f:
            f.write(json.dumps(feedback) + "\n")

        # Add to golden dataset
        self.golden.add_from_production_failure(
            query=query,
            wrong_answer=bad_answer,
            correct_answer=correct_answer,
            relevant_contexts=retrieved_contexts,
            tags=[f"severity:{severity}", f"reporter:{reporter}"],
        )

        return f"Feedback recorded and added to golden dataset"

    def analyze_failure_patterns(self) -> dict:
        """
        Analyze production failures to identify systematic issues.
        """
        import json
        from collections import Counter

        failures = []
        with open(self.feedback_log) as f:
            for line in f:
                failures.append(json.loads(line))

        if not failures:
            return {"total_failures": 0}

        # Basic analysis
        severities = Counter(f["severity"] for f in failures)
        reporters = Counter(f["reporter"] for f in failures)

        # Time trend
        by_week = Counter()
        for f in failures:
            week = f["timestamp"][:10]  # YYYY-MM-DD
            by_week[week] += 1

        return {
            "total_failures": len(failures),
            "by_severity": dict(severities),
            "by_reporter": dict(reporters),
            "by_date": dict(sorted(by_week.items())),
            "recent_failures": failures[-5:],
        }
```

---

## Common Pitfalls

1. **Golden dataset too small**: fewer than 50 examples gives noisy results. Start with 50, grow to 200+ through production feedback.

2. **Golden dataset not representative**: if all examples are easy factual lookups, you will miss regressions on reasoning queries. Cover all query types your users actually ask.

3. **No baseline management**: without a tracked baseline, you cannot detect gradual drift. Store baselines in version control alongside the golden dataset.

4. **Evaluating too infrequently**: monthly evaluation misses regressions that accumulate between evaluations. At minimum, evaluate on every merge to main.

5. **Ignoring per-query results**: aggregate metrics hide important patterns. A system scoring 0.80 overall might score 0.95 on 80% of queries and 0.20 on 20%. Analyze the tail.

6. **Not segmenting metrics**: track metrics separately by query category (factual, reasoning, comparison). A change that improves factual queries but regresses reasoning queries will look neutral in aggregate.

7. **Treating evaluation as a gate, not a signal**: evaluation should inform, not just block. When a regression is detected, make it easy to investigate which queries failed and why.

---

## References

- RAGAS: https://docs.ragas.io/ -- standard evaluation metrics
- DeepEval: https://docs.confident-ai.com/ -- pytest-based evaluation testing
- ARES: https://arxiv.org/abs/2311.09476 -- statistically rigorous evaluation
- LangSmith: https://docs.smith.langchain.com/evaluation -- production evaluation platform
- Langfuse: https://langfuse.com/docs/scores/overview -- open-source LLM observability
- Breck, E. et al. "The ML Test Score: A Rubric for ML Production Readiness." Google, 2017
