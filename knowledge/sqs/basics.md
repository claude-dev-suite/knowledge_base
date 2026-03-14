# Amazon SQS - Basics & Architecture

## Overview

Amazon Simple Queue Service (SQS) is a fully managed message queuing service. It offers two queue types: Standard (high throughput) and FIFO (ordering guarantee).

## Queue Types

| Feature | Standard | FIFO |
|---------|----------|------|
| Throughput | Unlimited | 300 TPS (3000 with batching) |
| Ordering | Best-effort | Strict (per message group) |
| Delivery | At-least-once | Exactly-once |
| Deduplication | None | 5-minute window |

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Queue** | Message storage endpoint |
| **Visibility Timeout** | Message lock period |
| **Dead Letter Queue** | Failed message destination |
| **Message Group ID** | FIFO ordering key |
| **Deduplication ID** | FIFO duplicate prevention |
| **Long Polling** | Efficient message retrieval |

## Producer Patterns

### Node.js
```typescript
import { SQSClient, SendMessageCommand } from '@aws-sdk/client-sqs';

const client = new SQSClient({ region: 'us-east-1' });

// Standard queue
await client.send(new SendMessageCommand({
  QueueUrl: queueUrl,
  MessageBody: JSON.stringify(order),
  MessageAttributes: {
    'OrderType': { DataType: 'String', StringValue: order.type }
  },
  DelaySeconds: 0
}));

// FIFO queue
await client.send(new SendMessageCommand({
  QueueUrl: fifoQueueUrl,
  MessageBody: JSON.stringify(order),
  MessageGroupId: order.customerId,
  MessageDeduplicationId: order.orderId
}));
```

### Java
```java
@Service
public class SqsProducer {
    @Autowired
    private SqsClient sqsClient;

    public void sendMessage(Order order) {
        sqsClient.sendMessage(SendMessageRequest.builder()
            .queueUrl(queueUrl)
            .messageBody(objectMapper.writeValueAsString(order))
            .messageAttributes(Map.of(
                "OrderType", MessageAttributeValue.builder()
                    .dataType("String")
                    .stringValue(order.getType())
                    .build()))
            .build());
    }
}
```

## Consumer Patterns

### Node.js
```typescript
async function pollMessages() {
  while (true) {
    const response = await client.send(new ReceiveMessageCommand({
      QueueUrl: queueUrl,
      MaxNumberOfMessages: 10,
      WaitTimeSeconds: 20,  // Long polling
      VisibilityTimeout: 30,
      MessageAttributeNames: ['All']
    }));

    for (const message of response.Messages || []) {
      try {
        await processMessage(JSON.parse(message.Body));
        await client.send(new DeleteMessageCommand({
          QueueUrl: queueUrl,
          ReceiptHandle: message.ReceiptHandle
        }));
      } catch (error) {
        // Message returns to queue after visibility timeout
      }
    }
  }
}
```

### Lambda Integration
```typescript
export const handler = async (event: SQSEvent): Promise<SQSBatchResponse> => {
  const failures: SQSBatchItemFailure[] = [];

  for (const record of event.Records) {
    try {
      await processMessage(JSON.parse(record.body));
    } catch (error) {
      failures.push({ itemIdentifier: record.messageId });
    }
  }

  return { batchItemFailures: failures };
};
```

## Dead Letter Queue

```hcl
# Terraform
resource "aws_sqs_queue" "orders_dlq" {
  name = "orders-dlq"
}

resource "aws_sqs_queue" "orders" {
  name = "orders"
  visibility_timeout_seconds = 30

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.orders_dlq.arn
    maxReceiveCount     = 3
  })
}
```

## Best Practices

### Producer
- Use batch operations for efficiency
- Set appropriate message attributes
- Use FIFO for ordering requirements
- Implement retry with backoff

### Consumer
- Use long polling (20 seconds)
- Delete messages only after successful processing
- Set appropriate visibility timeout
- Use batch delete for efficiency

### Security
- Use IAM roles, not access keys
- Enable server-side encryption
- Use VPC endpoints for private access
- Apply least privilege permissions

## References

- [SQS Documentation](https://docs.aws.amazon.com/sqs/)
- [SQS Best Practices](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-best-practices.html)
