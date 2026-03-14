# Spring Retry - Basics

## Overview

Spring Retry provides declarative retry support for Spring applications. It allows methods to be retried when they fail, with configurable backoff strategies.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

## Enable Retry

```java
@Configuration
@EnableRetry
public class RetryConfig {
}
```

## @Retryable Annotation

### Basic Usage
```java
@Service
public class ExternalApiService {

    @Retryable(maxAttempts = 3)
    public String callExternalApi(String request) {
        return externalClient.call(request);  // Will retry on any exception
    }
}
```

### Specify Exceptions
```java
@Retryable(
    retryFor = {IOException.class, TimeoutException.class},
    noRetryFor = {IllegalArgumentException.class},
    maxAttempts = 3
)
public String callApi(String request) {
    return apiClient.call(request);
}
```

### With Backoff
```java
@Retryable(
    maxAttempts = 5,
    backoff = @Backoff(
        delay = 1000,        // Initial delay: 1 second
        multiplier = 2,      // Double the delay each retry
        maxDelay = 10000     // Max delay: 10 seconds
    )
)
public String callWithBackoff(String request) {
    return apiClient.call(request);
}
```

### Random Backoff
```java
@Retryable(
    maxAttempts = 3,
    backoff = @Backoff(
        delay = 1000,
        maxDelay = 5000,
        random = true  // Add random jitter
    )
)
public String callWithJitter(String request) {
    return apiClient.call(request);
}
```

## @Recover Annotation

Fallback method when all retries are exhausted.

```java
@Service
@Slf4j
public class PaymentService {

    @Retryable(
        retryFor = PaymentException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 2000)
    )
    public PaymentResult processPayment(PaymentRequest request) {
        return paymentGateway.process(request);
    }

    @Recover
    public PaymentResult recoverPayment(PaymentException e, PaymentRequest request) {
        log.error("All retries exhausted for payment: {}", request.getId(), e);

        // Queue for manual processing
        paymentQueue.addForManualReview(request);

        return PaymentResult.deferred(request.getId());
    }
}
```

### Multiple Recover Methods
```java
@Service
public class OrderService {

    @Retryable(maxAttempts = 3)
    public Order createOrder(OrderRequest request) {
        return orderClient.create(request);
    }

    @Recover
    public Order recoverFromTimeout(TimeoutException e, OrderRequest request) {
        log.warn("Timeout creating order, using async fallback");
        return queueOrderAsync(request);
    }

    @Recover
    public Order recoverFromServiceError(ServiceException e, OrderRequest request) {
        log.error("Service error, order creation failed", e);
        throw new OrderCreationFailedException(e);
    }
}
```

## RetryTemplate (Programmatic)

```java
@Configuration
public class RetryConfig {

    @Bean
    public RetryTemplate retryTemplate() {
        RetryTemplate template = new RetryTemplate();

        // Retry policy
        SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy(3);
        template.setRetryPolicy(retryPolicy);

        // Backoff policy
        ExponentialBackOffPolicy backOffPolicy = new ExponentialBackOffPolicy();
        backOffPolicy.setInitialInterval(1000);
        backOffPolicy.setMultiplier(2.0);
        backOffPolicy.setMaxInterval(10000);
        template.setBackOffPolicy(backOffPolicy);

        return template;
    }
}
```

### Using RetryTemplate
```java
@Service
@RequiredArgsConstructor
public class ApiService {

    private final RetryTemplate retryTemplate;
    private final ApiClient apiClient;

    public ApiResponse callApi(ApiRequest request) {
        return retryTemplate.execute(context -> {
            log.info("Attempt {} of calling API", context.getRetryCount() + 1);
            return apiClient.call(request);
        }, context -> {
            log.error("All retries exhausted", context.getLastThrowable());
            return ApiResponse.failed("Service unavailable");
        });
    }
}
```

## Listeners

### RetryListener
```java
@Component
@Slf4j
public class LoggingRetryListener implements RetryListener {

    @Override
    public <T, E extends Throwable> boolean open(RetryContext context, RetryCallback<T, E> callback) {
        log.info("Starting retry operation");
        return true;  // Proceed with retry
    }

    @Override
    public <T, E extends Throwable> void close(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
        if (throwable != null) {
            log.error("Retry operation completed with error after {} attempts",
                context.getRetryCount(), throwable);
        } else {
            log.info("Retry operation completed successfully after {} attempts",
                context.getRetryCount());
        }
    }

    @Override
    public <T, E extends Throwable> void onError(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
        log.warn("Retry attempt {} failed: {}",
            context.getRetryCount(), throwable.getMessage());
    }
}
```

### Register Listener
```java
@Bean
public RetryTemplate retryTemplate(LoggingRetryListener listener) {
    RetryTemplate template = new RetryTemplate();
    template.registerListener(listener);
    return template;
}
```

## Stateful Retry

For distributed systems where retry state needs to be maintained.

```java
@Retryable(
    stateful = true,
    maxAttempts = 3,
    backoff = @Backoff(delay = 5000)
)
public Order processOrder(String orderId) {
    return orderService.process(orderId);
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use exponential backoff | Fixed delay for all retries |
| Add jitter to prevent thundering herd | Same retry time for all clients |
| Classify retryable vs non-retryable | Retry all exceptions |
| Implement meaningful recovery | Throw on recovery failure |
| Log retry attempts | Silent retries |
| Set max attempts reasonably | Infinite retries |

## Configuration Reference

```java
@Retryable(
    // What to retry on
    retryFor = {IOException.class, TimeoutException.class},
    noRetryFor = {IllegalArgumentException.class},

    // Retry limits
    maxAttempts = 3,
    maxAttemptsExpression = "#{@retryConfig.maxAttempts}",

    // Backoff
    backoff = @Backoff(
        delay = 1000,
        delayExpression = "#{@retryConfig.delay}",
        multiplier = 2.0,
        multiplierExpression = "#{@retryConfig.multiplier}",
        maxDelay = 10000,
        maxDelayExpression = "#{@retryConfig.maxDelay}",
        random = true
    ),

    // State
    stateful = false,

    // Custom listeners
    listeners = {"loggingRetryListener"}
)
```
