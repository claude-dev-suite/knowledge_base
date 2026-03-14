# Apache Kafka - Basics & Architecture

## Overview

Apache Kafka is a distributed event streaming platform capable of handling trillions of events per day. Originally developed at LinkedIn and open-sourced in 2011, it's now maintained by the Apache Software Foundation.

## Core Architecture

### Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kafka Cluster                             │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Broker 1   │  │   Broker 2   │  │   Broker 3   │          │
│  │              │  │              │  │              │          │
│  │  Topic A     │  │  Topic A     │  │  Topic A     │          │
│  │  Partition 0 │  │  Partition 1 │  │  Partition 2 │          │
│  │  (Leader)    │  │  (Leader)    │  │  (Leader)    │          │
│  │              │  │              │  │              │          │
│  │  Topic A     │  │  Topic A     │  │  Topic A     │          │
│  │  Partition 1 │  │  Partition 2 │  │  Partition 0 │          │
│  │  (Follower)  │  │  (Follower)  │  │  (Follower)  │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   KRaft Controller                       │   │
│  │  (Metadata management - replaces ZooKeeper)              │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Broker
- Kafka server instance
- Handles read/write requests
- Manages partition replicas
- Stores messages on disk
- Identified by unique broker ID

### Topic
- Named stream of records
- Append-only, immutable log
- Can have multiple partitions
- Identified by name (e.g., `orders`, `user-events`)

### Partition
- Ordered, immutable sequence of records
- Unit of parallelism
- Has one leader and zero or more followers
- Records identified by offset within partition

### Record (Message)
```
┌────────────────────────────────────────────────┐
│                    Record                       │
├────────────┬───────────────────────────────────┤
│ Key        │ Optional, used for partitioning   │
├────────────┼───────────────────────────────────┤
│ Value      │ Message payload (bytes)           │
├────────────┼───────────────────────────────────┤
│ Timestamp  │ Event time or ingestion time      │
├────────────┼───────────────────────────────────┤
│ Headers    │ Key-value metadata pairs          │
├────────────┼───────────────────────────────────┤
│ Offset     │ Position in partition (assigned)  │
└────────────┴───────────────────────────────────┘
```

## Data Flow

### Producer Flow
```
Producer ──▶ Serializer ──▶ Partitioner ──▶ Record Accumulator ──▶ Sender ──▶ Broker
                              │
                              ▼
                    ┌─────────────────┐
                    │ Key.hashCode()  │
                    │ % numPartitions │
                    │ = partition     │
                    └─────────────────┘
```

### Consumer Flow
```
Consumer Group
    │
    ├── Consumer 1 ◀── Partition 0
    ├── Consumer 2 ◀── Partition 1
    └── Consumer 3 ◀── Partition 2

Offset Management:
Consumer ──▶ Poll() ──▶ Process ──▶ Commit Offset ──▶ __consumer_offsets topic
```

## Partitioning Strategies

### Default Partitioner
```java
// With key: hash-based
partition = Math.abs(key.hashCode()) % numPartitions

// Without key: round-robin (sticky in newer versions)
partition = round_robin_counter++ % numPartitions
```

### Custom Partitioner
```java
public class GeoPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes, Cluster cluster) {
        // Route to partition based on geographic region
        String region = extractRegion(key);
        return regionToPartition.get(region);
    }
}
```

## Replication

### ISR (In-Sync Replicas)
- Set of replicas fully caught up with leader
- Only ISR members eligible to become leader
- Controlled by `replica.lag.time.max.ms`

### Leader Election
```
Broker 1 (Leader P0) fails
          │
          ▼
Controller detects failure
          │
          ▼
Controller selects new leader from ISR
          │
          ▼
Broker 2 becomes new Leader P0
```

### Acknowledgment Levels
```
acks=0    → Producer doesn't wait for ack (fire-and-forget)
acks=1    → Leader writes, then acks (leader failure = data loss)
acks=all  → All ISR replicas write, then ack (strongest guarantee)
```

## Offset Management

### Offset Types
```
Partition: [msg0][msg1][msg2][msg3][msg4][msg5][msg6]
                                    ▲           ▲
                            committed offset    log end offset
                            (last processed)    (latest available)
```

### Commit Strategies
| Strategy | Description | Trade-off |
|----------|-------------|-----------|
| Auto-commit | Periodic background commit | May lose messages on failure |
| Sync commit | Block until offset committed | Slower, exactly-once possible |
| Async commit | Non-blocking commit | Fast, may lose offset on failure |

## Consumer Groups

### Partition Assignment
```
Consumer Group: order-processors (3 consumers, 6 partitions)

Consumer 1: [P0, P1]
Consumer 2: [P2, P3]
Consumer 3: [P4, P5]

If Consumer 3 fails:
Consumer 1: [P0, P1, P4]
Consumer 2: [P2, P3, P5]
```

### Assignment Strategies
- **Range**: Contiguous partitions per consumer
- **RoundRobin**: Even distribution across all subscribed topics
- **Sticky**: Minimize partition movement on rebalance
- **CooperativeSticky**: Incremental rebalance (no stop-the-world)

## Log Compaction

For topics with `cleanup.policy=compact`:

```
Before compaction:
[K1:V1][K2:V2][K1:V3][K3:V4][K2:V5][K1:V6]

After compaction:
[K3:V4][K2:V5][K1:V6]  (keeps latest value per key)
```

Use cases:
- Database changelog
- Configuration updates
- Latest state per entity

## KRaft Mode (ZooKeeper Removal)

As of Kafka 3.3+, KRaft is production-ready:

```
Traditional:              KRaft Mode:
┌─────────────┐          ┌─────────────┐
│  ZooKeeper  │          │  KRaft      │
│  Ensemble   │          │  Controller │
└─────────────┘          │  (in Kafka) │
       │                 └─────────────┘
       ▼                        │
┌─────────────┐          ┌──────▼──────┐
│   Kafka     │          │   Kafka     │
│   Brokers   │          │   Brokers   │
└─────────────┘          └─────────────┘
```

Benefits:
- Simpler architecture (no external dependency)
- Faster controller failover
- More partitions per cluster
- Easier operations

## Message Delivery Semantics

| Semantic | Producer | Consumer | Description |
|----------|----------|----------|-------------|
| At-most-once | acks=0 or 1 | Auto-commit | May lose messages |
| At-least-once | acks=all | Manual commit | May have duplicates |
| Exactly-once | Idempotent + transactional | Transactional | No loss, no duplicates |

## Key Metrics to Monitor

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `UnderReplicatedPartitions` | Partitions with fewer ISR than RF | > 0 |
| `OfflinePartitionsCount` | Partitions without leader | > 0 |
| `ActiveControllerCount` | Controllers in cluster | != 1 |
| `RequestQueueSize` | Pending requests | > 100 |
| `BytesInPerSec` | Ingress throughput | Capacity planning |
| `BytesOutPerSec` | Egress throughput | Capacity planning |
| `ConsumerLag` | Messages behind | Depends on SLA |

## Best Practices

### Topic Design
- Use meaningful, namespaced names: `domain.entity.event`
- Partition count = max expected consumers × 2
- Consider data locality for partitioning
- Enable log compaction for state topics

### Producer
- Use `acks=all` for durability
- Enable idempotence for exactly-once
- Configure appropriate `batch.size` and `linger.ms`
- Implement retry with backoff

### Consumer
- Size consumer group to partition count
- Use `max.poll.records` to control batch size
- Implement idempotent processing
- Handle rebalances gracefully

## References

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [KRaft: Apache Kafka Without ZooKeeper](https://developer.confluent.io/learn/kraft/)
- [Kafka Design](https://kafka.apache.org/documentation/#design)
