# Spring Data JDBC - Aggregates

## Overview

Spring Data JDBC embraces Domain-Driven Design concepts, treating entities as aggregates with clear boundaries. An aggregate is a cluster of entities that are treated as a single unit.

## Aggregate Roots

### Basic Aggregate
```java
// Order is the aggregate root
@Table("orders")
public class Order {

    @Id
    private Long id;

    private String orderNumber;
    private OrderStatus status;

    // Order items belong to this aggregate
    @MappedCollection(idColumn = "order_id")
    private Set<OrderItem> items = new HashSet<>();

    private LocalDateTime createdAt;

    // Business methods
    public void addItem(Long productId, int quantity, BigDecimal unitPrice) {
        OrderItem item = new OrderItem(productId, quantity, unitPrice);
        items.add(item);
    }

    public void removeItem(Long productId) {
        items.removeIf(item -> item.getProductId().equals(productId));
    }

    public BigDecimal getTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

// OrderItem is part of the Order aggregate
@Table("order_items")
public class OrderItem {

    private Long productId;
    private Integer quantity;
    private BigDecimal unitPrice;

    public OrderItem(Long productId, Integer quantity, BigDecimal unitPrice) {
        this.productId = productId;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
    }

    public BigDecimal getSubtotal() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }
}
```

### Schema
```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(50) UNIQUE NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL,
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

## One-to-Many Relationships

### Set-based Collection
```java
@Table("posts")
public class Post {

    @Id
    private Long id;

    private String title;
    private String content;

    @MappedCollection(idColumn = "post_id")
    private Set<Comment> comments = new HashSet<>();

    public void addComment(String author, String text) {
        comments.add(new Comment(author, text));
    }
}

@Table("comments")
public class Comment {

    private String author;
    private String text;
    private LocalDateTime createdAt;

    public Comment(String author, String text) {
        this.author = author;
        this.text = text;
        this.createdAt = LocalDateTime.now();
    }
}
```

### List-based Collection (Ordered)
```java
@Table("surveys")
public class Survey {

    @Id
    private Long id;

    private String title;

    // Ordered collection
    @MappedCollection(idColumn = "survey_id", keyColumn = "question_order")
    private List<Question> questions = new ArrayList<>();

    public void addQuestion(String text, QuestionType type) {
        questions.add(new Question(text, type));
    }

    public void reorderQuestion(int fromIndex, int toIndex) {
        Question question = questions.remove(fromIndex);
        questions.add(toIndex, question);
    }
}

@Table("questions")
public class Question {

    private String text;
    private QuestionType type;

    @MappedCollection(idColumn = "question_id", keyColumn = "option_order")
    private List<Option> options = new ArrayList<>();
}
```

### Map-based Collection
```java
@Table("products")
public class Product {

    @Id
    private Long id;

    private String name;

    // Map with locale as key
    @MappedCollection(idColumn = "product_id", keyColumn = "locale")
    private Map<String, ProductTranslation> translations = new HashMap<>();

    public void addTranslation(String locale, String name, String description) {
        translations.put(locale, new ProductTranslation(name, description));
    }

    public String getNameForLocale(String locale) {
        ProductTranslation translation = translations.get(locale);
        return translation != null ? translation.getName() : name;
    }
}

@Table("product_translations")
public class ProductTranslation {

    private String name;
    private String description;
}
```

## One-to-One Relationships

### Embedded Entity
```java
@Table("users")
public class User {

    @Id
    private Long id;

    private String username;

    @Embedded(onEmpty = Embedded.OnEmpty.USE_NULL)
    private Profile profile;

    @Embedded(onEmpty = Embedded.OnEmpty.USE_NULL, prefix = "address_")
    private Address address;
}

public class Profile {
    private String bio;
    private String avatarUrl;
}

public class Address {
    private String street;
    private String city;
    private String postalCode;
}
```

### Separate Table
```java
@Table("accounts")
public class Account {

    @Id
    private Long id;

    private String accountNumber;

    @MappedCollection(idColumn = "account_id")
    private AccountSettings settings;
}

@Table("account_settings")
public class AccountSettings {

    private boolean emailNotifications;
    private boolean smsNotifications;
    private String timezone;
}
```

## References to Other Aggregates

```java
@Table("orders")
public class Order {

    @Id
    private Long id;

    // Reference to Customer aggregate (not embedded)
    private Long customerId;

    // Reference to Address aggregate
    private Long shippingAddressId;

    @MappedCollection(idColumn = "order_id")
    private Set<OrderItem> items = new HashSet<>();
}

// Customer is its own aggregate
@Table("customers")
public class Customer {

    @Id
    private Long id;

    private String name;
    private String email;

    @MappedCollection(idColumn = "customer_id")
    private Set<Address> addresses = new HashSet<>();
}

// Service handles cross-aggregate operations
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final CustomerRepository customerRepository;

    @Transactional
    public Order createOrder(Long customerId, List<OrderItemRequest> items) {
        // Verify customer exists
        Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new CustomerNotFoundException(customerId));

        Order order = new Order();
        order.setCustomerId(customerId);
        order.setOrderNumber(generateOrderNumber());

        for (OrderItemRequest item : items) {
            order.addItem(item.getProductId(), item.getQuantity(), item.getUnitPrice());
        }

        return orderRepository.save(order);
    }

    public OrderWithCustomer getOrderWithCustomer(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        Customer customer = customerRepository.findById(order.getCustomerId()).orElseThrow();
        return new OrderWithCustomer(order, customer);
    }
}
```

## Aggregate Loading

### Loading Complete Aggregate
```java
// When loading Order, all items are loaded
Order order = orderRepository.findById(orderId).orElseThrow();
// order.getItems() is already populated - no lazy loading
```

### Custom Query for Partial Load
```java
public interface OrderRepository extends CrudRepository<Order, Long> {

    // Load only order without items
    @Query("SELECT * FROM orders WHERE id = :id")
    Optional<Order> findByIdWithoutItems(Long id);

    // Load with specific projection
    @Query("""
        SELECT id, order_number, status
        FROM orders
        WHERE customer_id = :customerId
        """)
    List<OrderSummary> findSummariesByCustomerId(Long customerId);
}
```

## Aggregate Consistency

```java
@Table("bank_accounts")
public class BankAccount {

    @Id
    private Long id;

    private String accountNumber;
    private BigDecimal balance;

    @MappedCollection(idColumn = "account_id")
    private List<Transaction> transactions = new ArrayList<>();

    @Version
    private Long version;  // Optimistic locking

    public void deposit(BigDecimal amount) {
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }

        this.balance = this.balance.add(amount);
        this.transactions.add(new Transaction(TransactionType.DEPOSIT, amount));
    }

    public void withdraw(BigDecimal amount) {
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }

        if (amount.compareTo(this.balance) > 0) {
            throw new InsufficientFundsException();
        }

        this.balance = this.balance.subtract(amount);
        this.transactions.add(new Transaction(TransactionType.WITHDRAWAL, amount));
    }
}
```

## Aggregate Immutability

```java
// Immutable aggregate using records (Java 16+)
@Table("invoices")
public record Invoice(
    @Id Long id,
    String invoiceNumber,
    Long customerId,
    BigDecimal amount,
    InvoiceStatus status,
    @MappedCollection(idColumn = "invoice_id") Set<InvoiceLine> lines,
    LocalDateTime createdAt
) {
    // Factory methods for state transitions
    public Invoice markAsPaid() {
        return new Invoice(id, invoiceNumber, customerId, amount,
            InvoiceStatus.PAID, lines, createdAt);
    }

    public Invoice cancel() {
        return new Invoice(id, invoiceNumber, customerId, amount,
            InvoiceStatus.CANCELLED, lines, createdAt);
    }
}

// Usage
@Service
@Transactional
public class InvoiceService {

    private final InvoiceRepository repository;

    public Invoice payInvoice(Long invoiceId) {
        Invoice invoice = repository.findById(invoiceId).orElseThrow();
        Invoice paidInvoice = invoice.markAsPaid();
        return repository.save(paidInvoice);
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Keep aggregates small | Large aggregate graphs |
| Reference other aggregates by ID | Embed other aggregates |
| Put business logic in aggregate root | Anemic domain model |
| Use value objects for concepts | Primitive obsession |
| Enforce invariants in aggregate | Validate outside aggregate |
| Design for eventual consistency | Expect strong consistency across aggregates |

## Production Checklist

- [ ] Aggregate boundaries clearly defined
- [ ] References by ID to other aggregates
- [ ] Business rules in aggregate root
- [ ] Cascade delete configured in schema
- [ ] Optimistic locking where needed
- [ ] Aggregate size appropriate
- [ ] Cross-aggregate consistency handled
