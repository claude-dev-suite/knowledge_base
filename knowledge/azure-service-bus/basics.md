# Azure Service Bus - Basics & Architecture

## Overview

Azure Service Bus is a fully managed enterprise message broker with message queues and publish-subscribe topics.

## Tiers

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Queues | Yes | Yes | Yes |
| Topics | No | Yes | Yes |
| Sessions | No | Yes | Yes |
| Max message size | 256KB | 256KB | 100MB |
| Throughput | Shared | Shared | Dedicated |

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Namespace** | Container for messaging entities |
| **Queue** | Point-to-point messaging |
| **Topic** | Publish-subscribe messaging |
| **Subscription** | Topic consumer with filters |
| **Session** | Ordered message processing |

## Producer

```typescript
import { ServiceBusClient } from '@azure/service-bus';

const client = new ServiceBusClient(connectionString);
const sender = client.createSender('orders');

await sender.sendMessages({
  body: order,
  messageId: order.id,
  correlationId: correlationId,
  applicationProperties: { orderType: order.type }
});

await sender.close();
await client.close();
```

## Consumer

```typescript
const receiver = client.createReceiver('orders', { receiveMode: 'peekLock' });

const messageHandler = async (message) => {
  try {
    await processOrder(message.body);
    await receiver.completeMessage(message);
  } catch (error) {
    if (message.deliveryCount >= 3) {
      await receiver.deadLetterMessage(message);
    } else {
      await receiver.abandonMessage(message);
    }
  }
};

receiver.subscribe({ processMessage: messageHandler, processError: errorHandler });
```

## Sessions

For ordered message processing:

```typescript
const sessionReceiver = await client.acceptSession('orders', 'session-1');
const messages = await sessionReceiver.receiveMessages(10);
for (const msg of messages) {
  await processMessage(msg);
  await sessionReceiver.completeMessage(msg);
}
```

## Subscription Filters

```bash
# SQL filter
az servicebus topic subscription rule create \
  --subscription-name high-priority \
  --filter-sql-expression "priority > 5"
```

## Best Practices

- Use Managed Identity authentication
- Configure dead-letter queues
- Use sessions for ordering
- Enable duplicate detection
- Monitor with Azure Monitor

## References

- [Service Bus Documentation](https://docs.microsoft.com/azure/service-bus-messaging/)
