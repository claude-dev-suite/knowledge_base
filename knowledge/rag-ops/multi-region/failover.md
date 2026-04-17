# Multi-Region RAG -- Failover: Active-Active vs Active-Passive and Consistency

## Overview

This guide covers failover strategies for multi-region RAG systems: active-active vs active-passive architectures, consistency tradeoffs, automated failover implementation, and recovery procedures. The choice between these patterns determines your system's availability, data consistency, and operational complexity.

---

## Active-Active vs Active-Passive

### Comparison

| Aspect | Active-Active | Active-Passive |
|--------|--------------|----------------|
| Write path | Both regions accept writes | Only primary accepts writes |
| Read path | Both regions serve reads | Primary serves reads; secondary for overflow |
| Failover time | Near-zero (already serving) | Seconds to minutes (promote secondary) |
| Data consistency | Eventual (replication lag) | Strong (single writer) |
| Conflict resolution | Required (concurrent writes) | Not needed |
| Cost | 2x infrastructure | 1.5-2x infrastructure |
| Complexity | High | Medium |
| Best for | Global users, zero-downtime requirements | Cost-sensitive, strong consistency needs |

### Active-Passive Implementation

```python
import time
import logging
from enum import Enum

logger = logging.getLogger("failover")


class RegionRole(Enum):
    PRIMARY = "primary"
    SECONDARY = "secondary"
    PROMOTING = "promoting"


class ActivePassiveFailover:
    """Manages active-passive failover for multi-region RAG."""

    def __init__(
        self,
        primary_region: str,
        secondary_region: str,
        primary_client: object,
        secondary_client: object,
        health_check_fn: callable,
        health_check_interval: int = 10,
        failure_threshold: int = 3,
    ):
        self.regions = {
            primary_region: {
                "client": primary_client,
                "role": RegionRole.PRIMARY,
                "consecutive_failures": 0,
            },
            secondary_region: {
                "client": secondary_client,
                "role": RegionRole.SECONDARY,
                "consecutive_failures": 0,
            },
        }
        self.primary_region = primary_region
        self.secondary_region = secondary_region
        self.health_check_fn = health_check_fn
        self.health_check_interval = health_check_interval
        self.failure_threshold = failure_threshold
        self._failover_in_progress = False

    @property
    def active_region(self) -> str:
        """Return the current active (primary) region."""
        for region, info in self.regions.items():
            if info["role"] == RegionRole.PRIMARY:
                return region
        return self.primary_region

    @property
    def active_client(self):
        """Return the client for the active region."""
        return self.regions[self.active_region]["client"]

    def search(self, collection: str, query_vector: list, limit: int = 10):
        """Search the active region, fallback to secondary on error."""
        try:
            return self.active_client.search(
                collection_name=collection,
                query_vector=query_vector,
                limit=limit,
            )
        except Exception as e:
            logger.warning(f"Search failed in {self.active_region}: {e}")
            # Try secondary
            fallback_region = (
                self.secondary_region
                if self.active_region == self.primary_region
                else self.primary_region
            )
            try:
                return self.regions[fallback_region]["client"].search(
                    collection_name=collection,
                    query_vector=query_vector,
                    limit=limit,
                )
            except Exception as e2:
                logger.error(f"Search failed in fallback {fallback_region}: {e2}")
                raise

    def check_health(self) -> dict:
        """Check health of all regions and trigger failover if needed."""
        results = {}

        for region, info in self.regions.items():
            try:
                healthy = self.health_check_fn(info["client"])
                info["consecutive_failures"] = 0 if healthy else info["consecutive_failures"] + 1
                results[region] = {
                    "healthy": healthy,
                    "role": info["role"].value,
                    "consecutive_failures": info["consecutive_failures"],
                }
            except Exception:
                info["consecutive_failures"] += 1
                results[region] = {
                    "healthy": False,
                    "role": info["role"].value,
                    "consecutive_failures": info["consecutive_failures"],
                }

        # Check if primary needs failover
        primary_info = self.regions[self.active_region]
        if (
            primary_info["consecutive_failures"] >= self.failure_threshold
            and not self._failover_in_progress
        ):
            self._trigger_failover()

        return results

    def _trigger_failover(self):
        """Promote secondary to primary."""
        self._failover_in_progress = True
        old_primary = self.active_region
        new_primary = (
            self.secondary_region
            if old_primary == self.primary_region
            else self.primary_region
        )

        logger.critical(
            f"FAILOVER: {old_primary} -> {new_primary}. "
            f"Reason: {self.regions[old_primary]['consecutive_failures']} consecutive failures"
        )

        # Swap roles
        self.regions[old_primary]["role"] = RegionRole.SECONDARY
        self.regions[new_primary]["role"] = RegionRole.PRIMARY

        self._failover_in_progress = False
        logger.info(f"Failover complete. New primary: {new_primary}")

    def manual_failover(self, new_primary: str):
        """Manually trigger failover to a specific region."""
        if new_primary not in self.regions:
            raise ValueError(f"Unknown region: {new_primary}")

        old_primary = self.active_region
        if old_primary == new_primary:
            logger.info(f"{new_primary} is already primary")
            return

        logger.info(f"Manual failover: {old_primary} -> {new_primary}")
        self.regions[old_primary]["role"] = RegionRole.SECONDARY
        self.regions[new_primary]["role"] = RegionRole.PRIMARY

    def manual_failback(self):
        """Failback to the original primary region."""
        self.manual_failover(self.primary_region)
```

### Active-Active Implementation

```python
class ActiveActiveRouter:
    """Routes requests to the nearest healthy region in active-active setup."""

    def __init__(self, regions: dict[str, object]):
        """
        regions: {"us-east-1": client_us, "eu-west-1": client_eu}
        """
        self.regions = regions
        self.health_status: dict[str, bool] = {r: True for r in regions}

    def search(
        self,
        collection: str,
        query_vector: list,
        preferred_region: str,
        limit: int = 10,
    ):
        """Search the preferred region, fallback to other healthy regions."""
        # Try preferred region first
        if self.health_status.get(preferred_region, False):
            try:
                return self.regions[preferred_region].search(
                    collection_name=collection,
                    query_vector=query_vector,
                    limit=limit,
                )
            except Exception as e:
                logger.warning(f"Search failed in {preferred_region}: {e}")
                self.health_status[preferred_region] = False

        # Fallback to any healthy region
        for region, client in self.regions.items():
            if region == preferred_region:
                continue
            if not self.health_status.get(region, False):
                continue
            try:
                return client.search(
                    collection_name=collection,
                    query_vector=query_vector,
                    limit=limit,
                )
            except Exception as e:
                logger.warning(f"Fallback search failed in {region}: {e}")
                self.health_status[region] = False

        raise RuntimeError("All regions are unhealthy")

    def write(
        self,
        collection: str,
        points: list,
        source_region: str,
    ) -> dict:
        """Write to source region synchronously, replicate async."""
        # Write to source (synchronous)
        result = self.regions[source_region].upsert(
            collection_name=collection,
            points=points,
        )

        # Replicate to other regions (async, fire-and-forget)
        for region, client in self.regions.items():
            if region == source_region:
                continue
            try:
                client.upsert(collection_name=collection, points=points)
            except Exception as e:
                logger.error(f"Replication to {region} failed: {e}")
                # Queue for retry
                self._queue_retry(region, collection, points)

        return {"source": source_region, "result": result}

    def _queue_retry(self, region, collection, points):
        """Queue failed replication for retry."""
        # In production, use a persistent queue (Redis, SQS, Kafka)
        pass
```

---

## Consistency Tradeoffs

### CAP Theorem Applied to Multi-Region RAG

```
            Consistency
               /\
              /  \
             /    \
            / CP   \
           /        \
          /  Choose  \
         /   Two     \
        /              \
       /________________\
 Availability          Partition
                       Tolerance
```

For multi-region RAG, network partitions WILL happen. You must choose:

- **CP (Consistency + Partition tolerance)**: Active-passive. Writes go to one region only. During partition, secondary cannot accept writes. Data is always consistent but availability is reduced.

- **AP (Availability + Partition tolerance)**: Active-active. Both regions accept writes during partition. Data may be inconsistent (conflicts) but the system stays available.

### Conflict Resolution Strategies

```python
from datetime import datetime


def last_write_wins(existing: dict, incoming: dict) -> dict:
    """Resolve conflicts by keeping the most recent write."""
    existing_ts = datetime.fromisoformat(existing.get("updated_at", "1970-01-01"))
    incoming_ts = datetime.fromisoformat(incoming.get("updated_at", "1970-01-01"))
    return incoming if incoming_ts >= existing_ts else existing


def merge_metadata(existing: dict, incoming: dict) -> dict:
    """Merge metadata from both versions."""
    merged = {**existing}
    merged["metadata"] = {
        **existing.get("metadata", {}),
        **incoming.get("metadata", {}),
    }
    # Keep the newest embedding
    if incoming.get("updated_at", "") > existing.get("updated_at", ""):
        merged["embedding"] = incoming["embedding"]
        merged["text"] = incoming["text"]
    merged["updated_at"] = max(
        existing.get("updated_at", ""),
        incoming.get("updated_at", ""),
    )
    return merged
```

### Eventual Consistency Verification

```python
import asyncio
import hashlib


async def verify_consistency(
    regions: dict[str, object],
    collection: str,
    sample_size: int = 100,
) -> dict:
    """Verify data consistency across regions by sampling."""
    checksums = {}

    for region, client in regions.items():
        # Get a sample of documents
        docs = client.scroll(
            collection_name=collection,
            limit=sample_size,
            with_vectors=True,
        )

        # Compute checksum of IDs + vectors
        doc_hashes = []
        for doc in docs[0]:  # scroll returns (points, next_offset)
            doc_hash = hashlib.sha256(
                f"{doc.id}:{doc.vector[:5]}".encode()
            ).hexdigest()[:16]
            doc_hashes.append(doc_hash)

        checksums[region] = {
            "count": len(doc_hashes),
            "checksum": hashlib.sha256(
                "".join(sorted(doc_hashes)).encode()
            ).hexdigest()[:32],
        }

    # Compare checksums
    all_checksums = set(r["checksum"] for r in checksums.values())
    consistent = len(all_checksums) == 1

    return {
        "consistent": consistent,
        "regions": checksums,
        "mismatch": not consistent,
    }
```

---

## Recovery Procedures

### Failback After Regional Recovery

```python
class FailbackProcedure:
    """Procedure to failback to original primary after recovery."""

    def __init__(self, failover_manager, sync_service):
        self.failover = failover_manager
        self.sync = sync_service

    async def execute_failback(self, original_primary: str, collection: str):
        """
        Steps:
        1. Verify original primary is healthy
        2. Sync any writes that happened during failover
        3. Verify data consistency
        4. Switch traffic back to original primary
        5. Monitor for stability
        """
        logger.info(f"Starting failback to {original_primary}")

        # Step 1: Verify health
        health = self.failover.check_health()
        if not health[original_primary]["healthy"]:
            raise RuntimeError(f"{original_primary} is not healthy yet")

        # Step 2: Sync writes from current primary to original
        current_primary = self.failover.active_region
        logger.info(f"Syncing writes from {current_primary} to {original_primary}")
        await self.sync.sync_all(collection)

        # Step 3: Verify consistency
        consistency = await verify_consistency(
            self.failover.regions,
            collection,
        )
        if not consistency["consistent"]:
            logger.warning(f"Data inconsistency detected: {consistency}")
            # Continue anyway with a warning -- manual review may be needed

        # Step 4: Failback
        self.failover.manual_failback()
        logger.info(f"Failback complete. Primary is now {original_primary}")

        # Step 5: Monitor
        logger.info("Monitoring for 5 minutes post-failback")
        for i in range(5):
            await asyncio.sleep(60)
            health = self.failover.check_health()
            logger.info(f"Post-failback health check {i+1}/5: {health}")

        return {"status": "complete", "primary": original_primary}
```

### Data Recovery After Split-Brain

```python
async def resolve_split_brain(
    region_a_client,
    region_b_client,
    collection: str,
    conflict_resolution: str = "last_write_wins",
):
    """
    Resolve data conflicts after a network partition (split-brain).
    Both regions accepted writes during the partition.
    """
    # Get all documents from both regions
    docs_a = {}
    docs_b = {}

    # Scroll through region A
    offset = None
    while True:
        points, offset = region_a_client.scroll(
            collection_name=collection,
            limit=100,
            offset=offset,
            with_payload=True,
            with_vectors=True,
        )
        for p in points:
            docs_a[p.id] = {
                "id": p.id,
                "vector": p.vector,
                "payload": p.payload,
                "updated_at": p.payload.get("updated_at", ""),
            }
        if offset is None:
            break

    # Scroll through region B
    offset = None
    while True:
        points, offset = region_b_client.scroll(
            collection_name=collection,
            limit=100,
            offset=offset,
            with_payload=True,
            with_vectors=True,
        )
        for p in points:
            docs_b[p.id] = {
                "id": p.id,
                "vector": p.vector,
                "payload": p.payload,
                "updated_at": p.payload.get("updated_at", ""),
            }
        if offset is None:
            break

    # Find conflicts
    all_ids = set(docs_a.keys()) | set(docs_b.keys())
    only_a = set(docs_a.keys()) - set(docs_b.keys())
    only_b = set(docs_b.keys()) - set(docs_a.keys())
    both = set(docs_a.keys()) & set(docs_b.keys())

    conflicts = []
    for doc_id in both:
        if docs_a[doc_id]["updated_at"] != docs_b[doc_id]["updated_at"]:
            conflicts.append(doc_id)

    logger.info(
        f"Split-brain analysis: {len(only_a)} only in A, "
        f"{len(only_b)} only in B, {len(conflicts)} conflicts"
    )

    # Resolve conflicts
    resolved = {}
    for doc_id in conflicts:
        if conflict_resolution == "last_write_wins":
            resolved[doc_id] = last_write_wins(docs_a[doc_id], docs_b[doc_id])
        else:
            resolved[doc_id] = merge_metadata(docs_a[doc_id], docs_b[doc_id])

    # Apply resolved documents to both regions
    # ... (upsert resolved + missing documents)

    return {
        "only_a": len(only_a),
        "only_b": len(only_b),
        "conflicts_resolved": len(conflicts),
        "resolution_strategy": conflict_resolution,
    }
```

---

## Monitoring and Alerting

### Key Failover Metrics

| Metric | Normal | Warning | Critical |
|--------|--------|---------|----------|
| Replication lag | < 5s | 5-60s | > 60s |
| Health check failures | 0 | 1-2 consecutive | 3+ consecutive |
| Document count diff | < 0.1% | 0.1-1% | > 1% |
| Cross-region latency | < 100ms | 100-500ms | > 500ms |
| Failover events (24h) | 0 | 1 | 2+ |

---

## Common Pitfalls

1. **Not testing failover regularly**: Schedule monthly failover drills. Many teams discover their failover is broken only during a real outage.

2. **Split-brain without conflict resolution**: If you run active-active without a conflict resolution strategy, data loss is inevitable during network partitions.

3. **Failing back too quickly**: After an outage, the recovering region may still be unstable. Wait for several successful health checks (5-10 minutes) before failing back.

4. **Not persisting failover state**: If the failover controller crashes, it must know the current primary when it restarts. Persist failover state in a durable store (Redis, DynamoDB, etcd).

5. **Ignoring partial outages**: A region may be reachable for health checks but unable to serve search queries. Include functional health checks (execute a test query) not just connectivity checks.

6. **Automatic failback without verification**: Never automatically failback without verifying data consistency. Manual verification and approval should be required for failback.

---

## References

- AWS multi-region failover: https://aws.amazon.com/architecture/multi-region/
- CAP theorem: https://en.wikipedia.org/wiki/CAP_theorem
- Qdrant distributed mode: https://qdrant.tech/documentation/guides/distributed_deployment/
- MongoDB Atlas global clusters: https://www.mongodb.com/docs/atlas/global-clusters/
