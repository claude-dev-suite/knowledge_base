# Apache Kafka - Producer API

## Producer Architecture

```
Application
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│                       Kafka Producer                         │
│                                                              │
│  ┌──────────┐   ┌────────────┐   ┌────────────────────────┐ │
│  │Serializer│──▶│Partitioner │──▶│   Record Accumulator   │ │
│  └──────────┘   └────────────┘   │  ┌──────────────────┐  │ │
│                                   │  │ Batch (Topic-P0) │  │ │
│                                   │  ├──────────────────┤  │ │
│                                   │  │ Batch (Topic-P1) │  │ │
│                                   │  ├──────────────────┤  │ │
│                                   │  │ Batch (Topic-P2) │  │ │
│                                   │  └──────────────────┘  │ │
│                                   └────────────────────────┘ │
│                                              │                │
│                                              ▼                │
│                                   ┌────────────────────────┐ │
│                                   │    Sender Thread       │ │
│                                   │  (Background I/O)      │ │
│                                   └────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
                                      ┌──────────────┐
                                      │    Broker    │
                                      └──────────────┘
```

## Configuration Reference

### Essential Configurations

| Config | Default | Description | Production Value |
|--------|---------|-------------|------------------|
| `bootstrap.servers` | - | Broker addresses | Multiple brokers |
| `key.serializer` | - | Key serialization | StringSerializer |
| `value.serializer` | - | Value serialization | StringSerializer/JsonSerializer |
| `acks` | all | Acknowledgment level | `all` |
| `retries` | MAX_INT | Retry attempts | 3-5 |
| `enable.idempotence` | true | Exactly-once semantics | `true` |

### Performance Configurations

| Config | Default | Description | Tuning |
|--------|---------|-------------|--------|
| `batch.size` | 16384 | Max batch size (bytes) | 32768-65536 |
| `linger.ms` | 0 | Wait time before send | 5-100 |
| `buffer.memory` | 33554432 | Total buffer memory | 64MB+ for high throughput |
| `compression.type` | none | Compression codec | `lz4`, `snappy`, `zstd` |
| `max.in.flight.requests.per.connection` | 5 | Concurrent requests | 5 (with idempotence) |

### Reliability Configurations

| Config | Default | Description | Production Value |
|--------|---------|-------------|------------------|
| `acks` | all | Acknowledgment mode | `all` |
| `min.insync.replicas` | 1 | Min replicas to ack | 2 |
| `retries` | MAX_INT | Retry count | 3-5 |
| `retry.backoff.ms` | 100 | Retry delay | 100-500 |
| `delivery.timeout.ms` | 120000 | Max delivery time | 120000 |
| `request.timeout.ms` | 30000 | Request timeout | 30000 |

## Serialization

### Built-in Serializers
```java
// String
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");

// Integer
props.put("value.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");

// Bytes
props.put("value.serializer", "org.apache.kafka.common.serialization.ByteArraySerializer");
```

### JSON Serialization

#### Java (Jackson)
```java
public class JsonSerializer<T> implements Serializer<T> {
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public byte[] serialize(String topic, T data) {
        if (data == null) return null;
        try {
            return objectMapper.writeValueAsBytes(data);
        } catch (JsonProcessingException e) {
            throw new SerializationException("Error serializing JSON", e);
        }
    }
}
```

#### Node.js
```typescript
const producer = kafka.producer();

await producer.send({
  topic: 'orders',
  messages: [{
    key: orderId,
    value: JSON.stringify(order),  // Manual JSON serialization
  }],
});
```

### Avro Serialization (Schema Registry)

```java
// Configuration
props.put("key.serializer", "io.confluent.kafka.serializers.KafkaAvroSerializer");
props.put("value.serializer", "io.confluent.kafka.serializers.KafkaAvroSerializer");
props.put("schema.registry.url", "http://localhost:8081");

// Usage
GenericRecord order = new GenericData.Record(schema);
order.put("id", "123");
order.put("amount", 100.0);
producer.send(new ProducerRecord<>("orders", orderId, order));
```

## Producer Patterns

### Synchronous Send
```java
// Java
try {
    RecordMetadata metadata = producer.send(record).get();
    System.out.printf("Sent to partition %d, offset %d%n",
        metadata.partition(), metadata.offset());
} catch (ExecutionException e) {
    // Handle send failure
}
```

```typescript
// Node.js
const result = await producer.send({
  topic: 'orders',
  messages: [{ value: JSON.stringify(order) }],
});
console.log(`Sent to partition ${result[0].partition}, offset ${result[0].offset}`);
```

### Asynchronous Send with Callback
```java
// Java
producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        logger.error("Send failed", exception);
        // Handle failure (retry, DLQ, etc.)
    } else {
        logger.info("Sent to {}:{}", metadata.partition(), metadata.offset());
    }
});
```

```typescript
// Node.js - fire-and-forget with error handling
producer.send({
  topic: 'orders',
  messages: [{ value: JSON.stringify(order) }],
}).catch(error => {
  console.error('Send failed:', error);
});
```

### Idempotent Producer
```java
// Exactly-once semantics within a session
Properties props = new Properties();
props.put("enable.idempotence", true);
props.put("acks", "all");
props.put("retries", 3);
props.put("max.in.flight.requests.per.connection", 5);  // Safe with idempotence

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
```

```typescript
// Node.js (kafkajs)
const producer = kafka.producer({
  idempotent: true,
  maxInFlightRequests: 5,
});
```

### Transactional Producer
```java
// Exactly-once across multiple partitions/topics
Properties props = new Properties();
props.put("transactional.id", "order-processor-1");
props.put("enable.idempotence", true);

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.initTransactions();

try {
    producer.beginTransaction();

    producer.send(new ProducerRecord<>("orders", orderId, orderJson));
    producer.send(new ProducerRecord<>("inventory", itemId, updateJson));
    producer.send(new ProducerRecord<>("payments", paymentId, paymentJson));

    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
    throw e;
}
```

## Partitioning Strategies

### Default Partitioner
```java
// With key: hash-based partitioning (consistent)
producer.send(new ProducerRecord<>("orders", customerId, order));
// Same key always goes to same partition

// Without key: sticky partitioning (batches to same partition)
producer.send(new ProducerRecord<>("orders", order));
```

### Custom Partitioner
```java
public class PriorityPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes, Cluster cluster) {

        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();

        Order order = deserialize(valueBytes);

        if (order.getPriority() == Priority.HIGH) {
            return 0;  // High priority to partition 0
        }

        // Standard partitioning for others
        return Math.abs(key.hashCode()) % (numPartitions - 1) + 1;
    }
}

// Configuration
props.put("partitioner.class", "com.example.PriorityPartitioner");
```

### Round-Robin (No Key)
```java
// Explicitly null key for round-robin
producer.send(new ProducerRecord<>("logs", null, logMessage));
```

## Error Handling

### Retriable vs Non-Retriable Errors

| Retriable | Non-Retriable |
|-----------|---------------|
| `NetworkException` | `SerializationException` |
| `NotEnoughReplicasException` | `RecordTooLargeException` |
| `TimeoutException` | `InvalidTopicException` |
| `LeaderNotAvailableException` | `OffsetMetadataTooLarge` |

### Retry Configuration
```java
// Automatic retries
props.put("retries", 3);
props.put("retry.backoff.ms", 100);
props.put("delivery.timeout.ms", 120000);  // Total time for retries

// Manual retry with backoff
int maxRetries = 3;
for (int i = 0; i < maxRetries; i++) {
    try {
        producer.send(record).get();
        break;
    } catch (RetriableException e) {
        if (i == maxRetries - 1) throw e;
        Thread.sleep((long) Math.pow(2, i) * 100);  // Exponential backoff
    }
}
```

### Dead Letter Queue Pattern
```java
public class SafeProducer {
    private final KafkaProducer<String, String> producer;
    private final KafkaProducer<String, String> dlqProducer;

    public void send(String topic, String key, String value) {
        try {
            producer.send(new ProducerRecord<>(topic, key, value)).get();
        } catch (Exception e) {
            // Send to DLQ
            dlqProducer.send(new ProducerRecord<>(
                topic + ".dlq",
                key,
                createDlqMessage(value, e)
            ));
        }
    }
}
```

## Performance Optimization

### Batching
```java
// Increase batch size for throughput
props.put("batch.size", 65536);  // 64KB
props.put("linger.ms", 10);      // Wait up to 10ms to fill batch

// Trade-off: latency vs throughput
// linger.ms=0: lowest latency, smaller batches
// linger.ms=100: higher latency, larger batches, better throughput
```

### Compression
```java
// Compression at producer (decompressed at consumer)
props.put("compression.type", "lz4");  // Fast compression
// Options: none, gzip (slow, best ratio), snappy (balanced), lz4 (fast), zstd (best ratio after gzip)

// Benchmark results (typical):
// none:   100% size, fastest
// lz4:    ~25% size, very fast
// snappy: ~30% size, fast
// zstd:   ~20% size, moderate
// gzip:   ~25% size, slow
```

### Buffer Memory
```java
// For high-throughput producers
props.put("buffer.memory", 67108864);  // 64MB
props.put("max.block.ms", 60000);       // Block if buffer full
```

## Monitoring Producer

### Key Metrics
```java
// Get metrics
Map<MetricName, ? extends Metric> metrics = producer.metrics();

// Important metrics:
// record-send-rate: Records sent per second
// record-error-rate: Failed sends per second
// request-latency-avg: Average request latency
// batch-size-avg: Average batch size
// compression-rate-avg: Compression ratio
// buffer-available-bytes: Remaining buffer
```

### JMX Metrics
```
kafka.producer:type=producer-metrics,client-id=*
  - record-send-rate
  - record-error-rate
  - request-latency-avg
  - request-latency-max
  - outgoing-byte-rate
  - batch-size-avg
  - compression-rate-avg
```

## Production Best Practices

### Configuration Checklist
```java
Properties props = new Properties();

// Connection
props.put("bootstrap.servers", "broker1:9092,broker2:9092,broker3:9092");
props.put("client.id", "order-service-producer");

// Serialization
props.put("key.serializer", StringSerializer.class.getName());
props.put("value.serializer", StringSerializer.class.getName());

// Reliability
props.put("acks", "all");
props.put("enable.idempotence", true);
props.put("retries", 3);
props.put("max.in.flight.requests.per.connection", 5);

// Performance
props.put("batch.size", 32768);
props.put("linger.ms", 5);
props.put("compression.type", "lz4");
props.put("buffer.memory", 67108864);

// Timeouts
props.put("request.timeout.ms", 30000);
props.put("delivery.timeout.ms", 120000);
```

### Graceful Shutdown
```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    producer.flush();   // Send remaining messages
    producer.close();   // Release resources
}));
```

### Connection Management
```java
// Reuse producer instance (thread-safe)
// Don't create producer per message

// Single producer for application
private static final KafkaProducer<String, String> producer = createProducer();

public void sendMessage(String topic, String key, String value) {
    producer.send(new ProducerRecord<>(topic, key, value));
}
```

## Language-Specific Examples

### Python (confluent-kafka)
```python
from confluent_kafka import Producer
import json

conf = {
    'bootstrap.servers': 'localhost:9092',
    'client.id': 'order-service',
    'acks': 'all',
    'enable.idempotence': True,
    'retries': 3,
    'batch.size': 32768,
    'linger.ms': 5,
    'compression.type': 'lz4',
}

producer = Producer(conf)

def delivery_callback(err, msg):
    if err:
        print(f'Delivery failed: {err}')
    else:
        print(f'Delivered to {msg.topic()}[{msg.partition()}]@{msg.offset()}')

producer.produce(
    topic='orders',
    key=order_id.encode('utf-8'),
    value=json.dumps(order).encode('utf-8'),
    callback=delivery_callback,
    headers={'correlation-id': correlation_id}
)

producer.flush()  # Block until all messages sent
```

### Go (segmentio/kafka-go)
```go
package main

import (
    "context"
    "encoding/json"
    "github.com/segmentio/kafka-go"
    "time"
)

func main() {
    writer := &kafka.Writer{
        Addr:         kafka.TCP("localhost:9092"),
        Topic:        "orders",
        Balancer:     &kafka.Hash{},  // Key-based partitioning
        RequiredAcks: kafka.RequireAll,
        Compression:  kafka.Lz4,
        BatchSize:    100,
        BatchTimeout: 10 * time.Millisecond,
    }

    defer writer.Close()

    order := Order{ID: "123", Amount: 100}
    value, _ := json.Marshal(order)

    err := writer.WriteMessages(context.Background(),
        kafka.Message{
            Key:   []byte(order.ID),
            Value: value,
            Headers: []kafka.Header{
                {Key: "correlation-id", Value: []byte("abc123")},
            },
        },
    )

    if err != nil {
        log.Printf("Failed to write message: %v", err)
    }
}
```

## References

- [Producer Configs](https://kafka.apache.org/documentation/#producerconfigs)
- [Exactly-Once Semantics](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)
- [Producer Performance Tuning](https://docs.confluent.io/platform/current/installation/configuration/producer-configs.html)
