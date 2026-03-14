# Spring Data JDBC - Repositories

## Overview

Spring Data JDBC repositories provide a familiar repository abstraction with support for derived queries, custom queries, and aggregate-aware operations.

## Enable Repositories

```java
@Configuration
@EnableJdbcRepositories(basePackages = "com.example.repository")
public class JdbcRepositoryConfig extends AbstractJdbcConfiguration {
    // Configuration
}
```

## Basic Repository

```java
public interface CustomerRepository extends CrudRepository<Customer, Long> {

    // Derived queries
    List<Customer> findByLastName(String lastName);

    Optional<Customer> findByEmail(String email);

    List<Customer> findByFirstNameAndLastName(String firstName, String lastName);

    boolean existsByEmail(String email);

    long countByLastName(String lastName);

    void deleteByEmail(String email);
}
```

## Derived Query Keywords

```java
public interface ProductRepository extends CrudRepository<Product, Long> {

    // Equality
    List<Product> findByName(String name);
    List<Product> findByNameIs(String name);
    List<Product> findByNameEquals(String name);

    // Not equal
    List<Product> findByNameNot(String name);

    // Null checks
    List<Product> findByDescriptionIsNull();
    List<Product> findByDescriptionIsNotNull();

    // Like patterns
    List<Product> findByNameLike(String pattern);
    List<Product> findByNameStartingWith(String prefix);
    List<Product> findByNameEndingWith(String suffix);
    List<Product> findByNameContaining(String text);

    // Comparison
    List<Product> findByPriceLessThan(BigDecimal price);
    List<Product> findByPriceLessThanEqual(BigDecimal price);
    List<Product> findByPriceGreaterThan(BigDecimal price);
    List<Product> findByPriceGreaterThanEqual(BigDecimal price);
    List<Product> findByPriceBetween(BigDecimal min, BigDecimal max);

    // Boolean
    List<Product> findByActiveTrue();
    List<Product> findByActiveFalse();

    // In clause
    List<Product> findByCategoryIn(Collection<String> categories);
    List<Product> findByCategoryNotIn(Collection<String> categories);

    // Ordering
    List<Product> findByCategoryOrderByNameAsc(String category);
    List<Product> findByCategoryOrderByPriceDesc(String category);

    // Limiting
    List<Product> findTop10ByCategory(String category);
    Optional<Product> findFirstByCategoryOrderByPriceAsc(String category);

    // Combined
    List<Product> findByNameAndCategory(String name, String category);
    List<Product> findByNameOrCategory(String name, String category);
}
```

## Pagination and Sorting

```java
public interface OrderRepository extends PagingAndSortingRepository<Order, Long> {

    Page<Order> findByStatus(OrderStatus status, Pageable pageable);

    Slice<Order> findByCustomerId(Long customerId, Pageable pageable);

    List<Order> findByStatus(OrderStatus status, Sort sort);
}

@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository repository;

    public Page<Order> findOrders(OrderStatus status, int page, int size) {
        Pageable pageable = PageRequest.of(page, size,
            Sort.by(Sort.Direction.DESC, "createdAt"));

        return repository.findByStatus(status, pageable);
    }

    public List<Order> findSorted(OrderStatus status) {
        Sort sort = Sort.by(
            Sort.Order.desc("createdAt"),
            Sort.Order.asc("orderNumber")
        );

        return repository.findByStatus(status, sort);
    }
}
```

## Custom Queries with @Query

```java
public interface OrderRepository extends CrudRepository<Order, Long> {

    @Query("SELECT * FROM orders WHERE customer_id = :customerId")
    List<Order> findByCustomer(Long customerId);

    @Query("""
        SELECT * FROM orders
        WHERE status = :status
          AND created_at BETWEEN :startDate AND :endDate
        ORDER BY created_at DESC
        """)
    List<Order> findByStatusAndDateRange(
        String status,
        LocalDateTime startDate,
        LocalDateTime endDate
    );

    @Query("""
        SELECT o.* FROM orders o
        JOIN order_items oi ON o.id = oi.order_id
        WHERE oi.product_id = :productId
        """)
    List<Order> findByProduct(Long productId);

    @Query("SELECT COUNT(*) FROM orders WHERE status = :status")
    long countByStatus(String status);

    @Query("""
        SELECT status, COUNT(*) as count
        FROM orders
        GROUP BY status
        """)
    List<StatusCount> countGroupByStatus();
}

public interface StatusCount {
    String getStatus();
    Long getCount();
}
```

## Modifying Queries

```java
public interface OrderRepository extends CrudRepository<Order, Long> {

    @Modifying
    @Query("UPDATE orders SET status = :status WHERE id = :id")
    boolean updateStatus(Long id, String status);

    @Modifying
    @Query("""
        UPDATE orders
        SET status = :newStatus
        WHERE status = :oldStatus
          AND created_at < :cutoffDate
        """)
    int bulkUpdateStatus(String oldStatus, String newStatus, LocalDateTime cutoffDate);

    @Modifying
    @Query("DELETE FROM orders WHERE status = :status AND created_at < :cutoffDate")
    int deleteOldOrders(String status, LocalDateTime cutoffDate);
}
```

## Named Queries

```java
// Define in META-INF/jdbc-named-queries.properties
// Order.findByCustomerAndStatus=SELECT * FROM orders WHERE customer_id = :customerId AND status = :status

public interface OrderRepository extends CrudRepository<Order, Long> {

    // Uses named query from properties file
    List<Order> findByCustomerAndStatus(Long customerId, String status);
}
```

## Custom Repository Implementation

```java
public interface OrderRepositoryCustom {
    List<Order> findByComplexCriteria(OrderSearchCriteria criteria);
    void bulkInsert(List<Order> orders);
}

public class OrderRepositoryImpl implements OrderRepositoryCustom {

    private final JdbcTemplate jdbcTemplate;
    private final NamedParameterJdbcTemplate namedJdbcTemplate;

    public OrderRepositoryImpl(JdbcTemplate jdbcTemplate,
                               NamedParameterJdbcTemplate namedJdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
        this.namedJdbcTemplate = namedJdbcTemplate;
    }

    @Override
    public List<Order> findByComplexCriteria(OrderSearchCriteria criteria) {
        StringBuilder sql = new StringBuilder("SELECT * FROM orders WHERE 1=1");
        MapSqlParameterSource params = new MapSqlParameterSource();

        if (criteria.getStatus() != null) {
            sql.append(" AND status = :status");
            params.addValue("status", criteria.getStatus().name());
        }

        if (criteria.getCustomerId() != null) {
            sql.append(" AND customer_id = :customerId");
            params.addValue("customerId", criteria.getCustomerId());
        }

        if (criteria.getMinAmount() != null) {
            sql.append(" AND total_amount >= :minAmount");
            params.addValue("minAmount", criteria.getMinAmount());
        }

        if (criteria.getStartDate() != null) {
            sql.append(" AND created_at >= :startDate");
            params.addValue("startDate", criteria.getStartDate());
        }

        sql.append(" ORDER BY created_at DESC");

        if (criteria.getLimit() != null) {
            sql.append(" LIMIT :limit");
            params.addValue("limit", criteria.getLimit());
        }

        return namedJdbcTemplate.query(sql.toString(), params, new OrderRowMapper());
    }

    @Override
    public void bulkInsert(List<Order> orders) {
        String sql = """
            INSERT INTO orders (order_number, customer_id, status, created_at)
            VALUES (?, ?, ?, ?)
            """;

        jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                Order order = orders.get(i);
                ps.setString(1, order.getOrderNumber());
                ps.setLong(2, order.getCustomerId());
                ps.setString(3, order.getStatus().name());
                ps.setTimestamp(4, Timestamp.valueOf(order.getCreatedAt()));
            }

            @Override
            public int getBatchSize() {
                return orders.size();
            }
        });
    }
}

public interface OrderRepository extends
        CrudRepository<Order, Long>,
        OrderRepositoryCustom {
    // Standard methods plus custom
}
```

## Query by Example

```java
@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ProductRepository repository;

    public List<Product> searchByExample(Product probe) {
        ExampleMatcher matcher = ExampleMatcher.matching()
            .withIgnoreCase()
            .withStringMatcher(ExampleMatcher.StringMatcher.CONTAINING)
            .withIgnoreNullValues()
            .withIgnorePaths("id", "createdAt");

        Example<Product> example = Example.of(probe, matcher);
        return repository.findAll(example);
    }
}
```

## Projections

```java
// Interface projection
public interface OrderSummary {
    String getOrderNumber();
    String getStatus();
    LocalDateTime getCreatedAt();
}

// Record projection
public record OrderDto(
    String orderNumber,
    String status,
    BigDecimal totalAmount
) {}

public interface OrderRepository extends CrudRepository<Order, Long> {

    // Interface projection
    List<OrderSummary> findSummaryByStatus(String status);

    // Via @Query for DTO
    @Query("""
        SELECT order_number, status, total_amount
        FROM orders
        WHERE customer_id = :customerId
        """)
    List<OrderDto> findDtoByCustomerId(Long customerId);
}
```

## Streaming Results

```java
public interface OrderRepository extends CrudRepository<Order, Long> {

    @Query("SELECT * FROM orders WHERE status = :status")
    Stream<Order> streamByStatus(String status);
}

@Service
@RequiredArgsConstructor
public class OrderExportService {

    private final OrderRepository repository;

    @Transactional(readOnly = true)
    public void exportOrders(String status, Consumer<Order> consumer) {
        try (Stream<Order> orders = repository.streamByStatus(status)) {
            orders.forEach(consumer);
        }
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use derived queries for simple cases | Overcomplicate derived queries |
| Use @Query for complex queries | String concatenation in custom impl |
| Implement custom repository for complex logic | Put SQL in service layer |
| Use projections for partial data | Fetch entire entities always |
| Use streaming for large results | Load all results into memory |
| Test queries thoroughly | Deploy untested queries |

## Production Checklist

- [ ] Derived queries validated
- [ ] Custom queries tested
- [ ] Pagination implemented
- [ ] Projections used appropriately
- [ ] Custom implementations tested
- [ ] Transaction boundaries correct
