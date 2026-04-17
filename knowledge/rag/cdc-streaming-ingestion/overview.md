# CDC Streaming Ingestion -- Real-Time RAG Updates

## TL;DR

Change Data Capture (CDC) enables real-time RAG index updates by streaming database changes (inserts, updates, deletes) directly to the vector store. Instead of periodic batch ingestion that leaves the index stale for hours, CDC captures every mutation as it happens and propagates it through the embedding pipeline within seconds to minutes. Debezium is the standard open-source CDC platform, streaming database changelogs via Kafka to downstream consumers that process, embed, and upsert vectors. This overview covers CDC architecture for RAG, when real-time updates matter vs batch, and the key challenges of exactly-once semantics, tombstone handling, and schema evolution.

---

## When Real-Time RAG Updates Matter

### Staleness Impact by Domain

| Domain | Acceptable Staleness | Impact of Stale Data |
|--------|---------------------|---------------------|
| Customer support KB | 1-4 hours | Agents give outdated answers |
| Product catalog | Minutes | Wrong prices, missing products |
| Compliance/legal | Minutes | Regulatory violations |
| Internal wiki | 1-24 hours | Low impact, batch is fine |
| Code documentation | Hours-days | Low impact, batch is fine |
| Financial data | Seconds | Trading decisions on stale data |
| Incident response | Seconds | Missing critical updates |

### Batch vs CDC Decision Framework

```python
def recommend_ingestion_strategy(
    data_change_frequency: str,
    acceptable_staleness_minutes: int,
    source_type: str,
    document_count: int,
) -> dict:
    """Recommend batch vs CDC based on requirements."""
    if acceptable_staleness_minutes >= 60:
        return {
            "strategy": "batch",
            "reasoning": (
                f"Staleness tolerance of {acceptable_staleness_minutes} minutes "
                "is well served by hourly or daily batch ingestion."
            ),
            "recommended_schedule": (
                "hourly" if acceptable_staleness_minutes < 240 else "daily"
            ),
        }

    if source_type not in ("postgresql", "mysql", "mongodb", "sqlserver"):
        return {
            "strategy": "batch_with_polling",
            "reasoning": (
                f"CDC is not well supported for {source_type}. "
                "Use frequent polling (every 1-5 minutes) instead."
            ),
        }

    return {
        "strategy": "cdc",
        "reasoning": (
            f"Staleness tolerance of {acceptable_staleness_minutes} minutes "
            f"with {data_change_frequency} changes requires CDC."
        ),
        "recommended_stack": "Debezium + Kafka + custom consumer",
        "estimated_latency": "1-30 seconds end-to-end",
    }
```

---

## CDC Architecture for RAG

```
Source Database (PostgreSQL, MySQL, MongoDB)
    |
    | WAL / binlog / oplog
    v
[Debezium Connector]
    |
    | CDC events (JSON)
    v
[Apache Kafka]
    |
    | topic: db.schema.table
    v
[RAG Consumer Service]
    |
    +---> [Parse Change Event]
    |         |
    |         v
    |     [Extract Text Content]
    |         |
    |         v
    |     [Chunk Content]
    |         |
    |         v
    |     [Embed Chunks]   <--- embedding API
    |         |
    |         v
    |     [Upsert/Delete Vectors]  <--- vector DB
    |
    v
[Offset Commit]  (exactly-once guarantee)
```

### Core Components

```python
from dataclasses import dataclass
from enum import Enum
from typing import Any


class ChangeOperation(Enum):
    CREATE = "c"
    UPDATE = "u"
    DELETE = "d"
    READ = "r"      # snapshot read (initial load)
    TRUNCATE = "t"


@dataclass
class CDCEvent:
    """Represents a single CDC change event from Debezium."""
    operation: ChangeOperation
    table: str
    schema: str
    database: str
    before: dict[str, Any] | None   # row state before change
    after: dict[str, Any] | None    # row state after change
    timestamp_ms: int
    transaction_id: str | None = None
    source_offset: dict | None = None

    @property
    def primary_key(self) -> str:
        """Extract primary key from the event."""
        record = self.after or self.before
        if record and "id" in record:
            return str(record["id"])
        return ""

    @property
    def content_changed(self) -> bool:
        """Check if text content actually changed (for updates)."""
        if self.operation != ChangeOperation.UPDATE:
            return True
        if not self.before or not self.after:
            return True
        # Compare text-relevant fields
        text_fields = {"title", "content", "body", "description", "text"}
        for field in text_fields:
            if self.before.get(field) != self.after.get(field):
                return True
        return False


def parse_debezium_event(raw_event: dict) -> CDCEvent:
    """Parse a raw Debezium JSON event into a CDCEvent."""
    payload = raw_event.get("payload", raw_event)
    source = payload.get("source", {})

    op_map = {"c": ChangeOperation.CREATE, "u": ChangeOperation.UPDATE,
              "d": ChangeOperation.DELETE, "r": ChangeOperation.READ,
              "t": ChangeOperation.TRUNCATE}

    return CDCEvent(
        operation=op_map.get(payload.get("op", ""), ChangeOperation.READ),
        table=source.get("table", ""),
        schema=source.get("schema", ""),
        database=source.get("db", ""),
        before=payload.get("before"),
        after=payload.get("after"),
        timestamp_ms=payload.get("ts_ms", 0),
        transaction_id=payload.get("transaction", {}).get("id") if payload.get("transaction") else None,
    )
```

---

## Kafka Consumer for RAG Updates

```python
import json
import logging
from confluent_kafka import Consumer, KafkaError

logger = logging.getLogger(__name__)


class RAGCDCConsumer:
    """Kafka consumer that processes CDC events for RAG updates.

    Processes events in micro-batches for efficiency:
    - Accumulate events for a short window (1-5 seconds)
    - Batch embed all new/updated content
    - Batch upsert/delete in the vector store
    - Commit Kafka offsets after successful processing
    """

    def __init__(
        self,
        kafka_config: dict,
        topics: list[str],
        embedding_model,
        vector_store,
        batch_window_seconds: float = 2.0,
        max_batch_size: int = 50,
    ):
        self.consumer = Consumer(kafka_config)
        self.consumer.subscribe(topics)
        self.embedding_model = embedding_model
        self.vector_store = vector_store
        self.batch_window = batch_window_seconds
        self.max_batch = max_batch_size
        self.running = True

    def run(self) -> None:
        """Main consumer loop."""
        logger.info("CDC consumer started")

        while self.running:
            # Collect a micro-batch of events
            events = self._collect_batch()

            if not events:
                continue

            try:
                self._process_batch(events)
                self.consumer.commit()
                logger.info(f"Processed batch of {len(events)} events")
            except Exception as e:
                logger.error(f"Batch processing failed: {e}")
                # Do not commit offsets; events will be redelivered

    def _collect_batch(self) -> list[CDCEvent]:
        """Collect events until batch window expires or batch is full."""
        import time
        events = []
        deadline = time.time() + self.batch_window

        while time.time() < deadline and len(events) < self.max_batch:
            msg = self.consumer.poll(timeout=0.1)

            if msg is None:
                continue
            if msg.error():
                if msg.error().code() == KafkaError._PARTITION_EOF:
                    continue
                logger.error(f"Kafka error: {msg.error()}")
                continue

            try:
                raw = json.loads(msg.value().decode("utf-8"))
                event = parse_debezium_event(raw)
                events.append(event)
            except Exception as e:
                logger.warning(f"Failed to parse event: {e}")

        return events

    def _process_batch(self, events: list[CDCEvent]) -> None:
        """Process a batch of CDC events."""
        to_upsert = []
        to_delete = []

        for event in events:
            if event.operation == ChangeOperation.DELETE:
                to_delete.append(event.primary_key)
            elif event.operation == ChangeOperation.TRUNCATE:
                self._handle_truncate(event)
            elif event.operation in (
                ChangeOperation.CREATE,
                ChangeOperation.UPDATE,
                ChangeOperation.READ,
            ):
                if event.content_changed:
                    to_upsert.append(event)

        # Batch delete
        if to_delete:
            self._batch_delete(to_delete)

        # Batch upsert (embed + index)
        if to_upsert:
            self._batch_upsert(to_upsert)

    def _batch_upsert(self, events: list[CDCEvent]) -> None:
        """Extract text, chunk, embed, and upsert vectors."""
        texts = []
        metadata_list = []

        for event in events:
            record = event.after
            if not record:
                continue

            # Extract text content from the record
            text = self._extract_text(record)
            if not text:
                continue

            texts.append(text)
            metadata_list.append({
                "document_id": event.primary_key,
                "table": event.table,
                "schema": event.schema,
                "timestamp": event.timestamp_ms,
                "content": text[:1000],  # store truncated for retrieval
            })

        if not texts:
            return

        # Embed
        embeddings = self.embedding_model.embed(texts)

        # Upsert to vector store
        ids = [m["document_id"] for m in metadata_list]
        self.vector_store.upsert(
            ids=ids,
            embeddings=embeddings,
            metadatas=metadata_list,
        )

    def _batch_delete(self, document_ids: list[str]) -> None:
        """Delete vectors for removed documents."""
        self.vector_store.delete(ids=document_ids)

    def _handle_truncate(self, event: CDCEvent) -> None:
        """Handle table truncation (delete all vectors for this table)."""
        logger.warning(f"TRUNCATE on {event.schema}.{event.table}")
        self.vector_store.delete(
            filter={"table": event.table, "schema": event.schema}
        )

    @staticmethod
    def _extract_text(record: dict) -> str:
        """Extract text content from a database record."""
        text_fields = ["content", "body", "text", "description", "title"]
        parts = []
        for field in text_fields:
            if field in record and record[field]:
                parts.append(str(record[field]))
        return "\n\n".join(parts)

    def stop(self) -> None:
        """Gracefully stop the consumer."""
        self.running = False
        self.consumer.close()
```

---

## Common Pitfalls

1. **Not handling tombstone events.** Debezium sends a null-valued message (tombstone) after a delete event for log compaction. Your consumer must handle null payloads gracefully.

2. **Embedding every update, even metadata-only changes.** If only a `last_modified` timestamp changed, re-embedding is wasteful. Check if text content actually changed before re-embedding.

3. **Not committing offsets atomically with vector store writes.** If you commit offsets before the vector store confirms the write, a crash loses the vectors but the events are not replayed.

4. **Ignoring schema evolution.** When a database column is added or renamed, the CDC event structure changes. Use schema registry (Confluent, Apicurio) to handle evolution gracefully.

5. **Processing events one at a time.** Individual API calls for each event are 10-50x slower than micro-batching. Accumulate events for 1-5 seconds, then batch process.

6. **Not handling the initial snapshot.** Debezium snapshots the entire table on first connect, producing READ events for every existing row. Size your consumer to handle the burst.

---

## References

- Debezium: https://debezium.io/documentation/
- Confluent Kafka Python: https://docs.confluent.io/platform/current/clients/confluent-kafka-python/
- Apache Kafka: https://kafka.apache.org/documentation/
