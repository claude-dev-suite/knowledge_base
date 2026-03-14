# Micrometer Tracing - Configuration

## Overview

Micrometer Tracing configuration enables distributed tracing in Spring applications with support for multiple tracing backends through a vendor-neutral API.

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

## Basic Configuration

### Application Properties
```yaml
management:
  tracing:
    enabled: true
    sampling:
      probability: 1.0  # 100% sampling for dev, lower for prod

  # Zipkin configuration
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans

spring:
  application:
    name: my-service

logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

### OpenTelemetry Properties
```yaml
management:
  tracing:
    enabled: true
    sampling:
      probability: 1.0

  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces

  opentelemetry:
    resource-attributes:
      service.name: ${spring.application.name}
      deployment.environment: ${spring.profiles.active:default}
```

## Java Configuration

### Brave Configuration
```java
@Configuration
public class TracingConfig {

    @Bean
    public SpanHandler zipkinSpanHandler(Sender sender) {
        return AsyncZipkinSpanHandler.create(sender);
    }

    @Bean
    public Sender zipkinSender() {
        return URLConnectionSender.create("http://localhost:9411/api/v2/spans");
    }

    @Bean
    public Tracing braveTracing(SpanHandler spanHandler) {
        return Tracing.newBuilder()
            .localServiceName("my-service")
            .supportsJoin(false)
            .traceId128Bit(true)
            .sampler(Sampler.ALWAYS_SAMPLE)
            .addSpanHandler(spanHandler)
            .build();
    }

    @Bean
    public brave.Tracer braveTracer(Tracing tracing) {
        return tracing.tracer();
    }

    @Bean
    public Tracer tracer(brave.Tracer braveTracer, CurrentTraceContext context) {
        return new BraveTracer(braveTracer, context, new BraveBaggageManager());
    }
}
```

### OpenTelemetry Configuration
```java
@Configuration
public class OtelTracingConfig {

    @Bean
    public OpenTelemetry openTelemetry() {
        Resource resource = Resource.getDefault()
            .merge(Resource.create(Attributes.of(
                ResourceAttributes.SERVICE_NAME, "my-service",
                ResourceAttributes.SERVICE_VERSION, "1.0.0"
            )));

        SpanExporter spanExporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://localhost:4317")
            .build();

        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .setResource(resource)
            .addSpanProcessor(BatchSpanProcessor.builder(spanExporter).build())
            .setSampler(Sampler.alwaysOn())
            .build();

        return OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .setPropagators(ContextPropagators.create(W3CTraceContextPropagator.getInstance()))
            .buildAndRegisterGlobal();
    }

    @Bean
    public Tracer tracer(OpenTelemetry openTelemetry) {
        return new OtelTracer(
            openTelemetry.getTracer("my-service"),
            new OtelCurrentTraceContext(),
            event -> {}
        );
    }
}
```

## Sampling Configuration

### Probability-Based Sampling
```yaml
management:
  tracing:
    sampling:
      probability: 0.1  # 10% of traces
```

### Custom Sampler
```java
@Bean
public Sampler customSampler() {
    return new Sampler() {
        @Override
        public SamplingResult shouldSample(
                Context parentContext,
                String traceId,
                String name,
                SpanKind spanKind,
                Attributes attributes,
                List<LinkData> parentLinks) {

            // Always sample errors
            if (name.contains("error")) {
                return SamplingResult.recordAndSample();
            }

            // Sample based on path
            String path = attributes.get(AttributeKey.stringKey("http.url"));
            if (path != null && path.contains("/health")) {
                return SamplingResult.drop();
            }

            // 10% sampling for everything else
            return Math.random() < 0.1
                ? SamplingResult.recordAndSample()
                : SamplingResult.drop();
        }

        @Override
        public String getDescription() {
            return "CustomSampler";
        }
    };
}
```

### Rate-Limited Sampler
```java
@Bean
public Sampler rateLimitedSampler() {
    // Max 10 traces per second
    return Sampler.traceIdRatioBased(0.1);
}
```

## Propagation Configuration

### W3C Trace Context (Default)
```java
@Bean
public Propagator propagator() {
    return new CompositeTextMapPropagator(
        List.of(
            new W3CPropagator(),
            new BaggagePropagator()
        )
    );
}
```

### B3 Propagation (Zipkin)
```yaml
management:
  tracing:
    propagation:
      type: B3
      # or B3_MULTI for multiple headers
```

```java
@Bean
public Propagation.Factory b3Propagation() {
    return B3Propagation.newFactoryBuilder()
        .injectFormat(B3Propagation.Format.MULTI)
        .build();
}
```

### Multiple Propagation Formats
```yaml
management:
  tracing:
    propagation:
      consume: [W3C, B3, B3_MULTI]
      produce: [W3C]
```

## Context Propagation

### Automatic Context Propagation
```java
@Configuration
public class ContextPropagationConfig {

    @Bean
    public ContextSnapshotFactory contextSnapshotFactory() {
        return ContextSnapshotFactory.builder()
            .build();
    }

    // For async operations
    @Bean
    public TaskDecorator tracingTaskDecorator(Tracer tracer) {
        return runnable -> {
            Span currentSpan = tracer.currentSpan();
            return () -> {
                try (Tracer.SpanInScope ws = tracer.withSpan(currentSpan)) {
                    runnable.run();
                }
            };
        };
    }

    @Bean
    public ThreadPoolTaskExecutor tracingTaskExecutor(TaskDecorator decorator) {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setTaskDecorator(decorator);
        executor.initialize();
        return executor;
    }
}
```

### Manual Context Propagation
```java
@Service
public class AsyncService {

    private final Tracer tracer;
    private final Executor executor;

    public void asyncOperation() {
        Span span = tracer.currentSpan();

        executor.execute(() -> {
            try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
                // Trace context is available here
                doWork();
            }
        });
    }
}
```

## Baggage Configuration

```java
@Configuration
public class BaggageConfig {

    @Bean
    public BaggageFields baggageFields() {
        return BaggageFields.newFactory(
            List.of(
                BaggageField.create("user-id"),
                BaggageField.create("tenant-id"),
                BaggageField.create("request-id")
            ),
            2  // Max dynamic fields
        ).build();
    }

    // Configure which baggage fields to propagate
    @Bean
    public BaggagePropagationConfig baggagePropagationConfig() {
        return BaggagePropagationConfig.builder()
            .add(SingleBaggageField.remote(BaggageField.create("user-id")))
            .add(SingleBaggageField.local(BaggageField.create("internal-id")))
            .build();
    }
}
```

### Using Baggage
```java
@Service
public class BaggageService {

    private final Tracer tracer;

    public void setUserContext(String userId) {
        BaggageInScope baggage = tracer.createBaggageInScope("user-id", userId);
        // Baggage will be propagated to downstream services
    }

    public String getUserId() {
        return tracer.getBaggage("user-id").get();
    }
}
```

## Integration Configuration

### RestTemplate Integration
```java
@Bean
public RestTemplate restTemplate(RestTemplateBuilder builder) {
    return builder.build();
    // Tracing is auto-configured
}
```

### WebClient Integration
```java
@Bean
public WebClient webClient(WebClient.Builder builder) {
    return builder.build();
    // Tracing is auto-configured
}
```

### Kafka Integration
```yaml
spring:
  kafka:
    producer:
      properties:
        interceptor.classes: io.micrometer.tracing.kafka.KafkaTracingProducerInterceptor
    consumer:
      properties:
        interceptor.classes: io.micrometer.tracing.kafka.KafkaTracingConsumerInterceptor
```

## Logging Integration

### MDC Configuration
```yaml
logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

### Custom MDC Fields
```java
@Component
public class TracingMdcCustomizer implements SpanCustomizer {

    @Override
    public void customize(Span span) {
        MDC.put("traceId", span.context().traceId());
        MDC.put("spanId", span.context().spanId());
        MDC.put("parentSpanId", span.context().parentId());
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use appropriate sampling rates | 100% sampling in production |
| Configure baggage for context | Pass context in custom headers |
| Use W3C propagation | Mix propagation formats |
| Include trace IDs in logs | Log without trace context |
| Set service name consistently | Different names per instance |
| Configure resource attributes | Skip service metadata |

## Production Checklist

- [ ] Sampling rate configured
- [ ] Exporter endpoint verified
- [ ] Propagation format consistent
- [ ] Logging pattern includes trace IDs
- [ ] Baggage fields documented
- [ ] Context propagation for async
