# Apache Kafka - Kafka Connect

## Overview

Kafka Connect is a framework for scalably and reliably streaming data between Apache Kafka and other systems. It provides ready-to-use connectors for common data sources and sinks.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kafka Connect Cluster                         │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │   Worker 1   │    │   Worker 2   │    │   Worker 3   │      │
│  │              │    │              │    │              │      │
│  │  Task 0      │    │  Task 1      │    │  Task 2      │      │
│  │  Task 3      │    │  Task 4      │    │  Task 5      │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│          │                  │                  │                │
│          └──────────────────┴──────────────────┘                │
│                             │                                    │
│  ┌──────────────────────────┴──────────────────────────────┐   │
│  │              Internal Topics                              │   │
│  │  connect-configs, connect-offsets, connect-status        │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
       ┌─────────────┐                 ┌─────────────┐
       │   Source    │                 │    Sink     │
       │  (Database) │                 │ (Database)  │
       └─────────────┘                 └─────────────┘
```

## Deployment Modes

### Standalone Mode
```bash
# For development/testing
connect-standalone.sh connect-standalone.properties connector.properties
```

### Distributed Mode
```bash
# For production
connect-distributed.sh connect-distributed.properties
```

## Configuration

### Worker Configuration

```properties
# connect-distributed.properties
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092
group.id=connect-cluster

# Internal topics
config.storage.topic=connect-configs
offset.storage.topic=connect-offsets
status.storage.topic=connect-status

# Replication for internal topics
config.storage.replication.factor=3
offset.storage.replication.factor=3
status.storage.replication.factor=3

# Converters
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=false
value.converter.schemas.enable=false

# REST API
rest.port=8083
rest.advertised.host.name=connect-worker-1

# Plugin path
plugin.path=/opt/kafka/plugins
```

### Avro with Schema Registry

```properties
key.converter=io.confluent.connect.avro.AvroConverter
key.converter.schema.registry.url=http://schema-registry:8081
value.converter=io.confluent.connect.avro.AvroConverter
value.converter.schema.registry.url=http://schema-registry:8081
```

## Common Connectors

### Source Connectors

#### JDBC Source (Debezium)
```json
{
  "name": "postgres-source",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "password",
    "database.dbname": "orders_db",
    "database.server.name": "orders",
    "table.include.list": "public.orders,public.order_items",
    "plugin.name": "pgoutput",
    "slot.name": "orders_slot",
    "publication.name": "orders_publication",
    "topic.prefix": "db",
    "snapshot.mode": "initial",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false"
  }
}
```

#### File Source
```json
{
  "name": "file-source",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/input.txt",
    "topic": "file-data"
  }
}
```

#### MongoDB Source
```json
{
  "name": "mongodb-source",
  "config": {
    "connector.class": "com.mongodb.kafka.connect.MongoSourceConnector",
    "connection.uri": "mongodb://mongo:27017",
    "database": "orders",
    "collection": "orders",
    "topic.prefix": "mongo",
    "pipeline": "[{\"$match\": {\"operationType\": {\"$in\": [\"insert\", \"update\", \"replace\"]}}}]",
    "copy.existing": "true",
    "output.format.value": "json"
  }
}
```

### Sink Connectors

#### JDBC Sink
```json
{
  "name": "postgres-sink",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "1",
    "topics": "orders",
    "connection.url": "jdbc:postgresql://postgres:5432/analytics",
    "connection.user": "postgres",
    "connection.password": "password",
    "auto.create": "true",
    "auto.evolve": "true",
    "insert.mode": "upsert",
    "pk.mode": "record_value",
    "pk.fields": "id",
    "table.name.format": "orders"
  }
}
```

#### Elasticsearch Sink
```json
{
  "name": "elasticsearch-sink",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "tasks.max": "1",
    "topics": "orders",
    "connection.url": "http://elasticsearch:9200",
    "type.name": "_doc",
    "key.ignore": "false",
    "schema.ignore": "true",
    "behavior.on.malformed.documents": "warn",
    "behavior.on.null.values": "delete",
    "write.method": "upsert"
  }
}
```

#### S3 Sink
```json
{
  "name": "s3-sink",
  "config": {
    "connector.class": "io.confluent.connect.s3.S3SinkConnector",
    "tasks.max": "1",
    "topics": "orders",
    "s3.bucket.name": "my-bucket",
    "s3.region": "us-east-1",
    "storage.class": "io.confluent.connect.s3.storage.S3Storage",
    "format.class": "io.confluent.connect.s3.format.parquet.ParquetFormat",
    "partitioner.class": "io.confluent.connect.storage.partitioner.TimeBasedPartitioner",
    "path.format": "'year'=YYYY/'month'=MM/'day'=dd",
    "partition.duration.ms": "3600000",
    "flush.size": "10000",
    "rotate.interval.ms": "600000",
    "timezone": "UTC"
  }
}
```

## Transforms (SMT)

### Common Transforms

```json
{
  "transforms": "InsertField,ReplaceField,MaskField,Timestamp,Filter",

  "transforms.InsertField.type": "org.apache.kafka.connect.transforms.InsertField$Value",
  "transforms.InsertField.static.field": "source",
  "transforms.InsertField.static.value": "kafka-connect",

  "transforms.ReplaceField.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
  "transforms.ReplaceField.renames": "old_name:new_name",
  "transforms.ReplaceField.exclude": "sensitive_field",

  "transforms.MaskField.type": "org.apache.kafka.connect.transforms.MaskField$Value",
  "transforms.MaskField.fields": "credit_card,ssn",

  "transforms.Timestamp.type": "org.apache.kafka.connect.transforms.TimestampConverter$Value",
  "transforms.Timestamp.field": "created_at",
  "transforms.Timestamp.target.type": "Timestamp",

  "transforms.Filter.type": "org.apache.kafka.connect.transforms.Filter",
  "transforms.Filter.predicate": "IsDeleted",
  "transforms.Filter.negate": "true",

  "predicates": "IsDeleted",
  "predicates.IsDeleted.type": "org.apache.kafka.connect.transforms.predicates.RecordIsTombstone"
}
```

### Custom Transform

```java
public class CustomTransform<R extends ConnectRecord<R>> implements Transformation<R> {
    private String fieldName;

    @Override
    public void configure(Map<String, ?> configs) {
        this.fieldName = (String) configs.get("field.name");
    }

    @Override
    public R apply(R record) {
        Struct value = (Struct) record.value();

        // Modify value
        Struct newValue = new Struct(value.schema());
        for (Field field : value.schema().fields()) {
            if (field.name().equals(fieldName)) {
                newValue.put(field, transform(value.get(field)));
            } else {
                newValue.put(field, value.get(field));
            }
        }

        return record.newRecord(
            record.topic(),
            record.kafkaPartition(),
            record.keySchema(),
            record.key(),
            newValue.schema(),
            newValue,
            record.timestamp()
        );
    }

    @Override
    public ConfigDef config() {
        return new ConfigDef()
            .define("field.name", ConfigDef.Type.STRING, ConfigDef.Importance.HIGH, "Field to transform");
    }

    @Override
    public void close() {}
}
```

## REST API

### Connector Management

```bash
# List connectors
curl http://localhost:8083/connectors

# Get connector status
curl http://localhost:8083/connectors/postgres-source/status

# Create connector
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @connector.json

# Update connector
curl -X PUT http://localhost:8083/connectors/postgres-source/config \
  -H "Content-Type: application/json" \
  -d @connector-config.json

# Delete connector
curl -X DELETE http://localhost:8083/connectors/postgres-source

# Pause connector
curl -X PUT http://localhost:8083/connectors/postgres-source/pause

# Resume connector
curl -X PUT http://localhost:8083/connectors/postgres-source/resume

# Restart connector
curl -X POST http://localhost:8083/connectors/postgres-source/restart

# Restart task
curl -X POST http://localhost:8083/connectors/postgres-source/tasks/0/restart
```

### Cluster Information

```bash
# Get cluster info
curl http://localhost:8083/

# Get worker plugins
curl http://localhost:8083/connector-plugins

# Validate connector config
curl -X PUT http://localhost:8083/connector-plugins/JdbcSourceConnector/config/validate \
  -H "Content-Type: application/json" \
  -d @connector-config.json
```

## Error Handling

### Dead Letter Queue

```json
{
  "name": "jdbc-sink",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "topics": "orders",
    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "orders-dlq",
    "errors.deadletterqueue.topic.replication.factor": "3",
    "errors.deadletterqueue.context.headers.enable": "true",
    "errors.log.enable": "true",
    "errors.log.include.messages": "true"
  }
}
```

### Retry Configuration

```json
{
  "errors.retry.delay.max.ms": "60000",
  "errors.retry.timeout": "300000"
}
```

## Monitoring

### JMX Metrics

```
kafka.connect:type=connect-worker-metrics
  - connector-count
  - task-count
  - connector-startup-attempts-total
  - connector-startup-success-total
  - connector-startup-failure-total

kafka.connect:type=connector-task-metrics,connector=*,task=*
  - offset-commit-success-percentage
  - offset-commit-skip-rate
  - batch-size-avg
  - batch-size-max

kafka.connect:type=source-task-metrics,connector=*,task=*
  - source-record-poll-rate
  - source-record-write-rate
  - poll-batch-avg-time-ms

kafka.connect:type=sink-task-metrics,connector=*,task=*
  - sink-record-read-rate
  - sink-record-send-rate
  - put-batch-avg-time-ms
```

### Health Checks

```bash
# Worker health
curl http://localhost:8083/

# Connector health
curl http://localhost:8083/connectors/my-connector/status | jq '.connector.state'

# Task health
curl http://localhost:8083/connectors/my-connector/status | jq '.tasks[].state'
```

## Production Best Practices

### Configuration Checklist

```properties
# Worker configuration
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092
group.id=connect-cluster

# Internal topics - production settings
config.storage.topic=connect-configs
config.storage.replication.factor=3
offset.storage.topic=connect-offsets
offset.storage.replication.factor=3
offset.storage.partitions=25
status.storage.topic=connect-status
status.storage.replication.factor=3
status.storage.partitions=5

# Exactly-once
exactly.once.source.support=enabled

# Security
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="user" password="password";
ssl.truststore.location=/path/to/truststore.jks
ssl.truststore.password=password

# Producer/Consumer settings for connectors
producer.security.protocol=SASL_SSL
producer.sasl.mechanism=PLAIN
consumer.security.protocol=SASL_SSL
consumer.sasl.mechanism=PLAIN
```

### Connector Best Practices

```json
{
  "name": "production-connector",
  "config": {
    "tasks.max": "4",

    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "connector-dlq",
    "errors.deadletterqueue.topic.replication.factor": "3",
    "errors.deadletterqueue.context.headers.enable": "true",
    "errors.log.enable": "true",

    "errors.retry.delay.max.ms": "60000",
    "errors.retry.timeout": "300000",

    "topic.creation.enable": "true",
    "topic.creation.default.replication.factor": "3",
    "topic.creation.default.partitions": "6"
  }
}
```

### Scaling

```bash
# Increase tasks (parallel processing)
curl -X PUT http://localhost:8083/connectors/my-connector/config \
  -H "Content-Type: application/json" \
  -d '{"tasks.max": "8", ...}'

# Add workers (horizontal scaling)
# Start additional Connect workers with same group.id
```

## Docker Deployment

```yaml
# docker-compose.yml
services:
  connect:
    image: confluentinc/cp-kafka-connect:latest
    ports:
      - "8083:8083"
    environment:
      CONNECT_BOOTSTRAP_SERVERS: broker:9092
      CONNECT_REST_PORT: 8083
      CONNECT_GROUP_ID: connect-cluster
      CONNECT_CONFIG_STORAGE_TOPIC: connect-configs
      CONNECT_OFFSET_STORAGE_TOPIC: connect-offsets
      CONNECT_STATUS_STORAGE_TOPIC: connect-status
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_PLUGIN_PATH: /usr/share/java,/usr/share/confluent-hub-components
    volumes:
      - ./plugins:/usr/share/confluent-hub-components
    command:
      - bash
      - -c
      - |
        confluent-hub install --no-prompt debezium/debezium-connector-postgresql:latest
        confluent-hub install --no-prompt confluentinc/kafka-connect-elasticsearch:latest
        /etc/confluent/docker/run
```

## References

- [Kafka Connect Documentation](https://kafka.apache.org/documentation/#connect)
- [Confluent Hub](https://www.confluent.io/hub/)
- [Debezium Documentation](https://debezium.io/documentation/)
