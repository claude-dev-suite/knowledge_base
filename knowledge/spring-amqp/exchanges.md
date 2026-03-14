# Spring AMQP - Exchange Types and Patterns

## Exchange Types Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Exchange Types                                │
├─────────────────────────────────────────────────────────────────────┤
│  Direct     │ Exact routing key match                              │
│  Topic      │ Pattern-based routing (*, #)                         │
│  Fanout     │ Broadcast to all bound queues                        │
│  Headers    │ Route based on message headers                       │
└─────────────────────────────────────────────────────────────────────┘
```

## Direct Exchange

Routes messages to queues based on exact routing key match.

```java
@Configuration
public class DirectExchangeConfig {

    @Bean
    public DirectExchange ordersExchange() {
        return ExchangeBuilder.directExchange("orders.direct")
            .durable(true)
            .build();
    }

    @Bean
    public Queue ordersCreatedQueue() {
        return QueueBuilder.durable("orders.created").build();
    }

    @Bean
    public Queue ordersShippedQueue() {
        return QueueBuilder.durable("orders.shipped").build();
    }

    @Bean
    public Binding createdBinding(Queue ordersCreatedQueue, DirectExchange ordersExchange) {
        return BindingBuilder.bind(ordersCreatedQueue)
            .to(ordersExchange)
            .with("orders.created");
    }

    @Bean
    public Binding shippedBinding(Queue ordersShippedQueue, DirectExchange ordersExchange) {
        return BindingBuilder.bind(ordersShippedQueue)
            .to(ordersExchange)
            .with("orders.shipped");
    }
}

// Producer
public void publish(OrderEvent event, String routingKey) {
    rabbitTemplate.convertAndSend("orders.direct", routingKey, event);
}

// Consumer
@RabbitListener(queues = "orders.created")
public void onOrderCreated(OrderEvent event) {
    processCreatedOrder(event);
}
```

## Topic Exchange

Routes messages based on wildcard pattern matching.

- `*` matches exactly one word
- `#` matches zero or more words

```java
@Configuration
public class TopicExchangeConfig {

    @Bean
    public TopicExchange logsExchange() {
        return ExchangeBuilder.topicExchange("logs.topic")
            .durable(true)
            .build();
    }

    // Queue for all errors
    @Bean
    public Queue allErrorsQueue() {
        return QueueBuilder.durable("logs.all-errors").build();
    }

    @Bean
    public Binding allErrorsBinding(Queue allErrorsQueue, TopicExchange logsExchange) {
        return BindingBuilder.bind(allErrorsQueue)
            .to(logsExchange)
            .with("*.error");  // matches: app.error, service.error
    }

    // Queue for all order logs
    @Bean
    public Queue orderLogsQueue() {
        return QueueBuilder.durable("logs.orders").build();
    }

    @Bean
    public Binding orderLogsBinding(Queue orderLogsQueue, TopicExchange logsExchange) {
        return BindingBuilder.bind(orderLogsQueue)
            .to(logsExchange)
            .with("orders.#");  // matches: orders.created, orders.shipped.completed
    }

    // Queue for critical alerts
    @Bean
    public Queue criticalQueue() {
        return QueueBuilder.durable("logs.critical").build();
    }

    @Bean
    public Binding criticalBinding(Queue criticalQueue, TopicExchange logsExchange) {
        return BindingBuilder.bind(criticalQueue)
            .to(logsExchange)
            .with("#.critical.#");  // matches: app.critical, service.critical.alert
    }
}

// Producer
public void logEvent(String service, String level, LogEvent event) {
    String routingKey = service + "." + level;  // e.g., "orders.error"
    rabbitTemplate.convertAndSend("logs.topic", routingKey, event);
}
```

### Topic Routing Key Patterns

| Pattern | Matches | Does Not Match |
|---------|---------|----------------|
| `*.error` | `app.error`, `order.error` | `app.service.error` |
| `orders.#` | `orders`, `orders.created`, `orders.shipped.completed` | `inventory.orders` |
| `#.critical` | `critical`, `app.critical`, `app.service.critical` | `critical.alert` |
| `*.*.error` | `app.service.error` | `app.error` |

## Fanout Exchange

Broadcasts messages to all bound queues (ignores routing key).

```java
@Configuration
public class FanoutExchangeConfig {

    @Bean
    public FanoutExchange notificationsExchange() {
        return ExchangeBuilder.fanoutExchange("notifications.fanout")
            .durable(true)
            .build();
    }

    @Bean
    public Queue emailNotificationQueue() {
        return QueueBuilder.durable("notifications.email").build();
    }

    @Bean
    public Queue smsNotificationQueue() {
        return QueueBuilder.durable("notifications.sms").build();
    }

    @Bean
    public Queue pushNotificationQueue() {
        return QueueBuilder.durable("notifications.push").build();
    }

    @Bean
    public Binding emailBinding(Queue emailNotificationQueue, FanoutExchange notificationsExchange) {
        return BindingBuilder.bind(emailNotificationQueue).to(notificationsExchange);
    }

    @Bean
    public Binding smsBinding(Queue smsNotificationQueue, FanoutExchange notificationsExchange) {
        return BindingBuilder.bind(smsNotificationQueue).to(notificationsExchange);
    }

    @Bean
    public Binding pushBinding(Queue pushNotificationQueue, FanoutExchange notificationsExchange) {
        return BindingBuilder.bind(pushNotificationQueue).to(notificationsExchange);
    }
}

// Producer - routing key is ignored
public void broadcastNotification(NotificationEvent event) {
    rabbitTemplate.convertAndSend("notifications.fanout", "", event);
}

// Consumers - each receives a copy
@RabbitListener(queues = "notifications.email")
public void sendEmail(NotificationEvent event) { /* ... */ }

@RabbitListener(queues = "notifications.sms")
public void sendSms(NotificationEvent event) { /* ... */ }

@RabbitListener(queues = "notifications.push")
public void sendPush(NotificationEvent event) { /* ... */ }
```

## Headers Exchange

Routes based on message headers instead of routing key.

```java
@Configuration
public class HeadersExchangeConfig {

    @Bean
    public HeadersExchange reportExchange() {
        return ExchangeBuilder.headersExchange("reports.headers")
            .durable(true)
            .build();
    }

    @Bean
    public Queue pdfReportQueue() {
        return QueueBuilder.durable("reports.pdf").build();
    }

    @Bean
    public Queue csvReportQueue() {
        return QueueBuilder.durable("reports.csv").build();
    }

    // Match ALL headers (x-match = all)
    @Bean
    public Binding pdfBinding(Queue pdfReportQueue, HeadersExchange reportExchange) {
        return BindingBuilder.bind(pdfReportQueue)
            .to(reportExchange)
            .whereAll(Map.of(
                "format", "pdf",
                "type", "report"
            )).match();
    }

    // Match ANY header (x-match = any)
    @Bean
    public Binding csvBinding(Queue csvReportQueue, HeadersExchange reportExchange) {
        return BindingBuilder.bind(csvReportQueue)
            .to(reportExchange)
            .whereAny(Map.of(
                "format", "csv",
                "export", "true"
            )).match();
    }
}

// Producer
public void sendReport(ReportEvent event, String format, String type) {
    rabbitTemplate.convertAndSend("reports.headers", "", event, msg -> {
        msg.getMessageProperties().setHeader("format", format);
        msg.getMessageProperties().setHeader("type", type);
        return msg;
    });
}
```

## Exchange-to-Exchange Binding

Chain exchanges for complex routing scenarios.

```java
@Configuration
public class ExchangeChainConfig {

    @Bean
    public TopicExchange mainExchange() {
        return ExchangeBuilder.topicExchange("main.topic").durable(true).build();
    }

    @Bean
    public FanoutExchange broadcastExchange() {
        return ExchangeBuilder.fanoutExchange("broadcast.fanout").durable(true).build();
    }

    // Route critical messages from topic to fanout
    @Bean
    public Binding exchangeBinding(FanoutExchange broadcastExchange, TopicExchange mainExchange) {
        return BindingBuilder.bind(broadcastExchange)
            .to(mainExchange)
            .with("*.critical.#");
    }
}
```

## Alternate Exchange

Handle unroutable messages by routing to an alternate exchange.

```java
@Bean
public DirectExchange mainExchange() {
    return ExchangeBuilder.directExchange("main.direct")
        .durable(true)
        .alternate("unrouted.fanout")  // Alternate exchange
        .build();
}

@Bean
public FanoutExchange unroutedExchange() {
    return ExchangeBuilder.fanoutExchange("unrouted.fanout")
        .durable(true)
        .build();
}

@Bean
public Queue unroutedQueue() {
    return QueueBuilder.durable("unrouted.messages").build();
}

@Bean
public Binding unroutedBinding(Queue unroutedQueue, FanoutExchange unroutedExchange) {
    return BindingBuilder.bind(unroutedQueue).to(unroutedExchange);
}
```

## Delayed Message Exchange

Using the delayed message plugin.

```java
@Bean
public CustomExchange delayedExchange() {
    Map<String, Object> args = new HashMap<>();
    args.put("x-delayed-type", "direct");
    return new CustomExchange("delayed.exchange", "x-delayed-message", true, false, args);
}

// Send with delay
public void sendDelayed(Object message, int delayMs) {
    rabbitTemplate.convertAndSend("delayed.exchange", "delayed.routing", message, msg -> {
        msg.getMessageProperties().setHeader("x-delay", delayMs);
        return msg;
    });
}
```

## Best Practices

| Exchange Type | Use Case |
|--------------|----------|
| Direct | Point-to-point, exact routing |
| Topic | Flexible pattern-based routing |
| Fanout | Broadcast/publish-subscribe |
| Headers | Complex routing by attributes |

| Do | Don't |
|----|-------|
| Use topic for flexibility | Overuse direct exchanges |
| Configure alternate exchanges | Lose unroutable messages |
| Use consistent naming conventions | Mix naming patterns |
| Document routing patterns | Leave routing undocumented |
