# Redis Vector Search Deployment Guide

## Overview

This guide covers deploying Redis for vector search workloads: Redis Stack Docker, Redis Enterprise cluster, Redis Cloud, index creation commands, RediSearch module requirements, memory sizing, persistence configuration, and monitoring. The focus is on operational patterns for reliability and performance.

---

## Redis Stack Docker

### Quick Start

```bash
docker run -d \
  --name redis-stack \
  -p 6379:6379 \
  -p 8001:8001 \
  -v redis_data:/data \
  redis/redis-stack:7.4.2-v0
```

Port 6379 is the Redis server; port 8001 is the RedisInsight web UI.

### Production Docker Compose

```yaml
# docker-compose.yml
services:
  redis-stack:
    image: redis/redis-stack-server:7.4.2-v0
    restart: always
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
      - ./redis.conf:/redis-stack.conf
    environment:
      - REDIS_ARGS=--requirepass your-secure-password --maxmemory 12gb --maxmemory-policy noeviction
    deploy:
      resources:
        limits:
          memory: 16G
        reservations:
          memory: 8G
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "your-secure-password", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  redisinsight:
    image: redis/redisinsight:2.62
    restart: always
    ports:
      - "8001:5540"
    depends_on:
      - redis-stack

volumes:
  redis_data:
    driver: local
```

### Redis Configuration for Vector Workloads

```conf
# redis.conf

# Memory
maxmemory 12gb
maxmemory-policy noeviction    # CRITICAL: do not evict vector data

# Persistence (RDB snapshots)
save 900 1
save 300 10
save 60 10000

# Persistence (AOF for durability)
appendonly yes
appendfsync everysec

# Network
bind 0.0.0.0
protected-mode yes
requirepass your-secure-password

# Performance
io-threads 4
io-threads-do-reads yes

# RediSearch specific
# These are module-level settings
loadmodule /opt/redis-stack/lib/redisearch.so
  MAXDOCTABLESIZE 10000000
  FRISOINI /opt/redis-stack/etc/friso/friso.ini
```

---

## Redis Enterprise Cluster

Redis Enterprise provides clustering, high availability, and active-active replication for Redis.

### Cluster Architecture

```
                     +-- [Shard 1] (primary)
                     |     +-- [Shard 1] (replica)
Client --> [Proxy] --+
                     +-- [Shard 2] (primary)
                     |     +-- [Shard 2] (replica)
                     +-- [Shard 3] (primary)
                           +-- [Shard 3] (replica)
```

### Docker Compose (3-Node Enterprise Cluster)

```yaml
# docker-compose-enterprise.yml
services:
  redis-node-1:
    image: redislabs/redis:7.4.2-130
    restart: always
    ports:
      - "8443:8443"    # Admin UI
      - "9443:9443"    # REST API
      - "12000:12000"  # Database port
    cap_add:
      - sys_resource
    volumes:
      - node1_data:/var/opt/redislabs/persist
      - node1_log:/var/opt/redislabs/log

  redis-node-2:
    image: redislabs/redis:7.4.2-130
    restart: always
    cap_add:
      - sys_resource
    volumes:
      - node2_data:/var/opt/redislabs/persist
    depends_on:
      - redis-node-1

  redis-node-3:
    image: redislabs/redis:7.4.2-130
    restart: always
    cap_add:
      - sys_resource
    volumes:
      - node3_data:/var/opt/redislabs/persist
    depends_on:
      - redis-node-1

volumes:
  node1_data:
  node1_log:
  node2_data:
  node3_data:
```

**Note**: Redis Enterprise requires a license for production use. The Docker image is for evaluation.

---

## Redis Cloud

Redis Cloud is the fully managed Redis service with built-in RediSearch support.

```python
import redis

# Connect to Redis Cloud
r = redis.Redis(
    host="redis-12345.c1.us-east-1-1.ec2.cloud.redislabs.com",
    port=12345,
    password="your-cloud-password",
    ssl=True,
)

# Verify connection and modules
print(r.ping())
modules = r.module_list()
for mod in modules:
    print(f"Module: {mod[b'name'].decode()}, Version: {mod[b'ver']}")
```

**Redis Cloud tiers** (approximate as of 2025):

| Plan | RAM | Modules | Price/Month |
|------|-----|---------|-------------|
| Free | 30 MB | RediSearch, JSON | $0 |
| Essentials | 250 MB - 12 GB | RediSearch, JSON | $5 - $95 |
| Pro | 1 GB - 1 TB | All modules, HA | $85 - $10,000+ |

---

## Index Creation Commands

### Hash-Based Index

```bash
# Full-featured index for vector + hybrid search
FT.CREATE idx:documents
  ON HASH
  PREFIX 1 "doc:"
  SCHEMA
    title TEXT WEIGHT 2.0 SORTABLE
    content TEXT
    category TAG SEPARATOR "," SORTABLE
    year NUMERIC SORTABLE
    tags TAG SEPARATOR "|"
    embedding VECTOR HNSW 10
      TYPE FLOAT32
      DIM 1536
      DISTANCE_METRIC COSINE
      M 16
      EF_CONSTRUCTION 200
      EF_RUNTIME 100
      INITIAL_CAP 500000
```

### JSON-Based Index

```bash
FT.CREATE idx:json_docs
  ON JSON
  PREFIX 1 "jdoc:"
  SCHEMA
    $.title AS title TEXT WEIGHT 2.0
    $.content AS content TEXT
    $.metadata.category AS category TAG
    $.metadata.year AS year NUMERIC
    $.embedding AS embedding VECTOR HNSW 10
      TYPE FLOAT32
      DIM 1536
      DISTANCE_METRIC COSINE
      M 16
      EF_CONSTRUCTION 200
      EF_RUNTIME 100
```

### Index Management Commands

```bash
# List all indexes
FT._LIST

# Get index info
FT.INFO idx:documents

# Drop index (keeps data)
FT.DROPINDEX idx:documents

# Drop index and delete all indexed documents
FT.DROPINDEX idx:documents DD

# Alter index (add fields -- cannot remove or change existing)
FT.ALTER idx:documents SCHEMA ADD new_field TAG
```

---

## RediSearch Module Requirements

### Version Compatibility

| Feature | Minimum RediSearch Version |
|---------|--------------------------|
| FLAT vector index | 2.4.0 |
| HNSW vector index | 2.4.0 |
| Vector range queries | 2.6.0 |
| INT8 vector type | 2.8.0 |
| FLOAT16 vector type | 2.8.0 |
| JSON vector fields | 2.6.0 |
| Query dialect 2 | 2.4.0 |

### Verifying Module Installation

```python
import redis

r = redis.Redis(host="localhost", port=6379)

# Check RediSearch version
info = r.ft("idx:documents").info()
print(f"RediSearch version: {info.get('search_dialect_config', 'N/A')}")

# Check all modules
for mod in r.module_list():
    name = mod[b"name"].decode()
    version = mod[b"ver"]
    print(f"{name}: {version}")
```

---

## Memory Sizing

### Memory Estimation Formula

```
Total RAM = Vector Storage + Index Overhead + Hash/JSON Overhead + Redis Overhead

Vector Storage:
  FLOAT32: num_vectors * dimensions * 4 bytes
  FLOAT16: num_vectors * dimensions * 2 bytes

HNSW Index Overhead:
  num_vectors * M * 2 * 8 bytes (neighbor lists)
  + num_vectors * 32 bytes (metadata)

FLAT Index Overhead:
  Minimal (just the vectors)

Hash/JSON Overhead:
  num_vectors * avg_payload_bytes
  + num_vectors * 80 bytes (Redis key overhead)

Inverted Index (TAG/TEXT/NUMERIC fields):
  ~10-30% of payload data

Redis Overhead:
  ~15-20% for memory allocator fragmentation
```

### Sizing Table

| Vectors | Dims | Index | Payload (avg) | Estimated RAM | Recommended |
|---------|------|-------|-------------|-------------|-------------|
| 100K | 1536 | HNSW | 256 B | ~1.2 GB | 2 GB |
| 500K | 1536 | HNSW | 256 B | ~5.5 GB | 8 GB |
| 1M | 1536 | HNSW | 256 B | ~11 GB | 16 GB |
| 1M | 768 | HNSW | 256 B | ~6 GB | 8 GB |
| 5M | 1536 | HNSW | 256 B | ~55 GB | 64 GB |
| 1M | 1536 | FLAT | 256 B | ~8 GB | 12 GB |

### Memory Monitoring

```python
def check_redis_memory(r):
    """Monitor Redis memory usage."""
    info = r.info("memory")
    used = info["used_memory_human"]
    peak = info["used_memory_peak_human"]
    rss = info["used_memory_rss_human"]
    frag_ratio = info["mem_fragmentation_ratio"]

    print(f"Used memory: {used}")
    print(f"Peak memory: {peak}")
    print(f"RSS: {rss}")
    print(f"Fragmentation ratio: {frag_ratio:.2f}")

    if frag_ratio > 1.5:
        print("WARNING: High memory fragmentation (>1.5x)")

    # Index-specific memory
    try:
        idx_info = r.ft("idx:documents").info()
        num_docs = idx_info.get("num_docs", 0)
        idx_size = idx_info.get("inverted_sz_mb", 0)
        vector_idx_size = idx_info.get("vector_index_sz_mb", 0)
        print(f"Indexed documents: {num_docs}")
        print(f"Inverted index size: {idx_size} MB")
        print(f"Vector index size: {vector_idx_size} MB")
    except Exception:
        pass
```

---

## Persistence Configuration

### RDB Snapshots

```conf
# redis.conf
save 900 1        # Save every 15 min if 1+ keys changed
save 300 10       # Save every 5 min if 10+ keys changed
save 60 10000     # Save every 1 min if 10000+ keys changed

rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /data
```

### AOF (Append-Only File)

```conf
# redis.conf
appendonly yes
appendfsync everysec      # fsync every second (good balance)
# appendfsync always      # fsync on every write (safest, slowest)

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

aof-use-rdb-preamble yes  # hybrid AOF: RDB for bulk + AOF for recent
```

### Persistence Recommendations

| Requirement | Configuration |
|-------------|---------------|
| Maximum durability | AOF with `appendfsync always` |
| Good durability + performance | AOF with `appendfsync everysec` (recommended) |
| Fastest restarts | RDB only (risk: up to 15 min data loss) |
| Best of both | RDB + AOF with RDB preamble |
| Maximum performance (no persistence) | `save ""` + `appendonly no` (not for production) |

### Backup Strategy

```python
import subprocess
import shutil
from datetime import datetime

def backup_redis(
    redis_host: str = "localhost",
    redis_port: int = 6379,
    redis_password: str = None,
    backup_dir: str = "/backups/redis",
):
    """Trigger RDB save and copy the dump file."""
    import redis

    r = redis.Redis(host=redis_host, port=redis_port, password=redis_password)

    # Trigger background save
    r.bgsave()

    # Wait for save to complete
    import time
    while r.lastsave() == r.lastsave():
        time.sleep(1)

    # Copy dump file
    timestamp = datetime.now().strftime("%Y%m%d-%H%M%S")
    src = "/data/dump.rdb"
    dst = f"{backup_dir}/dump-{timestamp}.rdb"
    shutil.copy2(src, dst)
    print(f"Backup saved: {dst}")
```

---

## Monitoring

### Redis INFO Command

```bash
# Overall info
redis-cli -a your-password INFO

# Memory-specific
redis-cli -a your-password INFO memory

# Clients
redis-cli -a your-password INFO clients

# Stats
redis-cli -a your-password INFO stats
```

### Key Metrics for Monitoring

```python
def collect_redis_metrics(r):
    """Collect key metrics for monitoring/alerting."""
    info = r.info()

    metrics = {
        # Memory
        "used_memory_bytes": info["used_memory"],
        "used_memory_peak_bytes": info["used_memory_peak"],
        "mem_fragmentation_ratio": info["mem_fragmentation_ratio"],

        # Clients
        "connected_clients": info["connected_clients"],
        "blocked_clients": info["blocked_clients"],

        # Performance
        "ops_per_sec": info["instantaneous_ops_per_sec"],
        "hit_rate": info["keyspace_hits"] / max(info["keyspace_hits"] + info["keyspace_misses"], 1),

        # Persistence
        "rdb_last_save_time": info["rdb_last_save_time"],
        "rdb_changes_since_last_save": info["rdb_changes_since_last_save"],

        # Replication
        "connected_slaves": info.get("connected_slaves", 0),
    }

    return metrics
```

### Prometheus Exporter

```yaml
# docker-compose addition for Redis exporter
services:
  redis-exporter:
    image: oliver006/redis_exporter:v1.66.0
    ports:
      - "9121:9121"
    environment:
      REDIS_ADDR: redis-stack:6379
      REDIS_PASSWORD: your-secure-password
    depends_on:
      - redis-stack
```

### Alerting Rules

```yaml
groups:
  - name: redis-vector
    rules:
      - alert: RedisHighMemory
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.85
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Redis memory > 85% of maxmemory"

      - alert: RedisHighFragmentation
        expr: redis_mem_fragmentation_ratio > 1.5
        for: 15m
        labels:
          severity: warning

      - alert: RedisHighLatency
        expr: redis_commands_duration_seconds_total{cmd="ft.search"} / redis_commands_processed_total{cmd="ft.search"} > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis FT.SEARCH avg latency > 50ms"

      - alert: RedisRDBFailed
        expr: redis_rdb_last_bgsave_status == 0
        for: 1m
        labels:
          severity: critical
```

### Health Check Script

```python
import redis
import sys

def check_redis_health(host="localhost", port=6379, password=None):
    """Production health check for Redis vector search."""
    try:
        r = redis.Redis(host=host, port=port, password=password)
        r.ping()
    except Exception as e:
        print(f"CRITICAL: cannot connect: {e}")
        return False

    # Check memory
    info = r.info("memory")
    used_mb = info["used_memory"] / 1024 / 1024
    maxmemory = info.get("maxmemory", 0)
    if maxmemory > 0:
        usage_pct = info["used_memory"] / maxmemory * 100
        print(f"Memory: {used_mb:.0f} MB ({usage_pct:.1f}%)")
        if usage_pct > 90:
            print("WARNING: Memory usage > 90%")
    else:
        print(f"Memory: {used_mb:.0f} MB (no maxmemory set)")

    # Check modules
    modules = r.module_list()
    has_search = any(mod[b"name"] == b"search" for mod in modules)
    if not has_search:
        print("CRITICAL: RediSearch module not loaded")
        return False

    # Check indexes
    try:
        indexes = r.execute_command("FT._LIST")
        for idx_name in indexes:
            name = idx_name.decode() if isinstance(idx_name, bytes) else idx_name
            info = r.ft(name).info()
            num_docs = info.get("num_docs", 0)
            print(f"Index '{name}': {num_docs} documents")
    except Exception as e:
        print(f"WARNING: could not list indexes: {e}")

    return True

if __name__ == "__main__":
    ok = check_redis_health(password="your-password")
    sys.exit(0 if ok else 1)
```

---

## Common Pitfalls

1. **Using `maxmemory-policy allkeys-lru` with vector data**: LRU eviction will randomly delete vector keys, corrupting your search index. Always use `noeviction` for vector workloads.

2. **Not enabling persistence**: Redis is in-memory by default. Without RDB or AOF, all data is lost on restart.

3. **Forgetting to set a password**: Redis has no authentication by default. Any client on the network can read, write, and flush all data.

4. **Running standard Redis without RediSearch**: vector search requires the RediSearch module. Use Redis Stack or install the module separately.

5. **Not monitoring memory fragmentation**: high fragmentation (>1.5x) wastes RAM. Restart Redis with `MEMORY PURGE` or schedule periodic restarts to defragment.

6. **Using AOF `always` sync without understanding the performance cost**: `appendfsync always` fsyncs on every write, which can reduce write throughput by 10x. Use `everysec` for most workloads.

7. **Not planning for replica memory**: Redis replicas need the same amount of RAM as the primary. Factor in 2x memory for primary + replica setups.

---

## References

- Redis Stack: https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack/
- Redis vector search: https://redis.io/docs/latest/develop/interact/search-and-query/query/vector-search/
- Redis Enterprise: https://redis.io/docs/latest/operate/rs/
- Redis Cloud: https://redis.io/cloud/
- Redis persistence: https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- RedisVL: https://github.com/redis/redis-vl-python
- Redis Exporter: https://github.com/oliver006/redis_exporter
