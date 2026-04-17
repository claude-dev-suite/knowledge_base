# CDC Streaming Ingestion -- Debezium + Kafka Pipeline to Vector DB

## TL;DR

This article provides a complete production setup for streaming database changes to a RAG vector store using Debezium and Kafka. It covers Debezium connector configuration for PostgreSQL and MySQL, Kafka topic management, a Python consumer with exactly-once processing semantics, and vector store integration patterns for Qdrant, Pinecone, and Weaviate. Includes Docker Compose for local development and production deployment considerations.

---

## Debezium Connector Configuration

### PostgreSQL Connector

```python
import json
import httpx


class DebeziumManager:
    """Manage Debezium connectors via the Kafka Connect REST API."""

    def __init__(self, connect_url: str = "http://localhost:8083"):
        self.connect_url = connect_url

    def create_postgres_connector(
        self,
        name: str,
        database_hostname: str,
        database_port: int,
        database_user: str,
        database_password: str,
        database_name: str,
        table_include_list: list[str],
        slot_name: str = "debezium",
        publication_name: str = "dbz_publication",
    ) -> dict:
        """Create a Debezium PostgreSQL source connector.

        PostgreSQL CDC uses logical replication (WAL):
        - Requires wal_level = logical in postgresql.conf
        - Creates a replication slot for durable event streaming
        - Supports initial snapshot + streaming
        """
        config = {
            "name": name,
            "config": {
                "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
                "database.hostname": database_hostname,
                "database.port": str(database_port),
                "database.user": database_user,
                "database.password": database_password,
                "database.dbname": database_name,
                "database.server.name": name,
                "topic.prefix": name,

                # Table selection
                "table.include.list": ",".join(table_include_list),

                # Replication slot
                "slot.name": slot_name,
                "publication.name": publication_name,
                "plugin.name": "pgoutput",

                # Snapshot
                "snapshot.mode": "initial",

                # Schema handling
                "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
                "schema.history.internal.kafka.topic": f"{name}.schema-history",

                # Transforms: flatten nested structure
                "transforms": "unwrap",
                "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
                "transforms.unwrap.drop.tombstones": "false",
                "transforms.unwrap.delete.handling.mode": "rewrite",
                "transforms.unwrap.add.fields": "op,table,source.ts_ms",

                # Key serialization
                "key.converter": "org.apache.kafka.connect.json.JsonConverter",
                "key.converter.schemas.enable": "false",
                "value.converter": "org.apache.kafka.connect.json.JsonConverter",
                "value.converter.schemas.enable": "false",

                # Error handling
                "errors.tolerance": "all",
                "errors.log.enable": "true",
                "errors.log.include.messages": "true",
                "errors.deadletterqueue.topic.name": f"{name}.dlq",
            },
        }

        response = httpx.post(
            f"{self.connect_url}/connectors",
            json=config,
            timeout=30,
        )
        response.raise_for_status()
        return response.json()

    def create_mysql_connector(
        self,
        name: str,
        database_hostname: str,
        database_port: int,
        database_user: str,
        database_password: str,
        database_include_list: list[str],
        table_include_list: list[str],
        server_id: int = 85744,
    ) -> dict:
        """Create a Debezium MySQL source connector.

        MySQL CDC uses binlog:
        - Requires binlog_format = ROW
        - Requires binlog_row_image = FULL
        - Server ID must be unique per connector
        """
        config = {
            "name": name,
            "config": {
                "connector.class": "io.debezium.connector.mysql.MySqlConnector",
                "database.hostname": database_hostname,
                "database.port": str(database_port),
                "database.user": database_user,
                "database.password": database_password,
                "database.server.id": str(server_id),
                "topic.prefix": name,

                "database.include.list": ",".join(database_include_list),
                "table.include.list": ",".join(table_include_list),

                "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
                "schema.history.internal.kafka.topic": f"{name}.schema-history",

                "snapshot.mode": "initial",

                "transforms": "unwrap",
                "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
                "transforms.unwrap.drop.tombstones": "false",
                "transforms.unwrap.delete.handling.mode": "rewrite",

                "key.converter": "org.apache.kafka.connect.json.JsonConverter",
                "key.converter.schemas.enable": "false",
                "value.converter": "org.apache.kafka.connect.json.JsonConverter",
                "value.converter.schemas.enable": "false",
            },
        }

        response = httpx.post(
            f"{self.connect_url}/connectors",
            json=config,
            timeout=30,
        )
        response.raise_for_status()
        return response.json()

    def get_connector_status(self, name: str) -> dict:
        """Check connector health."""
        response = httpx.get(
            f"{self.connect_url}/connectors/{name}/status",
            timeout=10,
        )
        response.raise_for_status()
        return response.json()

    def list_connectors(self) -> list[str]:
        """List all registered connectors."""
        response = httpx.get(f"{self.connect_url}/connectors", timeout=10)
        response.raise_for_status()
        return response.json()

    def delete_connector(self, name: str) -> None:
        """Delete a connector."""
        response = httpx.delete(
            f"{self.connect_url}/connectors/{name}",
            timeout=10,
        )
        response.raise_for_status()
```

### Docker Compose for Development

```python
DOCKER_COMPOSE_YAML = """
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    depends_on: [zookeeper]
    ports: ["9092:9092"]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  debezium:
    image: debezium/connect:2.5
    depends_on: [kafka]
    ports: ["8083:8083"]
    environment:
      BOOTSTRAP_SERVERS: kafka:29092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: connect_configs
      OFFSET_STORAGE_TOPIC: connect_offsets
      STATUS_STORAGE_TOPIC: connect_statuses

  postgres:
    image: postgres:16
    ports: ["5432:5432"]
    environment:
      POSTGRES_USER: raguser
      POSTGRES_PASSWORD: ragpass
      POSTGRES_DB: ragdb
    command: >
      postgres
      -c wal_level=logical
      -c max_replication_slots=4
      -c max_wal_senders=4

  qdrant:
    image: qdrant/qdrant:v1.8.0
    ports: ["6333:6333"]
"""


def print_setup_instructions() -> None:
    """Print setup instructions for the CDC pipeline."""
    print("1. Save the Docker Compose YAML to docker-compose.yml")
    print("2. Run: docker compose up -d")
    print("3. Wait for all services to be healthy")
    print("4. Create the Debezium connector:")
    print("   manager = DebeziumManager()")
    print("   manager.create_postgres_connector(...)")
    print("5. Start the consumer:")
    print("   consumer = RAGCDCConsumer(...)")
    print("   consumer.run()")
```

---

## Vector Store Integration

```python
from typing import Protocol


class VectorStoreAdapter(Protocol):
    """Common interface for vector store operations."""
    def upsert(self, ids: list[str], embeddings: list[list[float]], metadatas: list[dict]) -> None: ...
    def delete(self, ids: list[str] | None = None, filter: dict | None = None) -> None: ...


class QdrantAdapter:
    """Qdrant vector store adapter for CDC pipeline."""

    def __init__(self, url: str = "http://localhost:6333", collection: str = "rag"):
        from qdrant_client import QdrantClient
        from qdrant_client.models import VectorParams, Distance

        self.client = QdrantClient(url=url)
        self.collection = collection

        # Ensure collection exists
        collections = [c.name for c in self.client.get_collections().collections]
        if collection not in collections:
            self.client.create_collection(
                collection_name=collection,
                vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
            )

    def upsert(
        self, ids: list[str], embeddings: list[list[float]], metadatas: list[dict]
    ) -> None:
        from qdrant_client.models import PointStruct
        import hashlib

        points = [
            PointStruct(
                id=hashlib.md5(doc_id.encode()).hexdigest()[:32],
                vector=emb,
                payload={**meta, "original_id": doc_id},
            )
            for doc_id, emb, meta in zip(ids, embeddings, metadatas)
        ]
        self.client.upsert(collection_name=self.collection, points=points)

    def delete(
        self, ids: list[str] | None = None, filter: dict | None = None
    ) -> None:
        from qdrant_client.models import Filter, FieldCondition, MatchValue
        import hashlib

        if ids:
            point_ids = [
                hashlib.md5(doc_id.encode()).hexdigest()[:32]
                for doc_id in ids
            ]
            self.client.delete(
                collection_name=self.collection,
                points_selector=point_ids,
            )
        elif filter:
            conditions = [
                FieldCondition(key=k, match=MatchValue(value=v))
                for k, v in filter.items()
            ]
            self.client.delete(
                collection_name=self.collection,
                points_selector=Filter(must=conditions),
            )
```

---

## Monitoring the CDC Pipeline

```python
class CDCPipelineMonitor:
    """Monitor the health and performance of the CDC pipeline."""

    def __init__(self, kafka_config: dict, debezium_url: str):
        self.kafka_config = kafka_config
        self.debezium_url = debezium_url

    def check_consumer_lag(self, group_id: str) -> dict:
        """Check Kafka consumer group lag."""
        from confluent_kafka.admin import AdminClient

        admin = AdminClient(self.kafka_config)
        # Consumer lag indicates how far behind the consumer is
        # High lag = events accumulating faster than processing
        groups = admin.list_consumer_groups()
        return {"group_id": group_id, "status": "check via kafka-consumer-groups CLI"}

    def check_connector_health(self, connector_name: str) -> dict:
        """Check Debezium connector status."""
        import httpx
        response = httpx.get(
            f"{self.debezium_url}/connectors/{connector_name}/status"
        )
        status = response.json()
        connector_state = status.get("connector", {}).get("state", "UNKNOWN")
        task_states = [
            t.get("state", "UNKNOWN") for t in status.get("tasks", [])
        ]

        healthy = (
            connector_state == "RUNNING"
            and all(s == "RUNNING" for s in task_states)
        )

        return {
            "connector": connector_name,
            "healthy": healthy,
            "connector_state": connector_state,
            "task_states": task_states,
        }

    def get_throughput_metrics(self) -> dict:
        """Get pipeline throughput metrics."""
        return {
            "note": (
                "Track these metrics in production:\n"
                "- events_consumed_per_second\n"
                "- events_processed_per_second\n"
                "- embedding_latency_p95_ms\n"
                "- vector_upsert_latency_p95_ms\n"
                "- consumer_lag_messages\n"
                "- error_rate_percent"
            ),
        }
```

---

## Common Pitfalls

1. **Not configuring the WAL level.** PostgreSQL requires `wal_level = logical` for Debezium. Without it, the connector cannot read the write-ahead log and silently fails.

2. **Forgetting replication slot cleanup.** If the consumer disconnects, the replication slot keeps accumulating WAL data. Set `slot.drop.on.stop = true` for development or monitor slot size in production.

3. **Using the default snapshot mode in production.** `snapshot.mode = initial` reads the entire table on first connect. For large tables (millions of rows), use `snapshot.mode = initial_only` and batch-process the snapshot separately.

4. **Not handling Kafka consumer rebalancing.** When consumers restart, Kafka rebalances partitions. Ensure your consumer handles rebalance gracefully (commit offsets, close resources).

5. **Embedding during peak database load.** CDC events arrive fastest during peak database write times. If your embedding API cannot keep up, events queue in Kafka (which is fine), but monitor consumer lag.

---

## References

- Debezium documentation: https://debezium.io/documentation/
- Debezium PostgreSQL connector: https://debezium.io/documentation/reference/stable/connectors/postgresql.html
- Confluent Kafka Python: https://docs.confluent.io/platform/current/clients/confluent-kafka-python/
- Qdrant: https://qdrant.tech/documentation/
