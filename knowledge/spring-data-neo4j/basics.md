# Spring Data Neo4j - Basics

## Overview

Spring Data Neo4j provides integration with Neo4j graph database, offering object-graph mapping (OGM), repository support, and Cypher query capabilities for modeling connected data.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-neo4j</artifactId>
</dependency>
```

## Configuration

### application.yml
```yaml
spring:
  neo4j:
    uri: bolt://localhost:7687
    authentication:
      username: neo4j
      password: ${NEO4J_PASSWORD}
    pool:
      max-connection-pool-size: 100
      connection-acquisition-timeout: 60s
      max-connection-lifetime: 1h
```

### Java Configuration
```java
@Configuration
@EnableNeo4jRepositories(basePackages = "com.example.repository")
@EnableTransactionManagement
public class Neo4jConfig extends AbstractNeo4jConfig {

    @Bean
    public Driver driver() {
        return GraphDatabase.driver(
            "bolt://localhost:7687",
            AuthTokens.basic("neo4j", "password"),
            Config.builder()
                .withMaxConnectionPoolSize(100)
                .withConnectionAcquisitionTimeout(60, TimeUnit.SECONDS)
                .withMaxConnectionLifetime(1, TimeUnit.HOURS)
                .withEncryption()
                .build()
        );
    }

    @Override
    protected Collection<String> getMappingBasePackages() {
        return Collections.singleton("com.example.domain");
    }
}
```

## Node Entities

### Basic Node
```java
@Node("Person")
public class Person {

    @Id
    @GeneratedValue
    private Long id;

    @Property("name")
    private String name;

    @Property("born")
    private Integer birthYear;

    @Property
    private String email;

    // Getters and setters
}
```

### Node with Relationships
```java
@Node("Movie")
public class Movie {

    @Id
    @GeneratedValue
    private Long id;

    private String title;
    private Integer released;
    private String tagline;

    @Relationship(type = "ACTED_IN", direction = Relationship.Direction.INCOMING)
    private List<Person> actors;

    @Relationship(type = "DIRECTED", direction = Relationship.Direction.INCOMING)
    private Person director;

    @Relationship(type = "PRODUCED", direction = Relationship.Direction.INCOMING)
    private List<Person> producers;

    // Getters and setters
}
```

### Custom ID
```java
@Node("Product")
public class Product {

    @Id
    private String sku;  // Application-provided ID

    private String name;
    private BigDecimal price;
}

@Node("Order")
public class Order {

    @Id
    @GeneratedValue(generatorClass = UUIDStringGenerator.class)
    private String id;

    private LocalDateTime createdAt;
}
```

## Neo4jTemplate

```java
@Service
@RequiredArgsConstructor
public class PersonService {

    private final Neo4jTemplate neo4jTemplate;

    public Person save(Person person) {
        return neo4jTemplate.save(person);
    }

    public Optional<Person> findById(Long id) {
        return neo4jTemplate.findById(id, Person.class);
    }

    public List<Person> findAll() {
        return neo4jTemplate.findAll(Person.class);
    }

    public long count() {
        return neo4jTemplate.count(Person.class);
    }

    public void deleteById(Long id) {
        neo4jTemplate.deleteById(id, Person.class);
    }

    public boolean existsById(Long id) {
        return neo4jTemplate.existsById(id, Person.class);
    }

    public List<Person> findByExample(Person example) {
        return neo4jTemplate.findAll(
            Example.of(example),
            Person.class
        );
    }
}
```

## Neo4jClient (Low-level)

```java
@Service
@RequiredArgsConstructor
public class CypherService {

    private final Neo4jClient client;

    public List<Person> findPeopleByName(String name) {
        return client
            .query("MATCH (p:Person) WHERE p.name CONTAINS $name RETURN p")
            .bind(name).to("name")
            .fetchAs(Person.class)
            .mappedBy((typeSystem, record) -> {
                Node node = record.get("p").asNode();
                Person person = new Person();
                person.setId(node.id());
                person.setName(node.get("name").asString());
                return person;
            })
            .all();
    }

    public void createRelationship(Long personId, Long movieId, String role) {
        client
            .query("""
                MATCH (p:Person), (m:Movie)
                WHERE id(p) = $personId AND id(m) = $movieId
                CREATE (p)-[:ACTED_IN {role: $role}]->(m)
                """)
            .bind(personId).to("personId")
            .bind(movieId).to("movieId")
            .bind(role).to("role")
            .run();
    }

    public Map<String, Object> getStatistics() {
        return client
            .query("""
                MATCH (p:Person)
                RETURN count(p) as personCount,
                       avg(p.born) as avgBirthYear
                """)
            .fetch()
            .one()
            .orElse(Collections.emptyMap());
    }
}
```

## Transactions

```java
@Service
@RequiredArgsConstructor
public class TransactionalService {

    private final Neo4jTemplate template;
    private final PersonRepository personRepository;

    @Transactional
    public void transferFriendship(Long fromId, Long toId, Long friendId) {
        // All operations in one transaction
        Person from = template.findById(fromId, Person.class)
            .orElseThrow();
        Person to = template.findById(toId, Person.class)
            .orElseThrow();
        Person friend = template.findById(friendId, Person.class)
            .orElseThrow();

        from.getFriends().remove(friend);
        to.getFriends().add(friend);

        template.save(from);
        template.save(to);
    }

    @Transactional(readOnly = true)
    public List<Person> findAllPeople() {
        return personRepository.findAll();
    }
}
```

## Reactive Neo4j

```java
@Configuration
@EnableReactiveNeo4jRepositories
public class ReactiveNeo4jConfig extends AbstractReactiveNeo4jConfig {

    @Bean
    public Driver driver() {
        return GraphDatabase.driver("bolt://localhost:7687",
            AuthTokens.basic("neo4j", "password"));
    }

    @Override
    protected Collection<String> getMappingBasePackages() {
        return Collections.singleton("com.example.domain");
    }
}

@Service
@RequiredArgsConstructor
public class ReactivePersonService {

    private final ReactiveNeo4jTemplate template;

    public Mono<Person> save(Person person) {
        return template.save(person);
    }

    public Mono<Person> findById(Long id) {
        return template.findById(id, Person.class);
    }

    public Flux<Person> findAll() {
        return template.findAll(Person.class);
    }

    public Mono<Long> count() {
        return template.count(Person.class);
    }
}
```

## Projections

```java
// Interface projection
public interface PersonSummary {
    String getName();
    Integer getBirthYear();
}

// DTO projection
public record PersonDto(String name, Integer birthYear) {}

@Service
public class ProjectionService {

    @Autowired
    private Neo4jTemplate template;

    public List<PersonSummary> findAllSummaries() {
        return template.findAll(Person.class).stream()
            .map(p -> new PersonSummary() {
                @Override
                public String getName() { return p.getName(); }
                @Override
                public Integer getBirthYear() { return p.getBirthYear(); }
            })
            .collect(Collectors.toList());
    }
}
```

## Error Handling

```java
@Service
@Slf4j
public class SafeNeo4jService {

    @Autowired
    private Neo4jTemplate template;

    public Optional<Person> safeFindById(Long id) {
        try {
            return template.findById(id, Person.class);
        } catch (Neo4jException e) {
            log.error("Neo4j error: {}", e.getMessage());
            return Optional.empty();
        }
    }

    @Transactional
    public Person saveWithRetry(Person person, int maxRetries) {
        int attempts = 0;
        while (attempts < maxRetries) {
            try {
                return template.save(person);
            } catch (TransientException e) {
                attempts++;
                if (attempts >= maxRetries) {
                    throw e;
                }
                log.warn("Transient error, retrying... attempt {}", attempts);
            }
        }
        throw new RuntimeException("Max retries exceeded");
    }
}
```

## Auditing

```java
@Configuration
@EnableNeo4jAuditing
public class AuditConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.of(
            SecurityContextHolder.getContext()
                .getAuthentication().getName()
        );
    }
}

@Node("AuditedEntity")
public class AuditedEntity {

    @Id
    @GeneratedValue
    private Long id;

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

## Best Practices

| Do | Don't |
|----|-------|
| Model relationships explicitly | Use implicit relationships |
| Use appropriate ID strategy | Always use auto-generated IDs |
| Define direction on relationships | Ignore relationship direction |
| Use projections for partial data | Fetch entire graphs |
| Enable transactions | Skip transaction management |
| Index frequently queried properties | Query unindexed properties |

## Production Checklist

- [ ] Connection pooling configured
- [ ] Transactions properly scoped
- [ ] Indexes created for queries
- [ ] Relationship directions correct
- [ ] Eager/lazy loading considered
- [ ] Error handling implemented
- [ ] Monitoring enabled
