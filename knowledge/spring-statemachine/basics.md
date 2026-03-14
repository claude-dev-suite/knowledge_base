# Spring Statemachine - Basics

## Overview

Spring Statemachine provides a framework for implementing finite state machines with Spring. It's useful for modeling workflows, order processing, and any state-driven logic.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.statemachine</groupId>
    <artifactId>spring-statemachine-starter</artifactId>
</dependency>
```

## State Machine Concepts

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    State Machine                                         │
│                                                                         │
│  ┌──────────┐  Event: PAY   ┌──────────┐  Event: SHIP  ┌──────────┐   │
│  │  CREATED │──────────────▶│   PAID   │──────────────▶│ SHIPPED  │   │
│  └──────────┘               └──────────┘               └──────────┘   │
│       │                          │                          │          │
│       │ Event: CANCEL           │ Event: CANCEL             │          │
│       ▼                          ▼                          ▼          │
│  ┌───────────────────────────────────────────────────────────────┐    │
│  │                        CANCELLED                               │    │
│  └───────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

## Define States and Events

```java
public enum OrderState {
    CREATED,
    PAID,
    SHIPPED,
    DELIVERED,
    CANCELLED
}

public enum OrderEvent {
    PAY,
    SHIP,
    DELIVER,
    CANCEL
}
```

## State Machine Configuration

```java
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
                .guard(paymentGuard())
                .action(paymentAction())
            .and()
            .withExternal()
                .source(OrderState.PAID).target(OrderState.SHIPPED)
                .event(OrderEvent.SHIP)
                .action(shipAction())
            .and()
            .withExternal()
                .source(OrderState.SHIPPED).target(OrderState.DELIVERED)
                .event(OrderEvent.DELIVER)
                .action(deliverAction())
            .and()
            .withExternal()
                .source(OrderState.CREATED).target(OrderState.CANCELLED)
                .event(OrderEvent.CANCEL)
            .and()
            .withExternal()
                .source(OrderState.PAID).target(OrderState.CANCELLED)
                .event(OrderEvent.CANCEL)
                .action(refundAction());
    }

    @Bean
    public Guard<OrderState, OrderEvent> paymentGuard() {
        return context -> {
            Order order = context.getExtendedState().get("order", Order.class);
            return order != null && order.getTotal().compareTo(BigDecimal.ZERO) > 0;
        };
    }

    @Bean
    public Action<OrderState, OrderEvent> paymentAction() {
        return context -> {
            Order order = context.getExtendedState().get("order", Order.class);
            paymentService.processPayment(order);
        };
    }

    @Bean
    public Action<OrderState, OrderEvent> shipAction() {
        return context -> {
            Order order = context.getExtendedState().get("order", Order.class);
            shippingService.createShipment(order);
        };
    }

    @Bean
    public Action<OrderState, OrderEvent> deliverAction() {
        return context -> {
            Order order = context.getExtendedState().get("order", Order.class);
            notificationService.notifyDelivered(order);
        };
    }

    @Bean
    public Action<OrderState, OrderEvent> refundAction() {
        return context -> {
            Order order = context.getExtendedState().get("order", Order.class);
            paymentService.refund(order);
        };
    }
}
```

## Using State Machine

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final StateMachineFactory<OrderState, OrderEvent> stateMachineFactory;

    public void processOrder(Order order) {
        StateMachine<OrderState, OrderEvent> sm = stateMachineFactory.getStateMachine();
        sm.getExtendedState().getVariables().put("order", order);
        sm.start();

        // Send event
        sm.sendEvent(OrderEvent.PAY);

        // Check current state
        OrderState currentState = sm.getState().getId();
    }

    public boolean sendEvent(String orderId, OrderEvent event) {
        StateMachine<OrderState, OrderEvent> sm = getStateMachine(orderId);

        Message<OrderEvent> message = MessageBuilder
            .withPayload(event)
            .setHeader("orderId", orderId)
            .build();

        return sm.sendEvent(Mono.just(message)).blockFirst().getResultType() ==
            StateMachineEventResult.ResultType.ACCEPTED;
    }
}
```

## State Machine Listener

```java
@Component
@Slf4j
public class OrderStateMachineListener
    extends StateMachineListenerAdapter<OrderState, OrderEvent> {

    @Override
    public void stateChanged(State<OrderState, OrderEvent> from,
                            State<OrderState, OrderEvent> to) {
        log.info("State changed from {} to {}",
            from != null ? from.getId() : "none",
            to.getId());
    }

    @Override
    public void eventNotAccepted(Message<OrderEvent> event) {
        log.warn("Event not accepted: {}", event.getPayload());
    }

    @Override
    public void transition(Transition<OrderState, OrderEvent> transition) {
        log.debug("Transition: {} -> {} via {}",
            transition.getSource().getId(),
            transition.getTarget().getId(),
            transition.getTrigger().getEvent());
    }
}
```

## Persisting State

### JPA Persistence
```java
@Entity
public class OrderStateEntity {
    @Id
    private String machineId;
    private String state;
    private byte[] stateMachineContext;
}

@Component
public class JpaStateMachinePersister implements StateMachinePersister<OrderState, OrderEvent, String> {

    private final StateMachineRuntimePersister<OrderState, OrderEvent, String> persister;

    @Override
    public void persist(StateMachine<OrderState, OrderEvent> stateMachine, String orderId) {
        persister.write(stateMachine.getStateMachineAccessor()
            .doWithAllRegions(accessor -> accessor.resetStateMachine(
                new DefaultStateMachineContext<>(
                    stateMachine.getState().getId(),
                    null, null, null, null, orderId
                )
            )), orderId);
    }

    @Override
    public StateMachine<OrderState, OrderEvent> restore(
            StateMachine<OrderState, OrderEvent> stateMachine, String orderId) {
        return persister.read(stateMachine, orderId);
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Define clear states and events | Overload state meanings |
| Use guards for preconditions | Skip validation |
| Persist state for long-running processes | Keep state in memory only |
| Use listeners for monitoring | Ignore state transitions |
| Handle errors in actions | Let exceptions propagate |
| Test all transitions | Assume transitions work |

## Production Checklist

- [ ] States and events clearly defined
- [ ] Guards validate preconditions
- [ ] Actions handle errors
- [ ] State persistence configured
- [ ] Transitions tested
- [ ] Monitoring in place
