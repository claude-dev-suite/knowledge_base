# Apache Kafka - Production Operations

## Cluster Sizing

### Hardware Recommendations

| Component | Minimum | Recommended | High Throughput |
|-----------|---------|-------------|-----------------|
| CPU | 4 cores | 8-16 cores | 24+ cores |
| RAM | 8GB | 32GB | 64GB+ |
| Disk | SSD 500GB | NVMe 1TB | NVMe RAID 2TB+ |
| Network | 1Gbps | 10Gbps | 25Gbps+ |

### JVM Configuration

```bash
# Production JVM settings
export KAFKA_HEAP_OPTS="-Xmx6g -Xms6g"
export KAFKA_JVM_PERFORMANCE_OPTS="
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=20
  -XX:InitiatingHeapOccupancyPercent=35
  -XX:+ExplicitGCInvokesConcurrent
  -XX:+ParallelRefProcEnabled
  -XX:+UseStringDeduplication
  -Djava.awt.headless=true
"

# For large heaps (>32GB)
export KAFKA_JVM_PERFORMANCE_OPTS="
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=20
  -XX:+UnlockExperimentalVMOptions
  -XX:+UseZGC
  -XX:+ZGenerational
"
```

### Partition Sizing

```
Partitions = max(T/P, T/C)

Where:
T = Target throughput (MB/s)
P = Producer throughput per partition (~10MB/s)
C = Consumer throughput per partition (~25MB/s)

Example: 200MB/s target
Partitions = max(200/10, 200/25) = max(20, 8) = 20 partitions

Rule of thumb:
- Start with 2× expected consumer count
- Max 4,000 partitions per broker (KRaft)
- Max 200,000 partitions per cluster (KRaft)
```

## Monitoring

### Key Metrics

#### Broker Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `UnderReplicatedPartitions` | Partitions with fewer ISR | > 0 |
| `OfflinePartitionsCount` | Partitions without leader | > 0 |
| `ActiveControllerCount` | Active controllers | != 1 |
| `RequestHandlerAvgIdlePercent` | Handler thread idle | < 0.3 |
| `NetworkProcessorAvgIdlePercent` | Network thread idle | < 0.3 |
| `RequestQueueSize` | Pending requests | > 100 |
| `BytesInPerSec` | Ingress rate | Capacity planning |
| `BytesOutPerSec` | Egress rate | Capacity planning |
| `MessagesInPerSec` | Message rate | Capacity planning |

#### Producer Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `record-send-rate` | Records sent/sec | Anomaly detection |
| `record-error-rate` | Failed sends/sec | > 0 |
| `request-latency-avg` | Avg request latency | > 100ms |
| `request-latency-p99` | P99 request latency | > 500ms |
| `batch-size-avg` | Avg batch size | Optimization |
| `buffer-available-bytes` | Available buffer | < 10% total |

#### Consumer Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `records-consumed-rate` | Records consumed/sec | Anomaly detection |
| `records-lag-max` | Max lag across partitions | > 10,000 |
| `records-lag` | Per-partition lag | > 10,000 |
| `fetch-latency-avg` | Avg fetch latency | > 500ms |
| `commit-latency-avg` | Avg commit latency | > 100ms |
| `rebalance-latency-avg` | Avg rebalance time | > 30s |

### Prometheus + Grafana Setup

```yaml
# docker-compose.yml
services:
  kafka-exporter:
    image: danielqsj/kafka-exporter
    command:
      - --kafka.server=broker:9092
      - --topic.filter=.*
      - --group.filter=.*
    ports:
      - "9308:9308"

  jmx-exporter:
    image: bitnami/jmx-exporter
    volumes:
      - ./jmx-config.yml:/etc/jmx-exporter/config.yml
    ports:
      - "5556:5556"

  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka-exporter:9308']

  - job_name: 'kafka-jmx'
    static_configs:
      - targets: ['jmx-exporter:5556']
```

### Alerting Rules

```yaml
# prometheus-alerts.yml
groups:
  - name: kafka
    rules:
      - alert: KafkaUnderReplicatedPartitions
        expr: kafka_server_replicamanager_underreplicatedpartitions > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Kafka has under-replicated partitions"

      - alert: KafkaOfflinePartitions
        expr: kafka_controller_kafkacontroller_offlinepartitionscount > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Kafka has offline partitions"

      - alert: KafkaConsumerLag
        expr: kafka_consumergroup_lag > 10000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Consumer group {{ $labels.consumergroup }} has high lag"

      - alert: KafkaBrokerDown
        expr: kafka_brokers < 3
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Kafka broker count is below expected"
```

## Operations

### Topic Management

```bash
# List topics
kafka-topics.sh --list --bootstrap-server localhost:9092

# Describe topic
kafka-topics.sh --describe --topic orders --bootstrap-server localhost:9092

# Create topic
kafka-topics.sh --create \
  --topic orders \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --bootstrap-server localhost:9092

# Alter partitions (increase only)
kafka-topics.sh --alter \
  --topic orders \
  --partitions 24 \
  --bootstrap-server localhost:9092

# Delete topic
kafka-topics.sh --delete --topic orders --bootstrap-server localhost:9092

# Alter topic config
kafka-configs.sh --alter \
  --topic orders \
  --add-config retention.ms=86400000 \
  --bootstrap-server localhost:9092
```

### Consumer Group Management

```bash
# List consumer groups
kafka-consumer-groups.sh --list --bootstrap-server localhost:9092

# Describe group
kafka-consumer-groups.sh --describe \
  --group order-processor \
  --bootstrap-server localhost:9092

# Reset offsets (stop consumers first!)
kafka-consumer-groups.sh --reset-offsets \
  --group order-processor \
  --topic orders \
  --to-earliest \
  --execute \
  --bootstrap-server localhost:9092

# Reset to specific offset
kafka-consumer-groups.sh --reset-offsets \
  --group order-processor \
  --topic orders:0 \
  --to-offset 1000 \
  --execute \
  --bootstrap-server localhost:9092

# Reset by timestamp
kafka-consumer-groups.sh --reset-offsets \
  --group order-processor \
  --topic orders \
  --to-datetime "2024-01-01T00:00:00.000" \
  --execute \
  --bootstrap-server localhost:9092

# Delete group
kafka-consumer-groups.sh --delete \
  --group order-processor \
  --bootstrap-server localhost:9092
```

### Partition Reassignment

```bash
# Generate reassignment plan
kafka-reassign-partitions.sh --generate \
  --topics-to-move-json-file topics.json \
  --broker-list "0,1,2,3" \
  --bootstrap-server localhost:9092

# Execute reassignment
kafka-reassign-partitions.sh --execute \
  --reassignment-json-file reassignment.json \
  --bootstrap-server localhost:9092

# Verify reassignment
kafka-reassign-partitions.sh --verify \
  --reassignment-json-file reassignment.json \
  --bootstrap-server localhost:9092

# Cancel reassignment
kafka-reassign-partitions.sh --cancel \
  --reassignment-json-file reassignment.json \
  --bootstrap-server localhost:9092
```

```json
// topics.json
{
  "topics": [
    {"topic": "orders"}
  ],
  "version": 1
}

// reassignment.json
{
  "version": 1,
  "partitions": [
    {"topic": "orders", "partition": 0, "replicas": [1, 2, 3]},
    {"topic": "orders", "partition": 1, "replicas": [2, 3, 1]},
    {"topic": "orders", "partition": 2, "replicas": [3, 1, 2]}
  ]
}
```

### Preferred Leader Election

```bash
# Trigger preferred leader election
kafka-leader-election.sh --election-type preferred \
  --topic orders \
  --partition 0 \
  --bootstrap-server localhost:9092

# All partitions
kafka-leader-election.sh --election-type preferred \
  --all-topic-partitions \
  --bootstrap-server localhost:9092
```

## Troubleshooting

### Common Issues

#### Under-Replicated Partitions

```bash
# Check ISR
kafka-topics.sh --describe --topic orders --bootstrap-server localhost:9092

# Possible causes:
# 1. Broker down
# 2. Network issues
# 3. Disk I/O slow
# 4. Broker overloaded

# Check broker logs
grep -i "isr" /var/log/kafka/server.log

# Force ISR shrink (last resort)
kafka-configs.sh --alter \
  --topic orders \
  --add-config min.insync.replicas=1 \
  --bootstrap-server localhost:9092
```

#### Consumer Lag

```bash
# Check lag
kafka-consumer-groups.sh --describe \
  --group order-processor \
  --bootstrap-server localhost:9092

# Possible causes:
# 1. Slow processing
# 2. Too few consumers
# 3. Partition imbalance
# 4. GC pauses

# Solutions:
# 1. Increase max.poll.records
# 2. Add more consumers (up to partition count)
# 3. Optimize processing logic
# 4. Check consumer JVM
```

#### Disk Full

```bash
# Check disk usage
df -h /var/kafka/data

# Emergency: delete old segments
kafka-delete-records.sh --offset-json-file delete.json \
  --bootstrap-server localhost:9092

# delete.json
{
  "partitions": [
    {"topic": "orders", "partition": 0, "offset": 1000000}
  ],
  "version": 1
}

# Reduce retention temporarily
kafka-configs.sh --alter \
  --topic orders \
  --add-config retention.ms=3600000 \
  --bootstrap-server localhost:9092
```

### Log Analysis

```bash
# Check for errors
grep -i "error\|exception\|warn" /var/log/kafka/server.log | tail -100

# Check controller logs
grep -i "controller" /var/log/kafka/controller.log | tail -100

# Check GC logs
grep -i "gc pause" /var/log/kafka/kafka-gc.log

# Request timing
grep "RequestSendTime" /var/log/kafka/server.log | \
  awk '{print $NF}' | sort -n | tail -20
```

## Backup & Recovery

### Topic Backup

```bash
# Using MirrorMaker 2
connect-mirror-maker.sh mm2.properties

# mm2.properties
clusters = source, backup
source.bootstrap.servers = source-broker:9092
backup.bootstrap.servers = backup-broker:9092

source->backup.enabled = true
source->backup.topics = orders, payments

replication.factor = 3
checkpoints.topic.replication.factor = 3
heartbeats.topic.replication.factor = 3
offset-syncs.topic.replication.factor = 3
```

### Consumer Offset Backup

```bash
# Export offsets
kafka-consumer-groups.sh --describe \
  --group order-processor \
  --bootstrap-server localhost:9092 \
  > offsets-backup.txt

# Import offsets (after topic restore)
kafka-consumer-groups.sh --reset-offsets \
  --group order-processor \
  --topic orders \
  --from-file offsets-backup.txt \
  --execute \
  --bootstrap-server localhost:9092
```

### Disaster Recovery

```bash
# 1. Stop producers
# 2. Wait for consumers to catch up
# 3. Stop consumers
# 4. Backup topic data (MirrorMaker or file copy)
# 5. Restore to new cluster
# 6. Update client configurations
# 7. Start consumers
# 8. Start producers
```

## Rolling Upgrade

```bash
# 1. Update broker config
inter.broker.protocol.version=3.6
log.message.format.version=3.6

# 2. Upgrade brokers one at a time
for broker in broker1 broker2 broker3; do
  # Stop broker
  kafka-server-stop.sh

  # Upgrade binaries
  # ...

  # Start broker
  kafka-server-start.sh -daemon server.properties

  # Wait for broker to rejoin
  while ! kafka-metadata.sh --snapshot /var/kafka/data/__cluster_metadata-0/00000000000000000000.log --cluster-id $CLUSTER_ID; do
    sleep 5
  done

  # Wait for ISR to recover
  while kafka-topics.sh --describe --bootstrap-server localhost:9092 | grep -q "Isr:.*$broker"; do
    sleep 5
  done
done

# 3. Remove protocol version configs
kafka-configs.sh --alter \
  --entity-type brokers \
  --entity-default \
  --delete-config inter.broker.protocol.version,log.message.format.version \
  --bootstrap-server localhost:9092
```

## Security Operations

### Certificate Rotation

```bash
# 1. Generate new certificates
# 2. Add new cert to truststore
keytool -importcert -alias kafka-new -file kafka-new.crt -keystore truststore.jks

# 3. Update keystore with new cert
keytool -importkeystore -srckeystore kafka-new.p12 -destkeystore keystore.jks

# 4. Rolling restart brokers
# 5. Remove old cert from truststore
keytool -delete -alias kafka-old -keystore truststore.jks
```

### ACL Audit

```bash
# List all ACLs
kafka-acls.sh --list --bootstrap-server localhost:9092

# Export ACLs
kafka-acls.sh --list --bootstrap-server localhost:9092 > acls-backup.txt

# Find unused permissions
# Compare with actual client usage from logs
```

## References

- [Kafka Operations](https://kafka.apache.org/documentation/#operations)
- [Monitoring Kafka](https://docs.confluent.io/platform/current/kafka/monitoring.html)
- [Kafka Security](https://kafka.apache.org/documentation/#security)
