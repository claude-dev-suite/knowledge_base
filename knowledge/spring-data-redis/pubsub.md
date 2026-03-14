# Spring Data Redis - Pub/Sub

## Overview

Redis Pub/Sub allows publishing messages to channels and subscribing to receive messages, enabling real-time messaging patterns in Spring applications.

## Basic Configuration

```java
@Configuration
public class RedisPubSubConfig {

    @Bean
    public RedisMessageListenerContainer redisMessageListenerContainer(
            RedisConnectionFactory connectionFactory,
            MessageListener messageListener) {

        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);

        // Subscribe to channel
        container.addMessageListener(messageListener,
            new ChannelTopic("notifications"));

        // Subscribe to pattern
        container.addMessageListener(messageListener,
            new PatternTopic("events.*"));

        return container;
    }

    @Bean
    public MessageListenerAdapter messageListener(NotificationHandler handler) {
        return new MessageListenerAdapter(handler, "handleMessage");
    }
}
```

## Publishing Messages

### Simple Publisher
```java
@Service
@RequiredArgsConstructor
public class RedisPublisher {

    private final RedisTemplate<String, Object> redisTemplate;

    public void publish(String channel, Object message) {
        redisTemplate.convertAndSend(channel, message);
    }
}
```

### Typed Publisher
```java
@Service
@RequiredArgsConstructor
public class NotificationPublisher {

    private final RedisTemplate<String, Object> redisTemplate;
    private final ObjectMapper objectMapper;

    public void publishNotification(String userId, Notification notification) {
        String channel = "notifications:" + userId;
        redisTemplate.convertAndSend(channel, notification);
    }

    public void publishEvent(String eventType, Object payload) {
        String channel = "events." + eventType;
        redisTemplate.convertAndSend(channel, payload);
    }

    public void broadcast(String message) {
        redisTemplate.convertAndSend("broadcast", message);
    }
}
```

## Message Listeners

### Basic Listener
```java
@Component
@Slf4j
public class NotificationHandler implements MessageListener {

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String channel = new String(message.getChannel());
        String body = new String(message.getBody());

        log.info("Received message on channel {}: {}", channel, body);
    }
}
```

### Object-based Listener
```java
@Component
@Slf4j
public class NotificationHandler {

    // Method name must match MessageListenerAdapter configuration
    public void handleMessage(Notification notification, String channel) {
        log.info("Received notification on {}: {}", channel, notification);
        processNotification(notification);
    }

    private void processNotification(Notification notification) {
        // Process the notification
    }
}
```

### JSON Deserialization Listener
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class JsonMessageListener implements MessageListener {

    private final ObjectMapper objectMapper;

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String channel = new String(message.getChannel());

        try {
            if (channel.startsWith("orders.")) {
                OrderEvent event = objectMapper.readValue(
                    message.getBody(), OrderEvent.class);
                handleOrderEvent(event);
            } else if (channel.startsWith("notifications.")) {
                Notification notification = objectMapper.readValue(
                    message.getBody(), Notification.class);
                handleNotification(notification);
            }
        } catch (Exception e) {
            log.error("Failed to process message on {}", channel, e);
        }
    }

    private void handleOrderEvent(OrderEvent event) {
        log.info("Processing order event: {}", event);
    }

    private void handleNotification(Notification notification) {
        log.info("Processing notification: {}", notification);
    }
}
```

## Dynamic Subscriptions

```java
@Service
@RequiredArgsConstructor
public class DynamicSubscriptionService {

    private final RedisMessageListenerContainer container;
    private final MessageListenerAdapter listenerAdapter;
    private final Map<String, ChannelTopic> activeSubscriptions = new ConcurrentHashMap<>();

    public void subscribe(String channel) {
        if (!activeSubscriptions.containsKey(channel)) {
            ChannelTopic topic = new ChannelTopic(channel);
            container.addMessageListener(listenerAdapter, topic);
            activeSubscriptions.put(channel, topic);
        }
    }

    public void unsubscribe(String channel) {
        ChannelTopic topic = activeSubscriptions.remove(channel);
        if (topic != null) {
            container.removeMessageListener(listenerAdapter, topic);
        }
    }

    public Set<String> getActiveSubscriptions() {
        return activeSubscriptions.keySet();
    }
}
```

## User-Specific Channels

```java
@Configuration
public class UserChannelConfig {

    @Bean
    public RedisMessageListenerContainer container(
            RedisConnectionFactory factory,
            UserMessageHandler handler) {

        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);

        // User-specific channels via pattern
        container.addMessageListener(
            new MessageListenerAdapter(handler),
            new PatternTopic("user.*")
        );

        return container;
    }
}

@Component
@Slf4j
public class UserMessageHandler implements MessageListener {

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String channel = new String(message.getChannel());
        String userId = extractUserId(channel);
        String body = new String(message.getBody());

        log.info("Message for user {}: {}", userId, body);
        deliverToUser(userId, body);
    }

    private String extractUserId(String channel) {
        return channel.replace("user.", "");
    }

    private void deliverToUser(String userId, String message) {
        // Deliver via WebSocket, SSE, etc.
    }
}

@Service
@RequiredArgsConstructor
public class UserNotificationService {

    private final RedisTemplate<String, Object> redisTemplate;

    public void notifyUser(String userId, Object message) {
        redisTemplate.convertAndSend("user." + userId, message);
    }

    public void broadcastToAll(Object message) {
        redisTemplate.convertAndSend("broadcast", message);
    }
}
```

## Reactive Pub/Sub

```java
@Configuration
public class ReactiveRedisPubSubConfig {

    @Bean
    public ReactiveRedisMessageListenerContainer reactiveContainer(
            ReactiveRedisConnectionFactory factory) {
        return new ReactiveRedisMessageListenerContainer(factory);
    }
}

@Service
@RequiredArgsConstructor
public class ReactiveSubscriber {

    private final ReactiveRedisMessageListenerContainer container;
    private final ObjectMapper objectMapper;

    public Flux<Notification> subscribeToNotifications() {
        return container.receive(ChannelTopic.of("notifications"))
            .map(message -> {
                try {
                    return objectMapper.readValue(
                        message.getMessage(), Notification.class);
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            });
    }

    public Flux<String> subscribeToPattern(String pattern) {
        return container.receive(PatternTopic.of(pattern))
            .map(message -> new String(message.getMessage()));
    }

    public Mono<Long> publish(String channel, String message) {
        return container.getConnectionFactory()
            .getReactiveConnection()
            .pubSubCommands()
            .publish(ByteBuffer.wrap(channel.getBytes()),
                    ByteBuffer.wrap(message.getBytes()));
    }
}
```

## WebSocket Integration

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(redisWebSocketHandler(), "/ws/notifications")
            .setAllowedOrigins("*");
    }

    @Bean
    public WebSocketHandler redisWebSocketHandler() {
        return new RedisWebSocketHandler();
    }
}

@Component
@RequiredArgsConstructor
@Slf4j
public class RedisWebSocketHandler extends TextWebSocketHandler {

    private final RedisMessageListenerContainer container;
    private final Map<String, WebSocketSession> sessions = new ConcurrentHashMap<>();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        String userId = extractUserId(session);
        sessions.put(userId, session);

        // Subscribe to user-specific channel
        container.addMessageListener(
            new SessionMessageListener(session),
            new ChannelTopic("user." + userId)
        );
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        String userId = extractUserId(session);
        sessions.remove(userId);
    }

    private String extractUserId(WebSocketSession session) {
        return session.getUri().getQuery().split("=")[1];
    }

    private static class SessionMessageListener implements MessageListener {
        private final WebSocketSession session;

        SessionMessageListener(WebSocketSession session) {
            this.session = session;
        }

        @Override
        public void onMessage(Message message, byte[] pattern) {
            try {
                if (session.isOpen()) {
                    session.sendMessage(
                        new TextMessage(new String(message.getBody())));
                }
            } catch (IOException e) {
                // Handle error
            }
        }
    }
}
```

## Event-Driven Architecture

```java
// Domain events
public record OrderCreatedEvent(String orderId, String customerId, BigDecimal total) {}
public record OrderShippedEvent(String orderId, String trackingNumber) {}

// Event publisher
@Service
@RequiredArgsConstructor
public class DomainEventPublisher {

    private final RedisTemplate<String, Object> redisTemplate;
    private final ObjectMapper objectMapper;

    public void publish(Object event) {
        String channel = "events." + event.getClass().getSimpleName();
        redisTemplate.convertAndSend(channel, event);
    }
}

// Event handlers configuration
@Configuration
public class EventHandlerConfig {

    @Bean
    public RedisMessageListenerContainer eventContainer(
            RedisConnectionFactory factory,
            OrderEventHandler orderHandler,
            InventoryEventHandler inventoryHandler) {

        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);

        container.addMessageListener(
            new MessageListenerAdapter(orderHandler, "handle"),
            new PatternTopic("events.Order*"));

        container.addMessageListener(
            new MessageListenerAdapter(inventoryHandler, "handle"),
            new PatternTopic("events.Inventory*"));

        return container;
    }
}

// Event handlers
@Component
@Slf4j
public class OrderEventHandler {

    public void handle(OrderCreatedEvent event, String channel) {
        log.info("Order created: {}", event.orderId());
        // Process order created event
    }

    public void handle(OrderShippedEvent event, String channel) {
        log.info("Order shipped: {} with tracking {}",
            event.orderId(), event.trackingNumber());
        // Process order shipped event
    }
}
```

## Error Handling

```java
@Configuration
public class PubSubErrorConfig {

    @Bean
    public RedisMessageListenerContainer container(RedisConnectionFactory factory) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);

        // Custom error handler
        container.setErrorHandler(t -> {
            log.error("Redis Pub/Sub error", t);
            // Custom error handling logic
        });

        // Recovery interval after connection failure
        container.setRecoveryInterval(5000);

        return container;
    }
}

@Component
@Slf4j
public class ResilientMessageListener implements MessageListener {

    @Override
    public void onMessage(Message message, byte[] pattern) {
        try {
            processMessage(message);
        } catch (Exception e) {
            log.error("Failed to process message, sending to DLQ", e);
            sendToDeadLetterQueue(message, e);
        }
    }

    private void processMessage(Message message) {
        // Process message
    }

    private void sendToDeadLetterQueue(Message message, Exception error) {
        // Store failed message for retry
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use pattern subscriptions wisely | Subscribe to broad patterns |
| Handle connection failures | Assume connection is stable |
| Process messages async | Block in listener |
| Use appropriate serialization | Serialize complex objects as strings |
| Implement error handling | Ignore message failures |
| Monitor subscription count | Let subscriptions grow unbounded |

## Production Checklist

- [ ] Connection recovery configured
- [ ] Error handling implemented
- [ ] Dead letter queue for failed messages
- [ ] Monitoring for channel activity
- [ ] Proper serialization configured
- [ ] Subscription lifecycle managed
- [ ] Resource cleanup on shutdown
