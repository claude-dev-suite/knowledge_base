# Spring AMQP - Consumer

## @RabbitListener

The `@RabbitListener` annotation enables declarative consumption of messages from RabbitMQ queues.

### Basic Consumer

```java
@Service
@Slf4j
public class OrderConsumer {

    @RabbitListener(queues = "orders.queue")
    public void consume(OrderEvent event) {
        log.info("Received order: {}", event.orderId());
        processOrder(event);
    }
}
```

### Consumer with Metadata

```java
@RabbitListener(queues = "orders.queue")
public void consume(
        @Payload OrderEvent event,
        @Header(AmqpHeaders.RECEIVED_ROUTING_KEY) String routingKey,
        @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag,
        @Header(AmqpHeaders.REDELIVERED) boolean redelivered,
        @Header(name = "X-Correlation-Id", required = false) String correlationId,
        Channel channel) throws IOException {

    log.info("Processing order {} from routing key {}", event.orderId(), routingKey);

    try {
        orderService.process(event);
        channel.basicAck(deliveryTag, false);  // Acknowledge
    } catch (Exception e) {
        log.error("Failed to process order", e);
        channel.basicNack(deliveryTag, false, !redelivered);  // Reject, requeue if first attempt
    }
}
```

### Message Object

```java
@RabbitListener(queues = "orders.queue")
public void consume(Message message) {
    String body = new String(message.getBody());
    MessageProperties props = message.getMessageProperties();

    log.info("Received message: correlationId={}, contentType={}",
        props.getCorrelationId(), props.getContentType());

    props.getHeaders().forEach((key, value) ->
        log.info("Header: {} = {}", key, value));
}
```

## Queue Declaration

### Inline Queue Declaration

```java
@RabbitListener(queuesToDeclare = @Queue(
    name = "notifications.queue",
    durable = "true",
    arguments = {
        @Argument(name = "x-message-ttl", value = "86400000", type = "java.lang.Integer"),
        @Argument(name = "x-dead-letter-exchange", value = "notifications.dlx")
    }
))
public void consume(NotificationEvent event) {
    sendNotification(event);
}
```

### Full Declaration with Binding

```java
@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "orders.queue", durable = "true"),
    exchange = @Exchange(name = "orders.exchange", type = ExchangeTypes.TOPIC),
    key = "orders.#"
))
public void consume(OrderEvent event) {
    processOrder(event);
}
```

## Concurrency

### Configuration

```java
@Configuration
public class RabbitListenerConfig {

    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
            ConnectionFactory connectionFactory) {

        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setConcurrentConsumers(3);
        factory.setMaxConcurrentConsumers(10);
        factory.setPrefetchCount(10);
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        return factory;
    }
}
```

### Per-Listener Concurrency

```java
@RabbitListener(
    queues = "high-volume.queue",
    concurrency = "5-10"  // 5 min, 10 max
)
public void consume(HighVolumeEvent event) {
    process(event);
}
```

## Manual Acknowledgment

### Configuration

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: manual
```

### Acknowledgment Patterns

```java
@RabbitListener(queues = "orders.queue")
public void consume(OrderEvent event, Channel channel,
                   @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {

    try {
        orderService.process(event);
        channel.basicAck(deliveryTag, false);  // Single message ack

    } catch (RetryableException e) {
        // Requeue for retry
        channel.basicNack(deliveryTag, false, true);

    } catch (NonRetryableException e) {
        // Send to DLQ (no requeue)
        channel.basicNack(deliveryTag, false, false);

    } catch (Exception e) {
        // Reject without requeue
        channel.basicReject(deliveryTag, false);
    }
}
```

### Batch Acknowledgment

```java
@RabbitListener(
    queues = "orders.queue",
    containerFactory = "batchListenerFactory"
)
public void consumeBatch(List<Message> messages, Channel channel) throws IOException {
    long lastTag = 0;

    for (Message message : messages) {
        try {
            processMessage(message);
            lastTag = message.getMessageProperties().getDeliveryTag();
        } catch (Exception e) {
            channel.basicNack(lastTag, true, false);  // Nack all up to this point
            return;
        }
    }

    channel.basicAck(lastTag, true);  // Ack all
}
```

## Reply Pattern

### Direct Reply

```java
@RabbitListener(queues = "rpc.orders.queue")
public OrderResponse processOrder(OrderRequest request) {
    // Return value is sent to reply-to address
    return orderService.process(request);
}
```

### SendTo Reply

```java
@RabbitListener(queues = "orders.queue")
@SendTo("orders.processed.queue")
public OrderProcessedEvent process(OrderEvent event) {
    Order order = orderService.process(event);
    return new OrderProcessedEvent(order.getId(), "PROCESSED");
}

// Conditional reply
@RabbitListener(queues = "orders.queue")
@SendTo("#{null}")  // Dynamic destination
public Message process(OrderEvent event, Channel channel) {
    OrderResult result = orderService.process(event);

    String replyTo = result.isSuccess()
        ? "orders.success.queue"
        : "orders.failure.queue";

    return MessageBuilder.withBody(toBytes(result))
        .setHeader(AmqpHeaders.RECEIVED_ROUTING_KEY, replyTo)
        .build();
}
```

## Container Lifecycle

```java
@Service
@RequiredArgsConstructor
public class ListenerControlService {

    private final RabbitListenerEndpointRegistry registry;

    public void pauseListener(String listenerId) {
        MessageListenerContainer container = registry.getListenerContainer(listenerId);
        if (container != null) {
            container.stop();
        }
    }

    public void resumeListener(String listenerId) {
        MessageListenerContainer container = registry.getListenerContainer(listenerId);
        if (container != null) {
            container.start();
        }
    }

    public boolean isRunning(String listenerId) {
        MessageListenerContainer container = registry.getListenerContainer(listenerId);
        return container != null && container.isRunning();
    }
}
```

## Error Handling

### ErrorHandler

```java
@Configuration
public class RabbitErrorConfig {

    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
            ConnectionFactory connectionFactory) {

        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setErrorHandler(throwable -> {
            log.error("Error in listener", throwable);
            if (throwable.getCause() instanceof NonRetryableException) {
                throw new AmqpRejectAndDontRequeueException("Non-retryable", throwable);
            }
            throw new AmqpException("Retryable error", throwable);
        });
        return factory;
    }
}
```

### @RabbitListener with Retry

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        retry:
          enabled: true
          initial-interval: 1000
          max-attempts: 3
          multiplier: 2.0
          max-interval: 10000
```

### Custom Retry

```java
@Bean
public SimpleRabbitListenerContainerFactory retryContainerFactory(
        ConnectionFactory connectionFactory) {

    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);

    factory.setAdviceChain(
        RetryInterceptorBuilder.stateless()
            .maxAttempts(3)
            .backOffOptions(1000, 2.0, 10000)
            .recoverer(new RejectAndDontRequeueRecoverer())
            .build()
    );

    return factory;
}
```

## Listener Configuration

```yaml
spring:
  rabbitmq:
    listener:
      type: simple  # or direct
      simple:
        acknowledge-mode: manual
        concurrency: 3
        max-concurrency: 10
        prefetch: 10
        default-requeue-rejected: false
        missing-queues-fatal: false
        retry:
          enabled: true
          initial-interval: 1000
          max-attempts: 3
          multiplier: 2.0
```

## Best Practices

| Do | Don't |
|----|-------|
| Use manual acknowledgment for critical messages | Auto-ack important messages |
| Configure appropriate prefetch | Overwhelm consumers |
| Handle errors explicitly | Let exceptions propagate |
| Use DLQ for failed messages | Lose unprocessable messages |
| Set proper concurrency limits | Starve or overwhelm consumers |
| Monitor queue depths | Ignore backlog buildup |
