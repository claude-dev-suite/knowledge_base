# RAG Latency Budgets

## Overview / TL;DR

A RAG query traverses six stages: query embedding, retrieval, re-ranking, context assembly, LLM generation (time-to-first-token and streaming), and post-processing. Each stage contributes measurable latency. Understanding where your milliseconds go is essential for meeting user-facing SLAs. This document breaks down real-world latency at p50/p95/p99 for each stage, identifies optimization opportunities, and quantifies the impact of caching, model selection, and infrastructure choices.

---

## End-to-End Latency Breakdown

The following numbers are based on production benchmarks using common infrastructure (cloud-hosted vector DB, API-based LLM, dedicated re-ranker). Your actual numbers will vary, but the relative proportions are remarkably consistent.

### Naive RAG (No Reranking)

| Stage | p50 | p95 | p99 | Notes |
|-------|-----|-----|-----|-------|
| Query embedding | 15ms | 30ms | 50ms | Local model (bge-base) |
| Query embedding | 40ms | 80ms | 150ms | API-based (OpenAI) |
| Vector retrieval | 10ms | 25ms | 50ms | Managed DB (Pinecone, Qdrant Cloud) |
| Vector retrieval | 3ms | 8ms | 15ms | Local DB (Chroma, Qdrant self-hosted) |
| Context assembly | 1ms | 2ms | 5ms | String concatenation + metadata |
| LLM TTFT | 300ms | 600ms | 1200ms | Claude Sonnet (API) |
| LLM TTFT | 200ms | 400ms | 800ms | GPT-4o-mini (API) |
| LLM streaming | 500ms | 1000ms | 2000ms | ~1000 token response |
| **Total (local embed + managed DB + Sonnet)** | **~830ms** | **~1660ms** | **~3300ms** | |
| **Total (API embed + managed DB + Sonnet)** | **~850ms** | **~1710ms** | **~3400ms** | |

### Advanced RAG (With Hybrid Search + Reranking)

| Stage | p50 | p95 | p99 | Notes |
|-------|-----|-----|-----|-------|
| Query rewriting (LLM) | 150ms | 300ms | 500ms | Haiku/GPT-4o-mini |
| Query embedding | 15ms | 30ms | 50ms | Local model |
| Vector retrieval | 10ms | 25ms | 50ms | Managed DB, top-20 |
| BM25 retrieval | 5ms | 15ms | 30ms | In-memory index |
| Score fusion | 1ms | 2ms | 3ms | RRF or weighted combination |
| Cross-encoder reranking | 50ms | 100ms | 200ms | Local model (bge-reranker), 20 candidates |
| Cross-encoder reranking | 200ms | 400ms | 800ms | API-based (Cohere Rerank) |
| Context assembly | 2ms | 3ms | 5ms | With citation formatting |
| LLM TTFT | 300ms | 600ms | 1200ms | Claude Sonnet |
| LLM streaming | 500ms | 1000ms | 2000ms | ~1000 token response |
| **Total (local reranker)** | **~1050ms** | **~2080ms** | **~4040ms** | |
| **Total (API reranker)** | **~1180ms** | **~2380ms** | **~4640ms** | |

### Agentic RAG (2-3 Retrieval Iterations)

| Stage | p50 | p95 | p99 | Notes |
|-------|-----|-----|-----|-------|
| Planning step | 500ms | 1000ms | 2000ms | Agent decides strategy |
| Iteration 1 (retrieve + evaluate) | 800ms | 1500ms | 3000ms | Full Advanced RAG |
| Iteration 2 (refine + retrieve) | 800ms | 1500ms | 3000ms | Second pass |
| Final generation | 800ms | 1500ms | 3000ms | Longer context = slower |
| **Total (2 iterations)** | **~2900ms** | **~5500ms** | **~11000ms** | |
| **Total (3 iterations)** | **~3700ms** | **~7000ms** | **~14000ms** | |

---

## Stage-by-Stage Optimization

### 1. Query Embedding (15-150ms)

**The optimization**: Use a local embedding model instead of an API call.

```python
# SLOW: API call adds network round-trip (~40-150ms)
from openai import OpenAI
client = OpenAI()
response = client.embeddings.create(
    input=query,
    model="text-embedding-3-small"
)
embedding = response.data[0].embedding

# FAST: Local model, GPU inference (~5-15ms with GPU, ~15-30ms CPU)
from sentence_transformers import SentenceTransformer
model = SentenceTransformer("BAAI/bge-base-en-v1.5", device="cuda")
embedding = model.encode(query, normalize_embeddings=True)
```

**Quantified impact**: Switching from API to local embedding saves 25-120ms per query at p50. For applications making multiple embedding calls (multi-query RAG), the savings multiply.

**Embedding cache**: Cache query embeddings for repeated or similar queries.

```python
import hashlib
from functools import lru_cache

@lru_cache(maxsize=10000)
def cached_embed(query_text: str) -> tuple:
    """Cache embeddings for repeated queries. Returns tuple for hashability."""
    embedding = model.encode(query_text, normalize_embeddings=True)
    return tuple(embedding.tolist())
```

### 2. Vector Retrieval (3-50ms)

Vector databases are already fast. Optimization focuses on index configuration and query patterns.

**HNSW tuning**:

| Parameter | Latency Impact | Recall Impact |
|-----------|---------------|---------------|
| `ef_search=50` (default) | Baseline | ~95% recall |
| `ef_search=100` | +30-50% latency | ~98% recall |
| `ef_search=200` | +100-150% latency | ~99.5% recall |
| `ef_search=20` | -40% latency | ~85% recall |

```python
# Qdrant example: tune ef at query time
from qdrant_client import QdrantClient
from qdrant_client.models import SearchParams

client = QdrantClient(host="localhost", port=6333)

# Fast but lower recall
fast_results = client.search(
    collection_name="docs",
    query_vector=embedding,
    limit=20,
    search_params=SearchParams(hnsw_ef=30),
)

# Slower but higher recall
accurate_results = client.search(
    collection_name="docs",
    query_vector=embedding,
    limit=20,
    search_params=SearchParams(hnsw_ef=200),
)
```

**Metadata filtering before vs after vector search**:

```python
# SLOW: Retrieve 1000 candidates, then filter in Python
results = collection.query(query_embeddings=[emb], n_results=1000)
filtered = [r for r in results if r.metadata["department"] == "engineering"]

# FAST: Filter at the DB level (pre-filtering)
results = collection.query(
    query_embeddings=[emb],
    n_results=20,
    where={"department": "engineering"},  # DB applies filter during search
)
```

Pre-filtering can reduce retrieval latency by 2-5x when the filter eliminates >50% of the corpus.

### 3. Re-ranking (50-800ms)

Re-ranking is typically the biggest optimization opportunity because it is the most variable stage.

**Local vs API re-ranker**:

| Re-ranker | 20 candidates p50 | 50 candidates p50 | Quality |
|-----------|-------------------|-------------------|---------|
| bge-reranker-v2-m3 (local GPU) | 30ms | 70ms | Good |
| bge-reranker-v2-m3 (local CPU) | 100ms | 250ms | Good |
| Cohere Rerank v3.5 (API) | 200ms | 350ms | Best |
| jina-reranker-v2 (API) | 180ms | 300ms | Good |
| LLM-based (Claude Haiku) | 400ms | 900ms | Variable |

**Optimization: Reduce candidate count**

The biggest lever is reducing the number of candidates passed to the re-ranker. Re-ranker latency scales roughly linearly with candidate count.

```python
# If you're passing 50 candidates to the reranker, try 20 first
# Measure recall@5 after reranking to verify quality doesn't drop

from sentence_transformers import CrossEncoder

reranker = CrossEncoder("BAAI/bge-reranker-v2-m3")

def rerank_with_budget(
    query: str,
    candidates: list[dict],
    max_candidates: int = 20,  # Limit candidates for speed
    top_k: int = 5,
) -> list[dict]:
    # Pre-filter to max_candidates by initial score
    candidates = sorted(candidates, key=lambda x: x["score"], reverse=True)
    candidates = candidates[:max_candidates]

    pairs = [(query, c["text"]) for c in candidates]
    scores = reranker.predict(pairs)
    for i, c in enumerate(candidates):
        c["rerank_score"] = float(scores[i])
    candidates.sort(key=lambda x: x["rerank_score"], reverse=True)
    return candidates[:top_k]
```

**Optimization: Batch re-ranking on GPU**

```python
# Single candidate at a time (slow)
for candidate in candidates:
    score = reranker.predict([(query, candidate["text"])])

# Batch all candidates (fast -- single GPU forward pass)
pairs = [(query, c["text"]) for c in candidates]
scores = reranker.predict(pairs, batch_size=32)
```

### 4. LLM Generation (200ms-3s+)

LLM generation is usually the largest single contributor to total latency.

**Model selection for latency**:

| Model | TTFT (p50) | Tokens/sec | 500-token response |
|-------|-----------|------------|-------------------|
| Claude Haiku 3.5 | 150ms | 120 tok/s | ~4.3s |
| Claude Sonnet 4 | 300ms | 80 tok/s | ~6.5s |
| GPT-4o-mini | 200ms | 100 tok/s | ~5.2s |
| GPT-4o | 350ms | 60 tok/s | ~8.7s |
| Llama 3.1 70B (self-hosted) | 100ms | 40-80 tok/s | ~6.4-12.6s |

Note: These numbers are approximate and vary by load, region, and prompt length. Longer prompts increase TTFT.

**Streaming hides perceived latency**:

```python
import anthropic

client = anthropic.Anthropic()

def stream_rag_response(question: str, context: str):
    """Stream the response -- user sees first token in ~300ms instead of
    waiting ~2-5s for the complete response."""
    with client.messages.stream(
        model="claude-sonnet-4-20250514",
        max_tokens=1500,
        system="Answer using the provided context. Cite sources.",
        messages=[{
            "role": "user",
            "content": f"Context:\n{context}\n\nQuestion: {question}",
        }],
    ) as stream:
        for text in stream.text_stream:
            yield text  # Send each chunk to the client immediately
```

**Reduce context size to reduce TTFT**:

TTFT scales with input token count. Every unnecessary token in the context adds latency.

| Context tokens | TTFT increase (approximate) |
|---------------|---------------------------|
| 1,000 | Baseline |
| 5,000 | +50-100ms |
| 10,000 | +100-200ms |
| 20,000 | +200-400ms |
| 50,000 | +500-1000ms |

**Practical implication**: Aggressive re-ranking that returns 3 focused chunks (1500 tokens) instead of 10 chunks (5000 tokens) can save 100-200ms on TTFT alone.

### 5. Query Rewriting (150-500ms)

Query rewriting adds a full LLM round-trip. This is often the most expensive pre-retrieval step.

**Use a fast model**:

```python
# SLOW: Using a large model for query rewriting
response = client.messages.create(
    model="claude-sonnet-4-20250514",  # Overkill for query rewriting
    max_tokens=100,
    messages=[{"role": "user", "content": f"Rewrite: {query}"}],
)

# FAST: Use the smallest model that works
response = client.messages.create(
    model="claude-haiku-4-20250514",  # 2-3x faster, 10x cheaper
    max_tokens=100,
    messages=[{"role": "user", "content": f"Rewrite: {query}"}],
)
```

**Cache rewrites for common queries**:

```python
import hashlib

_rewrite_cache = {}

def cached_rewrite(query: str) -> str:
    key = hashlib.sha256(query.lower().strip().encode()).hexdigest()
    if key in _rewrite_cache:
        return _rewrite_cache[key]
    rewritten = rewrite_query(query)
    _rewrite_cache[key] = rewritten
    return rewritten
```

**Skip rewriting for simple queries**: If the query is already a well-formed search query (short, specific, no conversational preamble), bypass rewriting entirely.

```python
def needs_rewriting(query: str) -> bool:
    """Heuristic: skip rewriting for simple, focused queries."""
    words = query.split()
    if len(words) <= 5:
        return False  # Already concise
    if query.endswith("?") and len(words) <= 8:
        return False  # Short question
    if any(word in query.lower() for word in ["please", "could you", "i want to"]):
        return True  # Conversational -> needs rewriting
    return len(words) > 12  # Long queries benefit from rewriting
```

---

## Caching Strategy

Caching is the single most impactful latency optimization. A cache hit eliminates the entire pipeline.

### Three-Layer Cache

```python
import hashlib
import json
import time
from typing import Optional


class RAGCache:
    """Three-layer cache: exact query, semantic similarity, component-level."""

    def __init__(self, redis_client, embed_model, ttl: int = 3600):
        self.redis = redis_client
        self.embed_model = embed_model
        self.ttl = ttl

    def _query_key(self, query: str) -> str:
        return f"rag:exact:{hashlib.sha256(query.lower().strip().encode()).hexdigest()}"

    # Layer 1: Exact query cache
    def get_exact(self, query: str) -> Optional[dict]:
        """Cache hit when the exact same query was asked before."""
        cached = self.redis.get(self._query_key(query))
        if cached:
            return json.loads(cached)
        return None

    def set_exact(self, query: str, result: dict):
        self.redis.setex(
            self._query_key(query),
            self.ttl,
            json.dumps(result),
        )

    # Layer 2: Semantic cache (similar queries)
    def get_semantic(self, query: str, threshold: float = 0.95) -> Optional[dict]:
        """Cache hit when a semantically similar query was asked before.
        Uses a small vector index of recent queries."""
        query_emb = self.embed_model.encode(query, normalize_embeddings=True)
        # Search the query cache index (a separate small vector collection)
        # Return cached result if similarity > threshold
        # Implementation depends on your vector DB
        pass

    # Layer 3: Component-level cache
    def get_embedding(self, text: str) -> Optional[list[float]]:
        """Cache embeddings to avoid re-computation."""
        cached = self.redis.get(f"rag:emb:{hashlib.md5(text.encode()).hexdigest()}")
        if cached:
            return json.loads(cached)
        return None

    def set_embedding(self, text: str, embedding: list[float]):
        self.redis.setex(
            f"rag:emb:{hashlib.md5(text.encode()).hexdigest()}",
            self.ttl * 24,  # Embeddings can be cached longer
            json.dumps(embedding),
        )
```

### Cache Hit Rates and Impact

| Cache Layer | Typical Hit Rate | Latency When Hit | Savings |
|-------------|-----------------|------------------|---------|
| Exact query | 10-30% (depends on query diversity) | 5ms (Redis lookup) | ~1-3s saved |
| Semantic cache | 5-15% | 20ms (embed + search) | ~1-3s saved |
| Embedding cache | 30-60% (queries repeat words) | 2ms (Redis lookup) | 15-40ms saved |
| Combined | 20-50% | 5-20ms | ~1-3s saved per hit |

---

## Latency Budget Templates

### User-Facing Chatbot (Target: < 2s p95 to first token)

```
Budget: 2000ms
  Query embedding (local):       15ms
  Hybrid retrieval:              20ms
  Re-ranking (local, 20 cands):  50ms
  Context assembly:               2ms
  LLM TTFT (streaming):        300ms
  --------------------------------
  Subtotal to first token:      387ms
  Buffer for p95:               600ms
  --------------------------------
  Headroom:                    1013ms  (comfortable)

Skip: Query rewriting (saves 150ms but not needed for chatbot)
```

### Internal Search Tool (Target: < 5s p95 full response)

```
Budget: 5000ms
  Query rewriting (Haiku):      200ms
  Multi-query (2 extra):        200ms  (parallel with rewrite)
  Query embedding (local):       15ms
  Hybrid retrieval (3 queries):  60ms  (parallel across queries)
  Re-ranking (local, 30 cands): 100ms
  Context assembly:               3ms
  LLM full response (Sonnet):  2500ms
  Post-processing:               10ms
  --------------------------------
  Subtotal:                    3088ms
  Buffer for p95:              1500ms
  --------------------------------
  Headroom:                     412ms  (tight -- monitor closely)
```

### Batch Processing (Target: throughput, not latency)

```
When latency doesn't matter, optimize for throughput:
  - Use larger batch sizes for embedding (64-128)
  - Use larger candidate sets for re-ranking (50-100)
  - Use the most accurate (but slower) LLM
  - Skip streaming (batch responses)
  - Target: 10-50 queries/minute sustained
```

---

## Measuring Latency in Production

```python
import time
import logging
from dataclasses import dataclass, field
from contextlib import contextmanager

logger = logging.getLogger(__name__)


@dataclass
class LatencyTracker:
    """Track per-stage latency for RAG pipelines."""
    stages: dict = field(default_factory=dict)

    @contextmanager
    def track(self, stage_name: str):
        start = time.perf_counter()
        yield
        elapsed_ms = (time.perf_counter() - start) * 1000
        self.stages[stage_name] = elapsed_ms

    def total_ms(self) -> float:
        return sum(self.stages.values())

    def to_dict(self) -> dict:
        return {**self.stages, "total_ms": self.total_ms()}

    def log(self, query_id: str):
        logger.info(
            "RAG latency breakdown",
            extra={
                "query_id": query_id,
                **self.to_dict(),
            },
        )


# Usage in a RAG pipeline
def rag_query_with_tracking(question: str) -> dict:
    tracker = LatencyTracker()

    with tracker.track("embed_query"):
        embedding = model.encode(question)

    with tracker.track("vector_search"):
        candidates = collection.query(query_embeddings=[embedding], n_results=20)

    with tracker.track("rerank"):
        top_chunks = reranker.predict([(question, c) for c in candidates])

    with tracker.track("llm_generate"):
        answer = generate(question, top_chunks)

    tracker.log(query_id="abc123")
    # Output: {"embed_query": 14.2, "vector_search": 8.1,
    #          "rerank": 52.3, "llm_generate": 1847.5, "total_ms": 1922.1}

    return {"answer": answer, "latency": tracker.to_dict()}
```

---

## Common Pitfalls

1. **Measuring only total latency.** Without per-stage breakdown, you optimize the wrong thing. The LLM usually dominates, but you might be wasting time optimizing it when an API re-ranker is the actual bottleneck.
2. **Ignoring p99 latency.** p50 looks great at 800ms, but p99 at 8s means 1 in 100 users waits 8 seconds. Set alerts on p95 and p99, not just average.
3. **Not streaming.** For user-facing applications, perceived latency (time-to-first-token) matters more than total latency. Streaming turns a 3s wait into a 300ms wait plus progressive rendering.
4. **Caching only at the response level.** Component-level caching (embeddings, retrievals) provides partial benefits even on cache misses at the response level.
5. **Using the same model for all LLM calls.** Query rewriting does not need Sonnet; Haiku is 3x faster and 10x cheaper for this task.
6. **Adding re-ranking without measuring its latency cost.** An API re-ranker adds 200-400ms. If your latency budget is 1.5s and the LLM takes 1s, you have no room for a slow re-ranker. Use a local model or reduce candidate count.
7. **Not parallelizing independent stages.** Multi-query retrieval can run in parallel. BM25 and vector search can run in parallel. Embedding and query rewriting can sometimes be parallelized (embed the original while rewriting).

---

## References

- Pinecone, "Understanding vector database performance" -- https://www.pinecone.io/learn/hnsw/
- Qdrant, "Benchmarks" -- https://qdrant.tech/benchmarks/
- Anthropic, "Streaming messages" -- https://docs.anthropic.com/en/api/streaming
- MTEB Leaderboard (embedding model benchmarks) -- https://huggingface.co/spaces/mteb/leaderboard
