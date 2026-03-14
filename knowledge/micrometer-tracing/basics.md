# Micrometer Tracing - Basics

## Overview

Micrometer Tracing provides a facade over tracing libraries (Brave, OpenTelemetry) for distributed tracing in Spring applications.

## Dependencies

### With Brave (Zipkin)
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

### With OpenTelemetry
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

## Configuration

### Zipkin
```yaml
management:
  tracing:
    enabled: true
    sampling:
      probability: 1.0  # 100% sampling for dev
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans

logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

### OpenTelemetry
```yaml
management:
  tracing:
    enabled: true
    sampling:
      probability: 1.0
  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces
```

## Tracer API

### Create Spans
```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final Tracer tracer;
    private final OrderRepository orderRepository;

    public Order processOrder(CreateOrderRequest request) {
        Span span = tracer.nextSpan().name("process-order").start();

        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            span.tag("order.customer", request.getCustomerId());
            span.tag("order.items", String.valueOf(request.getItems().size()));

            Order order = createOrder(request);

            span.tag("order.id", order.getId().toString());
            span.event("order.created");

            return order;
        } catch (Exception e) {
            span.error(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### Child Spans
```java
public void processPayment(Order order) {
    Span parentSpan = tracer.currentSpan();

    Span paymentSpan = tracer.nextSpan(parentSpan)
        .name("process-payment")
        .start();

    try (Tracer.SpanInScope ws = tracer.withSpan(paymentSpan)) {
        paymentSpan.tag("payment.amount", order.getTotal().toString());
        paymentService.charge(order);
    } finally {
        paymentSpan.end();
    }
}
```

## @Observed Annotation

### Enable Observation
```java
@Configuration
public class ObservationConfig {

    @Bean
    public ObservedAspect observedAspect(ObservationRegistry observationRegistry) {
        return new ObservedAspect(observationRegistry);
    }
}
```

### Usage
```java
@Service
public class OrderService {

    @Observed(
        name = "order.process",
        contextualName = "process-order",
        lowCardinalityKeyValues = {"order.type", "standard"}
    )
    public Order processOrder(CreateOrderRequest request) {
        return doProcess(request);
    }
}
```

## Context Propagation

### HTTP Client (RestTemplate)
```java
@Bean
public RestTemplate restTemplate(RestTemplateBuilder builder) {
    return builder.build();
    // Tracing is auto-configured for RestTemplate
}
```

### HTTP Client (WebClient)
```java
@Bean
public WebClient webClient(WebClient.Builder builder) {
    return builder.build();
    // Tracing is auto-configured for WebClient
}
```

### Messaging (Kafka)
```java
@KafkaListener(topics = "orders")
public void consume(ConsumerRecord<String, String> record,
                   @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
    // Span context is automatically extracted from headers
    log.info("Processing message from topic: {}", topic);
}
```

## Baggage

Propagate data across services:

```java
@Service
public class RequestService {

    private final Tracer tracer;
    private final BaggageManager baggageManager;

    public void processRequest(String tenantId) {
        try (BaggageInScope baggage = tracer.createBaggageInScope("tenant-id", tenantId)) {
            // tenant-id will propagate to all downstream services
            callDownstreamService();
        }
    }

    public String getCurrentTenant() {
        Baggage baggage = tracer.getBaggage("tenant-id");
        return baggage != null ? baggage.get() : null;
    }
}
```

```yaml
management:
  tracing:
    baggage:
      remote-fields:
        - tenant-id
        - correlation-id
      local-fields:
        - user-id
```

## Sampling

```yaml
management:
  tracing:
    sampling:
      probability: 0.1  # 10% sampling for production
```

### Custom Sampler
```java
@Bean
public Sampler customSampler() {
    return new Sampler() {
        @Override
        public SamplingDecision sample(SamplerInput input) {
            // Always sample errors
            if (input.name().contains("error")) {
                return SamplingDecision.SAMPLE;
            }
            // Sample 10% of requests
            return Math.random() < 0.1
                ? SamplingDecision.SAMPLE
                : SamplingDecision.DROP;
        }
    };
}
```

## Span Customization

```java
@Component
public class CustomSpanHandler implements SpanExportingPredicate, SpanReporter {

    @Override
    public boolean isExportable(FinishedSpan span) {
        // Don't export health check spans
        return !span.getName().contains("health");
    }

    @Override
    public void report(FinishedSpan span) {
        // Custom reporting logic
        log.debug("Span: {} duration: {}ms",
            span.getName(),
            span.getEndTimestamp() - span.getStartTimestamp());
    }
}
```

## Metrics Integration

Tracing and metrics work together:

```yaml
management:
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
  tracing:
    enabled: true
```

## Best Practices

| Do | Don't |
|----|-------|
| Use meaningful span names | Generic names like "span1" |
| Add relevant tags | Over-tag with sensitive data |
| Sample appropriately in production | 100% sampling in production |
| Propagate context | Break trace context |
| Use baggage for cross-cutting data | Overload baggage |
| Monitor trace collection | Ignore dropped spans |

## Production Checklist

- [ ] Appropriate sampling rate (1-10%)
- [ ] Trace collection endpoint configured
- [ ] Log correlation pattern set
- [ ] Baggage fields defined
- [ ] Health checks excluded
- [ ] Sensitive data filtered
- [ ] Retention policy configured
