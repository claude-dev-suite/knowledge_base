# Spring Cloud Gateway - Filters

## Filter Types

- **Pre-filters**: Execute before routing to downstream service
- **Post-filters**: Execute after receiving response from downstream service
- **Global filters**: Apply to all routes
- **Gateway filters**: Apply to specific routes

## Built-in Filters

### Request Modification

```yaml
filters:
  # Add header to downstream request
  - AddRequestHeader=X-Request-Source, Gateway

  # Add header from URI variable
  - AddRequestHeadersIfNotPresent=X-Request-Id:{requestId}

  # Remove header
  - RemoveRequestHeader=X-Sensitive-Header

  # Set header (replace if exists)
  - SetRequestHeader=X-Request-Version, v2

  # Add query parameter
  - AddRequestParameter=source, gateway

  # Remove query parameter
  - RemoveRequestParameter=debug
```

### Response Modification

```yaml
filters:
  # Add header to response
  - AddResponseHeader=X-Response-Time, ${responseTime}

  # Remove header from response
  - RemoveResponseHeader=Server

  # Set response header
  - SetResponseHeader=Cache-Control, max-age=3600

  # Modify response body
  - ModifyResponseBody=application/json, application/json, MyBodyModifier
```

### Path Manipulation

```yaml
filters:
  # Strip prefix: /api/orders/123 → /orders/123
  - StripPrefix=1

  # Add prefix: /orders/123 → /api/v2/orders/123
  - PrefixPath=/api/v2

  # Rewrite path: /api/orders/123 → /orders/api/123
  - RewritePath=/api/(?<segment>.*), /$\{segment}/api

  # Set path
  - SetPath=/orders/{orderId}
```

### Status Modification

```yaml
filters:
  # Set response status
  - SetStatus=401

  # Map status
  - MapRequestHeader=status, X-Status
```

### Circuit Breaker

```yaml
filters:
  - name: CircuitBreaker
    args:
      name: myCircuitBreaker
      fallbackUri: forward:/fallback
      statusCodes:
        - 500
        - 503
```

### Rate Limiting

```yaml
filters:
  - name: RequestRateLimiter
    args:
      redis-rate-limiter.replenishRate: 10
      redis-rate-limiter.burstCapacity: 20
      redis-rate-limiter.requestedTokens: 1
      key-resolver: "#{@userKeyResolver}"
```

### Retry

```yaml
filters:
  - name: Retry
    args:
      retries: 3
      statuses: BAD_GATEWAY,SERVICE_UNAVAILABLE
      methods: GET,POST
      backoff:
        firstBackoff: 100ms
        maxBackoff: 500ms
        factor: 2
        basedOnPreviousValue: true
```

### Request Size

```yaml
filters:
  - name: RequestSize
    args:
      maxSize: 5MB
```

## Custom Filter

### Gateway Filter Factory

```java
@Component
public class LoggingGatewayFilterFactory
    extends AbstractGatewayFilterFactory<LoggingGatewayFilterFactory.Config> {

    private static final Logger log = LoggerFactory.getLogger(LoggingGatewayFilterFactory.class);

    public LoggingGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            // Pre-filter
            ServerHttpRequest request = exchange.getRequest();
            log.info("[{}] {} {}", config.prefix, request.getMethod(), request.getURI());

            long startTime = System.currentTimeMillis();

            return chain.filter(exchange)
                .then(Mono.fromRunnable(() -> {
                    // Post-filter
                    long duration = System.currentTimeMillis() - startTime;
                    ServerHttpResponse response = exchange.getResponse();
                    log.info("[{}] {} {} - {} ({}ms)",
                        config.prefix,
                        request.getMethod(),
                        request.getURI(),
                        response.getStatusCode(),
                        duration);
                }));
        };
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return List.of("prefix");
    }

    @Data
    public static class Config {
        private String prefix = "GATEWAY";
    }
}
```

Usage:
```yaml
filters:
  - Logging=API
```

### Global Filter

```java
@Component
@Order(-1)  // Execute first
@Slf4j
public class RequestTracingFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String traceId = Optional.ofNullable(
                exchange.getRequest().getHeaders().getFirst("X-Trace-Id"))
            .orElse(UUID.randomUUID().toString());

        ServerHttpRequest request = exchange.getRequest().mutate()
            .header("X-Trace-Id", traceId)
            .build();

        ServerWebExchange mutatedExchange = exchange.mutate()
            .request(request)
            .build();

        return chain.filter(mutatedExchange)
            .then(Mono.fromRunnable(() -> {
                exchange.getResponse().getHeaders().add("X-Trace-Id", traceId);
            }));
    }
}
```

## Authentication Filter

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter implements GlobalFilter, Ordered {

    private final JwtService jwtService;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();

        // Skip public endpoints
        if (isPublicEndpoint(request.getPath().value())) {
            return chain.filter(exchange);
        }

        String authHeader = request.getHeaders().getFirst(HttpHeaders.AUTHORIZATION);

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return onError(exchange, HttpStatus.UNAUTHORIZED);
        }

        String token = authHeader.substring(7);

        try {
            Claims claims = jwtService.validateToken(token);

            ServerHttpRequest mutatedRequest = request.mutate()
                .header("X-User-Id", claims.getSubject())
                .header("X-User-Roles", String.join(",", claims.get("roles", List.class)))
                .build();

            return chain.filter(exchange.mutate().request(mutatedRequest).build());

        } catch (JwtException e) {
            return onError(exchange, HttpStatus.UNAUTHORIZED);
        }
    }

    private Mono<Void> onError(ServerWebExchange exchange, HttpStatus status) {
        exchange.getResponse().setStatusCode(status);
        return exchange.getResponse().setComplete();
    }

    @Override
    public int getOrder() {
        return -100;  // Execute early
    }
}
```

## Response Body Modification

```java
@Component
public class ResponseModificationFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return chain.filter(exchange.mutate()
            .response(new ServerHttpResponseDecorator(exchange.getResponse()) {
                @Override
                public Mono<Void> writeWith(Publisher<? extends DataBuffer> body) {
                    if (body instanceof Flux) {
                        Flux<? extends DataBuffer> fluxBody = (Flux<? extends DataBuffer>) body;

                        return super.writeWith(fluxBody.buffer().map(dataBuffers -> {
                            // Combine all buffers
                            DataBufferFactory bufferFactory = exchange.getResponse().bufferFactory();
                            DataBuffer join = bufferFactory.join(dataBuffers);

                            byte[] content = new byte[join.readableByteCount()];
                            join.read(content);
                            DataBufferUtils.release(join);

                            // Modify content
                            String responseBody = new String(content, StandardCharsets.UTF_8);
                            String modifiedBody = wrapResponse(responseBody);

                            return bufferFactory.wrap(modifiedBody.getBytes());
                        }));
                    }
                    return super.writeWith(body);
                }
            }).build());
    }

    @Override
    public int getOrder() {
        return -2;
    }
}
```

## Key Resolver for Rate Limiting

```java
@Configuration
public class RateLimiterConfig {

    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.justOrEmpty(
            exchange.getRequest().getHeaders().getFirst("X-User-Id"))
            .defaultIfEmpty("anonymous");
    }

    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            Optional.ofNullable(exchange.getRequest().getRemoteAddress())
                .map(InetSocketAddress::getHostString)
                .orElse("unknown"));
    }

    @Bean
    public KeyResolver apiKeyResolver() {
        return exchange -> Mono.justOrEmpty(
            exchange.getRequest().getHeaders().getFirst("X-Api-Key"));
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Order filters appropriately | Ignore filter order |
| Use built-in filters when possible | Reinvent the wheel |
| Handle errors in custom filters | Let exceptions propagate |
| Log filter operations | Lose observability |
| Test filters in isolation | Skip filter unit tests |
| Keep filters focused | Create monolithic filters |
