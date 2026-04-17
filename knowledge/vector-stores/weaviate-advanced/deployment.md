# Weaviate Production Deployment Guide

## Overview

This guide covers deploying Weaviate in production: Docker standalone and cluster, Weaviate Cloud, Kubernetes, backup/restore, module configuration, authentication, resource sizing, and monitoring. The focus is on operational patterns for reliability and performance at scale.

---

## Docker Standalone

### Quick Start

```bash
docker run -d \
  --name weaviate \
  -p 8080:8080 \
  -p 50051:50051 \
  -e QUERY_DEFAULTS_LIMIT=25 \
  -e AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED=true \
  -e PERSISTENCE_DATA_PATH=/var/lib/weaviate \
  -e DEFAULT_VECTORIZER_MODULE=none \
  -e CLUSTER_HOSTNAME=node1 \
  -v weaviate_data:/var/lib/weaviate \
  cr.weaviate.io/semitechnologies/weaviate:1.27.6
```

### Production Docker Compose (Standalone)

```yaml
# docker-compose.yml
services:
  weaviate:
    image: cr.weaviate.io/semitechnologies/weaviate:1.27.6
    restart: always
    ports:
      - "8080:8080"     # REST + GraphQL
      - "50051:50051"   # gRPC
    volumes:
      - weaviate_data:/var/lib/weaviate
    environment:
      QUERY_DEFAULTS_LIMIT: 25
      AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED: "false"
      AUTHENTICATION_APIKEY_ENABLED: "true"
      AUTHENTICATION_APIKEY_ALLOWED_KEYS: "admin-key-here,readonly-key-here"
      AUTHENTICATION_APIKEY_USERS: "admin@company.com,reader@company.com"
      AUTHORIZATION_ADMINLIST_ENABLED: "true"
      AUTHORIZATION_ADMINLIST_USERS: "admin@company.com"
      AUTHORIZATION_ADMINLIST_READONLY_USERS: "reader@company.com"
      PERSISTENCE_DATA_PATH: /var/lib/weaviate
      DEFAULT_VECTORIZER_MODULE: text2vec-openai
      ENABLE_MODULES: "text2vec-openai,generative-openai,text2vec-cohere"
      OPENAI_APIKEY: "${OPENAI_API_KEY}"
      COHERE_APIKEY: "${COHERE_API_KEY}"
      CLUSTER_HOSTNAME: node1
      GOMAXPROCS: "8"
      LIMIT_RESOURCES: "true"
      GOMEMLIMIT: "12GiB"
    deploy:
      resources:
        limits:
          memory: 16G
        reservations:
          memory: 8G
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8080/v1/.well-known/ready"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  weaviate_data:
    driver: local
```

---

## Docker Compose Cluster (3 Nodes)

```yaml
# docker-compose-cluster.yml
services:
  weaviate-node-0:
    image: cr.weaviate.io/semitechnologies/weaviate:1.27.6
    restart: always
    ports:
      - "8080:8080"
      - "50051:50051"
    volumes:
      - node0_data:/var/lib/weaviate
    environment:
      CLUSTER_HOSTNAME: node0
      CLUSTER_GOSSIP_BIND_PORT: 7100
      CLUSTER_DATA_BIND_PORT: 7101
      CLUSTER_JOIN: "weaviate-node-0:7100,weaviate-node-1:7100,weaviate-node-2:7100"
      PERSISTENCE_DATA_PATH: /var/lib/weaviate
      DEFAULT_VECTORIZER_MODULE: text2vec-openai
      ENABLE_MODULES: "text2vec-openai,generative-openai"
      OPENAI_APIKEY: "${OPENAI_API_KEY}"
      AUTHENTICATION_APIKEY_ENABLED: "true"
      AUTHENTICATION_APIKEY_ALLOWED_KEYS: "admin-key"
      AUTHENTICATION_APIKEY_USERS: "admin@company.com"
      AUTHORIZATION_ADMINLIST_ENABLED: "true"
      AUTHORIZATION_ADMINLIST_USERS: "admin@company.com"
      GOMEMLIMIT: "12GiB"
    networks:
      - weaviate-net
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8080/v1/.well-known/ready"]
      interval: 10s
      timeout: 5s
      retries: 5

  weaviate-node-1:
    image: cr.weaviate.io/semitechnologies/weaviate:1.27.6
    restart: always
    ports:
      - "8081:8080"
      - "50052:50051"
    volumes:
      - node1_data:/var/lib/weaviate
    environment:
      CLUSTER_HOSTNAME: node1
      CLUSTER_GOSSIP_BIND_PORT: 7100
      CLUSTER_DATA_BIND_PORT: 7101
      CLUSTER_JOIN: "weaviate-node-0:7100,weaviate-node-1:7100,weaviate-node-2:7100"
      PERSISTENCE_DATA_PATH: /var/lib/weaviate
      DEFAULT_VECTORIZER_MODULE: text2vec-openai
      ENABLE_MODULES: "text2vec-openai,generative-openai"
      OPENAI_APIKEY: "${OPENAI_API_KEY}"
      AUTHENTICATION_APIKEY_ENABLED: "true"
      AUTHENTICATION_APIKEY_ALLOWED_KEYS: "admin-key"
      AUTHENTICATION_APIKEY_USERS: "admin@company.com"
      AUTHORIZATION_ADMINLIST_ENABLED: "true"
      AUTHORIZATION_ADMINLIST_USERS: "admin@company.com"
      GOMEMLIMIT: "12GiB"
    depends_on:
      weaviate-node-0:
        condition: service_healthy
    networks:
      - weaviate-net

  weaviate-node-2:
    image: cr.weaviate.io/semitechnologies/weaviate:1.27.6
    restart: always
    ports:
      - "8082:8080"
      - "50053:50051"
    volumes:
      - node2_data:/var/lib/weaviate
    environment:
      CLUSTER_HOSTNAME: node2
      CLUSTER_GOSSIP_BIND_PORT: 7100
      CLUSTER_DATA_BIND_PORT: 7101
      CLUSTER_JOIN: "weaviate-node-0:7100,weaviate-node-1:7100,weaviate-node-2:7100"
      PERSISTENCE_DATA_PATH: /var/lib/weaviate
      DEFAULT_VECTORIZER_MODULE: text2vec-openai
      ENABLE_MODULES: "text2vec-openai,generative-openai"
      OPENAI_APIKEY: "${OPENAI_API_KEY}"
      AUTHENTICATION_APIKEY_ENABLED: "true"
      AUTHENTICATION_APIKEY_ALLOWED_KEYS: "admin-key"
      AUTHENTICATION_APIKEY_USERS: "admin@company.com"
      AUTHORIZATION_ADMINLIST_ENABLED: "true"
      AUTHORIZATION_ADMINLIST_USERS: "admin@company.com"
      GOMEMLIMIT: "12GiB"
    depends_on:
      weaviate-node-0:
        condition: service_healthy
    networks:
      - weaviate-net

volumes:
  node0_data:
  node1_data:
  node2_data:

networks:
  weaviate-net:
    driver: bridge
```

---

## Weaviate Cloud

Weaviate Cloud (WCD) is the fully managed option with automatic scaling, backups, and monitoring.

```python
import weaviate

# Connect to Weaviate Cloud
client = weaviate.connect_to_weaviate_cloud(
    cluster_url="https://your-cluster.weaviate.network",
    auth_credentials=weaviate.auth.AuthApiKey("your-wcd-api-key"),
    headers={
        "X-OpenAI-Api-Key": "your-openai-key",
    },
)

# Usage is identical to self-hosted
collection = client.collections.create(
    name="Document",
    vectorizer_config=weaviate.classes.config.Configure.Vectorizer.text2vec_openai(),
)
```

**Weaviate Cloud tiers** (approximate as of 2025):

| Tier | Vectors | RAM | Price/Month |
|------|---------|-----|-------------|
| Sandbox (free) | 50K | 1 GB | $0 (14-day expiry) |
| Starter | 500K | 4 GB | ~$25 |
| Standard | 5M | 16 GB | ~$150 |
| Business | 25M+ | 64 GB+ | ~$500+ |
| Enterprise | Custom | Custom | Contact sales |

---

## Kubernetes Deployment

### Helm Chart

```bash
# Add Weaviate Helm repo
helm repo add weaviate https://weaviate.github.io/weaviate-helm
helm repo update

# Install single-node
helm install weaviate weaviate/weaviate \
  --namespace weaviate \
  --create-namespace \
  --set replicas=1 \
  --set storage.size=50Gi \
  --set resources.requests.memory=8Gi \
  --set resources.limits.memory=16Gi \
  --set env.DEFAULT_VECTORIZER_MODULE=text2vec-openai \
  --set env.ENABLE_MODULES="text2vec-openai,generative-openai"
```

### Production Values File

```yaml
# values-production.yaml
replicas: 3

image:
  registry: cr.weaviate.io
  repo: semitechnologies/weaviate
  tag: 1.27.6

storage:
  size: 100Gi
  storageClassName: gp3

resources:
  requests:
    cpu: 4
    memory: 16Gi
  limits:
    cpu: 8
    memory: 32Gi

env:
  DEFAULT_VECTORIZER_MODULE: text2vec-openai
  ENABLE_MODULES: "text2vec-openai,generative-openai,text2vec-cohere"
  AUTHENTICATION_APIKEY_ENABLED: "true"
  AUTHENTICATION_APIKEY_ALLOWED_KEYS: "your-admin-key"
  AUTHENTICATION_APIKEY_USERS: "admin@company.com"
  AUTHORIZATION_ADMINLIST_ENABLED: "true"
  AUTHORIZATION_ADMINLIST_USERS: "admin@company.com"
  GOMEMLIMIT: "28GiB"
  QUERY_DEFAULTS_LIMIT: 25
  LIMIT_RESOURCES: "true"

envSecrets:
  OPENAI_APIKEY:
    secretName: weaviate-secrets
    secretKey: openai-api-key

service:
  type: ClusterIP
  ports:
    http: 8080
    grpc: 50051

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: "64m"
  hosts:
    - host: weaviate.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: weaviate-tls
      hosts:
        - weaviate.yourdomain.com

podDisruptionBudget:
  enabled: true
  minAvailable: 2

topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule

backups:
  enabled: true
  s3:
    enabled: true
    envconfig:
      BACKUP_S3_BUCKET: weaviate-backups
      BACKUP_S3_PATH: production
      BACKUP_S3_REGION: us-east-1
```

```bash
helm install weaviate weaviate/weaviate \
  --namespace weaviate \
  -f values-production.yaml
```

---

## Backup and Restore

### Built-in Backup (S3/GCS/Filesystem)

```python
# Configure backup module via environment:
# ENABLE_MODULES: "backup-s3"   (or backup-gcs, backup-filesystem)
# BACKUP_S3_BUCKET: "weaviate-backups"
# BACKUP_S3_REGION: "us-east-1"

# Create backup (all collections)
result = client.backup.create(
    backup_id="daily-2025-04-15",
    backend="s3",
    wait_for_completion=True,
)
print(f"Backup status: {result.status}")

# Create backup (specific collections)
result = client.backup.create(
    backup_id="docs-backup-2025-04-15",
    backend="s3",
    include_collections=["Document", "KnowledgeBase"],
    wait_for_completion=True,
)

# List backups
backups = client.backup.get_create_status(
    backup_id="daily-2025-04-15",
    backend="s3",
)

# Restore
result = client.backup.restore(
    backup_id="daily-2025-04-15",
    backend="s3",
    wait_for_completion=True,
)
print(f"Restore status: {result.status}")
```

### Automated Backup Script

```python
import schedule
import time
from datetime import datetime
import weaviate

client = weaviate.connect_to_local()

def daily_backup():
    backup_id = f"auto-{datetime.now():%Y%m%d-%H%M%S}"
    try:
        result = client.backup.create(
            backup_id=backup_id,
            backend="s3",
            wait_for_completion=True,
        )
        print(f"Backup {backup_id}: {result.status}")
    except Exception as e:
        print(f"Backup failed: {e}")
        # Alert via PagerDuty/Slack

schedule.every().day.at("02:00").do(daily_backup)

while True:
    schedule.run_pending()
    time.sleep(60)
```

---

## Modules Configuration

### Environment Variables for Modules

```bash
# Enable modules
ENABLE_MODULES="text2vec-openai,text2vec-cohere,generative-openai,backup-s3"

# Default vectorizer (applied when collection has no explicit vectorizer)
DEFAULT_VECTORIZER_MODULE=text2vec-openai

# Module-specific API keys
OPENAI_APIKEY=sk-...
COHERE_APIKEY=...

# Local transformer model (self-hosted)
TRANSFORMERS_INFERENCE_API=http://transformer-service:8080
```

### Running a Local Vectorizer

```yaml
# docker-compose with local transformer
services:
  weaviate:
    image: cr.weaviate.io/semitechnologies/weaviate:1.27.6
    environment:
      DEFAULT_VECTORIZER_MODULE: text2vec-transformers
      ENABLE_MODULES: "text2vec-transformers"
      TRANSFORMERS_INFERENCE_API: "http://t2v-transformers:8080"
    # ... other config

  t2v-transformers:
    image: cr.weaviate.io/semitechnologies/transformers-inference:sentence-transformers-all-MiniLM-L6-v2
    environment:
      ENABLE_CUDA: 0    # 1 for GPU
```

---

## Authentication

### API Key Authentication

```bash
# Environment variables
AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED=false
AUTHENTICATION_APIKEY_ENABLED=true
AUTHENTICATION_APIKEY_ALLOWED_KEYS="admin-key-123,readonly-key-456"
AUTHENTICATION_APIKEY_USERS="admin@company.com,reader@company.com"

# Authorization
AUTHORIZATION_ADMINLIST_ENABLED=true
AUTHORIZATION_ADMINLIST_USERS="admin@company.com"
AUTHORIZATION_ADMINLIST_READONLY_USERS="reader@company.com"
```

### OIDC Authentication

```bash
# OIDC with Auth0/Keycloak/Okta
AUTHENTICATION_OIDC_ENABLED=true
AUTHENTICATION_OIDC_ISSUER=https://auth.company.com/realms/weaviate
AUTHENTICATION_OIDC_CLIENT_ID=weaviate
AUTHENTICATION_OIDC_USERNAME_CLAIM=email
AUTHENTICATION_OIDC_GROUPS_CLAIM=groups
```

```python
import weaviate

# Connect with OIDC token
client = weaviate.connect_to_local(
    auth_credentials=weaviate.auth.AuthBearerToken(
        access_token="your-oidc-access-token",
        expires_in=3600,
        refresh_token="your-refresh-token",
    ),
)
```

---

## Resource Sizing

### Memory Estimation

```
Total RAM = Vector Memory + Object Memory + Index Overhead

Vector Memory (per collection):
  - float32: num_objects * dimensions * 4 bytes
  - With PQ: num_objects * segments * 1 byte (segments = num PQ segments)
  - With BQ: num_objects * dimensions / 8 bytes
  - With SQ: num_objects * dimensions * 1 byte

HNSW Graph:
  - num_objects * max_connections * 2 * 8 bytes

Object Storage (LSM tree):
  - num_objects * avg_object_size_bytes

Inverted Index:
  - Varies by property types and cardinality
  - Estimate: 20-50% of object storage

Overhead: 30% additional for Go runtime, caches, buffers
```

### Sizing Table

| Objects | Dims | Compression | Estimated RAM | Recommended Instance |
|---------|------|-------------|--------------|---------------------|
| 100K | 1536 | None | ~2 GB | 4 GB RAM |
| 500K | 1536 | None | ~6 GB | 12 GB RAM |
| 1M | 1536 | None | ~10 GB | 16 GB RAM |
| 1M | 1536 | PQ | ~4 GB | 8 GB RAM |
| 5M | 1536 | PQ | ~15 GB | 32 GB RAM |
| 10M | 1536 | PQ | ~28 GB | 48 GB RAM |
| 1M | 768 | None | ~6 GB | 12 GB RAM |
| 5M | 768 | SQ | ~12 GB | 24 GB RAM |

### GOMEMLIMIT Configuration

Always set `GOMEMLIMIT` to ~80-90% of the container memory limit to prevent OOM kills:

```bash
# Container limit: 16 GB
GOMEMLIMIT=14GiB

# Container limit: 32 GB
GOMEMLIMIT=28GiB

# Container limit: 64 GB
GOMEMLIMIT=56GiB
```

---

## Monitoring

### Built-in Metrics (Prometheus)

Weaviate exposes Prometheus metrics on port 2112.

```bash
curl http://localhost:2112/metrics
```

### Key Metrics

```yaml
# prometheus/alerts.yml
groups:
  - name: weaviate
    rules:
      - alert: WeaviateHighQueryLatency
        expr: histogram_quantile(0.99, rate(weaviate_query_dimensions_combined_total_bucket[5m])) > 0.5
        for: 5m
        labels:
          severity: warning

      - alert: WeaviateHighMemory
        expr: process_resident_memory_bytes{job="weaviate"} / 1024^3 > 28
        for: 5m
        labels:
          severity: critical

      - alert: WeaviateObjectInsertErrors
        expr: rate(weaviate_objects_batch_durations_total{status="error"}[5m]) > 0
        for: 2m
        labels:
          severity: warning
```

### Grafana Dashboard Queries

```
# Query latency p99
histogram_quantile(0.99, rate(weaviate_query_dimensions_combined_total_bucket[5m]))

# Objects per second (insert rate)
rate(weaviate_objects_durations_total_count{operation="put"}[1m])

# Total objects
weaviate_objects_count

# Memory usage
process_resident_memory_bytes{job="weaviate"}

# Go garbage collection pause
go_gc_duration_seconds{quantile="0.99"}

# Active goroutines (indicator of concurrency pressure)
go_goroutines{job="weaviate"}
```

### Health Check

```python
import requests

def check_weaviate_health(host: str = "localhost", port: int = 8080):
    """Production health check."""
    # Readiness
    try:
        resp = requests.get(f"http://{host}:{port}/v1/.well-known/ready", timeout=5)
        if resp.status_code != 200:
            print(f"CRITICAL: not ready ({resp.status_code})")
            return False
    except Exception as e:
        print(f"CRITICAL: unreachable: {e}")
        return False

    # Liveness
    resp = requests.get(f"http://{host}:{port}/v1/.well-known/live", timeout=5)
    if resp.status_code != 200:
        print(f"WARNING: liveness check failed")

    # Node status
    resp = requests.get(f"http://{host}:{port}/v1/nodes", timeout=5)
    nodes = resp.json().get("nodes", [])
    for node in nodes:
        name = node["name"]
        status = node["status"]
        shards = node.get("shards", [])
        obj_count = sum(s.get("objectCount", 0) for s in shards)
        print(f"Node '{name}': status={status}, objects={obj_count}")

    return True
```

---

## Common Pitfalls

1. **Not setting GOMEMLIMIT**: without this, the Go garbage collector does not know the container's memory limit and may trigger OOM kills. Always set to ~85% of container limit.

2. **Enabling all modules unnecessarily**: each enabled module consumes memory and startup time. Only enable modules you actually use.

3. **Not configuring authentication in production**: Weaviate defaults to anonymous access enabled. Any client can read, write, and delete data.

4. **Using filesystem backup on ephemeral storage**: Kubernetes pods with no persistent volume lose backups on restart. Use S3 or GCS backend for backups.

5. **Not testing restore from backup**: backups are only useful if restore works. Test the full backup-restore cycle before going to production.

6. **Undersizing storage for LSM compaction**: LSM trees need temporary space during compaction (up to 2x current size). Provision at least 2.5x the expected data size.

7. **Running cluster nodes without anti-affinity**: if all Weaviate nodes land on the same Kubernetes host, a single host failure takes down the entire cluster. Use pod anti-affinity or topology spread constraints.

---

## References

- Weaviate deployment guide: https://weaviate.io/developers/weaviate/installation
- Weaviate Helm chart: https://github.com/weaviate/weaviate-helm
- Weaviate Cloud: https://console.weaviate.cloud/
- Weaviate backup: https://weaviate.io/developers/weaviate/configuration/backups
- Weaviate monitoring: https://weaviate.io/developers/weaviate/configuration/monitoring
- Weaviate authentication: https://weaviate.io/developers/weaviate/configuration/authentication
