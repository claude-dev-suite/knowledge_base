# Spring AMQP - Basics

## Overview

Spring AMQP provides a high-level abstraction for AMQP-based messaging with RabbitMQ. It simplifies sending and receiving messages through templates and annotations.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

## Configuration

### application.yml
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /

    # Connection settings
    connection-timeout: 5000
    requested-heartbeat: 60

    # Listener settings
    listener:
      simple:
        acknowledge-mode: manual
        concurrency: 3
        max-concurrency: 10
        prefetch: 10
        retry:
          enabled: true
          initial-interval: 1000
          max-attempts: 3
          multiplier: 2.0
```

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         RabbitMQ Broker                             │
│                                                                     │
│  ┌──────────┐     ┌──────────────┐     ┌────────────────────────┐  │
│  │ Exchange │────▶│   Bindings   │────▶│        Queue           │  │
│  └──────────┘     └──────────────┘     └────────────────────────┘  │
│       │                                          │                  │
│       │  Routing Key                             │                  │
│       │                                          ▼                  │
│  ┌──────────┐                           ┌────────────────┐         │
│  │ Producer │                           │    Consumer    │         │
│  └──────────┘                           └────────────────┘         │
└─────────────────────────────────────────────────────────────────────┘
```

## Exchange Types

| Type | Routing |
|------|---------|
| Direct | Routes by exact routing key match |
| Topic | Routes by pattern matching (*.logs, #.error) |
| Fanout | Broadcasts to all bound queues |
| Headers | Routes based on message headers |

## Basic Setup

### Exchange and Queue Configuration

```java
@Configuration
public class RabbitMQConfig {

    public static final String ORDERS_EXCHANGE = "orders.exchange";
    public static final String ORDERS_QUEUE = "orders.queue";
    public static final String ORDERS_ROUTING_KEY = "orders.created";

    @Bean
    public DirectExchange ordersExchange() {
        return ExchangeBuilder.directExchange(ORDERS_EXCHANGE)
            .durable(true)
            .build();
    }

    @Bean
    public Queue ordersQueue() {
        return QueueBuilder.durable(ORDERS_QUEUE)
            .withArgument("x-dead-letter-exchange", "orders.dlx")
            .withArgument("x-dead-letter-routing-key", "orders.dlq")
            .withArgument("x-message-ttl", 86400000)  // 24 hours
            .build();
    }

    @Bean
    public Binding ordersBinding(Queue ordersQueue, DirectExchange ordersExchange) {
        return BindingBuilder.bind(ordersQueue)
            .to(ordersExchange)
            .with(ORDERS_ROUTING_KEY);
    }

    // DLQ setup
    @Bean
    public DirectExchange dlxExchange() {
        return new DirectExchange("orders.dlx");
    }

    @Bean
    public Queue dlqQueue() {
        return QueueBuilder.durable("orders.dlq").build();
    }

    @Bean
    public Binding dlqBinding() {
        return BindingBuilder.bind(dlqQueue())
            .to(dlxExchange())
            .with("orders.dlq");
    }
}
```

## Message Converter

### JSON Converter

```java
@Configuration
public class RabbitMQMessageConfig {

    @Bean
    public MessageConverter jsonMessageConverter() {
        Jackson2JsonMessageConverter converter = new Jackson2JsonMessageConverter();
        converter.setClassMapper(classMapper());
        return converter;
    }

    @Bean
    public DefaultClassMapper classMapper() {
        DefaultClassMapper classMapper = new DefaultClassMapper();
        classMapper.setTrustedPackages("com.example.events");
        return classMapper;
    }

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory,
                                         MessageConverter messageConverter) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(messageConverter);
        template.setExchange(RabbitMQConfig.ORDERS_EXCHANGE);
        return template;
    }
}
```

## Basic Producer

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderProducer {

    private final RabbitTemplate rabbitTemplate;

    public void sendOrder(OrderEvent event) {
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.ORDERS_EXCHANGE,
            RabbitMQConfig.ORDERS_ROUTING_KEY,
            event
        );
        log.info("Sent order event: {}", event.orderId());
    }
}
```

## Basic Consumer

```java
@Service
@Slf4j
public class OrderConsumer {

    @RabbitListener(queues = RabbitMQConfig.ORDERS_QUEUE)
    public void consume(OrderEvent event) {
        log.info("Received order: {}", event.orderId());
        processOrder(event);
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use durable exchanges/queues | Use transient for important data |
| Configure DLQ | Lose failed messages |
| Use manual acknowledgment | Auto-ack critical messages |
| Set message TTL | Keep messages forever |
| Configure prefetch count | Overwhelm consumers |
| Use JSON for messages | Use Java serialization |

## Production Checklist

- [ ] Exchanges and queues are durable
- [ ] Dead letter queues configured
- [ ] Message TTL set appropriately
- [ ] Connection retry enabled
- [ ] Heartbeat configured
- [ ] Prefetch count tuned
- [ ] Consumer concurrency appropriate
- [ ] Monitoring enabled
