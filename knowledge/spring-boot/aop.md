# Spring AOP Reference

## Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

## Basic Aspect

```java
@Aspect
@Component
public class LoggingAspect {

    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        log.info("Calling: {}", joinPoint.getSignature().getName());
    }

    @After("execution(* com.example.service.*.*(..))")
    public void logAfter(JoinPoint joinPoint) {
        log.info("Completed: {}", joinPoint.getSignature().getName());
    }
}
```

## Pointcut Expressions

```java
@Aspect
@Component
public class PointcutExamples {

    // All methods in service package
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}

    // Methods with specific annotation
    @Pointcut("@annotation(com.example.annotation.Loggable)")
    public void loggableMethods() {}

    // Methods in classes with annotation
    @Pointcut("@within(org.springframework.stereotype.Service)")
    public void serviceBeans() {}

    // Public methods only
    @Pointcut("execution(public * *(..))")
    public void publicMethods() {}

    // Methods with specific return type
    @Pointcut("execution(java.util.List com.example..*.*(..))")
    public void listReturningMethods() {}

    // Combined pointcuts
    @Pointcut("serviceLayer() && publicMethods()")
    public void publicServiceMethods() {}
}
```

## Advice Types

```java
@Aspect
@Component
public class ComprehensiveAspect {

    // Before method execution
    @Before("execution(* com.example.service.*.*(..))")
    public void beforeAdvice(JoinPoint jp) {
        log.info("Before: {}", jp.getSignature());
    }

    // After method returns successfully
    @AfterReturning(pointcut = "execution(* com.example.service.*.*(..))",
                    returning = "result")
    public void afterReturning(JoinPoint jp, Object result) {
        log.info("Returned: {}", result);
    }

    // After method throws exception
    @AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))",
                   throwing = "ex")
    public void afterThrowing(JoinPoint jp, Exception ex) {
        log.error("Exception in {}: {}", jp.getSignature(), ex.getMessage());
    }

    // After method completes (success or exception)
    @After("execution(* com.example.service.*.*(..))")
    public void afterAdvice(JoinPoint jp) {
        log.info("Completed: {}", jp.getSignature());
    }

    // Around: full control
    @Around("execution(* com.example.service.*.*(..))")
    public Object aroundAdvice(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            long duration = System.currentTimeMillis() - start;
            log.info("{} took {}ms", pjp.getSignature(), duration);
        }
    }
}
```

## Custom Annotation

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Timed {
    String value() default "";
}

@Aspect
@Component
public class TimingAspect {

    @Around("@annotation(timed)")
    public Object measureTime(ProceedingJoinPoint pjp, Timed timed) throws Throwable {
        String name = timed.value().isEmpty()
            ? pjp.getSignature().getName()
            : timed.value();

        long start = System.nanoTime();
        try {
            return pjp.proceed();
        } finally {
            long duration = System.nanoTime() - start;
            log.info("{}: {}ms", name, duration / 1_000_000);
        }
    }
}

// Usage
@Service
public class OrderService {

    @Timed("order-creation")
    public Order createOrder(OrderRequest request) { }
}
```

## Accessing Arguments

```java
@Aspect
@Component
public class ArgumentAccessAspect {

    // Access all arguments
    @Before("execution(* com.example.service.*.*(..))")
    public void logArguments(JoinPoint jp) {
        Object[] args = jp.getArgs();
        log.info("Arguments: {}", Arrays.toString(args));
    }

    // Bind specific argument by type
    @Before("execution(* com.example.service.*.*(..)) && args(request,..)")
    public void logRequest(CreateOrderRequest request) {
        log.info("Request: {}", request);
    }

    // Bind by name
    @Before("execution(* com.example.service.UserService.findById(Long)) && args(userId)")
    public void logUserId(Long userId) {
        log.info("Finding user: {}", userId);
    }
}
```

## Modifying Arguments

```java
@Aspect
@Component
public class SanitizationAspect {

    @Around("execution(* com.example.service.*.*(String)) && args(input)")
    public Object sanitize(ProceedingJoinPoint pjp, String input) throws Throwable {
        String sanitized = HtmlUtils.htmlEscape(input);
        return pjp.proceed(new Object[]{sanitized});
    }
}
```

## Exception Handling

```java
@Aspect
@Component
public class ExceptionHandlingAspect {

    @AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))",
                   throwing = "ex")
    public void handleException(JoinPoint jp, Exception ex) {
        log.error("Exception in {}.{}: {}",
            jp.getTarget().getClass().getSimpleName(),
            jp.getSignature().getName(),
            ex.getMessage());
        metricsService.incrementError(jp.getSignature().getName());
    }

    @Around("execution(* com.example.service.*.*(..))")
    public Object wrapException(ProceedingJoinPoint pjp) throws Throwable {
        try {
            return pjp.proceed();
        } catch (DataAccessException ex) {
            throw new ServiceException("Database error", ex);
        }
    }
}
```

## Aspect Ordering

```java
@Aspect
@Component
@Order(1)  // Lower value = higher priority
public class SecurityAspect { }

@Aspect
@Component
@Order(2)
public class LoggingAspect { }

@Aspect
@Component
@Order(3)
public class CachingAspect { }
```

## Retry Aspect

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Retryable {
    int maxAttempts() default 3;
    long delay() default 1000;
}

@Aspect
@Component
public class RetryAspect {

    @Around("@annotation(retryable)")
    public Object retry(ProceedingJoinPoint pjp, Retryable retryable) throws Throwable {
        int attempts = 0;
        Exception lastException = null;

        while (attempts < retryable.maxAttempts()) {
            try {
                return pjp.proceed();
            } catch (Exception e) {
                lastException = e;
                attempts++;
                if (attempts < retryable.maxAttempts()) {
                    Thread.sleep(retryable.delay());
                }
            }
        }
        throw lastException;
    }
}
```

## Testing Aspects

```java
@SpringBootTest
class LoggingAspectTest {

    @Autowired
    private UserService userService;

    @SpyBean
    private LoggingAspect loggingAspect;

    @Test
    void shouldLogMethodCall() {
        userService.findById(1L);

        verify(loggingAspect).logBefore(any(JoinPoint.class));
    }
}
```

## Limitations

- Only works with Spring-managed beans
- Self-invocation bypasses aspects
- Final methods cannot be advised
- Private methods cannot be advised
