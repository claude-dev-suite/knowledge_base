# Ingestion Orchestration -- Monitoring: Failure Handling, Backfill, and Data Lineage

## TL;DR

Production RAG ingestion pipelines require comprehensive monitoring beyond basic logging. This article covers three critical monitoring dimensions: (1) failure handling with dead letter queues, alerting, and automatic remediation, (2) backfill strategies for reprocessing historical documents when pipeline logic changes, and (3) data lineage tracking to trace every vector in the index back to its source document, chunk strategy, and embedding model version.

---

## Failure Handling

### Failure Categories in RAG Ingestion

```python
from enum import Enum
from dataclasses import dataclass, field
import time


class FailureCategory(Enum):
    TRANSIENT = "transient"           # Network timeout, rate limit -> auto-retry
    DOCUMENT_ERROR = "document_error" # Corrupt PDF, unsupported format -> DLQ
    PIPELINE_BUG = "pipeline_bug"     # Code error -> alert + investigate
    RESOURCE_LIMIT = "resource_limit" # OOM, disk full -> alert + scale
    API_CHANGE = "api_change"         # Embedding API response changed -> alert


@dataclass
class FailureRecord:
    document_id: str
    stage: str
    category: FailureCategory
    error_message: str
    timestamp: float = field(default_factory=time.time)
    retry_count: int = 0
    resolved: bool = False
    resolution: str = ""


class FailureTracker:
    """Track and categorize pipeline failures for monitoring."""

    def __init__(self):
        self.failures: list[FailureRecord] = []
        self._category_patterns = {
            FailureCategory.TRANSIENT: [
                "rate limit", "429", "timeout", "connection reset",
                "503", "temporary", "retry",
            ],
            FailureCategory.DOCUMENT_ERROR: [
                "corrupt", "invalid pdf", "unsupported format",
                "encoding error", "empty document",
            ],
            FailureCategory.RESOURCE_LIMIT: [
                "out of memory", "oom", "disk full", "no space",
                "memory error",
            ],
            FailureCategory.API_CHANGE: [
                "unexpected response", "schema changed",
                "deprecated", "invalid api key",
            ],
        }

    def record_failure(
        self, document_id: str, stage: str, error: Exception, retry_count: int = 0
    ) -> FailureRecord:
        """Record a failure and auto-categorize it."""
        error_msg = str(error).lower()
        category = self._categorize(error_msg)

        record = FailureRecord(
            document_id=document_id,
            stage=stage,
            category=category,
            error_message=str(error),
            retry_count=retry_count,
        )
        self.failures.append(record)
        return record

    def _categorize(self, error_msg: str) -> FailureCategory:
        """Auto-categorize a failure based on error message patterns."""
        for category, patterns in self._category_patterns.items():
            for pattern in patterns:
                if pattern in error_msg:
                    return category
        return FailureCategory.PIPELINE_BUG

    def get_summary(self) -> dict:
        """Get failure summary for monitoring dashboards."""
        by_category = {}
        by_stage = {}

        for f in self.failures:
            cat = f.category.value
            by_category[cat] = by_category.get(cat, 0) + 1
            by_stage[f.stage] = by_stage.get(f.stage, 0) + 1

        return {
            "total_failures": len(self.failures),
            "unresolved": sum(1 for f in self.failures if not f.resolved),
            "by_category": by_category,
            "by_stage": by_stage,
            "failure_rate": len(self.failures),  # divide by total docs externally
        }

    def should_alert(self, threshold: int = 10) -> bool:
        """Determine if failures warrant an alert."""
        recent = [
            f for f in self.failures
            if time.time() - f.timestamp < 3600  # last hour
            and not f.resolved
        ]
        return len(recent) >= threshold
```

### Dead Letter Queue with Retry Scheduling

```python
import json
from pathlib import Path


class PersistentDLQ:
    """Persistent dead letter queue with scheduled retry.

    Documents in the DLQ can be:
    1. Auto-retried after a delay (transient failures)
    2. Manually reviewed and requeued
    3. Permanently skipped with a reason
    """

    def __init__(self, dlq_dir: str = "./dlq"):
        self.dir = Path(dlq_dir)
        self.dir.mkdir(parents=True, exist_ok=True)

    def enqueue(self, failure: FailureRecord) -> None:
        """Add a failed document to the DLQ."""
        entry = {
            "document_id": failure.document_id,
            "stage": failure.stage,
            "category": failure.category.value,
            "error": failure.error_message,
            "timestamp": failure.timestamp,
            "retry_count": failure.retry_count,
            "next_retry": self._compute_next_retry(failure),
        }
        path = self.dir / f"{failure.document_id.replace('/', '_')}.json"
        path.write_text(json.dumps(entry, indent=2))

    def get_ready_for_retry(self) -> list[dict]:
        """Get documents ready for automatic retry."""
        ready = []
        now = time.time()
        for path in self.dir.glob("*.json"):
            entry = json.loads(path.read_text())
            if entry.get("next_retry") and entry["next_retry"] <= now:
                ready.append(entry)
        return ready

    def remove(self, document_id: str) -> None:
        """Remove a document from the DLQ after successful processing."""
        path = self.dir / f"{document_id.replace('/', '_')}.json"
        if path.exists():
            path.unlink()

    def _compute_next_retry(self, failure: FailureRecord) -> float | None:
        """Compute next retry time with exponential backoff."""
        if failure.category == FailureCategory.TRANSIENT:
            delay = min(300 * (2 ** failure.retry_count), 86400)  # max 24h
            return time.time() + delay
        if failure.category == FailureCategory.DOCUMENT_ERROR:
            return None  # manual review required
        return None
```

---

## Backfill Strategies

```python
from dataclasses import dataclass
from enum import Enum


class BackfillReason(Enum):
    CHUNKING_CHANGE = "chunking_strategy_changed"
    EMBEDDING_MODEL_CHANGE = "embedding_model_changed"
    PARSING_IMPROVEMENT = "parsing_logic_improved"
    BUG_FIX = "bug_fix_reprocessing"
    SCHEMA_MIGRATION = "metadata_schema_changed"


@dataclass
class BackfillConfig:
    reason: BackfillReason
    affected_documents: list[str] | None  # None = all documents
    batch_size: int = 100
    priority: str = "low"  # low, medium, high
    delete_before_reindex: bool = True


class BackfillManager:
    """Manage backfill operations for RAG ingestion pipelines.

    Backfill is needed when:
    1. Chunking strategy changes (e.g., 512 -> 1024 tokens)
    2. Embedding model upgrades (e.g., ada-002 -> text-embedding-3-small)
    3. Parsing bugs are fixed (documents need reprocessing)
    4. Metadata schema changes (new fields needed)
    """

    def __init__(self, state_file: str, vector_store_client):
        self.state_file = state_file
        self.vector_store = vector_store_client

    def plan_backfill(self, config: BackfillConfig) -> dict:
        """Plan a backfill operation without executing it."""
        # Determine affected documents
        if config.affected_documents:
            doc_ids = config.affected_documents
        else:
            state = json.loads(Path(self.state_file).read_text())
            doc_ids = list(state.keys())

        total_batches = (len(doc_ids) + config.batch_size - 1) // config.batch_size

        return {
            "reason": config.reason.value,
            "total_documents": len(doc_ids),
            "total_batches": total_batches,
            "estimated_duration_minutes": total_batches * 2,
            "delete_before_reindex": config.delete_before_reindex,
            "document_ids": doc_ids,
        }

    def execute_backfill(
        self,
        config: BackfillConfig,
        pipeline_fn,  # The ingestion pipeline function
    ) -> dict:
        """Execute a backfill operation in batches.

        Runs the pipeline on affected documents in batches,
        with optional deletion of old vectors before reindexing.
        """
        plan = self.plan_backfill(config)
        doc_ids = plan["document_ids"]
        stats = {"processed": 0, "failed": 0, "batches": 0}

        for i in range(0, len(doc_ids), config.batch_size):
            batch = doc_ids[i : i + config.batch_size]

            if config.delete_before_reindex:
                for doc_id in batch:
                    self._delete_document_vectors(doc_id)

            # Reset state for these documents so the pipeline reprocesses them
            self._reset_document_state(batch)

            # Run pipeline
            try:
                result = pipeline_fn()
                stats["processed"] += result.documents_processed
                stats["failed"] += result.documents_failed
            except Exception as e:
                stats["failed"] += len(batch)

            stats["batches"] += 1

        return stats

    def _delete_document_vectors(self, document_id: str) -> None:
        """Delete all vectors for a document from the vector store."""
        self.vector_store.delete(
            filter={"document_id": document_id}
        )

    def _reset_document_state(self, document_ids: list[str]) -> None:
        """Remove documents from the state file so they are reprocessed."""
        state_path = Path(self.state_file)
        if state_path.exists():
            state = json.loads(state_path.read_text())
            for doc_id in document_ids:
                state.pop(doc_id, None)
            state_path.write_text(json.dumps(state, indent=2))
```

---

## Data Lineage Tracking

```python
from dataclasses import dataclass
from datetime import datetime


@dataclass
class LineageRecord:
    """Track the full lineage of a vector from source to index."""
    vector_id: str
    chunk_id: str
    document_id: str
    source_path: str
    # Pipeline metadata
    pipeline_run_id: str
    pipeline_version: str
    # Processing details
    parser_version: str
    chunking_strategy: str
    chunk_size: int
    chunk_overlap: int
    embedding_model: str
    embedding_dimension: int
    # Timestamps
    document_modified_at: str
    ingested_at: str
    # Quality
    chunk_token_count: int
    document_content_hash: str


class LineageStore:
    """Store and query data lineage for RAG vectors.

    Enables:
    - "Which documents contributed to this answer?"
    - "Which vectors need updating after a model change?"
    - "When was this document last reprocessed?"
    - "What pipeline version produced this vector?"
    """

    def __init__(self, store_path: str = "./lineage"):
        self.path = Path(store_path)
        self.path.mkdir(parents=True, exist_ok=True)
        self._index: dict[str, LineageRecord] = {}

    def record(self, lineage: LineageRecord) -> None:
        """Record lineage for a vector."""
        self._index[lineage.vector_id] = lineage

    def get_by_vector(self, vector_id: str) -> LineageRecord | None:
        """Get lineage for a specific vector."""
        return self._index.get(vector_id)

    def get_by_document(self, document_id: str) -> list[LineageRecord]:
        """Get all vectors derived from a document."""
        return [
            r for r in self._index.values()
            if r.document_id == document_id
        ]

    def get_by_model(self, embedding_model: str) -> list[LineageRecord]:
        """Get all vectors created by a specific embedding model."""
        return [
            r for r in self._index.values()
            if r.embedding_model == embedding_model
        ]

    def get_by_pipeline_version(self, version: str) -> list[LineageRecord]:
        """Get all vectors created by a specific pipeline version."""
        return [
            r for r in self._index.values()
            if r.pipeline_version == version
        ]

    def find_stale_vectors(
        self,
        current_model: str,
        current_pipeline_version: str,
    ) -> list[LineageRecord]:
        """Find vectors that need reprocessing.

        A vector is stale if:
        - It was created with a different embedding model
        - It was created with an older pipeline version
        """
        stale = []
        for record in self._index.values():
            if (
                record.embedding_model != current_model
                or record.pipeline_version != current_pipeline_version
            ):
                stale.append(record)
        return stale

    def generate_report(self) -> dict:
        """Generate a lineage report for monitoring."""
        records = list(self._index.values())
        if not records:
            return {"total_vectors": 0}

        models = {}
        pipelines = {}
        documents = set()

        for r in records:
            models[r.embedding_model] = models.get(r.embedding_model, 0) + 1
            pipelines[r.pipeline_version] = pipelines.get(r.pipeline_version, 0) + 1
            documents.add(r.document_id)

        return {
            "total_vectors": len(records),
            "total_documents": len(documents),
            "embedding_models": models,
            "pipeline_versions": pipelines,
            "oldest_ingestion": min(r.ingested_at for r in records),
            "newest_ingestion": max(r.ingested_at for r in records),
        }
```

---

## Metrics and Alerting

```python
class IngestionMetrics:
    """Collect and expose metrics for RAG ingestion monitoring.

    Key metrics:
    - documents_processed_total: counter
    - documents_failed_total: counter
    - chunks_created_total: counter
    - vectors_indexed_total: counter
    - pipeline_duration_seconds: histogram
    - embedding_api_latency_seconds: histogram
    - dlq_size: gauge
    """

    def __init__(self):
        self.counters: dict[str, int] = {
            "documents_processed": 0,
            "documents_failed": 0,
            "chunks_created": 0,
            "vectors_indexed": 0,
        }
        self.histograms: dict[str, list[float]] = {
            "pipeline_duration_seconds": [],
            "embedding_latency_seconds": [],
        }
        self.gauges: dict[str, float] = {
            "dlq_size": 0,
            "stale_vectors": 0,
        }

    def increment(self, metric: str, value: int = 1) -> None:
        self.counters[metric] = self.counters.get(metric, 0) + value

    def observe(self, metric: str, value: float) -> None:
        self.histograms.setdefault(metric, []).append(value)

    def set_gauge(self, metric: str, value: float) -> None:
        self.gauges[metric] = value

    def get_dashboard_data(self) -> dict:
        """Get all metrics for a monitoring dashboard."""
        dashboard = {
            "counters": self.counters,
            "gauges": self.gauges,
        }

        for name, values in self.histograms.items():
            if values:
                dashboard[f"{name}_avg"] = sum(values) / len(values)
                dashboard[f"{name}_p95"] = sorted(values)[int(len(values) * 0.95)]
                dashboard[f"{name}_max"] = max(values)

        return dashboard

    def check_alerts(self) -> list[dict]:
        """Check for alert conditions."""
        alerts = []

        # High failure rate
        total = self.counters.get("documents_processed", 0) + self.counters.get("documents_failed", 0)
        if total > 0:
            failure_rate = self.counters.get("documents_failed", 0) / total
            if failure_rate > 0.1:
                alerts.append({
                    "severity": "critical",
                    "metric": "failure_rate",
                    "value": f"{failure_rate:.1%}",
                    "threshold": "10%",
                    "message": f"Document failure rate is {failure_rate:.1%}",
                })

        # DLQ growing
        if self.gauges.get("dlq_size", 0) > 100:
            alerts.append({
                "severity": "warning",
                "metric": "dlq_size",
                "value": self.gauges["dlq_size"],
                "message": "Dead letter queue has over 100 documents",
            })

        # Stale vectors
        if self.gauges.get("stale_vectors", 0) > 1000:
            alerts.append({
                "severity": "info",
                "metric": "stale_vectors",
                "value": self.gauges["stale_vectors"],
                "message": "Over 1000 vectors need reprocessing",
            })

        return alerts
```

---

## Common Pitfalls

1. **Not tracking pipeline version in vector metadata.** When you change the chunking strategy, you need to know which vectors were created with the old strategy. Tag every vector with the pipeline version.

2. **Backfilling without deleting old vectors.** If you change chunk size from 512 to 1024, the old 512-token vectors still exist. Delete before reindexing to avoid duplicate/conflicting results.

3. **Alerting on every failure.** Transient failures (rate limits) are expected and auto-retry. Only alert on persistent failures, high failure rates, or DLQ growth.

4. **Not monitoring embedding API latency.** A sudden increase in embedding latency indicates an API issue or throttling. Track P95 latency and alert on spikes.

5. **Ignoring data lineage for compliance.** In regulated domains (healthcare, finance), you must trace every answer back to its source document. Lineage tracking is not optional.

6. **Running backfills during peak hours.** Backfills are expensive (many API calls). Schedule them during off-peak hours to avoid impacting user-facing retrieval latency.

---

## References

- Prefect monitoring: https://docs.prefect.io/concepts/automations/
- Data lineage patterns: https://docs.dagster.io/concepts/assets/asset-observations
- OpenTelemetry for pipeline monitoring: https://opentelemetry.io/
