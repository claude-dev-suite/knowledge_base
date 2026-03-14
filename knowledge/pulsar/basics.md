# Apache Pulsar - Basics & Architecture

## Overview

Apache Pulsar is a cloud-native, distributed messaging and streaming platform with multi-tenancy, geo-replication, and tiered storage.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Pulsar Cluster                              │
│  ┌───────────┐   ┌───────────┐   ┌───────────┐                 │
│  │  Broker 1 │   │  Broker 2 │   │  Broker 3 │                 │
│  └───────────┘   └───────────┘   └───────────┘                 │
│         │               │               │                       │
│  ┌─────────────────────────────────────────────────┐           │
│  │            BookKeeper (Bookies)                  │           │
│  └─────────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Tenant** | Top-level isolation unit |
| **Namespace** | Administrative unit within tenant |
| **Topic** | Message stream |
| **Subscription** | Named cursor with delivery semantics |
| **Partition** | Topic partitioning for parallelism |

## Subscription Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Exclusive** | Single consumer | Ordered processing |
| **Shared** | Round-robin | Load balancing |
| **Failover** | Active-standby | High availability |
| **Key_Shared** | Key-based routing | Ordered per key |

## Producer

```typescript
import Pulsar from 'pulsar-client';

const client = new Pulsar.Client({ serviceUrl: 'pulsar://localhost:6650' });
const producer = await client.createProducer({
  topic: 'persistent://tenant/namespace/orders',
  batchingEnabled: true
});

await producer.send({
  data: Buffer.from(JSON.stringify(order)),
  properties: { 'correlation-id': correlationId },
  partitionKey: order.customerId
});

await producer.close();
await client.close();
```

## Consumer

```typescript
const consumer = await client.subscribe({
  topic: 'persistent://tenant/namespace/orders',
  subscription: 'order-processor',
  subscriptionType: 'Shared',
  deadLetterPolicy: {
    maxRedeliverCount: 3,
    deadLetterTopic: 'persistent://tenant/namespace/orders-dlq'
  }
});

while (true) {
  const msg = await consumer.receive();
  try {
    await processOrder(JSON.parse(msg.getData().toString()));
    consumer.acknowledge(msg);
  } catch (error) {
    consumer.negativeAcknowledge(msg);
  }
}
```

## Key Features

- **Multi-tenancy**: Tenant/namespace isolation
- **Geo-replication**: Cross-datacenter replication
- **Tiered storage**: Offload old data to S3/GCS
- **Schema registry**: Built-in schema management
- **Pulsar Functions**: Lightweight stream processing

## Best Practices

- Use partitioned topics for scale
- Configure appropriate retention
- Implement dead letter topics
- Use Key_Shared for ordered per-key delivery
- Monitor subscription lag

## References

- [Pulsar Documentation](https://pulsar.apache.org/docs/)
- [Pulsar Concepts](https://pulsar.apache.org/docs/concepts-overview/)
