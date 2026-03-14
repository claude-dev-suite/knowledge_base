# Spring Cloud CircuitBreaker - Basics

## Overview

Spring Cloud Circuit Breaker provides an abstraction across different circuit breaker implementations. The primary implementation is Resilience4j.

## Circuit Breaker States

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Circuit Breaker States                                │
│                                                                         │
│     ┌──────────┐    Failure Threshold    ┌────────┐                    │
│     │  CLOSED  │────────────────────────▶│  OPEN  │                    │
│     │ (Normal) │                          │(Block) │                    │
│     └────▲─────┘                          └───┬────┘                    │
│          │                                    │                         │
│          │      Success               Wait Duration                     │
│          │                                    │                         │
│          │         ┌───────────────┐          │                         │
│          └─────────│  HALF-OPEN    │◀─────────┘                         │
│                    │ (Test calls)  │                                    │
│                    └───────────────┘                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

## Basic Usage

### Programmatic
```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final CircuitBreakerFactory circuitBreakerFactory;
    private final UserClient userClient;

    public User getUser(Long id) {
        CircuitBreaker circuitBreaker = circuitBreakerFactory.create("userService");

        return circuitBreaker.run(
            () -> userClient.getUserById(id),
            throwable -> getDefaultUser(id)
        );
    }

    private User getDefaultUser(Long id) {
        return User.builder()
            .id(id)
            .name("Unknown")
            .email("unknown@example.com")
            .build();
    }
}
```

### With ReactiveCircuitBreaker
```java
@Service
@RequiredArgsConstructor
public class ReactiveUserService {

    private final ReactiveCircuitBreakerFactory circuitBreakerFactory;
    private final WebClient webClient;

    public Mono<User> getUser(Long id) {
        ReactiveCircuitBreaker circuitBreaker = circuitBreakerFactory.create("userService");

        return circuitBreaker.run(
            webClient.get()
                .uri("/api/users/{id}", id)
                .retrieve()
                .bodyToMono(User.class),
            throwable -> Mono.just(getDefaultUser(id))
        );
    }
}
```

## Configuration

### application.yml
```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 3
        automatic-transition-from-open-to-half-open-enabled: true
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - org.springframework.web.client.HttpServerErrorException
        ignore-exceptions:
          - com.example.BusinessException

    instances:
      userService:
        base-config: default
        failure-rate-threshold: 60
        wait-duration-in-open-state: 30s

      paymentService:
        sliding-window-size: 20
        failure-rate-threshold: 40
        wait-duration-in-open-state: 60s
        slow-call-rate-threshold: 100
        slow-call-duration-threshold: 2s
```

### Java Configuration
```java
@Configuration
public class CircuitBreakerConfiguration {

    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> defaultCustomizer() {
        return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
            .circuitBreakerConfig(CircuitBreakerConfig.custom()
                .slidingWindowSize(10)
                .failureRateThreshold(50)
                .waitDurationInOpenState(Duration.ofSeconds(10))
                .permittedNumberOfCallsInHalfOpenState(3)
                .build())
            .timeLimiterConfig(TimeLimiterConfig.custom()
                .timeoutDuration(Duration.ofSeconds(3))
                .build())
            .build());
    }

    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> slowServiceCustomizer() {
        return factory -> factory.configure(builder -> builder
            .circuitBreakerConfig(CircuitBreakerConfig.custom()
                .slowCallRateThreshold(80)
                .slowCallDurationThreshold(Duration.ofSeconds(5))
                .build())
            .timeLimiterConfig(TimeLimiterConfig.custom()
                .timeoutDuration(Duration.ofSeconds(10))
                .build()),
            "slowService");
    }
}
```

## Configuration Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `sliding-window-size` | Number of calls to evaluate | 100 |
| `sliding-window-type` | COUNT_BASED or TIME_BASED | COUNT_BASED |
| `minimum-number-of-calls` | Minimum calls before evaluating | 100 |
| `failure-rate-threshold` | Failure % to open circuit | 50 |
| `wait-duration-in-open-state` | Time before half-open | 60s |
| `permitted-number-of-calls-in-half-open-state` | Test calls in half-open | 10 |
| `slow-call-rate-threshold` | Slow call % to open circuit | 100 |
| `slow-call-duration-threshold` | Duration to consider slow | 60s |

## Monitoring

### Actuator Endpoints
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, circuitbreakers, circuitbreakerevents
  health:
    circuitbreakers:
      enabled: true
```

### Endpoints
- `GET /actuator/circuitbreakers` - List all circuit breakers
- `GET /actuator/circuitbreakers/{name}` - Specific circuit breaker
- `GET /actuator/circuitbreakerevents` - Recent events

## Best Practices

| Do | Don't |
|----|-------|
| Configure appropriate thresholds | Use defaults for all services |
| Implement fallbacks | Let failures cascade |
| Monitor circuit breaker state | Ignore circuit breaker events |
| Use different configs per service | One-size-fits-all configuration |
| Test failure scenarios | Assume happy path only |
| Log state transitions | Miss circuit breaker issues |

## Production Checklist

- [ ] Failure thresholds tuned per service
- [ ] Timeout durations appropriate
- [ ] Fallback behavior tested
- [ ] Monitoring and alerting configured
- [ ] Health endpoints exposed
- [ ] Slow call detection configured
- [ ] Exception classification correct
