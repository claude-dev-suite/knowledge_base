# Redis Pub/Sub & Streams - Basics

## Overview

Redis provides two messaging mechanisms: Pub/Sub (fire-and-forget) and Streams (persistent with consumer groups).

## Comparison

| Feature | Pub/Sub | Streams |
|---------|---------|---------|
| Persistence | No | Yes |
| Consumer Groups | No | Yes |
| Message History | No | Yes |
| Delivery | At-most-once | At-least-once |

## Pub/Sub

### Publisher
```typescript
import Redis from 'ioredis';
const publisher = new Redis();

await publisher.publish('orders', JSON.stringify(order));
await publisher.publish('order.created', JSON.stringify(order));
```

### Subscriber
```typescript
const subscriber = new Redis();

// Channel subscription
subscriber.subscribe('orders', (err, count) => {
  console.log(`Subscribed to ${count} channels`);
});

// Pattern subscription
subscriber.psubscribe('order.*');

subscriber.on('message', (channel, message) => {
  const data = JSON.parse(message);
  console.log(`${channel}: ${data}`);
});

subscriber.on('pmessage', (pattern, channel, message) => {
  console.log(`Pattern ${pattern}, Channel ${channel}`);
});
```

## Streams

### Producer
```typescript
// Add to stream
const messageId = await redis.xadd('orders-stream', '*',
  'orderId', order.id,
  'data', JSON.stringify(order),
  'timestamp', Date.now().toString()
);
```

### Consumer Group
```typescript
// Create consumer group
await redis.xgroup('CREATE', 'orders-stream', 'processors', '0', 'MKSTREAM');

// Consume messages
const results = await redis.xreadgroup(
  'GROUP', 'processors', 'consumer-1',
  'COUNT', 10,
  'BLOCK', 5000,
  'STREAMS', 'orders-stream', '>'
);

for (const [stream, messages] of results || []) {
  for (const [id, fields] of messages) {
    await processMessage(fields);
    await redis.xack('orders-stream', 'processors', id);
  }
}
```

### Pending Messages
```typescript
// Get pending messages
const pending = await redis.xpending('orders-stream', 'processors', '-', '+', 100);

// Claim old pending messages
for (const entry of pending) {
  if (entry[2] > 60000) {  // Idle > 1 minute
    await redis.xclaim('orders-stream', 'processors', 'recovery', 60000, entry[0]);
  }
}
```

## Stream Commands

| Command | Description |
|---------|-------------|
| `XADD` | Add entry to stream |
| `XREAD` | Read entries |
| `XREADGROUP` | Read with consumer group |
| `XACK` | Acknowledge processing |
| `XPENDING` | List pending entries |
| `XCLAIM` | Claim pending entry |
| `XLEN` | Stream length |
| `XTRIM` | Trim stream size |

## Best Practices

### Pub/Sub
- Use for real-time, ephemeral messages
- Implement reconnection logic
- Use patterns for flexible routing
- Consider message loss acceptable

### Streams
- Use for persistent messaging
- Implement proper acknowledgment
- Handle pending messages
- Configure stream trimming

## References

- [Redis Pub/Sub](https://redis.io/docs/manual/pubsub/)
- [Redis Streams](https://redis.io/docs/data-types/streams/)
