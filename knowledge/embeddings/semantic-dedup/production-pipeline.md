# Semantic Dedup -- Production Pipeline

## Overview / TL;DR

This guide covers the full production deduplication pipeline: ingest documents, embed them, cluster near-duplicates, select representatives, and index only unique content. It includes handling for incremental updates (is a new document a near-duplicate of an existing one?), dedup at chunk level vs document level, integration with ingestion pipelines using pre-index filtering, monitoring dedup rates, and orchestration code with Dagster and Prefect. A well-implemented dedup pipeline typically removes 10-25% of a corpus, saving proportional storage costs and improving retrieval quality by eliminating redundant results.

---

## Pipeline Architecture

```
New Documents
      |
      v
[Preprocessing]  -- text cleaning, normalization
      |
      v
[Document-Level Dedup]  -- hash-based (fast, exact/near-exact)
      |
      v
[Chunking]  -- split into retrieval-sized chunks
      |
      v
[Embedding]  -- encode chunks with embedding model
      |
      v
[Chunk-Level Dedup]  -- embedding similarity (catches paraphrases)
      |
      v
[Index Unique Chunks]  -- vector database
      |
      v
[Monitor Dedup Rate]  -- track % removed over time
```

---

## Full Pipeline Implementation

```python
import hashlib
import json
import time
import numpy as np
from dataclasses import dataclass, field, asdict
from pathlib import Path
from sentence_transformers import SentenceTransformer
import faiss


@dataclass
class DedupConfig:
    """Configuration for the dedup pipeline."""
    embedding_model: str = "BAAI/bge-base-en-v1.5"
    doc_hash_threshold: float = 0.95  # For normalized hash comparison
    chunk_similarity_threshold: float = 0.95  # For embedding comparison
    chunk_size: int = 500  # Characters per chunk
    chunk_overlap: int = 100
    n_neighbors: int = 20  # FAISS search neighbors
    batch_size: int = 256


@dataclass
class DedupStats:
    """Statistics from a dedup run."""
    total_documents: int = 0
    doc_duplicates_removed: int = 0
    total_chunks: int = 0
    chunk_duplicates_removed: int = 0
    unique_chunks_indexed: int = 0
    dedup_rate: float = 0.0
    processing_time_seconds: float = 0.0


class ProductionDedupPipeline:
    """Production deduplication pipeline with incremental support."""

    def __init__(self, config: DedupConfig, index_dir: str = "./dedup_index"):
        self.config = config
        self.index_dir = Path(index_dir)
        self.index_dir.mkdir(parents=True, exist_ok=True)

        self.model = SentenceTransformer(config.embedding_model)
        self.dim = self.model.get_sentence_embedding_dimension()

        # Persistent state
        self.doc_hashes: set = set()
        self.chunk_index: faiss.IndexFlatIP | None = None
        self.chunk_texts: list[str] = []
        self.chunk_metadata: list[dict] = []

        self._load_state()

    def ingest_documents(
        self,
        documents: list[dict],  # [{"id": "...", "text": "...", "metadata": {...}}]
    ) -> DedupStats:
        """Ingest new documents through the dedup pipeline.

        Returns statistics about how many duplicates were removed.
        """
        start_time = time.time()
        stats = DedupStats(total_documents=len(documents))

        # Stage 1: Document-level dedup (hash-based)
        unique_docs = self._doc_level_dedup(documents)
        stats.doc_duplicates_removed = len(documents) - len(unique_docs)

        # Stage 2: Chunk documents
        all_chunks = []
        for doc in unique_docs:
            chunks = self._chunk_document(doc)
            all_chunks.extend(chunks)
        stats.total_chunks = len(all_chunks)

        # Stage 3: Embed chunks
        chunk_texts = [c["text"] for c in all_chunks]
        embeddings = self.model.encode(
            chunk_texts,
            normalize_embeddings=True,
            batch_size=self.config.batch_size,
            show_progress_bar=True,
        ).astype(np.float32)

        # Stage 4: Chunk-level dedup (embedding-based)
        unique_chunks, unique_embeddings = self._chunk_level_dedup(
            all_chunks, embeddings,
        )
        stats.chunk_duplicates_removed = len(all_chunks) - len(unique_chunks)
        stats.unique_chunks_indexed = len(unique_chunks)

        # Stage 5: Add to index
        self._add_to_index(unique_chunks, unique_embeddings)

        stats.dedup_rate = (
            (stats.doc_duplicates_removed + stats.chunk_duplicates_removed)
            / max(stats.total_documents + stats.total_chunks, 1)
        )
        stats.processing_time_seconds = time.time() - start_time

        self._save_state()
        self._log_stats(stats)

        return stats

    def _doc_level_dedup(self, documents: list[dict]) -> list[dict]:
        """Remove exact and near-exact document duplicates using hashing."""
        unique = []
        for doc in documents:
            # Normalize text for near-exact matching
            normalized = " ".join(doc["text"].lower().split())
            doc_hash = hashlib.sha256(normalized.encode()).hexdigest()

            if doc_hash not in self.doc_hashes:
                self.doc_hashes.add(doc_hash)
                unique.append(doc)

        return unique

    def _chunk_document(self, document: dict) -> list[dict]:
        """Split a document into chunks with metadata."""
        text = document["text"]
        chunks = []

        step = self.config.chunk_size - self.config.chunk_overlap
        for i in range(0, len(text), step):
            chunk_text = text[i:i + self.config.chunk_size].strip()
            if len(chunk_text) < 50:  # Skip very short chunks
                continue

            chunks.append({
                "text": chunk_text,
                "doc_id": document["id"],
                "chunk_index": len(chunks),
                "metadata": document.get("metadata", {}),
            })

        return chunks

    def _chunk_level_dedup(
        self,
        chunks: list[dict],
        embeddings: np.ndarray,
    ) -> tuple[list[dict], np.ndarray]:
        """Remove semantically duplicate chunks using embedding similarity."""
        if self.chunk_index is None or self.chunk_index.ntotal == 0:
            # No existing index -- dedup within the batch only
            return self._dedup_within_batch(chunks, embeddings)

        # Check new chunks against existing index
        keep_mask = np.ones(len(chunks), dtype=bool)

        scores, indices = self.chunk_index.search(embeddings, self.config.n_neighbors)

        for i in range(len(chunks)):
            for score in scores[i]:
                if score >= self.config.chunk_similarity_threshold:
                    keep_mask[i] = False
                    break

        # Also dedup within the new batch
        for i in range(len(chunks)):
            if not keep_mask[i]:
                continue
            for j in range(i + 1, len(chunks)):
                if not keep_mask[j]:
                    continue
                sim = np.dot(embeddings[i], embeddings[j])
                if sim >= self.config.chunk_similarity_threshold:
                    keep_mask[j] = False

        unique_chunks = [c for c, keep in zip(chunks, keep_mask) if keep]
        unique_embeddings = embeddings[keep_mask]

        return unique_chunks, unique_embeddings

    def _dedup_within_batch(
        self,
        chunks: list[dict],
        embeddings: np.ndarray,
    ) -> tuple[list[dict], np.ndarray]:
        """Dedup within a single batch (no existing index)."""
        keep_mask = np.ones(len(chunks), dtype=bool)

        # Use FAISS for efficiency
        index = faiss.IndexFlatIP(self.dim)
        index.add(embeddings)

        scores, indices = index.search(embeddings, min(20, len(chunks)))

        for i in range(len(chunks)):
            if not keep_mask[i]:
                continue
            for score, idx in zip(scores[i], indices[i]):
                if idx <= i or idx == -1:
                    continue
                if score >= self.config.chunk_similarity_threshold:
                    keep_mask[idx] = False

        unique_chunks = [c for c, keep in zip(chunks, keep_mask) if keep]
        unique_embeddings = embeddings[keep_mask]

        return unique_chunks, unique_embeddings

    def _add_to_index(self, chunks: list[dict], embeddings: np.ndarray):
        """Add unique chunks to the persistent index."""
        if self.chunk_index is None:
            self.chunk_index = faiss.IndexFlatIP(self.dim)

        self.chunk_index.add(embeddings)
        self.chunk_texts.extend([c["text"] for c in chunks])
        self.chunk_metadata.extend(chunks)

    def _save_state(self):
        """Persist the dedup state to disk."""
        # Save FAISS index
        if self.chunk_index is not None:
            faiss.write_index(self.chunk_index, str(self.index_dir / "chunk_index.faiss"))

        # Save hashes
        with open(self.index_dir / "doc_hashes.json", "w") as f:
            json.dump(list(self.doc_hashes), f)

        # Save metadata
        with open(self.index_dir / "chunk_metadata.json", "w") as f:
            json.dump(self.chunk_metadata, f)

    def _load_state(self):
        """Load persisted state from disk."""
        index_path = self.index_dir / "chunk_index.faiss"
        if index_path.exists():
            self.chunk_index = faiss.read_index(str(index_path))

        hashes_path = self.index_dir / "doc_hashes.json"
        if hashes_path.exists():
            with open(hashes_path) as f:
                self.doc_hashes = set(json.load(f))

        meta_path = self.index_dir / "chunk_metadata.json"
        if meta_path.exists():
            with open(meta_path) as f:
                self.chunk_metadata = json.load(f)

    def _log_stats(self, stats: DedupStats):
        """Log dedup statistics."""
        print(f"\n{'='*50}")
        print(f"Dedup Pipeline Stats:")
        print(f"  Documents: {stats.total_documents} input, {stats.doc_duplicates_removed} doc-dupes")
        print(f"  Chunks: {stats.total_chunks} total, {stats.chunk_duplicates_removed} chunk-dupes")
        print(f"  Indexed: {stats.unique_chunks_indexed} unique chunks")
        print(f"  Dedup rate: {stats.dedup_rate*100:.1f}%")
        print(f"  Time: {stats.processing_time_seconds:.1f}s")
        print(f"{'='*50}\n")
```

---

## Handling Incremental Updates

When new documents arrive, check them against the existing index.

```python
def check_new_document(
    pipeline: ProductionDedupPipeline,
    document: dict,
) -> dict:
    """Check if a new document is a duplicate of an existing one.

    Returns:
        Dictionary with is_duplicate flag and details.
    """
    # Check document-level hash
    normalized = " ".join(document["text"].lower().split())
    doc_hash = hashlib.sha256(normalized.encode()).hexdigest()

    if doc_hash in pipeline.doc_hashes:
        return {"is_duplicate": True, "level": "exact", "action": "skip"}

    # Check chunk-level similarity
    chunks = pipeline._chunk_document(document)
    chunk_texts = [c["text"] for c in chunks]
    embeddings = pipeline.model.encode(
        chunk_texts, normalize_embeddings=True
    ).astype(np.float32)

    if pipeline.chunk_index is not None and pipeline.chunk_index.ntotal > 0:
        scores, indices = pipeline.chunk_index.search(embeddings, 5)
        max_sim = scores.max()

        if max_sim >= pipeline.config.chunk_similarity_threshold:
            return {
                "is_duplicate": True,
                "level": "semantic",
                "max_similarity": float(max_sim),
                "action": "skip_or_update",
            }

    return {"is_duplicate": False, "action": "ingest"}
```

---

## Monitoring Dedup Rate

```python
from datetime import datetime
import json


class DedupMonitor:
    """Monitor dedup rates over time to detect anomalies."""

    def __init__(self, log_path: str = "dedup_metrics.jsonl"):
        self.log_path = log_path

    def record(self, stats: DedupStats):
        """Record dedup stats for monitoring."""
        record = {
            "timestamp": datetime.utcnow().isoformat(),
            **asdict(stats),
        }

        with open(self.log_path, "a") as f:
            f.write(json.dumps(record) + "\n")

    def check_anomaly(self, stats: DedupStats, history_window: int = 30) -> list[str]:
        """Check if current dedup rate is anomalous.

        A sudden spike in dedup rate may indicate a data pipeline issue
        (e.g., the same batch being ingested twice).
        A sudden drop may indicate a preprocessing change that broke dedup.
        """
        alerts = []

        # Load recent history
        try:
            with open(self.log_path) as f:
                records = [json.loads(line) for line in f.readlines()[-history_window:]]
        except FileNotFoundError:
            return alerts

        if len(records) < 5:
            return alerts

        historical_rates = [r["dedup_rate"] for r in records]
        avg_rate = np.mean(historical_rates)
        std_rate = np.std(historical_rates)

        # Current rate anomaly
        if stats.dedup_rate > avg_rate + 3 * std_rate:
            alerts.append(
                f"HIGH_DEDUP_RATE: {stats.dedup_rate:.1%} vs avg {avg_rate:.1%}. "
                f"Possible duplicate batch ingestion."
            )

        if stats.dedup_rate < avg_rate - 3 * std_rate and avg_rate > 0.05:
            alerts.append(
                f"LOW_DEDUP_RATE: {stats.dedup_rate:.1%} vs avg {avg_rate:.1%}. "
                f"Preprocessing may have changed."
            )

        return alerts
```

---

## Dagster Integration

```python
from dagster import asset, op, job, Out, In
import json


@op
def fetch_new_documents() -> list[dict]:
    """Fetch new documents from the data source."""
    # Implementation depends on your data source
    # Example: read from S3, database, or API
    return [
        {"id": "doc_1", "text": "Document content here...", "metadata": {}},
    ]


@op
def run_dedup_pipeline(documents: list[dict]) -> dict:
    """Run the dedup pipeline on new documents."""
    config = DedupConfig(
        embedding_model="BAAI/bge-base-en-v1.5",
        chunk_similarity_threshold=0.95,
    )
    pipeline = ProductionDedupPipeline(config, index_dir="./production_dedup_index")
    stats = pipeline.ingest_documents(documents)
    return asdict(stats)


@op
def check_dedup_health(stats: dict) -> dict:
    """Check dedup statistics for anomalies."""
    monitor = DedupMonitor()
    dedup_stats = DedupStats(**stats)
    monitor.record(dedup_stats)
    alerts = monitor.check_anomaly(dedup_stats)

    if alerts:
        print(f"ALERTS: {alerts}")
        # Send to Slack/PagerDuty

    return {"stats": stats, "alerts": alerts}


@job
def dedup_ingestion_job():
    """Dagster job for dedup ingestion pipeline."""
    documents = fetch_new_documents()
    stats = run_dedup_pipeline(documents)
    check_dedup_health(stats)
```

---

## Prefect Integration

```python
from prefect import flow, task
from prefect.tasks import task_input_hash
from datetime import timedelta


@task(cache_key_fn=task_input_hash, cache_expiration=timedelta(hours=1))
def embed_documents(texts: list[str], model_name: str) -> np.ndarray:
    """Embed documents with caching."""
    model = SentenceTransformer(model_name)
    return model.encode(texts, normalize_embeddings=True, batch_size=256).astype(np.float32)


@task
def dedup_and_index(
    documents: list[dict],
    config_dict: dict,
) -> dict:
    """Run dedup pipeline and return stats."""
    config = DedupConfig(**config_dict)
    pipeline = ProductionDedupPipeline(config)
    stats = pipeline.ingest_documents(documents)
    return asdict(stats)


@flow(name="dedup-ingestion")
def dedup_ingestion_flow(
    documents: list[dict],
    config: dict | None = None,
):
    """Prefect flow for production dedup ingestion."""
    if config is None:
        config = {
            "embedding_model": "BAAI/bge-base-en-v1.5",
            "chunk_similarity_threshold": 0.95,
        }

    stats = dedup_and_index(documents, config)
    return stats


# Run the flow
# dedup_ingestion_flow(documents=[...])
```

---

## Production Recommendations

| Corpus Size | Recommended Approach | Expected Dedup Rate |
|-------------|---------------------|-------------------|
| <10K docs | Hash + DBSCAN | 5-15% |
| 10K-100K docs | Hash + FAISS | 10-20% |
| 100K-1M docs | Hash + FAISS + batching | 10-25% |
| 1M-10M docs | Hierarchical (MinHash + FAISS) | 15-30% |
| >10M docs | Hierarchical + distributed | 15-35% |

---

## Common Pitfalls

1. **Not persisting dedup state.** If the dedup index is lost between runs, previously-seen documents will be re-ingested as "new." Always save the hash set and FAISS index to disk.
2. **Deduplicating too aggressively.** A threshold of 0.90 removes documents that are similar but discuss different aspects of the same topic. Start at 0.95 and adjust based on manual review.
3. **Not handling updates to existing documents.** When a document is updated, the old version should be removed and the new version ingested. Without explicit update handling, both versions remain.
4. **Ignoring dedup monitoring.** A sudden spike in dedup rate (>50%) indicates a data pipeline bug (double ingestion). A sudden drop indicates a preprocessing change.
5. **Running dedup only at ingestion.** Periodic full-corpus dedup catches duplicates that incremental dedup misses (e.g., documents ingested before the dedup pipeline was deployed).

---

## References

- datasketch Library -- https://github.com/ekzhu/datasketch
- FAISS -- https://github.com/facebookresearch/faiss
- Dagster Documentation -- https://docs.dagster.io/
- Prefect Documentation -- https://docs.prefect.io/
- Union-Find for Dedup -- https://en.wikipedia.org/wiki/Disjoint-set_data_structure
