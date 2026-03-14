# Redis Production Configuration

> Comprehensive guide for running Redis in production with optimal performance, security, and high availability.

## Table of Contents

1. [Production Configuration](#production-configuration)
2. [Data Structures and Patterns](#data-structures-and-patterns)
3. [Caching Strategies](#caching-strategies)
4. [Persistence Configuration](#persistence-configuration)
5. [High Availability](#high-availability)
6. [Security Hardening](#security-hardening)
7. [Performance Optimization](#performance-optimization)
8. [Monitoring and Alerting](#monitoring-and-alerting)

---

## Production Configuration

### redis.conf Essential Settings

```conf
# /etc/redis/redis.conf

#------------------------------------------------------------------------------
# NETWORK
#------------------------------------------------------------------------------
bind 0.0.0.0
port 6379
protected-mode yes
tcp-backlog 511
timeout 0
tcp-keepalive 300

#------------------------------------------------------------------------------
# GENERAL
#------------------------------------------------------------------------------
daemonize no
supervised systemd
pidfile /var/run/redis/redis-server.pid
loglevel notice
logfile /var/log/redis/redis-server.log
databases 16

#------------------------------------------------------------------------------
# MEMORY MANAGEMENT
#------------------------------------------------------------------------------
maxmemory 4gb
maxmemory-policy allkeys-lru
maxmemory-samples 10

#------------------------------------------------------------------------------
# LAZY FREEING (for large key deletion)
#------------------------------------------------------------------------------
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
replica-lazy-flush yes

#------------------------------------------------------------------------------
# APPEND ONLY MODE
#------------------------------------------------------------------------------
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-use-rdb-preamble yes

#------------------------------------------------------------------------------
# RDB PERSISTENCE
#------------------------------------------------------------------------------
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /var/lib/redis

#------------------------------------------------------------------------------
# REPLICATION
#------------------------------------------------------------------------------
# replicaof <masterip> <masterport>
replica-serve-stale-data yes
replica-read-only yes
repl-diskless-sync yes
repl-diskless-sync-delay 5
repl-ping-replica-period 10
repl-timeout 60
repl-backlog-size 64mb
repl-backlog-ttl 3600
min-replicas-to-write 1
min-replicas-max-lag 10

#------------------------------------------------------------------------------
# SECURITY
#------------------------------------------------------------------------------
requirepass your-secure-password
# For Redis 6.0+, use ACLs instead
# user default on >password ~* &* +@all
# user readonly on >readonly_pass ~* &* +@read

#------------------------------------------------------------------------------
# CLIENT LIMITS
#------------------------------------------------------------------------------
maxclients 10000

#------------------------------------------------------------------------------
# SLOW LOG
#------------------------------------------------------------------------------
slowlog-log-slower-than 10000
slowlog-max-len 128

#------------------------------------------------------------------------------
# ADVANCED
#------------------------------------------------------------------------------
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
```

### Docker Production Configuration

```yaml
# docker-compose.redis-prod.yml
services:
  redis:
    image: redis:7-alpine
    container_name: redis-prod
    restart: unless-stopped

    command: >
      redis-server
      --appendonly yes
      --maxmemory 4gb
      --maxmemory-policy allkeys-lru
      --requirepass ${REDIS_PASSWORD}
      --save 900 1
      --save 300 10
      --save 60 10000

    volumes:
      - redis-data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf:ro

    deploy:
      resources:
        limits:
          memory: 5G
          cpus: "2"
        reservations:
          memory: 4G

    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

    networks:
      - cache

    sysctls:
      net.core.somaxconn: 1024

    ulimits:
      nofile:
        soft: 65536
        hard: 65536

networks:
  cache:
    internal: true

volumes:
  redis-data:
```

---

## Data Structures and Patterns

### Strings

```redis
# Basic key-value
SET user:1:name "John Doe"
GET user:1:name

# With expiration
SET session:abc123 "user_data" EX 3600  # 1 hour

# Atomic operations
INCR page:views
INCRBY user:1:score 10
DECR inventory:product:1

# Set if not exists
SETNX lock:resource "owner"

# Set with NX and EX (distributed lock pattern)
SET lock:resource "owner" NX EX 30

# Multiple operations
MSET user:1:name "John" user:1:email "john@example.com"
MGET user:1:name user:1:email
```

### Hashes

```redis
# User profile
HSET user:1 name "John Doe" email "john@example.com" age 30
HGET user:1 name
HGETALL user:1
HMGET user:1 name email

# Increment field
HINCRBY user:1 login_count 1

# Check field existence
HEXISTS user:1 email

# Get all fields or values
HKEYS user:1
HVALS user:1
HLEN user:1
```

### Lists

```redis
# Queue pattern (FIFO)
LPUSH queue:tasks "task1"
RPOP queue:tasks

# Stack pattern (LIFO)
LPUSH stack:undo "action1"
LPOP stack:undo

# Blocking pop (for workers)
BRPOP queue:tasks 30  # Wait 30 seconds

# Range operations
LRANGE recent:items 0 9  # Get first 10
LTRIM recent:items 0 99  # Keep only 100 items

# Insert at position
LINSERT list BEFORE "pivot" "new_value"
```

### Sets

```redis
# User tags
SADD user:1:tags "developer" "golang" "redis"
SMEMBERS user:1:tags
SISMEMBER user:1:tags "redis"

# Set operations
SINTER user:1:tags user:2:tags           # Common tags
SUNION user:1:tags user:2:tags           # All tags
SDIFF user:1:tags user:2:tags            # Tags in 1 but not 2

# Random member (for sampling)
SRANDMEMBER user:1:tags 3

# Move between sets
SMOVE set:pending set:processed "item1"
```

### Sorted Sets

```redis
# Leaderboard
ZADD leaderboard 1000 "player:1" 950 "player:2" 900 "player:3"
ZRANK leaderboard "player:1"              # Rank (0-indexed, ascending)
ZREVRANK leaderboard "player:1"           # Rank (descending)
ZSCORE leaderboard "player:1"             # Get score

# Range queries
ZRANGE leaderboard 0 9 WITHSCORES         # Top 10 (ascending)
ZREVRANGE leaderboard 0 9 WITHSCORES      # Top 10 (descending)
ZRANGEBYSCORE leaderboard 900 1000        # Score range

# Increment score
ZINCRBY leaderboard 50 "player:1"

# Time-based sorting (score = timestamp)
ZADD events 1705312800 "event:1"
ZRANGEBYSCORE events 1705312800 1705399200  # Events in time range

# Remove old entries
ZREMRANGEBYSCORE events -inf 1705226400     # Remove older than threshold
```

### Streams

```redis
# Add to stream
XADD events * type "click" user_id "123" page "/home"

# Read from stream
XREAD COUNT 10 STREAMS events 0            # From beginning
XREAD BLOCK 5000 STREAMS events $          # Block for new entries

# Consumer groups
XGROUP CREATE events mygroup $ MKSTREAM
XREADGROUP GROUP mygroup consumer1 COUNT 10 STREAMS events >

# Acknowledge processing
XACK events mygroup 1705312800000-0

# Pending entries (failed processing)
XPENDING events mygroup

# Claim abandoned messages
XCLAIM events mygroup consumer2 3600000 1705312800000-0
```

---

## Caching Strategies

### Cache-Aside Pattern

```python
# Python example
import redis
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def get_user(user_id):
    cache_key = f"user:{user_id}"

    # Try cache first
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    # Cache miss - fetch from database
    user = db.query(f"SELECT * FROM users WHERE id = {user_id}")

    # Store in cache with TTL
    r.setex(cache_key, 3600, json.dumps(user))

    return user

def update_user(user_id, data):
    # Update database
    db.execute(f"UPDATE users SET ... WHERE id = {user_id}")

    # Invalidate cache
    r.delete(f"user:{user_id}")
```

### Write-Through Pattern

```python
def update_user(user_id, data):
    cache_key = f"user:{user_id}"

    # Update database
    db.execute(f"UPDATE users SET ... WHERE id = {user_id}")

    # Update cache immediately
    user = db.query(f"SELECT * FROM users WHERE id = {user_id}")
    r.setex(cache_key, 3600, json.dumps(user))

    return user
```

### Cache Warming

```python
def warm_cache():
    # Pre-populate cache for hot data
    popular_users = db.query("SELECT id FROM users ORDER BY login_count DESC LIMIT 1000")

    pipe = r.pipeline()
    for user in popular_users:
        user_data = db.query(f"SELECT * FROM users WHERE id = {user['id']}")
        pipe.setex(f"user:{user['id']}", 3600, json.dumps(user_data))
    pipe.execute()
```

### Distributed Lock

```python
import time
import uuid

def acquire_lock(lock_name, timeout=10):
    lock_id = str(uuid.uuid4())
    lock_key = f"lock:{lock_name}"

    if r.set(lock_key, lock_id, nx=True, ex=timeout):
        return lock_id
    return None

def release_lock(lock_name, lock_id):
    lock_key = f"lock:{lock_name}"

    # Lua script for atomic check-and-delete
    script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """
    r.eval(script, 1, lock_key, lock_id)

# Usage
lock_id = acquire_lock("process:order:123")
if lock_id:
    try:
        # Do work
        process_order(123)
    finally:
        release_lock("process:order:123", lock_id)
```

### Rate Limiting

```python
def is_rate_limited(user_id, limit=100, window=60):
    key = f"rate:{user_id}"
    current = r.get(key)

    if current is None:
        r.setex(key, window, 1)
        return False

    if int(current) >= limit:
        return True

    r.incr(key)
    return False

# Sliding window rate limiting
def sliding_window_rate_limit(user_id, limit=100, window=60):
    key = f"rate:{user_id}"
    now = time.time()
    window_start = now - window

    pipe = r.pipeline()
    # Remove old entries
    pipe.zremrangebyscore(key, 0, window_start)
    # Add current request
    pipe.zadd(key, {str(now): now})
    # Count requests in window
    pipe.zcard(key)
    # Set TTL
    pipe.expire(key, window)

    results = pipe.execute()
    return results[2] > limit
```

---

## Persistence Configuration

### RDB Snapshots

```conf
# Save snapshot conditions
save 900 1      # After 900 seconds if 1 key changed
save 300 10     # After 300 seconds if 10 keys changed
save 60 10000   # After 60 seconds if 10000 keys changed

# Stop writes on bgsave error
stop-writes-on-bgsave-error yes

# Compress RDB files
rdbcompression yes
rdbchecksum yes

# RDB filename and directory
dbfilename dump.rdb
dir /var/lib/redis
```

### AOF (Append Only File)

```conf
# Enable AOF
appendonly yes
appendfilename "appendonly.aof"

# Sync frequency
appendfsync everysec  # Best balance (recommended)
# appendfsync always  # Safest but slower
# appendfsync no      # OS handles sync

# AOF rewrite settings
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# Use RDB preamble for faster loading
aof-use-rdb-preamble yes

# Don't sync during rewrite
no-appendfsync-on-rewrite no
```

### Hybrid Persistence

```conf
# Best of both worlds
appendonly yes
aof-use-rdb-preamble yes

# RDB for backups
save 900 1
save 300 10
save 60 10000
```

---

## High Availability

### Redis Sentinel

```conf
# sentinel.conf
port 26379
sentinel monitor mymaster 192.168.1.10 6379 2
sentinel auth-pass mymaster your-password
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```

```yaml
# docker-compose with Sentinel
services:
  redis-master:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-master-data:/data

  redis-replica:
    image: redis:7-alpine
    command: redis-server --appendonly yes --replicaof redis-master 6379 --masterauth ${REDIS_PASSWORD} --requirepass ${REDIS_PASSWORD}
    depends_on:
      - redis-master

  sentinel:
    image: redis:7-alpine
    command: redis-sentinel /etc/redis/sentinel.conf
    volumes:
      - ./sentinel.conf:/etc/redis/sentinel.conf
    depends_on:
      - redis-master
      - redis-replica
```

### Redis Cluster

```conf
# redis-cluster.conf
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

```bash
# Create cluster
redis-cli --cluster create \
    192.168.1.10:7000 \
    192.168.1.11:7000 \
    192.168.1.12:7000 \
    192.168.1.10:7001 \
    192.168.1.11:7001 \
    192.168.1.12:7001 \
    --cluster-replicas 1

# Check cluster status
redis-cli -c -h 192.168.1.10 -p 7000 cluster info
redis-cli -c -h 192.168.1.10 -p 7000 cluster nodes
```

---

## Security Hardening

### Authentication and ACLs

```conf
# Redis 6.0+ ACLs
requirepass default-password

# Define users with specific permissions
user default on >default-password ~* &* +@all

# Read-only user
user readonly on >readonly-pass ~* &* +@read

# Application user with limited commands
user app_user on >app-password ~app:* &* +@read +@write -@dangerous

# Admin user
user admin on >admin-password ~* &* +@all
```

```redis
# ACL commands
ACL LIST
ACL WHOAMI
ACL SETUSER newuser on >password ~keys:* +get +set
ACL DELUSER olduser
ACL CAT                   # List categories
ACL CAT dangerous         # List dangerous commands
```

### Network Security

```conf
# Bind to specific interfaces
bind 127.0.0.1 192.168.1.10

# Enable protected mode
protected-mode yes

# Rename dangerous commands
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command DEBUG ""
rename-command CONFIG "ADMIN_CONFIG"
rename-command SHUTDOWN "ADMIN_SHUTDOWN"
```

### TLS Configuration

```conf
# TLS/SSL
port 0
tls-port 6379
tls-cert-file /etc/redis/ssl/redis.crt
tls-key-file /etc/redis/ssl/redis.key
tls-ca-cert-file /etc/redis/ssl/ca.crt
tls-auth-clients yes
tls-protocols "TLSv1.2 TLSv1.3"
```

---

## Performance Optimization

### Memory Optimization

```conf
# Memory limit and eviction
maxmemory 4gb
maxmemory-policy allkeys-lru

# Eviction policies:
# volatile-lru    - LRU among keys with TTL
# allkeys-lru     - LRU among all keys
# volatile-lfu    - LFU among keys with TTL
# allkeys-lfu     - LFU among all keys
# volatile-random - Random among keys with TTL
# allkeys-random  - Random among all keys
# volatile-ttl    - Remove keys with shortest TTL
# noeviction      - Return error on write

# Memory sampling
maxmemory-samples 10

# Lazy freeing
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
```

### Connection Optimization

```conf
# Max clients
maxclients 10000

# TCP keepalive
tcp-keepalive 300

# Client output buffer limits
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
```

### Pipelining

```python
# Without pipelining - slow
for key in keys:
    r.get(key)  # Round trip for each

# With pipelining - fast
pipe = r.pipeline()
for key in keys:
    pipe.get(key)
results = pipe.execute()  # Single round trip

# With transactions
pipe = r.pipeline(transaction=True)
pipe.multi()
pipe.incr("counter")
pipe.expire("counter", 300)
pipe.execute()
```

---

## Monitoring and Alerting

### Key Metrics

```redis
# Server info
INFO

# Memory usage
INFO memory
MEMORY STATS
MEMORY DOCTOR

# Client connections
INFO clients
CLIENT LIST

# Slow log
SLOWLOG GET 10
SLOWLOG LEN
SLOWLOG RESET

# Command statistics
INFO commandstats

# Keyspace info
INFO keyspace
DBSIZE
```

### Prometheus Exporter

```yaml
# docker-compose with redis-exporter
services:
  redis-exporter:
    image: oliver006/redis_exporter:latest
    environment:
      REDIS_ADDR: redis://redis:6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
    ports:
      - "9121:9121"
```

### Alert Rules

```yaml
# prometheus/alerts/redis.yml
groups:
  - name: redis-alerts
    rules:
      - alert: RedisDown
        expr: redis_up == 0
        for: 1m
        labels:
          severity: critical

      - alert: RedisHighMemory
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.9
        for: 5m
        labels:
          severity: warning

      - alert: RedisTooManyConnections
        expr: redis_connected_clients > 9000
        for: 5m
        labels:
          severity: warning

      - alert: RedisReplicationBroken
        expr: redis_connected_slaves < 1
        for: 5m
        labels:
          severity: critical
```

---

## Quick Reference

### Common Commands

```redis
# Keys
KEYS pattern      # List keys (avoid in production)
SCAN 0 MATCH pattern COUNT 100  # Safe iteration
EXISTS key
DEL key
EXPIRE key seconds
TTL key
TYPE key

# Info
INFO
INFO memory
INFO replication
INFO clients

# Debug
MONITOR           # Watch all commands (careful!)
DEBUG SLEEP 0.1   # Block server
SLOWLOG GET 10    # Slow queries
CLIENT LIST       # Connected clients
```

### Best Practices

1. **Use appropriate data structures** - Hash for objects, not multiple strings
2. **Set TTL on keys** - Prevent memory growth
3. **Use pipelining** - Reduce round trips
4. **Avoid KEYS command** - Use SCAN instead
5. **Monitor memory** - Set maxmemory and eviction policy
6. **Enable persistence** - AOF with everysec for durability
7. **Use connection pooling** - Reuse connections
8. **Implement timeouts** - Prevent hanging connections
