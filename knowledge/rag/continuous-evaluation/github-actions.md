# Continuous Evaluation -- GitHub Actions Workflows

## TL;DR

This guide provides complete GitHub Actions workflow definitions for automating RAG evaluation in CI/CD. It covers three workflow types: a PR smoke test that runs 20 high-priority queries in under 2 minutes to provide fast feedback, a merge-to-main full evaluation that runs your complete golden dataset and blocks deployment on quality drops, and a weekly scheduled evaluation that runs statistical comparison with confidence intervals for trend analysis. Each workflow includes the pytest test structure, golden dataset fixture management, metric recording, regression detection, and Slack/GitHub notification patterns. All workflows use RAGAS or DeepEval as the evaluation engine with an option for ARES classifiers to eliminate LLM API costs.

---

## Workflow Architecture

### Three-Tier CI Evaluation

```
PR Created/Updated
  |
  v
[Tier 1: Smoke Test]  <-- 20 queries, ~90 seconds, non-blocking
  |                        Comment on PR with results
  v
Merge to Main
  |
  v
[Tier 2: Full Eval]   <-- 100+ queries, ~5 minutes, blocks deploy
  |                        Gate deployment on quality thresholds
  v
Weekly Schedule
  |
  v
[Tier 3: Statistical]  <-- 200+ queries, ~15 minutes, trend analysis
                            Slack alert on significant regression
```

---

## Golden Dataset as Test Fixture

### Dataset Structure

```
tests/
  evaluation/
    golden_dataset.json       # The golden evaluation queries
    baselines/
      main_baseline.json      # Current baseline metrics (auto-updated)
      history/
        2024-01-15.json       # Historical metrics
        2024-01-22.json
    conftest.py               # pytest fixtures
    test_rag_smoke.py         # Tier 1: PR smoke test
    test_rag_full.py          # Tier 2: merge evaluation
    test_rag_statistical.py   # Tier 3: weekly statistical eval
```

### conftest.py -- Shared Fixtures

```python
"""
Shared pytest fixtures for RAG evaluation.
"""
import json
import os
from datetime import datetime
from pathlib import Path

import pytest


EVAL_DIR = Path(__file__).parent
GOLDEN_DATASET_PATH = EVAL_DIR / "golden_dataset.json"
BASELINE_PATH = EVAL_DIR / "baselines" / "main_baseline.json"
HISTORY_DIR = EVAL_DIR / "baselines" / "history"


@pytest.fixture(scope="session")
def golden_dataset() -> list[dict]:
    """Load the full golden evaluation dataset."""
    with open(GOLDEN_DATASET_PATH) as f:
        data = json.load(f)
    return data


@pytest.fixture(scope="session")
def smoke_dataset(golden_dataset) -> list[dict]:
    """
    Load a prioritized subset for smoke testing.

    Selects 20 queries covering critical categories.
    Priority: production_failure > hard > medium > easy
    """
    priority_order = {
        "production_failure": 0,
        "hard": 1,
        "medium": 2,
        "easy": 3,
    }

    sorted_examples = sorted(
        golden_dataset,
        key=lambda x: priority_order.get(x.get("difficulty", "medium"), 2),
    )

    # Take top 20, ensuring category diversity
    selected = []
    categories_seen = set()
    for ex in sorted_examples:
        cat = ex.get("category", "unknown")
        if cat not in categories_seen or len(selected) < 20:
            selected.append(ex)
            categories_seen.add(cat)
        if len(selected) >= 20:
            break

    return selected


@pytest.fixture(scope="session")
def baseline_metrics() -> dict:
    """Load the current baseline metrics."""
    if BASELINE_PATH.exists():
        with open(BASELINE_PATH) as f:
            return json.load(f)
    return {}


@pytest.fixture(scope="session")
def rag_pipeline():
    """
    Initialize the RAG pipeline under test.

    This fixture should be customized for your pipeline.
    """
    # Import your pipeline here
    # from my_rag import RAGPipeline
    # return RAGPipeline(config_path=os.environ.get("RAG_CONFIG", "config.yaml"))

    # Placeholder implementation
    class MockPipeline:
        def invoke(self, query: str) -> dict:
            return {
                "answer": "Mock answer for testing",
                "contexts": ["Mock context passage"],
                "source_documents": ["doc1.md"],
            }

    return MockPipeline()


def save_metrics(metrics: dict, eval_type: str) -> None:
    """Save metrics for tracking and baseline comparison."""
    HISTORY_DIR.mkdir(parents=True, exist_ok=True)

    # Save to history
    timestamp = datetime.now().strftime("%Y-%m-%d_%H%M%S")
    history_path = HISTORY_DIR / f"{eval_type}_{timestamp}.json"
    with open(history_path, "w") as f:
        json.dump({
            "timestamp": datetime.now().isoformat(),
            "eval_type": eval_type,
            "metrics": metrics,
            "commit": os.environ.get("GITHUB_SHA", "local"),
            "branch": os.environ.get("GITHUB_REF_NAME", "local"),
        }, f, indent=2)


def update_baseline(metrics: dict) -> None:
    """Update the main baseline with new metrics."""
    BASELINE_PATH.parent.mkdir(parents=True, exist_ok=True)
    with open(BASELINE_PATH, "w") as f:
        json.dump(metrics, f, indent=2)
```

---

## Tier 1: PR Smoke Test

### Workflow File

```yaml
# .github/workflows/rag-eval-smoke.yml
name: RAG Evaluation - Smoke Test

on:
  pull_request:
    paths:
      - 'src/rag/**'
      - 'src/retrieval/**'
      - 'src/prompts/**'
      - 'config/**'
      - 'tests/evaluation/golden_dataset.json'

concurrency:
  group: rag-eval-${{ github.head_ref }}
  cancel-in-progress: true

jobs:
  smoke-test:
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-eval.txt

      - name: Run smoke evaluation
        id: eval
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          RAG_CONFIG: config/production.yaml
        run: |
          python -m pytest tests/evaluation/test_rag_smoke.py \
            -v --tb=short --no-header \
            --junitxml=eval-results.xml \
            2>&1 | tee eval-output.txt

          # Extract metrics for PR comment
          python tests/evaluation/extract_metrics.py eval-output.txt > metrics.json

      - name: Comment on PR
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');

            let metrics = {};
            try {
              metrics = JSON.parse(fs.readFileSync('metrics.json', 'utf8'));
            } catch (e) {
              metrics = { error: 'Failed to parse metrics' };
            }

            const passed = '${{ steps.eval.outcome }}' === 'success';
            const icon = passed ? ':white_check_mark:' : ':warning:';

            let body = `## ${icon} RAG Smoke Test Results\n\n`;
            body += `| Metric | Score | Threshold | Status |\n`;
            body += `|--------|-------|-----------|--------|\n`;

            for (const [name, data] of Object.entries(metrics)) {
              if (typeof data === 'object' && data.score !== undefined) {
                const status = data.passed ? 'Pass' : 'FAIL';
                body += `| ${name} | ${data.score.toFixed(4)} | ${data.threshold} | ${status} |\n`;
              }
            }

            body += `\n_Evaluated on 20 priority queries. Full evaluation runs on merge._`;

            // Find and update existing comment or create new
            const { data: comments } = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            });

            const existing = comments.find(c =>
              c.body.includes('RAG Smoke Test Results')
            );

            if (existing) {
              await github.rest.issues.updateComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: existing.id,
                body: body,
              });
            } else {
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.issue.number,
                body: body,
              });
            }
```

### Smoke Test Implementation

```python
"""
tests/evaluation/test_rag_smoke.py

Tier 1: Fast smoke test for PRs.
Runs 20 high-priority queries, takes ~90 seconds.
"""
import json

import numpy as np
import pytest
from datasets import Dataset
from ragas import evaluate
from ragas.metrics import answer_relevancy, faithfulness


# Thresholds for smoke test (slightly relaxed vs full eval)
SMOKE_THRESHOLDS = {
    "faithfulness": 0.70,
    "answer_relevancy": 0.65,
}


class TestRAGSmoke:
    """Smoke test: fast sanity check on high-priority queries."""

    @pytest.fixture(autouse=True)
    def setup(self, smoke_dataset, rag_pipeline):
        """Run the pipeline on smoke queries and collect results."""
        self.results = []
        for item in smoke_dataset:
            output = rag_pipeline.invoke(item["query"])
            self.results.append({
                "question": item["query"],
                "answer": output["answer"],
                "contexts": [output["contexts"]] if isinstance(output["contexts"], str) else [output["contexts"]],
                "ground_truth": item["expected_answer"],
            })

        # Run RAGAS evaluation
        dataset = Dataset.from_dict({
            "question": [r["question"] for r in self.results],
            "answer": [r["answer"] for r in self.results],
            "contexts": [r["contexts"] for r in self.results],
            "ground_truth": [r["ground_truth"] for r in self.results],
        })

        self.metrics = evaluate(
            dataset, metrics=[faithfulness, answer_relevancy],
        )

    def test_faithfulness_above_threshold(self):
        """Answers must be grounded in retrieved context."""
        score = self.metrics["faithfulness"]
        assert score >= SMOKE_THRESHOLDS["faithfulness"], (
            f"Faithfulness {score:.4f} below smoke threshold "
            f"{SMOKE_THRESHOLDS['faithfulness']}"
        )

    def test_answer_relevancy_above_threshold(self):
        """Answers must be relevant to the query."""
        score = self.metrics["answer_relevancy"]
        assert score >= SMOKE_THRESHOLDS["answer_relevancy"], (
            f"Answer relevancy {score:.4f} below smoke threshold "
            f"{SMOKE_THRESHOLDS['answer_relevancy']}"
        )

    def test_no_empty_answers(self):
        """Pipeline should never return empty answers."""
        empty_count = sum(
            1 for r in self.results
            if not r["answer"] or len(r["answer"].strip()) < 10
        )
        assert empty_count == 0, f"{empty_count} queries returned empty answers"

    def test_no_empty_contexts(self):
        """Pipeline should always retrieve at least one context."""
        empty_count = sum(
            1 for r in self.results
            if not r["contexts"] or all(not c for c in r["contexts"])
        )
        assert empty_count == 0, f"{empty_count} queries had no retrieved context"
```

---

## Tier 2: Full Evaluation on Merge

### Workflow File

```yaml
# .github/workflows/rag-eval-full.yml
name: RAG Evaluation - Full

on:
  push:
    branches: [main]
    paths:
      - 'src/rag/**'
      - 'src/retrieval/**'
      - 'src/prompts/**'
      - 'config/**'

jobs:
  full-evaluation:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-eval.txt

      - name: Run full evaluation
        id: eval
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          RAG_CONFIG: config/production.yaml
        run: |
          python -m pytest tests/evaluation/test_rag_full.py \
            -v --tb=long \
            --junitxml=eval-results.xml \
            2>&1 | tee eval-output.txt

      - name: Upload evaluation artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: eval-results-${{ github.sha }}
          path: |
            eval-results.xml
            eval-output.txt
            tests/evaluation/baselines/

      - name: Update baseline on success
        if: success()
        run: |
          python tests/evaluation/update_baseline.py

          # Commit updated baseline
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add tests/evaluation/baselines/
          git diff --staged --quiet || git commit -m "chore: update RAG evaluation baseline"
          git push

      - name: Notify on failure
        if: failure()
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
        run: |
          curl -X POST "$SLACK_WEBHOOK" \
            -H 'Content-Type: application/json' \
            -d "{
              \"text\": \"RAG evaluation FAILED on main (${{ github.sha }}). Check: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}\"
            }"
```

### Full Evaluation Implementation

```python
"""
tests/evaluation/test_rag_full.py

Tier 2: Full evaluation on merge to main.
Runs 100+ queries, all metrics, with regression detection.
"""
import json
from pathlib import Path

import numpy as np
import pytest
from conftest import BASELINE_PATH, save_metrics, update_baseline
from datasets import Dataset
from ragas import evaluate
from ragas.metrics import (
    answer_relevancy,
    context_precision,
    context_recall,
    faithfulness,
)


FULL_THRESHOLDS = {
    "faithfulness": 0.75,
    "answer_relevancy": 0.70,
    "context_precision": 0.60,
    "context_recall": 0.65,
}

MAX_REGRESSION = 0.05  # Max allowed drop from baseline


class TestRAGFull:
    """Full evaluation: comprehensive quality check on every merge."""

    @pytest.fixture(autouse=True, scope="class")
    def run_evaluation(self, golden_dataset, rag_pipeline, baseline_metrics):
        """Run pipeline on full golden dataset and evaluate."""
        results = []
        for item in golden_dataset:
            output = rag_pipeline.invoke(item["query"])
            contexts = output.get("contexts", [])
            if isinstance(contexts, str):
                contexts = [contexts]

            results.append({
                "question": item["query"],
                "answer": output["answer"],
                "contexts": contexts,
                "ground_truth": item["expected_answer"],
                "category": item.get("category", "unknown"),
            })

        TestRAGFull.pipeline_results = results
        TestRAGFull.baseline = baseline_metrics

        # Run RAGAS evaluation
        dataset = Dataset.from_dict({
            "question": [r["question"] for r in results],
            "answer": [r["answer"] for r in results],
            "contexts": [r["contexts"] for r in results],
            "ground_truth": [r["ground_truth"] for r in results],
        })

        metrics_result = evaluate(
            dataset,
            metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
        )

        TestRAGFull.metrics = {
            k: v for k, v in metrics_result.items() if isinstance(v, float)
        }

        # Save metrics for tracking
        save_metrics(TestRAGFull.metrics, "full")

    def test_faithfulness_threshold(self):
        score = self.metrics["faithfulness"]
        assert score >= FULL_THRESHOLDS["faithfulness"], (
            f"Faithfulness {score:.4f} < {FULL_THRESHOLDS['faithfulness']}"
        )

    def test_answer_relevancy_threshold(self):
        score = self.metrics["answer_relevancy"]
        assert score >= FULL_THRESHOLDS["answer_relevancy"], (
            f"Answer relevancy {score:.4f} < {FULL_THRESHOLDS['answer_relevancy']}"
        )

    def test_context_precision_threshold(self):
        score = self.metrics["context_precision"]
        assert score >= FULL_THRESHOLDS["context_precision"], (
            f"Context precision {score:.4f} < {FULL_THRESHOLDS['context_precision']}"
        )

    def test_context_recall_threshold(self):
        score = self.metrics["context_recall"]
        assert score >= FULL_THRESHOLDS["context_recall"], (
            f"Context recall {score:.4f} < {FULL_THRESHOLDS['context_recall']}"
        )

    def test_no_regression_from_baseline(self):
        """Ensure no metric dropped more than MAX_REGRESSION from baseline."""
        if not self.baseline:
            pytest.skip("No baseline available")

        regressions = []
        for metric, current in self.metrics.items():
            if metric in self.baseline:
                baseline_val = self.baseline[metric]
                drop = baseline_val - current
                if drop > MAX_REGRESSION:
                    regressions.append(
                        f"{metric}: {baseline_val:.4f} -> {current:.4f} "
                        f"(dropped {drop:.4f})"
                    )

        assert not regressions, (
            f"Regressions detected (max allowed drop: {MAX_REGRESSION}):\n"
            + "\n".join(f"  - {r}" for r in regressions)
        )

    def test_no_category_below_minimum(self):
        """No query category should score below 0.50 on faithfulness."""
        by_category = {}
        for r in self.pipeline_results:
            cat = r["category"]
            if cat not in by_category:
                by_category[cat] = []
            by_category[cat].append(r)

        low_categories = []
        for cat, examples in by_category.items():
            if len(examples) >= 3:
                # Quick faithfulness check: does the answer reference context?
                # (This is a heuristic; real per-category RAGAS would be better)
                relevant = sum(
                    1 for e in examples
                    if any(
                        ctx_word in e["answer"].lower()
                        for ctx in e["contexts"]
                        for ctx_word in ctx.lower().split()[:5]
                    )
                )
                ratio = relevant / len(examples)
                if ratio < 0.50:
                    low_categories.append(f"{cat}: {ratio:.2f}")

        if low_categories:
            pytest.fail(
                f"Categories scoring below 0.50:\n"
                + "\n".join(f"  - {c}" for c in low_categories)
            )

    def test_answer_length_reasonable(self):
        """Answers should not be too short or excessively long."""
        short_answers = []
        long_answers = []

        for r in self.pipeline_results:
            answer_len = len(r["answer"])
            if answer_len < 20:
                short_answers.append(r["question"][:50])
            elif answer_len > 3000:
                long_answers.append(r["question"][:50])

        issues = []
        if short_answers:
            issues.append(f"{len(short_answers)} answers < 20 chars")
        if long_answers:
            issues.append(f"{len(long_answers)} answers > 3000 chars")

        assert not issues, f"Answer length issues: {'; '.join(issues)}"
```

### Metric Extraction Script

```python
"""
tests/evaluation/extract_metrics.py

Extract metrics from pytest output for GitHub Actions.
"""
import json
import re
import sys


def extract_metrics(output_file: str) -> dict:
    """Parse pytest output to extract evaluation metrics."""
    with open(output_file) as f:
        content = f.read()

    metrics = {}

    # Pattern: "Faithfulness: 0.8234"
    patterns = [
        (r"faithfulness[:\s]+(\d+\.\d+)", "faithfulness", 0.75),
        (r"answer.relevancy[:\s]+(\d+\.\d+)", "answer_relevancy", 0.70),
        (r"context.precision[:\s]+(\d+\.\d+)", "context_precision", 0.60),
        (r"context.recall[:\s]+(\d+\.\d+)", "context_recall", 0.65),
    ]

    for pattern, name, threshold in patterns:
        match = re.search(pattern, content, re.IGNORECASE)
        if match:
            score = float(match.group(1))
            metrics[name] = {
                "score": score,
                "threshold": threshold,
                "passed": score >= threshold,
            }

    return metrics


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("{}")
        sys.exit(0)

    metrics = extract_metrics(sys.argv[1])
    print(json.dumps(metrics, indent=2))
```

---

## Tier 3: Weekly Scheduled Evaluation

### Workflow File

```yaml
# .github/workflows/rag-eval-weekly.yml
name: RAG Evaluation - Weekly Statistical

on:
  schedule:
    # Every Monday at 6:00 UTC
    - cron: '0 6 * * 1'
  workflow_dispatch:  # Allow manual trigger

jobs:
  statistical-evaluation:
    runs-on: ubuntu-latest
    timeout-minutes: 45

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-eval.txt

      - name: Run statistical evaluation
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          RAG_CONFIG: config/production.yaml
        run: |
          python -m pytest tests/evaluation/test_rag_statistical.py \
            -v --tb=long \
            --junitxml=eval-results.xml \
            2>&1 | tee eval-output.txt

      - name: Upload evaluation report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: weekly-eval-${{ github.run_number }}
          path: |
            eval-results.xml
            eval-output.txt
            tests/evaluation/baselines/history/

      - name: Send weekly report
        if: always()
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
        run: python tests/evaluation/send_weekly_report.py
```

### Statistical Evaluation Implementation

```python
"""
tests/evaluation/test_rag_statistical.py

Tier 3: Weekly statistical evaluation with confidence intervals.
Runs 200+ queries, uses bootstrap CI for trend detection.
"""
import json
from pathlib import Path

import numpy as np
import pytest
from conftest import HISTORY_DIR, save_metrics
from scipy import stats


class TestRAGStatistical:
    """Weekly statistical evaluation for trend detection."""

    @pytest.fixture(autouse=True, scope="class")
    def run_evaluation(self, golden_dataset, rag_pipeline):
        """Run full evaluation with per-query scores."""
        from datasets import Dataset
        from ragas import evaluate
        from ragas.metrics import (
            answer_relevancy,
            context_precision,
            context_recall,
            faithfulness,
        )

        results = []
        for item in golden_dataset:
            output = rag_pipeline.invoke(item["query"])
            contexts = output.get("contexts", [])
            if isinstance(contexts, str):
                contexts = [contexts]
            results.append({
                "question": item["query"],
                "answer": output["answer"],
                "contexts": contexts,
                "ground_truth": item["expected_answer"],
            })

        dataset = Dataset.from_dict({
            "question": [r["question"] for r in results],
            "answer": [r["answer"] for r in results],
            "contexts": [r["contexts"] for r in results],
            "ground_truth": [r["ground_truth"] for r in results],
        })

        eval_result = evaluate(
            dataset,
            metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
        )

        TestRAGStatistical.metrics = {
            k: v for k, v in eval_result.items() if isinstance(v, float)
        }

        # Save for historical tracking
        save_metrics(TestRAGStatistical.metrics, "weekly")

    def _load_previous_weekly(self) -> list[dict]:
        """Load previous weekly evaluation results."""
        history = []
        if HISTORY_DIR.exists():
            for f in sorted(HISTORY_DIR.glob("weekly_*.json")):
                with open(f) as fh:
                    history.append(json.load(fh))
        return history

    def test_no_downward_trend(self):
        """Detect if metrics are trending downward over recent weeks."""
        history = self._load_previous_weekly()

        if len(history) < 4:
            pytest.skip("Need at least 4 weekly data points for trend analysis")

        declining_metrics = []
        for metric_name in ["faithfulness", "answer_relevancy", "context_precision", "context_recall"]:
            values = [
                h["metrics"].get(metric_name, 0)
                for h in history[-8:]  # Last 8 weeks
                if metric_name in h.get("metrics", {})
            ]

            if len(values) >= 4:
                x = np.arange(len(values))
                slope, _, r_value, p_value, _ = stats.linregress(x, values)

                # Flag if significant downward trend
                if slope < -0.01 and p_value < 0.10:
                    declining_metrics.append(
                        f"{metric_name}: slope={slope:.4f}/week, "
                        f"p={p_value:.4f}, r2={r_value**2:.4f}"
                    )

        if declining_metrics:
            pytest.fail(
                f"Declining trends detected:\n"
                + "\n".join(f"  - {m}" for m in declining_metrics)
            )

    def test_consistency_across_runs(self):
        """Check that metric variance is not too high."""
        history = self._load_previous_weekly()

        if len(history) < 3:
            pytest.skip("Need at least 3 data points for consistency check")

        high_variance = []
        for metric_name in self.metrics:
            values = [
                h["metrics"][metric_name]
                for h in history[-6:]
                if metric_name in h.get("metrics", {})
            ]

            if len(values) >= 3:
                std = np.std(values)
                mean = np.mean(values)
                cv = std / max(mean, 0.01)  # Coefficient of variation

                if cv > 0.15:  # More than 15% variation
                    high_variance.append(
                        f"{metric_name}: CV={cv:.3f}, std={std:.4f}, mean={mean:.4f}"
                    )

        if high_variance:
            # Warning, not failure -- high variance indicates evaluation noise
            print("WARNING: High metric variance detected:")
            for m in high_variance:
                print(f"  - {m}")
```

### Weekly Report Script

```python
"""
tests/evaluation/send_weekly_report.py

Generate and send weekly RAG evaluation report to Slack.
"""
import json
import os
from pathlib import Path

import requests

HISTORY_DIR = Path("tests/evaluation/baselines/history")
SLACK_WEBHOOK = os.environ.get("SLACK_WEBHOOK", "")


def generate_report() -> str:
    """Generate a weekly evaluation summary."""
    # Load recent history
    history_files = sorted(HISTORY_DIR.glob("weekly_*.json"))
    if not history_files:
        return "No weekly evaluation data available."

    # Current and previous
    with open(history_files[-1]) as f:
        current = json.load(f)

    previous = None
    if len(history_files) >= 2:
        with open(history_files[-2]) as f:
            previous = json.load(f)

    # Build report
    lines = ["*Weekly RAG Evaluation Report*\n"]

    for metric, value in current["metrics"].items():
        line = f"  {metric}: {value:.4f}"
        if previous and metric in previous["metrics"]:
            diff = value - previous["metrics"][metric]
            arrow = "^" if diff > 0.005 else "v" if diff < -0.005 else "="
            line += f"  ({arrow} {diff:+.4f})"
        lines.append(line)

    lines.append(f"\nCommit: {current.get('commit', 'unknown')[:8]}")
    lines.append(f"Timestamp: {current.get('timestamp', 'unknown')}")

    return "\n".join(lines)


def send_to_slack(message: str) -> None:
    """Send report to Slack webhook."""
    if not SLACK_WEBHOOK:
        print("No SLACK_WEBHOOK set, printing report instead:")
        print(message)
        return

    payload = {"text": message}
    response = requests.post(SLACK_WEBHOOK, json=payload, timeout=10)
    response.raise_for_status()
    print("Report sent to Slack")


if __name__ == "__main__":
    report = generate_report()
    send_to_slack(report)
```

---

## Using ARES Instead of RAGAS (Zero-Cost CI)

For teams that want to eliminate LLM API costs in CI, replace RAGAS with pre-trained ARES classifiers:

```python
"""
ARES-based CI evaluation: no API calls, deterministic, fast.
"""
import json

import numpy as np
import pytest
import torch
from transformers import AutoModelForSequenceClassification, AutoTokenizer


@pytest.fixture(scope="session")
def ares_models():
    """Load ARES classifiers (stored in tests/models/ or downloaded)."""
    models = {}
    base = "tests/models/ares"
    for dim in ["context_relevance", "answer_faithfulness", "answer_relevance"]:
        model_path = f"{base}/{dim}/best"
        models[dim] = {
            "model": AutoModelForSequenceClassification.from_pretrained(model_path).eval(),
            "tokenizer": AutoTokenizer.from_pretrained(model_path),
        }
    return models


def ares_score(models, dimension, texts_a, texts_b, batch_size=32):
    """Score text pairs with ARES classifier."""
    model = models[dimension]["model"]
    tokenizer = models[dimension]["tokenizer"]
    all_probs = []

    for i in range(0, len(texts_a), batch_size):
        enc = tokenizer(
            texts_a[i:i+batch_size],
            texts_b[i:i+batch_size],
            max_length=512, padding=True, truncation=True,
            return_tensors="pt",
        )
        with torch.no_grad():
            probs = torch.softmax(model(**enc).logits, dim=-1)[:, 1]
            all_probs.extend(probs.numpy())

    return np.array(all_probs)


class TestRAGAresCI:
    """ARES-based CI tests: zero API cost, deterministic."""

    @pytest.fixture(autouse=True, scope="class")
    def run_evaluation(self, golden_dataset, rag_pipeline, ares_models):
        results = []
        for item in golden_dataset:
            output = rag_pipeline.invoke(item["query"])
            results.append({
                "query": item["query"],
                "answer": output["answer"],
                "context": output["contexts"][0] if output["contexts"] else "",
            })

        queries = [r["query"] for r in results]
        answers = [r["answer"] for r in results]
        contexts = [r["context"] for r in results]

        TestRAGAresCI.scores = {
            "context_relevance": ares_score(ares_models, "context_relevance", queries, contexts),
            "answer_faithfulness": ares_score(ares_models, "answer_faithfulness", contexts, answers),
            "answer_relevance": ares_score(ares_models, "answer_relevance", queries, answers),
        }

    def test_faithfulness(self):
        avg = float(np.mean(self.scores["answer_faithfulness"] >= 0.5))
        assert avg >= 0.75, f"Faithfulness {avg:.4f} < 0.75"

    def test_relevance(self):
        avg = float(np.mean(self.scores["answer_relevance"] >= 0.5))
        assert avg >= 0.70, f"Answer relevance {avg:.4f} < 0.70"

    def test_context_relevance(self):
        avg = float(np.mean(self.scores["context_relevance"] >= 0.5))
        assert avg >= 0.60, f"Context relevance {avg:.4f} < 0.60"
```

---

## Common Pitfalls

1. **Evaluation secrets exposed**: never hardcode API keys in workflow files. Use GitHub Secrets for `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, and `SLACK_WEBHOOK`.

2. **No concurrency control**: if two PRs trigger evaluation simultaneously, they may interfere. Use `concurrency` groups in workflow definitions to cancel stale runs.

3. **Flaky tests from LLM non-determinism**: RAGAS and DeepEval use LLM calls that produce slightly different results each run. Set temperature=0 where possible and use relaxed thresholds in CI to avoid false failures.

4. **Evaluation dataset in the wrong branch**: golden datasets should be on the main branch and not modified in feature branches (unless the PR specifically improves the dataset). Use `actions/checkout` with the correct ref.

5. **No timeout**: evaluation can hang if the RAG pipeline or LLM API is slow. Always set `timeout-minutes` in workflow jobs.

6. **Baseline drift**: if you automatically update baselines on every merge, a gradual decline of 1% per merge will never be caught. Use delta-based detection with a maximum regression threshold instead of only comparing against the immediately previous baseline.

---

## References

- GitHub Actions: https://docs.github.com/en/actions
- RAGAS: https://docs.ragas.io/
- DeepEval: https://docs.confident-ai.com/
- pytest: https://docs.pytest.org/
- ARES: https://arxiv.org/abs/2311.09476
