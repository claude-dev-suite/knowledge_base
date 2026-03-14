# NATS - Basics & Architecture

## Overview

NATS is a lightweight, high-performance messaging system designed for cloud-native applications. JetStream adds persistence and exactly-once delivery.

## Core NATS vs JetStream

| Feature | Core NATS | JetStream |
|---------|-----------|-----------|
| Persistence | No | Yes |
| Delivery | At-most-once | At-least-once / Exactly-once |
| Consumer Groups | Queue Groups | Consumer Groups |
| Replay | No | Yes |

## Subjects

```
orders.created     → Specific subject
orders.*           → Single wildcard (orders.created, orders.updated)
orders.>           → Multi wildcard (orders.created, orders.us.east)
```

## Core NATS Patterns

### Publish/Subscribe
```typescript
import { connect } from 'nats';

const nc = await connect({ servers: 'localhost:4222' });
const sc = StringCodec();

// Publish
nc.publish('orders.created', sc.encode(JSON.stringify(order)));

// Subscribe
const sub = nc.subscribe('orders.*');
for await (const msg of sub) {
  const order = JSON.parse(sc.decode(msg.data));
  console.log(order);
}
```

### Queue Groups (Load Balancing)
```typescript
// Multiple consumers in same group share messages
const sub = nc.subscribe('orders.process', { queue: 'workers' });
```

### Request/Reply
```typescript
// Requester
const response = await nc.request('orders.validate', sc.encode(JSON.stringify(order)), {
  timeout: 5000
});

// Responder
const sub = nc.subscribe('orders.validate');
for await (const msg of sub) {
  const isValid = await validateOrder(msg.data);
  msg.respond(sc.encode(JSON.stringify({ valid: isValid })));
}
```

## JetStream

### Stream Creation
```typescript
const js = nc.jetstream();
const jsm = await nc.jetstreamManager();

await jsm.streams.add({
  name: 'ORDERS',
  subjects: ['orders.*'],
  storage: 'file',
  retention: 'limits',
  max_msgs: 1000000,
  replicas: 3
});
```

### Publishing
```typescript
const pa = await js.publish('orders.created', sc.encode(JSON.stringify(order)), {
  msgID: order.id  // Deduplication
});
```

### Consuming
```typescript
// Create consumer
await jsm.consumers.add('ORDERS', {
  durable_name: 'order-processor',
  ack_policy: 'explicit',
  max_deliver: 3
});

// Pull consumer
const consumer = await js.consumers.get('ORDERS', 'order-processor');
const messages = await consumer.consume({ max_messages: 10 });

for await (const msg of messages) {
  try {
    await processOrder(JSON.parse(sc.decode(msg.data)));
    msg.ack();
  } catch (error) {
    msg.nak();
  }
}
```

## Best Practices

- Use JetStream for reliability requirements
- Implement proper acknowledgment
- Use queue groups for load balancing
- Configure appropriate retention
- Monitor consumer lag

## References

- [NATS Documentation](https://docs.nats.io/)
- [JetStream](https://docs.nats.io/nats-concepts/jetstream)
