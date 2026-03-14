# RabbitMQ - Exchanges

## Exchange Types Overview

| Type | Routing | Use Case |
|------|---------|----------|
| **direct** | Exact routing key match | Task queues, RPC |
| **topic** | Pattern matching (*, #) | Event routing, logging |
| **fanout** | Broadcast all | Notifications, pub/sub |
| **headers** | Header-based | Complex routing logic |

## Direct Exchange

Routes messages where routing key exactly matches binding key.

### Architecture
```
Publisher ─(order.created)─▶ Direct Exchange
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
            order.created    order.updated    order.deleted
                    │               │               │
                    ▼               ▼               ▼
              Queue A          Queue B          Queue C
```

### Implementation

```javascript
// Node.js (amqplib)
const exchange = 'orders';

// Declare exchange
await channel.assertExchange(exchange, 'direct', { durable: true });

// Declare and bind queue
await channel.assertQueue('order-created-queue', { durable: true });
await channel.bindQueue('order-created-queue', exchange, 'order.created');

// Publish
channel.publish(exchange, 'order.created', Buffer.from(JSON.stringify(order)), {
  persistent: true,
});
```

```java
// Java (Spring AMQP)
@Configuration
public class DirectExchangeConfig {
    @Bean
    public DirectExchange ordersExchange() {
        return new DirectExchange("orders");
    }

    @Bean
    public Queue orderCreatedQueue() {
        return QueueBuilder.durable("order-created-queue").build();
    }

    @Bean
    public Binding orderCreatedBinding() {
        return BindingBuilder.bind(orderCreatedQueue())
            .to(ordersExchange())
            .with("order.created");
    }
}
```

### Default Exchange

Pre-declared direct exchange with empty name (""). Routes to queue with name matching routing key.

```javascript
// Publish directly to queue
channel.sendToQueue('my-queue', Buffer.from(message));

// Equivalent to:
channel.publish('', 'my-queue', Buffer.from(message));
```

## Topic Exchange

Routes based on wildcard patterns in routing key.

### Wildcards
- `*` - matches exactly one word
- `#` - matches zero or more words

### Pattern Examples
```
Routing Keys:
  orders.us.created
  orders.eu.created
  orders.us.item.added
  logs.error.database
  logs.warn.api

Binding Patterns:
  orders.*.*        → orders.us.created, orders.eu.created
  orders.#          → all orders.*
  orders.us.#       → orders.us.created, orders.us.item.added
  *.error.*         → logs.error.database
  logs.#            → all logs.*
  #                 → all messages (catch-all)
```

### Implementation

```javascript
// Node.js
await channel.assertExchange('events', 'topic', { durable: true });

// Bind with pattern
await channel.bindQueue('us-orders', 'events', 'orders.us.*');
await channel.bindQueue('all-orders', 'events', 'orders.#');
await channel.bindQueue('all-errors', 'events', '*.error.*');

// Publish
channel.publish('events', 'orders.us.created', Buffer.from(data));
```

```java
// Java
@Bean
public TopicExchange eventsExchange() {
    return new TopicExchange("events");
}

@Bean
public Queue usOrdersQueue() {
    return new Queue("us-orders");
}

@Bean
public Binding usOrdersBinding() {
    return BindingBuilder.bind(usOrdersQueue())
        .to(eventsExchange())
        .with("orders.us.*");
}
```

## Fanout Exchange

Broadcasts messages to all bound queues. Routing key is ignored.

### Architecture
```
Publisher ──▶ Fanout Exchange
                    │
         ┌──────────┼──────────┐
         │          │          │
         ▼          ▼          ▼
     Queue A    Queue B    Queue C
     (Email)   (SMS)      (Push)
```

### Implementation

```javascript
// Node.js
await channel.assertExchange('notifications', 'fanout', { durable: true });

// Bind queues (no routing key needed)
await channel.bindQueue('email-notifications', 'notifications', '');
await channel.bindQueue('sms-notifications', 'notifications', '');
await channel.bindQueue('push-notifications', 'notifications', '');

// Publish (routing key ignored)
channel.publish('notifications', '', Buffer.from(notification));
```

### Use Cases
- Broadcasting notifications
- Cache invalidation
- Real-time updates to multiple services
- Log distribution

## Headers Exchange

Routes based on message header attributes instead of routing key.

### Matching Modes
- `x-match: all` - all headers must match
- `x-match: any` - at least one header must match

### Implementation

```javascript
// Node.js
await channel.assertExchange('documents', 'headers', { durable: true });

// Bind with headers
await channel.bindQueue('pdf-processor', 'documents', '', {
  'x-match': 'all',
  'format': 'pdf',
  'type': 'report'
});

await channel.bindQueue('image-processor', 'documents', '', {
  'x-match': 'any',
  'format': 'jpg',
  'format': 'png'
});

// Publish with headers
channel.publish('documents', '', Buffer.from(doc), {
  headers: {
    'format': 'pdf',
    'type': 'report'
  }
});
```

```java
// Java
@Bean
public HeadersExchange documentsExchange() {
    return new HeadersExchange("documents");
}

@Bean
public Binding pdfBinding() {
    return BindingBuilder.bind(pdfQueue())
        .to(documentsExchange())
        .whereAll(Map.of("format", "pdf", "type", "report"))
        .match();
}
```

## Exchange Properties

| Property | Description |
|----------|-------------|
| `name` | Exchange name |
| `type` | direct, topic, fanout, headers |
| `durable` | Survives broker restart |
| `autoDelete` | Deleted when last queue unbinds |
| `internal` | Cannot receive from publishers (only other exchanges) |
| `arguments` | Custom arguments |

### Declaration

```javascript
await channel.assertExchange('my-exchange', 'topic', {
  durable: true,
  autoDelete: false,
  internal: false,
  arguments: {
    'alternate-exchange': 'unrouted'
  }
});
```

## Alternate Exchange

Captures messages that can't be routed to any queue.

```javascript
// Declare alternate exchange
await channel.assertExchange('unrouted', 'fanout', { durable: true });
await channel.assertQueue('unrouted-messages', { durable: true });
await channel.bindQueue('unrouted-messages', 'unrouted', '');

// Declare main exchange with alternate
await channel.assertExchange('orders', 'direct', {
  durable: true,
  arguments: {
    'alternate-exchange': 'unrouted'
  }
});
```

## Exchange-to-Exchange Binding

Route messages between exchanges.

```javascript
// Source exchange
await channel.assertExchange('events', 'topic', { durable: true });

// Destination exchange
await channel.assertExchange('orders', 'direct', { durable: true });

// Bind exchanges
await channel.bindExchange('orders', 'events', 'orders.*');

// Messages to events with 'orders.*' go to orders exchange
```

## Dead Letter Exchange (DLX)

Receives messages that are:
- Rejected with requeue=false
- TTL expired
- Queue max-length exceeded

```javascript
// Declare DLX
await channel.assertExchange('orders.dlx', 'direct', { durable: true });
await channel.assertQueue('orders.dlq', { durable: true });
await channel.bindQueue('orders.dlq', 'orders.dlx', 'orders');

// Declare main queue with DLX
await channel.assertQueue('orders', {
  durable: true,
  arguments: {
    'x-dead-letter-exchange': 'orders.dlx',
    'x-dead-letter-routing-key': 'orders'
  }
});
```

## Best Practices

### Exchange Design
- Use meaningful, namespaced names
- Prefer topic over direct for flexibility
- Use fanout for true broadcast
- Set up alternate exchanges for unrouted messages

### Performance
- Minimize exchange-to-exchange bindings
- Use direct for high-volume, simple routing
- Limit topic pattern complexity
- Consider consistent hashing exchange (plugin) for load balancing

### Reliability
- Use durable exchanges
- Configure DLX for failure handling
- Test routing before production
- Monitor unrouted messages

## CLI Commands

```bash
# List exchanges
rabbitmqctl list_exchanges

# Declare exchange
rabbitmqadmin declare exchange name=orders type=topic durable=true

# List bindings
rabbitmqctl list_bindings

# Delete exchange
rabbitmqadmin delete exchange name=orders
```

## References

- [Exchange Documentation](https://www.rabbitmq.com/tutorials/amqp-concepts.html#exchanges)
- [Consistent Hash Exchange](https://github.com/rabbitmq/rabbitmq-consistent-hash-exchange)
