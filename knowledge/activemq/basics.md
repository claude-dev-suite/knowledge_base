# Apache ActiveMQ - Basics & Architecture

## Overview

Apache ActiveMQ is a popular open-source message broker that implements JMS (Java Message Service). ActiveMQ Artemis is the next-generation broker with improved performance and features.

## JMS Concepts

| Concept | Description |
|---------|-------------|
| **ConnectionFactory** | Creates connections to broker |
| **Connection** | TCP connection to broker |
| **Session** | Context for producing/consuming |
| **Destination** | Queue or Topic |
| **MessageProducer** | Sends messages |
| **MessageConsumer** | Receives messages |
| **Message** | Data being transmitted |

## Destination Types

### Queue (Point-to-Point)
```
Producer ──▶ Queue ──▶ Consumer
                │
                └── Only one consumer receives each message
```

### Topic (Publish/Subscribe)
```
Publisher ──▶ Topic ──▶ Subscriber 1
                   ──▶ Subscriber 2
                   ──▶ Subscriber N
                   │
                   └── All subscribers receive each message
```

## Message Types

| Type | Description | Use Case |
|------|-------------|----------|
| `TextMessage` | String content | JSON, XML |
| `BytesMessage` | Binary data | Files, serialized objects |
| `MapMessage` | Key-value pairs | Structured data |
| `ObjectMessage` | Serialized Java object | Java-to-Java |
| `StreamMessage` | Sequential primitives | Ordered data |

## Spring JMS Integration

### Configuration
```java
@Configuration
@EnableJms
public class JmsConfig {
    @Bean
    public ConnectionFactory connectionFactory() {
        ActiveMQConnectionFactory factory = new ActiveMQConnectionFactory();
        factory.setBrokerURL("tcp://localhost:61616");
        factory.setUserName("admin");
        factory.setPassword("admin");
        return new CachingConnectionFactory(factory);
    }

    @Bean
    public JmsTemplate jmsTemplate(ConnectionFactory connectionFactory) {
        JmsTemplate template = new JmsTemplate(connectionFactory);
        template.setDeliveryPersistent(true);
        return template;
    }

    @Bean
    public DefaultJmsListenerContainerFactory jmsListenerContainerFactory(
            ConnectionFactory connectionFactory) {
        DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setConcurrency("3-10");
        factory.setSessionAcknowledgeMode(Session.CLIENT_ACKNOWLEDGE);
        return factory;
    }
}
```

### Producer
```java
@Service
public class OrderProducer {
    @Autowired
    private JmsTemplate jmsTemplate;

    public void sendOrder(Order order) {
        jmsTemplate.convertAndSend("orders.queue", order, message -> {
            message.setJMSCorrelationID(UUID.randomUUID().toString());
            message.setStringProperty("orderType", order.getType());
            return message;
        });
    }
}
```

### Consumer
```java
@Service
public class OrderConsumer {
    @JmsListener(destination = "orders.queue")
    public void consume(@Payload Order order,
                       @Header(JmsHeaders.CORRELATION_ID) String correlationId,
                       Session session,
                       Message message) throws JMSException {
        try {
            processOrder(order);
            message.acknowledge();
        } catch (Exception e) {
            session.recover();
        }
    }
}
```

## Virtual Topics

Queue semantics with topic flexibility.

```
Publisher ──▶ VirtualTopic.Orders ──▶ Consumer.A.VirtualTopic.Orders (Group A)
                                 ──▶ Consumer.B.VirtualTopic.Orders (Group B)
```

```java
// Producer sends to virtual topic
jmsTemplate.convertAndSend("VirtualTopic.Orders", order);

// Consumers subscribe to virtual consumer queues
@JmsListener(destination = "Consumer.ProcessorA.VirtualTopic.Orders")
public void consumeA(Order order) { /* load balanced within group A */ }

@JmsListener(destination = "Consumer.ProcessorB.VirtualTopic.Orders")
public void consumeB(Order order) { /* load balanced within group B */ }
```

## Message Groups

Ensures ordered processing by grouping related messages.

```java
// Producer sets group
jmsTemplate.convertAndSend("orders.queue", order, message -> {
    message.setStringProperty("JMSXGroupID", order.getCustomerId());
    message.setIntProperty("JMSXGroupSeq", sequenceNumber);
    return message;
});

// Consumer receives messages in order per group
```

## Acknowledgment Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `AUTO_ACKNOWLEDGE` | Auto ack after receive | Simple cases |
| `CLIENT_ACKNOWLEDGE` | Manual ack | At-least-once |
| `DUPS_OK_ACKNOWLEDGE` | Lazy ack | High throughput |
| `SESSION_TRANSACTED` | Transactional | Exactly-once |

## Best Practices

### Producer
- Use persistent delivery for important messages
- Set appropriate message expiration
- Use message groups for ordering
- Enable producer flow control

### Consumer
- Use client acknowledge for reliability
- Implement idempotent processing
- Configure appropriate prefetch
- Handle redeliveries gracefully

### General
- Use virtual topics for flexible pub/sub
- Configure dead letter queues
- Monitor queue depths
- Use connection pooling

## References

- [ActiveMQ Documentation](https://activemq.apache.org/components/artemis/documentation/)
- [JMS Tutorial](https://docs.oracle.com/javaee/7/tutorial/jms-concepts.htm)
