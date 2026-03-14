# Micrometer Tracing - Exporters

## Overview

Micrometer Tracing supports multiple trace exporters including Zipkin, Jaeger, OTLP (OpenTelemetry Protocol), and custom exporters for sending trace data to observability backends.

## Zipkin Exporter

### Dependencies
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

### Configuration
```yaml
management:
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans
      connect-timeout: 1s
      read-timeout: 10s

  tracing:
    sampling:
      probability: 1.0
```

### Java Configuration
```java
@Configuration
public class ZipkinConfig {

    @Bean
    public Sender zipkinSender() {
        return URLConnectionSender.newBuilder()
            .endpoint("http://localhost:9411/api/v2/spans")
            .connectTimeout(1000)
            .readTimeout(10000)
            .build();
    }

    @Bean
    public AsyncZipkinSpanHandler spanHandler(Sender sender) {
        return AsyncZipkinSpanHandler.newBuilder(sender)
            .messageTimeout(1, TimeUnit.SECONDS)
            .queuedMaxSpans(1000)
            .build();
    }
}
```

### Kafka Transport
```java
@Bean
public Sender kafkaSender() {
    return KafkaSender.newBuilder()
        .bootstrapServers("localhost:9092")
        .topic("zipkin")
        .encoding(Encoding.JSON)
        .build();
}
```

## OTLP Exporter (OpenTelemetry)

### Dependencies
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

### Configuration
```yaml
management:
  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces
      # For gRPC:
      # endpoint: http://localhost:4317

  tracing:
    sampling:
      probability: 1.0
```

### gRPC Exporter
```java
@Configuration
public class OtlpGrpcConfig {

    @Bean
    public SpanExporter otlpGrpcSpanExporter() {
        return OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://localhost:4317")
            .setTimeout(Duration.ofSeconds(10))
            .addHeader("Authorization", "Bearer token")
            .build();
    }

    @Bean
    public SdkTracerProvider tracerProvider(SpanExporter spanExporter) {
        return SdkTracerProvider.builder()
            .addSpanProcessor(BatchSpanProcessor.builder(spanExporter)
                .setMaxQueueSize(2048)
                .setMaxExportBatchSize(512)
                .setScheduleDelay(Duration.ofSeconds(5))
                .build())
            .setResource(Resource.create(Attributes.of(
                ResourceAttributes.SERVICE_NAME, "my-service"
            )))
            .build();
    }
}
```

### HTTP Exporter
```java
@Bean
public SpanExporter otlpHttpSpanExporter() {
    return OtlpHttpSpanExporter.builder()
        .setEndpoint("http://localhost:4318/v1/traces")
        .setTimeout(Duration.ofSeconds(10))
        .setCompression("gzip")
        .addHeader("X-Custom-Header", "value")
        .build();
}
```

## Jaeger Exporter

### Dependencies
```xml
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-jaeger</artifactId>
</dependency>
```

### Configuration
```java
@Bean
public SpanExporter jaegerExporter() {
    return JaegerGrpcSpanExporter.builder()
        .setEndpoint("http://localhost:14250")
        .setTimeout(Duration.ofSeconds(10))
        .build();
}
```

### Jaeger via OTLP
```yaml
# Jaeger supports OTLP natively
management:
  otlp:
    tracing:
      endpoint: http://localhost:4317  # Jaeger OTLP gRPC
```

## Multiple Exporters

### Composite Exporter
```java
@Configuration
public class MultiExporterConfig {

    @Bean
    public SpanExporter compositeSpanExporter() {
        SpanExporter zipkinExporter = ZipkinSpanExporter.builder()
            .setEndpoint("http://zipkin:9411/api/v2/spans")
            .build();

        SpanExporter otlpExporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://otel-collector:4317")
            .build();

        return SpanExporter.composite(zipkinExporter, otlpExporter);
    }

    @Bean
    public SdkTracerProvider tracerProvider(SpanExporter compositeExporter) {
        return SdkTracerProvider.builder()
            .addSpanProcessor(BatchSpanProcessor.builder(compositeExporter).build())
            .build();
    }
}
```

### Conditional Exporters
```java
@Configuration
public class ConditionalExporterConfig {

    @Bean
    @ConditionalOnProperty(name = "tracing.exporter", havingValue = "zipkin")
    public SpanExporter zipkinExporter() {
        return ZipkinSpanExporter.builder()
            .setEndpoint("http://zipkin:9411/api/v2/spans")
            .build();
    }

    @Bean
    @ConditionalOnProperty(name = "tracing.exporter", havingValue = "otlp")
    public SpanExporter otlpExporter() {
        return OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://otel-collector:4317")
            .build();
    }

    @Bean
    @ConditionalOnProperty(name = "tracing.exporter", havingValue = "console")
    public SpanExporter consoleExporter() {
        return new LoggingSpanExporter();
    }
}
```

## Console/Logging Exporter

```java
@Bean
@Profile("dev")
public SpanExporter loggingSpanExporter() {
    return LoggingSpanExporter.create();
}

// Custom logging exporter
public class CustomLoggingExporter implements SpanExporter {

    private static final Logger log = LoggerFactory.getLogger(CustomLoggingExporter.class);

    @Override
    public CompletableResultCode export(Collection<SpanData> spans) {
        spans.forEach(span -> {
            log.info("Span: name={}, traceId={}, spanId={}, duration={}ms",
                span.getName(),
                span.getTraceId(),
                span.getSpanId(),
                span.getEndEpochNanos() - span.getStartEpochNanos() / 1_000_000);
        });
        return CompletableResultCode.ofSuccess();
    }

    @Override
    public CompletableResultCode flush() {
        return CompletableResultCode.ofSuccess();
    }

    @Override
    public CompletableResultCode shutdown() {
        return CompletableResultCode.ofSuccess();
    }
}
```

## Custom Exporter

### File Exporter
```java
public class FileSpanExporter implements SpanExporter {

    private final Path outputPath;
    private final ObjectMapper mapper = new ObjectMapper();

    public FileSpanExporter(Path outputPath) {
        this.outputPath = outputPath;
    }

    @Override
    public CompletableResultCode export(Collection<SpanData> spans) {
        try {
            List<Map<String, Object>> spanMaps = spans.stream()
                .map(this::toMap)
                .toList();

            String json = mapper.writeValueAsString(spanMaps);
            Files.writeString(outputPath,
                json + System.lineSeparator(),
                StandardOpenOption.CREATE,
                StandardOpenOption.APPEND);

            return CompletableResultCode.ofSuccess();
        } catch (IOException e) {
            return CompletableResultCode.ofFailure();
        }
    }

    private Map<String, Object> toMap(SpanData span) {
        return Map.of(
            "traceId", span.getTraceId(),
            "spanId", span.getSpanId(),
            "parentSpanId", span.getParentSpanId(),
            "name", span.getName(),
            "kind", span.getKind().name(),
            "startTime", span.getStartEpochNanos(),
            "endTime", span.getEndEpochNanos(),
            "attributes", span.getAttributes().asMap()
        );
    }

    @Override
    public CompletableResultCode flush() {
        return CompletableResultCode.ofSuccess();
    }

    @Override
    public CompletableResultCode shutdown() {
        return CompletableResultCode.ofSuccess();
    }
}
```

### HTTP Webhook Exporter
```java
public class WebhookSpanExporter implements SpanExporter {

    private final WebClient webClient;
    private final String webhookUrl;

    public WebhookSpanExporter(WebClient.Builder builder, String webhookUrl) {
        this.webClient = builder.build();
        this.webhookUrl = webhookUrl;
    }

    @Override
    public CompletableResultCode export(Collection<SpanData> spans) {
        List<SpanPayload> payloads = spans.stream()
            .map(SpanPayload::from)
            .toList();

        try {
            webClient.post()
                .uri(webhookUrl)
                .bodyValue(payloads)
                .retrieve()
                .toBodilessEntity()
                .block(Duration.ofSeconds(5));

            return CompletableResultCode.ofSuccess();
        } catch (Exception e) {
            return CompletableResultCode.ofFailure();
        }
    }

    @Override
    public CompletableResultCode flush() {
        return CompletableResultCode.ofSuccess();
    }

    @Override
    public CompletableResultCode shutdown() {
        return CompletableResultCode.ofSuccess();
    }

    record SpanPayload(String traceId, String spanId, String name, long durationMs) {
        static SpanPayload from(SpanData span) {
            long durationMs = (span.getEndEpochNanos() - span.getStartEpochNanos()) / 1_000_000;
            return new SpanPayload(span.getTraceId(), span.getSpanId(), span.getName(), durationMs);
        }
    }
}
```

## Span Processor Configuration

### Batch Processor
```java
@Bean
public SpanProcessor batchSpanProcessor(SpanExporter exporter) {
    return BatchSpanProcessor.builder(exporter)
        .setMaxQueueSize(2048)
        .setMaxExportBatchSize(512)
        .setExporterTimeout(Duration.ofSeconds(30))
        .setScheduleDelay(Duration.ofSeconds(5))
        .build();
}
```

### Simple Processor (Immediate Export)
```java
@Bean
@Profile("dev")
public SpanProcessor simpleSpanProcessor(SpanExporter exporter) {
    return SimpleSpanProcessor.create(exporter);
}
```

### Filtering Processor
```java
public class FilteringSpanProcessor implements SpanProcessor {

    private final SpanProcessor delegate;
    private final Predicate<ReadableSpan> filter;

    public FilteringSpanProcessor(SpanProcessor delegate, Predicate<ReadableSpan> filter) {
        this.delegate = delegate;
        this.filter = filter;
    }

    @Override
    public void onStart(Context parentContext, ReadWriteSpan span) {
        delegate.onStart(parentContext, span);
    }

    @Override
    public boolean isStartRequired() {
        return delegate.isStartRequired();
    }

    @Override
    public void onEnd(ReadableSpan span) {
        if (filter.test(span)) {
            delegate.onEnd(span);
        }
    }

    @Override
    public boolean isEndRequired() {
        return delegate.isEndRequired();
    }
}

// Usage
@Bean
public SpanProcessor filteringProcessor(SpanExporter exporter) {
    SpanProcessor batchProcessor = BatchSpanProcessor.builder(exporter).build();

    return new FilteringSpanProcessor(batchProcessor, span -> {
        // Filter out health check spans
        return !span.getName().contains("health");
    });
}
```

## Cloud Provider Exporters

### AWS X-Ray
```xml
<dependency>
    <groupId>io.opentelemetry.contrib</groupId>
    <artifactId>opentelemetry-aws-xray</artifactId>
</dependency>
```

```java
@Bean
public SpanExporter xrayExporter() {
    return OtlpGrpcSpanExporter.builder()
        .setEndpoint("https://xray.us-east-1.amazonaws.com:443")
        .build();
}
```

### Google Cloud Trace
```xml
<dependency>
    <groupId>com.google.cloud</groupId>
    <artifactId>google-cloud-trace</artifactId>
</dependency>
```

### Azure Monitor
```xml
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-monitor-opentelemetry-exporter</artifactId>
</dependency>
```

## Best Practices

| Do | Don't |
|----|-------|
| Use batch processing | Export each span immediately |
| Configure timeouts | Block indefinitely on export |
| Handle export failures | Crash on network issues |
| Use appropriate queue sizes | Unlimited queue growth |
| Compress payloads | Send uncompressed data |
| Retry with backoff | Retry aggressively |

## Production Checklist

- [ ] Exporter endpoint configured
- [ ] Batch processor tuned
- [ ] Timeouts set appropriately
- [ ] Authentication configured
- [ ] Compression enabled
- [ ] Fallback for failures
