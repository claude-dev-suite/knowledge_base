# Embedding Drift -- Production Alerting Pipeline

## Overview / TL;DR

This guide covers the end-to-end production alerting pipeline for embedding drift: periodic embedding sampling from your query and ingestion streams, metric computation on a schedule, statistical baseline and threshold management, Prometheus metric export, Grafana dashboard configuration, PagerDuty/Slack alert routing, and a root cause diagnosis playbook. The goal is to detect embedding drift within hours of onset and route the alert to the right team with enough context to diagnose the cause.

---

## Architecture

```
Query Stream -----> Embedding Sampler ----+
                                          |
Ingestion Stream -> Embedding Sampler ----+---> Metric Computer ---> Prometheus
                                          |        (hourly)              |
Reference Texts --> Stability Check  -----+                              v
                                                                    Grafana
                                                                        |
                                                                        v
                                                                  AlertManager
                                                                    /      \
                                                                Slack   PagerDuty
```

---

## Step 1: Periodic Embedding Sampling

Integrate samplers into your query and ingestion pipelines to collect embedding samples without storing every vector.

```python
import numpy as np
import time
import json
import threading
from pathlib import Path


class ProductionEmbeddingSampler:
    """Thread-safe embedding sampler for production pipelines.

    Collects a reservoir sample of embeddings and periodically
    flushes them to disk for metric computation.
    """

    def __init__(
        self,
        name: str,  # "query" or "corpus"
        sample_size: int = 2000,
        flush_interval_seconds: int = 3600,  # Flush every hour
        output_dir: str = "/tmp/embedding_samples",
    ):
        self.name = name
        self.sample_size = sample_size
        self.flush_interval = flush_interval_seconds
        self.output_dir = Path(output_dir) / name
        self.output_dir.mkdir(parents=True, exist_ok=True)

        self._samples: list[np.ndarray] = []
        self._count = 0
        self._lock = threading.Lock()
        self._last_flush = time.time()

    def observe(self, embedding: np.ndarray):
        """Add an embedding observation. Thread-safe.

        Call this from your query/ingestion pipeline for every embedding.
        The sampler uses reservoir sampling to maintain a fixed-size sample.
        """
        with self._lock:
            self._count += 1

            if len(self._samples) < self.sample_size:
                self._samples.append(embedding.copy())
            else:
                j = np.random.randint(0, self._count)
                if j < self.sample_size:
                    self._samples[j] = embedding.copy()

            # Auto-flush on interval
            if time.time() - self._last_flush > self.flush_interval:
                self._flush()

    def _flush(self):
        """Write current sample to disk and reset."""
        if not self._samples:
            return

        timestamp = int(time.time())
        sample_array = np.array(self._samples)
        output_path = self.output_dir / f"sample_{timestamp}.npy"
        np.save(output_path, sample_array)

        # Also save metadata
        meta_path = self.output_dir / f"sample_{timestamp}_meta.json"
        with open(meta_path, "w") as f:
            json.dump({
                "timestamp": timestamp,
                "name": self.name,
                "n_samples": len(self._samples),
                "total_observed": self._count,
                "dimensions": sample_array.shape[1],
            }, f)

        self._samples = []
        self._count = 0
        self._last_flush = time.time()

    def get_current_sample(self) -> np.ndarray | None:
        """Get the current sample without flushing."""
        with self._lock:
            if self._samples:
                return np.array(self._samples)
            return None


# Integration into a RAG pipeline
query_sampler = ProductionEmbeddingSampler(name="query", sample_size=2000)
corpus_sampler = ProductionEmbeddingSampler(name="corpus", sample_size=5000)


def embed_query_with_sampling(query: str, embedding_model) -> np.ndarray:
    """Wrap the query embedding call to include drift sampling."""
    embedding = embedding_model.encode([query], normalize_embeddings=True)[0]
    query_sampler.observe(embedding)
    return embedding


def embed_document_with_sampling(document: str, embedding_model) -> np.ndarray:
    """Wrap the document embedding call to include drift sampling."""
    embedding = embedding_model.encode([document], normalize_embeddings=True)[0]
    corpus_sampler.observe(embedding)
    return embedding
```

---

## Step 2: Metric Computation

A scheduled job that loads samples, computes drift metrics, and exports to Prometheus.

```python
import numpy as np
import json
from pathlib import Path
from datetime import datetime


class DriftMetricComputer:
    """Compute drift metrics from sampled embeddings.

    Run as a periodic job (hourly or daily).
    """

    def __init__(
        self,
        baseline_dir: str,
        sample_dir: str,
        reference_embeddings_path: str,
    ):
        self.baseline_dir = Path(baseline_dir)
        self.sample_dir = Path(sample_dir)

        # Load baselines
        self.baseline_query = np.load(self.baseline_dir / "query_baseline.npy")
        self.baseline_corpus = np.load(self.baseline_dir / "corpus_baseline.npy")
        self.reference_baseline = np.load(reference_embeddings_path)

        with open(self.baseline_dir / "thresholds.json") as f:
            self.thresholds = json.load(f)

    def compute_all_metrics(
        self,
        current_query_sample: np.ndarray,
        current_corpus_sample: np.ndarray,
        current_reference_embs: np.ndarray,
    ) -> dict:
        """Compute all drift metrics and determine alert status."""
        metrics = {}
        alerts = []

        # Model stability (reference text comparison)
        ref_sims = []
        for i in range(len(self.reference_baseline)):
            sim = float(np.dot(current_reference_embs[i], self.reference_baseline[i]))
            ref_sims.append(sim)
        metrics["model_stability"] = min(ref_sims)
        if metrics["model_stability"] < 0.999:
            alerts.append("MODEL_CHANGED")

        # Query centroid drift
        q_centroid_base = self.baseline_query.mean(axis=0)
        q_centroid_curr = current_query_sample.mean(axis=0)
        q_centroid_dist = 1.0 - float(np.dot(
            q_centroid_base / np.linalg.norm(q_centroid_base),
            q_centroid_curr / np.linalg.norm(q_centroid_curr),
        ))
        metrics["query_centroid_distance"] = q_centroid_dist
        if q_centroid_dist > self.thresholds.get("centroid_distance", 0.05):
            alerts.append("QUERY_CENTROID_DRIFT")

        # Corpus centroid drift
        c_centroid_base = self.baseline_corpus.mean(axis=0)
        c_centroid_curr = current_corpus_sample.mean(axis=0)
        c_centroid_dist = 1.0 - float(np.dot(
            c_centroid_base / np.linalg.norm(c_centroid_base),
            c_centroid_curr / np.linalg.norm(c_centroid_curr),
        ))
        metrics["corpus_centroid_distance"] = c_centroid_dist
        if c_centroid_dist > self.thresholds.get("centroid_distance", 0.05):
            alerts.append("CORPUS_CENTROID_DRIFT")

        # Similarity distribution shift (query side)
        from scipy import stats
        base_sims = self._sample_similarities(self.baseline_query, 5000)
        curr_sims = self._sample_similarities(current_query_sample, 5000)
        ks_stat, ks_pval = stats.ks_2samp(base_sims, curr_sims)
        metrics["query_ks_statistic"] = float(ks_stat)
        metrics["query_ks_pvalue"] = float(ks_pval)
        if ks_pval < 0.01:
            alerts.append("QUERY_DISTRIBUTION_SHIFT")

        metrics["alerts"] = alerts
        metrics["timestamp"] = datetime.utcnow().isoformat()
        metrics["has_alerts"] = len(alerts) > 0

        return metrics

    def _sample_similarities(self, embeddings: np.ndarray, n: int) -> np.ndarray:
        """Sample pairwise similarities from an embedding set."""
        n_embs = len(embeddings)
        idx_a = np.random.randint(0, n_embs, size=n)
        idx_b = np.random.randint(0, n_embs, size=n)
        mask = idx_a != idx_b
        return np.sum(embeddings[idx_a[mask]] * embeddings[idx_b[mask]], axis=1)
```

---

## Step 3: Prometheus Export

```python
from prometheus_client import Gauge, Counter, start_http_server


# Define Prometheus metrics
CENTROID_DISTANCE_QUERY = Gauge(
    "embedding_drift_centroid_distance_query",
    "Cosine distance between current and baseline query centroids",
)
CENTROID_DISTANCE_CORPUS = Gauge(
    "embedding_drift_centroid_distance_corpus",
    "Cosine distance between current and baseline corpus centroids",
)
MODEL_STABILITY = Gauge(
    "embedding_drift_model_stability",
    "Minimum cosine similarity of reference text embeddings vs baseline",
)
KS_STATISTIC = Gauge(
    "embedding_drift_ks_statistic",
    "Kolmogorov-Smirnov statistic for similarity distribution shift",
)
DRIFT_ALERTS = Counter(
    "embedding_drift_alerts_total",
    "Total number of drift alerts fired",
    ["alert_type"],
)


def export_metrics_to_prometheus(metrics: dict):
    """Push drift metrics to Prometheus gauges."""
    CENTROID_DISTANCE_QUERY.set(metrics.get("query_centroid_distance", 0))
    CENTROID_DISTANCE_CORPUS.set(metrics.get("corpus_centroid_distance", 0))
    MODEL_STABILITY.set(metrics.get("model_stability", 1.0))
    KS_STATISTIC.set(metrics.get("query_ks_statistic", 0))

    for alert in metrics.get("alerts", []):
        DRIFT_ALERTS.labels(alert_type=alert).inc()


# Start Prometheus HTTP server on port 9090
# start_http_server(9090)
```

---

## Step 4: Grafana Dashboard

### Dashboard JSON Configuration

```json
{
  "title": "Embedding Drift Monitoring",
  "panels": [
    {
      "title": "Query Centroid Distance",
      "type": "timeseries",
      "targets": [
        {"expr": "embedding_drift_centroid_distance_query"}
      ],
      "thresholds": [
        {"value": 0.05, "color": "orange"},
        {"value": 0.10, "color": "red"}
      ]
    },
    {
      "title": "Corpus Centroid Distance",
      "type": "timeseries",
      "targets": [
        {"expr": "embedding_drift_centroid_distance_corpus"}
      ],
      "thresholds": [
        {"value": 0.05, "color": "orange"},
        {"value": 0.10, "color": "red"}
      ]
    },
    {
      "title": "Model Stability",
      "type": "gauge",
      "targets": [
        {"expr": "embedding_drift_model_stability"}
      ],
      "thresholds": [
        {"value": 0.999, "color": "green"},
        {"value": 0.99, "color": "orange"},
        {"value": 0, "color": "red"}
      ]
    },
    {
      "title": "Drift Alerts (24h)",
      "type": "stat",
      "targets": [
        {"expr": "increase(embedding_drift_alerts_total[24h])"}
      ]
    }
  ]
}
```

---

## Step 5: Alert Routing

### Slack Alerts

```python
import requests
import json


def send_slack_alert(
    webhook_url: str,
    metrics: dict,
    channel: str = "#ml-monitoring",
):
    """Send a drift alert to Slack."""
    alerts = metrics.get("alerts", [])
    if not alerts:
        return

    severity = "critical" if "MODEL_CHANGED" in alerts else "warning"
    color = "#FF0000" if severity == "critical" else "#FFA500"

    blocks = [
        {
            "type": "header",
            "text": {"type": "plain_text", "text": f"Embedding Drift Alert ({severity.upper()})"},
        },
        {
            "type": "section",
            "fields": [
                {"type": "mrkdwn", "text": f"*Alerts:*\n{chr(10).join(alerts)}"},
                {"type": "mrkdwn", "text": f"*Timestamp:*\n{metrics['timestamp']}"},
            ],
        },
        {
            "type": "section",
            "fields": [
                {"type": "mrkdwn", "text": f"*Query Centroid Distance:*\n{metrics.get('query_centroid_distance', 0):.6f}"},
                {"type": "mrkdwn", "text": f"*Model Stability:*\n{metrics.get('model_stability', 1):.6f}"},
            ],
        },
    ]

    requests.post(webhook_url, json={"blocks": blocks})
```

### PagerDuty Integration

```python
def send_pagerduty_alert(
    routing_key: str,
    metrics: dict,
):
    """Send a critical drift alert to PagerDuty."""
    alerts = metrics.get("alerts", [])
    if "MODEL_CHANGED" not in alerts and "QUALITY_DROP" not in alerts:
        return  # Only page for critical alerts

    event = {
        "routing_key": routing_key,
        "event_action": "trigger",
        "payload": {
            "summary": f"Embedding drift detected: {', '.join(alerts)}",
            "severity": "critical",
            "source": "embedding-drift-monitor",
            "custom_details": {
                "query_centroid_distance": metrics.get("query_centroid_distance"),
                "model_stability": metrics.get("model_stability"),
                "alerts": alerts,
            },
        },
    }

    requests.post(
        "https://events.pagerduty.com/v2/enqueue",
        json=event,
    )
```

---

## Step 6: Root Cause Playbook

### Automated Diagnosis

```python
def diagnose_drift(metrics: dict) -> dict:
    """Automated root cause diagnosis based on drift metrics.

    Returns a diagnosis with probable cause and recommended actions.
    """
    alerts = set(metrics.get("alerts", []))
    diagnosis = {"probable_cause": "unknown", "confidence": "low", "actions": []}

    # Model changed: reference embeddings differ from baseline
    if "MODEL_CHANGED" in alerts:
        diagnosis["probable_cause"] = "EMBEDDING_MODEL_UPDATED"
        diagnosis["confidence"] = "high"
        diagnosis["actions"] = [
            "1. Confirm model version with embedding provider.",
            "2. Re-embed entire corpus with new model.",
            "3. Update reference and baseline embeddings.",
            "4. Re-run quality evaluation.",
            "5. If quality dropped, consider pinning to previous model version.",
        ]
        return diagnosis

    # Query drift without corpus drift: users changed behavior
    if "QUERY_CENTROID_DRIFT" in alerts and "CORPUS_CENTROID_DRIFT" not in alerts:
        diagnosis["probable_cause"] = "USER_BEHAVIOR_CHANGE"
        diagnosis["confidence"] = "medium"
        diagnosis["actions"] = [
            "1. Analyze recent query logs for new topics or patterns.",
            "2. Check if a new product/feature was launched that changed queries.",
            "3. If quality is stable, update the query baseline (normal evolution).",
            "4. If quality dropped, consider expanding corpus to cover new topics.",
        ]
        return diagnosis

    # Corpus drift without query drift: new content added
    if "CORPUS_CENTROID_DRIFT" in alerts and "QUERY_CENTROID_DRIFT" not in alerts:
        diagnosis["probable_cause"] = "CORPUS_EXPANSION"
        diagnosis["confidence"] = "medium"
        diagnosis["actions"] = [
            "1. Check recent document ingestion logs.",
            "2. Verify new documents are being chunked/embedded correctly.",
            "3. Update corpus baseline to include new content.",
            "4. Monitor quality metrics for any degradation.",
        ]
        return diagnosis

    # Both query and corpus drift: possible code change
    if "QUERY_CENTROID_DRIFT" in alerts and "CORPUS_CENTROID_DRIFT" in alerts:
        diagnosis["probable_cause"] = "PREPROCESSING_CHANGE"
        diagnosis["confidence"] = "medium"
        diagnosis["actions"] = [
            "1. Check recent deployments for preprocessing changes.",
            "2. Compare current text cleaning/chunking with previous version.",
            "3. Check for library updates that changed tokenization.",
            "4. If intentional, re-embed corpus and update baselines.",
        ]
        return diagnosis

    # Distribution shift only (no centroid drift): structural change
    if "QUERY_DISTRIBUTION_SHIFT" in alerts:
        diagnosis["probable_cause"] = "QUERY_PATTERN_CHANGE"
        diagnosis["confidence"] = "low"
        diagnosis["actions"] = [
            "1. Analyze query length distribution (shorter/longer queries?).",
            "2. Check for bot traffic or automated queries.",
            "3. If quality is stable, update baselines.",
        ]

    return diagnosis
```

---

## Scheduling

### Cron Configuration

```bash
# Hourly: compute drift metrics from collected samples
0 * * * * python /opt/drift-monitor/compute_metrics.py

# Daily: run full quality evaluation on golden eval set
0 6 * * * python /opt/drift-monitor/quality_evaluation.py

# Weekly: generate drift report and email to team
0 9 * * 1 python /opt/drift-monitor/weekly_report.py
```

### Docker Compose for Monitoring Stack

```yaml
services:
  drift-monitor:
    image: python:3.12-slim
    volumes:
      - ./drift_monitor:/app
      - embedding_samples:/data/samples
    environment:
      - PROMETHEUS_PORT=9090
      - SLACK_WEBHOOK_URL=${SLACK_WEBHOOK_URL}
    command: python /app/monitor_daemon.py

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    volumes:
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

volumes:
  embedding_samples:
```

---

## Common Pitfalls

1. **Alerting on every metric independently.** If centroid distance, PSI, and KL divergence all fire simultaneously, you get 3 alerts for 1 event. Correlate alerts before notifying.
2. **Not having a quiet period after intentional changes.** After deploying a new model or expanding the corpus, suppress alerts for 24-48 hours while baselines are updated.
3. **Monitoring only queries, not the corpus.** Corpus drift from new document ingestion is a common cause of quality degradation that query-side monitoring misses.
4. **Alert fatigue from tight thresholds.** If the team ignores drift alerts because they fire too often, real drift will be missed. Calibrate thresholds from your baseline period.
5. **Not testing the alerting pipeline.** Periodically inject synthetic drift (embed a random corpus) to verify the full pipeline works end-to-end.

---

## References

- Prometheus Client Python -- https://github.com/prometheus/client_python
- Grafana Dashboards -- https://grafana.com/docs/grafana/latest/dashboards/
- PagerDuty Events API v2 -- https://developer.pagerduty.com/docs/events-api-v2/trigger-events/
- Arize Embedding Monitoring -- https://docs.arize.com/arize/machine-learning/embeddings
- Evidently AI Drift Detection -- https://docs.evidentlyai.com/
