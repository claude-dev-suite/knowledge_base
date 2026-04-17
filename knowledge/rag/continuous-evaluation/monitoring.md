# Continuous Evaluation -- Production Monitoring

## TL;DR

Production RAG monitoring goes beyond CI/CD evaluation by continuously scoring live traffic, detecting quality drift in real time, and alerting when metrics degrade. This guide covers three monitoring approaches: LangSmith scheduled evaluation runs for teams using LangChain, Langfuse scoring and dashboards for open-source observability, and custom monitoring pipelines using ARES classifiers for zero-cost scoring. For each approach, it provides the instrumentation code, scheduled evaluation patterns, alerting rules, and dashboard configurations needed to maintain RAG quality in production.

---

## Why Production Monitoring Differs from CI/CD Evaluation

CI/CD evaluation tests against a fixed golden dataset before deployment. Production monitoring catches issues that golden datasets miss:

| Issue | CI/CD Catches? | Production Monitoring Catches? |
|---|---|---|
| Code regression | Yes | Yes (but after deployment) |
| Prompt regression | Yes | Yes |
| Corpus drift (new docs change retrieval) | No | Yes |
| Embedding provider silent update | No | Yes |
| LLM version change (API provider updates) | No | Yes |
| Query distribution shift (users ask new types) | No | Yes |
| Vector store index corruption | No | Yes |
| Latency degradation | Sometimes | Yes |
| Cascading failures (retriever timeout -> empty context) | Sometimes | Yes |

Production monitoring samples live traffic and scores it continuously, creating a time-series of quality metrics that reveals drift invisible to static test suites.

---

## Approach 1: LangSmith Scheduled Evaluation

LangSmith (by LangChain) provides hosted tracing, evaluation, and dashboards for LLM applications.

### Instrumenting Your Pipeline

```python
"""
LangSmith instrumentation for RAG pipeline monitoring.
"""
import os

from langsmith import Client, traceable
from langsmith.evaluation import evaluate as ls_evaluate
from langsmith.schemas import Example, Run


# Set environment variables
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-langsmith-api-key"
os.environ["LANGCHAIN_PROJECT"] = "rag-production"

client = Client()


@traceable(name="rag-pipeline", run_type="chain")
def rag_pipeline(query: str) -> dict:
    """
    Your RAG pipeline wrapped with LangSmith tracing.

    The @traceable decorator automatically logs:
    - Input query
    - Output answer
    - Intermediate steps (retrieval, generation)
    - Latency
    - Token usage
    """
    # Step 1: Retrieve
    contexts = retrieve_documents(query)

    # Step 2: Generate
    answer = generate_answer(query, contexts)

    return {
        "answer": answer,
        "contexts": contexts,
        "num_contexts": len(contexts),
    }


@traceable(name="retrieve", run_type="retriever")
def retrieve_documents(query: str) -> list[str]:
    """Retrieval step with tracing."""
    # Your retrieval logic here
    # results = vector_store.similarity_search(query, k=5)
    # return [doc.page_content for doc in results]
    return ["Retrieved context placeholder"]


@traceable(name="generate", run_type="llm")
def generate_answer(query: str, contexts: list[str]) -> str:
    """Generation step with tracing."""
    # Your generation logic here
    # response = llm.invoke(prompt.format(query=query, context=context))
    # return response.content
    return "Generated answer placeholder"
```

### Scheduled Evaluation with LangSmith

```python
"""
LangSmith scheduled evaluation: score production traces periodically.
"""
import json
from datetime import datetime, timedelta

from langsmith import Client
from langsmith.evaluation import evaluate
from langsmith.schemas import Example, Run


client = Client()


# Define custom evaluators
def faithfulness_evaluator(run: Run, example: Example) -> dict:
    """
    Score faithfulness of a production RAG response.

    Uses the traced retrieval context and generated answer
    from the production run.
    """
    import anthropic

    ai_client = anthropic.Anthropic()

    # Extract from the traced run
    answer = run.outputs.get("answer", "")
    contexts = run.outputs.get("contexts", [])
    context_text = "\n".join(contexts[:3])

    response = ai_client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=50,
        messages=[{
            "role": "user",
            "content": (
                "Is this answer fully supported by the context? "
                "Score 0.0 to 1.0.\n\n"
                f"Context:\n{context_text[:2000]}\n\n"
                f"Answer:\n{answer}\n\n"
                "Output ONLY a number between 0.0 and 1.0:"
            ),
        }],
    )

    try:
        score = float(response.content[0].text.strip())
        score = max(0.0, min(1.0, score))
    except (ValueError, IndexError):
        score = 0.0

    return {"key": "faithfulness", "score": score}


def relevancy_evaluator(run: Run, example: Example) -> dict:
    """Score answer relevancy for a production run."""
    import anthropic

    ai_client = anthropic.Anthropic()

    query = run.inputs.get("query", "")
    answer = run.outputs.get("answer", "")

    response = ai_client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=50,
        messages=[{
            "role": "user",
            "content": (
                "Does this answer address the question? "
                "Score 0.0 to 1.0.\n\n"
                f"Question:\n{query}\n\n"
                f"Answer:\n{answer}\n\n"
                "Output ONLY a number between 0.0 and 1.0:"
            ),
        }],
    )

    try:
        score = float(response.content[0].text.strip())
        score = max(0.0, min(1.0, score))
    except (ValueError, IndexError):
        score = 0.0

    return {"key": "relevancy", "score": score}


def run_scheduled_evaluation(
    project_name: str = "rag-production",
    sample_size: int = 50,
    lookback_hours: int = 24,
) -> dict:
    """
    Run scheduled evaluation on recent production traces.

    This function should be called by a cron job or scheduler
    (e.g., daily at 2 AM).
    """
    # Get recent production runs
    start_time = datetime.now() - timedelta(hours=lookback_hours)

    runs = list(client.list_runs(
        project_name=project_name,
        start_time=start_time,
        run_type="chain",
        is_root=True,
        limit=sample_size * 3,  # Over-sample to account for filtering
    ))

    if not runs:
        print("No production runs found in the last 24 hours")
        return {}

    # Sample a subset for evaluation
    import random
    sampled_runs = random.sample(runs, min(sample_size, len(runs)))

    # Create a temporary dataset from sampled runs
    dataset_name = f"prod-eval-{datetime.now().strftime('%Y%m%d')}"

    try:
        dataset = client.create_dataset(dataset_name)
    except Exception:
        # Dataset may already exist
        dataset = client.read_dataset(dataset_name=dataset_name)

    for run in sampled_runs:
        client.create_example(
            inputs=run.inputs,
            outputs=run.outputs,
            dataset_id=dataset.id,
        )

    # Run evaluation
    results = evaluate(
        lambda inputs: inputs,  # Identity -- we score existing outputs
        data=dataset_name,
        evaluators=[faithfulness_evaluator, relevancy_evaluator],
        experiment_prefix="scheduled-eval",
    )

    # Extract aggregate metrics
    metrics = {}
    for evaluator_name in ["faithfulness", "relevancy"]:
        scores = [
            r.evaluation_results.get(evaluator_name, {}).get("score", 0)
            for r in results
            if r.evaluation_results.get(evaluator_name)
        ]
        if scores:
            metrics[evaluator_name] = {
                "mean": sum(scores) / len(scores),
                "min": min(scores),
                "max": max(scores),
                "below_threshold": sum(1 for s in scores if s < 0.70),
                "sample_size": len(scores),
            }

    print(f"Scheduled evaluation complete ({len(sampled_runs)} runs)")
    for name, m in metrics.items():
        print(
            f"  {name}: mean={m['mean']:.3f}, "
            f"min={m['min']:.3f}, "
            f"below 0.70: {m['below_threshold']}/{m['sample_size']}"
        )

    return metrics
```

---

## Approach 2: Langfuse Open-Source Observability

Langfuse is an open-source LLM observability platform that can be self-hosted.

### Instrumenting with Langfuse

```python
"""
Langfuse instrumentation for RAG pipeline monitoring.
"""
import os

from langfuse import Langfuse
from langfuse.decorators import langfuse_context, observe


# Initialize Langfuse client
langfuse = Langfuse(
    public_key=os.environ.get("LANGFUSE_PUBLIC_KEY"),
    secret_key=os.environ.get("LANGFUSE_SECRET_KEY"),
    host=os.environ.get("LANGFUSE_HOST", "https://cloud.langfuse.com"),
)


@observe()
def rag_pipeline(query: str) -> dict:
    """RAG pipeline with Langfuse tracing."""
    # Retrieve
    contexts = retrieve_documents(query)

    # Generate
    answer = generate_answer(query, contexts)

    # Score the trace inline (optional -- can also do it in batch)
    langfuse_context.score_current_trace(
        name="context_count",
        value=len(contexts),
    )

    return {"answer": answer, "contexts": contexts}


@observe()
def retrieve_documents(query: str) -> list[str]:
    """Retrieval with Langfuse tracing."""
    # Your retrieval logic
    return ["Context passage"]


@observe()
def generate_answer(query: str, contexts: list[str]) -> str:
    """Generation with Langfuse tracing."""
    # Your generation logic
    return "Generated answer"
```

### Scheduled Scoring with Langfuse

```python
"""
Langfuse scheduled scoring: batch-score production traces.
"""
import os
from datetime import datetime, timedelta

import anthropic
from langfuse import Langfuse


langfuse = Langfuse()
ai_client = anthropic.Anthropic()


def score_recent_traces(
    hours: int = 24,
    sample_size: int = 50,
    project: str | None = None,
) -> dict:
    """
    Score recent production traces for quality monitoring.

    Runs as a scheduled job (cron) to maintain continuous quality scores.
    """
    # Fetch recent traces
    traces = langfuse.fetch_traces(
        limit=sample_size * 2,
        order_by="timestamp",
        order="DESC",
    )

    if not traces.data:
        print("No traces found")
        return {}

    # Filter to recent traces
    cutoff = datetime.now() - timedelta(hours=hours)
    recent = [
        t for t in traces.data
        if t.timestamp and t.timestamp >= cutoff
    ][:sample_size]

    print(f"Scoring {len(recent)} traces from the last {hours} hours")

    scores = {"faithfulness": [], "relevancy": [], "latency": []}

    for trace in recent:
        # Extract inputs/outputs from the trace
        query = ""
        answer = ""
        contexts = []

        if trace.input and isinstance(trace.input, dict):
            query = trace.input.get("query", "")
        if trace.output and isinstance(trace.output, dict):
            answer = trace.output.get("answer", "")
            contexts = trace.output.get("contexts", [])

        if not query or not answer:
            continue

        # Score faithfulness
        context_text = "\n".join(contexts[:3]) if contexts else ""
        if context_text:
            faith_score = _llm_score(
                f"Is this answer supported by the context?\n\n"
                f"Context:\n{context_text[:1500]}\n\n"
                f"Answer:\n{answer}\n\n"
                f"Score 0.0 to 1.0 (1.0 = fully supported):"
            )
            scores["faithfulness"].append(faith_score)

            # Write score back to Langfuse
            langfuse.score(
                trace_id=trace.id,
                name="faithfulness",
                value=faith_score,
                comment="Automated scoring by monitoring pipeline",
            )

        # Score relevancy
        rel_score = _llm_score(
            f"Does this answer address the question?\n\n"
            f"Question:\n{query}\n\n"
            f"Answer:\n{answer}\n\n"
            f"Score 0.0 to 1.0 (1.0 = perfectly relevant):"
        )
        scores["relevancy"].append(rel_score)

        langfuse.score(
            trace_id=trace.id,
            name="relevancy",
            value=rel_score,
            comment="Automated scoring by monitoring pipeline",
        )

        # Track latency
        if trace.latency:
            scores["latency"].append(trace.latency)

    # Aggregate
    results = {}
    for metric, values in scores.items():
        if values:
            import numpy as np
            results[metric] = {
                "mean": float(np.mean(values)),
                "p50": float(np.median(values)),
                "p95": float(np.percentile(values, 95)),
                "min": float(min(values)),
                "count": len(values),
            }

    langfuse.flush()
    return results


def _llm_score(prompt: str) -> float:
    """Get a numeric score from an LLM."""
    response = ai_client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=10,
        messages=[{"role": "user", "content": prompt}],
    )
    try:
        return max(0.0, min(1.0, float(response.content[0].text.strip())))
    except (ValueError, IndexError):
        return 0.0
```

### Langfuse Dashboard Queries

Langfuse supports custom dashboard panels. Key queries for RAG monitoring:

```python
"""
Langfuse dashboard configuration for RAG monitoring.
"""


def get_dashboard_queries() -> dict:
    """
    SQL-like queries for Langfuse dashboard panels.

    These queries are configured in the Langfuse UI dashboard builder.
    """
    return {
        "faithfulness_trend": {
            "description": "7-day faithfulness score trend",
            "type": "time_series",
            "score_name": "faithfulness",
            "aggregation": "avg",
            "interval": "1d",
            "lookback_days": 30,
        },
        "relevancy_distribution": {
            "description": "Distribution of relevancy scores",
            "type": "histogram",
            "score_name": "relevancy",
            "bins": 10,
            "lookback_days": 7,
        },
        "low_quality_traces": {
            "description": "Traces with faithfulness < 0.5",
            "type": "table",
            "filter": "scores.faithfulness < 0.5",
            "columns": ["timestamp", "input.query", "output.answer", "scores.faithfulness"],
            "limit": 20,
            "order_by": "scores.faithfulness ASC",
        },
        "quality_by_hour": {
            "description": "Average quality by hour of day",
            "type": "bar",
            "score_name": "faithfulness",
            "group_by": "HOUR(timestamp)",
            "aggregation": "avg",
        },
        "latency_p95": {
            "description": "95th percentile latency trend",
            "type": "time_series",
            "metric": "latency",
            "aggregation": "p95",
            "interval": "1h",
            "lookback_days": 7,
        },
    }
```

---

## Approach 3: ARES Classifier Monitoring (Zero API Cost)

For high-volume production systems, using LLM judges for every trace is too expensive. ARES classifiers provide free, fast, deterministic scoring.

### Real-Time Scoring Pipeline

```python
"""
ARES-based production monitoring: zero API cost scoring.
"""
import json
import time
from collections import deque
from datetime import datetime

import numpy as np
import torch
from transformers import AutoModelForSequenceClassification, AutoTokenizer


class ARESProductionMonitor:
    """
    Score production RAG outputs in real time using ARES classifiers.

    No API calls, ~5ms per scoring, deterministic results.
    """

    def __init__(
        self,
        classifiers_dir: str,
        window_size: int = 100,
        alert_threshold: float = 0.70,
        device: str = "cpu",
    ):
        self.window_size = window_size
        self.alert_threshold = alert_threshold
        self.device = device

        # Load classifiers
        self.models = {}
        self.tokenizers = {}
        for dim in ["context_relevance", "answer_faithfulness", "answer_relevance"]:
            model_path = f"{classifiers_dir}/{dim}/best"
            self.models[dim] = (
                AutoModelForSequenceClassification.from_pretrained(model_path)
                .eval()
                .to(device)
            )
            self.tokenizers[dim] = AutoTokenizer.from_pretrained(model_path)

        # Rolling windows for each metric
        self.windows = {
            "context_relevance": deque(maxlen=window_size),
            "answer_faithfulness": deque(maxlen=window_size),
            "answer_relevance": deque(maxlen=window_size),
        }

        # Alert state
        self.alert_active = {dim: False for dim in self.windows}

    def score(
        self,
        query: str,
        context: str,
        answer: str,
    ) -> dict:
        """
        Score a single RAG output. Call this for every production request.

        Returns per-dimension scores (0-1).
        """
        scores = {}

        dimension_inputs = {
            "context_relevance": (query, context),
            "answer_faithfulness": (context, answer),
            "answer_relevance": (query, answer),
        }

        for dim, (text_a, text_b) in dimension_inputs.items():
            enc = self.tokenizers[dim](
                text_a, text_b,
                max_length=512,
                padding=True,
                truncation=True,
                return_tensors="pt",
            ).to(self.device)

            with torch.no_grad():
                logits = self.models[dim](**enc).logits
                prob = torch.softmax(logits, dim=-1)[0, 1].item()

            scores[dim] = prob
            self.windows[dim].append(prob)

        # Check for alerts
        alerts = self._check_alerts()

        return {
            "scores": scores,
            "alerts": alerts,
            "window_means": {
                dim: float(np.mean(list(window))) if window else None
                for dim, window in self.windows.items()
            },
        }

    def _check_alerts(self) -> list[str]:
        """Check if rolling averages have dropped below threshold."""
        alerts = []

        for dim, window in self.windows.items():
            if len(window) < self.window_size // 2:
                continue  # Not enough data yet

            rolling_mean = np.mean(list(window))
            was_active = self.alert_active[dim]

            if rolling_mean < self.alert_threshold:
                if not was_active:
                    self.alert_active[dim] = True
                    alerts.append(
                        f"ALERT: {dim} dropped to {rolling_mean:.3f} "
                        f"(threshold: {self.alert_threshold})"
                    )
            else:
                if was_active:
                    self.alert_active[dim] = False
                    alerts.append(
                        f"RESOLVED: {dim} recovered to {rolling_mean:.3f}"
                    )

        return alerts

    def get_dashboard_data(self) -> dict:
        """Get current monitoring state for dashboard display."""
        data = {}
        for dim, window in self.windows.items():
            values = list(window)
            if values:
                data[dim] = {
                    "current": values[-1],
                    "rolling_mean": float(np.mean(values)),
                    "rolling_std": float(np.std(values)),
                    "min": float(min(values)),
                    "max": float(max(values)),
                    "below_threshold": sum(1 for v in values if v < self.alert_threshold),
                    "window_size": len(values),
                    "alert_active": self.alert_active[dim],
                }
            else:
                data[dim] = {"status": "no_data"}

        return data


# Usage in production
monitor = ARESProductionMonitor(
    classifiers_dir="./ares_models",
    window_size=100,
    alert_threshold=0.70,
)


def handle_rag_request(query: str) -> dict:
    """Production request handler with ARES monitoring."""
    # Run your RAG pipeline
    contexts = retrieve(query)
    answer = generate(query, contexts)

    # Score with ARES (adds ~5ms latency)
    monitoring = monitor.score(
        query=query,
        context=contexts[0] if contexts else "",
        answer=answer,
    )

    # Send alerts if any
    for alert in monitoring["alerts"]:
        send_alert(alert)

    return {"answer": answer, "contexts": contexts}


def retrieve(query):
    return ["placeholder context"]

def generate(query, contexts):
    return "placeholder answer"

def send_alert(message):
    print(f"[ALERT] {message}")
```

### Scheduled Statistical Analysis

```python
"""
Scheduled statistical analysis of ARES monitoring data.
"""
import json
from datetime import datetime, timedelta
from pathlib import Path

import numpy as np
from scipy import stats


class MonitoringAnalyzer:
    """
    Analyze monitoring data for trends and anomalies.

    Run hourly or daily to detect gradual quality drift.
    """

    def __init__(self, metrics_log_dir: str = "./monitoring_logs"):
        self.log_dir = Path(metrics_log_dir)
        self.log_dir.mkdir(parents=True, exist_ok=True)

    def log_batch(
        self,
        metrics: dict[str, list[float]],
        timestamp: str | None = None,
    ) -> None:
        """Log a batch of monitoring metrics."""
        ts = timestamp or datetime.now().isoformat()
        record = {
            "timestamp": ts,
            "metrics": {
                dim: {
                    "mean": float(np.mean(values)),
                    "std": float(np.std(values)),
                    "count": len(values),
                    "p5": float(np.percentile(values, 5)),
                    "p50": float(np.percentile(values, 50)),
                    "p95": float(np.percentile(values, 95)),
                }
                for dim, values in metrics.items()
                if values
            },
        }

        date_str = datetime.now().strftime("%Y-%m-%d")
        log_path = self.log_dir / f"{date_str}.jsonl"
        with open(log_path, "a") as f:
            f.write(json.dumps(record) + "\n")

    def detect_drift(
        self,
        metric: str,
        lookback_days: int = 7,
        reference_days: int = 30,
    ) -> dict:
        """
        Detect quality drift by comparing recent performance
        against a reference period.

        Uses a two-sample t-test to determine if the recent
        period is statistically different from the reference period.
        """
        all_records = self._load_records(reference_days)

        if len(all_records) < 5:
            return {"status": "insufficient_data", "records": len(all_records)}

        # Split into reference and recent
        cutoff_idx = max(1, len(all_records) - lookback_days)
        reference = all_records[:cutoff_idx]
        recent = all_records[cutoff_idx:]

        ref_means = [
            r["metrics"][metric]["mean"]
            for r in reference
            if metric in r.get("metrics", {})
        ]
        recent_means = [
            r["metrics"][metric]["mean"]
            for r in recent
            if metric in r.get("metrics", {})
        ]

        if len(ref_means) < 3 or len(recent_means) < 2:
            return {"status": "insufficient_data"}

        # Two-sample t-test
        t_stat, p_value = stats.ttest_ind(recent_means, ref_means)
        ref_mean = np.mean(ref_means)
        recent_mean = np.mean(recent_means)
        diff = recent_mean - ref_mean

        is_drift = p_value < 0.05 and diff < -0.02  # Significant and meaningful

        return {
            "status": "drift_detected" if is_drift else "stable",
            "reference_mean": float(ref_mean),
            "recent_mean": float(recent_mean),
            "difference": float(diff),
            "p_value": float(p_value),
            "t_statistic": float(t_stat),
            "reference_days": len(ref_means),
            "recent_days": len(recent_means),
        }

    def anomaly_detection(
        self,
        metric: str,
        z_threshold: float = 2.5,
    ) -> list[dict]:
        """
        Detect anomalous data points using z-score.

        Returns list of anomalous records.
        """
        records = self._load_records(30)
        values = [
            r["metrics"][metric]["mean"]
            for r in records
            if metric in r.get("metrics", {})
        ]

        if len(values) < 10:
            return []

        mean = np.mean(values)
        std = np.std(values)
        if std < 0.001:
            return []

        anomalies = []
        for i, (record, value) in enumerate(zip(records, values)):
            z_score = (value - mean) / std
            if abs(z_score) > z_threshold:
                anomalies.append({
                    "timestamp": record["timestamp"],
                    "value": value,
                    "z_score": float(z_score),
                    "expected_range": (
                        float(mean - z_threshold * std),
                        float(mean + z_threshold * std),
                    ),
                })

        return anomalies

    def _load_records(self, days: int) -> list[dict]:
        """Load monitoring records from the last N days."""
        records = []
        for log_file in sorted(self.log_dir.glob("*.jsonl")):
            with open(log_file) as f:
                for line in f:
                    records.append(json.loads(line))
        return records[-days * 24:]  # Rough daily-to-hourly conversion
```

---

## Alerting Patterns

### Multi-Level Alert Configuration

```python
"""
Alert configuration for production RAG monitoring.
"""
import json
import os
from dataclasses import dataclass
from enum import Enum

import requests


class AlertSeverity(Enum):
    INFO = "info"
    WARNING = "warning"
    CRITICAL = "critical"


@dataclass
class AlertRule:
    """A monitoring alert rule."""
    metric: str
    threshold: float
    direction: str  # "below" or "above"
    window_minutes: int
    severity: AlertSeverity
    message_template: str


# Default alert rules for RAG monitoring
DEFAULT_RULES = [
    AlertRule(
        metric="answer_faithfulness",
        threshold=0.60,
        direction="below",
        window_minutes=30,
        severity=AlertSeverity.CRITICAL,
        message_template=(
            "CRITICAL: Answer faithfulness dropped to {value:.3f} "
            "(threshold: {threshold}) over the last {window} minutes. "
            "This indicates the pipeline may be producing hallucinated answers."
        ),
    ),
    AlertRule(
        metric="answer_faithfulness",
        threshold=0.75,
        direction="below",
        window_minutes=60,
        severity=AlertSeverity.WARNING,
        message_template=(
            "WARNING: Answer faithfulness at {value:.3f} "
            "(threshold: {threshold}) over the last {window} minutes."
        ),
    ),
    AlertRule(
        metric="context_relevance",
        threshold=0.50,
        direction="below",
        window_minutes=30,
        severity=AlertSeverity.CRITICAL,
        message_template=(
            "CRITICAL: Context relevance dropped to {value:.3f}. "
            "Retrieval may be broken or index may be corrupted."
        ),
    ),
    AlertRule(
        metric="answer_relevance",
        threshold=0.60,
        direction="below",
        window_minutes=60,
        severity=AlertSeverity.WARNING,
        message_template=(
            "WARNING: Answer relevance at {value:.3f}. "
            "Answers may not be addressing user queries."
        ),
    ),
]


class AlertManager:
    """Send alerts through multiple channels."""

    def __init__(
        self,
        slack_webhook: str | None = None,
        pagerduty_key: str | None = None,
    ):
        self.slack_webhook = slack_webhook or os.environ.get("SLACK_WEBHOOK")
        self.pagerduty_key = pagerduty_key or os.environ.get("PAGERDUTY_KEY")
        self._cooldowns = {}  # Prevent alert storms

    def send_alert(
        self,
        rule: AlertRule,
        current_value: float,
    ) -> None:
        """Send an alert through configured channels."""
        # Cooldown: do not re-alert for the same rule within 15 minutes
        cooldown_key = f"{rule.metric}_{rule.severity.value}"
        import time
        now = time.time()
        if cooldown_key in self._cooldowns:
            if now - self._cooldowns[cooldown_key] < 900:
                return
        self._cooldowns[cooldown_key] = now

        message = rule.message_template.format(
            value=current_value,
            threshold=rule.threshold,
            window=rule.window_minutes,
        )

        if rule.severity == AlertSeverity.CRITICAL:
            self._send_slack(message, color="danger")
            self._send_pagerduty(message, severity="critical")
        elif rule.severity == AlertSeverity.WARNING:
            self._send_slack(message, color="warning")
        else:
            self._send_slack(message, color="good")

    def _send_slack(self, message: str, color: str = "good") -> None:
        if not self.slack_webhook:
            print(f"[Slack] {message}")
            return

        payload = {
            "attachments": [{
                "color": color,
                "text": message,
                "footer": "RAG Monitoring",
            }],
        }

        try:
            requests.post(
                self.slack_webhook, json=payload, timeout=10,
            )
        except requests.RequestException as e:
            print(f"Failed to send Slack alert: {e}")

    def _send_pagerduty(self, message: str, severity: str = "warning") -> None:
        if not self.pagerduty_key:
            return

        payload = {
            "routing_key": self.pagerduty_key,
            "event_action": "trigger",
            "payload": {
                "summary": message[:1024],
                "severity": severity,
                "source": "rag-monitoring",
            },
        }

        try:
            requests.post(
                "https://events.pagerduty.com/v2/enqueue",
                json=payload,
                timeout=10,
            )
        except requests.RequestException as e:
            print(f"Failed to send PagerDuty alert: {e}")
```

---

## Dashboard Patterns

### Key Panels for RAG Monitoring

| Panel | Type | Data Source | Refresh |
|---|---|---|---|
| Faithfulness trend (7d) | Time series line | ARES scores or LLM judge scores | 5 min |
| Relevancy trend (7d) | Time series line | ARES scores or LLM judge scores | 5 min |
| Context relevance (7d) | Time series line | ARES scores | 5 min |
| Quality distribution | Histogram | Latest 1000 scores | 15 min |
| Low-quality traces | Table | Traces with faithfulness < 0.5 | 5 min |
| Latency p50/p95/p99 | Time series (multi-line) | Request latency | 1 min |
| Error rate | Time series | HTTP 5xx or empty responses | 1 min |
| Retrieval empty rate | Gauge | % of requests with 0 contexts | 5 min |
| Score vs latency correlation | Scatter plot | Score and latency per request | 30 min |
| Weekly comparison | Bar chart | Current vs previous week metrics | Daily |

---

## Common Pitfalls

1. **Monitoring adds latency**: ARES classifier scoring adds ~5ms, but LLM-as-judge scoring adds 1-3 seconds. For real-time monitoring, use classifiers synchronously and LLM judges asynchronously or in batch.

2. **Alert fatigue**: too many alerts desensitize the team. Use multi-level severity (info/warning/critical) and cooldown periods to prevent alert storms.

3. **Sampling bias**: if you only score a random sample of production traffic, ensure the sample is representative. Stratify by query type, user segment, or time of day.

4. **No root cause analysis**: an alert says "faithfulness dropped" but not why. Pair monitoring with trace analysis -- when an alert fires, automatically pull the 10 lowest-scoring recent traces for investigation.

5. **Monitoring the wrong thing**: if users care about answer completeness but you only monitor faithfulness, you will miss the issues that matter. Align monitoring metrics with user-facing quality.

6. **No baseline establishment**: run monitoring for 1-2 weeks before setting alert thresholds. Initial thresholds based on guesses will either miss regressions or produce false alarms.

---

## References

- LangSmith: https://docs.smith.langchain.com/evaluation
- Langfuse: https://langfuse.com/docs
- ARES: https://arxiv.org/abs/2311.09476
- Prometheus + Grafana: https://prometheus.io/docs/ (for custom monitoring infrastructure)
- PagerDuty Events API: https://developer.pagerduty.com/api-reference/
