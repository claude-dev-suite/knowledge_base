# Spring Cloud OpenFeign - Basics

## Overview

Spring Cloud OpenFeign is a declarative HTTP client that simplifies service-to-service communication. It integrates with Spring Cloud LoadBalancer and Circuit Breaker patterns.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

## Enable Feign Clients

```java
@SpringBootApplication
@EnableFeignClients
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

## Basic Feign Client

```java
@FeignClient(name = "user-service")
public interface UserClient {

    @GetMapping("/api/users/{id}")
    User getUserById(@PathVariable("id") Long id);

    @GetMapping("/api/users")
    List<User> getAllUsers();

    @PostMapping("/api/users")
    User createUser(@RequestBody CreateUserRequest request);

    @PutMapping("/api/users/{id}")
    User updateUser(@PathVariable("id") Long id, @RequestBody UpdateUserRequest request);

    @DeleteMapping("/api/users/{id}")
    void deleteUser(@PathVariable("id") Long id);
}
```

## Using the Client

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final UserClient userClient;

    public Order createOrder(CreateOrderRequest request) {
        // Fetch user from user-service
        User user = userClient.getUserById(request.getUserId());

        Order order = new Order();
        order.setUserId(user.getId());
        order.setUserEmail(user.getEmail());
        // ... create order

        return orderRepository.save(order);
    }
}
```

## URL-Based Client

For external services not in service discovery:

```java
@FeignClient(name = "payment-gateway", url = "${payment.service.url}")
public interface PaymentClient {

    @PostMapping("/payments")
    PaymentResponse processPayment(@RequestBody PaymentRequest request);

    @GetMapping("/payments/{id}")
    PaymentStatus getPaymentStatus(@PathVariable("id") String paymentId);
}
```

```yaml
payment:
  service:
    url: https://api.payment-gateway.com
```

## Request/Response Examples

### Path Variables and Query Parameters
```java
@FeignClient(name = "product-service")
public interface ProductClient {

    @GetMapping("/api/products/{id}")
    Product getProduct(@PathVariable("id") Long id);

    @GetMapping("/api/products")
    List<Product> searchProducts(
        @RequestParam("category") String category,
        @RequestParam(value = "minPrice", required = false) BigDecimal minPrice,
        @RequestParam(value = "maxPrice", required = false) BigDecimal maxPrice
    );

    @GetMapping("/api/products")
    Page<Product> getProductsPage(
        @RequestParam("page") int page,
        @RequestParam("size") int size,
        @RequestParam("sort") String sort
    );
}
```

### Headers
```java
@FeignClient(name = "secure-service")
public interface SecureClient {

    @GetMapping("/api/protected")
    Data getProtectedData(@RequestHeader("Authorization") String token);

    @PostMapping("/api/data")
    void submitData(
        @RequestHeader("X-Request-Id") String requestId,
        @RequestHeader("X-Correlation-Id") String correlationId,
        @RequestBody DataRequest request
    );
}
```

### Form Data
```java
@FeignClient(name = "auth-service")
public interface AuthClient {

    @PostMapping(value = "/oauth/token", consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE)
    TokenResponse getToken(
        @RequestParam("grant_type") String grantType,
        @RequestParam("username") String username,
        @RequestParam("password") String password
    );
}
```

## Configuration

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            connect-timeout: 5000
            read-timeout: 10000
            logger-level: basic
          user-service:
            connect-timeout: 3000
            read-timeout: 5000
            logger-level: full
```

## Logging Levels

| Level | Output |
|-------|--------|
| NONE | No logging |
| BASIC | Request method, URL, response status, execution time |
| HEADERS | Basic + request/response headers |
| FULL | Headers + body + metadata |

```java
@Configuration
public class FeignConfig {

    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

```yaml
logging:
  level:
    com.example.client.UserClient: DEBUG
```

## Best Practices

| Do | Don't |
|----|-------|
| Use declarative interfaces | Manual HTTP client code |
| Configure timeouts | Use defaults |
| Implement fallbacks | Let failures cascade |
| Use circuit breakers | Retry indefinitely |
| Log requests in dev | Full logging in production |
| Handle exceptions properly | Ignore HTTP errors |

## Production Checklist

- [ ] Timeouts configured
- [ ] Circuit breaker enabled
- [ ] Fallbacks implemented
- [ ] Retry configured
- [ ] Error handling in place
- [ ] Request logging appropriate
- [ ] Compression enabled if needed
