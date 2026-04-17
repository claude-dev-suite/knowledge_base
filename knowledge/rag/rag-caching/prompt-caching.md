# Prompt Caching -- Anthropic Ephemeral Cache, OpenAI, and Hierarchical L1/L2

## TL;DR

Prompt caching is an LLM-provider-level optimization that reuses the key-value (KV) attention cache for repeated token prefixes across requests. In RAG systems, where every request shares a large system prompt plus potentially the same retrieved context, prompt caching can reduce input token costs by 50-90% and cut time-to-first-token (TTFT) by 50-80%. Anthropic provides explicit cache control markers that give fine-grained control over what gets cached. OpenAI implements automatic prefix caching that requires no code changes. This article covers both provider implementations, hierarchical L1/L2 caching strategies that combine prompt caching with semantic caching, and production patterns for maximizing cache hit rates in RAG pipelines.

---

## How Prompt Caching Works

### The KV-Cache Problem

During LLM inference, each input token generates key and value tensors in every attention layer. For a 128K-token prompt, this computation is expensive. Prompt caching stores these KV tensors on the server so subsequent requests with the same token prefix skip the computation entirely.

```
Request 1: [System prompt (2000 tokens)] [Context (5000 tokens)] [Query A]
            ^--- Compute KV-cache for 7000 tokens ---^           ^-- New --^

Request 2: [System prompt (2000 tokens)] [Context (5000 tokens)] [Query B]
            ^--- Reuse cached KV-cache (instant) ---^            ^-- New --^

Savings: 7000 tokens of computation skipped on Request 2
```

### What Gets Cached

The cache operates on a **prefix basis**: the longest common prefix of tokens between requests is cached. Anything after the first divergence point must be recomputed.

```
Prompt structure for RAG:

[System instructions]    <- Stable across ALL requests (always cached)
[Retrieved documents]    <- Stable when same docs retrieved (often cached)
[Chat history]           <- Grows per conversation (partially cached)
[Current user query]     <- Different every request (never cached)
```

---

## Anthropic Prompt Caching

### Explicit Cache Control

Anthropic uses `cache_control` markers to designate cache breakpoints. Content before a breakpoint is cached for 5 minutes (ephemeral cache).

```python
from anthropic import Anthropic

client = Anthropic()


def rag_query_with_caching(
    system_instructions: str,
    retrieved_context: str,
    user_query: str,
    chat_history: list[dict] | None = None,
) -> dict:
    """Execute a RAG query with Anthropic prompt caching.

    Cache breakpoints are placed after:
    1. System instructions (stable across all queries)
    2. Retrieved context (stable for same retrieval results)
    """
    system_blocks = [
        # Block 1: System instructions -- cached aggressively
        {
            "type": "text",
            "text": system_instructions,
            "cache_control": {"type": "ephemeral"},
        },
        # Block 2: Retrieved documents -- cached when same docs
        {
            "type": "text",
            "text": f"Reference documents:\n\n{retrieved_context}",
            "cache_control": {"type": "ephemeral"},
        },
    ]

    messages = []
    if chat_history:
        messages.extend(chat_history)
    messages.append({"role": "user", "content": user_query})

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        system=system_blocks,
        messages=messages,
    )

    return {
        "answer": response.content[0].text,
        "usage": {
            "input_tokens": response.usage.input_tokens,
            "output_tokens": response.usage.output_tokens,
            "cache_read_tokens": response.usage.cache_read_input_tokens,
            "cache_creation_tokens": response.usage.cache_creation_input_tokens,
        },
    }


# Example usage
SYSTEM = """You are a technical documentation assistant.
Answer questions based strictly on the provided reference documents.
If the documents do not contain the answer, say so explicitly.
Always cite the specific document section you are referencing."""

context = """
## Document: Authentication Guide
OAuth 2.0 is the recommended authentication method for all API endpoints.
Configure your client ID and secret in the dashboard under Settings > API Keys.
Token expiration is set to 3600 seconds by default.

## Document: Rate Limiting
API requests are limited to 1000 per minute per API key.
Exceeding this limit returns a 429 status code with a Retry-After header.
"""

# First query -- cache creation
result1 = rag_query_with_caching(SYSTEM, context, "How do I authenticate?")
print(f"Cache created: {result1['usage']['cache_creation_tokens']} tokens")
# Cache created: ~800 tokens (system + context now cached)

# Second query (same context) -- cache hit
result2 = rag_query_with_caching(SYSTEM, context, "What are the rate limits?")
print(f"Cache read: {result2['usage']['cache_read_tokens']} tokens")
# Cache read: ~800 tokens (system + context served from cache)
```

### Cache Breakpoint Strategy for RAG

Where you place `cache_control` markers determines what gets cached:

```python
def build_cached_prompt(
    system: str,
    rag_context: str,
    few_shot_examples: list[dict],
    chat_history: list[dict],
    user_query: str,
) -> tuple[list[dict], list[dict]]:
    """Build a prompt with optimal cache breakpoint placement.

    Cache layers (from most stable to least stable):
    1. System instructions -- identical for ALL requests
    2. Few-shot examples -- identical for all requests of the same type
    3. RAG context -- varies by retrieval results
    4. Chat history -- varies by conversation
    5. User query -- varies every request
    """
    system_blocks = [
        # Layer 1: System instructions (most stable)
        {
            "type": "text",
            "text": system,
            "cache_control": {"type": "ephemeral"},
        },
    ]

    # Layer 2: Few-shot examples (if present)
    if few_shot_examples:
        examples_text = "\n\n".join(
            f"Example query: {ex['query']}\nExample answer: {ex['answer']}"
            for ex in few_shot_examples
        )
        system_blocks.append({
            "type": "text",
            "text": f"Examples:\n{examples_text}",
            "cache_control": {"type": "ephemeral"},
        })

    # Layer 3: RAG context (varies by retrieval)
    if rag_context:
        system_blocks.append({
            "type": "text",
            "text": f"Reference documents:\n{rag_context}",
            "cache_control": {"type": "ephemeral"},
        })

    # Layer 4: Chat history (varies by conversation)
    messages = []
    if chat_history:
        # Mark the last history message as a cache breakpoint
        for i, msg in enumerate(chat_history):
            if i == len(chat_history) - 1:
                messages.append({
                    "role": msg["role"],
                    "content": [
                        {
                            "type": "text",
                            "text": msg["content"],
                            "cache_control": {"type": "ephemeral"},
                        }
                    ],
                })
            else:
                messages.append(msg)

    # Layer 5: Current query (never cached)
    messages.append({"role": "user", "content": user_query})

    return system_blocks, messages
```

### Minimum Token Requirements

Anthropic requires cache breakpoints to span at least 1024 tokens (for Claude 3.5 Sonnet) or 2048 tokens (for Claude 3 Opus). If your system prompt is shorter than this, pad it or combine it with the first retrieved document.

```python
def ensure_minimum_cache_size(
    text: str, min_tokens: int = 1024
) -> str:
    """Ensure cached content meets minimum token requirements.

    Anthropic requires at least 1024 tokens for a cache breakpoint
    (Claude 3.5 Sonnet). If the text is too short, the cache marker
    is silently ignored.
    """
    # Rough estimate: 1 token ~ 4 characters for English
    estimated_tokens = len(text) // 4

    if estimated_tokens < min_tokens:
        # Option 1: Pad with detailed instructions
        padding = (
            "\n\nAdditional guidelines:\n"
            "- Provide step-by-step explanations when appropriate\n"
            "- Use code examples in Python when illustrating concepts\n"
            "- Reference specific section numbers from the documents\n"
            "- If multiple documents are relevant, synthesize them\n"
            "- Acknowledge uncertainty when the documents are ambiguous\n"
        )
        text += padding

    return text
```

---

## OpenAI Automatic Prompt Caching

### How It Works

OpenAI caches automatically with no code changes. The cache key is the longest prefix of tokens that matches a previous request, aligned to 128-token boundaries.

```python
from openai import OpenAI

client = OpenAI()


def rag_query_openai(
    system_prompt: str,
    context: str,
    query: str,
) -> dict:
    """RAG query with OpenAI automatic prompt caching.

    No cache markers needed -- OpenAI detects common prefixes
    automatically and caches them for 5-10 minutes.
    """
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {
                "role": "system",
                "content": f"{system_prompt}\n\nContext:\n{context}",
            },
            {"role": "user", "content": query},
        ],
    )

    usage = response.usage
    cached_tokens = 0
    if hasattr(usage, "prompt_tokens_details") and usage.prompt_tokens_details:
        cached_tokens = usage.prompt_tokens_details.cached_tokens or 0

    return {
        "answer": response.choices[0].message.content,
        "prompt_tokens": usage.prompt_tokens,
        "cached_tokens": cached_tokens,
        "cache_hit_pct": (
            cached_tokens / usage.prompt_tokens * 100
            if usage.prompt_tokens > 0
            else 0
        ),
    }
```

### Maximizing OpenAI Cache Hits

Since OpenAI caches on prefix alignment, the order of your prompt matters:

```python
def build_cache_friendly_prompt_openai(
    system: str,
    context_docs: list[str],
    query: str,
) -> list[dict]:
    """Build prompt optimized for OpenAI automatic caching.

    Key principles:
    1. Static content first (system prompt)
    2. Semi-static content next (retrieved docs, sorted deterministically)
    3. Dynamic content last (user query)

    Sorting documents ensures the same set of docs produces
    the same token prefix regardless of retrieval order.
    """
    # Sort documents deterministically to maximize prefix overlap
    sorted_docs = sorted(context_docs)
    context = "\n\n---\n\n".join(sorted_docs)

    return [
        {
            "role": "system",
            "content": f"{system}\n\nReference documents:\n{context}",
        },
        {"role": "user", "content": query},
    ]
```

---

## Hierarchical L1/L2 Caching

### Architecture

Combine application-level semantic caching (L1) with provider-level prompt caching (L2) for maximum cost reduction:

```
User query
    |
    v
[L1: Semantic cache]  <-- Application-level (GPTCache / Redis)
    |                      Returns full answer on hit
    miss                   Cost: ~$0.0001 (embedding only)
    |
    v
[Embedding + retrieval]
    |
    v
[L2: Prompt cache]    <-- Provider-level (Anthropic / OpenAI)
    |                      Reduces input token cost on hit
    |                      Cost: 10% of input (Anthropic) or 50% (OpenAI)
    v
[LLM generation]
    |
    v
[Store answer in L1]
    |
    v
Return answer
```

### Implementation

```python
import hashlib
import json
import time

import numpy as np
import redis
from anthropic import Anthropic
from openai import OpenAI


class HierarchicalCachedRAG:
    """Two-layer caching: semantic (L1) + prompt (L2).

    L1 (semantic cache): Intercepts semantically similar queries
    and returns cached answers without any LLM call.

    L2 (prompt cache): When L1 misses, the LLM call benefits from
    provider-level prompt caching on the system prompt and context.
    """

    def __init__(
        self,
        retriever,
        redis_url: str = "redis://localhost:6379",
        similarity_threshold: float = 0.93,
        l1_ttl: int = 1800,
    ):
        self.retriever = retriever
        self.anthropic = Anthropic()
        self.openai = OpenAI()
        self.redis = redis.from_url(redis_url)
        self.threshold = similarity_threshold
        self.l1_ttl = l1_ttl

        # Metrics
        self.l1_hits = 0
        self.l1_misses = 0
        self.l2_tokens_saved = 0
        self.total_input_tokens = 0

    def _embed_query(self, query: str) -> list[float]:
        response = self.openai.embeddings.create(
            model="text-embedding-3-small", input=query
        )
        return response.data[0].embedding

    def _l1_lookup(
        self, query_embedding: list[float]
    ) -> dict | None:
        """Search L1 semantic cache."""
        # Scan cached entries (in production, use Redis vector search)
        cursor = 0
        while True:
            cursor, keys = self.redis.scan(
                cursor, match="l1:*", count=100
            )
            for key in keys:
                data = self.redis.hgetall(key)
                if not data:
                    continue
                cached_emb = json.loads(data[b"embedding"])
                similarity = self._cosine_sim(
                    query_embedding, cached_emb
                )
                if similarity >= self.threshold:
                    self.l1_hits += 1
                    return {
                        "answer": data[b"answer"].decode(),
                        "sources": json.loads(data[b"sources"]),
                        "cache_layer": "L1",
                        "similarity": similarity,
                    }
            if cursor == 0:
                break

        self.l1_misses += 1
        return None

    def _l1_store(
        self,
        query: str,
        query_embedding: list[float],
        answer: str,
        sources: list[dict],
    ) -> None:
        """Store in L1 semantic cache."""
        entry_id = hashlib.sha256(
            f"{query}:{time.time()}".encode()
        ).hexdigest()[:16]
        key = f"l1:{entry_id}"
        self.redis.hset(
            key,
            mapping={
                "query": query,
                "embedding": json.dumps(query_embedding),
                "answer": answer,
                "sources": json.dumps(sources),
                "created_at": str(time.time()),
            },
        )
        self.redis.expire(key, self.l1_ttl)

    def _l2_generate(
        self,
        system_prompt: str,
        context: str,
        query: str,
    ) -> dict:
        """Generate with L2 prompt caching (Anthropic)."""
        response = self.anthropic.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            system=[
                {
                    "type": "text",
                    "text": system_prompt,
                    "cache_control": {"type": "ephemeral"},
                },
                {
                    "type": "text",
                    "text": f"Context:\n{context}",
                    "cache_control": {"type": "ephemeral"},
                },
            ],
            messages=[{"role": "user", "content": query}],
        )

        usage = response.usage
        self.l2_tokens_saved += usage.cache_read_input_tokens
        self.total_input_tokens += usage.input_tokens

        return {
            "answer": response.content[0].text,
            "l2_cache_read_tokens": usage.cache_read_input_tokens,
            "l2_cache_creation_tokens": usage.cache_creation_input_tokens,
        }

    def query(
        self,
        user_query: str,
        system_prompt: str = "Answer based on context.",
    ) -> dict:
        """Execute query with L1 + L2 caching."""
        # Step 1: Embed query (needed for both L1 and retrieval)
        query_embedding = self._embed_query(user_query)

        # Step 2: L1 semantic cache lookup
        l1_result = self._l1_lookup(query_embedding)
        if l1_result:
            return l1_result

        # Step 3: Retrieve documents
        docs = self.retriever.invoke(user_query)
        context = "\n\n".join(d.page_content for d in docs)
        sources = [d.metadata for d in docs]

        # Step 4: Generate with L2 prompt caching
        gen_result = self._l2_generate(
            system_prompt, context, user_query
        )

        # Step 5: Store in L1 for future queries
        self._l1_store(
            user_query,
            query_embedding,
            gen_result["answer"],
            sources,
        )

        return {
            "answer": gen_result["answer"],
            "sources": sources,
            "cache_layer": "L2" if gen_result["l2_cache_read_tokens"] > 0 else "MISS",
            "l2_tokens_saved": gen_result["l2_cache_read_tokens"],
        }

    @staticmethod
    def _cosine_sim(a: list[float], b: list[float]) -> float:
        a_arr, b_arr = np.array(a), np.array(b)
        return float(
            np.dot(a_arr, b_arr)
            / (np.linalg.norm(a_arr) * np.linalg.norm(b_arr))
        )

    @property
    def metrics(self) -> dict:
        total = self.l1_hits + self.l1_misses
        return {
            "l1_hit_rate": self.l1_hits / total if total > 0 else 0,
            "l1_hits": self.l1_hits,
            "l1_misses": self.l1_misses,
            "l2_tokens_saved": self.l2_tokens_saved,
            "total_input_tokens": self.total_input_tokens,
            "l2_savings_pct": (
                self.l2_tokens_saved / self.total_input_tokens
                if self.total_input_tokens > 0
                else 0
            ),
        }
```

---

## Prompt Caching Optimization Patterns

### Pattern 1: Stable Prefix Ordering

Maximize cache hits by putting stable content at the beginning of the prompt:

```python
def order_prompt_for_caching(
    system: str,
    global_context: str,
    query_specific_context: str,
    query: str,
) -> list[dict]:
    """Order prompt components from most stable to least stable.

    Stability ranking:
    1. System instructions (identical across all queries)
    2. Global context (shared knowledge, updated infrequently)
    3. Query-specific context (varies by retrieval results)
    4. User query (different every request)
    """
    combined_system = (
        f"{system}\n\n"
        f"Global knowledge base:\n{global_context}"
    )

    return {
        "system": [
            {
                "type": "text",
                "text": combined_system,
                "cache_control": {"type": "ephemeral"},
            },
            {
                "type": "text",
                "text": f"Retrieved context:\n{query_specific_context}",
                "cache_control": {"type": "ephemeral"},
            },
        ],
        "messages": [{"role": "user", "content": query}],
    }
```

### Pattern 2: Context Deduplication

When multiple queries retrieve overlapping documents, deduplicate and sort to maximize prefix overlap:

```python
def deduplicate_context_for_caching(
    documents: list[dict],
    sort_key: str = "doc_id",
) -> str:
    """Deduplicate and deterministically sort retrieved documents.

    This ensures that when two different queries retrieve
    overlapping document sets, the shared prefix is maximized
    for prompt caching.
    """
    seen = set()
    unique_docs = []
    for doc in documents:
        doc_id = doc.get(sort_key, doc.get("content", "")[:100])
        if doc_id not in seen:
            seen.add(doc_id)
            unique_docs.append(doc)

    # Sort deterministically
    unique_docs.sort(key=lambda d: d.get(sort_key, ""))

    return "\n\n---\n\n".join(
        f"[{d.get(sort_key, 'unknown')}]\n{d['content']}"
        for d in unique_docs
    )
```

### Pattern 3: Cache Warming

Pre-warm the prompt cache by making dummy requests with the system prompt:

```python
async def warm_prompt_cache(
    client: Anthropic,
    system_blocks: list[dict],
    model: str = "claude-sonnet-4-20250514",
) -> dict:
    """Warm the Anthropic prompt cache with a minimal request.

    After warming, subsequent requests within 5 minutes that
    share the same system prefix will benefit from cache reads.
    """
    response = client.messages.create(
        model=model,
        max_tokens=10,
        system=system_blocks,
        messages=[
            {
                "role": "user",
                "content": "Acknowledge.",
            },
        ],
    )

    return {
        "cache_creation_tokens": response.usage.cache_creation_input_tokens,
        "warmed": True,
    }
```

---

## Cost Comparison

### Per-Query Cost Analysis

```
Scenario: RAG query with 500-token system prompt + 4000-token context

Without caching:
  Input tokens: 4500
  Cost (Claude Sonnet): 4500 * $3/M = $0.0135

With Anthropic prompt caching (cache hit):
  Cached input tokens: 4500 * $0.30/M = $0.00135   (90% discount)
  New input tokens: ~50 (query only) * $3/M = $0.00015
  Total: $0.0015 per query

With L1 semantic cache hit:
  Embedding only: ~$0.0001
  No LLM call at all

Combined L1 (30% hit) + L2 (70% of misses hit prompt cache):
  Effective cost per query: $0.0001 * 0.30 + $0.0015 * 0.49 + $0.0135 * 0.21
  = $0.00003 + $0.000735 + $0.002835
  = $0.0036 per query (73% savings vs no caching)
```

---

## Monitoring Prompt Cache Performance

```python
import logging
from dataclasses import dataclass, field

logger = logging.getLogger(__name__)


@dataclass
class PromptCacheMonitor:
    """Track prompt cache hit rates and cost savings."""

    requests: int = 0
    total_input_tokens: int = 0
    cache_read_tokens: int = 0
    cache_creation_tokens: int = 0
    uncached_tokens: int = 0

    def record(self, usage: dict) -> None:
        self.requests += 1
        self.total_input_tokens += usage.get("input_tokens", 0)
        self.cache_read_tokens += usage.get("cache_read_input_tokens", 0)
        self.cache_creation_tokens += usage.get(
            "cache_creation_input_tokens", 0
        )
        self.uncached_tokens += usage.get("input_tokens", 0) - usage.get(
            "cache_read_input_tokens", 0
        )

    @property
    def hit_rate(self) -> float:
        if self.total_input_tokens == 0:
            return 0.0
        return self.cache_read_tokens / self.total_input_tokens

    @property
    def estimated_savings_usd(self) -> float:
        """Estimate cost savings from cache reads (Anthropic pricing)."""
        # Cache reads cost 10% of normal input
        normal_cost = self.cache_read_tokens * 3.0 / 1_000_000
        cached_cost = self.cache_read_tokens * 0.30 / 1_000_000
        return normal_cost - cached_cost

    def log_summary(self) -> None:
        logger.info(
            "Prompt cache -- Requests: %d, Hit rate: %.1f%%, "
            "Tokens saved: %d, Est. savings: $%.4f",
            self.requests,
            self.hit_rate * 100,
            self.cache_read_tokens,
            self.estimated_savings_usd,
        )
```

---

## Common Pitfalls

1. **Putting dynamic content before static content.** If the user query appears before the system prompt in the message list, the entire prompt diverges on the first token and nothing is cached. Always put stable content first.
2. **Not meeting Anthropic's minimum token requirement.** Cache breakpoints require at least 1024 tokens (Sonnet) or 2048 tokens (Opus). Short system prompts will not be cached even with `cache_control` markers.
3. **Assuming prompt caching persists long-term.** Anthropic's ephemeral cache lasts 5 minutes. OpenAI's automatic cache lasts 5-10 minutes. For infrequent queries, the cache will be cold on every request.
4. **Randomizing document order in the prompt.** If retrieved documents appear in different orders across requests (due to non-deterministic scoring), the prefix diverges early and prompt caching is defeated. Sort documents deterministically.
5. **Ignoring the cache creation cost.** Anthropic charges 25% more for cache creation tokens. If your hit rate is low (below 3-4 requests within the 5-minute TTL), caching costs more than it saves.
6. **Not monitoring cache_read_input_tokens.** Without checking the usage object, you have no visibility into whether prompt caching is actually working. Log and alert on cache hit rates.

---

## References

- Anthropic prompt caching guide: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- OpenAI prompt caching: https://platform.openai.com/docs/guides/prompt-caching
- Google context caching: https://ai.google.dev/gemini-api/docs/caching
- Kwon et al. "Efficient Memory Management for Large Language Model Serving with PagedAttention" (2023)
