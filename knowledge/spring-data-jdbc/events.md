# Spring Data JDBC - Events and Callbacks

## Overview

Spring Data JDBC provides lifecycle callbacks and domain events for reacting to entity persistence operations. Unlike JPA, callbacks are handled through Spring's event mechanism.

## Entity Callbacks

### BeforeConvert Callback
```java
@Component
public class AuditingCallback implements BeforeConvertCallback<BaseEntity> {

    @Override
    public BaseEntity onBeforeConvert(BaseEntity entity) {
        if (entity.getId() == null) {
            entity.setCreatedAt(LocalDateTime.now());
        }
        entity.setUpdatedAt(LocalDateTime.now());
        return entity;
    }
}

// For specific entity
@Component
public class OrderCallback implements BeforeConvertCallback<Order> {

    @Override
    public Order onBeforeConvert(Order order) {
        if (order.getOrderNumber() == null) {
            order.setOrderNumber(generateOrderNumber());
        }
        order.recalculateTotal();
        return order;
    }

    private String generateOrderNumber() {
        return "ORD-" + System.currentTimeMillis();
    }
}
```

### BeforeSave Callback
```java
@Component
public class ValidationCallback implements BeforeSaveCallback<Product> {

    @Override
    public Product onBeforeSave(Product product) {
        if (product.getPrice() == null || product.getPrice().compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Invalid price");
        }
        if (product.getName() == null || product.getName().isBlank()) {
            throw new IllegalArgumentException("Name is required");
        }
        return product;
    }
}
```

### AfterSave Callback
```java
@Component
public class CacheInvalidationCallback implements AfterSaveCallback<Product> {

    private final CacheManager cacheManager;

    public CacheInvalidationCallback(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    @Override
    public Product onAfterSave(Product product) {
        Cache productCache = cacheManager.getCache("products");
        if (productCache != null) {
            productCache.evict(product.getId());
        }
        return product;
    }
}
```

### AfterConvert Callback (Loading)
```java
@Component
public class PostLoadCallback implements AfterConvertCallback<Order> {

    @Override
    public Order onAfterConvert(Order order) {
        // Compute derived fields after loading
        order.computeTotal();
        order.validateState();
        return order;
    }
}
```

### BeforeDelete and AfterDelete
```java
@Component
public class DeleteCallback implements BeforeDeleteCallback<Customer>, AfterDeleteCallback<Customer> {

    private final OrderRepository orderRepository;
    private final AuditLogService auditLog;

    @Override
    public Customer onBeforeDelete(Customer customer) {
        // Check for active orders
        if (orderRepository.existsByCustomerIdAndStatus(customer.getId(), "PENDING")) {
            throw new IllegalStateException("Cannot delete customer with pending orders");
        }
        return customer;
    }

    @Override
    public Customer onAfterDelete(Customer customer) {
        auditLog.logDeletion("Customer", customer.getId());
        return customer;
    }
}
```

## Domain Events

### Publishing Events
```java
@Table("orders")
public class Order extends AbstractAggregateRoot<Order> {

    @Id
    private Long id;
    private String orderNumber;
    private OrderStatus status;

    public void complete() {
        this.status = OrderStatus.COMPLETED;
        registerEvent(new OrderCompletedEvent(this.id, this.orderNumber));
    }

    public void cancel(String reason) {
        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelledEvent(this.id, this.orderNumber, reason));
    }

    public void ship(String trackingNumber) {
        this.status = OrderStatus.SHIPPED;
        registerEvent(new OrderShippedEvent(this.id, this.orderNumber, trackingNumber));
    }
}

// Event classes
public record OrderCompletedEvent(Long orderId, String orderNumber) {}
public record OrderCancelledEvent(Long orderId, String orderNumber, String reason) {}
public record OrderShippedEvent(Long orderId, String orderNumber, String trackingNumber) {}
```

### Handling Events
```java
@Component
@Slf4j
public class OrderEventHandler {

    private final EmailService emailService;
    private final InventoryService inventoryService;
    private final AnalyticsService analyticsService;

    @EventListener
    public void handleOrderCompleted(OrderCompletedEvent event) {
        log.info("Order completed: {}", event.orderNumber());
        emailService.sendOrderConfirmation(event.orderId());
        analyticsService.recordSale(event.orderId());
    }

    @EventListener
    public void handleOrderCancelled(OrderCancelledEvent event) {
        log.info("Order cancelled: {} - Reason: {}", event.orderNumber(), event.reason());
        inventoryService.releaseReservation(event.orderId());
        emailService.sendCancellationNotice(event.orderId(), event.reason());
    }

    @EventListener
    public void handleOrderShipped(OrderShippedEvent event) {
        log.info("Order shipped: {} - Tracking: {}", event.orderNumber(), event.trackingNumber());
        emailService.sendShippingNotification(event.orderId(), event.trackingNumber());
    }
}
```

### Async Event Handling
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
        executor.initialize();
        return executor;
    }
}

@Component
public class AsyncOrderEventHandler {

    @Async
    @EventListener
    public void handleOrderCompletedAsync(OrderCompletedEvent event) {
        // Long-running operations
        generateInvoicePdf(event.orderId());
        sendToExternalSystem(event.orderId());
    }
}
```

### Transactional Event Handling
```java
@Component
public class TransactionalEventHandler {

    // Execute after transaction commits
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleAfterCommit(OrderCompletedEvent event) {
        // Safe to send notifications, external calls
        sendNotification(event);
    }

    // Execute after rollback
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void handleAfterRollback(OrderCompletedEvent event) {
        // Cleanup or logging
        logFailedOrder(event);
    }

    // Execute before commit
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void handleBeforeCommit(OrderCompletedEvent event) {
        // Validation or additional persistence
        validateOrder(event);
    }
}
```

## Custom Event Publishing

```java
// Manual event publishing
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public Order createOrder(CreateOrderCommand command) {
        Order order = new Order();
        // ... set properties

        Order saved = orderRepository.save(order);

        // Publish custom event
        eventPublisher.publishEvent(new OrderCreatedEvent(saved.getId()));

        return saved;
    }
}

// Using aggregate root
@Transactional
public Order completeOrder(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.complete();  // Registers event internally
    return orderRepository.save(order);  // Events published after save
}
```

## Callback Registration

```java
@Configuration
public class CallbackConfig extends AbstractJdbcConfiguration {

    @Override
    protected List<?> userConverters() {
        return List.of(
            new MoneyToStringConverter(),
            new StringToMoneyConverter()
        );
    }

    // Callbacks are auto-discovered via @Component
    // Or register manually:

    @Bean
    public BeforeConvertCallback<BaseEntity> timestampCallback() {
        return entity -> {
            if (entity.getCreatedAt() == null) {
                entity.setCreatedAt(LocalDateTime.now());
            }
            entity.setUpdatedAt(LocalDateTime.now());
            return entity;
        };
    }
}
```

## Complete Callback Types

| Callback | When Called | Use Case |
|----------|-------------|----------|
| BeforeConvertCallback | Before entity conversion to SQL | Set defaults, generate IDs |
| AfterConvertCallback | After loading from database | Compute derived fields |
| BeforeSaveCallback | Before INSERT/UPDATE | Validation, audit fields |
| AfterSaveCallback | After INSERT/UPDATE | Cache invalidation, notifications |
| BeforeDeleteCallback | Before DELETE | Validation, cascading |
| AfterDeleteCallback | After DELETE | Audit logging, cleanup |

## Event-Driven Patterns

### Saga Pattern
```java
@Component
@Slf4j
public class OrderSaga {

    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final OrderRepository orderRepository;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            // Step 1: Reserve inventory
            inventoryService.reserve(event.orderId());

            // Step 2: Process payment
            paymentService.charge(event.orderId());

            // Step 3: Confirm order
            orderRepository.updateStatus(event.orderId(), OrderStatus.CONFIRMED);

        } catch (InventoryException e) {
            orderRepository.updateStatus(event.orderId(), OrderStatus.FAILED);
            log.error("Inventory reservation failed for order {}", event.orderId(), e);

        } catch (PaymentException e) {
            inventoryService.release(event.orderId());
            orderRepository.updateStatus(event.orderId(), OrderStatus.PAYMENT_FAILED);
            log.error("Payment failed for order {}", event.orderId(), e);
        }
    }
}
```

### Event Sourcing Light
```java
@Table("events")
public class DomainEvent {
    @Id
    private Long id;
    private String aggregateType;
    private Long aggregateId;
    private String eventType;
    private String payload;
    private LocalDateTime occurredAt;
}

@Component
public class EventStore {

    private final JdbcTemplate jdbcTemplate;
    private final ObjectMapper objectMapper;

    public void store(Object event, String aggregateType, Long aggregateId) {
        jdbcTemplate.update("""
            INSERT INTO events (aggregate_type, aggregate_id, event_type, payload, occurred_at)
            VALUES (?, ?, ?, ?::jsonb, ?)
            """,
            aggregateType,
            aggregateId,
            event.getClass().getSimpleName(),
            objectMapper.writeValueAsString(event),
            LocalDateTime.now()
        );
    }

    public List<DomainEvent> getEventsFor(String aggregateType, Long aggregateId) {
        return jdbcTemplate.query("""
            SELECT * FROM events
            WHERE aggregate_type = ? AND aggregate_id = ?
            ORDER BY occurred_at
            """,
            new BeanPropertyRowMapper<>(DomainEvent.class),
            aggregateType, aggregateId
        );
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use callbacks for cross-cutting concerns | Business logic in callbacks |
| Use domain events for decoupling | Direct service calls |
| Handle events async when possible | Blocking operations in sync handlers |
| Use transactional events appropriately | Assume events are transactional |
| Keep callbacks idempotent | Side effects that can't be repeated |
| Log event handling failures | Swallow exceptions silently |

## Production Checklist

- [ ] Callbacks implement required interfaces
- [ ] Events registered via AbstractAggregateRoot
- [ ] Event handlers are idempotent
- [ ] Async handlers configured properly
- [ ] Transactional phases considered
- [ ] Error handling in place
- [ ] Monitoring for event processing
