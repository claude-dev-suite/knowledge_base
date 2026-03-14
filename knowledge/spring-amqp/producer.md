# Spring AMQP - Producer

## RabbitTemplate

The `RabbitTemplate` is the main class for sending messages to RabbitMQ.

### Basic Sending

```java
@Service
@RequiredArgsConstructor
public class MessageProducer {

    private final RabbitTemplate rabbitTemplate;

    // Simple send
    public void send(String exchange, String routingKey, Object message) {
        rabbitTemplate.convertAndSend(exchange, routingKey, message);
    }

    // Send to default exchange
    public void sendToQueue(String queueName, Object message) {
        rabbitTemplate.convertAndSend(queueName, message);
    }

    // Send with post-processor
    public void sendWithHeaders(String exchange, String routingKey, Object message) {
        rabbitTemplate.convertAndSend(exchange, routingKey, message, msg -> {
            msg.getMessageProperties().setHeader("X-Correlation-Id", UUID.randomUUID().toString());
            msg.getMessageProperties().setHeader("X-Event-Type", "ORDER_CREATED");
            msg.getMessageProperties().setPriority(5);
            msg.getMessageProperties().setExpiration("60000");  // 60 seconds TTL
            return msg;
        });
    }
}
```

### Synchronous Send with Confirmation

```java
@Configuration
public class RabbitConfirmConfig {

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setConfirmCallback((correlationData, ack, cause) -> {
            if (ack) {
                log.info("Message confirmed: {}", correlationData);
            } else {
                log.error("Message not confirmed: {}, cause: {}", correlationData, cause);
            }
        });
        template.setReturnsCallback(returned -> {
            log.error("Message returned: {}, replyCode: {}, replyText: {}",
                returned.getMessage(), returned.getReplyCode(), returned.getReplyText());
        });
        return template;
    }
}
```

### Send and Receive (RPC Pattern)

```java
@Service
public class RpcClient {

    private final RabbitTemplate rabbitTemplate;

    public OrderResponse processOrderSync(OrderRequest request) {
        return rabbitTemplate.convertSendAndReceiveAsType(
            "rpc.exchange",
            "rpc.order",
            request,
            message -> {
                message.getMessageProperties().setReplyTo("rpc.reply.queue");
                message.getMessageProperties().setCorrelationId(UUID.randomUUID().toString());
                return message;
            },
            new ParameterizedTypeReference<OrderResponse>() {}
        );
    }
}
```

## Message Building

### MessageBuilder

```java
public void sendCustomMessage(OrderEvent event) {
    MessageProperties props = new MessageProperties();
    props.setContentType(MessageProperties.CONTENT_TYPE_JSON);
    props.setHeader("X-Event-Type", "ORDER_CREATED");
    props.setDeliveryMode(MessageDeliveryMode.PERSISTENT);
    props.setPriority(5);
    props.setCorrelationId(UUID.randomUUID().toString());

    ObjectMapper mapper = new ObjectMapper();
    byte[] body = mapper.writeValueAsBytes(event);

    Message message = MessageBuilder.withBody(body)
        .andProperties(props)
        .build();

    rabbitTemplate.send("orders.exchange", "orders.created", message);
}
```

### Message Post Processor

```java
@Bean
public MessagePostProcessor timestampPostProcessor() {
    return message -> {
        message.getMessageProperties().setTimestamp(new Date());
        message.getMessageProperties().setHeader("X-Sender", "order-service");
        return message;
    };
}

// Usage
rabbitTemplate.convertAndSend(exchange, routingKey, event, timestampPostProcessor());
```

## Publisher Confirms

### Configuration

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated
    publisher-returns: true
```

### Correlation Data

```java
@Service
@RequiredArgsConstructor
public class ConfirmedProducer {

    private final RabbitTemplate rabbitTemplate;

    public void sendWithConfirmation(OrderEvent event) {
        CorrelationData correlationData = new CorrelationData(event.orderId());

        rabbitTemplate.convertAndSend(
            "orders.exchange",
            "orders.created",
            event,
            correlationData
        );

        // Wait for confirmation (async alternative preferred)
        try {
            CorrelationData.Confirm confirm = correlationData.getFuture().get(5, TimeUnit.SECONDS);
            if (confirm.isAck()) {
                log.info("Message confirmed for order: {}", event.orderId());
            } else {
                log.error("Message not confirmed: {}", confirm.getReason());
            }
        } catch (Exception e) {
            log.error("Confirmation failed", e);
        }
    }
}
```

### Async Confirmation Handler

```java
@Configuration
public class AsyncConfirmConfig {

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);

        template.setConfirmCallback((correlationData, ack, cause) -> {
            if (correlationData != null) {
                if (ack) {
                    log.info("Confirmed: {}", correlationData.getId());
                } else {
                    log.error("Nacked: {}, cause: {}", correlationData.getId(), cause);
                    // Retry or handle failure
                    handleNack(correlationData);
                }
            }
        });

        template.setReturnsCallback(returned -> {
            log.error("Message returned: exchange={}, routingKey={}, replyCode={}, replyText={}",
                returned.getExchange(), returned.getRoutingKey(),
                returned.getReplyCode(), returned.getReplyText());
        });

        template.setMandatory(true);  // Enable returns for unroutable messages

        return template;
    }
}
```

## Batch Sending

```java
@Service
public class BatchProducer {

    private final RabbitTemplate rabbitTemplate;

    public void sendBatch(List<OrderEvent> events) {
        rabbitTemplate.invoke(operations -> {
            for (OrderEvent event : events) {
                operations.convertAndSend("orders.exchange", "orders.created", event);
            }
            return null;
        });
    }

    // With confirmation
    public void sendBatchWithConfirmation(List<OrderEvent> events) {
        rabbitTemplate.invoke(operations -> {
            for (OrderEvent event : events) {
                CorrelationData data = new CorrelationData(event.orderId());
                operations.convertAndSend("orders.exchange", "orders.created", event, data);
            }
            operations.waitForConfirmsOrDie(10_000);  // Wait up to 10 seconds
            return null;
        });
    }
}
```

## Scheduled Publishing

```java
@Service
@RequiredArgsConstructor
public class ScheduledProducer {

    private final RabbitTemplate rabbitTemplate;

    @Scheduled(fixedRate = 60000)  // Every minute
    public void publishHeartbeat() {
        HeartbeatEvent event = new HeartbeatEvent(
            "order-service",
            Instant.now(),
            "healthy"
        );
        rabbitTemplate.convertAndSend("monitoring.exchange", "heartbeat", event);
    }

    @Scheduled(cron = "0 0 * * * *")  // Every hour
    public void publishMetrics() {
        MetricsEvent metrics = collectMetrics();
        rabbitTemplate.convertAndSend("monitoring.exchange", "metrics", metrics);
    }
}
```

## Transactional Publishing

```java
@Configuration
public class TransactionalRabbitConfig {

    @Bean
    public RabbitTemplate transactionalRabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setChannelTransacted(true);
        return template;
    }
}

@Service
@Transactional
public class TransactionalOrderService {

    private final OrderRepository orderRepository;
    private final RabbitTemplate rabbitTemplate;

    @Transactional
    public void createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(toEntity(request));

        // Published only if transaction commits
        rabbitTemplate.convertAndSend(
            "orders.exchange",
            "orders.created",
            toEvent(order)
        );
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use publisher confirms for critical messages | Ignore send failures |
| Set appropriate message TTL | Let messages accumulate |
| Use correlation IDs | Lose message traceability |
| Handle returns for mandatory messages | Ignore unroutable messages |
| Use async confirmations | Block on every send |
| Batch related messages | Send one at a time inefficiently |
