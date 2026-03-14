# Spring Data Redis - Repositories

## Overview

Spring Data Redis provides repository support, allowing you to use the familiar Spring Data repository abstraction with Redis as the backing store.

## Enable Repositories

```java
@Configuration
@EnableRedisRepositories
public class RedisRepositoryConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

## Entity Definition

```java
@RedisHash("users")
public class User {

    @Id
    private String id;

    @Indexed
    private String email;

    @Indexed
    private String username;

    private String firstName;
    private String lastName;

    @TimeToLive
    private Long expiration;

    @Reference
    private Address address;

    // Getters and setters
}

@RedisHash("addresses")
public class Address {

    @Id
    private String id;

    private String street;
    private String city;
    private String country;

    // Getters and setters
}
```

## Repository Interface

```java
public interface UserRepository extends CrudRepository<User, String> {

    // Find by indexed field
    Optional<User> findByEmail(String email);

    List<User> findByUsername(String username);

    // Find by multiple indexed fields
    List<User> findByUsernameAndEmail(String username, String email);

    // Check existence
    boolean existsByEmail(String email);

    // Delete by indexed field
    void deleteByEmail(String email);
}
```

## Using Repositories

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    public User create(User user) {
        // ID is generated if not set
        return userRepository.save(user);
    }

    public Optional<User> findById(String id) {
        return userRepository.findById(id);
    }

    public Optional<User> findByEmail(String email) {
        return userRepository.findByEmail(email);
    }

    public List<User> findAll() {
        List<User> users = new ArrayList<>();
        userRepository.findAll().forEach(users::add);
        return users;
    }

    public void delete(String id) {
        userRepository.deleteById(id);
    }

    public long count() {
        return userRepository.count();
    }
}
```

## Time To Live (TTL)

### Field-based TTL
```java
@RedisHash("sessions")
public class Session {

    @Id
    private String id;

    private String userId;

    @TimeToLive
    private Long ttl;  // in seconds

    public Session(String userId, Duration duration) {
        this.userId = userId;
        this.ttl = duration.getSeconds();
    }
}
```

### Class-level TTL
```java
@RedisHash(value = "sessions", timeToLive = 3600)  // 1 hour
public class Session {

    @Id
    private String id;

    private String userId;
}
```

### Programmatic TTL
```java
@Service
public class SessionService {

    @Autowired
    private RedisKeyValueTemplate keyValueTemplate;

    public void updateTtl(String id, Duration newTtl) {
        keyValueTemplate.expire(id, newTtl);
    }
}
```

## Indexing

### Simple Index
```java
@RedisHash("products")
public class Product {

    @Id
    private String id;

    @Indexed
    private String category;

    @Indexed
    private String brand;

    private String name;
    private BigDecimal price;
}

public interface ProductRepository extends CrudRepository<Product, String> {

    List<Product> findByCategory(String category);

    List<Product> findByBrand(String brand);

    List<Product> findByCategoryAndBrand(String category, String brand);
}
```

### Compound Index (Manual)
```java
@RedisHash("orders")
public class Order {

    @Id
    private String id;

    @Indexed
    private String customerId;

    @Indexed
    private String status;

    private LocalDateTime createdAt;

    // Custom indexed field combining multiple values
    @Indexed
    public String getCustomerStatus() {
        return customerId + ":" + status;
    }
}

public interface OrderRepository extends CrudRepository<Order, String> {

    List<Order> findByCustomerStatus(String customerStatus);

    default List<Order> findByCustomerIdAndStatus(String customerId, String status) {
        return findByCustomerStatus(customerId + ":" + status);
    }
}
```

## References

### One-to-One Reference
```java
@RedisHash("users")
public class User {

    @Id
    private String id;

    private String name;

    @Reference
    private Profile profile;
}

@RedisHash("profiles")
public class Profile {

    @Id
    private String id;

    private String bio;
    private String avatarUrl;
}
```

### One-to-Many Reference
```java
@RedisHash("authors")
public class Author {

    @Id
    private String id;

    private String name;

    @Reference
    private List<Book> books = new ArrayList<>();
}

@RedisHash("books")
public class Book {

    @Id
    private String id;

    private String title;
    private String isbn;
}
```

## Partial Updates

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final RedisKeyValueTemplate keyValueTemplate;

    public void updateEmail(String userId, String newEmail) {
        keyValueTemplate.update(
            new PartialUpdate<>(userId, User.class)
                .set("email", newEmail)
        );
    }

    public void updateMultipleFields(String userId, String email, String firstName) {
        keyValueTemplate.update(
            new PartialUpdate<>(userId, User.class)
                .set("email", email)
                .set("firstName", firstName)
        );
    }

    public void deleteField(String userId) {
        keyValueTemplate.update(
            new PartialUpdate<>(userId, User.class)
                .del("temporaryField")
        );
    }
}
```

## Custom Repository Implementation

```java
public interface UserRepositoryCustom {
    List<User> findActiveUsers();
    void bulkUpdateStatus(List<String> userIds, String status);
}

public class UserRepositoryImpl implements UserRepositoryCustom {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Override
    public List<User> findActiveUsers() {
        Set<String> keys = redisTemplate.keys("users:*");
        return keys.stream()
            .map(key -> redisTemplate.opsForHash().entries(key))
            .filter(map -> "ACTIVE".equals(map.get("status")))
            .map(this::mapToUser)
            .collect(Collectors.toList());
    }

    @Override
    public void bulkUpdateStatus(List<String> userIds, String status) {
        redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            for (String userId : userIds) {
                connection.hSet(
                    ("users:" + userId).getBytes(),
                    "status".getBytes(),
                    status.getBytes()
                );
            }
            return null;
        });
    }

    private User mapToUser(Map<Object, Object> map) {
        // Map implementation
    }
}

public interface UserRepository extends CrudRepository<User, String>, UserRepositoryCustom {
    // Standard repository methods
}
```

## Keyspace Configuration

```java
@Configuration
public class RedisKeyspaceConfig {

    @Bean
    public RedisMappingContext keyValueMappingContext() {
        return new RedisMappingContext(
            new MappingConfiguration(
                new IndexConfiguration() {
                    @Override
                    protected Iterable<IndexDefinition> initialConfiguration() {
                        return Collections.singletonList(
                            new SimpleIndexDefinition("users", "email")
                        );
                    }
                },
                new KeyspaceConfiguration() {
                    @Override
                    protected Iterable<KeyspaceSettings> initialConfiguration() {
                        return Collections.singletonList(
                            new KeyspaceSettings(User.class, "app:users")
                        );
                    }
                }
            )
        );
    }
}
```

## Transactions

```java
@Service
public class TransactionalUserService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    public void transferPoints(String fromUserId, String toUserId, int points) {
        redisTemplate.execute(new SessionCallback<List<Object>>() {
            @Override
            public List<Object> execute(RedisOperations operations) {
                operations.watch("user:" + fromUserId + ":points");
                operations.watch("user:" + toUserId + ":points");

                int fromPoints = (int) operations.opsForValue()
                    .get("user:" + fromUserId + ":points");

                if (fromPoints < points) {
                    operations.unwatch();
                    throw new InsufficientPointsException();
                }

                operations.multi();
                operations.opsForValue()
                    .decrement("user:" + fromUserId + ":points", points);
                operations.opsForValue()
                    .increment("user:" + toUserId + ":points", points);

                return operations.exec();
            }
        });
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use @Indexed for queried fields | Index all fields |
| Set appropriate TTL | Let data grow unbounded |
| Use references for relationships | Embed large objects |
| Use partial updates | Save entire entity for small changes |
| Design keys with namespaces | Use flat key structure |
| Use pipelining for bulk ops | Individual saves in loops |

## Production Checklist

- [ ] Entities properly annotated
- [ ] Indexes defined for query fields
- [ ] TTL configured appropriately
- [ ] References used for related entities
- [ ] Key namespaces planned
- [ ] Serialization tested
- [ ] Repository methods tested
