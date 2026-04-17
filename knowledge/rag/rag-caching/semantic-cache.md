# Semantic Cache -- GPTCache, Redis Embedding Similarity, and TTL Management

## TL;DR

Semantic caching intercepts RAG queries before they hit the retriever and LLM by comparing the new query's embedding to previously answered queries. If a cached query is similar enough (cosine similarity above threshold), the stored answer is returned instantly. GPTCache is the leading open-source framework for this, supporting pluggable embedding backends, similarity evaluators, and storage adapters. Redis with its vector search module (RediSearch) provides a production-grade backend for storing and searching cached embeddings at scale. This article covers GPTCache setup and configuration, Redis-backed semantic caching with vector search, TTL management strategies, and production tuning for hit rate and correctness.

---

## GPTCache Architecture

### Core Components

GPTCache decomposes caching into four pluggable stages:

```
Query -> [Pre-processor] -> [Embedding] -> [Similarity Evaluator] -> [Cache Storage]
                                                                          |
                                                                     hit / miss
                                                                          |
                                                          Return cached / call LLM
```

| Stage | Responsibility | Options |
|-------|---------------|---------|
| Pre-processor | Normalize query (lowercase, strip whitespace, remove stopwords) | Default, custom function |
| Embedding | Convert query to vector | OpenAI, ONNX, Hugging Face, Cohere |
| Similarity Evaluator | Compare query embedding to cached embeddings | Cosine distance, ONNX cross-encoder, custom |
| Cache Storage | Store/retrieve embeddings and answers | SQLite + FAISS (default), Redis, PostgreSQL |

### Basic GPTCache Setup

```python
from gptcache import Cache
from gptcache.adapter import openai as gptcache_openai
from gptcache.embedding import OpenAI as OpenAIEmbedding
from gptcache.manager import CacheBase, VectorBase, get_data_manager
from gptcache.similarity_evaluation import SearchDistanceEvaluation


def initialize_gptcache() -> Cache:
    """Set up GPTCache with OpenAI embeddings and FAISS storage."""
    cache = Cache()

    # Embedding function -- uses OpenAI text-embedding-3-small
    embedding = OpenAIEmbedding(model="text-embedding-3-small")

    # Storage: SQLite for metadata, FAISS for vectors
    cache_base = CacheBase("sqlite")
    vector_base = VectorBase(
        "faiss",
        dimension=embedding.dimension,
    )
    data_manager = get_data_manager(cache_base, vector_base)

    # Similarity: cosine distance with threshold
    evaluation = SearchDistanceEvaluation()

    cache.init(
        embedding_func=embedding.to_embeddings,
        data_manager=data_manager,
        similarity_evaluation=evaluation,
    )

    return cache


# Initialize
cache = initialize_gptcache()


# Use with OpenAI (GPTCache wraps the client)
def cached_completion(query: str) -> str:
    """Query LLM with semantic caching."""
    response = gptcache_openai.ChatCompletion.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": query}],
        cache_obj=cache,
    )
    return response["choices"][0]["message"]["content"]


# First call: cache miss -> calls OpenAI
answer1 = cached_completion("What is retrieval augmented generation?")

# Second call: cache hit -> returns cached answer (no API call)
answer2 = cached_completion("Explain RAG to me")  # Semantically similar
```

### GPTCache with ONNX Embedding (No API Cost)

```python
from gptcache import Cache
from gptcache.embedding import Onnx as OnnxEmbedding
from gptcache.manager import CacheBase, VectorBase, get_data_manager
from gptcache.similarity_evaluation import SearchDistanceEvaluation


def initialize_local_cache() -> Cache:
    """GPTCache with local ONNX embeddings -- no API calls for caching."""
    cache = Cache()

    # Local ONNX model -- runs on CPU, no API cost
    embedding = OnnxEmbedding()

    cache_base = CacheBase("sqlite")
    vector_base = VectorBase(
        "faiss", dimension=embedding.dimension
    )
    data_manager = get_data_manager(cache_base, vector_base)

    cache.init(
        embedding_func=embedding.to_embeddings,
        data_manager=data_manager,
        similarity_evaluation=SearchDistanceEvaluation(),
    )
    return cache
```

### Custom Pre-Processor for RAG Queries

```python
import re


def rag_query_preprocessor(query: str) -> str:
    """Normalize RAG queries before embedding for cache lookup.

    Strips formatting artifacts, normalizes whitespace, and lowercases
    to increase cache hit rates without affecting semantic similarity.
    """
    # Remove common prefixes injected by chat UIs
    prefixes_to_strip = [
        "can you tell me",
        "please explain",
        "i want to know",
        "could you help me with",
        "what is",
        "what are",
    ]
    query_lower = query.lower().strip()
    for prefix in prefixes_to_strip:
        if query_lower.startswith(prefix):
            query_lower = query_lower[len(prefix):].strip()
            break

    # Normalize whitespace
    query_lower = re.sub(r"\s+", " ", query_lower)

    # Remove trailing punctuation that does not change meaning
    query_lower = query_lower.rstrip("?.!")

    return query_lower
```

---

## Redis-Backed Semantic Cache

### Why Redis for Production

SQLite + FAISS (GPTCache default) works for single-process development. In production with multiple API server replicas, you need a shared cache backend. Redis with the RediSearch module provides:

- Shared state across replicas
- Built-in TTL per key
- Vector similarity search (HNSW or FLAT index)
- Atomic operations for concurrent access
- Persistence and replication

### Redis Vector Search Setup

```python
import json
import time
import hashlib
from typing import Any

import numpy as np
import redis
from redis.commands.search.field import (
    TagField,
    TextField,
    NumericField,
    VectorField,
)
from redis.commands.search.indexDefinition import IndexDefinition, IndexType
from redis.commands.search.query import Query


class RedisSemanticCache:
    """Production-grade semantic cache using Redis vector search.

    Stores query embeddings in a Redis HNSW index and performs
    approximate nearest neighbor search to find cache hits.
    """

    INDEX_NAME = "idx:semantic_cache"
    PREFIX = "cache:"

    def __init__(
        self,
        redis_url: str = "redis://localhost:6379",
        embedding_dim: int = 1536,
        similarity_threshold: float = 0.92,
        default_ttl: int = 3600,
    ):
        self.redis = redis.from_url(redis_url)
        self.dim = embedding_dim
        self.threshold = similarity_threshold
        self.default_ttl = default_ttl
        self._ensure_index()

    def _ensure_index(self) -> None:
        """Create Redis search index if it does not exist."""
        try:
            self.redis.ft(self.INDEX_NAME).info()
        except redis.ResponseError:
            schema = (
                TextField("query"),
                TextField("answer"),
                TextField("sources"),
                NumericField("created_at"),
                NumericField("ttl"),
                TagField("namespace"),
                VectorField(
                    "embedding",
                    "HNSW",
                    {
                        "TYPE": "FLOAT32",
                        "DIM": self.dim,
                        "DISTANCE_METRIC": "COSINE",
                        "M": 16,
                        "EF_CONSTRUCTION": 200,
                    },
                ),
            )
            definition = IndexDefinition(
                prefix=[self.PREFIX], index_type=IndexType.HASH
            )
            self.redis.ft(self.INDEX_NAME).create_index(
                schema, definition=definition
            )

    def get(
        self,
        query_embedding: list[float],
        namespace: str = "default",
    ) -> dict | None:
        """Search for a semantically similar cached response."""
        query_vector = np.array(
            query_embedding, dtype=np.float32
        ).tobytes()

        q = (
            Query(
                f"(@namespace:{{{namespace}}})=>"
                f"[KNN 1 @embedding $vec AS score]"
            )
            .sort_by("score")
            .return_fields("query", "answer", "sources", "score", "created_at", "ttl")
            .dialect(2)
        )

        results = self.redis.ft(self.INDEX_NAME).search(
            q, query_params={"vec": query_vector}
        )

        if not results.docs:
            return None

        doc = results.docs[0]
        # Redis cosine distance: 0 = identical, 2 = opposite
        # Convert to similarity: 1 - distance
        similarity = 1.0 - float(doc.score)

        if similarity < self.threshold:
            return None

        # Check TTL expiration
        created_at = float(doc.created_at)
        ttl = int(doc.ttl)
        if time.time() - created_at > ttl:
            # Expired -- delete and return miss
            self.redis.delete(doc.id)
            return None

        return {
            "answer": doc.answer,
            "sources": json.loads(doc.sources),
            "similarity": similarity,
            "original_query": doc.query,
            "cache_hit": True,
        }

    def put(
        self,
        query: str,
        query_embedding: list[float],
        answer: str,
        sources: list[dict],
        namespace: str = "default",
        ttl: int | None = None,
    ) -> str:
        """Store a response in the cache."""
        entry_id = hashlib.sha256(
            f"{query}:{time.time()}".encode()
        ).hexdigest()[:16]
        key = f"{self.PREFIX}{entry_id}"

        embedding_bytes = np.array(
            query_embedding, dtype=np.float32
        ).tobytes()

        self.redis.hset(
            key,
            mapping={
                "query": query,
                "answer": answer,
                "sources": json.dumps(sources),
                "embedding": embedding_bytes,
                "created_at": time.time(),
                "ttl": ttl or self.default_ttl,
                "namespace": namespace,
            },
        )

        # Set Redis TTL as a safety net
        self.redis.expire(key, (ttl or self.default_ttl) + 60)

        return key

    def invalidate_by_source(
        self, doc_id: str, namespace: str = "default"
    ) -> int:
        """Remove cache entries that reference a specific source document."""
        # Search for entries containing the doc_id in sources
        q = (
            Query(f"@namespace:{{{namespace}}} @sources:{doc_id}")
            .return_fields("query")
            .dialect(2)
        )
        results = self.redis.ft(self.INDEX_NAME).search(q)

        count = 0
        for doc in results.docs:
            self.redis.delete(doc.id)
            count += 1
        return count

    def flush_namespace(self, namespace: str) -> int:
        """Delete all cache entries in a namespace."""
        q = (
            Query(f"@namespace:{{{namespace}}}")
            .return_fields("query")
            .no_content()
            .dialect(2)
        )
        results = self.redis.ft(self.INDEX_NAME).search(q)
        count = 0
        for doc in results.docs:
            self.redis.delete(doc.id)
            count += 1
        return count

    def stats(self, namespace: str = "default") -> dict:
        """Get cache statistics for a namespace."""
        info = self.redis.ft(self.INDEX_NAME).info()
        return {
            "total_docs": int(info["num_docs"]),
            "index_size_mb": int(info.get("inverted_sz_mb", 0)),
            "namespace": namespace,
        }
```

### Integration with RAG Pipeline

```python
from langchain_anthropic import ChatAnthropic
from langchain_openai import OpenAIEmbeddings


class RedisCachedRAG:
    """RAG pipeline with Redis semantic cache."""

    def __init__(
        self,
        retriever,
        redis_url: str = "redis://localhost:6379",
        namespace: str = "default",
    ):
        self.retriever = retriever
        self.llm = ChatAnthropic(model="claude-sonnet-4-20250514")
        self.embeddings = OpenAIEmbeddings(
            model="text-embedding-3-small"
        )
        self.cache = RedisSemanticCache(
            redis_url=redis_url, embedding_dim=1536
        )
        self.namespace = namespace

    def query(self, user_query: str) -> dict:
        # Step 1: Embed the query
        query_embedding = self.embeddings.embed_query(user_query)

        # Step 2: Check cache
        cached = self.cache.get(query_embedding, self.namespace)
        if cached:
            return cached

        # Step 3: Full retrieval
        docs = self.retriever.invoke(user_query)
        context = "\n\n".join(d.page_content for d in docs)

        # Step 4: Generate
        answer = self.llm.invoke(
            f"Context:\n{context}\n\nQuestion: {user_query}"
        ).content

        sources = [d.metadata for d in docs]

        # Step 5: Store in cache
        self.cache.put(
            query=user_query,
            query_embedding=query_embedding,
            answer=answer,
            sources=sources,
            namespace=self.namespace,
        )

        return {
            "answer": answer,
            "sources": sources,
            "cache_hit": False,
        }
```

---

## TTL Management Strategies

### Static TTL

Simplest approach: every cache entry expires after a fixed duration.

```python
# Configure in RedisSemanticCache
cache = RedisSemanticCache(
    default_ttl=1800,  # 30 minutes
)
```

**When to use**: Uniform content freshness requirements, simple deployments.

### Per-Source TTL

Different document types have different update frequencies:

```python
# TTL based on source type
SOURCE_TTL_MAP = {
    "api_docs": 7200,       # 2 hours (API docs change infrequently)
    "changelog": 300,        # 5 minutes (changes often)
    "pricing": 600,          # 10 minutes (critical to be current)
    "tutorial": 14400,       # 4 hours (rarely changes)
    "faq": 3600,             # 1 hour
}


def get_ttl_for_sources(sources: list[dict]) -> int:
    """Determine TTL based on the most volatile source used."""
    ttls = []
    for source in sources:
        source_type = source.get("type", "default")
        ttls.append(SOURCE_TTL_MAP.get(source_type, 3600))
    # Use the shortest TTL (most volatile source dictates freshness)
    return min(ttls) if ttls else 3600
```

### Adaptive TTL Based on Query Frequency

Frequently asked questions should be cached longer because (a) the benefit is higher and (b) they are more likely to be re-asked:

```python
import math


class AdaptiveTTLManager:
    """Adjust TTL based on query frequency and cache hit patterns.

    Frequently hit entries get longer TTLs (more value from caching).
    Rarely hit entries get shorter TTLs (free up cache space sooner).
    """

    def __init__(
        self,
        redis_client: redis.Redis,
        base_ttl: int = 1800,
        min_ttl: int = 300,
        max_ttl: int = 14400,
    ):
        self.redis = redis_client
        self.base_ttl = base_ttl
        self.min_ttl = min_ttl
        self.max_ttl = max_ttl

    def record_hit(self, cache_key: str) -> None:
        """Record a cache hit and extend TTL if warranted."""
        hit_count_key = f"hits:{cache_key}"
        hits = self.redis.incr(hit_count_key)
        self.redis.expire(hit_count_key, self.max_ttl)

        # Logarithmic TTL growth: more hits -> longer TTL
        new_ttl = int(
            self.base_ttl * (1 + math.log2(max(hits, 1)))
        )
        new_ttl = max(self.min_ttl, min(new_ttl, self.max_ttl))

        # Extend the cache entry's TTL
        remaining = self.redis.ttl(cache_key)
        if remaining < new_ttl:
            self.redis.expire(cache_key, new_ttl)

    def get_recommended_ttl(self, query: str) -> int:
        """Get recommended TTL for a new cache entry based on
        historical query frequency."""
        # Hash the query to look up prior frequency
        query_hash = hashlib.sha256(query.encode()).hexdigest()[:16]
        freq_key = f"freq:{query_hash}"

        freq = self.redis.get(freq_key)
        if freq is None:
            self.redis.setex(freq_key, self.max_ttl, 1)
            return self.base_ttl

        freq = int(freq)
        self.redis.incr(freq_key)

        # Higher frequency -> longer TTL
        ttl = int(self.base_ttl * (1 + math.log2(max(freq, 1))))
        return max(self.min_ttl, min(ttl, self.max_ttl))
```

---

## Multi-Tenant Caching

### Namespace Isolation

When serving multiple customers from the same cache infrastructure, use namespace isolation to prevent data leakage:

```python
class MultiTenantCache:
    """Semantic cache with tenant isolation via namespaces.

    Each tenant's cache entries are isolated -- a query from
    tenant A will never return a cached answer from tenant B.
    """

    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.cache = RedisSemanticCache(redis_url=redis_url)

    def query(
        self,
        tenant_id: str,
        query_embedding: list[float],
    ) -> dict | None:
        """Search cache scoped to a specific tenant."""
        return self.cache.get(
            query_embedding, namespace=f"tenant:{tenant_id}"
        )

    def store(
        self,
        tenant_id: str,
        query: str,
        query_embedding: list[float],
        answer: str,
        sources: list[dict],
        ttl: int = 3600,
    ) -> str:
        return self.cache.put(
            query=query,
            query_embedding=query_embedding,
            answer=answer,
            sources=sources,
            namespace=f"tenant:{tenant_id}",
            ttl=ttl,
        )

    def flush_tenant(self, tenant_id: str) -> int:
        """Clear all cached entries for a tenant."""
        return self.cache.flush_namespace(f"tenant:{tenant_id}")
```

---

## Testing Semantic Cache Correctness

### Measuring Cache Quality

A semantic cache can return wrong answers if the threshold is too low. Test systematically:

```python
import pytest


class TestSemanticCacheCorrectness:
    """Verify that cache hits return appropriate answers.

    The test pairs consist of:
    - Equivalent queries (should hit)
    - Related but different queries (should miss)
    """

    EQUIVALENT_PAIRS = [
        (
            "How do I reset my password?",
            "What is the password reset process?",
        ),
        (
            "What are the pricing tiers?",
            "Tell me about your pricing plans",
        ),
        (
            "How to configure SSL certificates?",
            "Setting up SSL/TLS certificates",
        ),
    ]

    DIFFERENT_PAIRS = [
        (
            "How do I reset my password?",
            "How do I change my email address?",
        ),
        (
            "What are the pricing tiers?",
            "What payment methods do you accept?",
        ),
        (
            "How to configure SSL certificates?",
            "How to configure load balancers?",
        ),
    ]

    def test_equivalent_queries_hit_cache(
        self, cache: RedisSemanticCache, embed_fn
    ):
        """Semantically equivalent queries should produce cache hits."""
        for q1, q2 in self.EQUIVALENT_PAIRS:
            emb1 = embed_fn(q1)
            cache.put(q1, emb1, "Answer for: " + q1, [], ttl=600)

            emb2 = embed_fn(q2)
            result = cache.get(emb2)

            assert result is not None, (
                f"Expected cache hit for equivalent queries:\n"
                f"  Original: {q1}\n"
                f"  Similar:  {q2}"
            )

    def test_different_queries_miss_cache(
        self, cache: RedisSemanticCache, embed_fn
    ):
        """Semantically different queries should NOT produce cache hits."""
        for q1, q2 in self.DIFFERENT_PAIRS:
            emb1 = embed_fn(q1)
            cache.put(q1, emb1, "Answer for: " + q1, [], ttl=600)

            emb2 = embed_fn(q2)
            result = cache.get(emb2)

            assert result is None, (
                f"Expected cache miss for different queries:\n"
                f"  Original: {q1}\n"
                f"  Different: {q2}\n"
                f"  Similarity: {result.get('similarity', 'N/A')}"
            )
```

---

## Performance Benchmarks

### Expected Latency by Backend

| Backend | Cache Lookup Latency | Throughput (queries/sec) |
|---------|---------------------|------------------------|
| In-memory (FAISS) | 1-5 ms | 10,000+ |
| Redis (local) | 2-10 ms | 5,000+ |
| Redis (network) | 10-30 ms | 1,000+ |
| PostgreSQL pgvector | 20-50 ms | 500+ |

### Break-Even Analysis

```
Cache hit saves: ~2 seconds (LLM call) + ~$0.02 (API cost)
Cache lookup costs: ~10 ms + ~$0.0001 (embedding for query)

Break-even hit rate: 0.5%
Typical hit rate: 30-50% for support bots
ROI at 30% hit rate on 10K daily queries: ~$60/day saved, 1000+ seconds of latency eliminated
```

---

## Common Pitfalls

1. **Using exact string matching instead of semantic similarity.** Exact match catches only identical queries. Semantic caching catches paraphrases, which represent the majority of repeated queries.
2. **Sharing cache across tenants without namespace isolation.** Tenant A sees tenant B's cached answers, which may contain confidential information. Always scope caches by tenant.
3. **Not testing cache correctness.** A semantic cache with a bad threshold silently returns wrong answers. Write tests with known equivalent and non-equivalent query pairs.
4. **Storing embeddings in SQLite in production.** SQLite locks on writes, making it unsuitable for multi-process deployments. Use Redis or PostgreSQL for shared caching.
5. **Forgetting to cache the embedding computation itself.** If you embed the query to check the semantic cache and then embed it again for retrieval on a miss, you have wasted an API call. Reuse the same embedding.
6. **Not setting a Redis key TTL as a safety net.** Even with application-level TTL logic, set a Redis-level EXPIRE to prevent leaked entries from accumulating indefinitely.

---

## References

- GPTCache GitHub: https://github.com/zilliztech/GPTCache
- GPTCache documentation: https://gptcache.readthedocs.io/
- Redis vector search: https://redis.io/docs/latest/develop/interact/search-and-query/query/vector-search/
- Redis HNSW configuration: https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/
