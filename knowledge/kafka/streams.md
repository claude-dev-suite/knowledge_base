# Apache Kafka - Kafka Streams

## Overview

Kafka Streams is a client library for building real-time streaming applications and microservices. It provides a high-level DSL for common operations and a low-level Processor API for custom processing.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kafka Streams Application                     │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                      Topology                             │  │
│  │  Source ──▶ Processor ──▶ Processor ──▶ Sink             │  │
│  │    │            │            │            │               │  │
│  │    ▼            ▼            ▼            ▼               │  │
│  │  Input      State        Aggregation   Output            │  │
│  │  Topic      Store                       Topic            │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Stream Tasks                            │  │
│  │  Task 0 (Partition 0)  Task 1 (Partition 1)  ...         │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Core Concepts

| Concept | Description |
|---------|-------------|
| **KStream** | Unbounded stream of records |
| **KTable** | Changelog stream (compacted) |
| **GlobalKTable** | Full table replicated to all instances |
| **State Store** | Local state for aggregations/joins |
| **Processor** | Node in processing topology |

## Configuration

```java
Properties props = new Properties();
props.put(StreamsConfig.APPLICATION_ID_CONFIG, "order-processor");
props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

// Processing guarantee
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);

// State store
props.put(StreamsConfig.STATE_DIR_CONFIG, "/tmp/kafka-streams");

// Performance
props.put(StreamsConfig.NUM_STREAM_THREADS_CONFIG, 4);
props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 1000);
props.put(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG, 10 * 1024 * 1024); // 10MB
```

## DSL Operations

### Basic Operations

```java
StreamsBuilder builder = new StreamsBuilder();

// Source
KStream<String, String> orders = builder.stream("orders");

// Filter
KStream<String, String> highValueOrders = orders
    .filter((key, value) -> parseOrder(value).getAmount() > 100);

// Map
KStream<String, Order> parsedOrders = orders
    .mapValues(value -> objectMapper.readValue(value, Order.class));

// FlatMap
KStream<String, OrderItem> orderItems = parsedOrders
    .flatMapValues(order -> order.getItems());

// Peek (for side effects)
orders.peek((key, value) -> logger.info("Processing: {}", key));

// Branch
Map<String, KStream<String, Order>> branches = parsedOrders
    .split(Named.as("branch-"))
    .branch((key, order) -> order.getType().equals("EXPRESS"), Branched.as("express"))
    .branch((key, order) -> order.getType().equals("STANDARD"), Branched.as("standard"))
    .defaultBranch(Branched.as("other"));

KStream<String, Order> expressOrders = branches.get("branch-express");
```

### Aggregations

```java
// Count by key
KTable<String, Long> ordersByCustomer = orders
    .groupBy((key, value) -> parseOrder(value).getCustomerId())
    .count();

// Aggregate
KTable<String, OrderStats> stats = parsedOrders
    .groupBy((key, order) -> order.getCustomerId())
    .aggregate(
        OrderStats::new,  // Initializer
        (key, order, stats) -> stats.add(order),  // Aggregator
        Materialized.with(Serdes.String(), orderStatsSerde)
    );

// Windowed aggregation
KTable<Windowed<String>, Long> ordersPerHour = orders
    .groupBy((key, value) -> parseOrder(value).getCustomerId())
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofHours(1)))
    .count();

// Session windows
KTable<Windowed<String>, Long> sessionOrders = orders
    .groupBy((key, value) -> parseOrder(value).getCustomerId())
    .windowedBy(SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(30)))
    .count();
```

### Joins

```java
// KStream-KStream Join (windowed)
KStream<String, String> orders = builder.stream("orders");
KStream<String, String> payments = builder.stream("payments");

KStream<String, OrderPayment> orderPayments = orders.join(
    payments,
    (order, payment) -> new OrderPayment(order, payment),
    JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(5)),
    StreamJoined.with(Serdes.String(), Serdes.String(), Serdes.String())
);

// KStream-KTable Join
KTable<String, String> customers = builder.table("customers");

KStream<String, EnrichedOrder> enrichedOrders = orders.join(
    customers,
    (order, customer) -> enrichOrder(order, customer)
);

// KStream-GlobalKTable Join
GlobalKTable<String, String> products = builder.globalTable("products");

KStream<String, EnrichedOrder> enrichedOrders = orders.join(
    products,
    (key, order) -> order.getProductId(),  // Key mapper
    (order, product) -> enrichOrder(order, product)
);
```

### KTable Operations

```java
// Create KTable from topic (changelog)
KTable<String, Customer> customers = builder.table("customers");

// Transform KTable
KTable<String, String> customerNames = customers
    .mapValues(customer -> customer.getName());

// Filter KTable
KTable<String, Customer> activeCustomers = customers
    .filter((key, customer) -> customer.isActive());

// KTable-KTable Join
KTable<String, Customer> customers = builder.table("customers");
KTable<String, CustomerPrefs> preferences = builder.table("preferences");

KTable<String, EnrichedCustomer> enriched = customers.join(
    preferences,
    (customer, prefs) -> new EnrichedCustomer(customer, prefs)
);

// Convert to stream
KStream<String, Customer> customerStream = customers.toStream();
```

## State Stores

### Built-in State Stores

```java
// Materialized state store
KTable<String, Long> counts = stream
    .groupByKey()
    .count(Materialized.<String, Long, KeyValueStore<Bytes, byte[]>>as("counts-store")
        .withKeySerde(Serdes.String())
        .withValueSerde(Serdes.Long()));

// Query state store
ReadOnlyKeyValueStore<String, Long> store =
    streams.store(StoreQueryParameters.fromNameAndType("counts-store", QueryableStoreTypes.keyValueStore()));

Long count = store.get("customer-123");
```

### Interactive Queries

```java
// Local state query
public class OrderService {
    private final KafkaStreams streams;

    public Long getOrderCount(String customerId) {
        ReadOnlyKeyValueStore<String, Long> store =
            streams.store(StoreQueryParameters.fromNameAndType("order-counts", QueryableStoreTypes.keyValueStore()));

        return store.get(customerId);
    }

    // Range query
    public Map<String, Long> getOrderCounts(String from, String to) {
        Map<String, Long> result = new HashMap<>();

        try (KeyValueIterator<String, Long> iterator = store.range(from, to)) {
            while (iterator.hasNext()) {
                KeyValue<String, Long> kv = iterator.next();
                result.put(kv.key, kv.value);
            }
        }

        return result;
    }
}
```

### RocksDB Configuration

```java
// Custom RocksDB config
public class CustomRocksDBConfig implements RocksDBConfigSetter {
    @Override
    public void setConfig(String storeName, Options options, Map<String, Object> configs) {
        // Block cache
        BlockBasedTableConfig tableConfig = new BlockBasedTableConfig();
        tableConfig.setBlockCache(new LRUCache(50 * 1024 * 1024)); // 50MB
        tableConfig.setBlockSize(4096);
        options.setTableFormatConfig(tableConfig);

        // Write buffer
        options.setWriteBufferSize(16 * 1024 * 1024); // 16MB
        options.setMaxWriteBufferNumber(3);

        // Compaction
        options.setLevelCompactionDynamicLevelBytes(true);
    }
}

props.put(StreamsConfig.ROCKSDB_CONFIG_SETTER_CLASS_CONFIG, CustomRocksDBConfig.class);
```

## Processor API

```java
public class OrderProcessor implements Processor<String, Order, String, ProcessedOrder> {
    private ProcessorContext<String, ProcessedOrder> context;
    private KeyValueStore<String, OrderStats> statsStore;

    @Override
    public void init(ProcessorContext<String, ProcessedOrder> context) {
        this.context = context;
        this.statsStore = context.getStateStore("stats-store");

        // Schedule punctuation
        context.schedule(Duration.ofMinutes(1), PunctuationType.WALL_CLOCK_TIME, timestamp -> {
            // Periodic processing
            flushStats();
        });
    }

    @Override
    public void process(Record<String, Order> record) {
        Order order = record.value();

        // Update stats
        OrderStats stats = statsStore.get(order.getCustomerId());
        if (stats == null) {
            stats = new OrderStats();
        }
        stats.add(order);
        statsStore.put(order.getCustomerId(), stats);

        // Forward processed record
        ProcessedOrder processed = new ProcessedOrder(order, stats);
        context.forward(new Record<>(record.key(), processed, record.timestamp()));
    }

    @Override
    public void close() {
        // Cleanup
    }
}

// Register processor
Topology topology = new Topology();
topology.addSource("source", "orders")
    .addProcessor("processor", OrderProcessor::new, "source")
    .addStateStore(
        Stores.keyValueStoreBuilder(
            Stores.persistentKeyValueStore("stats-store"),
            Serdes.String(),
            orderStatsSerde),
        "processor")
    .addSink("sink", "processed-orders", "processor");
```

## Error Handling

### Deserialization Errors

```java
props.put(StreamsConfig.DEFAULT_DESERIALIZATION_EXCEPTION_HANDLER_CLASS_CONFIG,
    LogAndContinueExceptionHandler.class);

// Or custom handler
public class CustomDeserializationHandler implements DeserializationExceptionHandler {
    @Override
    public DeserializationHandlerResponse handle(ProcessorContext context,
                                                  ConsumerRecord<byte[], byte[]> record,
                                                  Exception exception) {
        logger.error("Deserialization error for record: {}", record, exception);
        sendToDeadLetterTopic(record);
        return DeserializationHandlerResponse.CONTINUE;
    }
}
```

### Production Errors

```java
props.put(StreamsConfig.DEFAULT_PRODUCTION_EXCEPTION_HANDLER_CLASS_CONFIG,
    DefaultProductionExceptionHandler.class);

// Custom handler
public class CustomProductionHandler implements ProductionExceptionHandler {
    @Override
    public ProductionExceptionHandlerResponse handle(ProducerRecord<byte[], byte[]> record,
                                                     Exception exception) {
        if (exception instanceof RecordTooLargeException) {
            return ProductionExceptionHandlerResponse.CONTINUE;
        }
        return ProductionExceptionHandlerResponse.FAIL;
    }
}
```

### Uncaught Exceptions

```java
streams.setUncaughtExceptionHandler(exception -> {
    logger.error("Uncaught exception", exception);

    if (exception instanceof RetriableException) {
        return StreamsUncaughtExceptionHandler.StreamThreadExceptionResponse.REPLACE_THREAD;
    }

    return StreamsUncaughtExceptionHandler.StreamThreadExceptionResponse.SHUTDOWN_CLIENT;
});
```

## Testing

```java
// TopologyTestDriver
@Test
void testOrderProcessing() {
    StreamsBuilder builder = new StreamsBuilder();
    buildTopology(builder);

    TopologyTestDriver testDriver = new TopologyTestDriver(builder.build(), props);

    TestInputTopic<String, Order> inputTopic = testDriver.createInputTopic(
        "orders",
        new StringSerializer(),
        new OrderSerializer()
    );

    TestOutputTopic<String, ProcessedOrder> outputTopic = testDriver.createOutputTopic(
        "processed-orders",
        new StringDeserializer(),
        new ProcessedOrderDeserializer()
    );

    // Send test data
    inputTopic.pipeInput("order-1", new Order("order-1", 100));

    // Verify output
    KeyValue<String, ProcessedOrder> result = outputTopic.readKeyValue();
    assertEquals("order-1", result.key);
    assertEquals(100, result.value.getAmount());

    testDriver.close();
}

// Test state stores
@Test
void testStateStore() {
    // ... setup ...

    inputTopic.pipeInput("customer-1", new Order("order-1", 100));
    inputTopic.pipeInput("customer-1", new Order("order-2", 200));

    KeyValueStore<String, Long> store = testDriver.getKeyValueStore("counts-store");
    assertEquals(2L, store.get("customer-1"));
}
```

## Production Best Practices

### Deployment

```java
// Standby replicas for fast failover
props.put(StreamsConfig.NUM_STANDBY_REPLICAS_CONFIG, 1);

// State store cleanup
props.put(StreamsConfig.STATE_CLEANUP_DELAY_MS_CONFIG, 600000); // 10 minutes

// Commit interval
props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 100);

// Exactly-once
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);
```

### Monitoring

```java
// State listener
streams.setStateListener((newState, oldState) -> {
    logger.info("State changed from {} to {}", oldState, newState);
    if (newState == KafkaStreams.State.ERROR) {
        alerting.sendAlert("Kafka Streams in ERROR state");
    }
});

// Metrics
streams.metrics().forEach((name, metric) -> {
    if (name.name().contains("process-rate")) {
        metricsRegistry.gauge(name.name(), () -> metric.metricValue());
    }
});
```

### Graceful Shutdown

```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    streams.close(Duration.ofSeconds(30));
}));
```

## References

- [Kafka Streams Documentation](https://kafka.apache.org/documentation/streams/)
- [Kafka Streams DSL](https://docs.confluent.io/platform/current/streams/developer-guide/dsl-api.html)
- [Interactive Queries](https://docs.confluent.io/platform/current/streams/developer-guide/interactive-queries.html)
