# RAG Production Scaling -- Horizontal, Multi-Tenant, and SLA Targets

## TL;DR

Scaling a RAG system means handling more concurrent users, more documents, and more tenants without degrading latency or quality. The three dimensions are: horizontal scaling (adding retrieval replicas and distributing LLM load), multi-tenant isolation (ensuring one tenant's data and load cannot affect another), and SLA management (defining and enforcing latency, throughput, and quality targets per customer tier). This article covers replica architectures, namespace/partition/filter strategies for tenant isolation, load balancing across LLM providers, and tiered SLA configurations with automatic enforcement.

---

## Horizontal Scaling Architecture

### Scaling Each Stage Independently

```
                        Load Balancer
                             |
                    +--------+--------+
                    |        |        |
                [API-1]  [API-2]  [API-3]    <- Stateless API servers
                    |        |        |
                    +--------+--------+
                             |
              +--------------+--------------+
              |              |              |
         [Embed-1]     [Embed-2]     [Embed-3]   <- Embedding workers
              |              |              |
              +--------------+--------------+
                             |
              +--------------+--------------+
              |              |              |
         [Vector-R1]   [Vector-R2]   [Vector-R3]  <- Vector store replicas
              |              |              |
              +--------------+--------------+
                             |
              +--------------+--------------+
              |              |              |
          [LLM-Pool]    [LLM-Pool]    [LLM-Pool]  <- LLM with rate limiting
```

### Embedding Worker Pool

```python
import asyncio
import logging
from concurrent.futures import ThreadPoolExecutor

logger = logging.getLogger(__name__)


class EmbeddingWorkerPool:
    """Pool of embedding workers for parallel processing.

    Scales embedding throughput by distributing across
    multiple workers, each potentially using a different
    API key for higher rate limits.
    """

    def __init__(
        self,
        embedding_fns: list,
        max_workers: int = 10,
    ):
        self.workers = embedding_fns
        self.pool = ThreadPoolExecutor(max_workers=max_workers)
        self._worker_idx = 0

    def _get_next_worker(self):
        """Round-robin worker selection."""
        worker = self.workers[self._worker_idx % len(self.workers)]
        self._worker_idx += 1
        return worker

    def embed_batch(
        self, texts: list[str], batch_size: int = 50
    ) -> list[list[float]]:
        """Embed texts in parallel across workers."""
        results = [None] * len(texts)
        futures = []

        for i in range(0, len(texts), batch_size):
            batch = texts[i : i + batch_size]
            worker = self._get_next_worker()
            future = self.pool.submit(worker.embed_documents, batch)
            futures.append((i, future))

        for start_idx, future in futures:
            batch_results = future.result()
            for j, embedding in enumerate(batch_results):
                results[start_idx + j] = embedding

        return results

    async def embed_batch_async(
        self, texts: list[str], batch_size: int = 50
    ) -> list[list[float]]:
        """Async version for use in async pipelines."""
        loop = asyncio.get_event_loop()
        return await loop.run_in_executor(
            None, self.embed_batch, texts, batch_size
        )
```

### LLM Load Balancer

```python
import time
import random
from dataclasses import dataclass, field


@dataclass
class LLMEndpoint:
    name: str
    client: object
    model: str
    max_rpm: int  # requests per minute
    current_rpm: int = 0
    last_reset: float = field(default_factory=time.time)
    error_count: int = 0
    latency_sum: float = 0.0
    request_count: int = 0

    @property
    def avg_latency(self) -> float:
        if self.request_count == 0:
            return 0.0
        return self.latency_sum / self.request_count

    @property
    def available_capacity(self) -> float:
        """Fraction of capacity remaining (0.0 to 1.0)."""
        self._maybe_reset_window()
        return max(0, (self.max_rpm - self.current_rpm) / self.max_rpm)

    def _maybe_reset_window(self) -> None:
        if time.time() - self.last_reset > 60:
            self.current_rpm = 0
            self.last_reset = time.time()


class LLMLoadBalancer:
    """Load balance across multiple LLM endpoints.

    Supports multiple providers (Anthropic, OpenAI, etc.)
    with capacity-aware routing and automatic failover.
    """

    def __init__(self, endpoints: list[LLMEndpoint]):
        self.endpoints = endpoints

    def _select_endpoint(self) -> LLMEndpoint:
        """Select the best endpoint based on available capacity
        and recent performance."""
        available = [
            ep for ep in self.endpoints
            if ep.available_capacity > 0.1 and ep.error_count < 5
        ]

        if not available:
            # All endpoints saturated, reset error counts and try any
            for ep in self.endpoints:
                ep.error_count = 0
            available = self.endpoints

        # Weighted random selection by available capacity
        weights = [ep.available_capacity for ep in available]
        total = sum(weights)
        if total == 0:
            return random.choice(available)

        r = random.uniform(0, total)
        cumulative = 0
        for ep, w in zip(available, weights):
            cumulative += w
            if r <= cumulative:
                return ep

        return available[-1]

    def generate(self, prompt: str, **kwargs) -> dict:
        """Generate using the best available endpoint."""
        endpoint = self._select_endpoint()
        start = time.perf_counter()

        try:
            response = endpoint.client.messages.create(
                model=endpoint.model,
                messages=[{"role": "user", "content": prompt}],
                **kwargs,
            )
            latency = (time.perf_counter() - start) * 1000

            endpoint.current_rpm += 1
            endpoint.request_count += 1
            endpoint.latency_sum += latency

            return {
                "content": response.content[0].text,
                "endpoint": endpoint.name,
                "latency_ms": latency,
                "usage": {
                    "input_tokens": response.usage.input_tokens,
                    "output_tokens": response.usage.output_tokens,
                },
            }

        except Exception as e:
            endpoint.error_count += 1
            logger.warning(
                "Endpoint %s failed: %s, trying another",
                endpoint.name,
                str(e),
            )
            # Retry with a different endpoint
            return self.generate(prompt, **kwargs)
```

---

## Multi-Tenant Architecture

### Isolation Strategies

| Strategy | Isolation Level | Cost | Complexity | Use Case |
|----------|----------------|------|------------|----------|
| Namespace | Logical | Low | Low | SaaS with <100 tenants |
| Partition | Physical + Logical | Medium | Medium | SaaS with 100-1000 tenants |
| Metadata filter | Logical (shared index) | Lowest | Low | Simple multi-tenancy |
| Separate index | Full physical | High | High | Regulated industries |

### Namespace-Based Isolation (Pinecone)

```python
from pinecone import Pinecone


class NamespaceTenantManager:
    """Multi-tenant RAG using Pinecone namespaces.

    Each tenant gets a separate namespace within the same index.
    Data is isolated -- queries in one namespace never see data
    from another namespace.
    """

    def __init__(self, api_key: str, index_name: str):
        self.pc = Pinecone(api_key=api_key)
        self.index = self.pc.Index(index_name)

    def _namespace(self, tenant_id: str) -> str:
        return f"tenant_{tenant_id}"

    def upsert_documents(
        self,
        tenant_id: str,
        documents: list[dict],
        embedding_fn,
    ) -> int:
        """Index documents into a tenant's namespace."""
        namespace = self._namespace(tenant_id)
        vectors = []
        for doc in documents:
            embedding = embedding_fn(doc["content"])
            vectors.append({
                "id": doc["doc_id"],
                "values": embedding,
                "metadata": {
                    "content": doc["content"][:1000],
                    "source": doc.get("source", ""),
                    "tenant_id": tenant_id,
                },
            })

        # Upsert in batches
        batch_size = 100
        for i in range(0, len(vectors), batch_size):
            batch = vectors[i : i + batch_size]
            self.index.upsert(
                vectors=batch, namespace=namespace
            )

        return len(vectors)

    def query(
        self,
        tenant_id: str,
        query_vector: list[float],
        top_k: int = 10,
    ) -> list[dict]:
        """Query within a tenant's namespace only."""
        namespace = self._namespace(tenant_id)
        results = self.index.query(
            vector=query_vector,
            top_k=top_k,
            namespace=namespace,
            include_metadata=True,
        )
        return [
            {
                "id": match.id,
                "score": match.score,
                "metadata": match.metadata,
            }
            for match in results.matches
        ]

    def delete_tenant(self, tenant_id: str) -> dict:
        """Delete all data for a tenant."""
        namespace = self._namespace(tenant_id)
        self.index.delete(delete_all=True, namespace=namespace)
        return {"tenant_id": tenant_id, "deleted": True}

    def get_tenant_stats(self, tenant_id: str) -> dict:
        """Get vector count for a tenant."""
        namespace = self._namespace(tenant_id)
        stats = self.index.describe_index_stats(
            filter={"tenant_id": {"$eq": tenant_id}}
        )
        ns_stats = stats.namespaces.get(namespace, {})
        return {
            "tenant_id": tenant_id,
            "vector_count": getattr(ns_stats, "vector_count", 0),
        }
```

### Metadata Filter Isolation

```python
class MetadataFilterTenantManager:
    """Multi-tenant RAG using metadata filters on a shared index.

    All tenants share the same index and namespace.
    Isolation is enforced by adding a tenant_id metadata field
    to every vector and filtering on it during queries.

    Simpler than namespaces but relies on the application
    always applying the filter correctly.
    """

    def __init__(self, vector_store):
        self.store = vector_store

    def add_documents(
        self,
        tenant_id: str,
        texts: list[str],
        metadatas: list[dict],
    ) -> None:
        """Add documents with tenant_id in metadata."""
        for m in metadatas:
            m["tenant_id"] = tenant_id
        self.store.add_texts(texts, metadatas=metadatas)

    def query(
        self,
        tenant_id: str,
        query: str,
        top_k: int = 10,
    ) -> list:
        """Query with mandatory tenant filter."""
        return self.store.similarity_search(
            query,
            k=top_k,
            filter={"tenant_id": {"$eq": tenant_id}},
        )


class TenantFilterMiddleware:
    """Middleware that enforces tenant filtering on every query.

    Prevents accidental cross-tenant data access by injecting
    the tenant filter before the query reaches the vector store.
    """

    def __init__(self, retriever, tenant_extractor):
        self.retriever = retriever
        self.extract_tenant = tenant_extractor

    def invoke(self, query: str, **kwargs) -> list:
        tenant_id = self.extract_tenant(kwargs)
        if not tenant_id:
            raise ValueError(
                "tenant_id is required for all queries"
            )

        # Inject tenant filter
        existing_filter = kwargs.get("filter", {})
        existing_filter["tenant_id"] = {"$eq": tenant_id}

        return self.retriever.invoke(
            query, filter=existing_filter, **kwargs
        )
```

---

## SLA Management

### Tiered Configuration

```python
from pydantic import BaseModel, Field


class TierConfig(BaseModel):
    """Configuration for a customer tier."""

    name: str
    max_latency_ms: int
    max_concurrent_requests: int
    retrieval_top_k: int
    rerank_enabled: bool
    model: str
    max_tokens: int
    cache_enabled: bool
    cache_ttl: int
    rate_limit_rpm: int  # requests per minute
    priority: int  # Higher = processed first


TIER_CONFIGS = {
    "free": TierConfig(
        name="free",
        max_latency_ms=10000,
        max_concurrent_requests=5,
        retrieval_top_k=5,
        rerank_enabled=False,
        model="claude-haiku-4-20250514",
        max_tokens=1024,
        cache_enabled=True,
        cache_ttl=3600,
        rate_limit_rpm=20,
        priority=1,
    ),
    "pro": TierConfig(
        name="pro",
        max_latency_ms=5000,
        max_concurrent_requests=20,
        retrieval_top_k=10,
        rerank_enabled=True,
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        cache_enabled=True,
        cache_ttl=1800,
        rate_limit_rpm=100,
        priority=5,
    ),
    "enterprise": TierConfig(
        name="enterprise",
        max_latency_ms=3000,
        max_concurrent_requests=100,
        retrieval_top_k=20,
        rerank_enabled=True,
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        cache_enabled=True,
        cache_ttl=900,
        rate_limit_rpm=500,
        priority=10,
    ),
}


class TieredRAGPipeline:
    """RAG pipeline that adapts behavior based on customer tier."""

    def __init__(self, retriever, reranker, llm_pool):
        self.retriever = retriever
        self.reranker = reranker
        self.llm_pool = llm_pool

    def query(
        self, user_query: str, tenant_id: str, tier: str
    ) -> dict:
        config = TIER_CONFIGS.get(tier, TIER_CONFIGS["free"])

        # Retrieval with tier-appropriate top_k
        docs = self.retriever.invoke(
            user_query,
            filter={"tenant_id": {"$eq": tenant_id}},
            k=config.retrieval_top_k,
        )

        # Conditional reranking
        if config.rerank_enabled and len(docs) > 5:
            docs = self.reranker.rerank(user_query, docs)

        # Generation with tier-appropriate model
        context = "\n\n".join(d.page_content for d in docs[:5])
        response = self.llm_pool.generate(
            f"Context:\n{context}\n\nQuestion: {user_query}",
            model=config.model,
            max_tokens=config.max_tokens,
        )

        return {
            "answer": response["content"],
            "tier": config.name,
            "model_used": config.model,
            "docs_retrieved": len(docs),
        }
```

### Rate Limiting

```python
import time


class TokenBucketRateLimiter:
    """Per-tenant rate limiter using the token bucket algorithm.

    Each tenant has a bucket that fills at a fixed rate.
    Requests consume tokens. When the bucket is empty,
    requests are rejected until tokens refill.
    """

    def __init__(self, redis_client, default_rpm: int = 60):
        self.redis = redis_client
        self.default_rpm = default_rpm

    def check_and_consume(
        self, tenant_id: str, tier_rpm: int | None = None
    ) -> dict:
        """Check rate limit and consume a token if allowed."""
        rpm = tier_rpm or self.default_rpm
        key = f"ratelimit:{tenant_id}"

        pipe = self.redis.pipeline()
        now = time.time()
        window_start = now - 60  # 1-minute window

        # Remove old entries
        pipe.zremrangebyscore(key, 0, window_start)
        # Count current entries
        pipe.zcard(key)
        # Add current request
        pipe.zadd(key, {str(now): now})
        # Set expiry
        pipe.expire(key, 120)

        results = pipe.execute()
        current_count = results[1]

        if current_count >= rpm:
            return {
                "allowed": False,
                "current_rpm": current_count,
                "limit_rpm": rpm,
                "retry_after_seconds": 60 - (now - window_start),
            }

        return {
            "allowed": True,
            "current_rpm": current_count + 1,
            "limit_rpm": rpm,
            "remaining": rpm - current_count - 1,
        }
```

---

## Capacity Planning

### Estimating Resource Requirements

```python
def estimate_resources(
    daily_queries: int,
    avg_docs_per_query: int = 10,
    avg_doc_size_tokens: int = 500,
    total_documents: int = 100000,
    avg_chunks_per_doc: int = 5,
    embedding_dimension: int = 1536,
) -> dict:
    """Estimate infrastructure requirements for a RAG deployment."""
    total_chunks = total_documents * avg_chunks_per_doc

    # Vector storage (4 bytes per float32 dimension)
    vector_storage_gb = (
        total_chunks * embedding_dimension * 4 / (1024 ** 3)
    )

    # With HNSW index overhead (~1.5x)
    total_storage_gb = vector_storage_gb * 1.5

    # RAM requirement (vectors must fit in memory for HNSW)
    ram_gb = total_storage_gb * 1.2  # 20% overhead

    # Queries per second
    qps = daily_queries / 86400

    # Embedding API calls per day
    embedding_calls = daily_queries  # 1 per query

    # LLM tokens per day
    input_tokens_per_day = daily_queries * (
        avg_docs_per_query * avg_doc_size_tokens + 200  # system prompt
    )
    output_tokens_per_day = daily_queries * 500  # avg response

    # Cost estimates (approximate)
    embedding_cost_per_day = embedding_calls * 0.0001
    llm_input_cost = input_tokens_per_day * 3.0 / 1_000_000
    llm_output_cost = output_tokens_per_day * 15.0 / 1_000_000

    return {
        "total_chunks": total_chunks,
        "vector_storage_gb": round(vector_storage_gb, 2),
        "total_storage_gb": round(total_storage_gb, 2),
        "recommended_ram_gb": round(ram_gb, 2),
        "queries_per_second": round(qps, 2),
        "embedding_calls_per_day": embedding_calls,
        "input_tokens_per_day": input_tokens_per_day,
        "output_tokens_per_day": output_tokens_per_day,
        "estimated_daily_cost_usd": round(
            embedding_cost_per_day + llm_input_cost + llm_output_cost,
            2,
        ),
    }
```

---

## Common Pitfalls

1. **Scaling the API servers but not the vector store.** If your API servers can handle 1000 QPS but the vector store caps at 200 QPS, the bottleneck just moved. Scale all stages proportionally.
2. **Using metadata filters for tenant isolation without testing edge cases.** If a code path ever forgets to apply the tenant filter, one tenant can see another's data. Use middleware that enforces the filter at the infrastructure level.
3. **Same SLA for all tenants.** Free-tier users consuming the same model and resources as enterprise customers wastes money and degrades enterprise experience. Implement tiered configurations.
4. **Not rate limiting per tenant.** One tenant sending 1000 requests per minute consumes all LLM capacity. Per-tenant rate limits prevent noisy neighbor problems.
5. **Ignoring embedding throughput limits.** Embedding APIs have rate limits (e.g., OpenAI: 3000 RPM). During bulk ingestion, you can exhaust the limit and block real-time queries. Use separate API keys for ingestion and serving.
6. **Not planning for 2x storage during reindexing.** Blue-green reindexing requires two full indexes simultaneously. If your infrastructure is provisioned for exactly 1x, the reindex will fail.

---

## References

- Pinecone scaling guide: https://docs.pinecone.io/guides/operations/scaling
- Weaviate multi-tenancy: https://weaviate.io/developers/weaviate/manage-data/multi-tenancy
- Qdrant distributed deployment: https://qdrant.tech/documentation/guides/distributed_deployment/
