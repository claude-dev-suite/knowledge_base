# Spring Retry - Annotations

## Overview

Spring Retry provides declarative retry support through annotations, allowing you to add retry behavior to methods without boilerplate code.

## Enable Retry

```java
@Configuration
@EnableRetry
public class RetryConfig {
    // Configuration
}
```

## @Retryable

### Basic Usage
```java
@Service
public class ExternalApiService {

    @Retryable
    public String callExternalApi(String endpoint) {
        // Will retry on any exception
        return restTemplate.getForObject(endpoint, String.class);
    }
}
```

### Specify Exceptions
```java
@Service
public class PaymentService {

    @Retryable(
        retryFor = {TransientException.class, TimeoutException.class},
        noRetryFor = {InvalidRequestException.class}
    )
    public PaymentResult processPayment(PaymentRequest request) {
        return paymentGateway.process(request);
    }

    // Alternative syntax
    @Retryable(
        include = {IOException.class, TimeoutException.class},
        exclude = {IllegalArgumentException.class}
    )
    public void fetchData() {
        // ...
    }
}
```

### Configure Attempts and Backoff
```java
@Service
public class OrderService {

    @Retryable(
        maxAttempts = 5,
        backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 10000)
    )
    public Order createOrder(OrderRequest request) {
        return orderGateway.submit(request);
    }

    // Fixed delay
    @Retryable(
        maxAttempts = 3,
        backoff = @Backoff(delay = 2000)
    )
    public void syncInventory() {
        // Retries with 2 second delay
    }

    // Random backoff
    @Retryable(
        maxAttempts = 4,
        backoff = @Backoff(delay = 1000, maxDelay = 5000, random = true)
    )
    public void sendNotification() {
        // Random delay between 1-5 seconds
    }
}
```

### Expression-based Configuration
```java
@Service
public class ConfigurableRetryService {

    @Retryable(
        maxAttemptsExpression = "${retry.maxAttempts:3}",
        backoff = @Backoff(
            delayExpression = "${retry.delay:1000}",
            multiplierExpression = "${retry.multiplier:2}"
        )
    )
    public void configurableRetry() {
        // Retry config from properties
    }

    @Retryable(
        maxAttemptsExpression = "#{@retryConfig.maxAttempts}",
        backoff = @Backoff(delayExpression = "#{@retryConfig.delay}")
    )
    public void beanConfiguredRetry() {
        // Retry config from bean
    }
}
```

### Conditional Retry
```java
@Service
public class ConditionalRetryService {

    @Retryable(
        exceptionExpression = "#{#root.cause instanceof T(java.net.SocketTimeoutException)}"
    )
    public void retryOnTimeout() {
        // Only retry if root cause is SocketTimeoutException
    }

    @Retryable(
        exceptionExpression = "#{message.contains('temporary')}"
    )
    public void retryOnTemporaryError() {
        // Only retry if exception message contains 'temporary'
    }
}
```

## @Recover

### Basic Recovery
```java
@Service
public class ResilientService {

    @Retryable(maxAttempts = 3)
    public String fetchData(String id) {
        return externalService.getData(id);
    }

    @Recover
    public String recoverFetchData(Exception e, String id) {
        log.error("All retries failed for id: {}", id, e);
        return getFromCache(id);
    }
}
```

### Exception-specific Recovery
```java
@Service
public class MultiRecoveryService {

    @Retryable(retryFor = {TimeoutException.class, IOException.class})
    public Result processRequest(Request request) {
        return externalService.process(request);
    }

    @Recover
    public Result recoverTimeout(TimeoutException e, Request request) {
        log.warn("Timeout occurred, using cached result");
        return getCachedResult(request);
    }

    @Recover
    public Result recoverIO(IOException e, Request request) {
        log.warn("IO error, using fallback");
        return getFallbackResult(request);
    }

    @Recover
    public Result recoverDefault(Exception e, Request request) {
        log.error("Unexpected error", e);
        throw new ServiceException("Unable to process request", e);
    }
}
```

### Recovery with Return Type Matching
```java
@Service
public class TypedRecoveryService {

    @Retryable
    public Optional<User> findUser(Long id) {
        return userRepository.findById(id);
    }

    @Recover
    public Optional<User> recoverFindUser(Exception e, Long id) {
        // Must return same type as retryable method
        return Optional.empty();
    }

    @Retryable
    public List<Order> getOrders(String customerId) {
        return orderService.getByCustomer(customerId);
    }

    @Recover
    public List<Order> recoverGetOrders(Exception e, String customerId) {
        return Collections.emptyList();
    }
}
```

## @CircuitBreaker

```java
@Service
public class CircuitBreakerService {

    @CircuitBreaker(
        maxAttempts = 2,
        openTimeout = 5000,
        resetTimeout = 20000
    )
    public String callService() {
        return externalService.call();
    }

    @Recover
    public String recoverCallService(Exception e) {
        return "fallback-response";
    }
}
```

## Combining Annotations

```java
@Service
public class CompleteRetryService {

    @Retryable(
        retryFor = TransientException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2),
        listeners = "retryListener"
    )
    public OrderResult submitOrder(Order order) {
        return orderGateway.submit(order);
    }

    @Recover
    public OrderResult recoverSubmitOrder(TransientException e, Order order) {
        log.error("Order submission failed after retries: {}", order.getId());
        return OrderResult.failed(order.getId(), "Service unavailable");
    }
}

@Component
public class RetryListener extends RetryListenerSupport {

    @Override
    public <T, E extends Throwable> void onError(
            RetryContext context, RetryCallback<T, E> callback, Throwable t) {
        log.warn("Retry attempt {} failed", context.getRetryCount(), t);
    }
}
```

## Class-Level Retry

```java
@Service
@Retryable(maxAttempts = 3, backoff = @Backoff(delay = 500))
public class FullyRetryableService {

    public String methodA() {
        // Inherits class-level retry config
        return doWork();
    }

    public String methodB() {
        // Also retries with same config
        return doOtherWork();
    }

    @Retryable(maxAttempts = 5)
    public String methodC() {
        // Override: 5 attempts instead of 3
        return doCriticalWork();
    }
}
```

## Stateful Retry

```java
@Service
public class StatefulRetryService {

    @Retryable(stateful = true)
    public void processWithState(String key, Data data) {
        // State maintained between retries based on key
        processor.process(key, data);
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Specify retryFor exceptions | Retry all exceptions blindly |
| Use exponential backoff | Fixed aggressive retry intervals |
| Implement @Recover methods | Let exceptions propagate |
| Set maxAttempts appropriately | Unlimited retries |
| Log retry attempts | Ignore retry metrics |
| Use SpEL for configuration | Hardcode values |

## Production Checklist

- [ ] @EnableRetry configured
- [ ] Retryable exceptions specified
- [ ] Backoff strategy appropriate
- [ ] Recovery methods implemented
- [ ] Retry listeners for monitoring
- [ ] Max attempts reasonable
- [ ] Circuit breaker for cascading failures
