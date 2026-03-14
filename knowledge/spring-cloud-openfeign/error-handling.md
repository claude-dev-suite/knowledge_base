# Spring Cloud OpenFeign - Error Handling

## Error Handling Strategies

### 1. Error Decoder

```java
public class ServiceErrorDecoder implements ErrorDecoder {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public Exception decode(String methodKey, Response response) {
        try {
            if (response.body() == null) {
                return new FeignException.errorStatus(methodKey, response);
            }

            String body = StreamUtils.copyToString(
                response.body().asInputStream(),
                StandardCharsets.UTF_8
            );

            ErrorResponse error = objectMapper.readValue(body, ErrorResponse.class);

            return switch (response.status()) {
                case 400 -> new BadRequestException(error.getMessage(), error.getErrors());
                case 401 -> new UnauthorizedException(error.getMessage());
                case 403 -> new ForbiddenException(error.getMessage());
                case 404 -> new ResourceNotFoundException(error.getMessage());
                case 409 -> new ConflictException(error.getMessage());
                case 422 -> new ValidationException(error.getMessage(), error.getErrors());
                case 429 -> new TooManyRequestsException(error.getMessage());
                case 500, 502, 503, 504 -> new RetryableException(
                    response.status(),
                    error.getMessage(),
                    response.request().httpMethod(),
                    (Long) null,
                    response.request()
                );
                default -> new ServiceException(response.status(), error.getMessage());
            };

        } catch (IOException e) {
            return new FeignException.errorStatus(methodKey, response);
        }
    }
}
```

### 2. Circuit Breaker with Fallback

```java
@FeignClient(
    name = "user-service",
    fallback = UserClientFallback.class
)
public interface UserClient {

    @GetMapping("/api/users/{id}")
    User getUserById(@PathVariable("id") Long id);

    @GetMapping("/api/users")
    List<User> getAllUsers();
}

@Component
@Slf4j
public class UserClientFallback implements UserClient {

    @Override
    public User getUserById(Long id) {
        log.warn("Fallback: returning default user for id {}", id);
        return User.builder()
            .id(id)
            .name("Unknown User")
            .email("unknown@example.com")
            .build();
    }

    @Override
    public List<User> getAllUsers() {
        log.warn("Fallback: returning empty user list");
        return Collections.emptyList();
    }
}
```

### 3. Fallback Factory (with exception access)

```java
@FeignClient(
    name = "user-service",
    fallbackFactory = UserClientFallbackFactory.class
)
public interface UserClient {
    @GetMapping("/api/users/{id}")
    User getUserById(@PathVariable("id") Long id);
}

@Component
@Slf4j
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {

    @Override
    public UserClient create(Throwable cause) {
        return new UserClient() {
            @Override
            public User getUserById(Long id) {
                log.error("Error fetching user {}: {}", id, cause.getMessage());

                if (cause instanceof FeignException.NotFound) {
                    throw new UserNotFoundException(id);
                }
                if (cause instanceof FeignException.ServiceUnavailable) {
                    return getCachedUser(id);
                }

                return User.defaultUser(id);
            }
        };
    }

    private User getCachedUser(Long id) {
        // Return from cache
        return cacheService.getUser(id).orElse(User.defaultUser(id));
    }
}
```

## Circuit Breaker Configuration

```yaml
spring:
  cloud:
    openfeign:
      circuitbreaker:
        enabled: true
        alphanumeric-ids:
          enabled: true

resilience4j:
  circuitbreaker:
    instances:
      UserClientgetUserById:
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 3
        automatic-transition-from-open-to-half-open-enabled: true
      user-service:
        sliding-window-size: 20
        failure-rate-threshold: 60
        wait-duration-in-open-state: 30s

  timelimiter:
    instances:
      user-service:
        timeout-duration: 5s
```

## Exception Handling in Service

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderService {

    private final UserClient userClient;

    public Order createOrder(CreateOrderRequest request) {
        User user;
        try {
            user = userClient.getUserById(request.getUserId());
        } catch (ResourceNotFoundException e) {
            throw new OrderCreationException("User not found: " + request.getUserId());
        } catch (ServiceException e) {
            log.error("User service error", e);
            throw new ServiceUnavailableException("Unable to verify user");
        } catch (FeignException e) {
            log.error("Feign client error", e);
            throw new ServiceUnavailableException("Communication error with user service");
        }

        return createOrderForUser(user, request);
    }
}
```

## Retry with Error Handling

```java
@Configuration
public class FeignRetryConfig {

    @Bean
    public Retryer feignRetryer() {
        return new CustomRetryer();
    }
}

public class CustomRetryer implements Retryer {

    private final int maxAttempts = 3;
    private final long backoff = 1000;
    private int attempt = 1;

    @Override
    public void continueOrPropagate(RetryableException e) {
        if (attempt++ >= maxAttempts) {
            throw e;
        }

        // Only retry on server errors and timeouts
        if (!isRetryable(e)) {
            throw e;
        }

        try {
            Thread.sleep(backoff * attempt);
        } catch (InterruptedException ignored) {
            Thread.currentThread().interrupt();
            throw e;
        }
    }

    private boolean isRetryable(RetryableException e) {
        return e.status() >= 500 ||
               e.status() == -1 ||  // Connection error
               e.getCause() instanceof SocketTimeoutException;
    }

    @Override
    public Retryer clone() {
        return new CustomRetryer();
    }
}
```

## Global Exception Handler

```java
@RestControllerAdvice
@Slf4j
public class FeignExceptionHandler {

    @ExceptionHandler(FeignException.class)
    public ResponseEntity<ErrorResponse> handleFeignException(FeignException e) {
        log.error("Feign client error: {}", e.getMessage());

        HttpStatus status = HttpStatus.resolve(e.status());
        if (status == null) {
            status = HttpStatus.SERVICE_UNAVAILABLE;
        }

        return ResponseEntity.status(status)
            .body(new ErrorResponse(
                "SERVICE_ERROR",
                "Error communicating with external service",
                e.getMessage()
            ));
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("NOT_FOUND", e.getMessage(), null));
    }

    @ExceptionHandler(ServiceException.class)
    public ResponseEntity<ErrorResponse> handleServiceException(ServiceException e) {
        return ResponseEntity.status(HttpStatus.BAD_GATEWAY)
            .body(new ErrorResponse("SERVICE_ERROR", e.getMessage(), null));
    }
}
```

## Monitoring Failed Requests

```java
@Component
@Slf4j
public class FeignErrorLogger implements ErrorDecoder {

    private final MeterRegistry meterRegistry;
    private final ErrorDecoder delegate = new Default();

    @Override
    public Exception decode(String methodKey, Response response) {
        // Log error
        log.error("Feign error - method: {}, status: {}, reason: {}",
            methodKey, response.status(), response.reason());

        // Record metric
        meterRegistry.counter("feign.error",
            "method", methodKey,
            "status", String.valueOf(response.status())
        ).increment();

        return delegate.decode(methodKey, response);
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Implement fallbacks for critical calls | Let all errors propagate |
| Use FallbackFactory for exception access | Use simple fallback for debugging |
| Configure circuit breakers | Allow unlimited retries |
| Log errors with context | Swallow exceptions silently |
| Return safe defaults in fallbacks | Return null from fallbacks |
| Handle specific exception types | Catch generic Exception |
