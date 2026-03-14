# Spring Data JDBC - Queries

## Overview

Spring Data JDBC provides multiple ways to query data: derived queries, @Query annotations, and direct JDBC template usage.

## Derived Queries

```java
public interface ProductRepository extends CrudRepository<Product, Long> {

    // Simple conditions
    List<Product> findByName(String name);
    List<Product> findByCategory(String category);
    List<Product> findByActiveTrue();

    // Multiple conditions
    List<Product> findByNameAndCategory(String name, String category);
    List<Product> findByNameOrCategory(String name, String category);

    // Comparisons
    List<Product> findByPriceLessThan(BigDecimal price);
    List<Product> findByPriceGreaterThanEqual(BigDecimal price);
    List<Product> findByPriceBetween(BigDecimal min, BigDecimal max);

    // String matching
    List<Product> findByNameContaining(String text);
    List<Product> findByNameStartingWith(String prefix);
    List<Product> findByNameEndingWith(String suffix);
    List<Product> findByNameIgnoreCase(String name);

    // Ordering
    List<Product> findByCategoryOrderByNameAsc(String category);
    List<Product> findByCategoryOrderByPriceDesc(String category);

    // Limiting
    List<Product> findTop5ByCategoryOrderByPriceDesc(String category);
    Optional<Product> findFirstByCategoryOrderByPriceAsc(String category);

    // Counting
    long countByCategory(String category);
    long countByActiveTrue();

    // Existence
    boolean existsByName(String name);
    boolean existsByNameAndCategory(String name, String category);

    // Deletion
    void deleteByCategory(String category);
    long deleteByActivefalse();
}
```

## @Query Annotation

### Select Queries
```java
public interface OrderRepository extends CrudRepository<Order, Long> {

    @Query("SELECT * FROM orders WHERE customer_id = :customerId")
    List<Order> findByCustomer(Long customerId);

    @Query("""
        SELECT * FROM orders
        WHERE status = :status
        ORDER BY created_at DESC
        LIMIT :limit
        """)
    List<Order> findRecentByStatus(String status, int limit);

    @Query("""
        SELECT o.* FROM orders o
        JOIN order_items oi ON o.id = oi.order_id
        WHERE oi.product_id = :productId
        GROUP BY o.id
        """)
    List<Order> findOrdersContainingProduct(Long productId);

    @Query("""
        SELECT * FROM orders
        WHERE created_at >= :startDate
          AND created_at < :endDate
        ORDER BY created_at
        """)
    List<Order> findByDateRange(LocalDate startDate, LocalDate endDate);
}
```

### Aggregate Functions
```java
public interface OrderRepository extends CrudRepository<Order, Long> {

    @Query("SELECT COUNT(*) FROM orders WHERE status = :status")
    long countByStatus(String status);

    @Query("SELECT SUM(total_amount) FROM orders WHERE customer_id = :customerId")
    BigDecimal sumTotalByCustomer(Long customerId);

    @Query("SELECT AVG(total_amount) FROM orders WHERE status = 'COMPLETED'")
    BigDecimal averageCompletedOrderAmount();

    @Query("""
        SELECT MIN(created_at), MAX(created_at)
        FROM orders
        WHERE customer_id = :customerId
        """)
    Object[] getOrderDateRange(Long customerId);
}
```

### Group By Queries
```java
public interface OrderRepository extends CrudRepository<Order, Long> {

    @Query("""
        SELECT status, COUNT(*) as count
        FROM orders
        GROUP BY status
        """)
    List<StatusCount> countByStatusGrouped();

    @Query("""
        SELECT customer_id, SUM(total_amount) as total
        FROM orders
        WHERE created_at >= :since
        GROUP BY customer_id
        ORDER BY total DESC
        LIMIT :limit
        """)
    List<CustomerTotal> findTopCustomers(LocalDateTime since, int limit);

    @Query("""
        SELECT DATE_TRUNC('month', created_at) as month,
               COUNT(*) as order_count,
               SUM(total_amount) as total_amount
        FROM orders
        WHERE created_at >= :startDate
        GROUP BY DATE_TRUNC('month', created_at)
        ORDER BY month
        """)
    List<MonthlyStats> getMonthlyStats(LocalDate startDate);
}

public interface StatusCount {
    String getStatus();
    Long getCount();
}

public interface CustomerTotal {
    Long getCustomerId();
    BigDecimal getTotal();
}

public interface MonthlyStats {
    LocalDate getMonth();
    Long getOrderCount();
    BigDecimal getTotalAmount();
}
```

### Subqueries
```java
public interface ProductRepository extends CrudRepository<Product, Long> {

    @Query("""
        SELECT * FROM products
        WHERE id IN (
            SELECT product_id FROM order_items
            GROUP BY product_id
            HAVING COUNT(*) > :minOrders
        )
        """)
    List<Product> findPopularProducts(int minOrders);

    @Query("""
        SELECT * FROM products p
        WHERE price > (
            SELECT AVG(price) FROM products
            WHERE category = p.category
        )
        """)
    List<Product> findAboveAverageInCategory();

    @Query("""
        SELECT * FROM customers c
        WHERE EXISTS (
            SELECT 1 FROM orders o
            WHERE o.customer_id = c.id
              AND o.status = 'COMPLETED'
        )
        """)
    List<Customer> findCustomersWithCompletedOrders();
}
```

## Modifying Queries

```java
public interface ProductRepository extends CrudRepository<Product, Long> {

    @Modifying
    @Query("UPDATE products SET price = :price WHERE id = :id")
    boolean updatePrice(Long id, BigDecimal price);

    @Modifying
    @Query("""
        UPDATE products
        SET price = price * :multiplier
        WHERE category = :category
        """)
    int updatePriceByCategory(String category, BigDecimal multiplier);

    @Modifying
    @Query("UPDATE products SET active = false WHERE stock = 0")
    int deactivateOutOfStock();

    @Modifying
    @Query("DELETE FROM products WHERE category = :category AND active = false")
    int deleteInactiveByCategory(String category);

    @Modifying
    @Query("""
        INSERT INTO product_views (product_id, viewed_at)
        VALUES (:productId, :viewedAt)
        """)
    void recordView(Long productId, LocalDateTime viewedAt);
}
```

## JdbcTemplate Queries

```java
@Repository
@RequiredArgsConstructor
public class ProductQueryRepository {

    private final JdbcTemplate jdbcTemplate;
    private final NamedParameterJdbcTemplate namedJdbcTemplate;

    // Simple query
    public List<Product> findAll() {
        return jdbcTemplate.query(
            "SELECT * FROM products ORDER BY name",
            new BeanPropertyRowMapper<>(Product.class)
        );
    }

    // Query with parameters
    public Optional<Product> findById(Long id) {
        List<Product> results = jdbcTemplate.query(
            "SELECT * FROM products WHERE id = ?",
            new BeanPropertyRowMapper<>(Product.class),
            id
        );
        return results.isEmpty() ? Optional.empty() : Optional.of(results.get(0));
    }

    // Named parameters
    public List<Product> findByFilter(ProductFilter filter) {
        StringBuilder sql = new StringBuilder("SELECT * FROM products WHERE 1=1");
        MapSqlParameterSource params = new MapSqlParameterSource();

        if (filter.getName() != null) {
            sql.append(" AND name ILIKE :name");
            params.addValue("name", "%" + filter.getName() + "%");
        }

        if (filter.getCategory() != null) {
            sql.append(" AND category = :category");
            params.addValue("category", filter.getCategory());
        }

        if (filter.getMinPrice() != null) {
            sql.append(" AND price >= :minPrice");
            params.addValue("minPrice", filter.getMinPrice());
        }

        if (filter.getMaxPrice() != null) {
            sql.append(" AND price <= :maxPrice");
            params.addValue("maxPrice", filter.getMaxPrice());
        }

        sql.append(" ORDER BY name");

        return namedJdbcTemplate.query(sql.toString(), params,
            new BeanPropertyRowMapper<>(Product.class));
    }

    // Custom row mapper
    public List<ProductSummary> findSummaries() {
        return jdbcTemplate.query(
            """
            SELECT p.id, p.name, p.price, COUNT(oi.order_id) as order_count
            FROM products p
            LEFT JOIN order_items oi ON p.id = oi.product_id
            GROUP BY p.id, p.name, p.price
            ORDER BY order_count DESC
            """,
            (rs, rowNum) -> new ProductSummary(
                rs.getLong("id"),
                rs.getString("name"),
                rs.getBigDecimal("price"),
                rs.getInt("order_count")
            )
        );
    }
}
```

## Batch Operations

```java
@Repository
@RequiredArgsConstructor
public class BatchRepository {

    private final JdbcTemplate jdbcTemplate;
    private final NamedParameterJdbcTemplate namedJdbcTemplate;

    // Batch insert
    public void batchInsert(List<Product> products) {
        String sql = """
            INSERT INTO products (name, category, price, active)
            VALUES (?, ?, ?, ?)
            """;

        jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                Product p = products.get(i);
                ps.setString(1, p.getName());
                ps.setString(2, p.getCategory());
                ps.setBigDecimal(3, p.getPrice());
                ps.setBoolean(4, p.isActive());
            }

            @Override
            public int getBatchSize() {
                return products.size();
            }
        });
    }

    // Named parameter batch
    public void batchUpdate(List<Product> products) {
        String sql = """
            UPDATE products
            SET price = :price, active = :active
            WHERE id = :id
            """;

        SqlParameterSource[] params = products.stream()
            .map(p -> new MapSqlParameterSource()
                .addValue("id", p.getId())
                .addValue("price", p.getPrice())
                .addValue("active", p.isActive()))
            .toArray(SqlParameterSource[]::new);

        namedJdbcTemplate.batchUpdate(sql, params);
    }
}
```

## Pagination

```java
public interface OrderRepository extends PagingAndSortingRepository<Order, Long> {

    Page<Order> findByStatus(String status, Pageable pageable);

    @Query(value = """
        SELECT * FROM orders
        WHERE customer_id = :customerId
        ORDER BY created_at DESC
        LIMIT :#{#pageable.pageSize}
        OFFSET :#{#pageable.offset}
        """,
        countQuery = """
        SELECT COUNT(*) FROM orders
        WHERE customer_id = :customerId
        """)
    Page<Order> findByCustomerId(Long customerId, Pageable pageable);
}

@Service
public class OrderService {

    @Autowired
    private OrderRepository repository;

    public Page<Order> getOrders(String status, int page, int size) {
        Pageable pageable = PageRequest.of(page, size,
            Sort.by(Sort.Direction.DESC, "createdAt"));
        return repository.findByStatus(status, pageable);
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use derived queries for simple cases | Complex derived query names |
| Use @Query for complex SQL | Build SQL strings dynamically when not needed |
| Use named parameters | Positional parameters in complex queries |
| Create indexes for query patterns | Query unindexed columns |
| Use pagination for large results | Fetch all records into memory |
| Use projections when possible | Select all columns when not needed |

## Production Checklist

- [ ] Queries tested with realistic data
- [ ] Indexes created for query patterns
- [ ] Pagination implemented for lists
- [ ] Projections used to limit data
- [ ] Query execution plans analyzed
- [ ] Batch operations for bulk updates
