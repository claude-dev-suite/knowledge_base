# Spring Cloud Gateway - Basics

## Overview

Spring Cloud Gateway is an API Gateway built on Spring WebFlux, providing routing, filtering, and load balancing capabilities for microservices architectures.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        Spring Cloud Gateway                               │
│                                                                          │
│  ┌─────────┐    ┌──────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │ Client  │───▶│  Predicates  │───▶│   Filters   │───▶│  Upstream   │  │
│  │ Request │    │  (Routing)   │    │  (Pre/Post) │    │   Service   │  │
│  └─────────┘    └──────────────┘    └─────────────┘    └─────────────┘  │
│                                                                          │
│  Route = Predicate + Filters + URI                                       │
└──────────────────────────────────────────────────────────────────────────┘
```

## Basic Configuration

### application.yml
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: orders-service
          uri: http://localhost:8081
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1

        - id: users-service
          uri: http://localhost:8082
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1

        - id: products-service
          uri: lb://products-service  # Load balanced
          predicates:
            - Path=/api/products/**
```

### Java Configuration
```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("orders-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .addRequestHeader("X-Gateway", "true"))
                .uri("http://localhost:8081"))

            .route("users-service", r -> r
                .path("/api/users/**")
                .and()
                .method(HttpMethod.GET, HttpMethod.POST)
                .filters(f -> f
                    .stripPrefix(1)
                    .circuitBreaker(c -> c
                        .setName("users-cb")
                        .setFallbackUri("forward:/fallback/users")))
                .uri("lb://users-service"))
            .build();
    }
}
```

## Predicates

| Predicate | Description |
|-----------|-------------|
| Path | Match request path |
| Method | Match HTTP method |
| Header | Match request header |
| Query | Match query parameter |
| Host | Match hostname |
| Cookie | Match cookie value |
| After/Before/Between | Match by time |
| RemoteAddr | Match client IP |
| Weight | Traffic distribution |

## Common Filters

| Filter | Description |
|--------|-------------|
| AddRequestHeader | Add header to downstream request |
| AddResponseHeader | Add header to response |
| StripPrefix | Remove path prefix |
| RewritePath | Rewrite request path |
| CircuitBreaker | Apply circuit breaker |
| RequestRateLimiter | Rate limit requests |
| Retry | Retry failed requests |

## Load Balancing

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: service-lb
          uri: lb://my-service  # Discovery-based
          predicates:
            - Path=/api/**
```

## Global Configuration

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - AddRequestHeader=X-Request-Source, Gateway
        - AddResponseHeader=X-Response-Time, ${responseTime}
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins: "https://example.com"
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
            allowedHeaders: "*"
            exposedHeaders: "X-Request-Id"
            allowCredentials: true
            maxAge: 3600
```

## Best Practices

| Do | Don't |
|----|-------|
| Use service discovery for URIs | Hardcode service URLs |
| Implement circuit breakers | Let failures cascade |
| Configure rate limiting | Allow unbounded requests |
| Add request tracing | Lose observability |
| Configure CORS properly | Use wildcard CORS |
| Use HTTPS in production | Expose services unencrypted |

## Production Checklist

- [ ] HTTPS configured
- [ ] Circuit breakers enabled
- [ ] Rate limiting configured
- [ ] CORS properly restricted
- [ ] Health checks enabled
- [ ] Metrics and tracing configured
- [ ] Authentication/authorization set up
- [ ] Timeout configurations tuned
