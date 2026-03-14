# Apache Kafka - Configuration Reference

## Broker Configuration

### Essential Settings

```properties
# broker.properties

# Broker identity
broker.id=0
# Or for KRaft mode:
node.id=0
process.roles=broker,controller

# Listeners
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
advertised.listeners=PLAINTEXT://broker1.example.com:9092
listener.security.protocol.map=PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT

# KRaft controller
controller.quorum.voters=0@broker1:9093,1@broker2:9093,2@broker3:9093
controller.listener.names=CONTROLLER

# Log directories
log.dirs=/var/kafka/data

# ZooKeeper (legacy, not needed for KRaft)
# zookeeper.connect=zk1:2181,zk2:2181,zk3:2181
```

### Replication Settings

```properties
# Default replication factor for auto-created topics
default.replication.factor=3

# Minimum ISR for producer acks=all
min.insync.replicas=2

# Unclean leader election (data loss risk)
unclean.leader.election.enable=false

# Replica lag threshold
replica.lag.time.max.ms=30000

# Replica fetch settings
replica.fetch.max.bytes=1048576
replica.fetch.wait.max.ms=500
```

### Log Settings

```properties
# Segment settings
log.segment.bytes=1073741824          # 1GB
log.roll.hours=168                     # 7 days

# Retention
log.retention.hours=168                # 7 days
log.retention.bytes=-1                 # Unlimited
log.retention.check.interval.ms=300000 # 5 minutes

# Cleanup policy
log.cleanup.policy=delete              # delete or compact

# Compaction settings (for compact topics)
log.cleaner.enable=true
log.cleaner.min.cleanable.ratio=0.5
log.cleaner.threads=1
```

### Network Settings

```properties
# Network threads
num.network.threads=3
num.io.threads=8

# Socket settings
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600     # 100MB

# Request handling
queued.max.requests=500
request.timeout.ms=30000
```

### Performance Tuning

```properties
# Message batch settings
message.max.bytes=1048576              # 1MB
replica.fetch.max.bytes=1048576

# Compression
compression.type=producer              # Use producer's compression

# Memory
log.flush.interval.messages=10000
log.flush.interval.ms=1000

# Background threads
background.threads=10
num.recovery.threads.per.data.dir=1
```

## Producer Configuration

### Reliability Settings

```properties
# Acknowledgments
acks=all                               # -1, 0, 1, all

# Idempotence (exactly-once)
enable.idempotence=true

# Retries
retries=2147483647                     # MAX_INT
retry.backoff.ms=100
delivery.timeout.ms=120000             # 2 minutes

# In-flight requests (ordering with idempotence)
max.in.flight.requests.per.connection=5

# Transactions
transactional.id=my-transactional-id
transaction.timeout.ms=60000
```

### Performance Settings

```properties
# Batching
batch.size=16384                       # 16KB
linger.ms=0                            # or 5-100 for throughput

# Buffer
buffer.memory=33554432                 # 32MB
max.block.ms=60000

# Compression
compression.type=lz4                   # none, gzip, snappy, lz4, zstd

# Request size
max.request.size=1048576               # 1MB
```

### Connection Settings

```properties
# Bootstrap
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092

# Client ID
client.id=my-producer

# Timeouts
request.timeout.ms=30000
metadata.max.age.ms=300000             # 5 minutes
connections.max.idle.ms=540000         # 9 minutes
```

## Consumer Configuration

### Consumer Group Settings

```properties
# Group
group.id=my-consumer-group
group.instance.id=consumer-1           # Static membership

# Session management
session.timeout.ms=45000
heartbeat.interval.ms=3000
max.poll.interval.ms=300000            # 5 minutes

# Partition assignment
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

### Offset Settings

```properties
# Auto commit
enable.auto.commit=false               # Manual commit for at-least-once
auto.commit.interval.ms=5000

# Offset reset
auto.offset.reset=earliest             # earliest, latest, none

# Isolation level (for transactions)
isolation.level=read_committed         # read_uncommitted, read_committed
```

### Fetch Settings

```properties
# Fetch size
fetch.min.bytes=1
fetch.max.bytes=52428800               # 50MB
max.partition.fetch.bytes=1048576      # 1MB

# Poll settings
max.poll.records=500

# Fetch wait
fetch.max.wait.ms=500
```

### Connection Settings

```properties
# Bootstrap
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092

# Client ID
client.id=my-consumer

# Timeouts
request.timeout.ms=30000
default.api.timeout.ms=60000
```

## Topic Configuration

### Create Topic

```bash
kafka-topics.sh --create \
  --topic orders \
  --bootstrap-server localhost:9092 \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config min.insync.replicas=2 \
  --config cleanup.policy=delete
```

### Topic-Level Configs

| Config | Default | Description |
|--------|---------|-------------|
| `cleanup.policy` | delete | delete, compact, delete,compact |
| `compression.type` | producer | producer, gzip, snappy, lz4, zstd |
| `retention.ms` | 604800000 | 7 days |
| `retention.bytes` | -1 | No limit |
| `segment.bytes` | 1073741824 | 1GB |
| `segment.ms` | 604800000 | 7 days |
| `min.insync.replicas` | 1 | Min ISR for acks=all |
| `max.message.bytes` | 1048588 | Max message size |
| `min.cleanable.dirty.ratio` | 0.5 | Compaction threshold |
| `delete.retention.ms` | 86400000 | Tombstone retention |

### Compacted Topic

```bash
kafka-topics.sh --create \
  --topic user-profiles \
  --bootstrap-server localhost:9092 \
  --partitions 6 \
  --replication-factor 3 \
  --config cleanup.policy=compact \
  --config min.cleanable.dirty.ratio=0.3 \
  --config delete.retention.ms=86400000 \
  --config segment.ms=604800000
```

## Security Configuration

### SSL/TLS

```properties
# Broker
listeners=SSL://:9093
ssl.keystore.location=/path/to/kafka.server.keystore.jks
ssl.keystore.password=password
ssl.key.password=password
ssl.truststore.location=/path/to/kafka.server.truststore.jks
ssl.truststore.password=password
ssl.client.auth=required

# Client
security.protocol=SSL
ssl.truststore.location=/path/to/client.truststore.jks
ssl.truststore.password=password
ssl.keystore.location=/path/to/client.keystore.jks
ssl.keystore.password=password
ssl.key.password=password
```

### SASL/PLAIN

```properties
# Broker
listeners=SASL_SSL://:9094
sasl.enabled.mechanisms=PLAIN
sasl.mechanism.inter.broker.protocol=PLAIN

# JAAS config (broker)
listener.name.sasl_ssl.plain.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="admin" \
  password="admin-secret" \
  user_admin="admin-secret" \
  user_client="client-secret";

# Client
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="client" \
  password="client-secret";
```

### SASL/SCRAM

```bash
# Create SCRAM credentials
kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter \
  --add-config 'SCRAM-SHA-256=[password=password]' \
  --entity-type users \
  --entity-name admin
```

```properties
# Broker
sasl.enabled.mechanisms=SCRAM-SHA-256
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256

# Client
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="admin" \
  password="password";
```

### ACLs

```bash
# Allow producer
kafka-acls.sh --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:producer \
  --operation Write \
  --topic orders

# Allow consumer
kafka-acls.sh --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:consumer \
  --operation Read \
  --topic orders \
  --group order-processors

# List ACLs
kafka-acls.sh --bootstrap-server localhost:9092 --list
```

## Monitoring Configuration

### JMX

```properties
# Enable JMX
KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote \
  -Dcom.sun.management.jmxremote.authenticate=false \
  -Dcom.sun.management.jmxremote.ssl=false \
  -Dcom.sun.management.jmxremote.port=9999"
```

### Metrics Reporters

```properties
# Enable metrics reporters
metric.reporters=org.apache.kafka.common.metrics.JmxReporter

# Kafka Exporter (for Prometheus)
# Run separately or use JMX Exporter
```

## Production Checklist

### Broker Checklist

```properties
# Reliability
min.insync.replicas=2
unclean.leader.election.enable=false
default.replication.factor=3
auto.create.topics.enable=false

# Performance
num.network.threads=8
num.io.threads=16
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
log.flush.interval.messages=10000

# Retention
log.retention.hours=168
log.segment.bytes=1073741824

# Security
ssl.client.auth=required
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
```

### Producer Checklist

```properties
acks=all
enable.idempotence=true
retries=3
max.in.flight.requests.per.connection=5
compression.type=lz4
batch.size=32768
linger.ms=5
```

### Consumer Checklist

```properties
enable.auto.commit=false
auto.offset.reset=earliest
isolation.level=read_committed
max.poll.records=100
partition.assignment.strategy=CooperativeStickyAssignor
```

## Environment Variables

```bash
# Broker
KAFKA_BROKER_ID=0
KAFKA_LISTENERS=PLAINTEXT://:9092
KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://broker:9092
KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
KAFKA_LOG_DIRS=/var/kafka/data
KAFKA_NUM_PARTITIONS=12
KAFKA_DEFAULT_REPLICATION_FACTOR=3
KAFKA_MIN_INSYNC_REPLICAS=2
KAFKA_LOG_RETENTION_HOURS=168
KAFKA_AUTO_CREATE_TOPICS_ENABLE=false

# JVM
KAFKA_HEAP_OPTS="-Xmx6g -Xms6g"
KAFKA_JVM_PERFORMANCE_OPTS="-XX:+UseG1GC -XX:MaxGCPauseMillis=20"
```

## References

- [Broker Configs](https://kafka.apache.org/documentation/#brokerconfigs)
- [Producer Configs](https://kafka.apache.org/documentation/#producerconfigs)
- [Consumer Configs](https://kafka.apache.org/documentation/#consumerconfigs)
- [Topic Configs](https://kafka.apache.org/documentation/#topicconfigs)
