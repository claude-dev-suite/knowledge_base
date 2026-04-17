# Ingestion Orchestration -- Airflow, Prefect, and Dagster for RAG Pipelines

## TL;DR

RAG ingestion is not a single script -- it is a multi-stage pipeline (extract, parse, chunk, embed, index) that must handle failures gracefully, process incrementally, track data lineage, and scale to millions of documents. Workflow orchestration frameworks (Airflow, Prefect, Dagster) provide the infrastructure to build reliable ingestion pipelines with retry logic, dependency management, scheduling, and observability. This overview compares the three frameworks for RAG workloads, covers pipeline design patterns, and provides decision criteria for choosing between them.

---

## Why RAG Ingestion Needs Orchestration

### The Naive Approach Breaks

A simple Python script that processes documents end-to-end fails in production because:

1. **Partial failures**: If embedding fails on document 500 of 1000, you lose progress on the first 499
2. **No retry logic**: Transient API errors (rate limits, timeouts) cause full pipeline failure
3. **No backfill**: When you change your chunking strategy, you need to reprocess all documents
4. **No observability**: You cannot tell which documents succeeded, which failed, or why
5. **No scheduling**: Manual runs do not scale; you need periodic ingestion of new content
6. **No dependency management**: Embedding depends on chunking, which depends on parsing -- sequencing matters

### What Orchestration Adds

| Capability | Script | Orchestrator |
|-----------|--------|-------------|
| Retry on failure | Manual | Automatic (configurable) |
| Resume from checkpoint | No | Yes (task-level) |
| Scheduling | cron (basic) | Built-in, UI-managed |
| Dependency graph | Implicit | Explicit DAG |
| Observability | print() | Dashboard, logs, metrics |
| Backfill | Rerun everything | Selective reprocessing |
| Parallelism | Manual threading | Built-in (workers, pools) |
| Data lineage | None | Tracked per task |

---

## Framework Comparison

```python
def compare_orchestration_frameworks() -> dict:
    """Compare Airflow, Prefect, and Dagster for RAG ingestion."""
    return {
        "airflow": {
            "name": "Apache Airflow",
            "architecture": "Scheduler + Workers + Metadata DB + Web UI",
            "dag_definition": "Python decorators or operators",
            "strengths": [
                "Mature ecosystem (10+ years)",
                "Largest community and operator library",
                "Proven at massive scale (Airbnb, Lyft, Spotify)",
                "Rich integrations (AWS, GCP, Azure, databases)",
            ],
            "weaknesses": [
                "Complex setup (scheduler, workers, DB, Redis/RabbitMQ)",
                "DAGs cannot be parameterized dynamically (pre-2.0)",
                "Testing DAGs locally is difficult",
                "UI is functional but dated",
            ],
            "best_for": "Large teams with existing Airflow infrastructure",
            "rag_fit": "Good for batch ingestion, weak for event-driven",
            "deployment": "Self-hosted, MWAA (AWS), Cloud Composer (GCP)",
        },
        "prefect": {
            "name": "Prefect",
            "architecture": "Flow functions + Prefect Server/Cloud + Workers",
            "dag_definition": "Python functions with decorators (@flow, @task)",
            "strengths": [
                "Pythonic API (feels like writing normal Python)",
                "Easy local development and testing",
                "Dynamic task generation at runtime",
                "Built-in caching and result persistence",
                "Event-driven triggers + scheduling",
            ],
            "weaknesses": [
                "Smaller community than Airflow",
                "Fewer pre-built integrations",
                "Prefect Cloud pricing can be expensive at scale",
            ],
            "best_for": "Python-first teams, RAG pipelines with dynamic logic",
            "rag_fit": "Excellent (Pythonic, dynamic, easy to test)",
            "deployment": "Prefect Cloud (managed) or Prefect Server (self-hosted)",
        },
        "dagster": {
            "name": "Dagster",
            "architecture": "Assets + Ops + Jobs + Dagit UI",
            "dag_definition": "Software-defined assets (data-centric)",
            "strengths": [
                "Asset-centric (not task-centric): models data, not just tasks",
                "Built-in data lineage and partitioning",
                "Excellent testing framework",
                "Type system for IO (enforces data contracts)",
                "Dagster+ (cloud) includes asset catalog",
            ],
            "weaknesses": [
                "Steeper learning curve (asset concepts)",
                "Smaller community than Airflow",
                "Newer, less battle-tested at extreme scale",
            ],
            "best_for": "Data engineering teams, complex multi-source ingestion",
            "rag_fit": "Excellent (asset partitions map to document batches)",
            "deployment": "Dagster+ (cloud) or dagster-webserver (self-hosted)",
        },
    }
```

### Decision Matrix

| Criterion | Airflow | Prefect | Dagster |
|-----------|---------|---------|---------|
| Setup complexity | High | Low | Medium |
| Local development | Hard | Easy | Easy |
| Dynamic DAGs | Limited | Native | Via partitions |
| Testing support | Weak | Good | Excellent |
| Event-driven | Weak | Strong | Good |
| Data lineage | Weak | Moderate | Excellent |
| Scaling to 100K+ docs | Proven | Proven | Proven |
| Learning curve | Medium | Low | Medium-High |
| Cost (self-hosted) | Free | Free | Free |
| Cost (managed) | $200-2000/mo | $0-500/mo | $0-800/mo |

### Recommendation for RAG

```python
def recommend_orchestrator(
    team_size: int,
    existing_infrastructure: str,
    pipeline_complexity: str,
    event_driven_needed: bool,
) -> str:
    """Recommend an orchestrator for RAG ingestion."""
    if existing_infrastructure == "airflow":
        return "Airflow (leverage existing infrastructure)"

    if team_size <= 3 and pipeline_complexity in ("simple", "medium"):
        return "Prefect (easiest to start, Pythonic API)"

    if pipeline_complexity == "complex" or event_driven_needed:
        if team_size >= 5:
            return "Dagster (asset model fits complex multi-source ingestion)"
        return "Prefect (dynamic flows, event triggers)"

    return "Prefect (best balance of simplicity and power for RAG)"
```

---

## RAG Ingestion Pipeline Design

### Pipeline Stages

Every RAG ingestion pipeline has the same core stages:

```
[Source Detection]
    |
    v
[Document Extraction]  --> raw text, metadata
    |
    v
[Parsing & Cleaning]   --> clean text, tables, images
    |
    v
[Chunking]             --> text chunks with metadata
    |
    v
[Embedding]            --> vectors
    |
    v
[Indexing]             --> vector DB upsert
    |
    v
[Validation]           --> quality checks
```

### Stage Responsibilities

```python
from dataclasses import dataclass, field
from enum import Enum


class PipelineStage(Enum):
    DETECT = "detect"
    EXTRACT = "extract"
    PARSE = "parse"
    CHUNK = "chunk"
    EMBED = "embed"
    INDEX = "index"
    VALIDATE = "validate"


@dataclass
class StageConfig:
    stage: PipelineStage
    retry_count: int
    retry_delay_seconds: float
    timeout_seconds: float
    parallelism: int
    description: str


STAGE_CONFIGS = [
    StageConfig(
        stage=PipelineStage.DETECT,
        retry_count=2,
        retry_delay_seconds=5,
        timeout_seconds=60,
        parallelism=1,
        description="Scan sources for new/modified documents",
    ),
    StageConfig(
        stage=PipelineStage.EXTRACT,
        retry_count=3,
        retry_delay_seconds=10,
        timeout_seconds=300,
        parallelism=4,
        description="Download and extract raw content from sources",
    ),
    StageConfig(
        stage=PipelineStage.PARSE,
        retry_count=2,
        retry_delay_seconds=5,
        timeout_seconds=120,
        parallelism=8,
        description="Parse PDFs, HTML, DOCX into clean text + metadata",
    ),
    StageConfig(
        stage=PipelineStage.CHUNK,
        retry_count=1,
        retry_delay_seconds=0,
        timeout_seconds=60,
        parallelism=8,
        description="Split text into chunks with overlap",
    ),
    StageConfig(
        stage=PipelineStage.EMBED,
        retry_count=5,
        retry_delay_seconds=30,
        timeout_seconds=600,
        parallelism=2,
        description="Generate embeddings (API rate limited)",
    ),
    StageConfig(
        stage=PipelineStage.INDEX,
        retry_count=3,
        retry_delay_seconds=10,
        timeout_seconds=300,
        parallelism=2,
        description="Upsert vectors and metadata into vector DB",
    ),
    StageConfig(
        stage=PipelineStage.VALIDATE,
        retry_count=1,
        retry_delay_seconds=0,
        timeout_seconds=120,
        parallelism=1,
        description="Run quality checks on indexed content",
    ),
]
```

---

## Error Handling Patterns

```python
from typing import Callable, TypeVar
import time
import logging

T = TypeVar("T")
logger = logging.getLogger(__name__)


def retry_with_backoff(
    func: Callable[..., T],
    max_retries: int = 3,
    initial_delay: float = 1.0,
    backoff_factor: float = 2.0,
    retryable_exceptions: tuple = (Exception,),
) -> T:
    """Retry a function with exponential backoff.

    Common retryable exceptions in RAG pipelines:
    - HTTP 429 (rate limit) from embedding APIs
    - HTTP 503 (service unavailable) from vector DBs
    - TimeoutError from slow document parsing
    - ConnectionError from network issues
    """
    delay = initial_delay
    last_exception = None

    for attempt in range(max_retries + 1):
        try:
            return func()
        except retryable_exceptions as e:
            last_exception = e
            if attempt < max_retries:
                logger.warning(
                    f"Attempt {attempt + 1}/{max_retries + 1} failed: {e}. "
                    f"Retrying in {delay:.1f}s..."
                )
                time.sleep(delay)
                delay *= backoff_factor
            else:
                logger.error(
                    f"All {max_retries + 1} attempts failed for {func.__name__}"
                )

    raise last_exception


class DeadLetterQueue:
    """Store failed documents for later investigation and reprocessing.

    Instead of blocking the pipeline on a single failure, move the
    failed document to a dead letter queue and continue processing.
    """

    def __init__(self, store_path: str = "./dead_letter_queue"):
        from pathlib import Path
        import json
        self.path = Path(store_path)
        self.path.mkdir(parents=True, exist_ok=True)

    def add(
        self,
        document_id: str,
        stage: str,
        error: str,
        metadata: dict | None = None,
    ) -> None:
        """Add a failed document to the DLQ."""
        import json
        entry = {
            "document_id": document_id,
            "stage": stage,
            "error": str(error),
            "timestamp": time.time(),
            "metadata": metadata or {},
        }
        file_path = self.path / f"{document_id}.json"
        file_path.write_text(json.dumps(entry, indent=2))

    def list_failures(self) -> list[dict]:
        """List all documents in the DLQ."""
        import json
        failures = []
        for file in self.path.glob("*.json"):
            failures.append(json.loads(file.read_text()))
        return sorted(failures, key=lambda x: x["timestamp"], reverse=True)

    def requeue(self, document_id: str) -> bool:
        """Remove a document from the DLQ for reprocessing."""
        file_path = self.path / f"{document_id}.json"
        if file_path.exists():
            file_path.unlink()
            return True
        return False
```

---

## Common Pitfalls

1. **Running the embedding stage with full parallelism.** Embedding APIs have rate limits (typically 1000-3000 RPM). Set parallelism to 2-3 for the embedding stage, not 8+.

2. **Not implementing dead letter queues.** One corrupted PDF should not block 999 other documents. Move failures to a DLQ and continue.

3. **Scheduling too frequently.** If ingestion takes 2 hours and you schedule every 1 hour, runs will overlap and cause conflicts. Set schedule intervals longer than the expected pipeline duration.

4. **Not tracking which documents are already indexed.** Without change detection, every run reprocesses all documents. Use content hashes to skip unchanged documents.

5. **Ignoring the validation stage.** After indexing, verify that chunks are searchable, embeddings have correct dimensions, and metadata is intact. Catch issues before users encounter them.

6. **Over-engineering for small corpora.** If you have 100 documents that change monthly, a simple Python script with error handling is fine. Orchestration adds value at 1000+ documents with regular updates.

---

## References

- Apache Airflow: https://airflow.apache.org/
- Prefect: https://docs.prefect.io/
- Dagster: https://docs.dagster.io/
- Comparing workflow orchestrators: https://docs.dagster.io/about/other-tools
