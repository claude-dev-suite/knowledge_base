# Spring Cloud CircuitBreaker - Resilience4j

## Resilience4j Modules

| Module | Purpose |
|--------|---------|
| CircuitBreaker | Prevent cascading failures |
| RateLimiter | Limit request rate |
| Retry | Retry failed operations |
| Bulkhead | Limit concurrent calls |
| TimeLimiter | Set execution timeouts |

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
<!-- For @CircuitBreaker annotation -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

## Annotation-Based Configuration

### @CircuitBreaker
```java
@Service
@Slf4j
public class UserService {

    @CircuitBreaker(name = "userService", fallbackMethod = "getUserFallback")
    public User getUser(Long id) {
        return userClient.getUserById(id);
    }

    private User getUserFallback(Long id, Throwable t) {
        log.warn("Fallback triggered for user {}: {}", id, t.getMessage());
        return User.defaultUser(id);
    }
}
```

### @Retry
```java
@Service
public class PaymentService {

    @Retry(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResponse processPayment(PaymentRequest request) {
        return paymentClient.process(request);
    }

    private PaymentResponse paymentFallback(PaymentRequest request, Throwable t) {
        return PaymentResponse.failed("Service temporarily unavailable");
    }
}
```

### @RateLimiter
```java
@Service
public class ApiService {

    @RateLimiter(name = "apiService", fallbackMethod = "rateLimitFallback")
    public ApiResponse callExternalApi(ApiRequest request) {
        return externalApiClient.call(request);
    }

    private ApiResponse rateLimitFallback(ApiRequest request, RequestNotPermitted e) {
        throw new TooManyRequestsException("Rate limit exceeded");
    }
}
```

### @Bulkhead
```java
@Service
public class ReportService {

    @Bulkhead(name = "reportService", fallbackMethod = "bulkheadFallback")
    public Report generateReport(ReportRequest request) {
        return reportGenerator.generate(request);
    }

    private Report bulkheadFallback(ReportRequest request, BulkheadFullException e) {
        throw new ServiceOverloadedException("Report service is overloaded");
    }
}
```

### @TimeLimiter
```java
@Service
public class SlowService {

    @TimeLimiter(name = "slowService", fallbackMethod = "timeoutFallback")
    public CompletableFuture<Result> slowOperation() {
        return CompletableFuture.supplyAsync(() -> {
            // Long running operation
            return performSlowOperation();
        });
    }

    private CompletableFuture<Result> timeoutFallback(TimeoutException e) {
        return CompletableFuture.completedFuture(Result.timeout());
    }
}
```

### Combining Patterns
```java
@Service
public class ResilientService {

    @CircuitBreaker(name = "externalService", fallbackMethod = "fallback")
    @RateLimiter(name = "externalService")
    @Retry(name = "externalService")
    @Bulkhead(name = "externalService")
    @TimeLimiter(name = "externalService")
    public CompletableFuture<Response> callExternalService(Request request) {
        return CompletableFuture.supplyAsync(() -> externalClient.call(request));
    }
}
```

## Configuration

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 3
    instances:
      userService:
        base-config: default
      paymentService:
        sliding-window-size: 20
        failure-rate-threshold: 40

  retry:
    configs:
      default:
        max-attempts: 3
        wait-duration: 1s
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - com.example.BusinessException
    instances:
      paymentService:
        max-attempts: 5
        wait-duration: 2s

  ratelimiter:
    configs:
      default:
        limit-for-period: 10
        limit-refresh-period: 1s
        timeout-duration: 0s
    instances:
      apiService:
        limit-for-period: 100
        limit-refresh-period: 1s

  bulkhead:
    configs:
      default:
        max-concurrent-calls: 10
        max-wait-duration: 0s
    instances:
      reportService:
        max-concurrent-calls: 5
        max-wait-duration: 500ms

  timelimiter:
    configs:
      default:
        timeout-duration: 3s
        cancel-running-future: true
    instances:
      slowService:
        timeout-duration: 10s
```

## Event Listeners

```java
@Component
@Slf4j
public class CircuitBreakerEventListener {

    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;

    @PostConstruct
    public void registerEventConsumers() {
        circuitBreakerRegistry.circuitBreaker("userService")
            .getEventPublisher()
            .onStateTransition(event ->
                log.warn("CircuitBreaker {} state changed from {} to {}",
                    event.getCircuitBreakerName(),
                    event.getStateTransition().getFromState(),
                    event.getStateTransition().getToState()))
            .onFailureRateExceeded(event ->
                log.error("CircuitBreaker {} failure rate exceeded: {}",
                    event.getCircuitBreakerName(),
                    event.getFailureRate()))
            .onSlowCallRateExceeded(event ->
                log.warn("CircuitBreaker {} slow call rate exceeded: {}",
                    event.getCircuitBreakerName(),
                    event.getSlowCallRate()));
    }
}
```

## Programmatic Usage

```java
@Service
@RequiredArgsConstructor
public class ResilientUserService {

    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final RetryRegistry retryRegistry;
    private final RateLimiterRegistry rateLimiterRegistry;
    private final BulkheadRegistry bulkheadRegistry;

    public User getUser(Long id) {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("userService");
        Retry retry = retryRegistry.retry("userService");
        RateLimiter rateLimiter = rateLimiterRegistry.rateLimiter("userService");
        Bulkhead bulkhead = bulkheadRegistry.bulkhead("userService");

        Supplier<User> supplier = () -> userClient.getUserById(id);

        Supplier<User> decoratedSupplier = Decorators.ofSupplier(supplier)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .withRateLimiter(rateLimiter)
            .withBulkhead(bulkhead)
            .withFallback(Arrays.asList(
                IOException.class,
                TimeoutException.class
            ), throwable -> getDefaultUser(id))
            .decorate();

        return decoratedSupplier.get();
    }
}
```

## Metrics

```yaml
management:
  metrics:
    tags:
      application: ${spring.application.name}
    distribution:
      percentiles-histogram:
        resilience4j.circuitbreaker.calls: true
  endpoint:
    metrics:
      enabled: true
  endpoints:
    web:
      exposure:
        include: health, metrics, circuitbreakers, retries, ratelimiters, bulkheads
```

### Key Metrics
- `resilience4j.circuitbreaker.calls` - Call outcomes
- `resilience4j.circuitbreaker.state` - Current state
- `resilience4j.circuitbreaker.failure.rate` - Failure rate
- `resilience4j.retry.calls` - Retry outcomes
- `resilience4j.ratelimiter.available.permissions` - Available permits
- `resilience4j.bulkhead.available.concurrent.calls` - Available slots

## Best Practices

| Do | Don't |
|----|-------|
| Order decorators: Retry → CircuitBreaker → RateLimiter → Bulkhead | Random ordering |
| Configure exception lists | Catch all exceptions |
| Use exponential backoff for retries | Fixed retry intervals |
| Set appropriate bulkhead limits | Unlimited concurrency |
| Monitor and alert on state changes | Ignore circuit breaker events |
| Test failure scenarios | Only test happy path |
