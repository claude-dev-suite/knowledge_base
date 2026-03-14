# Google Cloud Pub/Sub - Basics & Architecture

## Overview

Google Cloud Pub/Sub is a fully-managed real-time messaging service for event-driven systems and streaming analytics.

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Topic** | Named resource for publishing |
| **Subscription** | Named resource for receiving |
| **Publisher** | Sends messages to topic |
| **Subscriber** | Receives from subscription |
| **Ack Deadline** | Time to acknowledge |

## Delivery Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Pull** | Subscriber requests messages | Batch processing |
| **Push** | Pub/Sub sends to endpoint | Serverless |
| **BigQuery** | Export to BigQuery | Analytics |
| **Cloud Storage** | Export to GCS | Archival |

## Producer

```typescript
import { PubSub } from '@google-cloud/pubsub';

const pubsub = new PubSub({ projectId: 'my-project' });
const topic = pubsub.topic('orders');

const messageId = await topic.publishMessage({
  data: Buffer.from(JSON.stringify(order)),
  attributes: { 'correlation-id': correlationId },
  orderingKey: order.customerId
});
```

## Consumer (Pull)

```typescript
const subscription = pubsub.subscription('order-processor', {
  flowControl: { maxMessages: 100 },
  ackDeadline: 30
});

subscription.on('message', async (message) => {
  try {
    await processOrder(JSON.parse(message.data.toString()));
    message.ack();
  } catch (error) {
    message.nack();
  }
});
```

## Consumer (Push - Cloud Run)

```typescript
app.post('/pubsub', async (req, res) => {
  const message = req.body.message;
  const order = JSON.parse(Buffer.from(message.data, 'base64').toString());

  try {
    await processOrder(order);
    res.status(200).send('OK');
  } catch (error) {
    res.status(500).send('Error');
  }
});
```

## Dead Letter Topics

```hcl
resource "google_pubsub_subscription" "order_processor" {
  name  = "order-processor"
  topic = google_pubsub_topic.orders.name

  dead_letter_policy {
    dead_letter_topic     = google_pubsub_topic.orders_dlq.id
    max_delivery_attempts = 5
  }
}
```

## Best Practices

- Use IAM for authentication
- Configure dead letter topics
- Set appropriate ack deadline
- Use ordering keys when needed
- Monitor with Cloud Monitoring

## References

- [Pub/Sub Documentation](https://cloud.google.com/pubsub/docs)
