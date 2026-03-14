# Spring Cloud CircuitBreaker - Patterns

## Pattern Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Resilience Patterns                                  │
│                                                                         │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐           │
│  │  Timeout  │  │   Retry   │  │ CircuitBr │  │ Bulkhead  │           │
│  │  Pattern  │  │  Pattern  │  │  Pattern  │  │  Pattern  │           │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘           │
│        │              │              │              │                   │
│        └──────────────┴──────────────┴──────────────┘                   │
│                               │                                          │
│                        ┌──────▼──────┐                                  │
│                        │   Service   │                                  │
│                        │    Call     │                                  │
│                        └─────────────┘                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

## Retry Pattern

For transient failures that may succeed on retry.

```java
@Service
public class OrderService {

    @Retry(name = "orderService", fallbackMethod = "createOrderFallback")
    public Order createOrder(CreateOrderRequest request) {
        return orderClient.create(request);
    }

    private Order createOrderFallback(CreateOrderRequest request, Exception e) {
        // Queue for later processing
        orderQueue.add(request);
        return Order.pending(request.getId());
    }
}
```

```yaml
resilience4j:
  retry:
    instances:
      orderService:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException
          - org.springframework.web.client.HttpServerErrorException
        ignore-exceptions:
          - com.example.ValidationException
          - org.springframework.web.client.HttpClientErrorException
```

## Circuit Breaker Pattern

Prevent cascading failures by stopping calls to failing services.

```java
@Service
public class PaymentService {

    @CircuitBreaker(name = "paymentGateway", fallbackMethod = "paymentFallback")
    public PaymentResponse processPayment(PaymentRequest request) {
        return paymentGateway.process(request);
    }

    private PaymentResponse paymentFallback(PaymentRequest request, Exception e) {
        if (e instanceof CallNotPermittedException) {
            // Circuit is open
            return PaymentResponse.deferred("Payment will be retried later");
        }
        return PaymentResponse.failed(e.getMessage());
    }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentGateway:
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
```

## Bulkhead Pattern

Isolate failures and limit concurrent access.

### Semaphore Bulkhead
```java
@Service
public class ReportService {

    @Bulkhead(name = "reportGeneration", fallbackMethod = "bulkheadFallback")
    public Report generateReport(ReportRequest request) {
        return reportGenerator.generate(request);
    }

    private Report bulkheadFallback(ReportRequest request, BulkheadFullException e) {
        throw new ServiceOverloadedException("Report service is busy. Try again later.");
    }
}
```

```yaml
resilience4j:
  bulkhead:
    instances:
      reportGeneration:
        max-concurrent-calls: 10
        max-wait-duration: 500ms
```

### ThreadPool Bulkhead
```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      reportGeneration:
        max-thread-pool-size: 10
        core-thread-pool-size: 5
        queue-capacity: 50
        keep-alive-duration: 20s
```

```java
@Bulkhead(name = "reportGeneration", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<Report> generateReportAsync(ReportRequest request) {
    return CompletableFuture.supplyAsync(() -> reportGenerator.generate(request));
}
```

## Rate Limiter Pattern

Control the rate of requests to a resource.

```java
@Service
public class ExternalApiService {

    @RateLimiter(name = "externalApi", fallbackMethod = "rateLimitFallback")
    public ApiResponse callApi(ApiRequest request) {
        return externalApiClient.call(request);
    }

    private ApiResponse rateLimitFallback(ApiRequest request, RequestNotPermitted e) {
        throw new TooManyRequestsException("Rate limit exceeded. Retry after " +
            e.getRetryAfterPermission().map(Duration::toSeconds).orElse(0L) + " seconds");
    }
}
```

```yaml
resilience4j:
  ratelimiter:
    instances:
      externalApi:
        limit-for-period: 50      # 50 requests
        limit-refresh-period: 1s  # per second
        timeout-duration: 0s      # Fail immediately if limit exceeded
```

## Time Limiter Pattern

Set timeouts for long-running operations.

```java
@Service
public class AsyncService {

    @TimeLimiter(name = "asyncOperation", fallbackMethod = "timeoutFallback")
    public CompletableFuture<Result> performLongOperation(Request request) {
        return CompletableFuture.supplyAsync(() -> {
            return longRunningService.process(request);
        });
    }

    private CompletableFuture<Result> timeoutFallback(Request request, TimeoutException e) {
        return CompletableFuture.completedFuture(Result.timeout());
    }
}
```

```yaml
resilience4j:
  timelimiter:
    instances:
      asyncOperation:
        timeout-duration: 5s
        cancel-running-future: true
```

## Combined Patterns

### Full Resilience Stack
```java
@Service
public class ResilientExternalService {

    @CircuitBreaker(name = "externalService", fallbackMethod = "fallback")
    @Retry(name = "externalService")
    @RateLimiter(name = "externalService")
    @Bulkhead(name = "externalService")
    @TimeLimiter(name = "externalService")
    public CompletableFuture<Response> call(Request request) {
        return CompletableFuture.supplyAsync(() -> externalClient.call(request));
    }

    private CompletableFuture<Response> fallback(Request request, Throwable t) {
        log.error("All resilience patterns exhausted for request {}", request.getId(), t);
        return CompletableFuture.completedFuture(Response.fallback());
    }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      externalService:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s

  retry:
    instances:
      externalService:
        max-attempts: 3
        wait-duration: 1s

  ratelimiter:
    instances:
      externalService:
        limit-for-period: 100
        limit-refresh-period: 1s

  bulkhead:
    instances:
      externalService:
        max-concurrent-calls: 20

  timelimiter:
    instances:
      externalService:
        timeout-duration: 5s
```

## Fallback Strategies

### Cache Fallback
```java
@CircuitBreaker(name = "productService", fallbackMethod = "getProductFromCache")
public Product getProduct(Long id) {
    return productClient.getProduct(id);
}

private Product getProductFromCache(Long id, Throwable t) {
    return cacheService.get("product:" + id, Product.class)
        .orElseThrow(() -> new ProductNotFoundException(id));
}
```

### Default Value Fallback
```java
@CircuitBreaker(name = "configService", fallbackMethod = "getDefaultConfig")
public Config getConfig(String key) {
    return configClient.getConfig(key);
}

private Config getDefaultConfig(String key, Throwable t) {
    return Config.defaults().get(key);
}
```

### Queue Fallback
```java
@CircuitBreaker(name = "notificationService", fallbackMethod = "queueNotification")
public void sendNotification(Notification notification) {
    notificationClient.send(notification);
}

private void queueNotification(Notification notification, Throwable t) {
    messageQueue.send("notifications-retry", notification);
    log.warn("Notification queued for retry: {}", notification.getId());
}
```

## Best Practices

| Pattern | When to Use |
|---------|-------------|
| Retry | Transient failures, network glitches |
| Circuit Breaker | Prevent cascading failures |
| Bulkhead | Isolate critical resources |
| Rate Limiter | Protect external APIs, prevent abuse |
| Time Limiter | Long-running operations |

| Do | Don't |
|----|-------|
| Combine patterns appropriately | Over-engineer with all patterns |
| Tune thresholds per service | Use same config everywhere |
| Implement meaningful fallbacks | Return null or rethrow |
| Monitor pattern metrics | Ignore resilience events |
| Test failure scenarios | Only test happy paths |
