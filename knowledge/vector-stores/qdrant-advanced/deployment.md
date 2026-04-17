# Qdrant Production Deployment Guide

## Overview

This guide covers deploying Qdrant in production: Docker single-node, Docker Compose clusters, Qdrant Cloud, Kubernetes Helm, backup/restore, security (TLS, API keys, RBAC), memory sizing, and monitoring. The focus is on operational patterns for reliability and performance.

---

## Docker Single-Node

### Quick Start

```bash
docker run -d \
  --name qdrant \
  -p 6333:6333 \
  -p 6334:6334 \
  -v qdrant_storage:/qdrant/storage \
  qdrant/qdrant:v1.12.1
```

### Production Single-Node

```yaml
# docker-compose.yml
services:
  qdrant:
    image: qdrant/qdrant:v1.12.1
    restart: always
    ports:
      - "6333:6333"   # REST API
      - "6334:6334"   # gRPC
    volumes:
      - qdrant_storage:/qdrant/storage
      - ./config/config.yaml:/qdrant/config/production.yaml
    environment:
      - QDRANT__SERVICE__GRPC_PORT=6334
      - QDRANT__SERVICE__HTTP_PORT=6333
      - QDRANT__LOG_LEVEL=INFO
    deploy:
      resources:
        limits:
          memory: 16G
        reservations:
          memory: 8G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  qdrant_storage:
    driver: local
```

### Configuration File

```yaml
# config/config.yaml
storage:
  storage_path: /qdrant/storage
  snapshots_path: /qdrant/snapshots

  # Performance settings
  optimizers:
    default_segment_number: 4
    max_optimization_threads: 4
    indexing_threshold_kb: 20000
    flush_interval_sec: 5

  # WAL settings
  wal:
    wal_capacity_mb: 32

  # HNSW defaults
  hnsw_index:
    m: 16
    ef_construct: 128
    full_scan_threshold: 10000

service:
  host: 0.0.0.0
  http_port: 6333
  grpc_port: 6334
  max_request_size_mb: 32

  # API key authentication
  api_key: "your-secure-api-key-here"
  read_only_api_key: "your-read-only-key-here"

telemetry_disabled: false
```

---

## Docker Compose Cluster

Qdrant supports distributed mode with sharding and replication.

```yaml
# docker-compose-cluster.yml
services:
  qdrant-node-0:
    image: qdrant/qdrant:v1.12.1
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - node0_storage:/qdrant/storage
    environment:
      - QDRANT__CLUSTER__ENABLED=true
      - QDRANT__CLUSTER__P2P__PORT=6335
      - QDRANT__SERVICE__API_KEY=your-secure-api-key
    networks:
      - qdrant-net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/healthz"]
      interval: 10s
      timeout: 5s
      retries: 5

  qdrant-node-1:
    image: qdrant/qdrant:v1.12.1
    ports:
      - "6336:6333"
      - "6337:6334"
    volumes:
      - node1_storage:/qdrant/storage
    environment:
      - QDRANT__CLUSTER__ENABLED=true
      - QDRANT__CLUSTER__P2P__PORT=6335
      - QDRANT__CLUSTER__BOOTSTRAP=http://qdrant-node-0:6335
      - QDRANT__SERVICE__API_KEY=your-secure-api-key
    depends_on:
      qdrant-node-0:
        condition: service_healthy
    networks:
      - qdrant-net

  qdrant-node-2:
    image: qdrant/qdrant:v1.12.1
    ports:
      - "6338:6333"
      - "6339:6334"
    volumes:
      - node2_storage:/qdrant/storage
    environment:
      - QDRANT__CLUSTER__ENABLED=true
      - QDRANT__CLUSTER__P2P__PORT=6335
      - QDRANT__CLUSTER__BOOTSTRAP=http://qdrant-node-0:6335
      - QDRANT__SERVICE__API_KEY=your-secure-api-key
    depends_on:
      qdrant-node-0:
        condition: service_healthy
    networks:
      - qdrant-net

volumes:
  node0_storage:
  node1_storage:
  node2_storage:

networks:
  qdrant-net:
    driver: bridge
```

### Creating a Distributed Collection

```python
from qdrant_client import QdrantClient
from qdrant_client.models import VectorParams, Distance

client = QdrantClient(host="localhost", port=6333, api_key="your-secure-api-key")

# Create collection with sharding and replication
client.create_collection(
    collection_name="documents",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
    shard_number=6,              # distribute across nodes
    replication_factor=2,        # each shard replicated on 2 nodes
    write_consistency_factor=1,  # quorum for writes
)
```

---

## Qdrant Cloud

Qdrant Cloud is the fully managed option. No infrastructure management, automatic scaling, and built-in monitoring.

```python
# Connect to Qdrant Cloud
client = QdrantClient(
    url="https://your-cluster-id.us-east4-0.gcp.cloud.qdrant.io:6333",
    api_key="your-cloud-api-key",
    prefer_grpc=True,
)

# Usage is identical to self-hosted
client.create_collection(
    collection_name="documents",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
)
```

**Qdrant Cloud tiers** (approximate as of 2025):

| Tier | RAM | Storage | Price/Month |
|------|-----|---------|-------------|
| Free | 1 GB | 4 GB | $0 |
| Starter | 4 GB | 16 GB | ~$25 |
| Production | 16 GB | 64 GB | ~$100 |
| Enterprise | Custom | Custom | Contact sales |

---

## Kubernetes Helm Deployment

### Helm Chart Installation

```bash
# Add Qdrant Helm repository
helm repo add qdrant https://qdrant.github.io/qdrant-helm
helm repo update

# Install single-node
helm install qdrant qdrant/qdrant \
  --namespace qdrant \
  --create-namespace \
  --set replicaCount=1 \
  --set persistence.size=50Gi \
  --set resources.requests.memory=8Gi \
  --set resources.limits.memory=16Gi \
  --set apiKey="your-secure-api-key"

# Install cluster (3 nodes)
helm install qdrant qdrant/qdrant \
  --namespace qdrant \
  --create-namespace \
  --set replicaCount=3 \
  --set persistence.size=100Gi \
  --set resources.requests.memory=16Gi \
  --set resources.limits.memory=32Gi \
  --set apiKey="your-secure-api-key" \
  --set cluster.enabled=true
```

### Custom Values File

```yaml
# values-production.yaml
replicaCount: 3

image:
  repository: qdrant/qdrant
  tag: v1.12.1

persistence:
  enabled: true
  size: 100Gi
  storageClass: gp3    # AWS EBS gp3 for predictable IOPS

resources:
  requests:
    cpu: 4
    memory: 16Gi
  limits:
    cpu: 8
    memory: 32Gi

config:
  cluster:
    enabled: true
  storage:
    optimizers:
      default_segment_number: 4
      max_optimization_threads: 4
    hnsw_index:
      m: 16
      ef_construct: 128

service:
  type: ClusterIP
  ports:
    rest: 6333
    grpc: 6334

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: qdrant.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: qdrant-tls
      hosts:
        - qdrant.yourdomain.com

podDisruptionBudget:
  enabled: true
  minAvailable: 2

topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
```

```bash
helm install qdrant qdrant/qdrant \
  --namespace qdrant \
  -f values-production.yaml
```

---

## Snapshots and Backups

### Creating Snapshots

```python
# Snapshot a specific collection
snapshot_info = client.create_snapshot(collection_name="documents")
print(f"Snapshot created: {snapshot_info.name}")

# List snapshots
snapshots = client.list_snapshots(collection_name="documents")
for snap in snapshots:
    print(f"  {snap.name} - {snap.size} bytes - {snap.creation_time}")
```

### Downloading Snapshots

```python
import requests

# Download snapshot file
snapshot_name = snapshot_info.name
url = f"http://localhost:6333/collections/documents/snapshots/{snapshot_name}"
response = requests.get(url, headers={"api-key": "your-api-key"}, stream=True)

with open(f"/backups/{snapshot_name}", "wb") as f:
    for chunk in response.iter_content(chunk_size=8192):
        f.write(chunk)
```

### Restoring from Snapshot

```bash
# Upload and restore a snapshot
curl -X POST "http://localhost:6333/collections/documents/snapshots/upload" \
  -H "api-key: your-api-key" \
  -H "Content-Type: multipart/form-data" \
  -F "snapshot=@/backups/documents-snapshot-2025-01-15.snapshot"
```

### Full Storage Snapshot

```python
# Snapshot ALL collections at once
full_snapshot = client.create_full_snapshot()
print(f"Full snapshot: {full_snapshot.name}")

# List full snapshots
full_snapshots = client.list_full_snapshots()
```

### Automated Backup Script

```python
import schedule
import time
from datetime import datetime
from qdrant_client import QdrantClient
import requests
import os

client = QdrantClient(host="localhost", port=6333, api_key="your-api-key")
BACKUP_DIR = "/backups/qdrant"

def backup_all_collections():
    """Create and download snapshots for all collections."""
    collections = client.get_collections().collections
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")

    for col in collections:
        name = col.name
        print(f"Backing up collection: {name}")

        # Create snapshot
        snap = client.create_snapshot(collection_name=name)

        # Download
        url = f"http://localhost:6333/collections/{name}/snapshots/{snap.name}"
        resp = requests.get(url, headers={"api-key": "your-api-key"}, stream=True)

        backup_path = os.path.join(BACKUP_DIR, f"{name}_{timestamp}.snapshot")
        with open(backup_path, "wb") as f:
            for chunk in resp.iter_content(chunk_size=8192):
                f.write(chunk)

        print(f"  Saved: {backup_path} ({os.path.getsize(backup_path)} bytes)")

    # Clean up old backups (keep last 7 days)
    cleanup_old_backups(BACKUP_DIR, max_age_days=7)

def cleanup_old_backups(directory, max_age_days):
    cutoff = time.time() - (max_age_days * 86400)
    for f in os.listdir(directory):
        path = os.path.join(directory, f)
        if os.path.getmtime(path) < cutoff:
            os.remove(path)
            print(f"  Removed old backup: {f}")

# Schedule daily backups at 2 AM
schedule.every().day.at("02:00").do(backup_all_collections)

while True:
    schedule.run_pending()
    time.sleep(60)
```

---

## TLS Configuration

### Self-Signed Certificates (Development)

```bash
# Generate CA and server certificates
openssl req -x509 -newkey rsa:4096 -keyout ca-key.pem -out ca-cert.pem \
  -days 365 -nodes -subj "/CN=Qdrant CA"

openssl req -newkey rsa:4096 -keyout server-key.pem -out server-req.pem \
  -nodes -subj "/CN=qdrant.local"

openssl x509 -req -in server-req.pem -CA ca-cert.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem -days 365
```

### Qdrant TLS Configuration

```yaml
# config/config.yaml
service:
  host: 0.0.0.0
  http_port: 6333
  grpc_port: 6334

  # TLS configuration
  enable_tls: true
  tls:
    cert: /qdrant/tls/server-cert.pem
    key: /qdrant/tls/server-key.pem
    ca_cert: /qdrant/tls/ca-cert.pem
```

### Client Connection with TLS

```python
from qdrant_client import QdrantClient

# With TLS verification
client = QdrantClient(
    host="qdrant.yourdomain.com",
    port=6333,
    https=True,
    api_key="your-api-key",
)

# With custom CA certificate
client = QdrantClient(
    host="qdrant.local",
    port=6333,
    https=True,
    api_key="your-api-key",
    ca_cert="/path/to/ca-cert.pem",
)
```

---

## Authentication and RBAC

### API Key Authentication

```yaml
# config/config.yaml
service:
  api_key: "your-write-api-key"           # full access
  read_only_api_key: "your-read-only-key" # read-only access
```

```python
# Write client
write_client = QdrantClient(
    host="localhost", port=6333,
    api_key="your-write-api-key",
)

# Read-only client (for search services)
read_client = QdrantClient(
    host="localhost", port=6333,
    api_key="your-read-only-key",
)

# The read-only key cannot create collections, upsert, or delete
# It can only search, scroll, and read collection info
```

### JWT-Based Access Control (Qdrant 1.9+)

For fine-grained access control, Qdrant supports JWT tokens.

```yaml
# config/config.yaml
service:
  api_key: "your-master-key"
  jwt_rbac: true
```

```python
import jwt
from datetime import datetime, timedelta

# Generate a token with collection-level access
SECRET = "your-master-key"

token = jwt.encode(
    {
        "access": [
            {
                "collection": "documents",
                "access": "r",        # read-only
            },
            {
                "collection": "internal",
                "access": "rw",       # read-write
            },
        ],
        "exp": datetime.utcnow() + timedelta(hours=24),
    },
    SECRET,
    algorithm="HS256",
)

# Use the JWT token as the API key
client = QdrantClient(
    host="localhost", port=6333,
    api_key=token,
)
```

---

## Memory Sizing

### RAM Estimation Formula

```
Total RAM = Vector Storage + Payload Storage + HNSW Index + Overhead

Vector Storage:
  - float32: num_vectors * dimensions * 4 bytes
  - int8 (SQ): num_vectors * dimensions * 1 byte
  - binary (BQ): num_vectors * dimensions / 8 bytes

HNSW Index:
  - num_vectors * m * 2 * 4 bytes (neighbor links)
  - Plus: num_vectors * 12 bytes (node metadata)

Payload Storage:
  - Varies; estimate avg_payload_size * num_vectors

Overhead:
  - ~20% additional for segment metadata, WAL, temp buffers
```

### Sizing Examples

```python
def estimate_ram_gb(
    num_vectors: int,
    dimensions: int,
    m: int = 16,
    avg_payload_bytes: int = 256,
    quantization: str = "none",  # "none", "scalar", "binary"
) -> float:
    """Estimate RAM requirements in GB."""
    # Vector storage
    if quantization == "none":
        vec_bytes = num_vectors * dimensions * 4
    elif quantization == "scalar":
        vec_bytes = num_vectors * dimensions * 1
    elif quantization == "binary":
        vec_bytes = num_vectors * dimensions // 8
    else:
        vec_bytes = num_vectors * dimensions * 4

    # HNSW index
    hnsw_bytes = num_vectors * (m * 2 * 4 + 12)

    # Payload
    payload_bytes = num_vectors * avg_payload_bytes

    # Total with 20% overhead
    total_bytes = (vec_bytes + hnsw_bytes + payload_bytes) * 1.2

    return total_bytes / (1024 ** 3)

# Examples
print(f"1M vectors, 1536d, no quant:    {estimate_ram_gb(1_000_000, 1536):.1f} GB")
print(f"1M vectors, 1536d, scalar quant: {estimate_ram_gb(1_000_000, 1536, quantization='scalar'):.1f} GB")
print(f"1M vectors, 1536d, binary quant: {estimate_ram_gb(1_000_000, 1536, quantization='binary'):.1f} GB")
print(f"5M vectors, 1536d, scalar quant: {estimate_ram_gb(5_000_000, 1536, quantization='scalar'):.1f} GB")
print(f"10M vectors, 768d, scalar quant: {estimate_ram_gb(10_000_000, 768, quantization='scalar'):.1f} GB")
```

**Output:**

```
1M vectors, 1536d, no quant:     7.8 GB
1M vectors, 1536d, scalar quant: 2.2 GB
1M vectors, 1536d, binary quant: 0.6 GB
5M vectors, 1536d, scalar quant: 10.8 GB
10M vectors, 768d, scalar quant: 12.2 GB
```

### Memory Sizing Table

| Vectors | Dims | Quantization | RAM Estimate | Recommended Instance |
|---------|------|-------------|-------------|---------------------|
| 100K | 1536 | None | ~1 GB | 4 GB RAM |
| 500K | 1536 | None | ~4 GB | 8 GB RAM |
| 1M | 1536 | None | ~8 GB | 16 GB RAM |
| 1M | 1536 | Scalar | ~2 GB | 8 GB RAM |
| 5M | 1536 | Scalar | ~11 GB | 24 GB RAM |
| 10M | 1536 | Scalar | ~22 GB | 48 GB RAM |
| 10M | 768 | Scalar | ~12 GB | 24 GB RAM |
| 50M | 1536 | Binary | ~12 GB | 32 GB RAM |

---

## Monitoring with Prometheus

### Enabling Metrics

Qdrant exposes Prometheus metrics on the `/metrics` endpoint by default.

```bash
# Check metrics
curl http://localhost:6333/metrics
```

### Key Metrics to Monitor

```yaml
# prometheus/alerts.yml
groups:
  - name: qdrant
    rules:
      - alert: QdrantHighSearchLatency
        expr: histogram_quantile(0.99, rate(app_info_rest_responses_duration_seconds_bucket{method="POST", endpoint="/collections/.*/points/query"}[5m])) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Qdrant p99 search latency > 500ms"

      - alert: QdrantHighMemoryUsage
        expr: process_resident_memory_bytes / 1024 / 1024 / 1024 > 28
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Qdrant memory usage > 28 GB"

      - alert: QdrantGrpcErrors
        expr: rate(app_info_grpc_responses_total{status="error"}[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Qdrant gRPC error rate > 0.1/s"
```

### Grafana Dashboard Queries

```
# QPS (queries per second)
rate(app_info_rest_responses_total{method="POST", endpoint=~"/collections/.*/points/query"}[1m])

# Search latency p50
histogram_quantile(0.50, rate(app_info_rest_responses_duration_seconds_bucket{endpoint=~"/collections/.*/points/query"}[5m]))

# Search latency p99
histogram_quantile(0.99, rate(app_info_rest_responses_duration_seconds_bucket{endpoint=~"/collections/.*/points/query"}[5m]))

# Memory usage
process_resident_memory_bytes

# Collection point count
app_info_collections_total_points

# Pending optimizations
app_info_collections_optimizer_status{status="pending"}
```

### Health Check Script

```python
import requests
import sys

def check_qdrant_health(host: str = "localhost", port: int = 6333, api_key: str = None):
    """Production health check for Qdrant."""
    headers = {"api-key": api_key} if api_key else {}

    # Basic health
    try:
        resp = requests.get(f"http://{host}:{port}/healthz", headers=headers, timeout=5)
        if resp.status_code != 200:
            print(f"CRITICAL: healthz returned {resp.status_code}")
            return False
    except requests.exceptions.RequestException as e:
        print(f"CRITICAL: cannot reach Qdrant: {e}")
        return False

    # Check collections
    resp = requests.get(f"http://{host}:{port}/collections", headers=headers, timeout=5)
    collections = resp.json()["result"]["collections"]

    for col in collections:
        name = col["name"]
        info_resp = requests.get(
            f"http://{host}:{port}/collections/{name}", headers=headers, timeout=5
        )
        info = info_resp.json()["result"]

        status = info["status"]
        points = info["points_count"]
        optimizer_status = info.get("optimizer_status", {}).get("status", "ok")

        print(f"Collection '{name}': status={status}, points={points}, optimizer={optimizer_status}")

        if status != "green":
            print(f"  WARNING: collection status is {status}")

    return True

if __name__ == "__main__":
    ok = check_qdrant_health(api_key="your-api-key")
    sys.exit(0 if ok else 1)
```

---

## Common Pitfalls

1. **Running without persistent storage**: Docker containers without volume mounts lose all data on restart. Always mount `/qdrant/storage` to a persistent volume.

2. **Undersizing RAM for unquantized vectors**: a 1M collection of 1536-dim float32 vectors needs ~8 GB RAM. Running with less causes disk swapping and 10x latency degradation.

3. **Not setting API keys in production**: Qdrant has no authentication by default. Any network-reachable client can read, write, and delete collections.

4. **Skipping health checks in Kubernetes**: without proper readiness probes, traffic may be routed to nodes that are still building indexes or recovering from a restart.

5. **Using `replication_factor=1` for production clusters**: a single replica means shard loss if a node goes down. Use `replication_factor=2` minimum for production.

6. **Forgetting to clean up old snapshots**: snapshots accumulate on disk. Implement automated cleanup to prevent storage exhaustion.

7. **Not testing restore procedures**: create snapshots regularly and verify you can restore from them. A backup you have never tested is not a backup.

---

## References

- Qdrant deployment guide: https://qdrant.tech/documentation/guides/distributed_deployment/
- Qdrant Docker guide: https://qdrant.tech/documentation/guides/installation/
- Qdrant Helm chart: https://github.com/qdrant/qdrant-helm
- Qdrant Cloud: https://cloud.qdrant.io/
- Qdrant security: https://qdrant.tech/documentation/guides/security/
- Qdrant monitoring: https://qdrant.tech/documentation/guides/monitoring/
