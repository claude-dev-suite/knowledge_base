# Giskard RAGET -- CI Integration and Regression Testing

## TL;DR

This guide covers integrating Giskard RAGET into your CI/CD pipeline for automated RAG regression testing. It provides complete pytest test files that run RAGET component-level evaluation (retriever and generator scored independently), regression threshold configuration that catches quality drops without false alarms, GitHub Actions workflow definitions for PR checks and merge gates, and strategies for managing generated test sets across branches. The key advantage of RAGET in CI is component-level scoring: when a test fails, you immediately know whether the retriever or generator regressed, eliminating diagnosis time.

---

## pytest Integration

### Project Structure

```
your-rag-project/
  src/
    rag/
      pipeline.py        # Your RAG pipeline
      retriever.py       # Retriever component
      generator.py       # Generator component
  tests/
    evaluation/
      conftest.py              # Shared fixtures
      test_rag_components.py   # Component-level RAGET tests
      test_rag_regression.py   # Regression detection tests
      fixtures/
        testset.json           # Pre-generated RAGET test set
        baseline.json          # Baseline component scores
  .github/
    workflows/
      rag-eval.yml       # GitHub Actions workflow
```

### conftest.py -- Shared Fixtures

```python
"""
tests/evaluation/conftest.py

Shared fixtures for RAGET-based evaluation tests.
"""
import json
import os
from pathlib import Path

import pandas as pd
import pytest
from giskard.rag import KnowledgeBase, QATestset


FIXTURES_DIR = Path(__file__).parent / "fixtures"
TESTSET_PATH = FIXTURES_DIR / "testset.json"
BASELINE_PATH = FIXTURES_DIR / "baseline.json"
HISTORY_DIR = FIXTURES_DIR / "history"


@pytest.fixture(scope="session")
def testset() -> QATestset:
    """
    Load the pre-generated RAGET test set.

    The test set is generated once and committed to the repo.
    Regenerate only when the knowledge base changes significantly.
    """
    if not TESTSET_PATH.exists():
        pytest.skip(
            "No test set found. Generate one with: "
            "python scripts/generate_testset.py"
        )
    return QATestset.load(str(TESTSET_PATH))


@pytest.fixture(scope="session")
def testset_df(testset) -> pd.DataFrame:
    """Test set as a DataFrame for analysis."""
    return testset.to_pandas()


@pytest.fixture(scope="session")
def knowledge_base() -> KnowledgeBase:
    """
    Load the knowledge base used for evaluation.

    This should match the actual documents in your RAG pipeline.
    """
    docs_path = Path(os.environ.get("DOCS_PATH", "docs/"))
    records = []
    for md_file in docs_path.rglob("*.md"):
        content = md_file.read_text(encoding="utf-8")
        if len(content) > 100:
            records.append({
                "text": content,
                "source": str(md_file),
            })

    if not records:
        pytest.skip("No documents found for knowledge base")

    df = pd.DataFrame(records)
    return KnowledgeBase.from_pandas(df)


@pytest.fixture(scope="session")
def baseline() -> dict:
    """Load baseline scores for regression detection."""
    if BASELINE_PATH.exists():
        with open(BASELINE_PATH) as f:
            return json.load(f)
    return {}


@pytest.fixture(scope="session")
def rag_pipeline():
    """
    Initialize your RAG pipeline.

    Replace this with your actual pipeline initialization.
    """
    # from src.rag.pipeline import RAGPipeline
    # return RAGPipeline(config="config/test.yaml")

    class MockPipeline:
        def __init__(self):
            pass

        def retrieve(self, query: str) -> list[str]:
            return ["Retrieved context"]

        def generate(self, query: str, contexts: list[str]) -> str:
            return "Generated answer"

        def invoke(self, query: str) -> dict:
            contexts = self.retrieve(query)
            answer = self.generate(query, contexts)
            return {"answer": answer, "contexts": contexts}

    return MockPipeline()


@pytest.fixture(scope="session")
def answer_fn(rag_pipeline):
    """Wrap pipeline as answer function for RAGET."""
    def fn(question: str, history: list | None = None) -> str:
        result = rag_pipeline.invoke(question)
        return result["answer"]
    return fn
```

### Component-Level Test File

```python
"""
tests/evaluation/test_rag_components.py

RAGET component-level evaluation tests.
Tests retriever and generator independently.
"""
import json
from datetime import datetime
from pathlib import Path

import numpy as np
import pytest
from giskard.rag import evaluate as raget_evaluate


# Component-level thresholds
RETRIEVER_THRESHOLDS = {
    "context_precision": 0.55,
    "context_recall": 0.60,
}

GENERATOR_THRESHOLDS = {
    "faithfulness": 0.70,
    "answer_relevancy": 0.65,
}

# Per question-type minimum thresholds (lower than overall)
QUESTION_TYPE_MINIMUMS = {
    "simple": {"faithfulness": 0.75, "context_recall": 0.70},
    "complex": {"faithfulness": 0.60, "context_recall": 0.50},
    "distracting": {"faithfulness": 0.65, "context_precision": 0.45},
    "conversational": {"faithfulness": 0.60, "answer_relevancy": 0.55},
}


class TestRAGComponents:
    """Component-level RAGET evaluation."""

    @pytest.fixture(autouse=True, scope="class")
    def run_raget_evaluation(self, answer_fn, testset, knowledge_base):
        """Run RAGET evaluation once for all tests in the class."""
        TestRAGComponents.report = raget_evaluate(
            answer_fn=answer_fn,
            testset=testset,
            knowledge_base=knowledge_base,
        )

        # Save results for history tracking
        history_dir = Path("tests/evaluation/fixtures/history")
        history_dir.mkdir(parents=True, exist_ok=True)
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")

        results = {
            "timestamp": datetime.now().isoformat(),
            "retriever": dict(self.report.retriever_metrics),
            "generator": dict(self.report.generator_metrics),
        }

        if hasattr(self.report, "metrics_by_question_type"):
            results["by_question_type"] = {
                qt: dict(metrics)
                for qt, metrics in self.report.metrics_by_question_type.items()
            }

        with open(history_dir / f"raget_{timestamp}.json", "w") as f:
            json.dump(results, f, indent=2)

    # --- Retriever Tests ---

    def test_retriever_context_precision(self):
        """Retriever must return relevant documents in top positions."""
        score = self.report.retriever_metrics.get("context_precision", 0)
        threshold = RETRIEVER_THRESHOLDS["context_precision"]
        assert score >= threshold, (
            f"Retriever context precision {score:.4f} < {threshold}. "
            "The retriever returns too many irrelevant documents. "
            "Consider adding a reranker or improving embedding quality."
        )

    def test_retriever_context_recall(self):
        """Retriever must find all relevant documents."""
        score = self.report.retriever_metrics.get("context_recall", 0)
        threshold = RETRIEVER_THRESHOLDS["context_recall"]
        assert score >= threshold, (
            f"Retriever context recall {score:.4f} < {threshold}. "
            "The retriever misses relevant documents. "
            "Consider increasing K, using hybrid search, or checking "
            "that documents are properly indexed."
        )

    # --- Generator Tests ---

    def test_generator_faithfulness(self):
        """Generator must ground answers in retrieved context."""
        score = self.report.generator_metrics.get("faithfulness", 0)
        threshold = GENERATOR_THRESHOLDS["faithfulness"]
        assert score >= threshold, (
            f"Generator faithfulness {score:.4f} < {threshold}. "
            "The generator produces claims not supported by context. "
            "Strengthen grounding instructions in the system prompt."
        )

    def test_generator_answer_relevancy(self):
        """Generator must produce answers that address the query."""
        score = self.report.generator_metrics.get("answer_relevancy", 0)
        threshold = GENERATOR_THRESHOLDS["answer_relevancy"]
        assert score >= threshold, (
            f"Generator answer relevancy {score:.4f} < {threshold}. "
            "Answers do not address the user's question. "
            "Review the generation prompt for clarity."
        )

    # --- Per Question-Type Tests ---

    def test_simple_questions_baseline(self):
        """Simple questions must meet minimum quality."""
        if not hasattr(self.report, "metrics_by_question_type"):
            pytest.skip("Per-type metrics not available")

        simple_metrics = self.report.metrics_by_question_type.get("simple", {})
        minimums = QUESTION_TYPE_MINIMUMS["simple"]

        failures = []
        for metric, threshold in minimums.items():
            score = simple_metrics.get(metric, 0)
            if score < threshold:
                failures.append(f"{metric}: {score:.4f} < {threshold}")

        assert not failures, (
            "Simple questions below minimum:\n"
            + "\n".join(f"  - {f}" for f in failures)
        )

    def test_complex_questions_baseline(self):
        """Complex questions should meet relaxed minimums."""
        if not hasattr(self.report, "metrics_by_question_type"):
            pytest.skip("Per-type metrics not available")

        complex_metrics = self.report.metrics_by_question_type.get("complex", {})
        minimums = QUESTION_TYPE_MINIMUMS["complex"]

        failures = []
        for metric, threshold in minimums.items():
            score = complex_metrics.get(metric, 0)
            if score < threshold:
                failures.append(f"{metric}: {score:.4f} < {threshold}")

        assert not failures, (
            "Complex questions below minimum:\n"
            + "\n".join(f"  - {f}" for f in failures)
        )

    def test_no_zero_score_questions(self):
        """No question type should score 0 on any metric."""
        if not hasattr(self.report, "metrics_by_question_type"):
            pytest.skip("Per-type metrics not available")

        zeros = []
        for qt, metrics in self.report.metrics_by_question_type.items():
            for metric, score in metrics.items():
                if score < 0.01:
                    zeros.append(f"{qt}/{metric} = {score:.4f}")

        assert not zeros, (
            "Zero-score metrics detected (possible pipeline failure):\n"
            + "\n".join(f"  - {z}" for z in zeros)
        )
```

### Regression Detection Test File

```python
"""
tests/evaluation/test_rag_regression.py

Regression detection using RAGET component scores.
Compares current evaluation against a stored baseline.
"""
import json
from pathlib import Path

import numpy as np
import pytest


BASELINE_PATH = Path("tests/evaluation/fixtures/baseline.json")
HISTORY_DIR = Path("tests/evaluation/fixtures/history")

# Maximum allowed regression per metric
MAX_REGRESSION = {
    "retriever": {
        "context_precision": 0.05,
        "context_recall": 0.05,
    },
    "generator": {
        "faithfulness": 0.04,
        "answer_relevancy": 0.04,
    },
}


class TestRAGRegression:
    """Regression detection comparing against baseline."""

    @pytest.fixture(autouse=True)
    def load_current_results(self):
        """Load the most recent RAGET evaluation results."""
        history_files = sorted(HISTORY_DIR.glob("raget_*.json"))
        if not history_files:
            pytest.skip("No evaluation results found. Run component tests first.")

        with open(history_files[-1]) as f:
            self.current = json.load(f)

    @pytest.fixture(autouse=True)
    def load_baseline(self):
        """Load baseline scores."""
        if not BASELINE_PATH.exists():
            pytest.skip("No baseline found. Run: python scripts/set_baseline.py")

        with open(BASELINE_PATH) as f:
            self.baseline = json.load(f)

    def test_retriever_no_regression(self):
        """Retriever metrics must not drop more than MAX_REGRESSION."""
        regressions = []

        for metric, max_drop in MAX_REGRESSION["retriever"].items():
            baseline_val = self.baseline.get("retriever", {}).get(metric)
            current_val = self.current.get("retriever", {}).get(metric)

            if baseline_val is None or current_val is None:
                continue

            drop = baseline_val - current_val
            if drop > max_drop:
                regressions.append(
                    f"{metric}: {baseline_val:.4f} -> {current_val:.4f} "
                    f"(dropped {drop:.4f}, max allowed: {max_drop})"
                )

        assert not regressions, (
            "Retriever regression detected:\n"
            + "\n".join(f"  - {r}" for r in regressions)
        )

    def test_generator_no_regression(self):
        """Generator metrics must not drop more than MAX_REGRESSION."""
        regressions = []

        for metric, max_drop in MAX_REGRESSION["generator"].items():
            baseline_val = self.baseline.get("generator", {}).get(metric)
            current_val = self.current.get("generator", {}).get(metric)

            if baseline_val is None or current_val is None:
                continue

            drop = baseline_val - current_val
            if drop > max_drop:
                regressions.append(
                    f"{metric}: {baseline_val:.4f} -> {current_val:.4f} "
                    f"(dropped {drop:.4f}, max allowed: {max_drop})"
                )

        assert not regressions, (
            "Generator regression detected:\n"
            + "\n".join(f"  - {r}" for r in regressions)
        )

    def test_no_question_type_regression(self):
        """No question type should regress more than 10%."""
        if "by_question_type" not in self.baseline:
            pytest.skip("No per-type baseline available")
        if "by_question_type" not in self.current:
            pytest.skip("No per-type current results available")

        regressions = []

        for qt in self.baseline["by_question_type"]:
            if qt not in self.current.get("by_question_type", {}):
                continue

            base_metrics = self.baseline["by_question_type"][qt]
            curr_metrics = self.current["by_question_type"][qt]

            for metric in base_metrics:
                if metric not in curr_metrics:
                    continue
                base_val = base_metrics[metric]
                curr_val = curr_metrics[metric]
                drop = base_val - curr_val

                if drop > 0.10:
                    regressions.append(
                        f"{qt}/{metric}: {base_val:.4f} -> {curr_val:.4f} "
                        f"(dropped {drop:.4f})"
                    )

        assert not regressions, (
            "Question-type regression detected (>10% drop):\n"
            + "\n".join(f"  - {r}" for r in regressions)
        )


class TestRAGTrend:
    """Trend analysis across multiple evaluation runs."""

    def test_no_downward_trend(self):
        """Detect sustained quality decline over recent runs."""
        from scipy import stats

        history_files = sorted(HISTORY_DIR.glob("raget_*.json"))
        if len(history_files) < 5:
            pytest.skip("Need at least 5 historical runs for trend analysis")

        # Load last 10 runs
        history = []
        for f in history_files[-10:]:
            with open(f) as fh:
                history.append(json.load(fh))

        declining = []
        for component in ["retriever", "generator"]:
            for metric in history[0].get(component, {}):
                values = [
                    h[component][metric]
                    for h in history
                    if metric in h.get(component, {})
                ]
                if len(values) < 5:
                    continue

                x = np.arange(len(values))
                slope, _, _, p_value, _ = stats.linregress(x, values)

                if slope < -0.005 and p_value < 0.10:
                    declining.append(
                        f"{component}/{metric}: slope={slope:.4f}/run, "
                        f"p={p_value:.4f}"
                    )

        if declining:
            # Warning rather than failure -- trends need human judgment
            import warnings
            warnings.warn(
                "Quality trend declining:\n"
                + "\n".join(f"  - {d}" for d in declining),
                UserWarning,
            )
```

---

## GitHub Actions Workflow

```yaml
# .github/workflows/rag-eval.yml
name: RAG Evaluation (RAGET)

on:
  pull_request:
    paths:
      - 'src/rag/**'
      - 'src/retrieval/**'
      - 'src/prompts/**'
      - 'config/**'
      - 'tests/evaluation/**'
  push:
    branches: [main]
    paths:
      - 'src/rag/**'

concurrency:
  group: rag-eval-${{ github.head_ref || github.sha }}
  cancel-in-progress: true

jobs:
  raget-evaluation:
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
          pip install "giskard[llm]>=2.0"

      - name: Run RAGET component evaluation
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          DOCS_PATH: docs/
        run: |
          python -m pytest tests/evaluation/test_rag_components.py \
            -v --tb=long \
            --junitxml=raget-results.xml \
            2>&1 | tee raget-output.txt

      - name: Run regression detection
        if: always()
        run: |
          python -m pytest tests/evaluation/test_rag_regression.py \
            -v --tb=long \
            --junitxml=regression-results.xml \
            2>&1 | tee regression-output.txt

      - name: Upload evaluation artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: raget-results-${{ github.sha }}
          path: |
            raget-results.xml
            regression-results.xml
            raget-output.txt
            regression-output.txt
            tests/evaluation/fixtures/history/

      - name: Comment on PR with results
        if: github.event_name == 'pull_request' && always()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            let output = '';
            try {
              output = fs.readFileSync('raget-output.txt', 'utf8');
            } catch (e) {
              output = 'Failed to read evaluation output';
            }

            // Extract key metrics from output
            const lines = output.split('\n');
            const metricLines = lines.filter(l =>
              l.includes('context_precision') ||
              l.includes('context_recall') ||
              l.includes('faithfulness') ||
              l.includes('answer_relevancy') ||
              l.includes('PASSED') ||
              l.includes('FAILED')
            ).slice(0, 15);

            const passed = !output.includes('FAILED');
            const icon = passed ? ':white_check_mark:' : ':x:';

            let body = `## ${icon} RAGET Component Evaluation\n\n`;
            body += '```\n' + metricLines.join('\n') + '\n```\n\n';

            if (!passed) {
              body += '**Action required**: Fix the failing component before merging.\n';
              body += 'Check the full output in the workflow artifacts.\n';
            }

            // Find or update existing comment
            const { data: comments } = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            });

            const existing = comments.find(c =>
              c.body.includes('RAGET Component Evaluation')
            );

            const commentBody = { body };

            if (existing) {
              await github.rest.issues.updateComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: existing.id,
                ...commentBody,
              });
            } else {
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.issue.number,
                ...commentBody,
              });
            }

      - name: Update baseline on main
        if: github.ref == 'refs/heads/main' && success()
        run: |
          # Copy latest results as the new baseline
          latest=$(ls -t tests/evaluation/fixtures/history/raget_*.json | head -1)
          if [ -n "$latest" ]; then
            cp "$latest" tests/evaluation/fixtures/baseline.json
            git config user.name "github-actions[bot]"
            git config user.email "github-actions[bot]@users.noreply.github.com"
            git add tests/evaluation/fixtures/baseline.json
            git add tests/evaluation/fixtures/history/
            git diff --staged --quiet || git commit -m "chore: update RAGET baseline"
            git push
          fi
```

---

## Baseline Management

### Setting the Initial Baseline

```python
"""
scripts/set_baseline.py

Set the initial baseline from a clean evaluation run.
"""
import json
import subprocess
import sys
from pathlib import Path


def set_baseline():
    """Run evaluation and save results as baseline."""
    # Run the evaluation
    result = subprocess.run(
        [sys.executable, "-m", "pytest",
         "tests/evaluation/test_rag_components.py",
         "-v", "--tb=short"],
        capture_output=True,
        text=True,
    )

    print(result.stdout)
    if result.returncode != 0:
        print("Evaluation failed. Fix issues before setting baseline.")
        print(result.stderr)
        sys.exit(1)

    # Find the latest results
    history_dir = Path("tests/evaluation/fixtures/history")
    history_files = sorted(history_dir.glob("raget_*.json"))

    if not history_files:
        print("No evaluation results found")
        sys.exit(1)

    latest = history_files[-1]
    baseline_path = Path("tests/evaluation/fixtures/baseline.json")

    # Copy to baseline
    with open(latest) as f:
        data = json.load(f)

    with open(baseline_path, "w") as f:
        json.dump(data, f, indent=2)

    print(f"\nBaseline set from {latest.name}:")
    print(f"  Retriever: {json.dumps(data.get('retriever', {}), indent=4)}")
    print(f"  Generator: {json.dumps(data.get('generator', {}), indent=4)}")
    print(f"\nSaved to {baseline_path}")


if __name__ == "__main__":
    set_baseline()
```

### Regenerating Test Sets

```python
"""
scripts/generate_testset.py

Generate or regenerate the RAGET test set.

Run this when your knowledge base changes significantly.
"""
import os
from pathlib import Path

import giskard
import pandas as pd
from giskard.rag import KnowledgeBase, generate_testset


def main():
    giskard.llm.set_default_client("anthropic")

    # Load knowledge base
    docs_path = Path(os.environ.get("DOCS_PATH", "docs/"))
    records = []
    for md_file in docs_path.rglob("*.md"):
        content = md_file.read_text(encoding="utf-8")
        if len(content) > 100:
            records.append({
                "text": content,
                "source": str(md_file.relative_to(docs_path)),
            })

    print(f"Loaded {len(records)} documents")

    kb = KnowledgeBase.from_pandas(pd.DataFrame(records))

    # Generate testset
    testset = generate_testset(
        kb,
        num_questions=150,
        language="en",
        agent_description=(
            "A RAG system that answers technical questions "
            "about our product documentation."
        ),
    )

    # Save
    output_path = "tests/evaluation/fixtures/testset.json"
    testset.save(output_path)

    print(f"\nGenerated {len(testset)} questions")
    print(f"Saved to {output_path}")

    # Summary by type
    df = testset.to_pandas()
    print(f"\nBy question type:")
    for qt, count in df["question_type"].value_counts().items():
        print(f"  {qt}: {count}")


if __name__ == "__main__":
    main()
```

---

## Common Pitfalls

1. **Running RAGET evaluation without a pre-generated test set**: generating test sets in CI is slow (5-15 min) and expensive (LLM calls). Pre-generate, commit the test set, and load it in CI.

2. **Not separating component thresholds**: using the same threshold for retriever and generator metrics hides which component is responsible. Set independent thresholds for each component.

3. **Baseline staleness**: if the baseline is never updated, gradual improvements will create an increasingly large gap, making regression thresholds meaningless. Update the baseline on every successful merge to main.

4. **Not tracking per-question-type metrics**: aggregate metrics can mask category-specific regressions. A prompt change that improves simple questions but breaks complex ones will look neutral in aggregate.

5. **Missing the CI timeout**: RAGET evaluation with LLM judges can take 10-20 minutes for 100+ questions. Set adequate timeouts (30+ minutes) and consider using ARES classifiers for faster scoring.

6. **Not pinning the giskard version**: RAGET's question generation and scoring can change between versions, causing spurious test failures. Pin the version in requirements-eval.txt.

---

## References

- Giskard RAGET: https://docs.giskard.ai/en/latest/open_source/testset_generation/
- Giskard RAG Evaluation: https://docs.giskard.ai/en/latest/open_source/testset_generation/rag_evaluation/
- pytest: https://docs.pytest.org/
- GitHub Actions: https://docs.github.com/en/actions
