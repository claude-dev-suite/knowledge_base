# Spring Retry - Policies

## Overview

Spring Retry provides flexible retry and backoff policies for programmatic control over retry behavior.

## Retry Policies

### SimpleRetryPolicy
```java
@Configuration
public class RetryPolicyConfig {

    @Bean
    public RetryTemplate simpleRetryTemplate() {
        RetryTemplate template = new RetryTemplate();

        SimpleRetryPolicy policy = new SimpleRetryPolicy();
        policy.setMaxAttempts(3);

        template.setRetryPolicy(policy);
        return template;
    }
}
```

### Exception-based Policy
```java
@Bean
public RetryTemplate exceptionBasedRetryTemplate() {
    RetryTemplate template = new RetryTemplate();

    Map<Class<? extends Throwable>, Boolean> retryableExceptions = new HashMap<>();
    retryableExceptions.put(TransientException.class, true);
    retryableExceptions.put(TimeoutException.class, true);
    retryableExceptions.put(IllegalArgumentException.class, false);

    SimpleRetryPolicy policy = new SimpleRetryPolicy(5, retryableExceptions);
    template.setRetryPolicy(policy);

    return template;
}
```

### MaxAttemptsRetryPolicy
```java
@Bean
public RetryTemplate maxAttemptsRetryTemplate() {
    RetryTemplate template = new RetryTemplate();

    MaxAttemptsRetryPolicy policy = new MaxAttemptsRetryPolicy();
    policy.setMaxAttempts(10);

    template.setRetryPolicy(policy);
    return template;
}
```

### TimeoutRetryPolicy
```java
@Bean
public RetryTemplate timeoutRetryTemplate() {
    RetryTemplate template = new RetryTemplate();

    TimeoutRetryPolicy policy = new TimeoutRetryPolicy();
    policy.setTimeout(30000L); // 30 seconds total

    template.setRetryPolicy(policy);
    return template;
}
```

### CompositeRetryPolicy
```java
@Bean
public RetryTemplate compositeRetryTemplate() {
    RetryTemplate template = new RetryTemplate();

    // Both policies must allow retry
    CompositeRetryPolicy composite = new CompositeRetryPolicy();
    composite.setPolicies(new RetryPolicy[]{
        new SimpleRetryPolicy(5),
        new TimeoutRetryPolicy(60000L)
    });

    template.setRetryPolicy(composite);
    return template;
}

@Bean
public RetryTemplate optimisticCompositeTemplate() {
    RetryTemplate template = new RetryTemplate();

    // Any policy can allow retry
    CompositeRetryPolicy composite = new CompositeRetryPolicy();
    composite.setOptimistic(true);
    composite.setPolicies(new RetryPolicy[]{
        new SimpleRetryPolicy(3),
        new TimeoutRetryPolicy(10000L)
    });

    template.setRetryPolicy(composite);
    return template;
}
```

### ExceptionClassifierRetryPolicy
```java
@Bean
public RetryTemplate classifierRetryTemplate() {
    RetryTemplate template = new RetryTemplate();

    ExceptionClassifierRetryPolicy policy = new ExceptionClassifierRetryPolicy();

    Map<Class<? extends Throwable>, RetryPolicy> policyMap = new HashMap<>();
    policyMap.put(TimeoutException.class, new SimpleRetryPolicy(5));
    policyMap.put(IOException.class, new SimpleRetryPolicy(3));
    policyMap.put(RuntimeException.class, new NeverRetryPolicy());

    policy.setPolicyMap(policyMap);
    template.setRetryPolicy(policy);

    return template;
}
```

### Custom Retry Policy
```java
public class ConditionalRetryPolicy implements RetryPolicy {

    private final int maxAttempts;
    private final Predicate<Throwable> retryPredicate;

    public ConditionalRetryPolicy(int maxAttempts, Predicate<Throwable> predicate) {
        this.maxAttempts = maxAttempts;
        this.retryPredicate = predicate;
    }

    @Override
    public boolean canRetry(RetryContext context) {
        Throwable lastException = context.getLastThrowable();
        return context.getRetryCount() < maxAttempts &&
               (lastException == null || retryPredicate.test(lastException));
    }

    @Override
    public RetryContext open(RetryContext parent) {
        return new RetryContextSupport(parent);
    }

    @Override
    public void close(RetryContext context) {
        // Cleanup
    }

    @Override
    public void registerThrowable(RetryContext context, Throwable throwable) {
        ((RetryContextSupport) context).registerThrowable(throwable);
    }
}

// Usage
@Bean
public RetryTemplate customPolicyTemplate() {
    RetryTemplate template = new RetryTemplate();

    ConditionalRetryPolicy policy = new ConditionalRetryPolicy(5,
        e -> e instanceof TransientException &&
             ((TransientException) e).isRetryable());

    template.setRetryPolicy(policy);
    return template;
}
```

## Backoff Policies

### FixedBackOffPolicy
```java
@Bean
public RetryTemplate fixedBackoffTemplate() {
    RetryTemplate template = new RetryTemplate();

    FixedBackOffPolicy backoff = new FixedBackOffPolicy();
    backoff.setBackOffPeriod(2000L); // 2 seconds

    template.setBackOffPolicy(backoff);
    template.setRetryPolicy(new SimpleRetryPolicy(3));

    return template;
}
```

### ExponentialBackOffPolicy
```java
@Bean
public RetryTemplate exponentialBackoffTemplate() {
    RetryTemplate template = new RetryTemplate();

    ExponentialBackOffPolicy backoff = new ExponentialBackOffPolicy();
    backoff.setInitialInterval(1000L);  // 1 second
    backoff.setMultiplier(2.0);          // Double each time
    backoff.setMaxInterval(30000L);      // Max 30 seconds

    template.setBackOffPolicy(backoff);
    template.setRetryPolicy(new SimpleRetryPolicy(5));

    return template;
}
```

### ExponentialRandomBackOffPolicy
```java
@Bean
public RetryTemplate randomExponentialTemplate() {
    RetryTemplate template = new RetryTemplate();

    ExponentialRandomBackOffPolicy backoff = new ExponentialRandomBackOffPolicy();
    backoff.setInitialInterval(1000L);
    backoff.setMultiplier(2.0);
    backoff.setMaxInterval(30000L);
    // Adds randomness to prevent thundering herd

    template.setBackOffPolicy(backoff);
    template.setRetryPolicy(new SimpleRetryPolicy(5));

    return template;
}
```

### UniformRandomBackOffPolicy
```java
@Bean
public RetryTemplate uniformRandomTemplate() {
    RetryTemplate template = new RetryTemplate();

    UniformRandomBackOffPolicy backoff = new UniformRandomBackOffPolicy();
    backoff.setMinBackOffPeriod(1000L); // Min 1 second
    backoff.setMaxBackOffPeriod(5000L); // Max 5 seconds

    template.setBackOffPolicy(backoff);
    template.setRetryPolicy(new SimpleRetryPolicy(4));

    return template;
}
```

### Custom Backoff Policy
```java
public class AdaptiveBackOffPolicy implements BackOffPolicy {

    private final MetricsService metricsService;

    @Override
    public BackOffContext start(RetryContext context) {
        return new AdaptiveBackOffContext();
    }

    @Override
    public void backOff(BackOffContext context) throws BackOffInterruptedException {
        AdaptiveBackOffContext ctx = (AdaptiveBackOffContext) context;

        // Adjust based on system load
        double load = metricsService.getSystemLoad();
        long baseDelay = 1000L;
        long delay = (long) (baseDelay * (1 + load) * ctx.getAttempt());

        try {
            Thread.sleep(Math.min(delay, 60000L));
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new BackOffInterruptedException("Interrupted", e);
        }

        ctx.incrementAttempt();
    }

    private static class AdaptiveBackOffContext implements BackOffContext {
        private int attempt = 1;

        public int getAttempt() {
            return attempt;
        }

        public void incrementAttempt() {
            attempt++;
        }
    }
}
```

## Policy Combinations

```java
@Configuration
public class ComprehensiveRetryConfig {

    @Bean
    public RetryTemplate productionRetryTemplate() {
        RetryTemplate template = new RetryTemplate();

        // Retry policy: max 5 attempts, only for specific exceptions
        Map<Class<? extends Throwable>, Boolean> exceptions = Map.of(
            TransientException.class, true,
            TimeoutException.class, true,
            IOException.class, true,
            IllegalStateException.class, false
        );
        SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy(5, exceptions);

        // Backoff: exponential with jitter
        ExponentialRandomBackOffPolicy backoff = new ExponentialRandomBackOffPolicy();
        backoff.setInitialInterval(500L);
        backoff.setMultiplier(2.0);
        backoff.setMaxInterval(10000L);

        template.setRetryPolicy(retryPolicy);
        template.setBackOffPolicy(backoff);

        // Add listener for monitoring
        template.registerListener(new RetryListenerSupport() {
            @Override
            public <T, E extends Throwable> void onError(
                    RetryContext context, RetryCallback<T, E> callback, Throwable t) {
                log.warn("Retry attempt {} failed: {}",
                    context.getRetryCount(), t.getMessage());
            }
        });

        return template;
    }
}
```

## Policy Factories

```java
@Component
public class RetryPolicyFactory {

    public RetryTemplate createForExternalApi(String apiName) {
        RetryTemplate template = new RetryTemplate();

        SimpleRetryPolicy policy = new SimpleRetryPolicy(3);
        ExponentialBackOffPolicy backoff = new ExponentialBackOffPolicy();
        backoff.setInitialInterval(1000L);
        backoff.setMultiplier(2.0);

        template.setRetryPolicy(policy);
        template.setBackOffPolicy(backoff);
        template.registerListener(new ApiRetryListener(apiName));

        return template;
    }

    public RetryTemplate createForDatabase() {
        RetryTemplate template = new RetryTemplate();

        Map<Class<? extends Throwable>, Boolean> exceptions = Map.of(
            TransientDataAccessException.class, true,
            DeadlockLoserDataAccessException.class, true
        );

        template.setRetryPolicy(new SimpleRetryPolicy(5, exceptions));
        template.setBackOffPolicy(new FixedBackOffPolicy() {{
            setBackOffPeriod(100L);
        }});

        return template;
    }

    public RetryTemplate createForMessaging() {
        RetryTemplate template = new RetryTemplate();

        template.setRetryPolicy(new TimeoutRetryPolicy(30000L));
        template.setBackOffPolicy(new ExponentialBackOffPolicy() {{
            setInitialInterval(500L);
            setMaxInterval(5000L);
        }});

        return template;
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Match policy to use case | One-size-fits-all approach |
| Use exponential backoff | Aggressive fixed intervals |
| Add jitter to prevent thundering herd | Synchronized retries |
| Combine timeout with attempts | Unlimited retries |
| Create policy factories | Duplicate configuration |
| Monitor retry metrics | Ignore retry patterns |

## Production Checklist

- [ ] Retry policy matches exception types
- [ ] Backoff prevents overload
- [ ] Timeout policy for bounded retries
- [ ] Composite policies for complex scenarios
- [ ] Custom policies for special cases
- [ ] Listeners for observability
