# Spring Cache Reference

## Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>

<!-- Caffeine (recommended) -->
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>

<!-- Or Redis -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

## Enable Caching

```java
@SpringBootApplication
@EnableCaching
public class Application { }
```

## Basic Caching

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    @Cacheable("users")
    public User findById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }

    @CachePut(value = "users", key = "#user.id")
    public User update(User user) {
        return userRepository.save(user);
    }

    @CacheEvict(value = "users", key = "#id")
    public void delete(Long id) {
        userRepository.deleteById(id);
    }

    @CacheEvict(value = "users", allEntries = true)
    public void clearCache() { }
}
```

## Cache Key Expressions

```java
// Simple key
@Cacheable(value = "users", key = "#id")
public User findById(Long id) { }

// Composite key
@Cacheable(value = "users", key = "#firstName + '_' + #lastName")
public User findByName(String firstName, String lastName) { }

// Object property
@Cacheable(value = "orders", key = "#request.userId")
public List<Order> findOrders(OrderRequest request) { }

// Method name as key
@Cacheable(value = "reports", key = "#root.methodName")
public Report generateReport() { }
```

## Conditional Caching

```java
// Only cache if result is not null
@Cacheable(value = "users", key = "#id", unless = "#result == null")
public User findById(Long id) { }

// Only cache for certain conditions
@Cacheable(value = "products", condition = "#price > 100")
public Product findByPrice(double price) { }

// Combined
@Cacheable(
    value = "orders",
    key = "#userId",
    condition = "#userId != null",
    unless = "#result.isEmpty()"
)
public List<Order> findByUserId(Long userId) { }
```

## Multiple Cache Operations

```java
@Caching(
    cacheable = @Cacheable(value = "users", key = "#id"),
    put = @CachePut(value = "users-email", key = "#result.email")
)
public User findById(Long id) { }

@Caching(evict = {
    @CacheEvict(value = "users", key = "#id"),
    @CacheEvict(value = "users-email", allEntries = true)
})
public void delete(Long id) { }
```

## Caffeine Configuration

```yaml
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=500,expireAfterWrite=10m
    cache-names:
      - users
      - products
      - orders
```

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .recordStats());
        return manager;
    }

    // Different configs per cache
    @Bean
    public CacheManager customCacheManager() {
        SimpleCacheManager manager = new SimpleCacheManager();
        manager.setCaches(List.of(
            buildCache("users", 500, Duration.ofMinutes(30)),
            buildCache("products", 1000, Duration.ofHours(1)),
            buildCache("sessions", 100, Duration.ofMinutes(5))
        ));
        return manager;
    }

    private CaffeineCache buildCache(String name, int size, Duration ttl) {
        return new CaffeineCache(name, Caffeine.newBuilder()
            .maximumSize(size)
            .expireAfterWrite(ttl)
            .build());
    }
}
```

## Redis Configuration

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 600000  # 10 minutes in ms
      cache-null-values: false
  data:
    redis:
      host: localhost
      port: 6379
```

```java
@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeValuesWith(SerializationPair.fromSerializer(
                new GenericJackson2JsonRedisSerializer()
            ));

        Map<String, RedisCacheConfiguration> configs = Map.of(
            "users", defaultConfig.entryTtl(Duration.ofMinutes(30)),
            "sessions", defaultConfig.entryTtl(Duration.ofMinutes(5))
        );

        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(configs)
            .build();
    }
}
```

## Cache at Class Level

```java
@Service
@CacheConfig(cacheNames = "users")
public class UserService {

    @Cacheable(key = "#id")
    public User findById(Long id) { }

    @CacheEvict(key = "#id")
    public void delete(Long id) { }
}
```

## Sync Mode

```java
// Ensures only one thread computes the value
@Cacheable(value = "expensive", sync = true)
public ExpensiveResult compute(String key) {
    return performExpensiveComputation(key);
}
```

## Cache Statistics

```java
@Autowired
private CacheManager cacheManager;

public CacheStats getStats(String cacheName) {
    CaffeineCache cache = (CaffeineCache) cacheManager.getCache(cacheName);
    return cache.getNativeCache().stats();
}
```

## Testing

```java
@SpringBootTest
class UserServiceCacheTest {

    @Autowired
    private UserService userService;

    @Autowired
    private CacheManager cacheManager;

    @BeforeEach
    void clearCache() {
        cacheManager.getCache("users").clear();
    }

    @Test
    void shouldCacheUser() {
        userService.findById(1L);  // Cache miss
        userService.findById(1L);  // Cache hit

        verify(userRepository, times(1)).findById(1L);
    }
}
```
