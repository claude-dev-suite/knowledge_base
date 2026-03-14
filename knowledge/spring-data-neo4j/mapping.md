# Spring Data Neo4j - Object-Graph Mapping

## Overview

Spring Data Neo4j provides object-graph mapping (OGM) for mapping Java objects to Neo4j nodes and relationships, handling the complexities of graph data structures.

## Node Mapping

### Basic Node
```java
@Node("Person")  // Label name
public class Person {

    @Id
    @GeneratedValue
    private Long id;

    @Property("name")  // Property name in Neo4j
    private String fullName;

    @Property  // Uses field name "email"
    private String email;

    private Integer birthYear;  // Implicit @Property

    // Getters and setters
}
```

### Multiple Labels
```java
@Node({"Person", "Employee"})
public class Employee {

    @Id
    @GeneratedValue
    private Long id;

    private String name;
    private String department;
}

// Dynamic labels
@Node
public class DynamicLabelEntity {

    @Id
    @GeneratedValue
    private Long id;

    @DynamicLabels
    private List<String> labels = new ArrayList<>();
}
```

### ID Strategies

```java
// Auto-generated (default)
@Node
public class AutoIdEntity {
    @Id
    @GeneratedValue
    private Long id;
}

// Application-provided
@Node
public class CustomIdEntity {
    @Id
    private String customId;
}

// UUID
@Node
public class UuidEntity {
    @Id
    @GeneratedValue(generatorClass = UUIDStringGenerator.class)
    private String id;
}

// Custom generator
@Node
public class CustomGeneratedEntity {
    @Id
    @GeneratedValue(generatorClass = MyIdGenerator.class)
    private String id;
}

public class MyIdGenerator implements IdGenerator<String> {
    @Override
    public String generateId(String primaryLabel, Object entity) {
        return primaryLabel + "-" + UUID.randomUUID().toString();
    }
}
```

## Property Mapping

### Property Types
```java
@Node("Product")
public class Product {

    @Id
    @GeneratedValue
    private Long id;

    // String
    private String name;

    // Numbers
    private Integer quantity;
    private Long viewCount;
    private Double price;
    private BigDecimal exactPrice;

    // Boolean
    private Boolean active;

    // Date/Time
    private LocalDate releaseDate;
    private LocalDateTime createdAt;
    private ZonedDateTime publishedAt;
    private Duration validityPeriod;

    // Arrays
    private String[] tags;
    private List<String> categories;

    // Enums (stored as string by default)
    private ProductStatus status;

    // Spatial
    @Property
    private Point location;
}

public enum ProductStatus {
    DRAFT, PUBLISHED, ARCHIVED
}
```

### Custom Converters
```java
@Configuration
public class Neo4jConverterConfig {

    @Bean
    public Neo4jConversions neo4jConversions() {
        return new Neo4jConversions(List.of(
            new MoneyToStringConverter(),
            new StringToMoneyConverter()
        ));
    }
}

@WritingConverter
public class MoneyToStringConverter implements Converter<Money, Value> {
    @Override
    public Value convert(Money source) {
        return Values.value(source.getCurrency() + ":" + source.getAmount());
    }
}

@ReadingConverter
public class StringToMoneyConverter implements Converter<Value, Money> {
    @Override
    public Money convert(Value source) {
        String[] parts = source.asString().split(":");
        return new Money(parts[0], new BigDecimal(parts[1]));
    }
}
```

### Computed Properties
```java
@Node
public class Order {

    @Id
    @GeneratedValue
    private Long id;

    private List<OrderItem> items;

    // Not persisted, computed on read
    @Transient
    private BigDecimal total;

    @PostLoad
    public void computeTotal() {
        this.total = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

## Relationship Mapping

### Simple Relationships
```java
@Node
public class Person {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    // Outgoing relationship
    @Relationship(type = "KNOWS", direction = Direction.OUTGOING)
    private List<Person> friends;

    // Incoming relationship
    @Relationship(type = "WORKS_AT", direction = Direction.INCOMING)
    private Company employer;

    // Undirected (both directions)
    @Relationship(type = "RELATED_TO")
    private List<Person> relatives;
}
```

### Relationship Properties
```java
@Node
public class Person {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @Relationship(type = "ACTED_IN")
    private List<ActedIn> roles;
}

@RelationshipProperties
public class ActedIn {

    @Id
    @GeneratedValue
    private Long id;

    @TargetNode
    private Movie movie;

    private String role;
    private Integer billing;
}

@Node
public class Movie {

    @Id
    @GeneratedValue
    private Long id;

    private String title;

    @Relationship(type = "ACTED_IN", direction = Direction.INCOMING)
    private List<ActedIn> cast;
}
```

### Self-referencing Relationships
```java
@Node
public class Category {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @Relationship(type = "PARENT_OF", direction = Direction.OUTGOING)
    private List<Category> children;

    @Relationship(type = "PARENT_OF", direction = Direction.INCOMING)
    private Category parent;
}
```

### Complex Graph Structures
```java
@Node
public class SocialUser {

    @Id
    @GeneratedValue
    private Long id;

    private String username;

    @Relationship(type = "FOLLOWS", direction = Direction.OUTGOING)
    private Set<SocialUser> following;

    @Relationship(type = "FOLLOWS", direction = Direction.INCOMING)
    private Set<SocialUser> followers;

    @Relationship(type = "POSTED")
    private List<Post> posts;

    @Relationship(type = "LIKED")
    private Set<Post> likedPosts;
}

@Node
public class Post {

    @Id
    @GeneratedValue
    private Long id;

    private String content;
    private LocalDateTime createdAt;

    @Relationship(type = "POSTED", direction = Direction.INCOMING)
    private SocialUser author;

    @Relationship(type = "REPLY_TO")
    private Post replyTo;

    @Relationship(type = "REPLY_TO", direction = Direction.INCOMING)
    private List<Post> replies;
}
```

## Inheritance

### Single Table Strategy
```java
@Node
public abstract class Vehicle {

    @Id
    @GeneratedValue
    private Long id;

    private String brand;
    private String model;
}

@Node("Car")
public class Car extends Vehicle {
    private Integer numberOfDoors;
    private String fuelType;
}

@Node("Motorcycle")
public class Motorcycle extends Vehicle {
    private Integer engineCC;
    private Boolean hasSidecar;
}
```

### Discriminator Pattern
```java
@Node
public class Content {

    @Id
    @GeneratedValue
    private Long id;

    private String title;
    private String type;  // Discriminator

    private String body;      // For articles
    private String videoUrl;  // For videos
    private String imageUrl;  // For images
}
```

## Callbacks and Lifecycle

```java
@Node
public class AuditedEntity {

    @Id
    @GeneratedValue
    private Long id;

    private String data;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    @PrePersist
    public void prePersist() {
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }

    @PostLoad
    public void postLoad() {
        // Initialize transient fields
    }

    @PreUpdate
    public void preUpdate() {
        this.updatedAt = LocalDateTime.now();
    }
}
```

## Composite IDs

```java
@Node
public class CompositeEntity {

    @Id
    private CompositeId id;

    private String data;
}

public class CompositeId implements Serializable {
    private String region;
    private String code;

    // equals, hashCode, constructors
}

// Or use a converter
@Node
public class CompositeEntityV2 {

    @Id
    @Convert(converter = CompositeIdConverter.class)
    private CompositeId id;
}

public class CompositeIdConverter implements Neo4jPersistentPropertyConverter<CompositeId> {
    @Override
    public Value write(CompositeId source) {
        return Values.value(source.getRegion() + ":" + source.getCode());
    }

    @Override
    public CompositeId read(Value source) {
        String[] parts = source.asString().split(":");
        return new CompositeId(parts[0], parts[1]);
    }
}
```

## Lazy Loading

```java
@Node
public class Person {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    // Eager by default
    @Relationship(type = "FRIENDS_WITH")
    private List<Person> friends;

    // Force lazy loading
    @Relationship(type = "KNOWS")
    private List<Person> acquaintances;
}

// Control with queries
public interface PersonRepository extends Neo4jRepository<Person, Long> {

    // Fetch only person
    @Query("MATCH (p:Person) WHERE p.name = $name RETURN p")
    Person findByNameOnly(String name);

    // Fetch with friends (1 level)
    @Query("""
        MATCH (p:Person)
        WHERE p.name = $name
        OPTIONAL MATCH (p)-[r:FRIENDS_WITH]->(f:Person)
        RETURN p, collect(r), collect(f)
        """)
    Person findByNameWithFriends(String name);
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use @Node explicitly | Rely on implicit mapping |
| Define relationship direction | Ignore relationship semantics |
| Use relationship properties for data | Store data in nodes only |
| Implement lifecycle callbacks | Repeat audit logic |
| Use converters for complex types | Store serialized blobs |
| Plan inheritance carefully | Deep inheritance hierarchies |

## Production Checklist

- [ ] All nodes properly annotated
- [ ] IDs strategy chosen appropriately
- [ ] Relationships correctly directed
- [ ] Relationship properties used where needed
- [ ] Converters implemented for custom types
- [ ] Lifecycle callbacks in place
- [ ] Lazy loading strategy defined
