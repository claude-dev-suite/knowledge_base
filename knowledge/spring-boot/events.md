# Spring Events Reference

## Custom Event

```java
public class OrderCreatedEvent extends ApplicationEvent {
    private final Order order;

    public OrderCreatedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }

    public Order getOrder() {
        return order;
    }
}

// Or use record (simpler)
public record OrderCreatedEvent(Order order) { }
```

## Publishing Events

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(new Order(request));

        eventPublisher.publishEvent(new OrderCreatedEvent(order));

        return order;
    }
}
```

## Event Listeners

```java
@Component
public class OrderEventListener {

    // Basic listener
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.info("Order created: {}", event.order().getId());
    }

    // Conditional listener
    @EventListener(condition = "#event.order.total > 1000")
    public void handleLargeOrder(OrderCreatedEvent event) {
        notifyManager(event.order());
    }

    // Return value publishes new event
    @EventListener
    public NotificationSentEvent onOrderCreated(OrderCreatedEvent event) {
        sendNotification(event.order());
        return new NotificationSentEvent(event.order().getId());
    }
}
```

## Async Event Listeners

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean
    public Executor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-event-");
        return executor;
    }
}

@Component
public class AsyncEventListener {

    @Async
    @EventListener
    public void handleOrderAsync(OrderCreatedEvent event) {
        // Runs in separate thread
        sendEmail(event.order());
    }
}
```

## Transaction-Bound Events

```java
@Component
public class TransactionalEventListener {

    // After transaction commits successfully
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void afterCommit(OrderCreatedEvent event) {
        sendNotification(event.order());
    }

    // After transaction rolls back
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void afterRollback(OrderCreatedEvent event) {
        logFailure(event.order());
    }

    // Before transaction commits
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void beforeCommit(OrderCreatedEvent event) {
        validateOrder(event.order());
    }

    // After completion (commit or rollback)
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
    public void afterCompletion(OrderCreatedEvent event) {
        cleanup(event.order());
    }
}
```

## Event Ordering

```java
@Component
public class OrderedEventListeners {

    @EventListener
    @Order(1)
    public void firstHandler(OrderCreatedEvent event) {
        log.info("First handler");
    }

    @EventListener
    @Order(2)
    public void secondHandler(OrderCreatedEvent event) {
        log.info("Second handler");
    }

    @EventListener
    @Order(Ordered.LOWEST_PRECEDENCE)
    public void lastHandler(OrderCreatedEvent event) {
        log.info("Last handler");
    }
}
```

## Multiple Event Types

```java
@Component
public class MultiEventListener {

    @EventListener({OrderCreatedEvent.class, OrderUpdatedEvent.class})
    public void handleOrderEvents(Object event) {
        if (event instanceof OrderCreatedEvent created) {
            handleCreated(created);
        } else if (event instanceof OrderUpdatedEvent updated) {
            handleUpdated(updated);
        }
    }
}
```

## Generic Events

```java
public class EntityCreatedEvent<T> {
    private final T entity;

    public EntityCreatedEvent(T entity) {
        this.entity = entity;
    }

    public T getEntity() {
        return entity;
    }
}

@Component
public class GenericEventListener {

    @EventListener
    public void handleUserCreated(EntityCreatedEvent<User> event) {
        log.info("User created: {}", event.getEntity().getName());
    }

    @EventListener
    public void handleProductCreated(EntityCreatedEvent<Product> event) {
        log.info("Product created: {}", event.getEntity().getName());
    }
}
```

## Built-in Events

```java
@Component
public class BuiltInEventListener {

    @EventListener
    public void onApplicationReady(ApplicationReadyEvent event) {
        log.info("Application started");
    }

    @EventListener
    public void onContextRefreshed(ContextRefreshedEvent event) {
        log.info("Context refreshed");
    }

    @EventListener
    public void onContextClosed(ContextClosedEvent event) {
        log.info("Context closed");
    }
}
```

## Testing Events

```java
@SpringBootTest
class OrderEventTest {

    @Autowired
    private OrderService orderService;

    @MockBean
    private OrderEventListener eventListener;

    @Test
    void shouldPublishEvent() {
        orderService.createOrder(new CreateOrderRequest(...));

        verify(eventListener).handleOrderCreated(any(OrderCreatedEvent.class));
    }
}

// Capture events
@SpringBootTest
class EventCaptureTest {

    @Autowired
    private ApplicationEventPublisher publisher;

    @Captor
    private ArgumentCaptor<OrderCreatedEvent> eventCaptor;

    @Test
    void shouldCaptureEvent() {
        // Test implementation
    }
}
```

## Event Store Pattern

```java
@Entity
public class DomainEvent {
    @Id
    private UUID id;
    private String eventType;
    private String payload;
    private Instant occurredAt;
    private boolean processed;
}

@Service
@RequiredArgsConstructor
public class EventStoreService {

    private final DomainEventRepository repository;
    private final ObjectMapper objectMapper;

    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void storeEvent(Object event) {
        DomainEvent domainEvent = new DomainEvent();
        domainEvent.setId(UUID.randomUUID());
        domainEvent.setEventType(event.getClass().getName());
        domainEvent.setPayload(objectMapper.writeValueAsString(event));
        domainEvent.setOccurredAt(Instant.now());
        repository.save(domainEvent);
    }
}
```
