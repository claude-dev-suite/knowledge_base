# Spring Cloud Gateway - Routes

## Route Components

A route consists of:
1. **ID**: Unique identifier
2. **URI**: Destination service
3. **Predicates**: Conditions to match
4. **Filters**: Transformations to apply

## Predicates

### Path Predicate
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: path-example
          uri: http://example.com
          predicates:
            - Path=/api/orders/**
            # Matches: /api/orders, /api/orders/123, /api/orders/123/items
```

### Method Predicate
```yaml
- id: method-example
  uri: http://example.com
  predicates:
    - Method=GET,POST,PUT
```

### Header Predicate
```yaml
- id: header-example
  uri: http://example.com
  predicates:
    - Header=X-Request-Id, \d+  # Regex match
    - Header=Authorization      # Header exists
```

### Query Predicate
```yaml
- id: query-example
  uri: http://example.com
  predicates:
    - Query=page           # Parameter exists
    - Query=sort, ^(asc|desc)$  # With value regex
```

### Host Predicate
```yaml
- id: host-example
  uri: http://example.com
  predicates:
    - Host=**.example.com,api.example.org
```

### Cookie Predicate
```yaml
- id: cookie-example
  uri: http://example.com
  predicates:
    - Cookie=session-id, .*
```

### Time-based Predicates
```yaml
- id: after-example
  uri: http://example.com
  predicates:
    - After=2024-01-01T00:00:00+00:00[UTC]

- id: before-example
  uri: http://example.com
  predicates:
    - Before=2025-12-31T23:59:59+00:00[UTC]

- id: between-example
  uri: http://example.com
  predicates:
    - Between=2024-01-01T00:00:00+00:00[UTC], 2024-12-31T23:59:59+00:00[UTC]
```

### Remote Address Predicate
```yaml
- id: remote-addr-example
  uri: http://example.com
  predicates:
    - RemoteAddr=192.168.1.0/24,10.0.0.0/8
```

### Weight Predicate (Canary/Blue-Green)
```yaml
- id: service-v1
  uri: http://v1.example.com
  predicates:
    - Path=/api/**
    - Weight=service-group, 80  # 80% traffic

- id: service-v2
  uri: http://v2.example.com
  predicates:
    - Path=/api/**
    - Weight=service-group, 20  # 20% traffic
```

## Combining Predicates

### AND Logic (Default)
```yaml
- id: combined-and
  uri: http://example.com
  predicates:
    - Path=/api/**
    - Method=GET
    - Header=X-Api-Key
    # All must match
```

### Java Configuration (Complex Logic)
```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("complex", r -> r
            .path("/api/**")
            .and()
            .method(HttpMethod.GET)
            .or()
            .path("/public/**")
            .uri("http://example.com"))
        .build();
}
```

## Custom Predicate

```java
@Component
public class CustomRoutePredicateFactory
    extends AbstractRoutePredicateFactory<CustomRoutePredicateFactory.Config> {

    public CustomRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return exchange -> {
            String header = exchange.getRequest()
                .getHeaders()
                .getFirst(config.headerName);
            return config.values.contains(header);
        };
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return List.of("headerName", "values");
    }

    @Data
    public static class Config {
        private String headerName;
        private List<String> values;
    }
}
```

Usage:
```yaml
predicates:
  - Custom=X-Tenant-Id, tenant1, tenant2
```

## Dynamic Routes

### From Service Discovery
```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
          # Automatically creates routes: /SERVICE-NAME/**
```

### Route Definition Locator
```java
@Component
@RequiredArgsConstructor
public class DatabaseRouteDefinitionLocator implements RouteDefinitionLocator {

    private final RouteRepository routeRepository;

    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return Flux.fromIterable(routeRepository.findAll())
            .map(this::toRouteDefinition);
    }

    private RouteDefinition toRouteDefinition(RouteEntity entity) {
        RouteDefinition route = new RouteDefinition();
        route.setId(entity.getId());
        route.setUri(URI.create(entity.getUri()));
        route.setPredicates(parsePredicates(entity.getPredicates()));
        route.setFilters(parseFilters(entity.getFilters()));
        return route;
    }
}
```

### Refresh Routes
```java
@RestController
@RequiredArgsConstructor
public class RouteRefreshController {

    private final ApplicationEventPublisher publisher;

    @PostMapping("/routes/refresh")
    public void refreshRoutes() {
        publisher.publishEvent(new RefreshRoutesEvent(this));
    }
}
```

## Route Metadata

```yaml
- id: orders-service
  uri: lb://orders-service
  predicates:
    - Path=/api/orders/**
  metadata:
    connect-timeout: 1000
    response-timeout: 5000
    cors:
      allowedOrigins: "*"
      allowedMethods:
        - GET
        - POST
```

Access in filters:
```java
Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
Map<String, Object> metadata = route.getMetadata();
Integer timeout = (Integer) metadata.get("response-timeout");
```

## Best Practices

| Do | Don't |
|----|-------|
| Use specific predicates | Use overly broad paths |
| Order routes by specificity | Let order be random |
| Use metadata for configuration | Hardcode values in filters |
| Test routing thoroughly | Deploy without route tests |
| Version your API routes | Break clients with changes |
