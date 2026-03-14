# Spring Kafka - Basics

## Overview

Spring for Apache Kafka provides a high-level abstraction for Kafka-based messaging solutions. It simplifies the use of Apache Kafka by providing template-based approaches for sending and receiving messages.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

## Configuration

### application.yml
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: my-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.example.dto"
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
```

### Java Configuration
```java
@Configuration
@EnableKafka
public class KafkaConfig {

    @Bean
    public ProducerFactory<String, Object> producerFactory(KafkaProperties properties) {
        Map<String, Object> props = new HashMap<>(properties.buildProducerProperties());
        return new DefaultKafkaProducerFactory<>(props);
    }

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate(ProducerFactory<String, Object> factory) {
        return new KafkaTemplate<>(factory);
    }

    @Bean
    public ConsumerFactory<String, Object> consumerFactory(KafkaProperties properties) {
        Map<String, Object> props = new HashMap<>(properties.buildConsumerProperties());
        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory) {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setConcurrency(3);
        return factory;
    }
}
```

## Topic Configuration

### Programmatic Topic Creation
```java
@Configuration
public class KafkaTopicConfig {

    @Bean
    public NewTopic ordersTopic() {
        return TopicBuilder.name("orders")
            .partitions(6)
            .replicas(3)
            .config(TopicConfig.RETENTION_MS_CONFIG, "604800000") // 7 days
            .build();
    }

    @Bean
    public NewTopic ordersDlqTopic() {
        return TopicBuilder.name("orders.dlq")
            .partitions(1)
            .replicas(3)
            .build();
    }
}
```

## Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Producer   │────▶│    Kafka    │────▶│  Consumer   │
│  Service    │     │   Broker    │     │  Service    │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │
       ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ KafkaTemplate│    │   Topics    │     │ @KafkaListener│
└─────────────┘     │ Partitions  │     └─────────────┘
                    └─────────────┘
```

## Message Types

### String Messages
```java
// Producer
kafkaTemplate.send("topic", "key", "Simple string message");

// Consumer
@KafkaListener(topics = "topic")
public void consume(String message) {
    log.info("Received: {}", message);
}
```

### JSON Messages
```java
// DTO
public record OrderEvent(
    String orderId,
    String customerId,
    BigDecimal amount,
    LocalDateTime timestamp
) {}

// Producer
kafkaTemplate.send("orders", order.orderId(), orderEvent);

// Consumer
@KafkaListener(topics = "orders")
public void consume(OrderEvent event) {
    log.info("Received order: {}", event.orderId());
}
```

## Headers

### Sending with Headers
```java
Message<OrderEvent> message = MessageBuilder
    .withPayload(orderEvent)
    .setHeader(KafkaHeaders.TOPIC, "orders")
    .setHeader(KafkaHeaders.KEY, orderId)
    .setHeader("X-Correlation-Id", correlationId)
    .setHeader("X-Event-Type", "ORDER_CREATED")
    .build();

kafkaTemplate.send(message);
```

### Reading Headers
```java
@KafkaListener(topics = "orders")
public void consume(
        @Payload OrderEvent event,
        @Header(KafkaHeaders.RECEIVED_KEY) String key,
        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
        @Header(KafkaHeaders.OFFSET) long offset,
        @Header("X-Correlation-Id") String correlationId) {

    log.info("Key: {}, Partition: {}, Offset: {}", key, partition, offset);
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Configure proper serializers | Use default Java serialization |
| Set appropriate acks and retries | Ignore delivery guarantees |
| Use consumer groups for scaling | Create single consumer bottleneck |
| Configure DLQ for failed messages | Lose failed messages |
| Use idempotent producers | Risk duplicate messages |
| Monitor consumer lag | Ignore backlog buildup |

## Monitoring

### Actuator Integration
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,kafka
  health:
    kafka:
      enabled: true
```

### Key Metrics
- `kafka.consumer.records.consumed.total` - Total records consumed
- `kafka.consumer.records.lag` - Consumer lag
- `kafka.producer.record.send.rate` - Producer send rate
- `kafka.producer.record.error.rate` - Producer error rate
