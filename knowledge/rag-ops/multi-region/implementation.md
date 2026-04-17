# Multi-Region RAG -- Implementation: Vector DB Replication and LLM Routing

## Overview

This guide covers implementing multi-region RAG with specific vector database configurations (Pinecone, Qdrant, MongoDB Atlas), LLM provider routing, geo-aware request handling, and the infrastructure code for a working multi-region deployment.

---

## Pinecone Multi-Region

Pinecone serverless supports multi-region through collections and cross-region index copying:

```python
from pinecone import Pinecone, ServerlessSpec

pc = Pinecone(api_key="your-api-key")

# Create index in US
pc.create_index(
    name="knowledge-base-us",
    dimension=1536,
    metric="cosine",
    spec=ServerlessSpec(cloud="aws", region="us-east-1"),
)

# Create index in EU
pc.create_index(
    name="knowledge-base-eu",
    dimension=1536,
    metric="cosine",
    spec=ServerlessSpec(cloud="aws", region="eu-west-1"),
)

# Application-level replication: write to both
def upsert_multi_region(vectors, namespace="default"):
    """Upsert vectors to all regional indexes."""
    us_index = pc.Index("knowledge-base-us")
    eu_index = pc.Index("knowledge-base-eu")

    us_result = us_index.upsert(vectors=vectors, namespace=namespace)
    eu_result = eu_index.upsert(vectors=vectors, namespace=namespace)

    return {"us": us_result, "eu": eu_result}


def search_nearest_region(query_vector, region="us-east-1", top_k=10, namespace="default"):
    """Search in the nearest regional index."""
    index_name = {
        "us-east-1": "knowledge-base-us",
        "eu-west-1": "knowledge-base-eu",
    }.get(region, "knowledge-base-us")

    index = pc.Index(index_name)
    return index.query(
        vector=query_vector,
        top_k=top_k,
        namespace=namespace,
        include_metadata=True,
    )
```

---

## Qdrant Multi-Region (Distributed Mode)

Qdrant supports distributed deployment with sharding and replication:

```python
from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance, VectorParams, PointStruct,
    ShardingMethod, ReplicationFactor,
)

# Connect to Qdrant clusters in different regions
clients = {
    "us-east-1": QdrantClient(url="https://qdrant-us.example.com:6333"),
    "eu-west-1": QdrantClient(url="https://qdrant-eu.example.com:6333"),
}


def create_collection_all_regions(
    collection_name: str,
    vector_size: int = 1536,
):
    """Create a collection in all regional clusters."""
    for region, client in clients.items():
        client.create_collection(
            collection_name=collection_name,
            vectors_config=VectorParams(
                size=vector_size,
                distance=Distance.COSINE,
            ),
            replication_factor=2,   # replicas within the cluster
            write_consistency_factor=1,
        )
        print(f"Created collection in {region}")


def upsert_all_regions(
    collection_name: str,
    points: list[PointStruct],
    batch_size: int = 100,
):
    """Upsert points to all regional clusters."""
    results = {}
    for region, client in clients.items():
        try:
            for i in range(0, len(points), batch_size):
                batch = points[i:i + batch_size]
                client.upsert(
                    collection_name=collection_name,
                    points=batch,
                )
            results[region] = {"status": "ok", "count": len(points)}
        except Exception as e:
            results[region] = {"status": "error", "error": str(e)}
    return results


def search_region(
    collection_name: str,
    query_vector: list[float],
    region: str,
    limit: int = 10,
):
    """Search in a specific region."""
    client = clients.get(region)
    if not client:
        raise ValueError(f"Unknown region: {region}")

    return client.search(
        collection_name=collection_name,
        query_vector=query_vector,
        limit=limit,
    )
```

---

## MongoDB Atlas Multi-Region

MongoDB Atlas Global Clusters provide automatic geo-sharding:

```python
from pymongo import MongoClient
from pymongo.operations import SearchIndexModel

# Atlas connection string with multi-region cluster
client = MongoClient(
    "mongodb+srv://user:pass@global-cluster.mongodb.net/?retryWrites=true"
)

db = client["rag_database"]
collection = db["documents"]

# Zone sharding: route documents to regions based on metadata
# This is configured in Atlas UI or CLI, not in application code
# Key: documents with region="eu" go to EU nodes, region="us" go to US nodes

# Insert document with region tag
def insert_document(doc_id, text, embedding, region="us"):
    """Insert a document routed to a specific region."""
    collection.insert_one({
        "_id": doc_id,
        "text": text,
        "embedding": embedding,
        "region": region,         # shard key for zone sharding
        "metadata": {"source": "knowledge_base"},
    })


# Create Atlas Vector Search index
# (done once via Atlas UI or API)
search_index = SearchIndexModel(
    definition={
        "fields": [
            {
                "type": "vector",
                "path": "embedding",
                "numDimensions": 1536,
                "similarity": "cosine",
            },
            {
                "type": "filter",
                "path": "region",
            },
        ],
    },
    name="vector_search_index",
    type="vectorSearch",
)


def search_documents(query_vector, region=None, limit=10):
    """Vector search with optional region filter."""
    pipeline = [
        {
            "$vectorSearch": {
                "index": "vector_search_index",
                "path": "embedding",
                "queryVector": query_vector,
                "numCandidates": limit * 10,
                "limit": limit,
            }
        },
        {
            "$project": {
                "text": 1,
                "region": 1,
                "score": {"$meta": "vectorSearchScore"},
            }
        },
    ]

    # Add region filter if specified
    if region:
        pipeline[0]["$vectorSearch"]["filter"] = {"region": {"$eq": region}}

    return list(collection.aggregate(pipeline))
```

---

## Geo-Aware Request Router

```python
from fastapi import FastAPI, Request, Header
from typing import Optional
import geoip2.database

app = FastAPI()

# GeoIP database for region detection
geo_reader = geoip2.database.Reader("./GeoLite2-City.mmdb")

REGION_MAP = {
    "NA": "us-east-1",    # North America
    "EU": "eu-west-1",    # Europe
    "AS": "ap-northeast-1",  # Asia
    "OC": "ap-southeast-2",  # Oceania
    "SA": "us-east-1",    # South America (fallback to US)
    "AF": "eu-west-1",    # Africa (fallback to EU)
}


def detect_region(request: Request, x_region: Optional[str] = Header(None)) -> str:
    """Detect the best region for a request."""
    # Explicit region header takes priority
    if x_region and x_region in REGION_MAP.values():
        return x_region

    # GeoIP detection
    client_ip = request.client.host
    try:
        geo = geo_reader.city(client_ip)
        continent = geo.continent.code
        return REGION_MAP.get(continent, "us-east-1")
    except Exception:
        return "us-east-1"  # default


@app.post("/search")
async def search(request: Request, query: str, top_k: int = 10):
    region = detect_region(request)

    # Route search to nearest region
    results = search_region(
        collection_name="knowledge_base",
        query_vector=embed_query(query),
        region=region,
        limit=top_k,
    )

    return {"region": region, "results": results}
```

---

## Document Sync Service

```python
import asyncio
import logging
from datetime import datetime, timedelta

logger = logging.getLogger("sync_service")


class DocumentSyncService:
    """Synchronize documents across regions on a schedule."""

    def __init__(self, regions: dict[str, object], primary: str):
        self.regions = regions
        self.primary = primary

    async def sync_all(self, collection: str, since: datetime = None):
        """Sync documents from primary to all replicas."""
        if since is None:
            since = datetime.utcnow() - timedelta(hours=1)

        # Get documents updated since last sync from primary
        primary_client = self.regions[self.primary]
        updated_docs = self._get_updated_docs(primary_client, collection, since)

        if not updated_docs:
            logger.info("No documents to sync")
            return

        logger.info(f"Syncing {len(updated_docs)} documents to replica regions")

        # Sync to each replica
        for region, client in self.regions.items():
            if region == self.primary:
                continue

            try:
                self._upsert_docs(client, collection, updated_docs)
                logger.info(f"Synced {len(updated_docs)} docs to {region}")
            except Exception as e:
                logger.error(f"Failed to sync to {region}: {e}")

    def _get_updated_docs(self, client, collection, since):
        """Get documents updated after a given timestamp."""
        # Implementation depends on vector DB
        # Qdrant: scroll with filter on timestamp field
        # Pinecone: list with filter
        # MongoDB: find with $gt on updated_at
        pass

    def _upsert_docs(self, client, collection, docs):
        """Upsert documents to a regional client."""
        pass

    async def run_continuous(self, collection: str, interval_seconds: int = 300):
        """Run sync continuously every N seconds."""
        last_sync = datetime.utcnow() - timedelta(hours=24)  # initial full sync

        while True:
            try:
                await self.sync_all(collection, since=last_sync)
                last_sync = datetime.utcnow()
            except Exception as e:
                logger.error(f"Sync failed: {e}")

            await asyncio.sleep(interval_seconds)
```

---

## Health Check Across Regions

```python
import httpx
import asyncio
import time


async def multi_region_health_check(
    regions: dict[str, str],  # {"us-east-1": "https://rag-us.example.com", ...}
) -> dict:
    """Check health of all regional deployments."""
    results = {}

    async with httpx.AsyncClient(timeout=10.0) as client:
        for region, url in regions.items():
            start = time.perf_counter()
            try:
                response = await client.get(f"{url}/health")
                latency = (time.perf_counter() - start) * 1000

                results[region] = {
                    "status": "healthy" if response.status_code == 200 else "unhealthy",
                    "latency_ms": round(latency, 2),
                    "status_code": response.status_code,
                }
            except Exception as e:
                latency = (time.perf_counter() - start) * 1000
                results[region] = {
                    "status": "unreachable",
                    "latency_ms": round(latency, 2),
                    "error": str(e),
                }

    return results
```

---

## Common Pitfalls

1. **Not handling split-brain in active-active**: If network between regions fails, both regions accept writes independently. When connectivity restores, conflicting writes need resolution. Implement last-write-wins or operational transforms.

2. **Replicating embeddings without source text**: If you replicate only vectors, you cannot rebuild the index if the embedding model changes. Always replicate both source text and vectors.

3. **Ignoring replication cost**: Cross-region data transfer is expensive ($0.01-0.09/GB on AWS). For large collections, replication bandwidth costs may exceed compute costs.

4. **Not testing cross-region search quality**: Different regions may have different document counts during replication delays. Monitor search quality metrics per-region, not just globally.

5. **Hardcoding region endpoints**: Use service discovery or configuration management for region endpoints. Hardcoded URLs break when regions change.

6. **Forgetting DNS TTL**: When failing over, DNS changes take time to propagate based on TTL. Use low TTLs (60s) for failover DNS records.

---

## References

- Pinecone regions: https://docs.pinecone.io/guides/indexes/create-an-index
- Qdrant distributed: https://qdrant.tech/documentation/guides/distributed_deployment/
- MongoDB Atlas global clusters: https://www.mongodb.com/docs/atlas/global-clusters/
- AWS multi-region: https://aws.amazon.com/architecture/multi-region/
