# Spring Cloud Function - Basics

## Overview

Spring Cloud Function provides a uniform programming model for serverless and FaaS deployments. Write once, deploy anywhere: AWS Lambda, Azure Functions, GCP Cloud Functions, or as a standalone application.

## Dependencies

```xml
<!-- Core -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-function-web</artifactId>
</dependency>
```

## Function Types

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Function Types                                      │
│                                                                         │
│  Function<I, O>    │ Input → Processing → Output                       │
│  Consumer<I>       │ Input → Processing (no output)                    │
│  Supplier<O>       │ (no input) → Generate Output                      │
│                                                                         │
│  ┌───────┐    ┌──────────┐    ┌────────┐                               │
│  │ Input │───▶│ Function │───▶│ Output │                               │
│  └───────┘    └──────────┘    └────────┘                               │
└─────────────────────────────────────────────────────────────────────────┘
```

## Basic Functions

### Function (Input → Output)
```java
@Configuration
public class FunctionConfig {

    @Bean
    public Function<String, String> uppercase() {
        return value -> value.toUpperCase();
    }

    @Bean
    public Function<Order, OrderResult> processOrder() {
        return order -> {
            // Process order
            return new OrderResult(order.getId(), "PROCESSED", Instant.now());
        };
    }
}
```

### Consumer (Input → void)
```java
@Bean
public Consumer<Event> logEvent() {
    return event -> {
        log.info("Event received: {}", event);
        eventRepository.save(event);
    };
}
```

### Supplier (void → Output)
```java
@Bean
public Supplier<String> hello() {
    return () -> "Hello, World!";
}

@Bean
public Supplier<List<Product>> getProducts() {
    return () -> productRepository.findAll();
}
```

## Configuration

### application.yml
```yaml
spring:
  cloud:
    function:
      # Default function to invoke
      definition: processOrder

      # For composition
      # definition: validate|process|notify
```

## HTTP Endpoints

When using `spring-cloud-starter-function-web`, functions are automatically exposed as HTTP endpoints:

```bash
# Invoke function
curl -X POST http://localhost:8080/uppercase -d "hello" -H "Content-Type: text/plain"
# Response: HELLO

# Invoke with JSON
curl -X POST http://localhost:8080/processOrder \
  -H "Content-Type: application/json" \
  -d '{"id": "123", "amount": 99.99}'

# Invoke supplier
curl http://localhost:8080/hello
# Response: Hello, World!
```

## Function Composition

### YAML Configuration
```yaml
spring:
  cloud:
    function:
      definition: validate|sanitize|process
```

### Composed Functions
```java
@Bean
public Function<String, String> validate() {
    return input -> {
        if (input == null || input.isEmpty()) {
            throw new IllegalArgumentException("Input cannot be empty");
        }
        return input;
    };
}

@Bean
public Function<String, String> sanitize() {
    return input -> input.trim().toLowerCase();
}

@Bean
public Function<String, String> process() {
    return input -> "Processed: " + input;
}
```

### Programmatic Composition
```java
@Bean
public Function<String, String> composedFunction() {
    return validate().andThen(sanitize()).andThen(process());
}
```

## Reactive Functions

```java
@Bean
public Function<Flux<String>, Flux<String>> reactiveUppercase() {
    return flux -> flux.map(String::toUpperCase);
}

@Bean
public Function<Flux<Order>, Flux<OrderResult>> processOrdersStream() {
    return orders -> orders
        .filter(order -> order.getAmount().compareTo(BigDecimal.ZERO) > 0)
        .map(order -> new OrderResult(order.getId(), "PROCESSED"))
        .onErrorContinue((error, obj) -> log.error("Error processing: {}", obj, error));
}
```

## Function Routing

### Header-Based Routing
```yaml
spring:
  cloud:
    function:
      routing-expression: "headers['function-name']"
```

```bash
curl -X POST http://localhost:8080/ \
  -H "function-name: uppercase" \
  -d "hello"
```

### SpEL-Based Routing
```yaml
spring:
  cloud:
    function:
      routing-expression: "headers['type'] == 'order' ? 'processOrder' : 'defaultHandler'"
```

## Testing

```java
@SpringBootTest
class FunctionTests {

    @Autowired
    private Function<String, String> uppercase;

    @Autowired
    private FunctionCatalog catalog;

    @Test
    void shouldUppercase() {
        String result = uppercase.apply("hello");
        assertThat(result).isEqualTo("HELLO");
    }

    @Test
    void shouldLookupFunction() {
        Function<String, String> fn = catalog.lookup("uppercase");
        assertThat(fn.apply("test")).isEqualTo("TEST");
    }

    @Test
    void shouldComposeFunction() {
        Function<String, String> composed = catalog.lookup("validate|sanitize|process");
        String result = composed.apply("  HELLO  ");
        assertThat(result).isEqualTo("Processed: hello");
    }
}
```

### Integration Test
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class FunctionHttpTests {

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void shouldInvokeViaHttp() {
        webTestClient.post()
            .uri("/uppercase")
            .contentType(MediaType.TEXT_PLAIN)
            .bodyValue("hello")
            .exchange()
            .expectStatus().isOk()
            .expectBody(String.class)
            .isEqualTo("HELLO");
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Keep functions focused/small | Create monolithic functions |
| Use reactive types for streaming | Block on large datasets |
| Handle errors gracefully | Let exceptions propagate |
| Test functions in isolation | Skip unit tests |
| Use composition for pipelines | Duplicate logic |
| Configure appropriate timeouts | Ignore cold start |

## Production Checklist

- [ ] Function definition configured
- [ ] Error handling implemented
- [ ] Timeouts set appropriately
- [ ] Logging configured
- [ ] Monitoring enabled
- [ ] Input validation in place
- [ ] Cold start optimized
