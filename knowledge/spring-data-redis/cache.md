# Spring Data Redis - Caching

## Overview

Spring Data Redis integrates with Spring's caching abstraction, providing Redis as a cache provider for @Cacheable annotations.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

## Enable Caching

```java
@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .disableCachingNullValues()
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new StringRedisSerializer()))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig)
            .build();
    }
}
```

## Cache Configuration

### Multiple Cache Configurations
```java
@Bean
public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
    RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(30))
        .disableCachingNullValues();

    Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();

    // Short-lived cache
    cacheConfigurations.put("sessionCache", RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(5)));

    // Long-lived cache
    cacheConfigurations.put("referenceDataCache", RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofHours(24)));

    // Product cache with custom serializer
    cacheConfigurations.put("productCache", RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofHours(1))
        .serializeValuesWith(
            RedisSerializationContext.SerializationPair.fromSerializer(
                new Jackson2JsonRedisSerializer<>(Product.class))));

    return RedisCacheManager.builder(connectionFactory)
        .cacheDefaults(defaultConfig)
        .withInitialCacheConfigurations(cacheConfigurations)
        .transactionAware()
        .build();
}
```

### Configuration Properties
```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 30m
      key-prefix: "myapp:"
      use-key-prefix: true
      cache-null-values: false
```

## Basic Caching

### @Cacheable
```java
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        // This method is only called if the result is not in cache
        return productRepository.findById(id).orElseThrow();
    }

    @Cacheable(value = "products", key = "#sku", unless = "#result == null")
    public Product findBySku(String sku) {
        return productRepository.findBySku(sku);
    }

    @Cacheable(value = "productsByCategory", key = "#category")
    public List<Product> findByCategory(String category) {
        return productRepository.findByCategory(category);
    }
}
```

### @CachePut
```java
@Service
public class ProductService {

    @CachePut(value = "products", key = "#product.id")
    public Product update(Product product) {
        // Always executes and updates the cache
        return productRepository.save(product);
    }

    @CachePut(value = "products", key = "#result.id")
    public Product create(CreateProductRequest request) {
        Product product = new Product(request);
        return productRepository.save(product);
    }
}
```

### @CacheEvict
```java
@Service
public class ProductService {

    @CacheEvict(value = "products", key = "#id")
    public void delete(Long id) {
        productRepository.deleteById(id);
    }

    @CacheEvict(value = "products", allEntries = true)
    public void clearProductCache() {
        // Clears all entries in the "products" cache
    }

    @CacheEvict(value = {"products", "productsByCategory"}, allEntries = true)
    public void clearAllProductCaches() {
        // Clears multiple caches
    }

    @CacheEvict(value = "products", key = "#product.id", beforeInvocation = true)
    public void deleteWithEvictFirst(Product product) {
        // Evicts before method execution (even if exception occurs)
        productRepository.delete(product);
    }
}
```

### @Caching
```java
@Service
public class ProductService {

    @Caching(
        put = {
            @CachePut(value = "products", key = "#result.id"),
            @CachePut(value = "productsBySku", key = "#result.sku")
        },
        evict = {
            @CacheEvict(value = "productsByCategory", key = "#result.category")
        }
    )
    public Product update(Product product) {
        return productRepository.save(product);
    }
}
```

## Key Generation

### SpEL Key Expressions
```java
@Service
public class UserService {

    // Simple parameter
    @Cacheable(value = "users", key = "#id")
    public User findById(Long id) { ... }

    // Object property
    @Cacheable(value = "users", key = "#request.userId")
    public User findByRequest(UserRequest request) { ... }

    // Multiple parameters
    @Cacheable(value = "users", key = "#firstName + '_' + #lastName")
    public User findByName(String firstName, String lastName) { ... }

    // Method result
    @CachePut(value = "users", key = "#result.id")
    public User create(UserRequest request) { ... }

    // Root object
    @Cacheable(value = "users", key = "#root.methodName + '_' + #id")
    public User findById(Long id) { ... }

    // Conditional
    @Cacheable(value = "users", key = "#id", condition = "#id > 0")
    public User findById(Long id) { ... }
}
```

### Custom Key Generator
```java
@Component
public class CustomKeyGenerator implements KeyGenerator {

    @Override
    public Object generate(Object target, Method method, Object... params) {
        StringBuilder sb = new StringBuilder();
        sb.append(target.getClass().getSimpleName()).append(":");
        sb.append(method.getName()).append(":");
        for (Object param : params) {
            sb.append(param != null ? param.toString() : "null").append(":");
        }
        return sb.toString();
    }
}

@Service
public class ProductService {

    @Cacheable(value = "products", keyGenerator = "customKeyGenerator")
    public Product findByComplexCriteria(SearchCriteria criteria) {
        return productRepository.findByCriteria(criteria);
    }
}
```

## Conditional Caching

```java
@Service
public class ProductService {

    // Only cache if condition is true
    @Cacheable(value = "products", key = "#id", condition = "#id > 100")
    public Product findById(Long id) { ... }

    // Cache unless result matches
    @Cacheable(value = "products", key = "#id", unless = "#result == null")
    public Product findByIdNullable(Long id) { ... }

    // Combined conditions
    @Cacheable(
        value = "products",
        key = "#id",
        condition = "#id > 0",
        unless = "#result.price < 10"
    )
    public Product findById(Long id) { ... }

    // SpEL with boolean expressions
    @Cacheable(
        value = "products",
        key = "#sku",
        condition = "#sku != null && #sku.length() > 0",
        unless = "#result?.active == false"
    )
    public Product findBySku(String sku) { ... }
}
```

## Cache Synchronization

```java
@Service
public class InventoryService {

    // Synchronized cache access (prevents cache stampede)
    @Cacheable(value = "inventory", key = "#productId", sync = true)
    public Integer getStock(Long productId) {
        // Only one thread will execute this for the same key
        return inventoryRepository.findStockByProductId(productId);
    }
}
```

## Programmatic Caching

```java
@Service
@RequiredArgsConstructor
public class CacheService {

    private final CacheManager cacheManager;

    public void put(String cacheName, String key, Object value) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            cache.put(key, value);
        }
    }

    public <T> T get(String cacheName, String key, Class<T> type) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            Cache.ValueWrapper wrapper = cache.get(key);
            if (wrapper != null) {
                return type.cast(wrapper.get());
            }
        }
        return null;
    }

    public <T> T getOrLoad(String cacheName, String key, Callable<T> loader) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            return cache.get(key, loader);
        }
        try {
            return loader.call();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public void evict(String cacheName, String key) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            cache.evict(key);
        }
    }

    public void clear(String cacheName) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            cache.clear();
        }
    }
}
```

## TTL Management

### Per-Cache TTL
```java
@Configuration
public class CacheTTLConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        Map<String, RedisCacheConfiguration> configs = Map.of(
            "shortLived", createConfig(Duration.ofMinutes(5)),
            "mediumLived", createConfig(Duration.ofHours(1)),
            "longLived", createConfig(Duration.ofDays(1))
        );

        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(createConfig(Duration.ofMinutes(30)))
            .withInitialCacheConfigurations(configs)
            .build();
    }

    private RedisCacheConfiguration createConfig(Duration ttl) {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(ttl)
            .disableCachingNullValues();
    }
}
```

### Dynamic TTL
```java
@Component
public class DynamicTtlCacheManager extends RedisCacheManager {

    private final Map<String, Duration> ttlMap = new ConcurrentHashMap<>();

    public DynamicTtlCacheManager(RedisCacheWriter cacheWriter,
                                   RedisCacheConfiguration defaultConfig) {
        super(cacheWriter, defaultConfig);
    }

    public void setTtl(String cacheName, Duration ttl) {
        ttlMap.put(cacheName, ttl);
    }

    @Override
    protected RedisCache createRedisCache(String name,
                                          RedisCacheConfiguration config) {
        Duration ttl = ttlMap.get(name);
        if (ttl != null) {
            config = config.entryTtl(ttl);
        }
        return super.createRedisCache(name, config);
    }
}
```

## Cache Events

```java
@Component
@Slf4j
public class CacheEventListener {

    @EventListener
    public void onCacheEvict(CacheEvictEvent event) {
        log.info("Cache evicted: {} - key: {}",
            event.getCacheName(), event.getKey());
    }
}

// Custom event publishing
@Service
@RequiredArgsConstructor
public class CacheEventPublisher {

    private final ApplicationEventPublisher publisher;
    private final CacheManager cacheManager;

    public void evictAndPublish(String cacheName, String key) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            cache.evict(key);
            publisher.publishEvent(new CacheEvictEvent(this, cacheName, key));
        }
    }
}

public class CacheEvictEvent extends ApplicationEvent {
    private final String cacheName;
    private final String key;

    public CacheEvictEvent(Object source, String cacheName, String key) {
        super(source);
        this.cacheName = cacheName;
        this.key = key;
    }

    // Getters
}
```

## Cache Statistics

```java
@Configuration
public class CacheStatisticsConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig())
            .enableStatistics()
            .build();
    }
}

@RestController
@RequiredArgsConstructor
public class CacheStatsController {

    private final RedisCacheManager cacheManager;

    @GetMapping("/cache/stats")
    public Map<String, CacheStatistics> getStats() {
        Map<String, CacheStatistics> stats = new HashMap<>();
        cacheManager.getCacheNames().forEach(name -> {
            Cache cache = cacheManager.getCache(name);
            if (cache instanceof RedisCache redisCache) {
                stats.put(name, redisCache.getStatistics());
            }
        });
        return stats;
    }
}
```

## Cache Warming

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class CacheWarmer implements ApplicationRunner {

    private final ProductService productService;
    private final CategoryService categoryService;

    @Override
    public void run(ApplicationArguments args) {
        log.info("Warming up caches...");

        // Pre-load frequently accessed data
        categoryService.findAllCategories()
            .forEach(category ->
                productService.findByCategory(category.getName()));

        // Pre-load top products
        productService.findTopProducts(100);

        log.info("Cache warm-up complete");
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Set appropriate TTL per cache | Use infinite TTL |
| Use sync=true for expensive operations | Allow cache stampede |
| Invalidate related caches together | Leave stale data |
| Monitor cache hit rates | Ignore cache statistics |
| Use meaningful cache names | Generic names |
| Test cache behavior | Assume caching works |

## Production Checklist

- [ ] TTL configured per cache type
- [ ] Cache null values handled
- [ ] Serializers configured
- [ ] Cache statistics enabled
- [ ] Eviction strategy defined
- [ ] Cache warming implemented
- [ ] Monitoring in place
- [ ] Error handling for cache failures
