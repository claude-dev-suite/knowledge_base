# Elasticsearch Vector Search Deployment Guide

## Overview

This guide covers deploying Elasticsearch for vector search workloads: Elastic Cloud, Docker, index mappings for vectors, shard sizing, force-merge optimization, monitoring, and security. The focus is on configuration patterns specific to kNN/dense_vector workloads, which have different resource requirements than traditional text search.

---

## Elastic Cloud (Managed)

Elastic Cloud is the fastest path to production Elasticsearch with vector search.

```python
from elasticsearch import Elasticsearch

# Connect to Elastic Cloud
es = Elasticsearch(
    cloud_id="my-deployment:dXMtZWFzdC0xLmF3cy5mb3VuZC5pbyQ...",
    api_key="your-api-key",
)

# Verify connection
print(es.info())
```

**Elastic Cloud tiers for vector workloads** (approximate as of 2025):

| Profile | vCPU | RAM | Storage | Monthly Cost | Max Vectors (1536d) |
|---------|------|-----|---------|-------------|-------------------|
| General Purpose (4 GB) | 2 | 4 GB | 120 GB | ~$100 | ~200K |
| General Purpose (16 GB) | 8 | 16 GB | 480 GB | ~$400 | ~1M |
| Vector Optimized (32 GB) | 16 | 32 GB | 960 GB | ~$750 | ~3M |
| Vector Optimized (64 GB) | 32 | 64 GB | 1920 GB | ~$1,500 | ~6M |

**Note**: vector workloads are RAM-intensive. The HNSW graph must fit in memory for optimal performance. Use the "Vector Optimized" hardware profile for large collections.

---

## Docker Deployment

### Single Node (Development)

```yaml
# docker-compose.yml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.15.3
    restart: always
    ports:
      - "9200:9200"
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=false
      - ELASTIC_PASSWORD=changeme
      - ES_JAVA_OPTS=-Xms4g -Xmx4g
      - xpack.ml.enabled=true
    volumes:
      - esdata:/usr/share/elasticsearch/data
    deploy:
      resources:
        limits:
          memory: 8G
    healthcheck:
      test: ["CMD", "curl", "-f", "-u", "elastic:changeme", "http://localhost:9200/_cluster/health"]
      interval: 30s
      timeout: 10s
      retries: 5
    ulimits:
      memlock:
        soft: -1
        hard: -1

volumes:
  esdata:
    driver: local
```

### Three-Node Cluster (Production)

```yaml
# docker-compose-cluster.yml
services:
  es-node-0:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.15.3
    restart: always
    ports:
      - "9200:9200"
    environment:
      - node.name=es-node-0
      - cluster.name=vector-cluster
      - discovery.seed_hosts=es-node-1,es-node-2
      - cluster.initial_master_nodes=es-node-0,es-node-1,es-node-2
      - xpack.security.enabled=true
      - ELASTIC_PASSWORD=changeme
      - ES_JAVA_OPTS=-Xms8g -Xmx8g
    volumes:
      - es_node0:/usr/share/elasticsearch/data
    networks:
      - es-net
    ulimits:
      memlock:
        soft: -1
        hard: -1

  es-node-1:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.15.3
    restart: always
    environment:
      - node.name=es-node-1
      - cluster.name=vector-cluster
      - discovery.seed_hosts=es-node-0,es-node-2
      - cluster.initial_master_nodes=es-node-0,es-node-1,es-node-2
      - xpack.security.enabled=true
      - ELASTIC_PASSWORD=changeme
      - ES_JAVA_OPTS=-Xms8g -Xmx8g
    volumes:
      - es_node1:/usr/share/elasticsearch/data
    networks:
      - es-net
    ulimits:
      memlock:
        soft: -1
        hard: -1

  es-node-2:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.15.3
    restart: always
    environment:
      - node.name=es-node-2
      - cluster.name=vector-cluster
      - discovery.seed_hosts=es-node-0,es-node-1
      - cluster.initial_master_nodes=es-node-0,es-node-1,es-node-2
      - xpack.security.enabled=true
      - ELASTIC_PASSWORD=changeme
      - ES_JAVA_OPTS=-Xms8g -Xmx8g
    volumes:
      - es_node2:/usr/share/elasticsearch/data
    networks:
      - es-net
    ulimits:
      memlock:
        soft: -1
        hard: -1

volumes:
  es_node0:
  es_node1:
  es_node2:

networks:
  es-net:
    driver: bridge
```

---

## Index Mappings for Vector Workloads

### Standard Vector Index

```json
PUT /documents
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "index.codec": "best_compression",
    "refresh_interval": "30s"
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "standard"
      },
      "content": {
        "type": "text",
        "analyzer": "standard"
      },
      "category": {
        "type": "keyword"
      },
      "year": {
        "type": "integer"
      },
      "created_at": {
        "type": "date"
      },
      "embedding": {
        "type": "dense_vector",
        "dims": 1536,
        "index": true,
        "similarity": "cosine",
        "index_options": {
          "type": "hnsw",
          "m": 16,
          "ef_construction": 128
        }
      }
    }
  }
}
```

### Quantized Vector Index (Memory Optimized)

```json
PUT /documents_quantized
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "title": { "type": "text" },
      "content": { "type": "text" },
      "embedding": {
        "type": "dense_vector",
        "dims": 1536,
        "index": true,
        "similarity": "cosine",
        "index_options": {
          "type": "int8_hnsw",
          "m": 16,
          "ef_construction": 128,
          "confidence_interval": 0.99
        }
      }
    }
  }
}
```

### Hybrid Index (BM25 + kNN + ELSER)

```json
PUT /documents_hybrid
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "title": { "type": "text" },
      "content": { "type": "text" },
      "category": { "type": "keyword" },
      "embedding": {
        "type": "dense_vector",
        "dims": 1536,
        "index": true,
        "similarity": "cosine",
        "index_options": {
          "type": "int8_hnsw",
          "m": 16,
          "ef_construction": 128
        }
      },
      "content_tokens": {
        "type": "sparse_vector"
      }
    }
  }
}
```

---

## Shard Sizing for Vector Workloads

### Shard Size Guidelines

Vector workloads have different shard sizing rules than text-only indexes because HNSW graphs are memory-intensive.

| Vectors per Shard | Dims | Index Type | Shard RAM | Recommendation |
|------------------|------|-----------|-----------|---------------|
| < 500K | 1536 | hnsw | < 4 GB | 1 shard |
| 500K - 2M | 1536 | hnsw | 4-15 GB | 1-2 shards |
| 2M - 5M | 1536 | hnsw | 15-35 GB | 2-4 shards |
| 5M - 10M | 1536 | int8_hnsw | 8-20 GB | 4-8 shards |
| > 10M | 1536 | int8_hnsw | 20+ GB | 8+ shards |

### Shard Allocation

```json
PUT /documents/_settings
{
  "index.routing.allocation.total_shards_per_node": 2,
  "index.max_result_window": 10000
}
```

### ILM for Vector Indexes

```json
PUT /_ilm/policy/vector-lifecycle
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_primary_shard_size": "30gb",
            "max_docs": 5000000
          },
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "allocate": {
            "number_of_replicas": 0
          },
          "shrink": {
            "number_of_shards": 1
          }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

---

## Force Merge for kNN Performance

Force merge is critical for vector search performance. After bulk indexing, run:

```python
# Step 1: Disable refresh during bulk indexing
es.indices.put_settings(
    index="documents",
    settings={"index.refresh_interval": "-1"},
)

# Step 2: Bulk index (see overview.md for bulk indexing patterns)

# Step 3: Re-enable refresh
es.indices.put_settings(
    index="documents",
    settings={"index.refresh_interval": "30s"},
)

# Step 4: Force refresh
es.indices.refresh(index="documents")

# Step 5: Force merge to single segment
es.indices.forcemerge(
    index="documents",
    max_num_segments=1,
    wait_for_completion=True,
    request_timeout=7200,
)
```

### Automated Post-Bulk Script

```python
def optimize_vector_index(es, index_name: str):
    """Run post-bulk optimization for vector indexes."""
    print(f"Refreshing {index_name}...")
    es.indices.refresh(index=index_name)

    # Get segment count
    stats = es.indices.segments(index=index_name)
    total_segments = sum(
        len(shard_info[0]["segments"])
        for shard_info in stats["indices"][index_name]["shards"].values()
    )
    print(f"Current segments: {total_segments}")

    if total_segments > 1:
        print(f"Force merging {index_name} to 1 segment per shard...")
        es.indices.forcemerge(
            index=index_name,
            max_num_segments=1,
            wait_for_completion=True,
            request_timeout=7200,
        )
        print("Force merge complete")

    # Verify
    stats = es.indices.segments(index=index_name)
    total_segments = sum(
        len(shard_info[0]["segments"])
        for shard_info in stats["indices"][index_name]["shards"].values()
    )
    print(f"Segments after merge: {total_segments}")
```

---

## Monitoring Vector Search Metrics

### Cluster Health

```python
def check_cluster_health(es):
    health = es.cluster.health()
    print(f"Status: {health['status']}")
    print(f"Nodes: {health['number_of_nodes']}")
    print(f"Active shards: {health['active_shards']}")
    print(f"Unassigned shards: {health['unassigned_shards']}")
    return health["status"] != "red"
```

### Vector Index Stats

```python
def get_vector_index_stats(es, index_name: str):
    """Get detailed stats for a vector index."""
    stats = es.indices.stats(index=index_name)
    index_stats = stats["indices"][index_name]["total"]

    # Index size
    store_size = index_stats["store"]["size_in_bytes"]

    # Document count
    doc_count = index_stats["docs"]["count"]

    # Search stats
    search_stats = index_stats["search"]
    query_total = search_stats["query_total"]
    query_time = search_stats["query_time_in_millis"]
    avg_query_time = query_time / max(query_total, 1)

    # Segment stats
    segments = index_stats["segments"]
    segment_count = segments["count"]
    segment_memory = segments["memory_in_bytes"]

    print(f"Index: {index_name}")
    print(f"  Documents: {doc_count:,}")
    print(f"  Size: {store_size / 1024**3:.2f} GB")
    print(f"  Segments: {segment_count}")
    print(f"  Segment memory: {segment_memory / 1024**2:.1f} MB")
    print(f"  Total queries: {query_total:,}")
    print(f"  Avg query time: {avg_query_time:.1f} ms")
```

### Key Metrics for Prometheus/Grafana

```yaml
# Elasticsearch Exporter metrics to watch:

# Search latency
elasticsearch_indices_search_query_time_seconds

# Search rate
rate(elasticsearch_indices_search_query_total[1m])

# JVM heap usage (critical for vector workloads)
elasticsearch_jvm_memory_used_bytes{area="heap"}

# Segment memory (HNSW graphs live here)
elasticsearch_indices_segments_memory_bytes

# Circuit breaker trips (OOM protection)
elasticsearch_breaker_tripped_total

# Merge activity (high during ingestion)
elasticsearch_indices_merges_current
```

### Alerting Rules

```yaml
groups:
  - name: elasticsearch-vector
    rules:
      - alert: ESHighSearchLatency
        expr: rate(elasticsearch_indices_search_query_time_seconds_total[5m]) / rate(elasticsearch_indices_search_query_total[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "ES avg search latency > 100ms"

      - alert: ESHighHeapUsage
        expr: elasticsearch_jvm_memory_used_bytes{area="heap"} / elasticsearch_jvm_memory_max_bytes{area="heap"} > 0.85
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "ES JVM heap > 85%"

      - alert: ESClusterRed
        expr: elasticsearch_cluster_health_status{color="red"} == 1
        for: 1m
        labels:
          severity: critical

      - alert: ESUnassignedShards
        expr: elasticsearch_cluster_health_unassigned_shards > 0
        for: 10m
        labels:
          severity: warning
```

---

## Security Configuration

### TLS Setup

```yaml
# elasticsearch.yml
xpack.security.enabled: true

xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12

xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: /usr/share/elasticsearch/config/certs/http.p12
```

### API Key Authentication

```python
# Create an API key
api_key_response = es.security.create_api_key(
    name="vector-search-service",
    role_descriptors={
        "vector-reader": {
            "cluster": ["monitor"],
            "indices": [
                {
                    "names": ["documents*"],
                    "privileges": ["read", "view_index_metadata"],
                }
            ],
        }
    },
    expiration="30d",
)

api_key_id = api_key_response["id"]
api_key_encoded = api_key_response["encoded"]

# Use the API key
es_reader = Elasticsearch(
    "https://localhost:9200",
    api_key=api_key_encoded,
    ca_certs="/path/to/ca.crt",
)
```

### Role-Based Access

```json
POST /_security/role/vector_reader
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["documents*"],
      "privileges": ["read"],
      "field_security": {
        "grant": ["title", "content", "category", "year"]
      }
    }
  ]
}

POST /_security/role/vector_writer
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["documents*"],
      "privileges": ["read", "write", "create_index", "manage"]
    }
  ]
}
```

---

## Resource Sizing for Vector Workloads

### Memory Formula

```
Heap memory per shard (float32 HNSW):
  vectors * dims * 4 bytes (vector data)
  + vectors * m * 2 * 4 bytes (HNSW graph)
  + ~20% overhead

Heap memory per shard (int8_hnsw):
  vectors * dims * 1 byte (quantized data)
  + vectors * dims * 4 bytes (full vectors for rescoring, on disk)
  + vectors * m * 2 * 4 bytes (HNSW graph)
  + ~20% overhead
```

### Sizing Table

| Vectors | Dims | Index Type | Heap per Shard | Recommended JVM Heap |
|---------|------|-----------|---------------|---------------------|
| 100K | 1536 | hnsw | ~0.8 GB | 2 GB |
| 500K | 1536 | hnsw | ~4 GB | 8 GB |
| 1M | 1536 | hnsw | ~8 GB | 16 GB |
| 1M | 1536 | int8_hnsw | ~2.5 GB | 8 GB |
| 5M | 1536 | int8_hnsw | ~12 GB | 24 GB |
| 10M | 1536 | int8_hnsw | ~24 GB | 48 GB (multi-shard) |

**Critical**: never set JVM heap above 30.5 GB (compressed oops threshold). For larger workloads, add more shards across more nodes rather than increasing heap per node.

### JVM Settings

```bash
# elasticsearch.yml or ES_JAVA_OPTS environment variable
ES_JAVA_OPTS=-Xms16g -Xmx16g

# Always set min = max heap to avoid resizing
# Vector workloads benefit from larger heap for HNSW graphs
```

---

## Common Pitfalls

1. **Not force-merging after bulk index**: multiple Lucene segments mean multiple independent HNSW graphs. This is the single biggest performance mistake for ES vector search.

2. **Setting JVM heap too small for vector indexes**: HNSW graphs consume significant heap. Monitor `elasticsearch_jvm_memory_used_bytes` and increase heap if it consistently exceeds 75%.

3. **Too many shards for small datasets**: each shard has its own HNSW graph with independent overhead. For < 1M vectors, use 1 shard.

4. **Not using int8_hnsw for cost optimization**: int8 quantization reduces heap by ~4x with < 1% recall loss. Use it for any collection over 500K vectors.

5. **Forgetting to disable refresh during bulk indexing**: each refresh creates a new segment, fragmenting the HNSW graph. Set `refresh_interval: -1` during bulk loads.

6. **Running ELSER without dedicated ML nodes**: ELSER inference competes with search for CPU/RAM. Deploy ELSER on dedicated ML nodes to avoid impacting search latency.

7. **Ignoring `num_candidates` tuning**: the default `num_candidates=100` is often insufficient for filtered queries. Increase to 200-500 when using restrictive pre-filters.

---

## References

- Elasticsearch vector search tuning: https://www.elastic.co/guide/en/elasticsearch/reference/current/tune-knn-search.html
- Elasticsearch deployment guide: https://www.elastic.co/guide/en/elasticsearch/reference/current/setup.html
- Elasticsearch security: https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-cluster.html
- Elastic Cloud: https://cloud.elastic.co/
- Elasticsearch Docker: https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html
