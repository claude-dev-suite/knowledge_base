# Spring Kafka - Producer

## KafkaTemplate

The `KafkaTemplate` is the main class for producing messages to Kafka topics.

### Basic Usage

```java
@Service
@RequiredArgsConstructor
public class OrderProducer {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    // Fire and forget
    public void sendAsync(String topic, String key, Object message) {
        kafkaTemplate.send(topic, key, message);
    }

    // With callback
    public void sendWithCallback(String topic, String key, Object message) {
        CompletableFuture<SendResult<String, Object>> future =
            kafkaTemplate.send(topic, key, message);

        future.whenComplete((result, ex) -> {
            if (ex != null) {
                log.error("Failed to send message", ex);
            } else {
                RecordMetadata metadata = result.getRecordMetadata();
                log.info("Sent to partition {} offset {}",
                    metadata.partition(), metadata.offset());
            }
        });
    }

    // Synchronous send
    public void sendSync(String topic, String key, Object message) {
        try {
            SendResult<String, Object> result =
                kafkaTemplate.send(topic, key, message).get(10, TimeUnit.SECONDS);

            log.info("Message sent successfully to offset {}",
                result.getRecordMetadata().offset());
        } catch (ExecutionException | InterruptedException | TimeoutException e) {
            throw new MessageSendException("Failed to send message", e);
        }
    }
}
```

## ProducerRecord

### Explicit Partition
```java
public void sendToPartition(Object message, int partition) {
    ProducerRecord<String, Object> record = new ProducerRecord<>(
        "orders",      // topic
        partition,     // partition
        "order-123",   // key
        message        // value
    );

    kafkaTemplate.send(record);
}
```

### With Timestamp
```java
public void sendWithTimestamp(Object message, long timestamp) {
    ProducerRecord<String, Object> record = new ProducerRecord<>(
        "orders",       // topic
        null,           // partition (auto)
        timestamp,      // timestamp
        "order-123",    // key
        message         // value
    );

    kafkaTemplate.send(record);
}
```

## Message Builder Pattern

```java
public void sendWithHeaders(OrderEvent event, String correlationId) {
    Message<OrderEvent> message = MessageBuilder
        .withPayload(event)
        .setHeader(KafkaHeaders.TOPIC, "orders")
        .setHeader(KafkaHeaders.KEY, event.orderId())
        .setHeader(KafkaHeaders.PARTITION, 0)
        .setHeader("X-Correlation-Id", correlationId)
        .setHeader("X-Event-Type", "ORDER_CREATED")
        .setHeader("X-Timestamp", Instant.now().toEpochMilli())
        .build();

    kafkaTemplate.send(message);
}
```

## Transactional Producer

### Configuration
```yaml
spring:
  kafka:
    producer:
      transaction-id-prefix: tx-
```

### Transactional Sending
```java
@Configuration
public class KafkaTransactionConfig {

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate(ProducerFactory<String, Object> pf) {
        KafkaTemplate<String, Object> template = new KafkaTemplate<>(pf);
        template.setTransactionIdPrefix("order-tx-");
        return template;
    }
}

@Service
@RequiredArgsConstructor
public class TransactionalOrderService {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    @Transactional
    public void processOrder(Order order) {
        kafkaTemplate.executeInTransaction(operations -> {
            operations.send("orders", order.getId(), new OrderCreatedEvent(order));
            operations.send("inventory", order.getId(), new InventoryReserveEvent(order));
            operations.send("payments", order.getId(), new PaymentRequestEvent(order));
            return null;
        });
    }
}
```

## Idempotent Producer

Ensures exactly-once semantics for single partitions:

```yaml
spring:
  kafka:
    producer:
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 5
        acks: all
        retries: 2147483647
```

## Batch Sending

```java
@Service
@RequiredArgsConstructor
public class BatchProducer {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public void sendBatch(List<OrderEvent> events) {
        List<CompletableFuture<SendResult<String, Object>>> futures = events.stream()
            .map(event -> kafkaTemplate.send("orders", event.orderId(), event))
            .toList();

        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Batch send failed", ex);
                } else {
                    log.info("Batch of {} messages sent successfully", events.size());
                }
            });
    }
}
```

## Producer Configuration

```yaml
spring:
  kafka:
    producer:
      # Serializers
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

      # Delivery guarantees
      acks: all                    # Wait for all replicas
      retries: 3                   # Retry count

      # Batching
      batch-size: 16384            # Batch size in bytes
      linger-ms: 5                 # Wait for batch to fill
      buffer-memory: 33554432      # Total buffer memory

      # Compression
      compression-type: snappy     # none, gzip, snappy, lz4, zstd

      # Properties
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 5
        delivery.timeout.ms: 120000
        request.timeout.ms: 30000
```

## Routing Messages

### By Key (Default Partitioner)
```java
// Same key always goes to same partition
kafkaTemplate.send("orders", customerId, order);  // Partitioned by customerId
```

### Custom Partitioner
```java
public class RegionPartitioner implements Partitioner {

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes, Cluster cluster) {
        if (key == null) {
            return 0;
        }

        // Route by region
        String region = extractRegion(key.toString());
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);

        return switch (region) {
            case "EU" -> 0;
            case "US" -> 1;
            case "ASIA" -> 2;
            default -> Math.abs(key.hashCode()) % partitions.size();
        };
    }

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs) {}
}
```

## ProducerListener

```java
@Component
@Slf4j
public class OrderProducerListener implements ProducerListener<String, Object> {

    @Override
    public void onSuccess(ProducerRecord<String, Object> record, RecordMetadata metadata) {
        log.info("Message sent to {} partition {} offset {}",
            record.topic(), metadata.partition(), metadata.offset());
    }

    @Override
    public void onError(ProducerRecord<String, Object> record,
                       RecordMetadata metadata, Exception exception) {
        log.error("Failed to send message to {}: {}",
            record.topic(), exception.getMessage());
        // Trigger alert or fallback
    }
}

// Register listener
@Configuration
public class KafkaConfig {

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate(
            ProducerFactory<String, Object> pf,
            OrderProducerListener listener) {
        KafkaTemplate<String, Object> template = new KafkaTemplate<>(pf);
        template.setProducerListener(listener);
        return template;
    }
}
```

## RoutingKafkaTemplate

Route to different producer configurations based on topic pattern:

```java
@Bean
public RoutingKafkaTemplate routingTemplate(GenericApplicationContext context) {
    // Standard producer
    Map<String, Object> standardProps = new HashMap<>();
    standardProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    standardProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
    DefaultKafkaProducerFactory<Object, Object> standardFactory =
        new DefaultKafkaProducerFactory<>(standardProps);

    // High-throughput producer
    Map<String, Object> highThroughputProps = new HashMap<>(standardProps);
    highThroughputProps.put(ProducerConfig.BATCH_SIZE_CONFIG, 65536);
    highThroughputProps.put(ProducerConfig.LINGER_MS_CONFIG, 20);
    DefaultKafkaProducerFactory<Object, Object> highThroughputFactory =
        new DefaultKafkaProducerFactory<>(highThroughputProps);

    Map<Pattern, ProducerFactory<Object, Object>> map = new LinkedHashMap<>();
    map.put(Pattern.compile("events.*"), highThroughputFactory);
    map.put(Pattern.compile(".*"), standardFactory);

    return new RoutingKafkaTemplate(map);
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use async sending with callbacks | Block on every send |
| Enable idempotence for critical data | Risk duplicate messages |
| Configure proper retries and timeouts | Use defaults blindly |
| Use transactions for multi-topic writes | Send related messages non-atomically |
| Batch small messages | Send one message at a time |
| Use compression for large messages | Send uncompressed large payloads |
