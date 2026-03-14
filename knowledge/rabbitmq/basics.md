# RabbitMQ - Basics & Architecture

## Overview

RabbitMQ is a robust, open-source message broker implementing AMQP (Advanced Message Queuing Protocol). It supports multiple messaging protocols and provides features like message queuing, routing, reliability, and clustering.

## AMQP Model

```
┌─────────────────────────────────────────────────────────────────┐
│                        RabbitMQ Broker                          │
│                                                                  │
│  Publisher ──▶ Exchange ──binding──▶ Queue ──▶ Consumer         │
│                   │                    │                        │
│                   │  Routing Key       │  Messages              │
│                   │                    │                        │
│              ┌────┴────┐          ┌────┴────┐                  │
│              │ direct  │          │ durable │                  │
│              │ topic   │          │ exclusive│                  │
│              │ fanout  │          │ auto-del │                  │
│              │ headers │          └─────────┘                  │
│              └─────────┘                                        │
└─────────────────────────────────────────────────────────────────┘
```

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Connection** | TCP connection to broker |
| **Channel** | Virtual connection within connection |
| **Exchange** | Routes messages to queues |
| **Queue** | Buffer that stores messages |
| **Binding** | Rule linking exchange to queue |
| **Routing Key** | Message attribute for routing decisions |
| **Virtual Host** | Logical grouping for isolation |

## Exchange Types

### Direct Exchange
Routes messages to queues with exact routing key match.

```
Message(routing_key="order.created")
         │
         ▼
    ┌─────────┐
    │ Direct  │
    │Exchange │
    └────┬────┘
         │
    routing_key="order.created"
         │
         ▼
  ┌──────────────┐
  │ orders-queue │
  └──────────────┘
```

### Topic Exchange
Routes based on routing key patterns with wildcards.

```
Patterns:
  * - matches exactly one word
  # - matches zero or more words

Examples:
  order.* → order.created, order.updated
  order.# → order.created, order.item.added
  *.created → order.created, user.created
```

### Fanout Exchange
Broadcasts to all bound queues (ignores routing key).

```
Message
    │
    ▼
┌─────────┐
│ Fanout  │──────▶ Queue 1
│Exchange │──────▶ Queue 2
└─────────┘──────▶ Queue 3
```

### Headers Exchange
Routes based on message headers (not routing key).

```json
{
  "headers": {
    "x-match": "all",  // or "any"
    "format": "pdf",
    "type": "report"
  }
}
```

## Queue Properties

| Property | Description |
|----------|-------------|
| `durable` | Survives broker restart |
| `exclusive` | Used by only one connection |
| `auto-delete` | Deleted when last consumer unsubscribes |
| `arguments` | Optional queue arguments (TTL, DLX, etc.) |

### Queue Arguments

```javascript
{
  'x-message-ttl': 86400000,           // Message TTL (ms)
  'x-max-length': 10000,               // Max messages
  'x-max-length-bytes': 1073741824,    // Max size (1GB)
  'x-dead-letter-exchange': 'dlx',     // Dead letter exchange
  'x-dead-letter-routing-key': 'dlq',  // DLQ routing key
  'x-queue-type': 'quorum',            // Queue type
  'x-overflow': 'reject-publish',       // Overflow behavior
}
```

## Message Properties

```javascript
{
  contentType: 'application/json',
  contentEncoding: 'utf-8',
  deliveryMode: 2,                     // 1=transient, 2=persistent
  priority: 5,                         // 0-9
  correlationId: 'abc123',
  replyTo: 'reply-queue',
  expiration: '60000',                 // Message TTL (ms string)
  messageId: 'msg-123',
  timestamp: Date.now(),
  type: 'order.created',
  userId: 'guest',
  appId: 'order-service',
  headers: {
    'x-retry-count': 0,
    'x-source': 'api'
  }
}
```

## Acknowledgment Modes

| Mode | Description | Risk |
|------|-------------|------|
| `auto-ack` | Ack on delivery | Message loss on failure |
| `manual-ack` | Explicit ack after processing | Redelivery on failure |
| `nack` | Negative ack (requeue or discard) | Controlled rejection |
| `reject` | Single message rejection | Discard or requeue |

### Acknowledgment Flow

```
Broker ──deliver──▶ Consumer
                       │
                 ┌─────┴─────┐
                 │ Process   │
                 └─────┬─────┘
                       │
            ┌──────────┼──────────┐
            │          │          │
         success    failure   fatal
            │          │          │
           ack       nack      reject
          (done)   (requeue)  (discard/DLQ)
```

## Prefetch (QoS)

Controls how many messages a consumer receives before acknowledging.

```javascript
// Per consumer
channel.prefetch(10);

// Global (all consumers on channel)
channel.prefetch(100, true);
```

| Prefetch | Use Case |
|----------|----------|
| 1 | Fair distribution, slow consumers |
| 10-50 | Balanced throughput |
| 100+ | High throughput, fast consumers |

## Connection Management

### Connection vs Channel

```
Application
    │
    └── Connection (TCP)
            │
            ├── Channel 1 (Producer)
            ├── Channel 2 (Consumer A)
            └── Channel 3 (Consumer B)
```

- **Connection**: Expensive, one per application
- **Channel**: Lightweight, one per thread/operation

### Connection Recovery

```javascript
// amqplib with reconnection
const amqp = require('amqplib');

async function connect() {
  while (true) {
    try {
      const connection = await amqp.connect('amqp://localhost');
      connection.on('error', handleError);
      connection.on('close', reconnect);
      return connection;
    } catch (err) {
      console.log('Connection failed, retrying...');
      await sleep(5000);
    }
  }
}
```

## Virtual Hosts

Logical separation of resources.

```bash
# Create vhost
rabbitmqctl add_vhost /production

# Set permissions
rabbitmqctl set_permissions -p /production user ".*" ".*" ".*"

# List vhosts
rabbitmqctl list_vhosts
```

Use cases:
- Environment separation (dev/staging/prod)
- Multi-tenant applications
- Microservice isolation

## Message Flow

```
1. Publisher connects
2. Publisher declares exchange (idempotent)
3. Consumer declares queue (idempotent)
4. Consumer binds queue to exchange
5. Consumer starts consuming
6. Publisher publishes message to exchange
7. Exchange routes to matching queues
8. Broker delivers to consumer
9. Consumer processes and acknowledges
10. Broker removes message from queue
```

## Reliability Features

### Publisher Confirms

```javascript
// Enable confirms
await channel.confirmSelect();

// Publish with confirmation
channel.publish(exchange, routingKey, content);
await channel.waitForConfirms();
```

### Mandatory Flag

```javascript
// Return unroutable messages
channel.publish(exchange, routingKey, content, { mandatory: true });

channel.on('return', (msg) => {
  console.log('Message returned:', msg);
});
```

### Transactions (slow, prefer confirms)

```javascript
await channel.txSelect();
channel.publish(exchange, routingKey, content);
await channel.txCommit();
// or channel.txRollback();
```

## Best Practices

### Connection Management
- One connection per application
- One channel per thread
- Implement connection recovery
- Use heartbeats (default 60s)

### Queue Design
- Use durable queues for important messages
- Set appropriate TTL and max-length
- Configure dead-letter exchanges
- Use quorum queues for HA

### Publisher
- Enable publisher confirms
- Use persistent messages (deliveryMode: 2)
- Implement retry with backoff
- Use mandatory flag for routing verification

### Consumer
- Use manual acknowledgments
- Set appropriate prefetch
- Implement idempotent processing
- Handle redeliveries gracefully

## References

- [RabbitMQ Documentation](https://www.rabbitmq.com/documentation.html)
- [AMQP 0-9-1 Model](https://www.rabbitmq.com/tutorials/amqp-concepts.html)
- [RabbitMQ Best Practices](https://www.rabbitmq.com/production-checklist.html)
