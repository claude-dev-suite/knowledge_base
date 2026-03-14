# Spring Cloud OpenFeign - Configuration

## Global Configuration

### application.yml
```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            # Timeouts
            connect-timeout: 5000
            read-timeout: 10000

            # Logging
            logger-level: basic

            # Retry
            retryer: feign.Retryer.Default

            # Error decoder
            error-decoder: com.example.CustomErrorDecoder

            # Request interceptors
            request-interceptors:
              - com.example.AuthRequestInterceptor

            # Encoder/Decoder
            encoder: feign.jackson.JacksonEncoder
            decoder: feign.jackson.JacksonDecoder

      # Compression
      compression:
        request:
          enabled: true
          mime-types: application/json
          min-request-size: 2048
        response:
          enabled: true
```

### Per-Client Configuration
```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          user-service:
            connect-timeout: 3000
            read-timeout: 5000
            logger-level: full
          payment-service:
            connect-timeout: 10000
            read-timeout: 30000
            logger-level: basic
```

## Java Configuration

```java
@Configuration
public class FeignConfig {

    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }

    @Bean
    public Retryer feignRetryer() {
        return new Retryer.Default(1000, 5000, 3);
    }

    @Bean
    public ErrorDecoder errorDecoder() {
        return new CustomErrorDecoder();
    }

    @Bean
    public Request.Options requestOptions() {
        return new Request.Options(
            5, TimeUnit.SECONDS,    // Connect timeout
            10, TimeUnit.SECONDS,   // Read timeout
            true                    // Follow redirects
        );
    }
}
```

### Client-Specific Configuration
```java
public class UserClientConfig {

    @Bean
    public Logger.Level loggerLevel() {
        return Logger.Level.BASIC;
    }

    @Bean
    public Request.Options options() {
        return new Request.Options(3000, 5000);
    }
}

@FeignClient(name = "user-service", configuration = UserClientConfig.class)
public interface UserClient {
    // ...
}
```

## Request Interceptors

### Authentication Interceptor
```java
@Component
public class AuthRequestInterceptor implements RequestInterceptor {

    @Value("${service.api-key}")
    private String apiKey;

    @Override
    public void apply(RequestTemplate template) {
        template.header("X-Api-Key", apiKey);
    }
}
```

### JWT Token Interceptor
```java
@Component
@RequiredArgsConstructor
public class JwtRequestInterceptor implements RequestInterceptor {

    private final TokenProvider tokenProvider;

    @Override
    public void apply(RequestTemplate template) {
        String token = tokenProvider.getToken();
        if (token != null) {
            template.header("Authorization", "Bearer " + token);
        }
    }
}
```

### Tracing Interceptor
```java
@Component
public class TracingInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        String traceId = MDC.get("traceId");
        String spanId = MDC.get("spanId");

        if (traceId != null) {
            template.header("X-Trace-Id", traceId);
        }
        if (spanId != null) {
            template.header("X-Span-Id", spanId);
        }
        template.header("X-Request-Time", Instant.now().toString());
    }
}
```

## Error Decoder

### Custom Error Decoder
```java
public class CustomErrorDecoder implements ErrorDecoder {

    private final ErrorDecoder defaultDecoder = new ErrorDecoder.Default();

    @Override
    public Exception decode(String methodKey, Response response) {
        if (response.status() >= 400 && response.status() < 500) {
            return handleClientError(methodKey, response);
        }
        if (response.status() >= 500) {
            return handleServerError(methodKey, response);
        }
        return defaultDecoder.decode(methodKey, response);
    }

    private Exception handleClientError(String methodKey, Response response) {
        return switch (response.status()) {
            case 400 -> new BadRequestException(readBody(response));
            case 401 -> new UnauthorizedException("Authentication required");
            case 403 -> new ForbiddenException("Access denied");
            case 404 -> new ResourceNotFoundException(readBody(response));
            case 422 -> new ValidationException(readBody(response));
            default -> new ClientException(response.status(), readBody(response));
        };
    }

    private Exception handleServerError(String methodKey, Response response) {
        // Return RetryableException to trigger retry
        return new RetryableException(
            response.status(),
            "Server error: " + readBody(response),
            response.request().httpMethod(),
            (Long) null,
            response.request()
        );
    }

    private String readBody(Response response) {
        try {
            if (response.body() != null) {
                return StreamUtils.copyToString(
                    response.body().asInputStream(),
                    StandardCharsets.UTF_8
                );
            }
        } catch (IOException e) {
            // ignore
        }
        return "";
    }
}
```

## Custom Encoder/Decoder

### Jackson Configuration
```java
@Configuration
public class FeignJacksonConfig {

    @Bean
    public Encoder feignEncoder(ObjectMapper objectMapper) {
        return new JacksonEncoder(objectMapper);
    }

    @Bean
    public Decoder feignDecoder(ObjectMapper objectMapper) {
        return new JacksonDecoder(objectMapper);
    }

    @Bean
    public ObjectMapper feignObjectMapper() {
        return new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            .setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
    }
}
```

## Retry Configuration

### Spring Retry
```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

```java
@Bean
public Retryer feignRetryer() {
    // period: 1s, maxPeriod: 5s, maxAttempts: 3
    return new Retryer.Default(1000, 5000, 3);
}
```

### No Retry
```java
@Bean
public Retryer feignRetryer() {
    return Retryer.NEVER_RETRY;
}
```

## Micrometer Integration

```yaml
spring:
  cloud:
    openfeign:
      micrometer:
        enabled: true
```

### Metrics Available
- `feign.client.requests` - Request count and timing
- `feign.client.response.size` - Response body size
- `feign.client.request.size` - Request body size

## OkHttp/Apache HttpClient

### OkHttp
```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-okhttp</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    openfeign:
      okhttp:
        enabled: true
```

### Apache HttpClient 5
```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-hc5</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    openfeign:
      httpclient:
        hc5:
          enabled: true
```

## Connection Pooling

```yaml
spring:
  cloud:
    openfeign:
      httpclient:
        max-connections: 200
        max-connections-per-route: 50
        connection-timeout: 2000
        time-to-live: 900
        time-to-live-unit: seconds
```

## Best Practices

| Do | Don't |
|----|-------|
| Configure timeouts explicitly | Rely on defaults |
| Use connection pooling | Create new connections per request |
| Implement error decoder | Ignore error responses |
| Add request interceptors for auth | Hardcode credentials |
| Enable compression for large payloads | Transfer uncompressed data |
| Configure retry for transient failures | Retry all errors |
