# Apache Kafka - Consumer API

## Consumer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                       Kafka Consumer                             │
│                                                                  │
│  ┌────────────────┐   ┌─────────────────┐   ┌────────────────┐ │
│  │  Coordinator   │   │    Fetcher      │   │  Deserializer  │ │
│  │  (Group mgmt)  │   │ (Data fetching) │   │                │ │
│  └────────────────┘   └─────────────────┘   └────────────────┘ │
│          │                    │                     │           │
│          ▼                    ▼                     ▼           │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Consumer Records                         │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                   │
│                              ▼                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                 Offset Committer                            │ │
│  │  (Tracks position in __consumer_offsets topic)              │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Configuration Reference

### Essential Configurations

| Config | Default | Description | Production Value |
|--------|---------|-------------|------------------|
| `bootstrap.servers` | - | Broker addresses | Multiple brokers |
| `group.id` | - | Consumer group name | Meaningful name |
| `key.deserializer` | - | Key deserialization | StringDeserializer |
| `value.deserializer` | - | Value deserialization | StringDeserializer |
| `auto.offset.reset` | latest | Behavior when no offset | `earliest` or `latest` |

### Consumer Group Configurations

| Config | Default | Description | Production Value |
|--------|---------|-------------|------------------|
| `session.timeout.ms` | 45000 | Heartbeat timeout | 30000-45000 |
| `heartbeat.interval.ms` | 3000 | Heartbeat frequency | 10000 |
| `max.poll.interval.ms` | 300000 | Max poll interval | Based on processing time |
| `max.poll.records` | 500 | Max records per poll | 100-500 |

### Offset Configurations

| Config | Default | Description | Production Value |
|--------|---------|-------------|------------------|
| `enable.auto.commit` | true | Auto commit offsets | `false` for at-least-once |
| `auto.commit.interval.ms` | 5000 | Auto commit interval | 5000 |
| `auto.offset.reset` | latest | When no offset exists | `earliest` |

### Fetch Configurations

| Config | Default | Description | Tuning |
|--------|---------|-------------|--------|
| `fetch.min.bytes` | 1 | Min bytes per fetch | 1-1024 |
| `fetch.max.wait.ms` | 500 | Max wait for min bytes | 100-500 |
| `fetch.max.bytes` | 52428800 | Max fetch size | 50MB |
| `max.partition.fetch.bytes` | 1048576 | Max per partition | 1MB |

## Consumer Group Mechanics

### Partition Assignment

```
Consumer Group: order-processors
Topic: orders (6 partitions)

Initial State (3 consumers):
┌────────────┬────────────┬────────────┐
│ Consumer 1 │ Consumer 2 │ Consumer 3 │
├────────────┼────────────┼────────────┤
│    P0      │    P2      │    P4      │
│    P1      │    P3      │    P5      │
└────────────┴────────────┴────────────┘

After Consumer 3 fails:
┌────────────┬────────────┐
│ Consumer 1 │ Consumer 2 │
├────────────┼────────────┤
│    P0      │    P3      │
│    P1      │    P4      │
│    P2      │    P5      │
└────────────┴────────────┘
```

### Assignment Strategies

```java
// Range Assignor (default)
props.put("partition.assignment.strategy",
    "org.apache.kafka.clients.consumer.RangeAssignor");

// Round Robin Assignor
props.put("partition.assignment.strategy",
    "org.apache.kafka.clients.consumer.RoundRobinAssignor");

// Sticky Assignor (minimizes movement)
props.put("partition.assignment.strategy",
    "org.apache.kafka.clients.consumer.StickyAssignor");

// Cooperative Sticky (incremental rebalance)
props.put("partition.assignment.strategy",
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
```

### Rebalancing

```
Rebalance Triggers:
1. Consumer joins group
2. Consumer leaves group (failure or shutdown)
3. Consumer fails to send heartbeat
4. Subscription changes
5. Partition count changes

Cooperative Rebalancing (Incremental):
┌─────────────────────────────────────────────────────────┐
│ Step 1: Identify partitions to revoke                   │
│ Step 2: Revoke only those partitions                    │
│ Step 3: Assign revoked partitions to new consumer       │
│ (Other partitions continue processing)                  │
└─────────────────────────────────────────────────────────┘

Stop-the-World Rebalancing (Eager):
┌─────────────────────────────────────────────────────────┐
│ Step 1: Revoke ALL partitions from ALL consumers        │
│ Step 2: Reassign ALL partitions                         │
│ (Complete processing halt during rebalance)             │
└─────────────────────────────────────────────────────────┘
```

## Offset Management

### Offset Types
```
Log:     [0][1][2][3][4][5][6][7][8][9]
                      ▲           ▲
              committed        log end
              offset           offset

Current Position: The next offset to be fetched
Committed Offset: Last processed offset (persisted)
Log End Offset: Latest message in partition
Consumer Lag: Log End Offset - Committed Offset
```

### Auto Commit
```java
// Configuration
props.put("enable.auto.commit", true);
props.put("auto.commit.interval.ms", 5000);

// Risk: Messages may be committed before processing completes
// If consumer crashes, messages between last commit and crash are lost
```

### Manual Commit (Synchronous)
```java
props.put("enable.auto.commit", false);

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, String> record : records) {
        processRecord(record);
    }

    // Block until offset committed
    consumer.commitSync();
}
```

### Manual Commit (Asynchronous)
```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, String> record : records) {
        processRecord(record);
    }

    // Non-blocking commit
    consumer.commitAsync((offsets, exception) -> {
        if (exception != null) {
            logger.error("Commit failed", exception);
        }
    });
}
```

### Commit Specific Offsets
```java
Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();

for (ConsumerRecord<String, String> record : records) {
    processRecord(record);

    // Commit after each record (fine-grained)
    offsets.put(
        new TopicPartition(record.topic(), record.partition()),
        new OffsetAndMetadata(record.offset() + 1)
    );
}

consumer.commitSync(offsets);
```

### Seek Operations
```java
// Seek to beginning
consumer.seekToBeginning(consumer.assignment());

// Seek to end
consumer.seekToEnd(consumer.assignment());

// Seek to specific offset
consumer.seek(new TopicPartition("orders", 0), 100);

// Seek by timestamp
Map<TopicPartition, Long> timestamps = new HashMap<>();
timestamps.put(new TopicPartition("orders", 0), System.currentTimeMillis() - 86400000);
Map<TopicPartition, OffsetAndTimestamp> offsets = consumer.offsetsForTimes(timestamps);

for (Map.Entry<TopicPartition, OffsetAndTimestamp> entry : offsets.entrySet()) {
    consumer.seek(entry.getKey(), entry.getValue().offset());
}
```

## Consumer Patterns

### Basic Consumption Loop
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "order-processor");
props.put("key.deserializer", StringDeserializer.class.getName());
props.put("value.deserializer", StringDeserializer.class.getName());
props.put("enable.auto.commit", false);
props.put("auto.offset.reset", "earliest");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("orders"));

try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

        for (ConsumerRecord<String, String> record : records) {
            System.out.printf("Received: key=%s, value=%s, partition=%d, offset=%d%n",
                record.key(), record.value(), record.partition(), record.offset());

            processRecord(record);
        }

        consumer.commitSync();
    }
} finally {
    consumer.close();
}
```

### At-Least-Once Processing
```java
props.put("enable.auto.commit", false);

while (running) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, String> record : records) {
        try {
            // Process first
            processRecord(record);

            // Then commit (if crash before commit, message reprocessed)
            consumer.commitSync(Map.of(
                new TopicPartition(record.topic(), record.partition()),
                new OffsetAndMetadata(record.offset() + 1)
            ));
        } catch (Exception e) {
            // Handle error - message will be reprocessed
            logger.error("Processing failed", e);
        }
    }
}
```

### Idempotent Consumer
```java
public class IdempotentOrderProcessor {
    private final Set<String> processedIds = ConcurrentHashMap.newKeySet();

    public void process(ConsumerRecord<String, Order> record) {
        String messageId = record.headers()
            .lastHeader("message-id")
            .value()
            .toString();

        // Skip if already processed
        if (!processedIds.add(messageId)) {
            return;
        }

        // Process the order
        orderService.processOrder(record.value());
    }
}
```

### Multi-Threaded Consumer
```java
public class MultiThreadedConsumer {
    private final KafkaConsumer<String, String> consumer;
    private final ExecutorService executor;

    public void consume() {
        while (running) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

            List<Future<?>> futures = new ArrayList<>();
            for (ConsumerRecord<String, String> record : records) {
                futures.add(executor.submit(() -> processRecord(record)));
            }

            // Wait for all processing to complete
            for (Future<?> future : futures) {
                future.get();
            }

            consumer.commitSync();
        }
    }
}
```

### Pause/Resume
```java
// Pause consumption (for backpressure)
consumer.pause(consumer.assignment());

// Resume consumption
consumer.resume(consumer.assignment());

// Check paused partitions
Set<TopicPartition> paused = consumer.paused();
```

## Rebalance Listener

```java
consumer.subscribe(Arrays.asList("orders"), new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // Called before rebalance
        // Commit offsets for revoked partitions
        consumer.commitSync();
        logger.info("Partitions revoked: {}", partitions);
    }

    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // Called after rebalance
        // Initialize state for new partitions
        logger.info("Partitions assigned: {}", partitions);

        // Optionally seek to specific position
        for (TopicPartition partition : partitions) {
            Long offset = getStoredOffset(partition);
            if (offset != null) {
                consumer.seek(partition, offset);
            }
        }
    }

    @Override
    public void onPartitionsLost(Collection<TopicPartition> partitions) {
        // Called when partitions lost unexpectedly (cooperative rebalancing)
        logger.warn("Partitions lost: {}", partitions);
    }
});
```

## Error Handling

### Handling Deserialization Errors
```java
props.put("key.deserializer", ErrorHandlingDeserializer.class.getName());
props.put("value.deserializer", ErrorHandlingDeserializer.class.getName());
props.put(ErrorHandlingDeserializer.KEY_DESERIALIZER_CLASS, StringDeserializer.class);
props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class);

// Access error in headers
for (ConsumerRecord<String, Order> record : records) {
    Header errorHeader = record.headers().lastHeader("springDeserializerExceptionValue");
    if (errorHeader != null) {
        // Handle deserialization error
        sendToDeadLetterQueue(record);
        continue;
    }
    processOrder(record.value());
}
```

### Dead Letter Queue Pattern
```java
public class DLQConsumer {
    private final KafkaConsumer<String, String> consumer;
    private final KafkaProducer<String, String> dlqProducer;
    private static final int MAX_RETRIES = 3;

    public void consume() {
        while (running) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

            for (ConsumerRecord<String, String> record : records) {
                int retryCount = getRetryCount(record);

                try {
                    processRecord(record);
                } catch (Exception e) {
                    if (retryCount >= MAX_RETRIES) {
                        sendToDLQ(record, e);
                    } else {
                        sendForRetry(record, retryCount + 1);
                    }
                }
            }

            consumer.commitSync();
        }
    }

    private void sendToDLQ(ConsumerRecord<String, String> record, Exception e) {
        ProducerRecord<String, String> dlqRecord = new ProducerRecord<>(
            record.topic() + ".dlq",
            record.key(),
            record.value()
        );
        dlqRecord.headers().add("error", e.getMessage().getBytes());
        dlqRecord.headers().add("original-topic", record.topic().getBytes());
        dlqRecord.headers().add("original-partition", String.valueOf(record.partition()).getBytes());
        dlqRecord.headers().add("original-offset", String.valueOf(record.offset()).getBytes());

        dlqProducer.send(dlqRecord);
    }
}
```

## Language-Specific Examples

### Node.js (kafkajs)
```typescript
import { Kafka } from 'kafkajs';

const kafka = new Kafka({ clientId: 'my-app', brokers: ['localhost:9092'] });
const consumer = kafka.consumer({ groupId: 'order-processor' });

await consumer.connect();
await consumer.subscribe({ topics: ['orders'], fromBeginning: true });

await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    const order = JSON.parse(message.value.toString());
    console.log({
      partition,
      offset: message.offset,
      key: message.key?.toString(),
      value: order,
    });

    await processOrder(order);
  },
});

// Batch processing
await consumer.run({
  eachBatch: async ({ batch, resolveOffset, heartbeat, commitOffsetsIfNecessary }) => {
    for (const message of batch.messages) {
      await processMessage(message);
      resolveOffset(message.offset);
      await heartbeat();
    }
    await commitOffsetsIfNecessary();
  },
});
```

### Python (confluent-kafka)
```python
from confluent_kafka import Consumer, KafkaError
import json

conf = {
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'order-processor',
    'auto.offset.reset': 'earliest',
    'enable.auto.commit': False,
}

consumer = Consumer(conf)
consumer.subscribe(['orders'])

try:
    while True:
        msg = consumer.poll(timeout=1.0)

        if msg is None:
            continue
        if msg.error():
            if msg.error().code() == KafkaError._PARTITION_EOF:
                continue
            raise KafkaException(msg.error())

        order = json.loads(msg.value().decode('utf-8'))
        print(f"Received: {order} from {msg.topic()}[{msg.partition()}]@{msg.offset()}")

        process_order(order)
        consumer.commit(msg)

finally:
    consumer.close()
```

### Go (segmentio/kafka-go)
```go
package main

import (
    "context"
    "encoding/json"
    "github.com/segmentio/kafka-go"
)

func main() {
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers:        []string{"localhost:9092"},
        Topic:          "orders",
        GroupID:        "order-processor",
        MinBytes:       10e3, // 10KB
        MaxBytes:       10e6, // 10MB
        CommitInterval: time.Second,
    })

    defer reader.Close()

    for {
        msg, err := reader.FetchMessage(context.Background())
        if err != nil {
            log.Printf("Error fetching message: %v", err)
            continue
        }

        var order Order
        json.Unmarshal(msg.Value, &order)
        log.Printf("Received: %v from %s[%d]@%d",
            order, msg.Topic, msg.Partition, msg.Offset)

        processOrder(order)

        if err := reader.CommitMessages(context.Background(), msg); err != nil {
            log.Printf("Failed to commit: %v", err)
        }
    }
}
```

## Monitoring

### Key Metrics
```
kafka.consumer:type=consumer-metrics,client-id=*
  - records-consumed-rate
  - bytes-consumed-rate
  - records-lag-max
  - fetch-latency-avg

kafka.consumer:type=consumer-fetch-manager-metrics,client-id=*,topic=*
  - records-lag
  - records-lead

kafka.consumer:type=consumer-coordinator-metrics,client-id=*
  - commit-latency-avg
  - rebalance-latency-avg
  - assigned-partitions
```

## Production Best Practices

### Configuration Checklist
```java
Properties props = new Properties();

// Connection
props.put("bootstrap.servers", "broker1:9092,broker2:9092,broker3:9092");
props.put("group.id", "order-processor");
props.put("client.id", "order-processor-1");

// Deserialization
props.put("key.deserializer", StringDeserializer.class.getName());
props.put("value.deserializer", StringDeserializer.class.getName());

// Offset management
props.put("enable.auto.commit", false);
props.put("auto.offset.reset", "earliest");

// Consumer group
props.put("session.timeout.ms", 30000);
props.put("heartbeat.interval.ms", 10000);
props.put("max.poll.interval.ms", 300000);
props.put("max.poll.records", 100);

// Rebalancing
props.put("partition.assignment.strategy", CooperativeStickyAssignor.class.getName());

// Fetching
props.put("fetch.min.bytes", 1);
props.put("fetch.max.wait.ms", 500);
```

### Graceful Shutdown
```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    running.set(false);
    consumer.wakeup();  // Interrupt poll()
}));

try {
    while (running.get()) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
        // Process records
        consumer.commitSync();
    }
} catch (WakeupException e) {
    // Ignore for shutdown
} finally {
    consumer.close();
}
```

## References

- [Consumer Configs](https://kafka.apache.org/documentation/#consumerconfigs)
- [Consumer Group Protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-848%3A+The+Next+Generation+of+the+Consumer+Rebalance+Protocol)
- [Incremental Cooperative Rebalancing](https://www.confluent.io/blog/incremental-cooperative-rebalancing-in-kafka/)
