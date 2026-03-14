# Spring Kafka - Error Handling

## Default Error Handling

Spring Kafka provides multiple strategies for handling consumer errors.

### DefaultErrorHandler (Spring Kafka 2.8+)

```java
@Configuration
public class KafkaErrorConfig {

    @Bean
    public DefaultErrorHandler errorHandler(KafkaOperations<String, Object> kafkaOperations) {
        // Retry 3 times with 1 second delay, then send to DLQ
        DeadLetterPublishingRecoverer recoverer =
            new DeadLetterPublishingRecoverer(kafkaOperations,
                (record, ex) -> new TopicPartition(record.topic() + ".dlq", record.partition()));

        DefaultErrorHandler handler = new DefaultErrorHandler(recoverer,
            new FixedBackOff(1000L, 3L));  // 1s delay, 3 attempts

        // Don't retry these exceptions
        handler.addNotRetryableExceptions(
            ValidationException.class,
            DeserializationException.class
        );

        return handler;
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory,
            DefaultErrorHandler errorHandler) {

        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setCommonErrorHandler(errorHandler);
        return factory;
    }
}
```

## Retry Mechanisms

### FixedBackOff

```java
// Fixed delay between retries
DefaultErrorHandler handler = new DefaultErrorHandler(
    recoverer,
    new FixedBackOff(1000L, 3L)  // 1s delay, 3 retries
);
```

### ExponentialBackOff

```java
// Exponential backoff
ExponentialBackOffWithMaxRetries backOff = new ExponentialBackOffWithMaxRetries(5);
backOff.setInitialInterval(1000L);
backOff.setMultiplier(2.0);
backOff.setMaxInterval(10000L);

DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);
```

### Custom BackOff

```java
// Custom backoff intervals
DefaultErrorHandler handler = new DefaultErrorHandler(
    recoverer,
    new FixedBackOff(0, 0)  // Immediate first retry
);

handler.setBackOffFunction((record, exception) -> {
    int attempts = getAttemptCount(record);
    return switch (attempts) {
        case 1 -> new FixedBackOff(1000L, 1L);
        case 2 -> new FixedBackOff(5000L, 1L);
        case 3 -> new FixedBackOff(30000L, 1L);
        default -> new FixedBackOff(0L, 0L);  // Go to DLQ
    };
});
```

## Dead Letter Queue (DLQ)

### Basic DLQ

```java
@Bean
public DeadLetterPublishingRecoverer dlqRecoverer(KafkaOperations<String, Object> operations) {
    return new DeadLetterPublishingRecoverer(operations);
    // Sends to <topic>.DLT by default
}
```

### Custom DLQ Topic

```java
@Bean
public DeadLetterPublishingRecoverer dlqRecoverer(KafkaOperations<String, Object> operations) {
    return new DeadLetterPublishingRecoverer(operations,
        (record, ex) -> {
            String dlqTopic = record.topic() + ".dlq";
            return new TopicPartition(dlqTopic, record.partition());
        });
}
```

### DLQ with Headers

```java
@Bean
public DeadLetterPublishingRecoverer dlqRecoverer(KafkaOperations<String, Object> operations) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(operations,
        (record, ex) -> new TopicPartition(record.topic() + ".dlq", -1));  // Any partition

    // Add custom headers
    recoverer.setHeadersFunction((record, ex) -> {
        RecordHeaders headers = new RecordHeaders();
        headers.add("X-Error-Message", ex.getMessage().getBytes());
        headers.add("X-Error-Type", ex.getClass().getName().getBytes());
        headers.add("X-Failed-Timestamp", Long.toString(System.currentTimeMillis()).getBytes());
        headers.add("X-Original-Topic", record.topic().getBytes());
        headers.add("X-Original-Partition", Integer.toString(record.partition()).getBytes());
        headers.add("X-Original-Offset", Long.toString(record.offset()).getBytes());
        return headers;
    });

    return recoverer;
}
```

## @RetryableTopic

Spring Kafka 2.7+ provides declarative retry topics:

```java
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 10000),
    topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE,
    dltStrategy = DltStrategy.FAIL_ON_ERROR
)
@KafkaListener(topics = "orders", groupId = "order-service")
public void consume(OrderEvent event) {
    processOrder(event);  // Throws exception on failure
}

@DltHandler
public void handleDlt(OrderEvent event, @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
    log.error("DLT received: {} from {}", event, topic);
    alertService.notifyDltMessage(event);
}
```

### Retry Topics Created

```
orders-retry-0  (1s delay)
orders-retry-1  (2s delay)
orders-retry-2  (4s delay)
orders-dlt      (dead letter topic)
```

### Configuration

```java
@Configuration
@EnableKafka
public class KafkaRetryConfig implements RetryTopicConfigurationSupport {

    @Override
    protected void configureBlockingRetries(BlockingRetriesConfigurer configurer) {
        configurer
            .retryOn(RetryableException.class)
            .backOff(new FixedBackOff(1000, 3));
    }

    @Override
    protected void configureNonBlockingRetries(NonBlockingRetriesConfigurer configurer) {
        configurer
            .addToFatalExceptions(DeserializationException.class)
            .setCommitAckAfterRecoverable(true);
    }
}
```

## Exception Classification

### Retryable vs Non-Retryable

```java
@Bean
public DefaultErrorHandler errorHandler(DeadLetterPublishingRecoverer recoverer) {
    DefaultErrorHandler handler = new DefaultErrorHandler(recoverer,
        new FixedBackOff(1000L, 3L));

    // These exceptions go directly to DLQ (no retry)
    handler.addNotRetryableExceptions(
        DeserializationException.class,
        ValidationException.class,
        NullPointerException.class,
        ClassCastException.class
    );

    // These exceptions are retried
    handler.addRetryableExceptions(
        TimeoutException.class,
        ConnectException.class,
        TransientDataAccessException.class
    );

    return handler;
}
```

## Custom Error Handler

```java
@Component
@Slf4j
public class CustomErrorHandler implements CommonErrorHandler {

    private final AlertService alertService;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @Override
    public void handleRemaining(Exception thrownException,
                                List<ConsumerRecord<?, ?>> records,
                                Consumer<?, ?> consumer,
                                MessageListenerContainer container) {

        for (ConsumerRecord<?, ?> record : records) {
            log.error("Error processing record from topic {} partition {} offset {}",
                record.topic(), record.partition(), record.offset(), thrownException);

            // Send to DLQ with metadata
            sendToDlq(record, thrownException);

            // Alert on critical failures
            if (isCriticalError(thrownException)) {
                alertService.sendAlert("Kafka processing failure", thrownException);
            }
        }
    }

    @Override
    public boolean handleOne(Exception thrownException,
                            ConsumerRecord<?, ?> record,
                            Consumer<?, ?> consumer,
                            MessageListenerContainer container) {
        log.error("Error processing record: {}", record, thrownException);
        sendToDlq(record, thrownException);
        return true;  // Message handled, continue processing
    }

    private void sendToDlq(ConsumerRecord<?, ?> record, Exception ex) {
        ProducerRecord<String, Object> dlqRecord = new ProducerRecord<>(
            record.topic() + ".dlq",
            null,
            record.key() != null ? record.key().toString() : null,
            record.value()
        );

        dlqRecord.headers().add("X-Error", ex.getMessage().getBytes());
        dlqRecord.headers().add("X-Original-Topic", record.topic().getBytes());

        kafkaTemplate.send(dlqRecord);
    }
}
```

## Deserialization Errors

### ErrorHandlingDeserializer

```yaml
spring:
  kafka:
    consumer:
      key-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        spring.deserializer.key.delegate.class: org.apache.kafka.common.serialization.StringDeserializer
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
        spring.json.trusted.packages: "com.example.events"
```

### Handling in Listener

```java
@KafkaListener(topics = "orders", groupId = "order-service")
public void consume(ConsumerRecord<String, OrderEvent> record,
                   @Header(KafkaHeaders.EXCEPTION_FQCN) String exceptionType,
                   @Header(KafkaHeaders.EXCEPTION_MESSAGE) String exceptionMessage) {

    if (exceptionType != null) {
        log.error("Deserialization error: {} - {}", exceptionType, exceptionMessage);
        return;  // Skip bad messages
    }

    processOrder(record.value());
}
```

## DLQ Consumer

Process messages from DLQ for analysis or reprocessing:

```java
@Service
@Slf4j
public class DlqProcessor {

    @KafkaListener(topics = "orders.dlq", groupId = "dlq-processor")
    public void processDlq(
            ConsumerRecord<String, byte[]> record,
            @Header("X-Error-Message") byte[] errorMessage,
            @Header("X-Original-Topic") byte[] originalTopic) {

        log.info("DLQ message from {}: error={}",
            new String(originalTopic), new String(errorMessage));

        // Store for later analysis
        dlqRepository.save(DlqMessage.builder()
            .key(record.key())
            .payload(record.value())
            .errorMessage(new String(errorMessage))
            .originalTopic(new String(originalTopic))
            .receivedAt(Instant.now())
            .build());
    }
}
```

## Logging and Monitoring

```java
@Component
@Slf4j
public class KafkaErrorLoggingListener implements RecordInterceptor<String, Object> {

    private final MeterRegistry meterRegistry;

    @Override
    public ConsumerRecord<String, Object> intercept(ConsumerRecord<String, Object> record,
                                                    Consumer<String, Object> consumer) {
        return record;
    }

    @Override
    public void failure(ConsumerRecord<String, Object> record, Exception exception,
                       Consumer<String, Object> consumer) {
        log.error("Failed to process record: topic={}, partition={}, offset={}, error={}",
            record.topic(), record.partition(), record.offset(), exception.getMessage());

        meterRegistry.counter("kafka.consumer.error",
            "topic", record.topic(),
            "exception", exception.getClass().getSimpleName()
        ).increment();
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Configure DLQ for all topics | Lose failed messages |
| Use exponential backoff | Hammer services on retry |
| Classify exceptions properly | Retry non-retryable errors |
| Add error headers to DLQ | Lose error context |
| Monitor DLQ size | Ignore accumulated failures |
| Set up alerts for DLQ | Let problems accumulate |
