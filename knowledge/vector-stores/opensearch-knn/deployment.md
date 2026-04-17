# OpenSearch k-NN -- Deployment and Operations

## Overview

This guide covers deploying OpenSearch with the k-NN plugin for vector search: Docker single-node and multi-node setups, Amazon OpenSearch Service (managed), index mapping configuration, shard and replica sizing, engine selection guidance, memory estimation, and security configuration with IAM and fine-grained access control.

---

## Docker Deployment

### Single Node (Development)

```yaml
# docker-compose.yml
version: "3.8"

services:
  opensearch:
    image: opensearchproject/opensearch:2.13.0
    container_name: opensearch
    environment:
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=Admin_12345!
      - "OPENSEARCH_JAVA_OPTS=-Xms4g -Xmx4g"
      - plugins.security.disabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    ports:
      - "9200:9200"
      - "9600:9600"
    volumes:
      - opensearch_data:/usr/share/opensearch/data

  opensearch-dashboards:
    image: opensearchproject/opensearch-dashboards:2.13.0
    container_name: opensearch-dashboards
    ports:
      - "5601:5601"
    environment:
      - OPENSEARCH_HOSTS=["https://opensearch:9200"]
    depends_on:
      - opensearch

volumes:
  opensearch_data:
    driver: local
```

```bash
# Start
docker compose up -d

# Verify k-NN plugin is loaded
curl -k -u admin:Admin_12345! \
    https://localhost:9200/_cat/plugins?v | grep knn
```

### Multi-Node Cluster (Production-Like)

```yaml
# docker-compose-cluster.yml
version: "3.8"

services:
  opensearch-node1:
    image: opensearchproject/opensearch:2.13.0
    container_name: opensearch-node1
    environment:
      - cluster.name=vector-cluster
      - node.name=opensearch-node1
      - node.roles=cluster_manager,data,ingest
      - discovery.seed_hosts=opensearch-node1,opensearch-node2,opensearch-node3
      - cluster.initial_cluster_manager_nodes=opensearch-node1
      - bootstrap.memory_lock=true
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=Admin_12345!
      - "OPENSEARCH_JAVA_OPTS=-Xms8g -Xmx8g"
      - knn.memory.circuit_breaker.limit=60%
      - knn.algo_param.index_thread_qty=4
    ulimits:
      memlock:
        soft: -1
        hard: -1
    ports:
      - "9200:9200"
    volumes:
      - node1_data:/usr/share/opensearch/data

  opensearch-node2:
    image: opensearchproject/opensearch:2.13.0
    container_name: opensearch-node2
    environment:
      - cluster.name=vector-cluster
      - node.name=opensearch-node2
      - node.roles=data,ingest
      - discovery.seed_hosts=opensearch-node1,opensearch-node2,opensearch-node3
      - cluster.initial_cluster_manager_nodes=opensearch-node1
      - bootstrap.memory_lock=true
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=Admin_12345!
      - "OPENSEARCH_JAVA_OPTS=-Xms8g -Xmx8g"
      - knn.memory.circuit_breaker.limit=60%
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - node2_data:/usr/share/opensearch/data

  opensearch-node3:
    image: opensearchproject/opensearch:2.13.0
    container_name: opensearch-node3
    environment:
      - cluster.name=vector-cluster
      - node.name=opensearch-node3
      - node.roles=data,ingest
      - discovery.seed_hosts=opensearch-node1,opensearch-node2,opensearch-node3
      - cluster.initial_cluster_manager_nodes=opensearch-node1
      - bootstrap.memory_lock=true
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=Admin_12345!
      - "OPENSEARCH_JAVA_OPTS=-Xms8g -Xmx8g"
      - knn.memory.circuit_breaker.limit=60%
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - node3_data:/usr/share/opensearch/data

volumes:
  node1_data:
  node2_data:
  node3_data:
```

---

## Amazon OpenSearch Service (Managed)

### Terraform Deployment

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_opensearch_domain" "vector_search" {
  domain_name    = "vector-search"
  engine_version = "OpenSearch_2.13"

  cluster_config {
    instance_type            = "r6g.2xlarge.search"
    instance_count           = 3
    zone_awareness_enabled   = true
    zone_awareness_config {
      availability_zone_count = 3
    }

    # Dedicated master nodes for cluster stability
    dedicated_master_enabled = true
    dedicated_master_type    = "m6g.large.search"
    dedicated_master_count   = 3
  }

  ebs_options {
    ebs_enabled = true
    volume_size = 500       # GB per node
    volume_type = "gp3"
    iops        = 3000
    throughput  = 250       # MB/s
  }

  encrypt_at_rest {
    enabled = true
  }

  node_to_node_encryption {
    enabled = true
  }

  domain_endpoint_options {
    enforce_https       = true
    tls_security_policy = "Policy-Min-TLS-1-2-2019-07"
  }

  advanced_security_options {
    enabled                        = true
    internal_user_database_enabled = true
    master_user_options {
      master_user_name     = "admin"
      master_user_password = var.master_password
    }
  }

  vpc_options {
    subnet_ids         = var.private_subnet_ids
    security_group_ids = [aws_security_group.opensearch.id]
  }

  # k-NN specific settings
  advanced_options = {
    "knn.memory.circuit_breaker.limit" = "60"
  }

  tags = {
    Environment = "production"
    Service     = "vector-search"
  }
}

resource "aws_security_group" "opensearch" {
  name_prefix = "opensearch-"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [var.app_security_group_id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

output "domain_endpoint" {
  value = aws_opensearch_domain.vector_search.endpoint
}
```

### Instance Sizing for k-NN

| Vectors | Dimensions | Engine | Recommended Instance | Nodes | Estimated Cost/mo |
|---|---|---|---|---|---|
| < 500K | 1536 | Lucene | r6g.large (16 GB) | 2 | $400 |
| 500K - 2M | 1536 | Lucene | r6g.xlarge (32 GB) | 3 | $900 |
| 2M - 10M | 1536 | Lucene | r6g.2xlarge (64 GB) | 3 | $1,800 |
| 10M - 50M | 1536 | Faiss + PQ | r6g.4xlarge (128 GB) | 3 | $3,600 |
| 50M - 200M | 1536 | Faiss + PQ | r6g.8xlarge (256 GB) | 6 | $14,400 |

---

## Index Mapping Configuration

### Production Index Template

```python
from opensearchpy import OpenSearch

client = OpenSearch(
    hosts=[{"host": "localhost", "port": 9200}],
    http_auth=("admin", "Admin_12345!"),
    use_ssl=True,
    verify_certs=False
)

# Create index with production settings
index_body = {
    "settings": {
        "index": {
            "knn": True,
            "knn.algo_param.ef_search": 100,
            "number_of_shards": 3,
            "number_of_replicas": 1,
            "refresh_interval": "30s",  # Reduce for bulk indexing
            "search.concurrent_segment_search.enabled": True
        }
    },
    "mappings": {
        "properties": {
            "embedding": {
                "type": "knn_vector",
                "dimension": 1536,
                "method": {
                    "engine": "lucene",
                    "space_type": "cosinesimil",
                    "name": "hnsw",
                    "parameters": {
                        "m": 24,
                        "ef_construction": 200
                    }
                }
            },
            "content": {
                "type": "text",
                "analyzer": "standard"
            },
            "title": {
                "type": "text",
                "analyzer": "standard"
            },
            "metadata": {
                "type": "object",
                "properties": {
                    "tenant_id": {"type": "keyword"},
                    "category": {"type": "keyword"},
                    "source": {"type": "keyword"},
                    "language": {"type": "keyword"},
                    "created_at": {"type": "date"},
                    "updated_at": {"type": "date"}
                }
            },
            "chunk_index": {"type": "integer"}
        }
    }
}

client.indices.create(index="documents", body=index_body)
```

### Byte Vector Index (int8, 4x Memory Savings)

```python
byte_index_body = {
    "settings": {
        "index": {
            "knn": True,
            "number_of_shards": 3,
            "number_of_replicas": 1
        }
    },
    "mappings": {
        "properties": {
            "embedding": {
                "type": "knn_vector",
                "dimension": 1536,
                "data_type": "byte",
                "method": {
                    "engine": "lucene",
                    "space_type": "cosinesimil",
                    "name": "hnsw",
                    "parameters": {
                        "m": 24,
                        "ef_construction": 200
                    }
                }
            },
            "content": {"type": "text"},
            "metadata": {
                "properties": {
                    "tenant_id": {"type": "keyword"}
                }
            }
        }
    }
}

client.indices.create(index="byte_documents", body=byte_index_body)
```

### Faiss Engine with Product Quantization

```python
faiss_pq_index = {
    "settings": {
        "index": {
            "knn": True,
            "knn.algo_param.ef_search": 128,
            "number_of_shards": 6,
            "number_of_replicas": 1
        }
    },
    "mappings": {
        "properties": {
            "embedding": {
                "type": "knn_vector",
                "dimension": 1536,
                "method": {
                    "engine": "faiss",
                    "space_type": "innerproduct",
                    "name": "hnsw",
                    "parameters": {
                        "m": 32,
                        "ef_construction": 256,
                        "encoder": {
                            "name": "pq",
                            "parameters": {
                                "code_size": 48,
                                "m": 48
                            }
                        }
                    }
                }
            }
        }
    }
}

client.indices.create(index="large_documents", body=faiss_pq_index)
```

---

## Shard and Replica Sizing

### Shard Sizing Guidelines

| Total Vectors | Shard Size Target | Shards | Vectors per Shard |
|---|---|---|---|
| < 1M | Single shard OK | 1 | < 1M |
| 1M - 5M | 1-2M per shard | 3 | 333K - 1.7M |
| 5M - 20M | 2-5M per shard | 6 | 833K - 3.3M |
| 20M - 100M | 5-10M per shard | 12 | 1.7M - 8.3M |
| 100M+ | 10M per shard | 20+ | ~5M |

### Replica Configuration

```python
# Set replicas based on read throughput needs
# More replicas = more read throughput, more memory

# Development: 0 replicas (saves memory)
client.indices.put_settings(
    index="documents",
    body={"index": {"number_of_replicas": 0}}
)

# Production: 1-2 replicas (availability + read throughput)
client.indices.put_settings(
    index="documents",
    body={"index": {"number_of_replicas": 1}}
)
```

**Memory impact of replicas**: each replica holds a complete copy of the k-NN index in memory. With 10M vectors (Lucene engine, float32, 1536 dims):

| Replicas | Primary Memory | Total Memory (3 shards) |
|---|---|---|
| 0 | 6.3 GB per shard | 18.9 GB |
| 1 | 6.3 GB per shard | 37.8 GB |
| 2 | 6.3 GB per shard | 56.7 GB |

---

## Engine Selection Guide

### Decision Tree

```
Start
  |
  +-- Need filtered search?
  |     YES --> Lucene (best filter integration)
  |
  +-- Need product quantization?
  |     YES --> Faiss (only engine with PQ)
  |
  +-- Dataset > 50M vectors?
  |     YES --> Faiss with PQ
  |
  +-- Need byte vectors (int8)?
  |     YES --> Lucene (2.13+)
  |
  +-- Default choice?
        --> Lucene (default since 2.13, best all-around)
```

### Engine Comparison Table

| Feature | Lucene | Faiss | nmslib |
|---|---|---|---|
| Status | Default, recommended | Advanced use | Legacy, avoid |
| HNSW | Yes | Yes | Yes |
| IVF | No | Yes | No |
| Product Quantization | No | Yes | No |
| Byte vectors (int8) | Yes (2.13+) | Yes | No |
| Memory location | JVM heap | Native (off-heap) | Native (off-heap) |
| Filter integration | Deep (during traversal) | Post-filter | Post-filter |
| Segment merging | Automatic with Lucene | Automatic | Manual |
| Concurrent segment search | Yes (2.12+) | Yes | Yes |
| Warmup on shard open | Automatic | Requires config | Requires config |

---

## Memory Estimation Formula

### Lucene Engine

```
Total Memory = N * (D * bytes_per_dim + M * 2 * 4 + 48) + JVM_overhead

Where:
  N = number of vectors
  D = dimensions
  bytes_per_dim = 4 (float32) or 1 (byte/int8)
  M = HNSW max connections per node
  48 = per-vector metadata overhead (approx)
  JVM_overhead = ~30% of index size
```

### Faiss Engine (Native Memory)

```
HNSW Memory = N * (D * bytes_per_dim + M * 2 * 4)
HNSW+PQ Memory = N * (PQ_m * 1 + M * 2 * 4)  # PQ codes are 1 byte each

Where:
  PQ_m = number of PQ subquantizers (e.g., 48)
```

### Practical Calculation

```python
def estimate_knn_memory(
    num_vectors: int,
    dimensions: int,
    engine: str = "lucene",
    data_type: str = "float32",
    hnsw_m: int = 16,
    pq_m: int = None,
    replicas: int = 1,
    shards: int = 3
) -> dict:
    """Estimate k-NN memory requirements."""
    bytes_per_dim = {"float32": 4, "byte": 1, "float16": 2}[data_type]

    if pq_m and engine == "faiss":
        # PQ: each vector stored as pq_m bytes (codebook codes)
        vector_bytes = pq_m * 1
    else:
        vector_bytes = dimensions * bytes_per_dim

    graph_bytes = hnsw_m * 2 * 4  # Bidirectional links, 4 bytes each
    metadata_bytes = 48

    per_vector = vector_bytes + graph_bytes + metadata_bytes
    raw_index = num_vectors * per_vector

    # Overhead factor
    overhead = 1.3 if engine == "lucene" else 1.1  # Lucene has JVM overhead
    index_memory = raw_index * overhead

    # Total across shards and replicas
    total_copies = shards * (1 + replicas)
    per_shard = index_memory / shards
    total = per_shard * total_copies

    return {
        "per_vector_bytes": per_vector,
        "per_shard_gb": per_shard / 1e9,
        "total_gb": total / 1e9,
        "total_copies": total_copies,
        "recommendation": _recommend_instance(total / 1e9)
    }


def _recommend_instance(total_gb: float) -> str:
    if total_gb < 8:
        return "r6g.large (16 GB RAM)"
    elif total_gb < 16:
        return "r6g.xlarge (32 GB RAM)"
    elif total_gb < 32:
        return "r6g.2xlarge (64 GB RAM)"
    elif total_gb < 64:
        return "r6g.4xlarge (128 GB RAM)"
    elif total_gb < 128:
        return "r6g.8xlarge (256 GB RAM)"
    else:
        return f"r6g.8xlarge cluster ({int(total_gb / 200) + 1} nodes)"


# Example calculations
print(estimate_knn_memory(1_000_000, 1536, "lucene", "float32", 16, replicas=1))
print(estimate_knn_memory(10_000_000, 1536, "lucene", "byte", 16, replicas=1))
print(estimate_knn_memory(50_000_000, 1536, "faiss", "float32", 32, pq_m=48, replicas=1))
```

---

## Bulk Indexing

### Python Bulk Indexing Pattern

```python
from opensearchpy import OpenSearch, helpers
import numpy as np


def bulk_index_vectors(
    client: OpenSearch,
    index_name: str,
    documents: list[dict],
    batch_size: int = 500
):
    """Bulk index documents with embeddings."""
    def generate_actions():
        for doc in documents:
            yield {
                "_index": index_name,
                "_id": doc["id"],
                "_source": {
                    "embedding": doc["embedding"],
                    "content": doc["content"],
                    "metadata": doc.get("metadata", {})
                }
            }

    # Disable refresh during bulk indexing
    client.indices.put_settings(
        index=index_name,
        body={"index": {"refresh_interval": "-1"}}
    )

    try:
        success, errors = helpers.bulk(
            client,
            generate_actions(),
            chunk_size=batch_size,
            max_retries=3,
            request_timeout=120
        )
        print(f"Indexed {success} documents, {len(errors)} errors")
    finally:
        # Re-enable refresh
        client.indices.put_settings(
            index=index_name,
            body={"index": {"refresh_interval": "30s"}}
        )

    # Force merge for optimal search performance
    client.indices.forcemerge(
        index=index_name,
        max_num_segments=1,
        request_timeout=600
    )

    # Refresh to make documents searchable
    client.indices.refresh(index=index_name)
```

### Optimizing Bulk Indexing

```python
# Optimal settings during bulk load
client.indices.put_settings(
    index="documents",
    body={
        "index": {
            "refresh_interval": "-1",        # Disable refresh
            "number_of_replicas": 0,         # Disable replicas during load
            "translog.flush_threshold_size": "1gb",
            "translog.durability": "async"    # Async translog for speed
        }
    }
)

# After bulk load completes:
client.indices.put_settings(
    index="documents",
    body={
        "index": {
            "refresh_interval": "30s",
            "number_of_replicas": 1,
            "translog.durability": "request"
        }
    }
)

# Force merge to optimize HNSW graph
client.indices.forcemerge(
    index="documents",
    max_num_segments=1,
    request_timeout=600
)
```

---

## Security Configuration

### IAM Authentication (Amazon OpenSearch Service)

```python
from opensearchpy import OpenSearch, RequestsHttpConnection
from requests_aws4auth import AWS4Auth
import boto3

credentials = boto3.Session().get_credentials()
region = "us-east-1"

awsauth = AWS4Auth(
    credentials.access_key,
    credentials.secret_key,
    region,
    "es",
    session_token=credentials.token
)

client = OpenSearch(
    hosts=[{"host": "search-vector-search-abc123.us-east-1.es.amazonaws.com",
            "port": 443}],
    http_auth=awsauth,
    use_ssl=True,
    verify_certs=True,
    connection_class=RequestsHttpConnection
)
```

### Fine-Grained Access Control (FGAC)

```python
# Create a role that only allows k-NN search on specific indices
role_body = {
    "cluster_permissions": [
        "cluster_composite_ops_ro"
    ],
    "index_permissions": [
        {
            "index_patterns": ["documents*"],
            "allowed_actions": [
                "indices:data/read/search",
                "indices:data/read/get"
            ],
            "fls": [],
            "masked_fields": [],
            "dls": '{"term": {"metadata.tenant_id": "${user.name}"}}'
        }
    ]
}

# Create role
client.transport.perform_request(
    "PUT",
    "/_plugins/_security/api/roles/vector_search_reader",
    body=role_body
)

# Map role to backend role
client.transport.perform_request(
    "PUT",
    "/_plugins/_security/api/rolesmapping/vector_search_reader",
    body={
        "backend_roles": ["vector-search-readers"],
        "hosts": [],
        "users": ["search-user"]
    }
)
```

### Document-Level Security for Multi-Tenancy

```python
# DLS (Document Level Security) restricts which documents a user can see
# This is applied at the index level and works with k-NN queries

tenant_role = {
    "cluster_permissions": [],
    "index_permissions": [
        {
            "index_patterns": ["documents"],
            "allowed_actions": [
                "indices:data/read/search"
            ],
            "dls": '{"term": {"metadata.tenant_id": "tenant_123"}}'
        }
    ]
}
```

---

## Monitoring

### k-NN Statistics

```python
# Get k-NN plugin statistics
stats = client.transport.perform_request("GET", "/_plugins/_knn/stats")

for node_id, node_stats in stats["nodes"].items():
    print(f"Node: {node_id}")
    print(f"  Graph memory: {node_stats.get('graph_memory_usage_percentage', 'N/A')}%")
    print(f"  Graph count: {node_stats.get('graph_count', 'N/A')}")
    print(f"  Cache capacity: {node_stats.get('cache_capacity_reached', False)}")
    print(f"  Circuit breaker triggered: {node_stats.get('circuit_breaker_triggered', False)}")
    print(f"  Total load time (ns): {node_stats.get('total_load_time', 0)}")
    print(f"  Eviction count: {node_stats.get('eviction_count', 0)}")
```

### Index Health Check

```python
def check_knn_index_health(client: OpenSearch, index_name: str):
    """Check the health of a k-NN index."""
    # Index stats
    stats = client.indices.stats(index=index_name)
    index_stats = stats["indices"][index_name]

    total_docs = index_stats["primaries"]["docs"]["count"]
    store_size = index_stats["primaries"]["store"]["size_in_bytes"]
    segment_count = index_stats["primaries"]["segments"]["count"]

    print(f"Index: {index_name}")
    print(f"  Documents: {total_docs:,}")
    print(f"  Store size: {store_size / 1e9:.1f} GB")
    print(f"  Segments: {segment_count}")
    print(f"  Segments > 1: {'force merge recommended' if segment_count > 1 else 'OK'}")

    # Check settings
    settings = client.indices.get_settings(index=index_name)
    knn_enabled = settings[index_name]["settings"]["index"].get("knn", "false")
    print(f"  k-NN enabled: {knn_enabled}")

    return {
        "docs": total_docs,
        "store_gb": store_size / 1e9,
        "segments": segment_count,
        "knn_enabled": knn_enabled
    }
```

---

## Common Pitfalls

1. **Not setting `bootstrap.memory_lock=true`**: without memory lock, the OS can swap JVM memory to disk, causing severe latency spikes for k-NN queries.

2. **Undersizing JVM heap for Lucene engine**: Lucene k-NN indexes live on JVM heap. If the index exceeds available heap, you get OutOfMemoryError. Set `-Xmx` to at least 2x the expected index size.

3. **Not force-merging after bulk indexing**: multiple segments mean multiple small HNSW graphs. Force merge to `max_num_segments=1` for optimal search performance.

4. **Using nmslib engine for new projects**: nmslib is legacy and receives no new features. Use Lucene (default) or Faiss.

5. **Not accounting for replicas in memory planning**: each replica holds its own copy of the k-NN graph. With 3 shards and 1 replica, you have 6 copies of the index in memory.

6. **Setting k-NN circuit breaker too high**: the default 60% is a good starting point. Setting it to 90% leaves insufficient memory for search operations, aggregations, and GC.

7. **Not disabling refresh during bulk indexing**: each refresh creates a new segment with a new HNSW graph. During bulk load, set `refresh_interval: -1` and refresh manually after completion.

---

## References

- OpenSearch k-NN documentation: https://opensearch.org/docs/latest/search-plugins/knn/
- Amazon OpenSearch Service: https://docs.aws.amazon.com/opensearch-service/
- OpenSearch security: https://opensearch.org/docs/latest/security/
- opensearch-py client: https://opensearch-project.github.io/opensearch-py/
- OpenSearch Docker: https://opensearch.org/docs/latest/install-and-configure/install-opensearch/docker/
