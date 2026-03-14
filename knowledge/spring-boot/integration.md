# Spring Integration Reference

## Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-integration</artifactId>
</dependency>
```

## Basic Concepts

```
Producer → Channel → Consumer
           ↓
    Message (payload + headers)
```

## Channels

```java
@Configuration
public class ChannelConfig {

    // Direct channel (point-to-point)
    @Bean
    public DirectChannel orderChannel() {
        return new DirectChannel();
    }

    // Publish-subscribe channel
    @Bean
    public PublishSubscribeChannel notificationChannel() {
        return new PublishSubscribeChannel();
    }

    // Queue channel (async, buffered)
    @Bean
    public QueueChannel processingQueue() {
        return new QueueChannel(100);
    }

    // Priority channel
    @Bean
    public PriorityChannel priorityChannel() {
        return new PriorityChannel(100,
            Comparator.comparing(m -> m.getHeaders().get("priority", Integer.class)));
    }
}
```

## Gateway

```java
@MessagingGateway
public interface OrderGateway {

    @Gateway(requestChannel = "orderChannel")
    void submitOrder(Order order);

    @Gateway(requestChannel = "orderChannel", replyChannel = "responseChannel")
    OrderConfirmation processOrder(Order order);

    @Gateway(requestChannel = "orderChannel", replyTimeout = 5000)
    OrderConfirmation processOrderWithTimeout(Order order);
}
```

## Service Activator

```java
@Component
public class OrderProcessor {

    @ServiceActivator(inputChannel = "orderChannel", outputChannel = "processedChannel")
    public ProcessedOrder process(Order order) {
        // Process order
        return new ProcessedOrder(order, Instant.now());
    }

    @ServiceActivator(inputChannel = "orderChannel")
    public void handleOrder(@Payload Order order, @Headers Map<String, Object> headers) {
        log.info("Processing order {} with headers {}", order.getId(), headers);
    }
}
```

## Transformer

```java
@Component
public class OrderTransformer {

    @Transformer(inputChannel = "rawOrderChannel", outputChannel = "orderChannel")
    public Order transform(RawOrder raw) {
        return Order.builder()
            .id(raw.getOrderId())
            .product(raw.getProductCode())
            .quantity(raw.getQty())
            .build();
    }

    // Generic transformer
    @Transformer(inputChannel = "jsonChannel", outputChannel = "objectChannel")
    public Object transformJson(@Payload String json, @Header("type") String type)
            throws Exception {
        Class<?> clazz = Class.forName(type);
        return objectMapper.readValue(json, clazz);
    }
}
```

## Router

```java
@Component
public class OrderRouter {

    @Router(inputChannel = "orderChannel")
    public String route(Order order) {
        return switch (order.getPriority()) {
            case HIGH -> "highPriorityChannel";
            case MEDIUM -> "mediumPriorityChannel";
            case LOW -> "lowPriorityChannel";
        };
    }

    // Recipient list router
    @Router(inputChannel = "orderChannel")
    public List<String> multiRoute(Order order) {
        List<String> channels = new ArrayList<>();
        channels.add("auditChannel");

        if (order.getTotal() > 1000) {
            channels.add("largeOrderChannel");
        }

        return channels;
    }
}
```

## Filter

```java
@Component
public class OrderFilter {

    @Filter(inputChannel = "orderChannel", outputChannel = "validOrderChannel")
    public boolean filterOrder(Order order) {
        return order.getQuantity() > 0 && order.getTotal() > 0;
    }

    // With discard channel
    @Filter(inputChannel = "orderChannel",
            outputChannel = "validOrderChannel",
            discardChannel = "invalidOrderChannel")
    public boolean filterWithDiscard(Order order) {
        return order.isValid();
    }
}
```

## Splitter & Aggregator

```java
@Component
public class OrderSplitter {

    @Splitter(inputChannel = "batchOrderChannel", outputChannel = "orderChannel")
    public List<Order> split(BatchOrder batch) {
        return batch.getOrders();
    }
}

@Component
public class OrderAggregator {

    @Aggregator(inputChannel = "processedOrderChannel",
                outputChannel = "batchResultChannel")
    public BatchResult aggregate(List<ProcessedOrder> orders) {
        return new BatchResult(orders);
    }

    @CorrelationStrategy
    public String correlate(ProcessedOrder order) {
        return order.getBatchId();
    }

    @ReleaseStrategy
    public boolean canRelease(List<ProcessedOrder> orders) {
        return orders.size() >= 10;
    }
}
```

## File Adapter

```java
@Configuration
public class FileIntegrationConfig {

    @Bean
    public IntegrationFlow fileReadingFlow() {
        return IntegrationFlow
            .from(Files.inboundAdapter(new File("/input"))
                    .patternFilter("*.csv")
                    .preventDuplicates(true),
                e -> e.poller(Pollers.fixedDelay(1000)))
            .transform(Files.toStringTransformer())
            .handle(fileProcessor())
            .get();
    }

    @Bean
    public IntegrationFlow fileWritingFlow() {
        return IntegrationFlow
            .from("outputChannel")
            .handle(Files.outboundAdapter(new File("/output"))
                .fileNameGenerator(m -> "output-" + System.currentTimeMillis() + ".txt")
                .autoCreateDirectory(true))
            .get();
    }
}
```

## HTTP Adapter

```java
@Configuration
public class HttpIntegrationConfig {

    @Bean
    public IntegrationFlow httpInboundFlow() {
        return IntegrationFlow
            .from(Http.inboundGateway("/api/orders")
                .requestMapping(m -> m.methods(HttpMethod.POST))
                .requestPayloadType(Order.class))
            .handle(orderService())
            .get();
    }

    @Bean
    public IntegrationFlow httpOutboundFlow() {
        return IntegrationFlow
            .from("requestChannel")
            .handle(Http.outboundGateway("https://api.example.com/orders")
                .httpMethod(HttpMethod.POST)
                .expectedResponseType(OrderResponse.class))
            .get();
    }
}
```

## JMS/Kafka Adapter

```java
@Configuration
public class KafkaIntegrationConfig {

    @Bean
    public IntegrationFlow kafkaProducerFlow() {
        return IntegrationFlow
            .from("orderChannel")
            .handle(Kafka.outboundChannelAdapter(kafkaTemplate)
                .topic("orders")
                .messageKey(m -> m.getHeaders().get("orderId")))
            .get();
    }

    @Bean
    public IntegrationFlow kafkaConsumerFlow() {
        return IntegrationFlow
            .from(Kafka.messageDrivenChannelAdapter(consumerFactory, "orders")
                .configureListenerContainer(c -> c.groupId("order-processor")))
            .handle(orderProcessor())
            .get();
    }
}
```

## DSL Flow

```java
@Configuration
public class OrderFlowConfig {

    @Bean
    public IntegrationFlow orderProcessingFlow() {
        return IntegrationFlow
            .from("orderInput")
            .filter(Order.class, o -> o.isValid())
            .transform(Order.class, this::enrich)
            .<Order, String>route(o -> o.getPriority().name(),
                mapping -> mapping
                    .subFlowMapping("HIGH", sf -> sf.channel("highPriorityChannel"))
                    .subFlowMapping("LOW", sf -> sf.channel("lowPriorityChannel"))
                    .defaultOutputToParentFlow())
            .handle(orderService(), "process")
            .channel("completedChannel")
            .get();
    }
}
```

## Error Handling

```java
@Bean
public IntegrationFlow errorHandlingFlow() {
    return IntegrationFlow
        .from("inputChannel")
        .handle(m -> {
            throw new RuntimeException("Error!");
        }, e -> e.advice(retryAdvice()))
        .get();
}

@Bean
public RequestHandlerRetryAdvice retryAdvice() {
    RequestHandlerRetryAdvice advice = new RequestHandlerRetryAdvice();
    advice.setRetryTemplate(RetryTemplate.builder()
        .maxAttempts(3)
        .fixedBackoff(1000)
        .build());
    return advice;
}
```
