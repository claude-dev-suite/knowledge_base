# Spring Statemachine - Configuration

## Overview

Spring Statemachine configuration defines states, transitions, guards, and actions that control state machine behavior.

## Basic Configuration

### Enum-based States and Events
```java
public enum OrderState {
    CREATED, PAID, SHIPPED, DELIVERED, CANCELLED
}

public enum OrderEvent {
    PAY, SHIP, DELIVER, CANCEL
}

@Configuration
@EnableStateMachine
public class OrderStateMachineConfig
        extends StateMachineConfigurerAdapter<OrderState, OrderEvent> {

    @Override
    public void configure(StateMachineStateConfigurer<OrderState, OrderEvent> states)
            throws Exception {
        states
            .withStates()
            .initial(OrderState.CREATED)
            .end(OrderState.DELIVERED)
            .end(OrderState.CANCELLED)
            .states(EnumSet.allOf(OrderState.class));
    }

    @Override
    public void configure(StateMachineTransitionConfigurer<OrderState, OrderEvent> transitions)
            throws Exception {
        transitions
            .withExternal()
                .source(OrderState.CREATED).target(OrderState.PAID)
                .event(OrderEvent.PAY)
            .and()
            .withExternal()
                .source(OrderState.PAID).target(OrderState.SHIPPED)
                .event(OrderEvent.SHIP)
            .and()
            .withExternal()
                .source(OrderState.SHIPPED).target(OrderState.DELIVERED)
                .event(OrderEvent.DELIVER)
            .and()
            .withExternal()
                .source(OrderState.CREATED).target(OrderState.CANCELLED)
                .event(OrderEvent.CANCEL)
            .and()
            .withExternal()
                .source(OrderState.PAID).target(OrderState.CANCELLED)
                .event(OrderEvent.CANCEL);
    }
}
```

### State Machine Factory
```java
@Configuration
@EnableStateMachineFactory
public class OrderStateMachineFactoryConfig
        extends StateMachineConfigurerAdapter<OrderState, OrderEvent> {

    @Override
    public void configure(StateMachineConfigurationConfigurer<OrderState, OrderEvent> config)
            throws Exception {
        config
            .withConfiguration()
            .machineId("orderMachine")
            .autoStartup(true);
    }

    // States and transitions as above
}

@Service
@RequiredArgsConstructor
public class OrderMachineService {

    private final StateMachineFactory<OrderState, OrderEvent> factory;

    public StateMachine<OrderState, OrderEvent> createMachine(String orderId) {
        StateMachine<OrderState, OrderEvent> sm = factory.getStateMachine(orderId);
        sm.start();
        return sm;
    }
}
```

## Guards

### Guard Configuration
```java
@Override
public void configure(StateMachineTransitionConfigurer<OrderState, OrderEvent> transitions)
        throws Exception {
    transitions
        .withExternal()
            .source(OrderState.CREATED).target(OrderState.PAID)
            .event(OrderEvent.PAY)
            .guard(paymentGuard())
            .action(paymentAction())
        .and()
        .withExternal()
            .source(OrderState.PAID).target(OrderState.SHIPPED)
            .event(OrderEvent.SHIP)
            .guard(inventoryGuard());
}

@Bean
public Guard<OrderState, OrderEvent> paymentGuard() {
    return context -> {
        Order order = context.getExtendedState().get("order", Order.class);
        return order != null &&
               order.getTotal().compareTo(BigDecimal.ZERO) > 0 &&
               order.getPaymentMethod() != null;
    };
}

@Bean
public Guard<OrderState, OrderEvent> inventoryGuard() {
    return context -> {
        Order order = context.getExtendedState().get("order", Order.class);
        return inventoryService.isAvailable(order.getItems());
    };
}
```

### SpEL Guard
```java
@Override
public void configure(StateMachineTransitionConfigurer<OrderState, OrderEvent> transitions)
        throws Exception {
    transitions
        .withExternal()
            .source(OrderState.CREATED).target(OrderState.PAID)
            .event(OrderEvent.PAY)
            .guardExpression("extendedState.variables['paymentValid'] == true");
}
```

## Actions

### Entry/Exit Actions
```java
@Override
public void configure(StateMachineStateConfigurer<OrderState, OrderEvent> states)
        throws Exception {
    states
        .withStates()
        .initial(OrderState.CREATED, initAction())
        .state(OrderState.PAID, entryPaidAction(), exitPaidAction())
        .state(OrderState.SHIPPED, entryShippedAction(), null)
        .end(OrderState.DELIVERED);
}

@Bean
public Action<OrderState, OrderEvent> initAction() {
    return context -> {
        context.getExtendedState().getVariables().put("createdAt", LocalDateTime.now());
    };
}

@Bean
public Action<OrderState, OrderEvent> entryPaidAction() {
    return context -> {
        Order order = context.getExtendedState().get("order", Order.class);
        notificationService.sendPaymentConfirmation(order);
    };
}

@Bean
public Action<OrderState, OrderEvent> exitPaidAction() {
    return context -> {
        log.info("Exiting PAID state");
    };
}

@Bean
public Action<OrderState, OrderEvent> entryShippedAction() {
    return context -> {
        Order order = context.getExtendedState().get("order", Order.class);
        String trackingNumber = shippingService.createShipment(order);
        context.getExtendedState().getVariables().put("trackingNumber", trackingNumber);
    };
}
```

### Transition Actions
```java
@Override
public void configure(StateMachineTransitionConfigurer<OrderState, OrderEvent> transitions)
        throws Exception {
    transitions
        .withExternal()
            .source(OrderState.CREATED).target(OrderState.PAID)
            .event(OrderEvent.PAY)
            .action(paymentAction(), errorAction());
}

@Bean
public Action<OrderState, OrderEvent> paymentAction() {
    return context -> {
        Order order = context.getExtendedState().get("order", Order.class);
        PaymentResult result = paymentService.processPayment(order);
        context.getExtendedState().getVariables().put("paymentResult", result);
    };
}

@Bean
public Action<OrderState, OrderEvent> errorAction() {
    return context -> {
        Exception exception = context.getException();
        log.error("Action failed", exception);
        // Handle error
    };
}
```

## Hierarchical States

```java
@Override
public void configure(StateMachineStateConfigurer<OrderState, OrderEvent> states)
        throws Exception {
    states
        .withStates()
            .initial(OrderState.NEW)
            .state(OrderState.PROCESSING)
            .end(OrderState.COMPLETED)
        .and()
        .withStates()
            .parent(OrderState.PROCESSING)
            .initial(OrderState.VALIDATING)
            .state(OrderState.CHARGING)
            .state(OrderState.SHIPPING);
}
```

## Choice States

```java
@Override
public void configure(StateMachineStateConfigurer<OrderState, OrderEvent> states)
        throws Exception {
    states
        .withStates()
            .initial(OrderState.SUBMITTED)
            .choice(OrderState.CHECK_INVENTORY)
            .state(OrderState.IN_STOCK)
            .state(OrderState.BACKORDER)
            .end(OrderState.FULFILLED);
}

@Override
public void configure(StateMachineTransitionConfigurer<OrderState, OrderEvent> transitions)
        throws Exception {
    transitions
        .withExternal()
            .source(OrderState.SUBMITTED).target(OrderState.CHECK_INVENTORY)
            .event(OrderEvent.PROCESS)
        .and()
        .withChoice()
            .source(OrderState.CHECK_INVENTORY)
            .first(OrderState.IN_STOCK, inStockGuard())
            .last(OrderState.BACKORDER);
}

@Bean
public Guard<OrderState, OrderEvent> inStockGuard() {
    return context -> {
        Order order = context.getExtendedState().get("order", Order.class);
        return inventoryService.checkStock(order.getItems());
    };
}
```

## Junction States

```java
@Override
public void configure(StateMachineStateConfigurer<OrderState, OrderEvent> states)
        throws Exception {
    states
        .withStates()
            .initial(OrderState.START)
            .junction(OrderState.JUNCTION)
            .state(OrderState.PATH_A)
            .state(OrderState.PATH_B)
            .state(OrderState.PATH_C);
}

@Override
public void configure(StateMachineTransitionConfigurer<OrderState, OrderEvent> transitions)
        throws Exception {
    transitions
        .withJunction()
            .source(OrderState.JUNCTION)
            .first(OrderState.PATH_A, conditionA())
            .then(OrderState.PATH_B, conditionB())
            .last(OrderState.PATH_C);
}
```

## Fork/Join States

```java
@Override
public void configure(StateMachineStateConfigurer<OrderState, OrderEvent> states)
        throws Exception {
    states
        .withStates()
            .initial(OrderState.READY)
            .fork(OrderState.FORK)
            .join(OrderState.JOIN)
            .state(OrderState.TASKS_DONE)
        .and()
        .withStates()
            .parent(OrderState.TASKS)
            .initial(OrderState.TASK1)
            .end(OrderState.TASK1_DONE)
        .and()
        .withStates()
            .parent(OrderState.TASKS)
            .initial(OrderState.TASK2)
            .end(OrderState.TASK2_DONE);
}

@Override
public void configure(StateMachineTransitionConfigurer<OrderState, OrderEvent> transitions)
        throws Exception {
    transitions
        .withExternal()
            .source(OrderState.READY).target(OrderState.FORK)
        .and()
        .withFork()
            .source(OrderState.FORK)
            .target(OrderState.TASK1)
            .target(OrderState.TASK2)
        .and()
        .withJoin()
            .source(OrderState.TASK1_DONE)
            .source(OrderState.TASK2_DONE)
            .target(OrderState.JOIN);
}
```

## Configuration Properties

```yaml
spring:
  statemachine:
    monitor:
      enabled: true
```

## Best Practices

| Do | Don't |
|----|-------|
| Use enums for states/events | String-based definitions |
| Keep guards simple | Complex business logic in guards |
| Use factory for multiple instances | Single shared instance |
| Define error actions | Ignore action failures |
| Use hierarchical states | Flat complex machines |
| Document state transitions | Undocumented flow |

## Production Checklist

- [ ] States and events defined as enums
- [ ] Guards validate preconditions
- [ ] Actions handle errors
- [ ] Factory configured for instances
- [ ] Listeners for monitoring
- [ ] State machine tested
