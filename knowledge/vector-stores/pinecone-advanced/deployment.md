# Pinecone Production Deployment Guide

## Overview

This guide covers deploying Pinecone in production: account setup, index creation (serverless vs pod-based), environment selection, upsert batching, namespace strategy, backup via collections, monitoring, and cost optimization. As a fully managed service, Pinecone requires no infrastructure management, but proper configuration is essential for performance and cost control.

---

## Account Setup

### API Key Management

```python
from pinecone import Pinecone

# Create client with API key
pc = Pinecone(api_key="your-api-key")

# Verify connection
indexes = pc.list_indexes()
print(f"Available indexes: {[idx.name for idx in indexes]}")
```

**API key best practices**:
- Create separate API keys for different environments (dev, staging, production)
- Rotate keys regularly (at least quarterly)
- Store keys in environment variables or a secrets manager, never in code
- Use project-level API keys (not organization-level) for production services

### Organization and Project Structure

```
Organization
  +-- Project: production
  |     +-- Index: documents-prod
  |     +-- Index: embeddings-prod
  +-- Project: staging
  |     +-- Index: documents-staging
  +-- Project: development
        +-- Index: documents-dev
```

---

## Index Creation

### Serverless Index (Recommended)

```python
from pinecone import Pinecone, ServerlessSpec

pc = Pinecone(api_key="your-api-key")

# Create serverless index
pc.create_index(
    name="documents-prod",
    dimension=1536,
    metric="cosine",
    spec=ServerlessSpec(
        cloud="aws",
        region="us-east-1",
    ),
    deletion_protection="enabled",   # prevent accidental deletion
)

# Wait for index to be ready
import time
while not pc.describe_index("documents-prod").status.ready:
    time.sleep(5)

print("Index is ready")
```

### Pod-Based Index

```python
from pinecone import Pinecone, PodSpec

pc = Pinecone(api_key="your-api-key")

# Create pod-based index
pc.create_index(
    name="documents-pod-prod",
    dimension=1536,
    metric="cosine",
    spec=PodSpec(
        environment="us-east-1-aws",
        pod_type="p2.x1",     # p1 = performance, p2 = balanced, s1 = storage
        pods=2,               # number of pods
        replicas=2,           # replicas per pod
        shards=1,             # shards per pod
        metadata_config={
            "indexed": ["category", "year"],   # only index these metadata fields
        },
    ),
    deletion_protection="enabled",
)
```

### Pod Types

| Pod Type | Optimized For | Max Vectors (1536d) | QPS | Price/Pod/Month |
|----------|-------------|--------------------|----|----------------|
| s1.x1 | Storage | ~5M | ~50 | ~$70 |
| s1.x2 | Storage | ~10M | ~50 | ~$140 |
| p1.x1 | Performance | ~1M | ~200 | ~$70 |
| p1.x2 | Performance | ~2M | ~200 | ~$140 |
| p2.x1 | Balanced | ~1M | ~200 | ~$90 |
| p2.x2 | Balanced | ~2M | ~200 | ~$180 |

**Scaling formula**: total capacity = base capacity * pods * shards

---

## Environment Selection

### Available Regions

| Cloud | Region | Environment ID | Notes |
|-------|--------|---------------|-------|
| AWS | us-east-1 | us-east-1-aws | Lowest latency for US East workloads |
| AWS | us-west-2 | us-west-2-aws | US West |
| AWS | eu-west-1 | eu-west-1-aws | GDPR-compliant EU region |
| GCP | us-central1 | us-central1-gcp | GCP users |
| Azure | eastus | eastus-azure | Azure users |

**Region selection criteria**:
- Choose the region closest to your application servers
- For multi-region apps, create separate indexes in each region
- Serverless indexes are available in fewer regions than pod-based
- Cross-region queries add 20-80ms network latency

---

## Upsert Batching Strategy

### Optimal Batch Sizes

| Dimensions | Recommended Batch Size | Max Batch Size | Vectors/Second (1 thread) |
|-----------|----------------------|---------------|-------------------------|
| 384 | 500 | 1000 | ~5,000 |
| 768 | 300 | 1000 | ~3,000 |
| 1536 | 100-200 | 1000 | ~1,500 |
| 3072 | 50-100 | 1000 | ~800 |

### Production Upsert Pipeline

```python
import numpy as np
from pinecone import Pinecone
from concurrent.futures import ThreadPoolExecutor
import time

pc = Pinecone(api_key="your-api-key")
index = pc.Index("documents-prod")

def chunked_upsert(
    vectors: list[dict],
    namespace: str = "",
    batch_size: int = 100,
    max_workers: int = 4,
    retry_count: int = 3,
):
    """Production-grade batched upsert with parallelism and retry."""
    batches = [vectors[i:i + batch_size] for i in range(0, len(vectors), batch_size)]
    total_upserted = 0
    failed_batches = []

    def upsert_batch(batch):
        for attempt in range(retry_count):
            try:
                index.upsert(vectors=batch, namespace=namespace)
                return len(batch)
            except Exception as e:
                if attempt == retry_count - 1:
                    return batch  # return failed batch for retry
                time.sleep(2 ** attempt)
        return batch

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        results = list(executor.map(upsert_batch, batches))

    for result in results:
        if isinstance(result, int):
            total_upserted += result
        else:
            failed_batches.extend(result)

    print(f"Upserted: {total_upserted}, Failed: {len(failed_batches)}")
    return failed_batches

# Usage
vectors = [
    {
        "id": f"doc-{i}",
        "values": np.random.rand(1536).tolist(),
        "metadata": {"category": f"cat-{i % 10}", "idx": i},
    }
    for i in range(100_000)
]

failed = chunked_upsert(vectors, namespace="default", batch_size=100, max_workers=8)
```

### Rate Limiting

Pinecone applies rate limits based on your plan:

| Plan | Write Rate | Read Rate |
|------|-----------|----------|
| Starter (free) | 100 upserts/sec | 100 queries/sec |
| Standard | 1000 upserts/sec | 1000 queries/sec |
| Enterprise | Custom | Custom |

```python
import time
from collections import deque

class RateLimiter:
    """Simple rate limiter for Pinecone API calls."""

    def __init__(self, max_per_second: int):
        self.max_per_second = max_per_second
        self.timestamps = deque()

    def wait(self):
        now = time.time()
        # Remove timestamps older than 1 second
        while self.timestamps and now - self.timestamps[0] > 1.0:
            self.timestamps.popleft()

        if len(self.timestamps) >= self.max_per_second:
            sleep_time = 1.0 - (now - self.timestamps[0])
            if sleep_time > 0:
                time.sleep(sleep_time)

        self.timestamps.append(time.time())

# Usage
limiter = RateLimiter(max_per_second=500)
for batch in batches:
    limiter.wait()
    index.upsert(vectors=batch)
```

---

## Namespace Strategy for Multi-Tenancy

### Design Patterns

**Pattern 1: One namespace per tenant** (recommended for < 100K tenants)

```python
# Upsert for tenant
index.upsert(vectors=vectors, namespace=f"tenant-{tenant_id}")

# Query for tenant (isolated)
results = index.query(
    vector=query_vector,
    top_k=10,
    namespace=f"tenant-{tenant_id}",
)

# Delete tenant data
index.delete(delete_all=True, namespace=f"tenant-{tenant_id}")
```

**Pattern 2: Metadata filter per tenant** (for > 100K tenants or cross-tenant search)

```python
# Upsert with tenant in metadata
index.upsert(
    vectors=[{
        "id": "doc-1",
        "values": vector,
        "metadata": {"tenant_id": tenant_id, "title": "..."},
    }],
)

# Query with tenant filter
results = index.query(
    vector=query_vector,
    top_k=10,
    filter={"tenant_id": {"$eq": tenant_id}},
)
```

**Comparison**:

| Aspect | Namespace | Metadata Filter |
|--------|-----------|----------------|
| Isolation | Strong (separate vector space) | Logical (same index) |
| Cross-tenant search | Not possible | Possible |
| Performance | Consistent | Degrades at high cardinality |
| Tenant deletion | Fast (`delete_all`) | Slow (filter delete) |
| Max tenants | ~100K recommended | Unlimited |
| Storage overhead | Minimal | None |

---

## Backup via Collections

### Automated Backup Strategy

```python
from pinecone import Pinecone
from datetime import datetime

pc = Pinecone(api_key="your-api-key")

def create_backup(index_name: str, max_backups: int = 7):
    """Create a timestamped backup collection."""
    timestamp = datetime.now().strftime("%Y%m%d-%H%M%S")
    backup_name = f"{index_name}-backup-{timestamp}"

    print(f"Creating backup: {backup_name}")
    pc.create_collection(
        name=backup_name,
        source=index_name,
    )

    # Wait for collection to be ready
    import time
    while True:
        collections = pc.list_collections()
        col = next((c for c in collections if c.name == backup_name), None)
        if col and col.status == "Ready":
            break
        time.sleep(10)

    print(f"Backup ready: {backup_name}")

    # Clean up old backups
    cleanup_old_backups(index_name, max_backups)
    return backup_name

def cleanup_old_backups(index_name: str, max_backups: int):
    """Remove oldest backups beyond max_backups."""
    prefix = f"{index_name}-backup-"
    collections = pc.list_collections()
    backups = sorted(
        [c for c in collections if c.name.startswith(prefix)],
        key=lambda c: c.name,
    )

    while len(backups) > max_backups:
        oldest = backups.pop(0)
        print(f"Deleting old backup: {oldest.name}")
        pc.delete_collection(oldest.name)

def restore_from_backup(backup_name: str, new_index_name: str):
    """Restore an index from a backup collection."""
    # Get backup info
    collections = pc.list_collections()
    backup = next((c for c in collections if c.name == backup_name), None)
    if not backup:
        raise ValueError(f"Backup not found: {backup_name}")

    print(f"Restoring {backup_name} to {new_index_name}")
    pc.create_index(
        name=new_index_name,
        dimension=backup.dimension,
        metric=backup.metric,
        spec=ServerlessSpec(cloud="aws", region="us-east-1"),
        source_collection=backup_name,
    )

    # Wait for index to be ready
    import time
    while not pc.describe_index(new_index_name).status.ready:
        time.sleep(5)

    print(f"Restore complete: {new_index_name}")
```

---

## Monitoring Dashboard

### Index Health Monitoring

```python
from pinecone import Pinecone
import time

pc = Pinecone(api_key="your-api-key")

def monitor_index(index_name: str):
    """Monitor index health and performance."""
    index = pc.Index(index_name)

    # Get index description
    desc = pc.describe_index(index_name)
    print(f"Index: {index_name}")
    print(f"  Status: {desc.status}")
    print(f"  Dimension: {desc.dimension}")
    print(f"  Metric: {desc.metric}")
    print(f"  Deletion protection: {desc.deletion_protection}")

    # Get stats
    stats = index.describe_index_stats()
    print(f"  Total vectors: {stats.total_vector_count:,}")
    print(f"  Index fullness: {stats.index_fullness:.2%}")

    # Namespace breakdown
    for ns, ns_stats in stats.namespaces.items():
        print(f"  Namespace '{ns}': {ns_stats.vector_count:,} vectors")

    # Alert conditions
    if stats.index_fullness > 0.8:
        print("  WARNING: Index is > 80% full. Consider scaling.")
    if stats.index_fullness > 0.95:
        print("  CRITICAL: Index is > 95% full. Upserts may fail.")

    return stats

# Periodic monitoring
def continuous_monitor(index_name: str, interval_seconds: int = 300):
    """Monitor index every N seconds."""
    while True:
        try:
            stats = monitor_index(index_name)
            # Send metrics to your monitoring system (Datadog, Prometheus, etc.)
            emit_metric("pinecone.vector_count", stats.total_vector_count)
            emit_metric("pinecone.index_fullness", stats.index_fullness)
        except Exception as e:
            print(f"Monitoring error: {e}")
        time.sleep(interval_seconds)

def emit_metric(name: str, value: float):
    """Placeholder for your metrics system."""
    print(f"  METRIC: {name} = {value}")
```

### Query Performance Monitoring

```python
import time
import numpy as np

def measure_query_latency(index, num_queries: int = 100, dimension: int = 1536):
    """Measure query latency for monitoring."""
    latencies = []

    for _ in range(num_queries):
        query_vector = np.random.rand(dimension).tolist()
        start = time.perf_counter()
        index.query(vector=query_vector, top_k=10, include_metadata=False)
        latencies.append((time.perf_counter() - start) * 1000)

    latencies_arr = np.array(latencies)
    return {
        "p50_ms": float(np.percentile(latencies_arr, 50)),
        "p95_ms": float(np.percentile(latencies_arr, 95)),
        "p99_ms": float(np.percentile(latencies_arr, 99)),
        "mean_ms": float(np.mean(latencies_arr)),
        "qps": 1000.0 / float(np.mean(latencies_arr)),
    }

# Usage
index = pc.Index("documents-prod")
perf = measure_query_latency(index)
print(f"p50: {perf['p50_ms']:.1f}ms, p99: {perf['p99_ms']:.1f}ms, QPS: {perf['qps']:.0f}")
```

---

## Cost Optimization

### Serverless Pricing (as of 2025)

| Component | Cost |
|-----------|------|
| Read units | $8.25 per 1M read units |
| Write units | $2.00 per 1M write units |
| Storage | $0.33 per GB/month |

**Read units per query**: depends on dimensions and top_k. Approximate:
- 1536 dims, top_k=10: ~6 read units per query
- 1536 dims, top_k=100: ~60 read units per query

**Cost estimation** (1M vectors, 1536 dims, 10K queries/day):

```python
def estimate_monthly_cost(
    num_vectors: int,
    dimensions: int,
    queries_per_day: int,
    top_k: int = 10,
    upserts_per_day: int = 0,
):
    """Estimate monthly Pinecone serverless cost."""
    # Storage (rough: 4 bytes * dims per vector + metadata overhead)
    storage_gb = (num_vectors * dimensions * 4 * 1.5) / (1024 ** 3)
    storage_cost = storage_gb * 0.33

    # Read units (approximate: top_k * 0.6 read units per query)
    ru_per_query = top_k * 0.6
    monthly_queries = queries_per_day * 30
    read_cost = (monthly_queries * ru_per_query / 1_000_000) * 8.25

    # Write units (approximate: 1 write unit per vector upserted)
    monthly_upserts = upserts_per_day * 30
    write_cost = (monthly_upserts / 1_000_000) * 2.00

    total = storage_cost + read_cost + write_cost

    print(f"Storage: ${storage_cost:.2f}/month ({storage_gb:.1f} GB)")
    print(f"Reads:   ${read_cost:.2f}/month ({monthly_queries:,} queries)")
    print(f"Writes:  ${write_cost:.2f}/month ({monthly_upserts:,} upserts)")
    print(f"Total:   ${total:.2f}/month")
    return total

# Examples
print("=== 1M vectors, 10K queries/day ===")
estimate_monthly_cost(1_000_000, 1536, 10_000)
print()
print("=== 5M vectors, 50K queries/day ===")
estimate_monthly_cost(5_000_000, 1536, 50_000)
```

### Cost Optimization Tips

1. **Reduce dimensions**: use Matryoshka embeddings at 512 or 768 dims instead of 1536. This cuts storage by 50-67%.
2. **Use `include_values=False`**: returning vector values increases response size and read units.
3. **Batch queries**: if you have multiple queries, use list operations instead of individual calls where possible.
4. **Clean up unused namespaces**: empty namespaces cost nothing, but forgotten data in old namespaces costs storage.
5. **Use collections for archival**: collections cost storage only, no compute. Archive old indexes as collections.

---

## Common Pitfalls

1. **Not enabling deletion protection on production indexes**: a single `pc.delete_index()` call destroys all data irreversibly. Always enable `deletion_protection`.

2. **Using pod-based when serverless is sufficient**: serverless is cheaper for variable workloads. Pod-based only makes sense for sustained >100 QPS with latency SLAs.

3. **Not warming up serverless indexes**: cold start adds 100-500ms. Send periodic queries to keep the index warm if latency matters.

4. **Upserting vectors one at a time**: single upserts are 10-50x slower than batched. Always batch 100+ vectors per call.

5. **Storing large metadata payloads**: metadata is loaded into memory during search. Keep metadata lean (< 40 KB per vector). Store large text in an external database and reference by ID.

6. **Not specifying `indexed` metadata for pod-based indexes**: by default, all metadata fields are indexed. Indexing high-cardinality fields wastes memory. Explicitly list only the fields you filter on.

7. **Expecting strong consistency**: Pinecone is eventually consistent. After upsert, wait a few seconds before querying for the new vectors.

---

## References

- Pinecone documentation: https://docs.pinecone.io/
- Pinecone pricing: https://www.pinecone.io/pricing/
- Pinecone Python client: https://github.com/pinecone-io/pinecone-python-client
- Pinecone serverless architecture: https://www.pinecone.io/blog/serverless/
- Pinecone collections: https://docs.pinecone.io/guides/indexes/back-up-an-index
- Pinecone monitoring: https://docs.pinecone.io/guides/operations/monitoring
