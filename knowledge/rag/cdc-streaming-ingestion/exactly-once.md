# CDC Streaming Ingestion -- Exactly-Once Semantics: Tombstones, Upserts, and Schema Evolution

## TL;DR

Achieving exactly-once processing in a CDC-to-vector-store pipeline is the hardest engineering challenge in real-time RAG. Without it, vectors can be duplicated (at-least-once) or lost (at-most-once) during failures. This article covers three critical aspects: (1) tombstone handling for reliable deletes, (2) idempotent upserts that make at-least-once delivery safe, and (3) schema evolution strategies for when source database schemas change without breaking the pipeline.

---

## The Exactly-Once Problem

### Why It Matters for RAG

```python
def illustrate_delivery_semantics() -> dict:
    """Show the impact of different delivery semantics on RAG quality."""
    return {
        "at_most_once": {
            "mechanism": "Commit offset before processing",
            "failure_mode": "Consumer crashes after commit but before vector upsert",
            "rag_impact": "Missing vectors: documents not searchable",
            "severity": "HIGH: silent data loss, degraded retrieval",
        },
        "at_least_once": {
            "mechanism": "Commit offset after processing",
            "failure_mode": "Consumer crashes after vector upsert but before commit",
            "rag_impact": "Duplicate processing on replay, but idempotent upserts prevent duplication",
            "severity": "LOW if upserts are idempotent, HIGH if they are not",
        },
        "exactly_once": {
            "mechanism": "Atomic: process + commit in same transaction",
            "failure_mode": "Requires transactional support in both Kafka and vector store",
            "rag_impact": "No duplicates, no losses",
            "severity": "NONE (but complex to implement)",
        },
    }
```

### Practical Approach: Idempotent At-Least-Once

True exactly-once requires distributed transactions, which most vector stores do not support. The practical approach is at-least-once delivery with idempotent operations:

```python
import hashlib
import json
import time
from dataclasses import dataclass, field


@dataclass
class ProcessingState:
    """Track processing state for exactly-once semantics."""
    last_offset: dict[str, int] = field(default_factory=dict)  # partition -> offset
    processed_ids: set[str] = field(default_factory=set)  # dedup within window
    dedup_window_seconds: float = 300  # 5 minute dedup window
    _timestamps: dict[str, float] = field(default_factory=dict)

    def is_duplicate(self, event_id: str) -> bool:
        """Check if this event was already processed."""
        if event_id in self.processed_ids:
            return True
        return False

    def mark_processed(self, event_id: str) -> None:
        """Mark an event as processed."""
        self.processed_ids.add(event_id)
        self._timestamps[event_id] = time.time()
        self._cleanup_old_entries()

    def _cleanup_old_entries(self) -> None:
        """Remove entries older than the dedup window."""
        cutoff = time.time() - self.dedup_window_seconds
        expired = [
            eid for eid, ts in self._timestamps.items()
            if ts < cutoff
        ]
        for eid in expired:
            self.processed_ids.discard(eid)
            del self._timestamps[eid]


class IdempotentCDCProcessor:
    """Process CDC events with idempotent upserts.

    Idempotent upserts ensure that processing the same event twice
    produces the same result. This is achieved by:
    1. Using deterministic vector IDs (hash of document ID)
    2. Using upsert (not insert) operations
    3. Tracking processed event IDs for short-term dedup
    """

    def __init__(self, embedding_model, vector_store):
        self.embedding_model = embedding_model
        self.vector_store = vector_store
        self.state = ProcessingState()

    def process_event(self, event: dict) -> dict:
        """Process a single CDC event idempotently."""
        event_id = self._compute_event_id(event)

        # Dedup check
        if self.state.is_duplicate(event_id):
            return {"status": "skipped", "reason": "duplicate"}

        operation = event.get("op", event.get("__op", ""))

        if operation in ("c", "u", "r"):
            result = self._handle_upsert(event)
        elif operation == "d":
            result = self._handle_delete(event)
        else:
            result = {"status": "skipped", "reason": f"unknown op: {operation}"}

        self.state.mark_processed(event_id)
        return result

    def _handle_upsert(self, event: dict) -> dict:
        """Idempotent upsert: same input always produces same output."""
        record = event.get("after", event)
        if not record:
            return {"status": "skipped", "reason": "no after state"}

        # Deterministic vector ID from document ID
        doc_id = str(record.get("id", ""))
        vector_id = self._deterministic_id(doc_id)

        # Extract and embed text
        text = self._extract_text(record)
        if not text:
            return {"status": "skipped", "reason": "no text content"}

        embedding = self.embedding_model.embed([text])[0]

        # Upsert (idempotent: same doc_id always overwrites)
        self.vector_store.upsert(
            ids=[vector_id],
            embeddings=[embedding],
            metadatas=[{
                "document_id": doc_id,
                "content": text[:1000],
                "table": event.get("__table", ""),
                "updated_at": event.get("__source_ts_ms", 0),
            }],
        )

        return {"status": "upserted", "document_id": doc_id}

    def _handle_delete(self, event: dict) -> dict:
        """Idempotent delete: deleting a non-existent vector is a no-op."""
        record = event.get("before", event)
        if not record:
            return {"status": "skipped", "reason": "no before state"}

        doc_id = str(record.get("id", ""))
        vector_id = self._deterministic_id(doc_id)

        # Delete is idempotent: deleting non-existent ID is safe
        self.vector_store.delete(ids=[vector_id])
        return {"status": "deleted", "document_id": doc_id}

    @staticmethod
    def _deterministic_id(doc_id: str) -> str:
        """Generate a deterministic vector ID from a document ID."""
        return hashlib.sha256(doc_id.encode()).hexdigest()[:32]

    @staticmethod
    def _extract_text(record: dict) -> str:
        """Extract text content from a database record."""
        text_fields = ["content", "body", "text", "description", "title"]
        parts = []
        for field in text_fields:
            if field in record and record[field]:
                parts.append(str(record[field]))
        return "\n\n".join(parts)

    @staticmethod
    def _compute_event_id(event: dict) -> str:
        """Compute a unique ID for deduplication."""
        key_parts = [
            str(event.get("__table", "")),
            str(event.get("id", event.get("before", {}).get("id", ""))),
            str(event.get("__source_ts_ms", "")),
            str(event.get("op", event.get("__op", ""))),
        ]
        return hashlib.md5("|".join(key_parts).encode()).hexdigest()
```

---

## Tombstone Handling

### What Are Tombstones

Kafka tombstones are messages with a null value. Debezium emits tombstones after delete events to enable Kafka log compaction:

```python
class TombstoneHandler:
    """Handle Kafka tombstones in the CDC pipeline.

    Debezium delete flow:
    1. Delete event: key={id: 1}, value={before: {id: 1, name: "doc"}, op: "d"}
    2. Tombstone:    key={id: 1}, value=null

    The tombstone tells Kafka to remove the key during log compaction.
    Your consumer must handle both:
    - Delete event: remove vector from index
    - Tombstone: acknowledge and skip (already handled by delete event)
    """

    def __init__(self, processor: IdempotentCDCProcessor):
        self.processor = processor

    def handle_message(self, key: bytes | None, value: bytes | None) -> dict:
        """Handle a Kafka message, including tombstones."""
        # Tombstone: null value
        if value is None:
            if key:
                # Extract document ID from key
                try:
                    key_data = json.loads(key.decode("utf-8"))
                    doc_id = str(key_data.get("id", key_data))
                    return {
                        "status": "tombstone_acknowledged",
                        "document_id": doc_id,
                    }
                except (json.JSONDecodeError, UnicodeDecodeError):
                    pass
            return {"status": "tombstone_skipped", "reason": "unparseable key"}

        # Normal message
        try:
            event = json.loads(value.decode("utf-8"))

            # Check for Debezium's delete handling mode "rewrite"
            # which adds a __deleted field
            if event.get("__deleted") == "true":
                return self.processor._handle_delete(event)

            return self.processor.process_event(event)

        except json.JSONDecodeError as e:
            return {"status": "error", "reason": f"Invalid JSON: {e}"}
```

---

## Schema Evolution

### Handling Database Schema Changes

```python
from dataclasses import dataclass
from enum import Enum


class SchemaChangeType(Enum):
    COLUMN_ADDED = "column_added"
    COLUMN_REMOVED = "column_removed"
    COLUMN_RENAMED = "column_renamed"
    COLUMN_TYPE_CHANGED = "column_type_changed"
    TABLE_RENAMED = "table_renamed"


@dataclass
class SchemaVersion:
    version: int
    text_fields: list[str]
    metadata_fields: list[str]
    primary_key_field: str


class SchemaEvolutionHandler:
    """Handle database schema changes without breaking the CDC pipeline.

    Schema evolution scenarios:
    1. Column added: new field in CDC events (backward compatible)
    2. Column removed: field missing from CDC events (handle gracefully)
    3. Column renamed: old field gone, new field appears
    4. Type changed: field value format changes
    5. Table renamed: topic name changes

    Strategy: define schema versions and handle events based on their version.
    """

    def __init__(self):
        self.schemas: dict[int, SchemaVersion] = {}
        self.current_version: int = 0

    def register_schema(self, schema: SchemaVersion) -> None:
        """Register a schema version."""
        self.schemas[schema.version] = schema
        self.current_version = max(self.current_version, schema.version)

    def extract_text(self, record: dict, schema_version: int | None = None) -> str:
        """Extract text from a record using the appropriate schema version.

        If schema_version is not provided, try the current version
        then fall back to older versions.
        """
        if schema_version and schema_version in self.schemas:
            return self._extract_with_schema(record, self.schemas[schema_version])

        # Try current version first, then fall back
        for version in sorted(self.schemas.keys(), reverse=True):
            text = self._extract_with_schema(record, self.schemas[version])
            if text:
                return text

        # Last resort: try common field names
        return self._extract_generic(record)

    def _extract_with_schema(
        self, record: dict, schema: SchemaVersion
    ) -> str:
        """Extract text using a specific schema version."""
        parts = []
        for field in schema.text_fields:
            value = record.get(field)
            if value:
                parts.append(str(value))
        return "\n\n".join(parts)

    @staticmethod
    def _extract_generic(record: dict) -> str:
        """Fallback: extract from common text field names."""
        common_fields = [
            "content", "body", "text", "description",
            "title", "summary", "name",
        ]
        parts = []
        for field in common_fields:
            value = record.get(field)
            if value and isinstance(value, str) and len(value) > 10:
                parts.append(value)
        return "\n\n".join(parts)

    def detect_schema_change(
        self, current_fields: set[str], expected_fields: set[str]
    ) -> list[dict]:
        """Detect what changed between expected and actual record fields."""
        changes = []

        added = current_fields - expected_fields
        removed = expected_fields - current_fields

        for field in added:
            changes.append({
                "type": SchemaChangeType.COLUMN_ADDED.value,
                "field": field,
                "action": "Consider adding to text_fields if it contains relevant content",
            })

        for field in removed:
            changes.append({
                "type": SchemaChangeType.COLUMN_REMOVED.value,
                "field": field,
                "action": "Remove from text_fields to avoid KeyError",
            })

        return changes


# Example usage:
# handler = SchemaEvolutionHandler()
# handler.register_schema(SchemaVersion(
#     version=1,
#     text_fields=["title", "content"],
#     metadata_fields=["author", "category"],
#     primary_key_field="id",
# ))
# handler.register_schema(SchemaVersion(
#     version=2,
#     text_fields=["title", "content", "summary"],  # summary added
#     metadata_fields=["author", "category", "tags"],
#     primary_key_field="id",
# ))
```

---

## Offset Management

```python
from pathlib import Path


class OffsetManager:
    """Manage Kafka consumer offsets for reliable processing.

    Offset commit strategies:
    1. Auto-commit (risky): offsets committed on a timer, not tied to processing
    2. Manual after batch (recommended): commit after successful batch processing
    3. External store (safest): store offsets alongside processing results

    This implementation uses manual commit after each successful batch.
    """

    def __init__(self, consumer, checkpoint_path: str = "./offsets.json"):
        self.consumer = consumer
        self.checkpoint_path = Path(checkpoint_path)
        self._pending_offsets: dict[tuple[str, int], int] = {}

    def record_offset(self, topic: str, partition: int, offset: int) -> None:
        """Record an offset for later commit."""
        key = (topic, partition)
        current = self._pending_offsets.get(key, -1)
        if offset > current:
            self._pending_offsets[key] = offset

    def commit(self) -> None:
        """Commit all pending offsets to Kafka."""
        if not self._pending_offsets:
            return

        from confluent_kafka import TopicPartition
        offsets = [
            TopicPartition(topic, partition, offset + 1)
            for (topic, partition), offset in self._pending_offsets.items()
        ]
        self.consumer.commit(offsets=offsets, asynchronous=False)
        self._save_checkpoint()
        self._pending_offsets.clear()

    def _save_checkpoint(self) -> None:
        """Save offsets to disk as backup."""
        data = {
            f"{topic}:{partition}": offset
            for (topic, partition), offset in self._pending_offsets.items()
        }
        self.checkpoint_path.write_text(json.dumps(data))

    def load_checkpoint(self) -> dict[tuple[str, int], int]:
        """Load offsets from disk checkpoint."""
        if not self.checkpoint_path.exists():
            return {}
        data = json.loads(self.checkpoint_path.read_text())
        return {
            tuple(k.split(":")): int(v) for k, v in data.items()
        }
```

---

## Common Pitfalls

1. **Using auto-commit for Kafka offsets.** Auto-commit may commit offsets before processing completes, causing data loss on crash. Always use manual commit after confirmed processing.

2. **Not making vector IDs deterministic.** If vector IDs are random UUIDs, replaying the same event creates a duplicate vector. Use `hash(document_id)` for deterministic IDs.

3. **Ignoring tombstones.** If your consumer crashes on null values, tombstone messages block the partition. Always handle null values gracefully.

4. **Hardcoding field names in the consumer.** When a column is renamed in the database, the consumer crashes. Use schema versioning with fallback extraction.

5. **Not monitoring consumer lag.** If your consumer falls behind (lag grows), events accumulate in Kafka. This is fine temporarily, but sustained lag means the pipeline cannot keep up.

6. **Deleting before checking multi-source.** If a vector was indexed from multiple sources, deleting it when one source removes the record incorrectly removes all data. Track provenance.

---

## References

- Kafka exactly-once semantics: https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/
- Debezium tombstones: https://debezium.io/documentation/reference/stable/transformations/event-flattening.html
- Schema Registry: https://docs.confluent.io/platform/current/schema-registry/
