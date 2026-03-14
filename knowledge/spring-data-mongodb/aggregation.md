# Spring Data MongoDB Aggregation Framework

> Source: https://docs.spring.io/spring-data/mongodb/reference/mongodb/aggregation-framework.html

## Overview

Spring Data MongoDB provides a fluent API for building MongoDB aggregation pipelines.

## Basic Aggregation

```java
Aggregation aggregation = Aggregation.newAggregation(
    Aggregation.match(Criteria.where("status").is("COMPLETED")),
    Aggregation.group("category")
        .count().as("count")
        .sum("amount").as("total"),
    Aggregation.sort(Sort.Direction.DESC, "total")
);

AggregationResults<CategoryStats> results = mongoTemplate.aggregate(
    aggregation,
    "orders",           // Collection name
    CategoryStats.class // Output type
);

List<CategoryStats> stats = results.getMappedResults();
```

## Pipeline Stages

### $match - Filter Documents

```java
// Simple match
Aggregation.match(Criteria.where("active").is(true))

// Multiple conditions
Aggregation.match(
    Criteria.where("status").is("COMPLETED")
        .and("amount").gte(100)
)

// Date range
Aggregation.match(
    Criteria.where("createdAt")
        .gte(startDate)
        .lt(endDate)
)
```

### $group - Group and Aggregate

```java
Aggregation.group("category")
    .count().as("count")
    .sum("amount").as("totalAmount")
    .avg("amount").as("avgAmount")
    .min("amount").as("minAmount")
    .max("amount").as("maxAmount")
    .first("name").as("firstName")
    .last("name").as("lastName")
    .addToSet("tags").as("allTags")
    .push("items").as("allItems")
```

### $project - Shape Output

```java
// Include/exclude fields
Aggregation.project()
    .andInclude("name", "price")
    .andExclude("_id")

// Rename fields
Aggregation.project()
    .and("category").as("productCategory")
    .and("price").as("productPrice")

// Computed fields
Aggregation.project()
    .and("price").multiply(1.1).as("priceWithTax")
    .and("quantity").multiply("price").as("lineTotal")

// Conditional
Aggregation.project()
    .and(ConditionalOperators
        .when(Criteria.where("price").gt(100))
        .then("expensive")
        .otherwise("cheap"))
    .as("priceCategory")
```

### $sort - Order Results

```java
Aggregation.sort(Sort.Direction.DESC, "totalAmount")

// Multiple fields
Aggregation.sort(Sort.by(
    Sort.Order.desc("category"),
    Sort.Order.asc("name")
))
```

### $limit and $skip

```java
Aggregation.skip(20L)
Aggregation.limit(10L)
```

### $unwind - Deconstruct Arrays

```java
// Simple unwind
Aggregation.unwind("items")

// Preserve null/empty arrays
Aggregation.unwind("items", true)

// With array index
Aggregation.unwind("items", "itemIndex", true)
```

### $lookup - Join Collections

```java
// Simple lookup
Aggregation.lookup("products", "productId", "_id", "product")

// With pipeline
LookupOperation lookup = LookupOperation.newLookup()
    .from("products")
    .localField("productId")
    .foreignField("_id")
    .pipeline(
        Aggregation.match(Criteria.where("active").is(true)),
        Aggregation.project("name", "price")
    )
    .as("product");
```

### $addFields - Add Computed Fields

```java
Aggregation.addFields()
    .addField("fullName")
        .withValue(StringOperators.Concat.valueOf("firstName").concat(" ").concatValueOf("lastName"))
    .addField("total")
        .withValue(ArithmeticOperators.Multiply.valueOf("quantity").multiplyBy("price"))
    .build()
```

## Complete Examples

### Sales by Category

```java
Aggregation aggregation = Aggregation.newAggregation(
    Aggregation.match(Criteria.where("status").is("COMPLETED")),
    Aggregation.group("category")
        .count().as("orderCount")
        .sum("total").as("revenue")
        .avg("total").as("avgOrderValue"),
    Aggregation.sort(Sort.Direction.DESC, "revenue"),
    Aggregation.limit(10)
);

List<CategorySales> results = mongoTemplate.aggregate(
    aggregation, "orders", CategorySales.class
).getMappedResults();
```

### Top Selling Products

```java
Aggregation aggregation = Aggregation.newAggregation(
    Aggregation.unwind("items"),
    Aggregation.group("items.productId")
        .sum("items.quantity").as("totalSold")
        .first("items.productName").as("productName"),
    Aggregation.sort(Sort.Direction.DESC, "totalSold"),
    Aggregation.limit(10)
);
```

### Orders with Customer Details

```java
Aggregation aggregation = Aggregation.newAggregation(
    Aggregation.lookup("customers", "customerId", "_id", "customer"),
    Aggregation.unwind("customer"),
    Aggregation.project()
        .andInclude("orderId", "total", "status", "createdAt")
        .and("customer.name").as("customerName")
        .and("customer.email").as("customerEmail")
);
```

### Date-based Grouping

```java
Aggregation aggregation = Aggregation.newAggregation(
    Aggregation.match(Criteria.where("createdAt")
        .gte(startOfYear).lt(endOfYear)),
    Aggregation.project()
        .and(DateOperators.Month.monthOf("createdAt")).as("month")
        .and("total").as("total"),
    Aggregation.group("month")
        .sum("total").as("monthlyRevenue")
        .count().as("orderCount"),
    Aggregation.sort(Sort.Direction.ASC, "_id")
);
```

### Faceted Search

```java
Aggregation aggregation = Aggregation.newAggregation(
    Aggregation.match(Criteria.where("name").regex(keyword, "i")),
    Aggregation.facet()
        .and(Aggregation.count().as("total")).as("metadata")
        .and(
            Aggregation.group("category").count().as("count"),
            Aggregation.sort(Sort.Direction.DESC, "count")
        ).as("categories")
        .and(
            Aggregation.group("brand").count().as("count")
        ).as("brands")
        .and(
            Aggregation.skip(skip),
            Aggregation.limit(limit)
        ).as("results")
);
```

### Bucket (Range Grouping)

```java
Aggregation aggregation = Aggregation.newAggregation(
    Aggregation.bucket("price")
        .withBoundaries(0, 50, 100, 200, 500, 1000)
        .withDefaultBucket("1000+")
        .andOutput(AccumulatorOperators.Sum.sumOf(1)).as("count")
        .andOutput(AccumulatorOperators.Avg.avgOf("price")).as("avgPrice")
);
```

## Aggregation Options

```java
AggregationOptions options = AggregationOptions.builder()
    .allowDiskUse(true)           // For large datasets
    .collation(Collation.of("en")) // Language-specific sorting
    .maxTime(Duration.ofSeconds(30))
    .build();

Aggregation aggregation = Aggregation.newAggregation(stages)
    .withOptions(options);
```

## TypedAggregation

```java
TypedAggregation<Order> aggregation = Aggregation.newAggregation(
    Order.class,
    Aggregation.match(Criteria.where("status").is("COMPLETED")),
    Aggregation.group("customerId").sum("total").as("totalSpent")
);

// Collection name derived from Order.class
AggregationResults<CustomerSpend> results =
    mongoTemplate.aggregate(aggregation, CustomerSpend.class);
```

## Best Practices

1. **Use `$match` early** to reduce documents processed
2. **Create indexes** on fields used in `$match` and `$sort`
3. **Enable `allowDiskUse`** for large aggregations
4. **Limit output** with `$limit` when possible
5. **Use projections** to reduce document size
6. **Test with `explain()`** to verify index usage
