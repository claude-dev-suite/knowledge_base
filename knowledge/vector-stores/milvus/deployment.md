# Milvus Production Deployment Guide

## Overview

This guide covers deploying Milvus in production: standalone Docker, cluster with Kubernetes Helm, Zilliz Cloud managed service, the Attu web UI, sizing guidelines, persistence configuration, security, and pymilvus v2 connection patterns. The focus is on operational patterns for reliability and performance at scale.

---

## Milvus Standalone (Docker)

### Quick Start

```bash
# Download docker-compose
curl -sfL https://raw.githubusercontent.com/milvus-io/milvus/master/deployments/docker/standalone/docker-compose.yml \
  -o docker-compose.yml

docker compose up -d
```

### Production Standalone

```yaml
# docker-compose.yml
services:
  etcd:
    image: quay.io/coreos/etcd:v3.5.16
    restart: always
    environment:
      - ETCD_AUTO_COMPACTION_MODE=revision
      - ETCD_AUTO_COMPACTION_RETENTION=1000
      - ETCD_QUOTA_BACKEND_BYTES=4294967296
      - ETCD_SNAPSHOT_COUNT=50000
    volumes:
      - etcd_data:/etcd
    command: etcd -advertise-client-urls=http://127.0.0.1:2379 -listen-client-urls http://0.0.0.0:2379 --data-dir /etcd
    healthcheck:
      test: ["CMD", "etcdctl", "endpoint", "health"]
      interval: 30s
      timeout: 10s
      retries: 3

  minio:
    image: minio/minio:RELEASE.2024-11-07T00-52-20Z
    restart: always
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - minio_data:/minio_data
    command: minio server /minio_data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3

  milvus:
    image: milvusdb/milvus:v2.4.17
    restart: always
    ports:
      - "19530:19530"   # gRPC
      - "9091:9091"     # metrics
    depends_on:
      etcd:
        condition: service_healthy
      minio:
        condition: service_healthy
    environment:
      ETCD_ENDPOINTS: etcd:2379
      MINIO_ADDRESS: minio:9000
      MINIO_ACCESS_KEY: minioadmin
      MINIO_SECRET_KEY: minioadmin
    volumes:
      - milvus_data:/var/lib/milvus
      - ./milvus.yaml:/milvus/configs/milvus.yaml
    deploy:
      resources:
        limits:
          memory: 16G
        reservations:
          memory: 8G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9091/healthz"]
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  etcd_data:
  minio_data:
  milvus_data:
```

### Milvus Configuration File

```yaml
# milvus.yaml
etcd:
  endpoints:
    - etcd:2379
  rootPath: by-dev

minio:
  address: minio
  port: 9000
  accessKeyID: minioadmin
  secretAccessKey: minioadmin
  bucketName: milvus-bucket

proxy:
  port: 19530
  maxTaskNum: 1024

queryNode:
  gracefulTime: 5000
  cache:
    enabled: true
    memoryLimit: 2147483648    # 2 GB cache

dataNode:
  segment:
    maxSize: 1024              # max segment size in MB

indexNode:
  scheduler:
    buildParallel: 2

common:
  retentionDuration: 432000    # 5 days (for time travel)
  security:
    authorizationEnabled: true
```

---

## Milvus Cluster (Kubernetes Helm)

### Helm Installation

```bash
# Add Milvus Helm repo
helm repo add milvus https://zilliztech.github.io/milvus-helm
helm repo update

# Install with default settings
helm install milvus milvus/milvus \
  --namespace milvus \
  --create-namespace \
  --set cluster.enabled=true \
  --set queryNode.replicas=2 \
  --set dataNode.replicas=2 \
  --set indexNode.replicas=1
```

### Production Values File

```yaml
# values-production.yaml
cluster:
  enabled: true

image:
  repository: milvusdb/milvus
  tag: v2.4.17

queryNode:
  replicas: 3
  resources:
    requests:
      cpu: 4
      memory: 16Gi
    limits:
      cpu: 8
      memory: 32Gi

dataNode:
  replicas: 2
  resources:
    requests:
      cpu: 2
      memory: 8Gi
    limits:
      cpu: 4
      memory: 16Gi

indexNode:
  replicas: 2
  resources:
    requests:
      cpu: 4
      memory: 16Gi
    limits:
      cpu: 8
      memory: 32Gi

proxy:
  replicas: 2
  resources:
    requests:
      cpu: 2
      memory: 4Gi
    limits:
      cpu: 4
      memory: 8Gi

# etcd (built-in)
etcd:
  replicaCount: 3
  persistence:
    enabled: true
    size: 10Gi

# MinIO (built-in)
minio:
  mode: distributed
  replicas: 4
  persistence:
    enabled: true
    size: 100Gi

# Pulsar (message queue)
pulsar:
  enabled: true
  broker:
    replicaCount: 2
  bookkeeper:
    replicaCount: 3
    persistence:
      size: 50Gi

# Kafka alternative (use ONE of Pulsar or Kafka)
# kafka:
#   enabled: true
#   replicaCount: 3

service:
  type: ClusterIP
  port: 19530

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: milvus.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: milvus-tls
      hosts:
        - milvus.yourdomain.com

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

```bash
helm install milvus milvus/milvus \
  --namespace milvus \
  --create-namespace \
  -f values-production.yaml
```

### Scaling the Cluster

```bash
# Scale query nodes (for more search throughput)
helm upgrade milvus milvus/milvus \
  --namespace milvus \
  --set queryNode.replicas=5

# Scale data nodes (for more insert throughput)
helm upgrade milvus milvus/milvus \
  --namespace milvus \
  --set dataNode.replicas=4

# Scale index nodes (for faster index building)
helm upgrade milvus milvus/milvus \
  --namespace milvus \
  --set indexNode.replicas=4
```

---

## Zilliz Cloud (Managed)

Zilliz Cloud is the fully managed Milvus service by the Milvus creators.

```python
from pymilvus import MilvusClient

# Connect to Zilliz Cloud
client = MilvusClient(
    uri="https://your-cluster-id.api.gcp-us-west1.zillizcloud.com",
    token="your-api-key",
)

# Usage is identical to self-hosted Milvus
client.create_collection(
    collection_name="documents",
    dimension=1536,
    metric_type="COSINE",
)
```

**Zilliz Cloud tiers** (approximate as of 2025):

| Tier | CU (Compute Units) | Storage | Price/Month |
|------|----|---------|----|
| Free | 1 CU | 5 GB | $0 |
| Starter | 2 CU | 25 GB | ~$65 |
| Standard | 4 CU | 100 GB | ~$250 |
| Enterprise | 8+ CU | Custom | ~$500+ |

---

## Attu Web UI

Attu is the official GUI for Milvus management.

```yaml
# Add to docker-compose.yml
services:
  attu:
    image: zilliz/attu:v2.4
    ports:
      - "8000:3000"
    environment:
      MILVUS_URL: milvus:19530
    depends_on:
      - milvus
```

```bash
# Or run standalone
docker run -d \
  --name attu \
  -p 8000:3000 \
  -e MILVUS_URL=host.docker.internal:19530 \
  zilliz/attu:v2.4
```

Access Attu at `http://localhost:8000`. It provides:
- Collection management (create, drop, load, release)
- Data browsing and search
- Index management
- System metrics visualization
- User and role management

---

## Sizing Guidelines

### RAM Estimation Formula

```
Query Node RAM = Vector Data + Index Overhead + Cache

Vector Data (loaded collections):
  float32: num_vectors * dimensions * 4 bytes
  float16: num_vectors * dimensions * 2 bytes

Index Overhead:
  HNSW: num_vectors * M * 2 * 4 bytes (graph) + num_vectors * dimensions * 4 bytes (vectors)
  IVF_FLAT: num_vectors * dimensions * 4 bytes + nlist * dimensions * 4 bytes (centroids)
  IVF_PQ: num_vectors * (m_pq * nbits / 8) bytes + codebook
  DiskANN: num_vectors * dimensions * 1 byte (PQ compressed) in RAM; full vectors on disk

Cache: ~20% of vector data

Data Node RAM: 4-8 GB base + buffered inserts
Index Node RAM: similar to query node (needs to load data for index building)
```

### Sizing Table

| Vectors | Dims | Index | Query Node RAM | Recommended Cluster |
|---------|------|-------|---------------|-------------------|
| 1M | 1536 | HNSW | ~10 GB | 1 query node (16 GB) |
| 5M | 1536 | HNSW | ~48 GB | 2 query nodes (32 GB each) |
| 10M | 1536 | IVF_FLAT | ~65 GB | 3 query nodes (32 GB each) |
| 10M | 1536 | IVF_PQ | ~8 GB | 1 query node (16 GB) |
| 100M | 1536 | DiskANN | ~20 GB | 2 query nodes (16 GB) + fast SSD |
| 1B | 768 | DiskANN | ~100 GB | 4 query nodes (32 GB) + NVMe |

### Disk Requirements

```
MinIO/S3 Storage:
  Raw vectors: num_vectors * dimensions * 4 bytes
  Indexes: varies by type (1x-3x of raw vectors)
  Segments: ~1.5x of raw vectors (with metadata)
  
  Total estimate: num_vectors * dimensions * 4 * 3 bytes (with indexes and overhead)

Example: 10M vectors, 1536 dims
  Raw: ~57 GB
  Total: ~170 GB on MinIO/S3
```

---

## Persistence Configuration

### MinIO Persistence

```yaml
# For production, configure MinIO with persistence
minio:
  persistence:
    enabled: true
    storageClass: gp3
    size: 500Gi
```

### S3-Compatible Storage (Production)

```yaml
# milvus.yaml
minio:
  address: s3.amazonaws.com
  port: 443
  accessKeyID: ${AWS_ACCESS_KEY_ID}
  secretAccessKey: ${AWS_SECRET_ACCESS_KEY}
  bucketName: milvus-production
  useSSL: true
  region: us-east-1
  useIAM: true    # use IAM role instead of access keys
```

### etcd Persistence

```yaml
etcd:
  persistence:
    enabled: true
    storageClass: gp3
    size: 20Gi
  compaction:
    auto: true
    retention: 1h
```

---

## Security

### TLS Configuration

```yaml
# milvus.yaml
tls:
  serverPemPath: /milvus/tls/server.pem
  serverKeyPath: /milvus/tls/server.key
  caPemPath: /milvus/tls/ca.pem

proxy:
  tls:
    mode: 2    # 0=disabled, 1=server-only, 2=mutual TLS
```

### User Authentication

```python
from pymilvus import connections, utility

# Connect as root
connections.connect(
    host="localhost",
    port="19530",
    user="root",
    password="Milvus",
)

# Create a user
utility.create_user(user="app_user", password="secure-password-here")

# Create a role
utility.create_role("reader")
utility.grant_privilege("reader", "Global", "*", "DescribeCollection")
utility.grant_privilege("reader", "Global", "*", "Search")
utility.grant_privilege("reader", "Global", "*", "Query")

# Assign role to user
utility.add_user_to_role("app_user", "reader")

# Connect as app_user
connections.connect(
    alias="app",
    host="localhost",
    port="19530",
    user="app_user",
    password="secure-password-here",
)
```

### Collection-Level Access Control

```python
# Grant access to specific collections
utility.grant_privilege("reader", "Collection", "documents", "Search")
utility.grant_privilege("reader", "Collection", "documents", "Query")

# Grant write access
utility.create_role("writer")
utility.grant_privilege("writer", "Collection", "documents", "Insert")
utility.grant_privilege("writer", "Collection", "documents", "Delete")
utility.grant_privilege("writer", "Collection", "documents", "Upsert")
```

---

## pymilvus v2 Connection Patterns

### Basic Connection

```python
from pymilvus import connections

# Simple connection
connections.connect(host="localhost", port="19530")

# Connection with authentication
connections.connect(
    host="localhost",
    port="19530",
    user="root",
    password="Milvus",
    secure=False,
)

# TLS connection
connections.connect(
    host="milvus.yourdomain.com",
    port="19530",
    user="root",
    password="Milvus",
    secure=True,
    server_pem_path="/path/to/server.pem",
    server_name="milvus.yourdomain.com",
)
```

### MilvusClient (Simplified API)

```python
from pymilvus import MilvusClient

# Standalone
client = MilvusClient(uri="http://localhost:19530", token="root:Milvus")

# Zilliz Cloud
client = MilvusClient(
    uri="https://your-cluster.api.gcp-us-west1.zillizcloud.com",
    token="your-api-key",
)

# Create, insert, search in one flow
client.create_collection(
    collection_name="documents",
    dimension=1536,
    metric_type="COSINE",
    auto_id=True,
)

client.insert(
    collection_name="documents",
    data=[
        {"title": "Doc 1", "vector": [0.1, 0.2, ...]},
    ],
)

results = client.search(
    collection_name="documents",
    data=[[0.1, 0.2, ...]],
    limit=10,
    output_fields=["title"],
)
```

### Connection Retry Pattern

```python
from pymilvus import connections
import time

def connect_with_retry(host, port, max_retries=5, user="root", password="Milvus"):
    """Connect to Milvus with exponential backoff."""
    for attempt in range(max_retries):
        try:
            connections.connect(
                alias="default",
                host=host,
                port=port,
                user=user,
                password=password,
            )
            print(f"Connected to Milvus at {host}:{port}")
            return True
        except Exception as e:
            wait = 2 ** attempt
            print(f"Connection attempt {attempt + 1} failed: {e}. Retrying in {wait}s...")
            time.sleep(wait)

    raise ConnectionError(f"Failed to connect after {max_retries} attempts")
```

---

## Monitoring

### Prometheus Metrics

Milvus exposes Prometheus metrics on port 9091 by default.

```bash
curl http://localhost:9091/metrics
```

### Key Metrics

```yaml
# prometheus/alerts.yml
groups:
  - name: milvus
    rules:
      - alert: MilvusHighSearchLatency
        expr: histogram_quantile(0.99, rate(milvus_proxy_sq_latency_bucket[5m])) > 0.5
        for: 5m
        labels:
          severity: warning

      - alert: MilvusHighMemoryUsage
        expr: milvus_querynode_memory_usage_bytes / 1024^3 > 28
        for: 5m
        labels:
          severity: critical

      - alert: MilvusSegmentNotSealed
        expr: milvus_datacoord_growing_segment_count > 100
        for: 15m
        labels:
          severity: warning
```

### Grafana Dashboard Queries

```
# Search QPS
rate(milvus_proxy_sq_count[1m])

# Search latency p99
histogram_quantile(0.99, rate(milvus_proxy_sq_latency_bucket[5m]))

# Insert rate
rate(milvus_proxy_mutation_count[1m])

# Query node memory
milvus_querynode_memory_usage_bytes

# Collection loaded status
milvus_querycoord_loaded_collection_count

# Growing segments (should be bounded)
milvus_datacoord_growing_segment_count
```

### Health Check Script

```python
from pymilvus import connections, utility

def check_milvus_health(host="localhost", port="19530"):
    """Production health check for Milvus."""
    try:
        connections.connect(host=host, port=port, timeout=5)
    except Exception as e:
        print(f"CRITICAL: cannot connect: {e}")
        return False

    # Check server version
    version = utility.get_server_version()
    print(f"Milvus version: {version}")

    # Check loaded collections
    collections = utility.list_collections()
    for name in collections:
        from pymilvus import Collection
        col = Collection(name)
        loaded = utility.load_state(name)
        print(f"Collection '{name}': entities={col.num_entities}, loaded={loaded}")

    return True
```

---

## Common Pitfalls

1. **Not provisioning enough etcd storage**: etcd stores all metadata. If it fills up, Milvus stops accepting writes. Monitor etcd disk usage and set quotas appropriately.

2. **Running Pulsar without persistence**: Pulsar bookkeeper stores the write-ahead log. Without persistent volumes, data loss occurs on restart.

3. **Undersizing query nodes for loaded collections**: all searched collections must fit in query node RAM. If a collection is too large, use IVF_PQ or DiskANN to reduce memory requirements.

4. **Not configuring S3/MinIO for production**: the default MinIO single-node setup has no redundancy. Use S3 or a distributed MinIO deployment for production.

5. **Skipping the Attu UI**: Attu provides immediate visibility into collection status, segment health, and query performance. Deploy it alongside Milvus.

6. **Not setting `GOMAXPROCS` in containers**: without proper CPU limits, Go processes may compete for CPU. Set `GOMAXPROCS` to the container CPU limit.

7. **Forgetting to release unused collections**: loaded collections consume query node RAM even when not searched. Release collections that are not actively queried.

---

## References

- Milvus deployment guide: https://milvus.io/docs/install_standalone-docker.md
- Milvus Helm chart: https://github.com/zilliztech/milvus-helm
- Zilliz Cloud: https://cloud.zilliz.com/
- Attu: https://github.com/zilliztech/attu
- Milvus security: https://milvus.io/docs/authenticate.md
- Milvus monitoring: https://milvus.io/docs/monitor.md
- pymilvus: https://github.com/milvus-io/pymilvus
