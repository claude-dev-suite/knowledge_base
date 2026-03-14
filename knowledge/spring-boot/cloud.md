# Spring Cloud Basics Reference

## Service Discovery (Eureka)

### Server
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication { }
```

```yaml
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
```

### Client
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```yaml
spring:
  application:
    name: user-service

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
  instance:
    prefer-ip-address: true
```

## Config Server

### Server
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication { }
```

```yaml
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/org/config-repo
          default-label: main
```

### Client
```yaml
spring:
  config:
    import: configserver:http://localhost:8888
  application:
    name: user-service
  profiles:
    active: dev
```

## API Gateway

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://USER-SERVICE
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Request-Source, gateway

        - id: order-service
          uri: lb://ORDER-SERVICE
          predicates:
            - Path=/api/orders/**
            - Method=GET,POST
          filters:
            - RewritePath=/api/orders/(?<segment>.*), /orders/${segment}
            - CircuitBreaker=name=orderService,fallbackUri=forward:/fallback

      default-filters:
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 10
            redis-rate-limiter.burstCapacity: 20
```

### Custom Filter
```java
@Component
public class LoggingFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("Request: {} {}", exchange.getRequest().getMethod(),
                                   exchange.getRequest().getPath());
        return chain.filter(exchange)
            .then(Mono.fromRunnable(() ->
                log.info("Response: {}", exchange.getResponse().getStatusCode())
            ));
    }

    @Override
    public int getOrder() {
        return -1;
    }
}
```

## Circuit Breaker (Resilience4j)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 3
    instances:
      userService:
        base-config: default
        sliding-window-size: 20

  retry:
    instances:
      userService:
        max-attempts: 3
        wait-duration: 500ms

  timelimiter:
    instances:
      userService:
        timeout-duration: 3s
```

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserClient userClient;

    @CircuitBreaker(name = "userService", fallbackMethod = "fallback")
    @Retry(name = "userService")
    @TimeLimiter(name = "userService")
    public CompletableFuture<User> getUser(Long id) {
        return CompletableFuture.supplyAsync(() -> userClient.getUser(id));
    }

    private CompletableFuture<User> fallback(Long id, Throwable t) {
        log.error("Fallback for user {}: {}", id, t.getMessage());
        return CompletableFuture.completedFuture(new User(id, "Unknown", "N/A"));
    }
}
```

## OpenFeign

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableFeignClients
public class Application { }

@FeignClient(name = "user-service", fallback = UserClientFallback.class)
public interface UserClient {

    @GetMapping("/users/{id}")
    User getUser(@PathVariable Long id);

    @PostMapping("/users")
    User createUser(@RequestBody CreateUserRequest request);

    @GetMapping("/users")
    List<User> searchUsers(@RequestParam String name);
}

@Component
public class UserClientFallback implements UserClient {

    @Override
    public User getUser(Long id) {
        return new User(id, "Fallback", "N/A");
    }

    @Override
    public User createUser(CreateUserRequest request) {
        throw new ServiceUnavailableException("User service unavailable");
    }

    @Override
    public List<User> searchUsers(String name) {
        return Collections.emptyList();
    }
}
```

## Load Balancing

```yaml
spring:
  cloud:
    loadbalancer:
      ribbon:
        enabled: false
      configurations: default
```

```java
@Configuration
public class LoadBalancerConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    @Bean
    @LoadBalanced
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}

@Service
public class OrderService {

    private final RestTemplate restTemplate;

    public User getUser(Long id) {
        // Uses service name, not direct URL
        return restTemplate.getForObject(
            "http://USER-SERVICE/users/{id}",
            User.class,
            id
        );
    }
}
```

## Distributed Tracing

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans

logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

## Common Dependencies (BOM)

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```
