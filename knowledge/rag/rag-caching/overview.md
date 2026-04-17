# RAG Caching -- Semantic and Prompt Caching Layers

## TL;DR

RAG pipelines are expensive: every query triggers an embedding call, a vector search, and at least one LLM generation. Caching at multiple layers -- semantic query caching (returning stored answers for similar queries), embedding caching (avoiding recomputation of identical text), and prompt caching (reusing prefilled LLM context across requests) -- can cut latency by 60-80% and costs by 40-70% for production workloads with any degree of query repetition. This overview explains the three caching layers, when each applies, how they compose, and the invalidation strategies that keep cached answers fresh. Subsequent articles cover GPTCache + Redis semantic caching and provider-level prompt caching (Anthropic, OpenAI) in depth.

---

## Why Caching Matters for RAG

### Cost Breakdown of a Single RAG Query

```
Step                     Latency       Cost (approximate)
----------------------------------------------------------
Embedding (query)        50-100 ms     $0.0001 per query
Vector search            20-50 ms      Infrastructure cost
Reranking (optional)     100-300 ms    $0.001 per query
LLM generation           1-5 s         $0.01-0.05 per query
----------------------------------------------------------
Total                    ~1.5-5.5 s    ~$0.01-0.05
```

At 10,000 queries per day, that is $100-500/day just for LLM generation. In support bots, internal knowledge bases, and documentation assistants, 30-60% of queries are semantically identical or near-identical. Caching those eliminates both latency and cost.

### The Three Caching Layers

| Layer | What It Caches | Cache Key | Hit Rate | Savings |
|-------|---------------|-----------|----------|---------|
| Semantic query cache | Full answer + sources | Embedding similarity of query | 20-50% | Eliminates LLM call entirely |
| Embedding cache | Vector representation | Hash of input text | 60-90% | Eliminates embedding API call |
| Prompt cache | Prefilled LLM context | Token prefix match | 70-95% | Reduces LLM input cost 50-90% |

### How They Compose

```
User Query
    |
    v
[Semantic Query Cache] --hit--> Return cached answer (fastest, cheapest)
    |
    miss
    v
[Embedding Cache] --hit--> Skip embedding API call, use cached vector
    |
    miss
    v
[Compute embedding] --> [Vector search] --> [Retrieve documents]
    |
    v
[Prompt Cache (LLM)] --partial hit--> Reduced cost for prefilled context
    |
    v
[Generate answer] --> [Store in semantic cache] --> Return answer
```

---

## Layer 1: Semantic Query Cache

### Concept

Instead of exact string matching, semantic caching uses embedding similarity to detect when a new query is "close enough" to a previously answered query. "How do I reset my password?" and "What is the password reset process?" have different tokens but identical intent.

### Architecture

```python
import hashlib
import json
import time
from dataclasses import dataclass, field

import numpy as np
import redis
from openai import OpenAI


@dataclass
class CacheEntry:
    query: str
    query_embedding: list[float]
    answer: str
    sources: list[dict]
    created_at: float
    ttl_seconds: int = 3600
    hit_count: int = 0

    def is_expired(self) -> bool:
        return time.time() - self.created_at > self.ttl_seconds

    def to_dict(self) -> dict:
        return {
            "query": self.query,
            "query_embedding": self.query_embedding,
            "answer": self.answer,
            "sources": self.sources,
            "created_at": self.created_at,
            "ttl_seconds": self.ttl_seconds,
            "hit_count": self.hit_count,
        }


class SemanticCache:
    """Semantic similarity-based cache for RAG responses.

    Uses cosine similarity between query embeddings to detect
    cache hits, even when queries use different wording.
    """

    def __init__(
        self,
        embedding_client: OpenAI,
        embedding_model: str = "text-embedding-3-small",
        similarity_threshold: float = 0.92,
        max_entries: int = 10000,
        default_ttl: int = 3600,
    ):
        self.client = embedding_client
        self.model = embedding_model
        self.threshold = similarity_threshold
        self.max_entries = max_entries
        self.default_ttl = default_ttl
        self.entries: list[CacheEntry] = []

    def _embed(self, text: str) -> list[float]:
        response = self.client.embeddings.create(
            model=self.model, input=text
        )
        return response.data[0].embedding

    def _cosine_similarity(
        self, a: list[float], b: list[float]
    ) -> float:
        a_arr = np.array(a)
        b_arr = np.array(b)
        return float(
            np.dot(a_arr, b_arr)
            / (np.linalg.norm(a_arr) * np.linalg.norm(b_arr))
        )

    def get(self, query: str) -> dict | None:
        """Look up a semantically similar cached response."""
        query_embedding = self._embed(query)

        best_match = None
        best_score = 0.0

        for entry in self.entries:
            if entry.is_expired():
                continue
            score = self._cosine_similarity(
                query_embedding, entry.query_embedding
            )
            if score > best_score:
                best_score = score
                best_match = entry

        if best_match and best_score >= self.threshold:
            best_match.hit_count += 1
            return {
                "answer": best_match.answer,
                "sources": best_match.sources,
                "cache_hit": True,
                "similarity_score": best_score,
                "original_query": best_match.query,
            }
        return None

    def put(
        self,
        query: str,
        answer: str,
        sources: list[dict],
        ttl: int | None = None,
    ) -> None:
        """Store a response in the cache."""
        query_embedding = self._embed(query)
        entry = CacheEntry(
            query=query,
            query_embedding=query_embedding,
            answer=answer,
            sources=sources,
            created_at=time.time(),
            ttl_seconds=ttl or self.default_ttl,
        )
        self.entries.append(entry)
        self._evict_if_needed()

    def _evict_if_needed(self) -> None:
        """Remove expired entries and enforce max size."""
        self.entries = [
            e for e in self.entries if not e.is_expired()
        ]
        if len(self.entries) > self.max_entries:
            # Evict least recently used (lowest hit count)
            self.entries.sort(key=lambda e: e.hit_count)
            self.entries = self.entries[-self.max_entries :]

    def invalidate(self, query: str, threshold: float = 0.95) -> int:
        """Invalidate entries similar to the given query."""
        query_embedding = self._embed(query)
        before = len(self.entries)
        self.entries = [
            e
            for e in self.entries
            if self._cosine_similarity(
                query_embedding, e.query_embedding
            )
            < threshold
        ]
        return before - len(self.entries)

    def clear(self) -> None:
        self.entries = []

    @property
    def stats(self) -> dict:
        active = [e for e in self.entries if not e.is_expired()]
        total_hits = sum(e.hit_count for e in active)
        return {
            "total_entries": len(active),
            "total_hits": total_hits,
            "avg_hits_per_entry": (
                total_hits / len(active) if active else 0
            ),
        }
```

### Similarity Threshold Selection

The threshold is the most critical tuning parameter:

| Threshold | Behavior | Use Case |
|-----------|----------|----------|
| 0.98+ | Near-exact match only | Financial, medical (precision critical) |
| 0.92-0.97 | Paraphrase detection | General Q&A, support bots |
| 0.85-0.91 | Topically similar | Low-risk, high-volume |
| < 0.85 | Too loose | Not recommended -- returns wrong answers |

### Integration with RAG Pipeline

```python
class CachedRAGPipeline:
    """RAG pipeline with semantic caching as a first-class layer."""

    def __init__(self, retriever, llm, cache: SemanticCache):
        self.retriever = retriever
        self.llm = llm
        self.cache = cache

    def query(self, user_query: str) -> dict:
        # Layer 1: Check semantic cache
        cached = self.cache.get(user_query)
        if cached:
            return cached

        # Layer 2: Full RAG pipeline
        documents = self.retriever.invoke(user_query)
        context = "\n\n".join(doc.page_content for doc in documents)

        answer = self.llm.invoke(
            f"Answer based on context:\n{context}\n\n"
            f"Question: {user_query}"
        ).content

        sources = [doc.metadata for doc in documents]

        # Store in cache
        self.cache.put(user_query, answer, sources)

        return {
            "answer": answer,
            "sources": sources,
            "cache_hit": False,
        }
```

---

## Layer 2: Embedding Cache

### Concept

Embedding the same chunk of text multiple times during ingestion or retrieval is wasteful. An embedding cache stores computed vectors keyed by a hash of the input text.

```python
class EmbeddingCache:
    """Cache embedding vectors to avoid redundant API calls.

    Useful during both ingestion (many documents share similar
    passages) and retrieval (repeated queries).
    """

    def __init__(
        self,
        redis_client: redis.Redis,
        embedding_fn,
        prefix: str = "emb:",
        ttl: int = 86400,
    ):
        self.redis = redis_client
        self.embed_fn = embedding_fn
        self.prefix = prefix
        self.ttl = ttl
        self._hits = 0
        self._misses = 0

    def _cache_key(self, text: str) -> str:
        text_hash = hashlib.sha256(text.encode()).hexdigest()
        return f"{self.prefix}{text_hash}"

    def embed(self, text: str) -> list[float]:
        """Get embedding, using cache when available."""
        key = self._cache_key(text)
        cached = self.redis.get(key)

        if cached:
            self._hits += 1
            return json.loads(cached)

        self._misses += 1
        embedding = self.embed_fn(text)

        self.redis.setex(key, self.ttl, json.dumps(embedding))
        return embedding

    def embed_batch(self, texts: list[str]) -> list[list[float]]:
        """Batch embed with per-item caching."""
        results = [None] * len(texts)
        uncached_indices = []
        uncached_texts = []

        for i, text in enumerate(texts):
            key = self._cache_key(text)
            cached = self.redis.get(key)
            if cached:
                results[i] = json.loads(cached)
                self._hits += 1
            else:
                uncached_indices.append(i)
                uncached_texts.append(text)
                self._misses += 1

        if uncached_texts:
            new_embeddings = self.embed_fn(uncached_texts)
            for idx, emb in zip(uncached_indices, new_embeddings):
                results[idx] = emb
                key = self._cache_key(texts[idx])
                self.redis.setex(key, self.ttl, json.dumps(emb))

        return results

    @property
    def hit_rate(self) -> float:
        total = self._hits + self._misses
        return self._hits / total if total > 0 else 0.0
```

---

## Layer 3: Prompt Cache (LLM Provider Level)

### Concept

LLM providers cache the KV-cache for token prefixes. If the beginning of your prompt (system instructions + retrieved context) is identical across requests, the provider can skip re-processing those tokens.

### How It Works Across Providers

| Provider | Mechanism | Cache Granularity | Cost Savings | TTL |
|----------|-----------|-------------------|--------------|-----|
| Anthropic | Explicit `cache_control` markers | Block-level (1024+ tokens) | 90% on cached input | 5 min (ephemeral) |
| OpenAI | Automatic prefix matching | 128-token aligned | 50% on cached input | 5-10 min |
| Google | Context caching API | Full context | Up to 75% | Configurable |

### Anthropic Prompt Caching (Preview)

```python
from anthropic import Anthropic

client = Anthropic()

# The system prompt + context is cached across requests
# Only the user query changes per request
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "You are a helpful assistant for our documentation...",
        },
        {
            "type": "text",
            "text": large_context_from_retrieved_docs,  # 5000+ tokens
            "cache_control": {"type": "ephemeral"},
        },
    ],
    messages=[{"role": "user", "content": user_query}],
)

# Check cache performance
usage = response.usage
print(f"Input tokens: {usage.input_tokens}")
print(f"Cache read tokens: {usage.cache_read_input_tokens}")
print(f"Cache creation tokens: {usage.cache_creation_input_tokens}")
```

### OpenAI Automatic Caching

```python
from openai import OpenAI

client = OpenAI()

# OpenAI caches automatically when the prompt prefix matches
# No explicit cache markers needed
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {
            "role": "system",
            "content": system_prompt + "\n\n" + retrieved_context,
        },
        {"role": "user", "content": user_query},
    ],
)

# Cached tokens show in usage
print(f"Prompt tokens: {response.usage.prompt_tokens}")
print(
    f"Cached tokens: "
    f"{response.usage.prompt_tokens_details.cached_tokens}"
)
```

---

## Cache Invalidation Strategies

### The Fundamental Problem

RAG caches must be invalidated when the underlying knowledge base changes. A cached answer about "pricing" becomes wrong when the pricing page is updated.

```python
class CacheInvalidationManager:
    """Coordinate cache invalidation when documents change."""

    def __init__(
        self,
        semantic_cache: SemanticCache,
        embedding_cache: EmbeddingCache,
        redis_client: redis.Redis,
    ):
        self.semantic_cache = semantic_cache
        self.embedding_cache = embedding_cache
        self.redis = redis_client

    def on_document_updated(self, doc_id: str, doc_content: str) -> dict:
        """Called when a source document is updated."""
        invalidated = {
            "semantic_entries_removed": 0,
            "embedding_keys_removed": 0,
        }

        # 1. Invalidate semantic cache entries that used this document
        # Strategy: find cached entries whose sources include this doc
        for entry in list(self.semantic_cache.entries):
            source_ids = [s.get("doc_id") for s in entry.sources]
            if doc_id in source_ids:
                self.semantic_cache.entries.remove(entry)
                invalidated["semantic_entries_removed"] += 1

        # 2. Invalidate embedding cache for the old document content
        old_key = self.embedding_cache._cache_key(doc_content)
        if self.redis.delete(old_key):
            invalidated["embedding_keys_removed"] += 1

        return invalidated

    def on_bulk_reindex(self) -> None:
        """Called during full reindexing -- flush all caches."""
        self.semantic_cache.clear()
        # Embedding cache uses TTL, so it will expire naturally
        # For immediate flush:
        keys = self.redis.keys(f"{self.embedding_cache.prefix}*")
        if keys:
            self.redis.delete(*keys)
```

### TTL-Based vs Event-Based Invalidation

| Strategy | Mechanism | Staleness Window | Complexity |
|----------|-----------|------------------|------------|
| TTL only | Entries expire after fixed time | Up to TTL duration | Low |
| Event-driven | Invalidate on document change | Near-zero | Medium |
| Hybrid (recommended) | Events + TTL as safety net | Near-zero with fallback | Medium |
| Version-based | Cache key includes doc version hash | Zero | High |

---

## Monitoring and Observability

```python
import logging
from dataclasses import dataclass, field

logger = logging.getLogger(__name__)


@dataclass
class CacheMetrics:
    """Track cache performance across all layers."""

    semantic_hits: int = 0
    semantic_misses: int = 0
    embedding_hits: int = 0
    embedding_misses: int = 0
    prompt_cache_tokens_saved: int = 0
    total_tokens_processed: int = 0

    @property
    def semantic_hit_rate(self) -> float:
        total = self.semantic_hits + self.semantic_misses
        return self.semantic_hits / total if total > 0 else 0.0

    @property
    def embedding_hit_rate(self) -> float:
        total = self.embedding_hits + self.embedding_misses
        return self.embedding_hits / total if total > 0 else 0.0

    @property
    def prompt_cache_savings_pct(self) -> float:
        if self.total_tokens_processed == 0:
            return 0.0
        return self.prompt_cache_tokens_saved / self.total_tokens_processed

    def log_summary(self) -> None:
        logger.info(
            "Cache metrics -- "
            "Semantic hit rate: %.2f%%, "
            "Embedding hit rate: %.2f%%, "
            "Prompt cache savings: %.2f%%",
            self.semantic_hit_rate * 100,
            self.embedding_hit_rate * 100,
            self.prompt_cache_savings_pct * 100,
        )
```

---

## Choosing the Right Caching Strategy

### Decision Matrix

```
Q: Do users ask similar questions repeatedly?
  Yes -> Semantic query cache (biggest impact)
  No  -> Skip semantic cache

Q: Are you embedding the same text during ingestion?
  Yes -> Embedding cache (saves on ingestion costs)
  No  -> Skip embedding cache

Q: Do you use a large system prompt or retrieved context?
  Yes -> Prompt caching (saves on LLM input costs)
  No  -> Skip prompt cache

Q: How stale can cached answers be?
  Not at all  -> Event-driven invalidation + low TTL
  Minutes     -> TTL of 5-15 minutes
  Hours       -> TTL of 1-4 hours (typical for docs)
```

### Recommended Configurations by Use Case

| Use Case | Semantic Cache | Embedding Cache | Prompt Cache | TTL |
|----------|---------------|-----------------|-------------|-----|
| Customer support bot | Yes (high repetition) | Yes | Yes (shared system prompt) | 30 min |
| Internal docs search | Yes (moderate repetition) | Yes | Yes | 2 hours |
| Code generation | No (unique queries) | Yes | Yes (large context) | N/A |
| Real-time data (stocks) | No | Yes | Yes | N/A |

---

## Common Pitfalls

1. **Setting the similarity threshold too low.** A threshold of 0.85 for a semantic cache will return answers for tangentially related questions. Start at 0.95 and lower gradually while monitoring answer quality.
2. **Not invalidating on document updates.** Cached answers become stale when source documents change. Always implement invalidation hooks tied to your ingestion pipeline.
3. **Caching errors and empty results.** If the LLM returns "I don't know" or the retriever returns zero documents, caching that response means future users with valid queries get the same unhelpful answer. Only cache confident, well-sourced responses.
4. **Ignoring cache warm-up.** A cold cache provides no benefit. For support bots, pre-populate the cache with answers to the top 100 most common questions.
5. **Mixing cache layers incorrectly.** Semantic caching should be the outermost layer (checked first). Prompt caching is handled at the LLM provider level. Embedding caching sits in the middle. Inverting the order wastes resources.
6. **Not monitoring hit rates.** Without metrics, you cannot tell if caching is helping. Track hit rates per layer and alert when they drop below expected thresholds.

---

## References

- GPTCache documentation: https://gptcache.readthedocs.io/
- Anthropic prompt caching: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- OpenAI prompt caching: https://platform.openai.com/docs/guides/prompt-caching
- Redis vector similarity search: https://redis.io/docs/latest/develop/interact/search-and-query/query/vector-search/
- Bang et al. "GPTCache: An Open-Source Semantic Cache for LLM Applications" (2023)
