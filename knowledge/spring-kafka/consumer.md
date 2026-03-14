# Spring Kafka - Consumer

## @KafkaListener

The `@KafkaListener` annotation enables declarative consumption of Kafka messages.

### Basic Consumer

```java
@Service
@Slf4j
public class OrderConsumer {

    @KafkaListener(topics = "orders", groupId = "order-service")
    public void consume(OrderEvent event) {
        log.info("Received order: {}", event.orderId());
        processOrder(event);
    }
}
```

### Multiple Topics

```java
@KafkaListener(topics = {"orders", "returns", "refunds"}, groupId = "order-service")
public void consume(Object event) {
    if (event instanceof OrderEvent order) {
        processOrder(order);
    } else if (event instanceof ReturnEvent returnEvent) {
        processReturn(returnEvent);
    }
}

// Or with topic pattern
@KafkaListener(topicPattern = "order.*", groupId = "order-service")
public void consumePattern(Object event, @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
    log.info("Received from topic {}: {}", topic, event);
}
```

## Consumer with Metadata

```java
@KafkaListener(topics = "orders", groupId = "order-service")
public void consume(
        @Payload OrderEvent event,
        @Header(KafkaHeaders.RECEIVED_KEY) String key,
        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
        @Header(KafkaHeaders.OFFSET) long offset,
        @Header(KafkaHeaders.RECEIVED_TIMESTAMP) long timestamp,
        @Header(name = "X-Correlation-Id", required = false) String correlationId,
        Acknowledgment ack) {

    log.info("Processing order {} from partition {} offset {}",
        event.orderId(), partition, offset);

    try {
        orderService.process(event);
        ack.acknowledge();  // Manual ack
    } catch (Exception e) {
        log.error("Failed to process order", e);
        // Don't acknowledge - message will be redelivered
    }
}
```

## ConsumerRecord

```java
@KafkaListener(topics = "orders", groupId = "order-service")
public void consume(ConsumerRecord<String, OrderEvent> record) {
    log.info("Key: {}, Value: {}", record.key(), record.value());
    log.info("Partition: {}, Offset: {}", record.partition(), record.offset());
    log.info("Timestamp: {}", record.timestamp());

    record.headers().forEach(header ->
        log.info("Header: {} = {}", header.key(), new String(header.value())));
}
```

## Batch Consumption

```java
@KafkaListener(
    topics = "orders",
    groupId = "order-batch-service",
    containerFactory = "batchKafkaListenerContainerFactory"
)
public void consumeBatch(List<OrderEvent> events) {
    log.info("Received batch of {} orders", events.size());
    orderService.processBatch(events);
}

// Or with ConsumerRecords
@KafkaListener(topics = "orders", groupId = "order-service")
public void consumeBatch(ConsumerRecords<String, OrderEvent> records) {
    for (ConsumerRecord<String, OrderEvent> record : records) {
        process(record.value());
    }
}
```

### Batch Configuration

```java
@Configuration
public class KafkaBatchConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> batchKafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory) {

        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setBatchListener(true);
        factory.getContainerProperties().setIdleBetweenPolls(100);

        // Batch size control
        factory.getContainerProperties().setPollTimeout(3000);

        return factory;
    }
}
```

## Manual Offset Management

### Configuration

```yaml
spring:
  kafka:
    consumer:
      enable-auto-commit: false
    listener:
      ack-mode: manual
```

### Manual Acknowledgment

```java
@KafkaListener(topics = "orders", groupId = "order-service")
public void consume(OrderEvent event, Acknowledgment ack) {
    try {
        orderService.process(event);
        ack.acknowledge();  // Commit offset
    } catch (RetryableException e) {
        // Don't ack - will be redelivered
        throw e;
    }
}
```

### Ack Modes

| Mode | Description |
|------|-------------|
| `RECORD` | Ack after each record |
| `BATCH` | Ack after batch processed |
| `TIME` | Ack after time interval |
| `COUNT` | Ack after count threshold |
| `COUNT_TIME` | Ack after count OR time |
| `MANUAL` | Explicit ack required |
| `MANUAL_IMMEDIATE` | Immediate ack on call |

## Concurrency

### Partition-Based Concurrency

```java
@KafkaListener(
    topics = "orders",
    groupId = "order-service",
    concurrency = "3"  // 3 consumer threads
)
public void consume(OrderEvent event) {
    // Each thread handles specific partitions
    processOrder(event);
}
```

### Configuration

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
        ConsumerFactory<String, Object> consumerFactory) {

    ConcurrentKafkaListenerContainerFactory<String, Object> factory =
        new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory);
    factory.setConcurrency(3);  // 3 threads per listener
    factory.getContainerProperties().setAckMode(AckMode.MANUAL);
    return factory;
}
```

## Seek Operations

### Seek to Beginning

```java
@KafkaListener(
    id = "orderListener",
    topics = "orders",
    groupId = "order-service"
)
public void consume(OrderEvent event) {
    processOrder(event);
}

// Seek to beginning via ConsumerSeekAware
@Component
public class SeekToBeginningListener implements ConsumerSeekAware {

    private ConsumerSeekCallback callback;

    @Override
    public void registerSeekCallback(ConsumerSeekCallback callback) {
        this.callback = callback;
    }

    @Override
    public void onPartitionsAssigned(Map<TopicPartition, Long> assignments,
                                     ConsumerSeekCallback callback) {
        assignments.keySet().forEach(tp -> callback.seekToBeginning(tp.topic(), tp.partition()));
    }
}
```

### Seek to Timestamp

```java
public void seekToTimestamp(String topic, long timestamp) {
    KafkaListenerEndpointRegistry registry;

    MessageListenerContainer container = registry.getListenerContainer("orderListener");
    if (container instanceof ConcurrentMessageListenerContainer<?, ?> concurrent) {
        concurrent.getContainers().forEach(c -> {
            c.getAssignedPartitions().forEach(tp -> {
                c.seekToTimestamp(tp.topic(), tp.partition(), timestamp);
            });
        });
    }
}
```

## Filtering Messages

### RecordFilterStrategy

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, Object> filteringFactory(
        ConsumerFactory<String, Object> consumerFactory) {

    ConcurrentKafkaListenerContainerFactory<String, Object> factory =
        new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory);

    // Filter out messages
    factory.setRecordFilterStrategy(record -> {
        OrderEvent event = (OrderEvent) record.value();
        return event.amount().compareTo(BigDecimal.ZERO) <= 0;  // Filter if true
    });

    factory.setAckDiscarded(true);  // Ack filtered messages

    return factory;
}
```

### @KafkaListener Filter

```java
@KafkaListener(
    topics = "orders",
    filter = "orderFilter",
    groupId = "order-service"
)
public void consume(OrderEvent event) {
    // Only receives non-filtered messages
}

@Bean
public RecordFilterStrategy<String, OrderEvent> orderFilter() {
    return record -> record.value().status().equals("CANCELLED");
}
```

## Pausing and Resuming

```java
@Service
@RequiredArgsConstructor
public class ConsumerControlService {

    private final KafkaListenerEndpointRegistry registry;

    public void pauseConsumer(String listenerId) {
        MessageListenerContainer container = registry.getListenerContainer(listenerId);
        if (container != null) {
            container.pause();
            log.info("Paused consumer: {}", listenerId);
        }
    }

    public void resumeConsumer(String listenerId) {
        MessageListenerContainer container = registry.getListenerContainer(listenerId);
        if (container != null) {
            container.resume();
            log.info("Resumed consumer: {}", listenerId);
        }
    }

    public boolean isRunning(String listenerId) {
        MessageListenerContainer container = registry.getListenerContainer(listenerId);
        return container != null && container.isRunning();
    }
}
```

## Consumer Configuration

```yaml
spring:
  kafka:
    consumer:
      group-id: my-service
      auto-offset-reset: earliest  # earliest, latest, none
      enable-auto-commit: false

      # Polling
      max-poll-records: 500
      fetch-min-size: 1
      fetch-max-wait: 500ms

      # Session
      heartbeat-interval: 3s
      session-timeout: 45s

      # Deserializers
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer

      properties:
        spring.json.trusted.packages: "com.example.events"
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

## Best Practices

| Do | Don't |
|----|-------|
| Use manual acks for critical messages | Rely on auto-commit for important data |
| Match concurrency to partition count | Over-provision consumers |
| Handle exceptions properly | Swallow exceptions silently |
| Use idempotent consumers | Assume no redelivery |
| Configure proper timeouts | Use infinite blocking |
| Monitor consumer lag | Ignore backlog buildup |
