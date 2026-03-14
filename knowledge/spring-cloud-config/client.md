# Spring Cloud Config - Client

## Client Setup

### Dependencies
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

## Configuration

### Spring Boot 2.4+ (application.yml)
```yaml
spring:
  application:
    name: order-service
  profiles:
    active: dev
  config:
    import: optional:configserver:http://localhost:8888
  cloud:
    config:
      fail-fast: true
      retry:
        initial-interval: 1000
        max-attempts: 6
        max-interval: 2000
        multiplier: 1.1
```

### Legacy (bootstrap.yml)
```yaml
spring:
  application:
    name: order-service
  profiles:
    active: dev
  cloud:
    config:
      uri: http://localhost:8888
      fail-fast: true
      retry:
        enabled: true
        initial-interval: 1000
        max-attempts: 6
```

## Service Discovery Integration

```yaml
spring:
  cloud:
    config:
      discovery:
        enabled: true
        service-id: config-server
      fail-fast: true

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

## Reading Configuration

### @Value
```java
@Service
public class OrderService {

    @Value("${order.max-items:100}")
    private int maxItems;

    @Value("${order.processing.timeout:30s}")
    private Duration timeout;

    @Value("${order.enabled:true}")
    private boolean enabled;
}
```

### @ConfigurationProperties
```java
@Configuration
@EnableConfigurationProperties(OrderProperties.class)
public class OrderConfig {
}

@ConfigurationProperties(prefix = "order")
@Validated
public class OrderProperties {

    @NotNull
    private int maxItems = 100;

    @NotNull
    private Duration processingTimeout = Duration.ofSeconds(30);

    private boolean enabled = true;

    @Valid
    private Retry retry = new Retry();

    // getters and setters

    public static class Retry {
        private int maxAttempts = 3;
        private Duration initialInterval = Duration.ofSeconds(1);
        // getters and setters
    }
}
```

## Dynamic Refresh

### Enable Actuator Refresh
```yaml
management:
  endpoints:
    web:
      exposure:
        include: refresh, health, info
```

### @RefreshScope
```java
@RestController
@RefreshScope  // Bean recreated on refresh
public class OrderController {

    @Value("${order.discount-percentage:0}")
    private int discountPercentage;

    @GetMapping("/orders/discount")
    public int getDiscount() {
        return discountPercentage;
    }
}
```

### Trigger Refresh
```bash
# Manual refresh
curl -X POST http://localhost:8080/actuator/refresh

# Response shows changed properties
["order.discount-percentage", "order.max-items"]
```

### Programmatic Refresh
```java
@Service
@RequiredArgsConstructor
public class ConfigRefreshService {

    private final ContextRefresher contextRefresher;

    public Set<String> refresh() {
        return contextRefresher.refresh();
    }
}
```

## Bus Refresh (Multi-Instance)

### Dependencies
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

### Configuration
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
```

### Trigger Bus Refresh
```bash
# Refresh all instances
curl -X POST http://localhost:8888/actuator/busrefresh

# Refresh specific service
curl -X POST http://localhost:8888/actuator/busrefresh/order-service:**
```

### Listen to Refresh Events
```java
@Component
@Slf4j
public class RefreshEventListener {

    @EventListener
    public void onRefreshScope(RefreshScopeRefreshedEvent event) {
        log.info("Configuration refreshed: {}", event.getName());
    }

    @EventListener
    public void onEnvironmentChange(EnvironmentChangeEvent event) {
        log.info("Environment changed, keys: {}", event.getKeys());
    }
}
```

## Fail-Fast and Retry

### Fail-Fast Configuration
```yaml
spring:
  cloud:
    config:
      fail-fast: true  # Fail startup if config server unavailable
      retry:
        enabled: true
        initial-interval: 1000
        max-attempts: 6
        max-interval: 10000
        multiplier: 1.5
```

### With Circuit Breaker
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

## Authentication

### Basic Auth
```yaml
spring:
  cloud:
    config:
      uri: http://localhost:8888
      username: configuser
      password: ${CONFIG_PASSWORD}
```

### OAuth2 Token
```yaml
spring:
  cloud:
    config:
      uri: http://localhost:8888
      headers:
        Authorization: Bearer ${CONFIG_TOKEN}
```

### Custom Headers
```java
@Configuration
public class ConfigClientConfig {

    @Bean
    public ConfigClientRequestTemplateFactory configClientRequestTemplateFactory() {
        return new ConfigClientRequestTemplateFactory() {
            @Override
            public RestTemplate create() {
                RestTemplate template = new RestTemplate();
                template.getInterceptors().add((request, body, execution) -> {
                    request.getHeaders().add("X-Custom-Header", "value");
                    return execution.execute(request, body);
                });
                return template;
            }
        };
    }
}
```

## Local Fallback

### Fallback Properties
```yaml
# application.yml (local fallback)
spring:
  config:
    import:
      - optional:configserver:http://localhost:8888
      - classpath:fallback-config.yml

# fallback-config.yml
order:
  max-items: 50
  enabled: true
```

### Profile-Based Fallback
```yaml
spring:
  config:
    import:
      - optional:configserver:http://localhost:8888
    on-profile-not-active:
      - classpath:local-config.yml
```

## Configuration Reference

```yaml
spring:
  application:
    name: my-service
  profiles:
    active: dev
  config:
    import: optional:configserver:http://localhost:8888
  cloud:
    config:
      # Server connection
      uri: http://localhost:8888
      username: configuser
      password: ${CONFIG_PASSWORD}

      # Behavior
      fail-fast: true
      enabled: true

      # Label (Git branch)
      label: main

      # Profile override
      profile: custom-profile

      # Retry
      retry:
        enabled: true
        initial-interval: 1000
        max-attempts: 6
        max-interval: 10000
        multiplier: 1.5

      # Discovery
      discovery:
        enabled: false
        service-id: config-server

      # Request customization
      request-connect-timeout: 1000
      request-read-timeout: 5000
```

## Best Practices

| Do | Don't |
|----|-------|
| Use @RefreshScope for dynamic configs | Restart for config changes |
| Configure fail-fast in production | Silently fail on config errors |
| Use bus refresh for multiple instances | Manual refresh per instance |
| Implement fallback configuration | Fail without fallback |
| Validate configuration at startup | Discover issues at runtime |
| Use typed @ConfigurationProperties | Scattered @Value annotations |
