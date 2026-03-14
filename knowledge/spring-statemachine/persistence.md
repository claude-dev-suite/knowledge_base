# Spring Statemachine - Persistence

## Overview

Spring Statemachine provides persistence support for saving and restoring state machine state, enabling long-running processes and recovery from failures.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.statemachine</groupId>
    <artifactId>spring-statemachine-data-jpa</artifactId>
</dependency>
```

## JPA Persistence

### Entity Configuration
```java
@Entity
@Table(name = "state_machine_context")
public class StateMachineContextEntity {

    @Id
    private String machineId;

    @Lob
    private byte[] stateMachineContext;

    @Column(name = "current_state")
    private String state;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    // Getters and setters
}

@Repository
public interface StateMachineContextRepository
        extends JpaRepository<StateMachineContextEntity, String> {
}
```

### JPA Persister
```java
@Component
public class JpaStateMachinePersister
        implements StateMachinePersister<OrderState, OrderEvent, String> {

    private final StateMachineContextRepository repository;
    private final StateMachineRuntimePersister<OrderState, OrderEvent, String> persister;

    @Override
    public void persist(StateMachine<OrderState, OrderEvent> stateMachine, String contextObj) {
        StateMachineContext<OrderState, OrderEvent> context =
            stateMachine.getStateMachineAccessor()
                .doWithRegion(accessor -> {
                    return accessor.getStateMachineContext();
                });

        byte[] serialized = serialize(context);

        StateMachineContextEntity entity = new StateMachineContextEntity();
        entity.setMachineId(contextObj);
        entity.setStateMachineContext(serialized);
        entity.setState(stateMachine.getState().getId().name());
        entity.setUpdatedAt(LocalDateTime.now());

        repository.save(entity);
    }

    @Override
    public StateMachine<OrderState, OrderEvent> restore(
            StateMachine<OrderState, OrderEvent> stateMachine, String contextObj) {

        StateMachineContextEntity entity = repository.findById(contextObj)
            .orElseThrow(() -> new RuntimeException("State machine not found: " + contextObj));

        StateMachineContext<OrderState, OrderEvent> context =
            deserialize(entity.getStateMachineContext());

        stateMachine.getStateMachineAccessor()
            .doWithAllRegions(accessor -> {
                accessor.resetStateMachine(context);
            });

        return stateMachine;
    }

    private byte[] serialize(StateMachineContext<OrderState, OrderEvent> context) {
        // Serialization logic
    }

    private StateMachineContext<OrderState, OrderEvent> deserialize(byte[] data) {
        // Deserialization logic
    }
}
```

### Using Built-in JPA Persister
```java
@Configuration
@EnableStateMachine
public class PersistenceConfig extends StateMachineConfigurerAdapter<OrderState, OrderEvent> {

    @Autowired
    private JpaStateMachineRepository jpaRepository;

    @Override
    public void configure(StateMachineConfigurationConfigurer<OrderState, OrderEvent> config)
            throws Exception {
        config
            .withPersistence()
            .runtimePersister(stateMachineRuntimePersister());
    }

    @Bean
    public StateMachineRuntimePersister<OrderState, OrderEvent, String> stateMachineRuntimePersister() {
        return new JpaPersistingStateMachineInterceptor<>(jpaRepository);
    }
}
```

## Redis Persistence

### Redis Configuration
```java
@Configuration
public class RedisStateMachineConfig {

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory("localhost", 6379);
    }

    @Bean
    public RedisTemplate<String, byte[]> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, byte[]> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new JdkSerializationRedisSerializer());
        return template;
    }
}

@Component
public class RedisStateMachinePersister
        implements StateMachinePersister<OrderState, OrderEvent, String> {

    private final RedisTemplate<String, byte[]> redisTemplate;
    private static final String KEY_PREFIX = "statemachine:";

    @Override
    public void persist(StateMachine<OrderState, OrderEvent> stateMachine, String machineId) {
        StateMachineContext<OrderState, OrderEvent> context =
            stateMachine.getStateMachineAccessor()
                .doWithRegion(StateMachineAccessor::getStateMachineContext);

        byte[] serialized = serialize(context);
        redisTemplate.opsForValue().set(KEY_PREFIX + machineId, serialized);
    }

    @Override
    public StateMachine<OrderState, OrderEvent> restore(
            StateMachine<OrderState, OrderEvent> stateMachine, String machineId) {

        byte[] data = redisTemplate.opsForValue().get(KEY_PREFIX + machineId);
        if (data == null) {
            throw new RuntimeException("State machine not found: " + machineId);
        }

        StateMachineContext<OrderState, OrderEvent> context = deserialize(data);

        stateMachine.getStateMachineAccessor()
            .doWithAllRegions(accessor -> accessor.resetStateMachine(context));

        return stateMachine;
    }
}
```

## MongoDB Persistence

```java
@Document(collection = "state_machines")
public class StateMachineDocument {

    @Id
    private String machineId;

    private String currentState;
    private Map<String, Object> extendedState;
    private List<String> historyStates;
    private LocalDateTime updatedAt;

    // Getters and setters
}

@Repository
public interface StateMachineMongoRepository
        extends MongoRepository<StateMachineDocument, String> {
}

@Component
public class MongoStateMachinePersister
        implements StateMachinePersister<OrderState, OrderEvent, String> {

    private final StateMachineMongoRepository repository;

    @Override
    public void persist(StateMachine<OrderState, OrderEvent> stateMachine, String machineId) {
        StateMachineDocument doc = new StateMachineDocument();
        doc.setMachineId(machineId);
        doc.setCurrentState(stateMachine.getState().getId().name());
        doc.setExtendedState(new HashMap<>(stateMachine.getExtendedState().getVariables()));
        doc.setUpdatedAt(LocalDateTime.now());

        repository.save(doc);
    }

    @Override
    public StateMachine<OrderState, OrderEvent> restore(
            StateMachine<OrderState, OrderEvent> stateMachine, String machineId) {

        StateMachineDocument doc = repository.findById(machineId)
            .orElseThrow();

        OrderState state = OrderState.valueOf(doc.getCurrentState());

        StateMachineContext<OrderState, OrderEvent> context =
            new DefaultStateMachineContext<>(state, null, null,
                new DefaultExtendedState(doc.getExtendedState()), null, machineId);

        stateMachine.getStateMachineAccessor()
            .doWithAllRegions(accessor -> accessor.resetStateMachine(context));

        return stateMachine;
    }
}
```

## Service Integration

```java
@Service
@RequiredArgsConstructor
public class OrderStateMachineService {

    private final StateMachineFactory<OrderState, OrderEvent> factory;
    private final StateMachinePersister<OrderState, OrderEvent, String> persister;

    public OrderState processOrder(String orderId, Order order) {
        StateMachine<OrderState, OrderEvent> sm = acquireStateMachine(orderId);

        try {
            sm.getExtendedState().getVariables().put("order", order);
            sm.start();
            return sm.getState().getId();
        } finally {
            releaseStateMachine(orderId, sm);
        }
    }

    public OrderState sendEvent(String orderId, OrderEvent event) {
        StateMachine<OrderState, OrderEvent> sm = acquireStateMachine(orderId);

        try {
            sm.sendEvent(event);
            return sm.getState().getId();
        } finally {
            releaseStateMachine(orderId, sm);
        }
    }

    public OrderState getCurrentState(String orderId) {
        StateMachine<OrderState, OrderEvent> sm = acquireStateMachine(orderId);
        try {
            return sm.getState().getId();
        } finally {
            // Don't persist on read-only
            sm.stop();
        }
    }

    private StateMachine<OrderState, OrderEvent> acquireStateMachine(String orderId) {
        StateMachine<OrderState, OrderEvent> sm = factory.getStateMachine(orderId);

        try {
            persister.restore(sm, orderId);
        } catch (Exception e) {
            // New state machine, start fresh
            sm.start();
        }

        return sm;
    }

    private void releaseStateMachine(String orderId, StateMachine<OrderState, OrderEvent> sm) {
        try {
            persister.persist(sm, orderId);
        } catch (Exception e) {
            log.error("Failed to persist state machine: {}", orderId, e);
        }
        sm.stop();
    }
}
```

## Interceptor-based Persistence

```java
@Configuration
@EnableStateMachineFactory
public class InterceptorPersistenceConfig
        extends StateMachineConfigurerAdapter<OrderState, OrderEvent> {

    @Override
    public void configure(StateMachineConfigurationConfigurer<OrderState, OrderEvent> config)
            throws Exception {
        config
            .withConfiguration()
            .listener(new StateMachineListenerAdapter<>() {
                @Override
                public void stateChanged(State<OrderState, OrderEvent> from,
                                        State<OrderState, OrderEvent> to) {
                    log.info("State changed from {} to {}",
                        from != null ? from.getId() : "none",
                        to.getId());
                }
            });
    }
}

public class PersistingStateMachineInterceptor
        extends StateMachineInterceptorAdapter<OrderState, OrderEvent> {

    private final StateMachinePersister<OrderState, OrderEvent, String> persister;

    @Override
    public void postStateChange(State<OrderState, OrderEvent> state,
                                Message<OrderEvent> message,
                                Transition<OrderState, OrderEvent> transition,
                                StateMachine<OrderState, OrderEvent> stateMachine,
                                StateMachine<OrderState, OrderEvent> rootStateMachine) {

        String machineId = stateMachine.getId();
        try {
            persister.persist(stateMachine, machineId);
        } catch (Exception e) {
            log.error("Failed to persist after state change", e);
        }
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Persist after state changes | Persist on every event |
| Handle persistence failures | Ignore persistence errors |
| Use transactions where possible | Skip atomicity |
| Clean up old state machines | Let data grow unbounded |
| Version serialized context | Break on format changes |
| Test restore scenarios | Only test happy path |

## Production Checklist

- [ ] Persistence storage selected
- [ ] Serialization strategy defined
- [ ] Error handling implemented
- [ ] Cleanup job scheduled
- [ ] Recovery tested
- [ ] Monitoring in place
