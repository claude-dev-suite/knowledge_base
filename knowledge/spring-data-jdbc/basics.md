# Spring Data JDBC - Basics

## Overview

Spring Data JDBC provides a simple, domain-driven approach to database access. Unlike JPA, it follows Domain-Driven Design principles with explicit aggregate boundaries and no lazy loading.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
<!-- Or PostgreSQL, MySQL, etc. -->
```

## Configuration

### application.yml
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  sql:
    init:
      mode: always
      schema-locations: classpath:schema.sql
```

### Java Configuration
```java
@Configuration
@EnableJdbcRepositories
public class JdbcConfig extends AbstractJdbcConfiguration {

    @Bean
    public DataSource dataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        dataSource.setUsername("user");
        dataSource.setPassword("password");
        dataSource.setMaximumPoolSize(10);
        return dataSource;
    }

    @Override
    public JdbcCustomConversions jdbcCustomConversions() {
        return new JdbcCustomConversions(List.of(
            new MoneyToStringConverter(),
            new StringToMoneyConverter()
        ));
    }
}
```

## Entity Definition

### Basic Entity
```java
@Table("customers")
public class Customer {

    @Id
    private Long id;

    private String firstName;
    private String lastName;
    private String email;

    @Column("created_at")
    private LocalDateTime createdAt;

    // Constructors, getters, setters
}
```

### With Embedded Value
```java
@Table("orders")
public class Order {

    @Id
    private Long id;

    private String orderNumber;

    @Embedded(onEmpty = Embedded.OnEmpty.USE_NULL)
    private Address shippingAddress;

    @Embedded(onEmpty = Embedded.OnEmpty.USE_EMPTY, prefix = "billing_")
    private Address billingAddress;

    private LocalDateTime createdAt;
}

public class Address {
    private String street;
    private String city;
    private String postalCode;
    private String country;
}
```

### ID Generation Strategies
```java
// Database auto-generated (IDENTITY, SERIAL)
@Table("products")
public class Product {
    @Id
    private Long id;  // Set by database
    private String name;
}

// Application-provided
@Table("categories")
public class Category {
    @Id
    private String code;  // Provided by application
    private String name;
}

// BeforeSave callback for UUID
@Table("events")
public class Event {
    @Id
    private UUID id;
    private String type;
}

@Component
public class EventIdGenerator implements BeforeConvertCallback<Event> {
    @Override
    public Event onBeforeConvert(Event event) {
        if (event.getId() == null) {
            event.setId(UUID.randomUUID());
        }
        return event;
    }
}
```

## Schema Definition

```sql
-- schema.sql
CREATE TABLE IF NOT EXISTS customers (
    id BIGSERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(50) UNIQUE NOT NULL,
    customer_id BIGINT REFERENCES customers(id),
    shipping_street VARCHAR(255),
    shipping_city VARCHAR(100),
    shipping_postal_code VARCHAR(20),
    shipping_country VARCHAR(100),
    billing_street VARCHAR(255),
    billing_city VARCHAR(100),
    billing_postal_code VARCHAR(20),
    billing_country VARCHAR(100),
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS order_items (
    order_id BIGINT REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL,
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

## JdbcTemplate Integration

```java
@Service
@RequiredArgsConstructor
public class CustomerService {

    private final JdbcTemplate jdbcTemplate;

    public Customer findById(Long id) {
        return jdbcTemplate.queryForObject(
            "SELECT * FROM customers WHERE id = ?",
            (rs, rowNum) -> new Customer(
                rs.getLong("id"),
                rs.getString("first_name"),
                rs.getString("last_name"),
                rs.getString("email")
            ),
            id
        );
    }

    public List<Customer> findAll() {
        return jdbcTemplate.query(
            "SELECT * FROM customers ORDER BY last_name",
            (rs, rowNum) -> new Customer(
                rs.getLong("id"),
                rs.getString("first_name"),
                rs.getString("last_name"),
                rs.getString("email")
            )
        );
    }

    public void updateEmail(Long id, String email) {
        jdbcTemplate.update(
            "UPDATE customers SET email = ? WHERE id = ?",
            email, id
        );
    }
}
```

## NamedParameterJdbcTemplate

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final NamedParameterJdbcTemplate namedJdbcTemplate;

    public List<Order> findByCustomer(Long customerId) {
        String sql = """
            SELECT * FROM orders
            WHERE customer_id = :customerId
            ORDER BY created_at DESC
            """;

        MapSqlParameterSource params = new MapSqlParameterSource()
            .addValue("customerId", customerId);

        return namedJdbcTemplate.query(sql, params, new OrderRowMapper());
    }

    public List<Order> findByDateRange(LocalDate start, LocalDate end) {
        String sql = """
            SELECT * FROM orders
            WHERE created_at BETWEEN :start AND :end
            ORDER BY created_at
            """;

        Map<String, Object> params = Map.of(
            "start", start,
            "end", end
        );

        return namedJdbcTemplate.query(sql, params, new OrderRowMapper());
    }
}
```

## Custom Converters

```java
// Write converter
@WritingConverter
public class MoneyToStringConverter implements Converter<Money, String> {
    @Override
    public String convert(Money source) {
        return source.getCurrency() + ":" + source.getAmount();
    }
}

// Read converter
@ReadingConverter
public class StringToMoneyConverter implements Converter<String, Money> {
    @Override
    public Money convert(String source) {
        String[] parts = source.split(":");
        return new Money(parts[0], new BigDecimal(parts[1]));
    }
}

// Enum converter
@WritingConverter
public class StatusToStringConverter implements Converter<OrderStatus, String> {
    @Override
    public String convert(OrderStatus source) {
        return source.name();
    }
}

@ReadingConverter
public class StringToStatusConverter implements Converter<String, OrderStatus> {
    @Override
    public OrderStatus convert(String source) {
        return OrderStatus.valueOf(source);
    }
}
```

## Auditing

```java
@Configuration
@EnableJdbcAuditing
public class AuditConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(
            SecurityContextHolder.getContext().getAuthentication()
        ).map(Authentication::getName);
    }
}

@Table("products")
public class Product {

    @Id
    private Long id;

    private String name;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;
}
```

## Transactions

```java
@Service
@Transactional
@RequiredArgsConstructor
public class OrderProcessingService {

    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;

    public Order processOrder(Order order) {
        // Check inventory
        for (OrderItem item : order.getItems()) {
            inventoryService.reserveStock(item.getProductId(), item.getQuantity());
        }

        // Save order
        return orderRepository.save(order);
    }

    @Transactional(readOnly = true)
    public List<Order> findAllOrders() {
        return orderRepository.findAll();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logOrderEvent(Long orderId, String event) {
        // Separate transaction for logging
    }
}
```

## Comparison with JPA

| Aspect | Spring Data JDBC | Spring Data JPA |
|--------|------------------|-----------------|
| Lazy loading | Not supported | Supported |
| Caching | Not supported | First/Second level |
| Dirty tracking | Not supported | Automatic |
| Aggregate roots | Explicit boundaries | Implicit |
| Complexity | Simple | Complex |
| Learning curve | Lower | Higher |
| Control | More explicit | More magic |

## Best Practices

| Do | Don't |
|----|-------|
| Define clear aggregate boundaries | Mix aggregates |
| Use immutable entities when possible | Mutate loaded entities |
| Keep aggregates small | Large object graphs |
| Handle references explicitly | Expect lazy loading |
| Use embedded for value objects | Create separate tables unnecessarily |
| Write explicit queries | Rely on query derivation for complex queries |

## Production Checklist

- [ ] Schema managed properly (migrations)
- [ ] ID generation strategy chosen
- [ ] Converters registered for custom types
- [ ] Auditing configured
- [ ] Transaction boundaries defined
- [ ] Connection pooling configured
- [ ] Indexes created for queries
