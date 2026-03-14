# Spring Boot Actuator Reference

## Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

## Configuration

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,env,loggers
      base-path: /actuator
  endpoint:
    health:
      show-details: when_authorized
      show-components: always
  info:
    env:
      enabled: true
```

## Built-in Endpoints

| Endpoint | Description |
|----------|-------------|
| `/actuator/health` | Application health |
| `/actuator/info` | Application info |
| `/actuator/metrics` | Metrics data |
| `/actuator/env` | Environment properties |
| `/actuator/loggers` | Logger configuration |
| `/actuator/beans` | Spring beans |
| `/actuator/mappings` | Request mappings |
| `/actuator/prometheus` | Prometheus metrics |

## Health Indicators

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    private final DataSource dataSource;

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(1)) {
                return Health.up()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("status", "Connected")
                    .build();
            }
        } catch (SQLException e) {
            return Health.down()
                .withException(e)
                .build();
        }
        return Health.down().build();
    }
}
```

## Reactive Health Indicator

```java
@Component
public class ExternalApiHealthIndicator implements ReactiveHealthIndicator {

    private final WebClient webClient;

    @Override
    public Mono<Health> health() {
        return webClient.get()
            .uri("/health")
            .retrieve()
            .toBodilessEntity()
            .map(response -> Health.up().build())
            .onErrorResume(e -> Mono.just(
                Health.down().withException(e).build()
            ))
            .timeout(Duration.ofSeconds(3));
    }
}
```

## Health Groups

```yaml
management:
  endpoint:
    health:
      group:
        readiness:
          include: db,redis,kafka
        liveness:
          include: ping
```

## Custom Metrics

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final MeterRegistry meterRegistry;
    private final Counter ordersCreated;

    public OrderService(MeterRegistry registry) {
        this.meterRegistry = registry;
        this.ordersCreated = Counter.builder("orders.created")
            .description("Total orders created")
            .tag("type", "all")
            .register(registry);
    }

    public Order createOrder(OrderRequest request) {
        Order order = processOrder(request);
        ordersCreated.increment();
        return order;
    }
}
```

## Timer Metrics

```java
@Service
public class PaymentService {

    private final Timer paymentTimer;

    public PaymentService(MeterRegistry registry) {
        this.paymentTimer = Timer.builder("payment.processing")
            .description("Payment processing time")
            .register(registry);
    }

    public PaymentResult process(Payment payment) {
        return paymentTimer.record(() -> {
            // Process payment
            return doPayment(payment);
        });
    }
}
```

## Gauge Metrics

```java
@Component
public class QueueMetrics {

    public QueueMetrics(MeterRegistry registry, TaskQueue queue) {
        Gauge.builder("queue.size", queue, TaskQueue::size)
            .description("Current queue size")
            .register(registry);
    }
}
```

## Custom Endpoint

```java
@Component
@Endpoint(id = "features")
public class FeaturesEndpoint {

    private final Map<String, Boolean> features = new ConcurrentHashMap<>();

    @ReadOperation
    public Map<String, Boolean> features() {
        return features;
    }

    @ReadOperation
    public Boolean feature(@Selector String name) {
        return features.get(name);
    }

    @WriteOperation
    public void setFeature(@Selector String name, boolean enabled) {
        features.put(name, enabled);
    }

    @DeleteOperation
    public void deleteFeature(@Selector String name) {
        features.remove(name);
    }
}
```

## Info Endpoint

```yaml
# application.yml
info:
  app:
    name: ${spring.application.name}
    version: '@project.version@'
    description: My Application
  java:
    version: ${java.version}
```

```java
@Component
public class CustomInfoContributor implements InfoContributor {

    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("customInfo", Map.of(
            "activeUsers", getActiveUsers(),
            "serverTime", Instant.now()
        ));
    }
}
```

## Security Configuration

```java
@Configuration
public class ActuatorSecurityConfig {

    @Bean
    public SecurityFilterChain actuatorSecurity(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/actuator/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/**").permitAll()
                .requestMatchers("/actuator/info").permitAll()
                .requestMatchers("/actuator/**").hasRole("ADMIN")
            )
            .httpBasic(Customizer.withDefaults())
            .build();
    }
}
```

## Prometheus Integration

```yaml
management:
  endpoints:
    web:
      exposure:
        include: prometheus
  prometheus:
    metrics:
      export:
        enabled: true
```

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

## Kubernetes Probes

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

Endpoints:
- `/actuator/health/liveness` - Is app alive?
- `/actuator/health/readiness` - Can app handle traffic?
