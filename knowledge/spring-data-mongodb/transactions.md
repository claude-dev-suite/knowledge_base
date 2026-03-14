# Spring Data MongoDB Transactions

> Source: https://docs.spring.io/spring-data/mongodb/reference/mongodb/client-session-transactions.html

## Overview

MongoDB supports multi-document ACID transactions starting from version 4.0. Spring Data MongoDB provides seamless transaction support through `@Transactional` and programmatic APIs.

## Prerequisites

- MongoDB 4.0+ (replica set required)
- Spring Data MongoDB 2.1+

## Configuration

### Enable Transaction Management

```java
@Configuration
@EnableMongoRepositories
public class MongoConfig extends AbstractMongoClientConfiguration {

    @Override
    protected String getDatabaseName() {
        return "mydb";
    }

    @Bean
    MongoTransactionManager transactionManager(MongoDatabaseFactory dbFactory) {
        return new MongoTransactionManager(dbFactory);
    }
}
```

### Spring Boot Auto-Configuration

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://localhost:27017/mydb?replicaSet=rs0
```

```java
@Configuration
public class MongoConfig {

    @Bean
    MongoTransactionManager transactionManager(MongoDatabaseFactory dbFactory) {
        return new MongoTransactionManager(dbFactory);
    }
}
```

## Declarative Transactions (@Transactional)

### Basic Usage

```java
@Service
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;

    public Order createOrder(CreateOrderRequest request) {
        // All operations in single transaction
        Order order = new Order(request);
        order = orderRepository.save(order);

        for (OrderItem item : order.getItems()) {
            productRepository.decrementStock(
                item.getProductId(),
                item.getQuantity()
            );
        }

        return order;
    }
}
```

### Method-Level Transaction

```java
@Service
public class TransferService {

    @Transactional
    public void transfer(String fromAccount, String toAccount, BigDecimal amount) {
        accountRepository.debit(fromAccount, amount);
        accountRepository.credit(toAccount, amount);
        // Both succeed or both fail
    }

    // Non-transactional method
    public Account getAccount(String accountId) {
        return accountRepository.findById(accountId).orElseThrow();
    }
}
```

### Read-Only Transactions

```java
@Service
public class ReportService {

    @Transactional(readOnly = true)
    public List<OrderSummary> generateReport(LocalDate date) {
        // Consistent read across multiple queries
        List<Order> orders = orderRepository.findByDate(date);
        List<Product> products = productRepository.findAll();
        // Process and return
    }
}
```

### Transaction Propagation

```java
@Service
public class OuterService {

    @Transactional
    public void outerMethod() {
        // Start transaction
        innerService.innerMethod(); // Joins existing transaction
    }
}

@Service
public class InnerService {

    @Transactional(propagation = Propagation.REQUIRED) // Default
    public void innerMethod() {
        // Runs in existing transaction or creates new one
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void independentMethod() {
        // Always creates new transaction
    }

    @Transactional(propagation = Propagation.MANDATORY)
    public void mustHaveTransaction() {
        // Throws exception if no existing transaction
    }
}
```

### Rollback Rules

```java
@Service
public class OrderService {

    // Rollback on any exception
    @Transactional(rollbackFor = Exception.class)
    public void createOrder(OrderRequest request) {
        // ...
    }

    // Don't rollback on specific exception
    @Transactional(noRollbackFor = BusinessException.class)
    public void processOrder(String orderId) {
        // ...
    }
}
```

## Programmatic Transactions

### Using TransactionTemplate

```java
@Service
public class OrderService {

    private final TransactionTemplate transactionTemplate;
    private final OrderRepository orderRepository;

    public OrderService(MongoTransactionManager transactionManager,
                        OrderRepository orderRepository) {
        this.transactionTemplate = new TransactionTemplate(transactionManager);
        this.orderRepository = orderRepository;
    }

    public Order createOrder(CreateOrderRequest request) {
        return transactionTemplate.execute(status -> {
            try {
                Order order = orderRepository.save(new Order(request));
                updateInventory(order);
                return order;
            } catch (Exception e) {
                status.setRollbackOnly();
                throw e;
            }
        });
    }
}
```

### Using MongoTemplate with Session

```java
@Service
public class TransferService {

    private final MongoTemplate mongoTemplate;
    private final MongoClient mongoClient;

    public void transfer(String from, String to, BigDecimal amount) {
        ClientSession session = mongoClient.startSession();

        try {
            session.startTransaction();

            mongoTemplate.withSession(session).execute(operations -> {
                operations.updateFirst(
                    Query.query(Criteria.where("_id").is(from)),
                    new Update().inc("balance", amount.negate()),
                    Account.class
                );

                operations.updateFirst(
                    Query.query(Criteria.where("_id").is(to)),
                    new Update().inc("balance", amount),
                    Account.class
                );

                return null;
            });

            session.commitTransaction();
        } catch (Exception e) {
            session.abortTransaction();
            throw e;
        } finally {
            session.close();
        }
    }
}
```

### Using ReactiveMongoTemplate (Reactive)

```java
@Service
public class ReactiveOrderService {

    private final ReactiveMongoTemplate mongoTemplate;

    public Mono<Order> createOrder(CreateOrderRequest request) {
        return mongoTemplate.inTransaction()
            .execute(operations -> {
                return operations.insert(new Order(request))
                    .flatMap(order -> updateInventory(operations, order)
                        .thenReturn(order));
            })
            .single();
    }
}
```

## Transaction Options

```java
@Bean
MongoTransactionManager transactionManager(MongoDatabaseFactory dbFactory) {
    TransactionOptions options = TransactionOptions.builder()
        .readConcern(ReadConcern.SNAPSHOT)
        .writeConcern(WriteConcern.MAJORITY)
        .readPreference(ReadPreference.primary())
        .maxCommitTime(30, TimeUnit.SECONDS)
        .build();

    return new MongoTransactionManager(dbFactory, options);
}
```

## Testing Transactions

### With Testcontainers (Replica Set)

```java
@SpringBootTest
@Testcontainers
class TransactionTest {

    @Container
    @ServiceConnection
    static MongoDBContainer mongo = new MongoDBContainer("mongo:7.0")
        .withSharding();  // Enables replica set

    @Test
    @Transactional
    void shouldRollbackOnError() {
        // Test transactional behavior
    }
}
```

### Embedded MongoDB with Replica Set

```java
@DataMongoTest
@AutoConfigureDataMongo
class TransactionTest {

    @Test
    void testTransaction() {
        // Requires replica set configuration
    }
}
```

## Common Patterns

### Saga Pattern (Compensation)

```java
@Service
public class OrderSagaService {

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(new Order(request));

        try {
            paymentService.processPayment(order);
            inventoryService.reserveItems(order);
            notificationService.sendConfirmation(order);
        } catch (PaymentException e) {
            // Compensation already handled by transaction rollback
            throw e;
        } catch (InventoryException e) {
            paymentService.refund(order);  // Manual compensation
            throw e;
        }

        return order;
    }
}
```

### Optimistic Locking

```java
@Document
public class Product {
    @Id
    private String id;
    private String name;

    @Version
    private Long version;  // Automatic version increment
}

@Service
public class ProductService {

    @Transactional
    public void updateProduct(String id, UpdateRequest request) {
        Product product = productRepository.findById(id).orElseThrow();
        product.setName(request.getName());
        productRepository.save(product);  // Throws OptimisticLockingFailureException if version mismatch
    }
}
```

## Limitations

1. **Replica Set Required** - Transactions don't work on standalone MongoDB
2. **16MB Document Limit** - Still applies within transactions
3. **60-second Timeout** - Default transaction timeout
4. **No Cross-Database** - Transactions limited to single database
5. **Performance Impact** - Transactions add overhead

## Best Practices

1. **Keep transactions short** - Minimize lock time
2. **Use appropriate read/write concerns** - Balance consistency vs performance
3. **Handle retries** - Transient errors may require retry
4. **Test with replica set** - Match production configuration
5. **Consider eventual consistency** - Not all operations need transactions
6. **Monitor transaction metrics** - Watch for long-running transactions
