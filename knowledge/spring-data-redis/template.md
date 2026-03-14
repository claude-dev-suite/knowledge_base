# Spring Data Redis - RedisTemplate

## Overview

RedisTemplate is the central class for Redis operations, providing high-level abstractions for working with Redis data structures.

## Configuration

```java
@Configuration
public class RedisTemplateConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);

        // Configure serializers
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());

        template.setEnableTransactionSupport(true);
        template.afterPropertiesSet();

        return template;
    }

    @Bean
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory connectionFactory) {
        return new StringRedisTemplate(connectionFactory);
    }
}
```

## Operations Overview

```java
@Service
@RequiredArgsConstructor
public class RedisOperationsService {

    private final RedisTemplate<String, Object> redisTemplate;

    // Value Operations (Strings)
    public ValueOperations<String, Object> valueOps() {
        return redisTemplate.opsForValue();
    }

    // Hash Operations
    public HashOperations<String, String, Object> hashOps() {
        return redisTemplate.opsForHash();
    }

    // List Operations
    public ListOperations<String, Object> listOps() {
        return redisTemplate.opsForList();
    }

    // Set Operations
    public SetOperations<String, Object> setOps() {
        return redisTemplate.opsForSet();
    }

    // Sorted Set Operations
    public ZSetOperations<String, Object> zSetOps() {
        return redisTemplate.opsForZSet();
    }

    // Stream Operations
    public StreamOperations<String, String, Object> streamOps() {
        return redisTemplate.opsForStream();
    }

    // Geo Operations
    public GeoOperations<String, Object> geoOps() {
        return redisTemplate.opsForGeo();
    }

    // HyperLogLog Operations
    public HyperLogLogOperations<String, Object> hyperLogLogOps() {
        return redisTemplate.opsForHyperLogLog();
    }

    // Cluster Operations
    public ClusterOperations<String, Object> clusterOps() {
        return redisTemplate.opsForCluster();
    }
}
```

## Value Operations (Strings)

```java
@Service
public class ValueOperationsService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    public void stringOperations() {
        ValueOperations<String, Object> ops = redisTemplate.opsForValue();

        // Set value
        ops.set("key", "value");

        // Set with expiry
        ops.set("key", "value", Duration.ofMinutes(30));

        // Set if absent
        Boolean wasSet = ops.setIfAbsent("key", "value");
        ops.setIfAbsent("key", "value", Duration.ofMinutes(30));

        // Set if present
        Boolean wasUpdated = ops.setIfPresent("key", "newValue");

        // Get value
        Object value = ops.get("key");

        // Get and set
        Object oldValue = ops.getAndSet("key", "newValue");

        // Get and delete
        Object deleted = ops.getAndDelete("key");

        // Get and expire
        Object valueWithNewTtl = ops.getAndExpire("key", Duration.ofHours(1));

        // Multi-get
        List<Object> values = ops.multiGet(List.of("key1", "key2", "key3"));

        // Multi-set
        Map<String, Object> keyValues = Map.of(
            "key1", "value1",
            "key2", "value2"
        );
        ops.multiSet(keyValues);

        // Increment/Decrement
        Long incrementedLong = ops.increment("counter");
        Long incrementedBy = ops.increment("counter", 5);
        Double incrementedDouble = ops.increment("price", 0.99);
        Long decremented = ops.decrement("counter");

        // Append
        Integer newLength = ops.append("key", " appended");

        // Bit operations
        Boolean bit = ops.setBit("bitmap", 7, true);
        Boolean getBit = ops.getBit("bitmap", 7);
    }
}
```

## Hash Operations

```java
@Service
public class HashOperationsService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    public void hashOperations() {
        HashOperations<String, String, Object> ops = redisTemplate.opsForHash();

        // Put single field
        ops.put("user:1", "name", "John");
        ops.put("user:1", "email", "john@example.com");

        // Put if absent
        Boolean wasSet = ops.putIfAbsent("user:1", "name", "Jane");

        // Put all
        Map<String, Object> fields = Map.of(
            "name", "John",
            "email", "john@example.com",
            "age", 30
        );
        ops.putAll("user:1", fields);

        // Get single field
        Object name = ops.get("user:1", "name");

        // Get multiple fields
        List<Object> values = ops.multiGet("user:1", List.of("name", "email"));

        // Get all entries
        Map<String, Object> allFields = ops.entries("user:1");

        // Get all keys
        Set<String> keys = ops.keys("user:1");

        // Get all values
        List<Object> allValues = ops.values("user:1");

        // Has key
        Boolean hasName = ops.hasKey("user:1", "name");

        // Delete fields
        Long deleted = ops.delete("user:1", "age", "temporary");

        // Size
        Long size = ops.size("user:1");

        // Increment
        Long newAge = ops.increment("user:1", "age", 1);
        Double newBalance = ops.increment("user:1", "balance", 99.99);

        // Scan fields
        Cursor<Map.Entry<String, Object>> cursor = ops.scan("user:1",
            ScanOptions.scanOptions().match("*").count(100).build());
    }
}
```

## List Operations

```java
@Service
public class ListOperationsService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    public void listOperations() {
        ListOperations<String, Object> ops = redisTemplate.opsForList();

        // Push operations
        Long leftPush = ops.leftPush("queue", "item1");
        Long rightPush = ops.rightPush("queue", "item2");

        // Push multiple
        Long leftPushAll = ops.leftPushAll("queue", "a", "b", "c");
        Long rightPushAll = ops.rightPushAll("queue", List.of("x", "y", "z"));

        // Push if exists
        Long pushIfPresent = ops.leftPushIfPresent("queue", "item");

        // Push before/after pivot
        Long pushBefore = ops.leftPush("queue", "pivot", "beforePivot");
        Long pushAfter = ops.rightPush("queue", "pivot", "afterPivot");

        // Pop operations
        Object leftPop = ops.leftPop("queue");
        Object rightPop = ops.rightPop("queue");

        // Pop with count
        List<Object> popped = ops.leftPop("queue", 5);

        // Blocking pop
        Object blockedPop = ops.leftPop("queue", Duration.ofSeconds(10));

        // Move between lists
        Object moved = ops.rightPopAndLeftPush("source", "destination");
        Object movedBlocking = ops.rightPopAndLeftPush("source", "destination",
            Duration.ofSeconds(10));

        // Range
        List<Object> range = ops.range("queue", 0, -1);  // All elements
        List<Object> first10 = ops.range("queue", 0, 9);

        // Get by index
        Object element = ops.index("queue", 0);

        // Set by index
        ops.set("queue", 0, "newValue");

        // Trim list
        ops.trim("queue", 0, 99);  // Keep first 100

        // Remove elements
        Long removed = ops.remove("queue", 0, "value");  // Remove all occurrences
        Long removedFirst = ops.remove("queue", 1, "value");  // Remove first
        Long removedLast = ops.remove("queue", -1, "value");  // Remove last

        // Size
        Long size = ops.size("queue");
    }
}
```

## Set Operations

```java
@Service
public class SetOperationsService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    public void setOperations() {
        SetOperations<String, Object> ops = redisTemplate.opsForSet();

        // Add members
        Long added = ops.add("tags", "java", "spring", "redis");

        // Remove members
        Long removed = ops.remove("tags", "oldTag");

        // Pop random
        Object popped = ops.pop("tags");
        List<Object> poppedMultiple = ops.pop("tags", 3);

        // Get random
        Object random = ops.randomMember("tags");
        List<Object> randomMultiple = ops.randomMembers("tags", 3);
        Set<Object> distinctRandom = ops.distinctRandomMembers("tags", 3);

        // Move between sets
        Boolean moved = ops.move("source", "member", "destination");

        // Check membership
        Boolean isMember = ops.isMember("tags", "java");
        Map<Object, Boolean> areMember = ops.isMember("tags", "java", "python");

        // Get all members
        Set<Object> members = ops.members("tags");

        // Size
        Long size = ops.size("tags");

        // Set operations
        Set<Object> intersection = ops.intersect("set1", "set2");
        Set<Object> intersectMultiple = ops.intersect("set1", List.of("set2", "set3"));

        Set<Object> union = ops.union("set1", "set2");
        Set<Object> unionMultiple = ops.union("set1", List.of("set2", "set3"));

        Set<Object> difference = ops.difference("set1", "set2");
        Set<Object> differenceMultiple = ops.difference("set1", List.of("set2", "set3"));

        // Store results
        Long storedIntersect = ops.intersectAndStore("set1", "set2", "result");
        Long storedUnion = ops.unionAndStore("set1", "set2", "result");
        Long storedDiff = ops.differenceAndStore("set1", "set2", "result");

        // Scan
        Cursor<Object> cursor = ops.scan("tags",
            ScanOptions.scanOptions().match("*").count(100).build());
    }
}
```

## Sorted Set Operations

```java
@Service
public class ZSetOperationsService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    public void zSetOperations() {
        ZSetOperations<String, Object> ops = redisTemplate.opsForZSet();

        // Add with score
        Boolean added = ops.add("leaderboard", "player1", 100.0);

        // Add multiple
        Set<ZSetOperations.TypedTuple<Object>> tuples = Set.of(
            ZSetOperations.TypedTuple.of("player2", 90.0),
            ZSetOperations.TypedTuple.of("player3", 85.0)
        );
        Long addedCount = ops.add("leaderboard", tuples);

        // Add if absent
        Boolean addedIfAbsent = ops.addIfAbsent("leaderboard", "player4", 80.0);

        // Get score
        Double score = ops.score("leaderboard", "player1");

        // Get multiple scores
        List<Double> scores = ops.score("leaderboard", "player1", "player2");

        // Increment score
        Double newScore = ops.incrementScore("leaderboard", "player1", 10.0);

        // Get rank
        Long rank = ops.rank("leaderboard", "player1");  // 0-based, low to high
        Long reverseRank = ops.reverseRank("leaderboard", "player1");  // High to low

        // Range by rank
        Set<Object> top10 = ops.reverseRange("leaderboard", 0, 9);
        Set<ZSetOperations.TypedTuple<Object>> top10WithScores =
            ops.reverseRangeWithScores("leaderboard", 0, 9);

        // Range by score
        Set<Object> highScorers = ops.rangeByScore("leaderboard", 80, 100);
        Set<Object> highScorersReverse = ops.reverseRangeByScore("leaderboard", 80, 100);

        // Range by lex (for same scores)
        RedisZSetCommands.Range range = RedisZSetCommands.Range.range()
            .gte("a").lte("z");
        Set<Object> lexRange = ops.rangeByLex("alphabetSet", range);

        // Pop operations
        ZSetOperations.TypedTuple<Object> min = ops.popMin("leaderboard");
        ZSetOperations.TypedTuple<Object> max = ops.popMax("leaderboard");
        Set<ZSetOperations.TypedTuple<Object>> mins = ops.popMin("leaderboard", 3);

        // Blocking pop
        ZSetOperations.TypedTuple<Object> blockedMin =
            ops.popMin("leaderboard", Duration.ofSeconds(10));

        // Remove
        Long removed = ops.remove("leaderboard", "player1", "player2");
        Long removedByRank = ops.removeRange("leaderboard", 0, 5);
        Long removedByScore = ops.removeRangeByScore("leaderboard", 0, 50);

        // Count
        Long count = ops.count("leaderboard", 80, 100);
        Long lexCount = ops.lexCount("alphabetSet", range);
        Long size = ops.size("leaderboard");

        // Set operations
        Long intersected = ops.intersectAndStore("zset1", "zset2", "result");
        Long unioned = ops.unionAndStore("zset1", "zset2", "result");

        // Scan
        Cursor<ZSetOperations.TypedTuple<Object>> cursor = ops.scan("leaderboard",
            ScanOptions.scanOptions().match("*").count(100).build());
    }
}
```

## Stream Operations

```java
@Service
public class StreamOperationsService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    public void streamOperations() {
        StreamOperations<String, String, Object> ops = redisTemplate.opsForStream();

        // Add message
        RecordId id = ops.add("stream:events", Map.of(
            "type", "order",
            "orderId", "12345",
            "amount", "99.99"
        ));

        // Add with explicit ID
        RecordId explicitId = ops.add(StreamRecords.mapBacked(Map.of("key", "value"))
            .withStreamKey("stream:events")
            .withId(RecordId.of("1234567890-0")));

        // Read messages
        List<MapRecord<String, String, Object>> messages =
            ops.read(StreamOffset.fromStart("stream:events"));

        // Read with count
        List<MapRecord<String, String, Object>> limited = ops.read(
            StreamReadOptions.empty().count(10),
            StreamOffset.fromStart("stream:events")
        );

        // Blocking read
        List<MapRecord<String, String, Object>> blocked = ops.read(
            StreamReadOptions.empty().block(Duration.ofSeconds(10)),
            StreamOffset.latest("stream:events")
        );

        // Consumer groups
        String groupCreated = ops.createGroup("stream:events", "mygroup");
        ops.createGroup("stream:events", ReadOffset.from("0"), "mygroup2");

        // Read as consumer group
        List<MapRecord<String, String, Object>> groupMessages = ops.read(
            Consumer.from("mygroup", "consumer1"),
            StreamReadOptions.empty().count(10),
            StreamOffset.create("stream:events", ReadOffset.lastConsumed())
        );

        // Acknowledge
        Long acknowledged = ops.acknowledge("stream:events", "mygroup", id);

        // Pending messages
        PendingMessagesSummary pending = ops.pending("stream:events", "mygroup");
        PendingMessages pendingMessages = ops.pending("stream:events",
            Consumer.from("mygroup", "consumer1"));

        // Claim messages
        List<MapRecord<String, String, Object>> claimed = ops.claim("stream:events",
            "mygroup", "consumer2", Duration.ofMinutes(5), id);

        // Delete message
        Long deleted = ops.delete("stream:events", id);

        // Trim stream
        Long trimmed = ops.trim("stream:events", 1000);  // Keep last 1000

        // Stream info
        StreamInfo.XInfoStream info = ops.info("stream:events");
        StreamInfo.XInfoGroups groups = ops.groups("stream:events");
        StreamInfo.XInfoConsumers consumers = ops.consumers("stream:events", "mygroup");
    }
}
```

## Pipelining

```java
@Service
public class PipeliningService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    public List<Object> executePipeline() {
        return redisTemplate.executePipelined(new RedisCallback<Object>() {
            @Override
            public Object doInRedis(RedisConnection connection) {
                StringRedisConnection stringConn = (StringRedisConnection) connection;

                for (int i = 0; i < 1000; i++) {
                    stringConn.set("key:" + i, "value:" + i);
                }

                return null;  // Must return null
            }
        });
    }

    public List<Object> executePipelineWithOperations() {
        return redisTemplate.executePipelined((RedisOperations<String, Object> operations) -> {
            ValueOperations<String, Object> valueOps = operations.opsForValue();

            for (int i = 0; i < 1000; i++) {
                valueOps.set("key:" + i, "value:" + i);
            }

            return null;
        });
    }
}
```

## Scripting with Lua

```java
@Service
public class LuaScriptService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    // Rate limiter using Lua script
    private final RedisScript<Long> rateLimiterScript = new DefaultRedisScript<>(
        """
        local key = KEYS[1]
        local limit = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        local current = redis.call('INCR', key)
        if current == 1 then
            redis.call('EXPIRE', key, window)
        end
        if current > limit then
            return 0
        end
        return 1
        """,
        Long.class
    );

    public boolean isAllowed(String clientId, int limit, int windowSeconds) {
        Long result = redisTemplate.execute(
            rateLimiterScript,
            List.of("rate:" + clientId),
            String.valueOf(limit),
            String.valueOf(windowSeconds)
        );
        return result != null && result == 1;
    }

    // Atomic get and increment
    private final RedisScript<Long> getAndIncrScript = new DefaultRedisScript<>(
        """
        local current = redis.call('GET', KEYS[1])
        redis.call('INCR', KEYS[1])
        return current
        """,
        Long.class
    );
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Reuse RedisTemplate instances | Create new template per operation |
| Use pipelining for bulk operations | Individual commands in loops |
| Choose appropriate serializers | Use JDK serialization |
| Use Lua scripts for atomicity | Multiple round-trips for atomic ops |
| Bound operations for repeated key access | Create BoundOperations every call |
| Configure connection pooling | Create new connections |

## Production Checklist

- [ ] Connection pooling configured
- [ ] Appropriate serializers set
- [ ] Pipelining used for bulk operations
- [ ] Lua scripts for atomic operations
- [ ] Error handling in place
- [ ] Monitoring configured
