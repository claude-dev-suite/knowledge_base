# Spring Data MongoDB Template API

> Source: https://docs.spring.io/spring-data/mongodb/reference/mongodb/template-api.html

## Overview

`MongoTemplate` provides low-level access to MongoDB operations with full control over queries, updates, and aggregations.

## Configuration

```java
@Configuration
public class MongoConfig {

    @Bean
    public MongoTemplate mongoTemplate(MongoDatabaseFactory factory) {
        return new MongoTemplate(factory);
    }
}
```

Or with Spring Boot auto-configuration:

```java
@Service
@RequiredArgsConstructor
public class ProductService {

    private final MongoTemplate mongoTemplate;
}
```

## CRUD Operations

### Create

```java
// Insert single document
Product product = new Product("Phone", "electronics", BigDecimal.valueOf(999));
Product saved = mongoTemplate.insert(product);

// Insert multiple documents
List<Product> products = Arrays.asList(product1, product2, product3);
Collection<Product> inserted = mongoTemplate.insertAll(products);

// Save (insert or update)
Product saved = mongoTemplate.save(product);  // Upsert based on _id
```

### Read

```java
// Find by ID
Product product = mongoTemplate.findById(id, Product.class);

// Find one
Query query = new Query(Criteria.where("name").is("Phone"));
Product product = mongoTemplate.findOne(query, Product.class);

// Find all matching
List<Product> products = mongoTemplate.find(query, Product.class);

// Find all
List<Product> all = mongoTemplate.findAll(Product.class);

// Count
long count = mongoTemplate.count(query, Product.class);

// Exists
boolean exists = mongoTemplate.exists(query, Product.class);
```

### Update

```java
// Update first matching
Query query = new Query(Criteria.where("id").is(id));
Update update = new Update().set("price", BigDecimal.valueOf(899));
UpdateResult result = mongoTemplate.updateFirst(query, update, Product.class);

// Update all matching
UpdateResult result = mongoTemplate.updateMulti(query, update, Product.class);

// Upsert (update or insert)
UpdateResult result = mongoTemplate.upsert(query, update, Product.class);

// Find and modify
Product updated = mongoTemplate.findAndModify(
    query,
    update,
    FindAndModifyOptions.options().returnNew(true),
    Product.class
);
```

### Delete

```java
// Remove matching documents
Query query = new Query(Criteria.where("id").is(id));
DeleteResult result = mongoTemplate.remove(query, Product.class);

// Remove and return
Product removed = mongoTemplate.findAndRemove(query, Product.class);

// Remove all from collection
mongoTemplate.remove(new Query(), Product.class);

// Drop collection
mongoTemplate.dropCollection(Product.class);
```

## Query Building

### Criteria

```java
// Simple criteria
Criteria.where("name").is("Phone")
Criteria.where("price").gt(100)
Criteria.where("active").is(true)

// Comparison
Criteria.where("price").gte(100).lte(500)
Criteria.where("stock").ne(0)

// String matching
Criteria.where("name").regex("^Phone", "i")  // Case-insensitive
Criteria.where("description").regex(".*wireless.*")

// Null checks
Criteria.where("description").is(null)
Criteria.where("description").exists(true)

// Array operations
Criteria.where("tags").in("electronics", "mobile")
Criteria.where("tags").all("new", "featured")
Criteria.where("tags").size(3)
```

### Combining Criteria

```java
// AND
Query query = new Query(
    Criteria.where("category").is("electronics")
        .and("price").lte(1000)
        .and("active").is(true)
);

// OR
Query query = new Query(
    new Criteria().orOperator(
        Criteria.where("category").is("electronics"),
        Criteria.where("category").is("computers")
    )
);

// Complex combinations
Query query = new Query(
    Criteria.where("active").is(true)
        .andOperator(
            new Criteria().orOperator(
                Criteria.where("price").lt(100),
                Criteria.where("featured").is(true)
            )
        )
);
```

### Sorting and Pagination

```java
Query query = new Query()
    .addCriteria(Criteria.where("category").is("electronics"))
    .with(Sort.by(Sort.Direction.DESC, "price"))
    .skip(20)
    .limit(10);

// With PageRequest
Query query = new Query()
    .with(PageRequest.of(2, 10, Sort.by("createdAt").descending()));
```

### Field Selection (Projection)

```java
Query query = new Query();
query.fields()
    .include("name")
    .include("price")
    .exclude("_id");

// Slice arrays
query.fields().slice("comments", 10);  // First 10 comments
```

## Update Operations

### Basic Updates

```java
Update update = new Update()
    .set("name", "New Name")
    .set("updatedAt", LocalDateTime.now());
```

### Field Operations

```java
Update update = new Update()
    .set("price", 999)           // Set field
    .unset("deprecatedField")    // Remove field
    .rename("oldName", "newName") // Rename field
    .inc("viewCount", 1)         // Increment
    .multiply("price", 1.1)      // Multiply
    .min("minPrice", 50)         // Set if less than current
    .max("maxPrice", 100)        // Set if greater than current
    .currentDate("lastModified"); // Set to current date
```

### Array Operations

```java
Update update = new Update()
    .push("tags", "new-tag")              // Add to array
    .pushAll("tags", Arrays.asList("a", "b"))
    .addToSet("tags", "unique-tag")       // Add if not exists
    .pull("tags", "old-tag")              // Remove from array
    .pop("comments", Update.Position.LAST) // Remove last element
    .set("tags.0", "first-tag");          // Update by index
```

### Conditional Updates

```java
// Update with condition
Update update = new Update()
    .set("status", "approved")
    .filterArray("elem", Criteria.where("elem.score").gt(80));
```

## Bulk Operations

```java
BulkOperations bulkOps = mongoTemplate.bulkOps(
    BulkOperations.BulkMode.ORDERED,
    Product.class
);

// Add operations
bulkOps.insert(product1);
bulkOps.insert(product2);
bulkOps.updateOne(query1, update1);
bulkOps.updateMulti(query2, update2);
bulkOps.remove(query3);

// Execute
BulkWriteResult result = bulkOps.execute();
```

## Distinct and Group

```java
// Distinct values
List<String> categories = mongoTemplate.findDistinct(
    new Query(),
    "category",
    Product.class,
    String.class
);

// With filter
List<String> activeCategories = mongoTemplate.findDistinct(
    Query.query(Criteria.where("active").is(true)),
    "category",
    Product.class,
    String.class
);
```

## Text Search

```java
// Create text index first
mongoTemplate.indexOps(Product.class).ensureIndex(
    new TextIndexDefinition.TextIndexDefinitionBuilder()
        .onField("name")
        .onField("description")
        .build()
);

// Search
Query query = TextQuery
    .queryText(new TextCriteria().matchingAny("phone", "mobile"))
    .sortByScore();

List<Product> results = mongoTemplate.find(query, Product.class);
```

## GeoSpatial Queries

```java
// Near point
Query query = new Query(
    Criteria.where("location").near(new Point(-73.99, 40.73))
);

// Within circle
Query query = new Query(
    Criteria.where("location").withinSphere(
        new Circle(new Point(-73.99, 40.73), 0.01)
    )
);

// Within box
Query query = new Query(
    Criteria.where("location").within(
        new Box(new Point(-74.0, 40.7), new Point(-73.9, 40.8))
    )
);
```

## Best Practices

1. **Use repositories for simple CRUD**
2. **Use MongoTemplate for complex queries**
3. **Create appropriate indexes**
4. **Use projections to limit data transfer**
5. **Use bulk operations for batch updates**
6. **Handle update results for error checking**
