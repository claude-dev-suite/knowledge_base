# Multi-Region RAG -- Geo-Replicated Architecture

## Overview

Multi-region RAG deploys retrieval and generation infrastructure across geographic regions to achieve low latency for global users, compliance with data residency regulations, and resilience against regional outages. This guide covers the architectural patterns, data replication strategies, and design decisions for building geo-distributed RAG systems.

---

## Why Multi-Region

### Latency

Vector search latency is dominated by network round-trips, not computation:

| User Location | Single-Region (us-east-1) | Multi-Region (nearest) |
|---------------|--------------------------|------------------------|
| New York | 5-10ms | 5-10ms (us-east-1) |
| London | 80-120ms | 5-10ms (eu-west-1) |
| Tokyo | 150-200ms | 5-10ms (ap-northeast-1) |
| Sydney | 200-250ms | 5-10ms (ap-southeast-2) |

For RAG pipelines, total latency includes embedding (5-50ms), search (5-20ms), and generation (200-2000ms). Multi-region reduces the first two components but generation latency depends on the LLM provider's location.

### Data Residency

Regulations like GDPR (EU), CCPA (California), and PDPA (Singapore) may require that user data stays within specific geographic boundaries. Multi-region RAG ensures embeddings and source documents remain in compliant regions.

### Availability

Single-region deployments have a single point of failure. Multi-region provides resilience:

| Configuration | Availability | RTO | RPO |
|---------------|-------------|-----|-----|
| Single region | 99.9% (8.7h downtime/year) | Hours | Depends on backup |
| Active-passive | 99.95% (4.4h/year) | Minutes | Minutes-hours |
| Active-active | 99.99% (52min/year) | Seconds | Near-zero |

---

## Architecture Patterns

### Pattern 1: Active-Passive (Warm Standby)

```
Primary Region (us-east-1)          Secondary Region (eu-west-1)
+---------------------+            +---------------------+
| Load Balancer       |            | Load Balancer       |
+----------+----------+            +----------+----------+
           |                                  |
+----------v----------+            +----------v----------+
| RAG Service         |            | RAG Service         |
| (handles all        |  replicate | (standby, receives  |
|  traffic)           | ---------> |  replicated data)   |
+----------+----------+            +----------+----------+
           |                                  |
+----------v----------+            +----------v----------+
| Vector DB           |  async     | Vector DB           |
| (primary)           | ---------> | (replica, read-only)|
+---------------------+  repl     +---------------------+
```

**Characteristics:**
- All writes go to primary
- Reads go to primary (or secondary for read scaling)
- Failover: promote secondary to primary on outage
- Simple to implement, but secondary may have stale data during failover

### Pattern 2: Active-Active (Multi-Master)

```
Region A (us-east-1)               Region B (eu-west-1)
+---------------------+            +---------------------+
| Global LB / GeoDNS  | <--------> | Global LB / GeoDNS  |
+----------+----------+            +----------+----------+
           |                                  |
+----------v----------+            +----------v----------+
| RAG Service         | <--------> | RAG Service         |
| (handles local      |   sync    | (handles local      |
|  traffic)           |            |  traffic)           |
+----------+----------+            +----------+----------+
           |                                  |
+----------v----------+            +----------v----------+
| Vector DB           | <--------> | Vector DB           |
| (read-write)        |   repl    | (read-write)        |
+---------------------+            +---------------------+
```

**Characteristics:**
- Both regions handle reads AND writes
- Documents ingested in any region replicate to all regions
- Eventual consistency (writes in region A visible in region B after replication delay)
- Higher complexity, but best latency and availability

### Pattern 3: Region-Specific (Data Residency)

```
EU Region (eu-west-1)              US Region (us-east-1)
+---------------------+            +---------------------+
| EU Load Balancer    |            | US Load Balancer    |
+----------+----------+            +----------+----------+
           |                                  |
+----------v----------+            +----------v----------+
| RAG Service         |            | RAG Service         |
| (EU data only)      |            | (US data only)      |
+----------+----------+            +----------+----------+
           |                                  |
+----------v----------+            +----------v----------+
| Vector DB           |            | Vector DB           |
| (EU documents)      |            | (US documents)      |
+---------------------+            +---------------------+

NO replication between regions (data residency compliance)
Global routing based on user/tenant region assignment
```

**Characteristics:**
- Strict data isolation between regions
- Each region has its own document corpus
- No cross-region queries
- Compliant with GDPR, CCPA, etc.

---

## Data Replication Strategies

### Vector Database Replication

| Strategy | Latency | Consistency | Complexity |
|----------|---------|-------------|------------|
| Synchronous replication | High (write waits for all replicas) | Strong | Medium |
| Asynchronous replication | Low (write returns immediately) | Eventual (seconds-minutes) | Medium |
| Application-level dual-write | Low | Eventual (depends on app) | High |
| Periodic full sync | N/A | Stale (hours) | Low |

### Application-Level Dual-Write

```python
import asyncio
from typing import Optional


class MultiRegionVectorStore:
    """Write to multiple vector stores across regions."""

    def __init__(self, stores: dict[str, object], primary_region: str):
        """
        stores: {"us-east-1": qdrant_client_us, "eu-west-1": qdrant_client_eu}
        primary_region: region that handles conflicting writes
        """
        self.stores = stores
        self.primary_region = primary_region

    async def upsert(
        self,
        collection: str,
        points: list,
        source_region: Optional[str] = None,
    ) -> dict:
        """Upsert to all regions. Primary write is synchronous, replicas are async."""
        results = {}

        # Synchronous write to primary
        primary_store = self.stores[self.primary_region]
        results[self.primary_region] = await self._upsert_region(
            primary_store, collection, points
        )

        # Async write to replicas
        replica_tasks = []
        for region, store in self.stores.items():
            if region == self.primary_region:
                continue
            if region == source_region:
                continue  # skip source to avoid loops
            replica_tasks.append(
                self._upsert_region_safe(region, store, collection, points)
            )

        replica_results = await asyncio.gather(*replica_tasks)
        for region, result in replica_results:
            results[region] = result

        return results

    async def _upsert_region(self, store, collection, points):
        """Upsert to a single region."""
        return store.upsert(collection_name=collection, points=points)

    async def _upsert_region_safe(self, region, store, collection, points):
        """Upsert with error handling for replica regions."""
        try:
            result = await self._upsert_region(store, collection, points)
            return region, {"status": "ok", "result": result}
        except Exception as e:
            return region, {"status": "error", "error": str(e)}

    async def search(
        self,
        collection: str,
        query_vector: list[float],
        limit: int = 10,
        region: Optional[str] = None,
    ) -> list:
        """Search in a specific region (defaults to primary)."""
        target_region = region or self.primary_region
        store = self.stores[target_region]

        return store.search(
            collection_name=collection,
            query_vector=query_vector,
            limit=limit,
        )
```

---

## LLM Provider Routing

LLM providers have region-specific endpoints. Route generation requests to the nearest endpoint:

```python
LLM_ENDPOINTS = {
    "us-east-1": {
        "openai": "https://api.openai.com/v1",           # US-based
        "anthropic": "https://api.anthropic.com",         # US-based
    },
    "eu-west-1": {
        "openai": "https://api.openai.com/v1",           # No EU endpoint (yet)
        "anthropic": "https://api.anthropic.com",         # No EU endpoint (yet)
        "azure_openai": "https://your-eu-resource.openai.azure.com",  # EU Azure region
    },
    "ap-northeast-1": {
        "azure_openai": "https://your-japan-resource.openai.azure.com",
    },
}


def get_llm_endpoint(region: str, provider: str) -> str:
    """Get the nearest LLM endpoint for a region."""
    region_endpoints = LLM_ENDPOINTS.get(region, {})

    if provider in region_endpoints:
        return region_endpoints[provider]

    # Fallback to default endpoint
    return LLM_ENDPOINTS["us-east-1"].get(provider, "")
```

---

## Monitoring Multi-Region Systems

### Key Metrics

| Metric | Purpose | Alert Threshold |
|--------|---------|----------------|
| Replication lag (seconds) | Data freshness across regions | > 60 seconds |
| Cross-region query rate | Detect routing issues | > 5% of queries cross-region |
| Per-region error rate | Detect regional outages | > 1% error rate |
| Per-region latency P95 | Detect performance degradation | > 2x normal |
| Document count delta | Detect replication failures | > 1% difference between regions |

---

## Common Pitfalls

1. **Assuming LLM providers have regional endpoints**: Most LLM APIs (OpenAI, Anthropic) are US-based only. For true EU data residency, use Azure OpenAI with an EU region or self-hosted models.

2. **Not handling replication lag**: In active-active setups, a document ingested in region A may not be searchable in region B for seconds to minutes. Design your UX for eventual consistency.

3. **Cross-region writes without conflict resolution**: If two regions write to the same document ID simultaneously, one write may be lost. Use last-write-wins with vector clocks or single-writer-per-document assignment.

4. **Over-replicating**: Not all data needs to be in all regions. If a tenant is only in EU, do not replicate their data to US regions -- it wastes storage and may violate data residency.

5. **Not testing failover**: Multi-region is useless if failover does not work. Regularly test promoting secondary to primary, routing changes, and recovery procedures.

6. **Ignoring embedding model consistency**: All regions must use the same embedding model and version. A model upgrade in one region but not another produces incompatible embeddings.

---

## References

- AWS multi-region architecture: https://aws.amazon.com/architecture/multi-region/
- Pinecone multi-region: https://docs.pinecone.io/guides/indexes/create-an-index
- Qdrant distributed deployment: https://qdrant.tech/documentation/guides/distributed_deployment/
- MongoDB Atlas global clusters: https://www.mongodb.com/docs/atlas/global-clusters/
