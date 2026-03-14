# Spring Modulith Reference

## Setup

```xml
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-core</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

## Module Structure

```
com.example.app/
├── Application.java
├── order/                    # Order module
│   ├── Order.java           # Public API
│   ├── OrderService.java    # Public API
│   ├── package-info.java    # Module declaration
│   └── internal/            # Internal implementation
│       ├── OrderRepository.java
│       └── OrderValidator.java
├── inventory/               # Inventory module
│   ├── Inventory.java
│   ├── InventoryService.java
│   └── internal/
│       └── InventoryRepository.java
└── notification/            # Notification module
    ├── NotificationService.java
    └── internal/
        └── EmailSender.java
```

## Package Info

```java
// com/example/app/order/package-info.java
@ApplicationModule(
    displayName = "Order Management",
    allowedDependencies = {"inventory", "notification"}
)
package com.example.app.order;

import org.springframework.modulith.ApplicationModule;
```

## Public API

```java
// order/OrderService.java - Public API
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher events;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);

        events.publishEvent(new OrderCreatedEvent(order.getId()));

        return order;
    }

    public Optional<Order> findById(Long id) {
        return orderRepository.findById(id);
    }
}
```

## Events Between Modules

```java
// order/OrderEvents.java - Public event
public record OrderCreatedEvent(Long orderId) {}
public record OrderCompletedEvent(Long orderId, BigDecimal total) {}

// inventory/internal/InventoryEventListener.java
@Component
@RequiredArgsConstructor
class InventoryEventListener {

    private final InventoryService inventoryService;

    @EventListener
    void on(OrderCreatedEvent event) {
        inventoryService.reserveStock(event.orderId());
    }
}

// notification/internal/NotificationEventListener.java
@Component
@RequiredArgsConstructor
class NotificationEventListener {

    private final NotificationService notificationService;

    @ApplicationModuleListener
    void on(OrderCompletedEvent event) {
        notificationService.sendOrderConfirmation(event.orderId());
    }
}
```

## Async Events

```java
@Component
@RequiredArgsConstructor
class AsyncEventListener {

    @Async
    @ApplicationModuleListener
    void onOrderCreated(OrderCreatedEvent event) {
        // Processed asynchronously
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    void onOrderCreatedAfterCommit(OrderCreatedEvent event) {
        // Only after transaction commits
    }
}
```

## Event Publication Registry

```java
// Persist incomplete events
@Configuration
public class ModulithConfig {

    @Bean
    public EventPublicationRepository eventPublicationRepository(JdbcOperations jdbc) {
        return new JdbcEventPublicationRepository(jdbc);
    }
}
```

```yaml
spring:
  modulith:
    events:
      jdbc:
        schema-initialization:
          enabled: true
    republish-outstanding-events-on-restart: true
```

## Named Interfaces

```java
// order/package-info.java
@ApplicationModule(
    allowedDependencies = {
        "inventory :: InventoryManagement",
        "notification"
    }
)
package com.example.app.order;

// inventory/package-info.java
@NamedInterface("InventoryManagement")
package com.example.app.inventory;
```

## Architecture Testing

```java
@SpringBootTest
class ModularityTests {

    ApplicationModules modules = ApplicationModules.of(Application.class);

    @Test
    void verifyModularity() {
        modules.verify();
    }

    @Test
    void documentModules() {
        new Documenter(modules)
            .writeModulesAsPlantUml()
            .writeIndividualModulesAsPlantUml();
    }

    @Test
    void detectCycles() {
        assertThat(modules.detectDependencyCycles()).isEmpty();
    }
}
```

## Module Testing

```java
@ApplicationModuleTest
class OrderModuleTests {

    @Autowired
    private OrderService orderService;

    @Autowired
    private TestPublishedEvents events;

    @Test
    void shouldPublishEventOnOrderCreation() {
        orderService.createOrder(new CreateOrderRequest(...));

        assertThat(events)
            .contains(OrderCreatedEvent.class)
            .matching(e -> e.orderId() != null);
    }
}

// Integration test
@ApplicationModuleTest(mode = BootstrapMode.ALL_DEPENDENCIES)
class OrderModuleIntegrationTests {

    @Autowired
    private OrderService orderService;

    @Autowired
    private InventoryService inventoryService;

    @Test
    void shouldReserveInventoryOnOrderCreation() {
        orderService.createOrder(request);

        assertThat(inventoryService.getReserved(productId))
            .isEqualTo(expectedQuantity);
    }
}
```

## Scenario Testing

```java
@ApplicationModuleTest
class OrderScenarioTests {

    @Autowired
    private Scenario scenario;

    @Test
    void orderCreationScenario() {
        scenario.stimulate(() -> orderService.createOrder(request))
            .andWaitForEventOfType(OrderCreatedEvent.class)
            .toArriveAndVerify((event, result) -> {
                assertThat(event.orderId()).isNotNull();
            });
    }

    @Test
    void fullOrderFlow() {
        scenario.stimulate(() -> orderService.createOrder(request))
            .andWaitForEventOfType(OrderCreatedEvent.class)
            .toArrive()
            .andWaitForEventOfType(InventoryReservedEvent.class)
            .toArrive()
            .andWaitForEventOfType(NotificationSentEvent.class)
            .toArriveAndVerify((event, result) -> {
                // Final verification
            });
    }
}
```

## Documentation

```java
@Test
void generateDocs() {
    ApplicationModules modules = ApplicationModules.of(Application.class);

    new Documenter(modules)
        .writeModulesAsPlantUml(
            DiagramOptions.defaults()
                .withStyle(DiagramStyles.C4)
        )
        .writeModuleCanvases();
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Keep modules loosely coupled | Direct module-to-module calls |
| Use events for cross-module communication | Share internal classes |
| Define clear public APIs | Expose implementation details |
| Test modules in isolation | Skip architecture tests |
| Document module boundaries | Let modules grow unbounded |
