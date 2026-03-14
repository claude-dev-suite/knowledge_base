# Spring Data Neo4j - Relationships

## Overview

Relationships are first-class citizens in Neo4j, connecting nodes and optionally carrying properties. Spring Data Neo4j provides rich support for modeling and querying graph relationships.

## Basic Relationships

### Simple Relationship
```java
@Node
public class Person {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @Relationship(type = "KNOWS")
    private List<Person> friends;
}
```

### Relationship Direction
```java
@Node
public class Person {

    @Id
    @GeneratedValue
    private Long id;

    // Person -[:FOLLOWS]-> Person
    @Relationship(type = "FOLLOWS", direction = Direction.OUTGOING)
    private List<Person> following;

    // Person <-[:FOLLOWS]- Person
    @Relationship(type = "FOLLOWS", direction = Direction.INCOMING)
    private List<Person> followers;

    // Both directions (undirected in query)
    @Relationship(type = "RELATED_TO")
    private List<Person> relatives;
}
```

### Single vs Collection
```java
@Node
public class Movie {

    @Id
    @GeneratedValue
    private Long id;

    // Single relationship (to-one)
    @Relationship(type = "DIRECTED", direction = Direction.INCOMING)
    private Person director;

    // Collection relationship (to-many)
    @Relationship(type = "ACTED_IN", direction = Direction.INCOMING)
    private List<Person> actors;

    // Set for unique relationships
    @Relationship(type = "PRODUCED", direction = Direction.INCOMING)
    private Set<Person> producers;
}
```

## Relationship Properties

### Defining Relationship Entity
```java
@RelationshipProperties
public class ActedIn {

    @Id
    @GeneratedValue
    private Long id;

    @TargetNode
    private Movie movie;

    private String role;
    private Integer billing;
    private Integer earnings;
}

@Node
public class Person {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @Relationship(type = "ACTED_IN")
    private List<ActedIn> roles;
}
```

### Bidirectional with Properties
```java
@RelationshipProperties
public class Friendship {

    @Id
    @GeneratedValue
    private Long id;

    @TargetNode
    private Person friend;

    private LocalDate since;
    private String howMet;
}

@Node
public class Person {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @Relationship(type = "FRIENDS_WITH", direction = Direction.OUTGOING)
    private List<Friendship> friendships;
}
```

### Rating Relationship
```java
@RelationshipProperties
public class Rating {

    @Id
    @GeneratedValue
    private Long id;

    @TargetNode
    private Movie movie;

    private Integer stars;
    private String review;
    private LocalDateTime ratedAt;
}

@Node
public class User {

    @Id
    @GeneratedValue
    private Long id;

    private String username;

    @Relationship(type = "RATED")
    private List<Rating> ratings;

    public void rateMovie(Movie movie, int stars, String review) {
        Rating rating = new Rating();
        rating.setMovie(movie);
        rating.setStars(stars);
        rating.setReview(review);
        rating.setRatedAt(LocalDateTime.now());
        this.ratings.add(rating);
    }
}
```

## Complex Relationship Patterns

### Self-Referencing Hierarchy
```java
@Node
public class Category {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @Relationship(type = "PARENT_OF", direction = Direction.OUTGOING)
    private List<Category> children = new ArrayList<>();

    @Relationship(type = "PARENT_OF", direction = Direction.INCOMING)
    private Category parent;

    public void addChild(Category child) {
        this.children.add(child);
        child.setParent(this);
    }
}
```

### Social Network Model
```java
@Node
public class SocialUser {

    @Id
    @GeneratedValue
    private Long id;

    private String username;

    @Relationship(type = "FOLLOWS", direction = Direction.OUTGOING)
    private Set<SocialUser> following = new HashSet<>();

    @Relationship(type = "FOLLOWS", direction = Direction.INCOMING)
    private Set<SocialUser> followers = new HashSet<>();

    @Relationship(type = "BLOCKED", direction = Direction.OUTGOING)
    private Set<SocialUser> blocked = new HashSet<>();

    @Relationship(type = "POSTED")
    private List<Post> posts = new ArrayList<>();

    @Relationship(type = "LIKED")
    private Set<Post> likedPosts = new HashSet<>();

    public void follow(SocialUser user) {
        this.following.add(user);
        user.getFollowers().add(this);
    }

    public void unfollow(SocialUser user) {
        this.following.remove(user);
        user.getFollowers().remove(this);
    }
}
```

### E-Commerce Order Model
```java
@RelationshipProperties
public class OrderItem {

    @Id
    @GeneratedValue
    private Long id;

    @TargetNode
    private Product product;

    private Integer quantity;
    private BigDecimal unitPrice;

    public BigDecimal getSubtotal() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }
}

@Node
public class Order {

    @Id
    @GeneratedValue
    private Long id;

    private String orderNumber;
    private LocalDateTime createdAt;
    private OrderStatus status;

    @Relationship(type = "PLACED_BY", direction = Direction.OUTGOING)
    private Customer customer;

    @Relationship(type = "CONTAINS")
    private List<OrderItem> items = new ArrayList<>();

    @Relationship(type = "SHIPPED_TO")
    private Address shippingAddress;

    public BigDecimal getTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

## Relationship Queries

### Repository Methods
```java
public interface PersonRepository extends Neo4jRepository<Person, Long> {

    @Query("""
        MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
        WHERE p.name = $name
        RETURN p, collect(m) as movies
        """)
    Person findWithMovies(String name);

    @Query("""
        MATCH (p1:Person {name: $name})-[:KNOWS*1..3]-(p2:Person)
        WHERE p1 <> p2
        RETURN DISTINCT p2
        """)
    List<Person> findFriendsOfFriends(String name);

    @Query("""
        MATCH (p1:Person)-[:ACTED_IN]->(m:Movie)<-[:ACTED_IN]-(p2:Person)
        WHERE p1.name = $name AND p1 <> p2
        RETURN DISTINCT p2, count(m) as sharedMovies
        ORDER BY sharedMovies DESC
        """)
    List<CoActorCount> findCoActors(String name);
}
```

### Fetching Relationships
```java
public interface MovieRepository extends Neo4jRepository<Movie, Long> {

    // Fetch movie with all actors and director
    @Query("""
        MATCH (m:Movie {title: $title})
        OPTIONAL MATCH (m)<-[acted:ACTED_IN]-(actor:Person)
        OPTIONAL MATCH (m)<-[directed:DIRECTED]-(director:Person)
        RETURN m, collect(acted), collect(actor), directed, director
        """)
    Movie findByTitleWithCast(String title);

    // Fetch with relationship properties
    @Query("""
        MATCH (m:Movie {title: $title})<-[r:ACTED_IN]-(p:Person)
        RETURN m, collect(r), collect(p)
        """)
    Movie findByTitleWithRoles(String title);
}
```

## Managing Relationships

### Creating Relationships
```java
@Service
@Transactional
public class RelationshipService {

    @Autowired
    private Neo4jTemplate template;

    @Autowired
    private Neo4jClient client;

    // Via entity
    public Person addFriend(Long personId, Long friendId) {
        Person person = template.findById(personId, Person.class).orElseThrow();
        Person friend = template.findById(friendId, Person.class).orElseThrow();

        person.getFriends().add(friend);
        return template.save(person);
    }

    // Via Cypher
    public void createActedIn(String personName, String movieTitle, String role) {
        client.query("""
            MATCH (p:Person {name: $personName}), (m:Movie {title: $movieTitle})
            CREATE (p)-[:ACTED_IN {role: $role}]->(m)
            """)
            .bind(personName).to("personName")
            .bind(movieTitle).to("movieTitle")
            .bind(role).to("role")
            .run();
    }

    // MERGE for idempotent creation
    public void ensureRelationship(String person1, String person2) {
        client.query("""
            MATCH (p1:Person {name: $name1}), (p2:Person {name: $name2})
            MERGE (p1)-[:KNOWS]->(p2)
            """)
            .bind(person1).to("name1")
            .bind(person2).to("name2")
            .run();
    }
}
```

### Updating Relationship Properties
```java
public void updateRole(String personName, String movieTitle, String newRole) {
    client.query("""
        MATCH (p:Person {name: $personName})-[r:ACTED_IN]->(m:Movie {title: $movieTitle})
        SET r.role = $newRole, r.updatedAt = datetime()
        """)
        .bind(personName).to("personName")
        .bind(movieTitle).to("movieTitle")
        .bind(newRole).to("newRole")
        .run();
}
```

### Deleting Relationships
```java
public void removeFriendship(String person1, String person2) {
    client.query("""
        MATCH (p1:Person {name: $name1})-[r:KNOWS]-(p2:Person {name: $name2})
        DELETE r
        """)
        .bind(person1).to("name1")
        .bind(person2).to("name2")
        .run();
}

public void removeAllRelationships(String personName) {
    client.query("""
        MATCH (p:Person {name: $name})-[r]-()
        DELETE r
        """)
        .bind(personName).to("name")
        .run();
}
```

## Path Analysis

```java
public interface GraphAnalysisRepository {

    @Query("""
        MATCH path = shortestPath(
            (p1:Person {name: $name1})-[*]-(p2:Person {name: $name2})
        )
        RETURN length(path) as distance
        """)
    Integer findDegreeOfSeparation(String name1, String name2);

    @Query("""
        MATCH (p:Person {name: $name})
        RETURN size((p)-[:KNOWS]->()) as outgoing,
               size((p)<-[:KNOWS]-()) as incoming
        """)
    ConnectionCount countConnections(String name);

    @Query("""
        MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
        WITH p, count(m) as movies
        ORDER BY movies DESC
        LIMIT 10
        RETURN p.name as name, movies
        """)
    List<ActorMovieCount> findMostConnectedActors();

    @Query("""
        MATCH (p1:Person)-[:KNOWS]-(p2:Person)-[:KNOWS]-(p3:Person)
        WHERE p1 <> p3 AND NOT (p1)-[:KNOWS]-(p3)
        RETURN DISTINCT p1.name as person, collect(DISTINCT p3.name) as suggestions
        """)
    List<FriendSuggestion> suggestFriends();
}
```

## Relationship Indexes

```cypher
-- Create relationship index
CREATE INDEX acted_in_role FOR ()-[r:ACTED_IN]-() ON (r.role)

-- Create composite relationship index
CREATE INDEX friendship_since FOR ()-[r:FRIENDS_WITH]-() ON (r.since)

-- Full-text relationship index
CREATE FULLTEXT INDEX review_text FOR ()-[r:REVIEWED]-() ON EACH [r.text]
```

## Best Practices

| Do | Don't |
|----|-------|
| Model relationships explicitly | Use node properties for connections |
| Use relationship properties | Create intermediate nodes |
| Define direction semantically | Ignore relationship direction |
| Index frequently queried rel props | Query unindexed relationships |
| Use MERGE for idempotent ops | CREATE duplicates |
| Fetch only needed relationships | Eager load entire graph |

## Production Checklist

- [ ] Relationships properly typed
- [ ] Directions semantically correct
- [ ] Relationship properties defined
- [ ] Indexes on queried rel properties
- [ ] Fetch strategies optimized
- [ ] Bidirectional updates handled
- [ ] Orphan cleanup implemented
