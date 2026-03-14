# Spring Data MongoDB Repositories

> Source: https://docs.spring.io/spring-data/mongodb/reference/mongodb/repositories/repositories.html

## Overview

Spring Data MongoDB provides repository support through `MongoRepository`, enabling CRUD operations and query derivation.

## Basic Repository

```java
public interface ProductRepository extends MongoRepository<Product, String> {

    // Derived queries
    List<Product> findByCategory(String category);
    List<Product> findByPriceLessThan(BigDecimal price);
    Optional<Product> findByName(String name);

    // Multiple conditions
    List<Product> findByCategoryAndPriceBetween(
        String category, BigDecimal min, BigDecimal max);

    // Sorting
    List<Product> findByCategoryOrderByPriceDesc(String category);

    // Limiting
    List<Product> findTop10ByCategoryOrderByCreatedAtDesc(String category);

    // Exists and count
    boolean existsByName(String name);
    long countByCategory(String category);
}
```

## Entity Mapping

```java
@Document(collection = "products")
public class Product {

    @Id
    private String id;

    @Field("product_name")
    private String name;

    @Indexed
    private String category;

    private BigDecimal price;

    private List<String> tags;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}
```

## Query Methods

### Comparison

| Keyword | Sample | MongoDB Query |
|---------|--------|---------------|
| `Is`, `Equals` | `findByName(name)` | `{name: name}` |
| `Not` | `findByNameNot(name)` | `{name: {$ne: name}}` |
| `LessThan` | `findByPriceLessThan(price)` | `{price: {$lt: price}}` |
| `LessThanEqual` | `findByPriceLessThanEqual(price)` | `{price: {$lte: price}}` |
| `GreaterThan` | `findByPriceGreaterThan(price)` | `{price: {$gt: price}}` |
| `Between` | `findByPriceBetween(min, max)` | `{price: {$gte: min, $lte: max}}` |

### String Matching

| Keyword | Sample | MongoDB Query |
|---------|--------|---------------|
| `Like` | `findByNameLike(pattern)` | `{name: {$regex: pattern}}` |
| `StartingWith` | `findByNameStartingWith(prefix)` | `{name: {$regex: ^prefix}}` |
| `EndingWith` | `findByNameEndingWith(suffix)` | `{name: {$regex: suffix$}}` |
| `Containing` | `findByNameContaining(part)` | `{name: {$regex: part}}` |
| `Regex` | `findByNameRegex(regex)` | `{name: {$regex: regex}}` |

### Collection Operations

| Keyword | Sample | MongoDB Query |
|---------|--------|---------------|
| `In` | `findByCategoryIn(list)` | `{category: {$in: list}}` |
| `NotIn` | `findByCategoryNotIn(list)` | `{category: {$nin: list}}` |

### Logical Operators

```java
// AND (implicit)
List<Product> findByCategoryAndActive(String category, boolean active);

// OR
List<Product> findByCategoryOrBrand(String category, String brand);
```

### Null Handling

```java
List<Product> findByDescriptionIsNull();
List<Product> findByDescriptionIsNotNull();
List<Product> findByTagsExists(boolean exists);
```

## @Query Annotation

### Basic Query

```java
@Query("{ 'category': ?0, 'price': { $lte: ?1 } }")
List<Product> findByCategoryWithMaxPrice(String category, BigDecimal maxPrice);
```

### Named Parameters

```java
@Query("{ 'category': :#{#category}, 'active': :#{#active} }")
List<Product> findProducts(
    @Param("category") String category,
    @Param("active") boolean active);
```

### Projection (Select Fields)

```java
@Query(value = "{ 'category': ?0 }", fields = "{ 'name': 1, 'price': 1, '_id': 0 }")
List<ProductSummary> findSummaryByCategory(String category);
```

### Regex Search

```java
@Query("{ 'name': { $regex: ?0, $options: 'i' } }")
List<Product> searchByName(String keyword);
```

### Array Queries

```java
// Match any tag
@Query("{ 'tags': { $in: ?0 } }")
List<Product> findByAnyTag(List<String> tags);

// Match all tags
@Query("{ 'tags': { $all: ?0 } }")
List<Product> findByAllTags(List<String> tags);

// Match array size
@Query("{ 'tags': { $size: ?0 } }")
List<Product> findByTagCount(int count);
```

### Nested Documents

```java
@Query("{ 'address.city': ?0 }")
List<User> findByCity(String city);

@Query("{ 'orders.total': { $gte: ?0 } }")
List<User> findByMinOrderTotal(BigDecimal minTotal);
```

## Aggregation in Repository

```java
public interface OrderRepository extends MongoRepository<Order, String> {

    @Aggregation(pipeline = {
        "{ $match: { status: 'COMPLETED' } }",
        "{ $group: { _id: '$customerId', total: { $sum: '$amount' } } }",
        "{ $sort: { total: -1 } }",
        "{ $limit: 10 }"
    })
    List<CustomerTotal> findTopCustomers();

    @Aggregation(pipeline = {
        "{ $unwind: '$items' }",
        "{ $group: { _id: '$items.productId', count: { $sum: 1 } } }",
        "{ $sort: { count: -1 } }",
        "{ $limit: ?0 }"
    })
    List<ProductCount> findTopSellingProducts(int limit);

    @Aggregation(pipeline = {
        "{ $match: { customerId: ?0 } }",
        "{ $group: { _id: null, total: { $sum: '$amount' } } }"
    })
    BigDecimal getTotalAmountByCustomer(String customerId);
}
```

## Pagination and Sorting

```java
// Pagination
Page<Product> findByCategory(String category, Pageable pageable);

// Usage
Page<Product> page = repository.findByCategory(
    "electronics",
    PageRequest.of(0, 10, Sort.by("price").descending())
);

// Slice (more efficient for large datasets)
Slice<Product> findSliceByCategory(String category, Pageable pageable);
```

## Projections

### Interface Projection

```java
public interface ProductSummary {
    String getName();
    BigDecimal getPrice();
}

List<ProductSummary> findByCategory(String category, Class<ProductSummary> type);
```

### DTO Projection

```java
public record ProductDTO(String name, BigDecimal price) {}

@Query(value = "{ 'category': ?0 }", fields = "{ 'name': 1, 'price': 1 }")
List<ProductDTO> findDTOByCategory(String category);
```

## Delete Operations

```java
void deleteByCategory(String category);

long deleteByActiveIsFalse();

@Query(delete = true, value = "{ 'createdAt': { $lt: ?0 } }")
long deleteOlderThan(LocalDateTime date);
```

## Update Operations

```java
@Query("{ '_id': ?0 }")
@Update("{ $set: { 'price': ?1 } }")
void updatePrice(String id, BigDecimal price);

@Query("{ 'category': ?0 }")
@Update("{ $inc: { 'viewCount': 1 } }")
void incrementViewCount(String category);
```

## Best Practices

1. **Use derived queries for simple cases**
2. **Use `@Query` for complex MongoDB queries**
3. **Use projections to limit returned fields**
4. **Use pagination for large result sets**
5. **Create indexes for query fields**
6. **Prefer `Slice` over `Page` for better performance**
