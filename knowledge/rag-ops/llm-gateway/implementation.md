# LLM Gateway -- Implementation: Setup, Fallback Chains, and Semantic Caching

## Overview

This guide covers implementing an LLM gateway for RAG systems: setting up LiteLLM as a self-hosted proxy, configuring fallback chains for reliability, implementing semantic caching to reduce costs, and integrating the gateway into a RAG pipeline.

---

## LiteLLM Proxy Setup

### Docker Deployment

```yaml
# docker-compose.yml
version: "3.8"

services:
  litellm:
    image: ghcr.io/berriai/litellm:main-latest
    ports:
      - "4000:4000"
    volumes:
      - ./litellm_config.yaml:/app/config.yaml
    environment:
      - LITELLM_MASTER_KEY=sk-my-secret-master-key
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - DATABASE_URL=postgresql://litellm:password@postgres:5432/litellm
    command: --config /app/config.yaml --port 4000 --detailed_debug
    depends_on:
      - postgres
    restart: unless-stopped

  postgres:
    image: postgres:16
    environment:
      - POSTGRES_DB=litellm
      - POSTGRES_USER=litellm
      - POSTGRES_PASSWORD=password
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  pgdata:
```

### Configuration

```yaml
# litellm_config.yaml
model_list:
  # Primary models
  - model_name: gpt-4o-mini
    litellm_params:
      model: openai/gpt-4o-mini
      api_key: os.environ/OPENAI_API_KEY
      max_retries: 2
      timeout: 30

  - model_name: gpt-4o-mini  # duplicate for load balancing
    litellm_params:
      model: openai/gpt-4o-mini
      api_key: os.environ/OPENAI_API_KEY_2  # second API key
      max_retries: 2

  - model_name: claude-sonnet
    litellm_params:
      model: anthropic/claude-sonnet-4-20250514
      api_key: os.environ/ANTHROPIC_API_KEY
      max_retries: 2
      timeout: 60

  - model_name: claude-haiku
    litellm_params:
      model: anthropic/claude-haiku-4-20250514
      api_key: os.environ/ANTHROPIC_API_KEY

  # Embedding models
  - model_name: embed-small
    litellm_params:
      model: openai/text-embedding-3-small
      api_key: os.environ/OPENAI_API_KEY

  - model_name: embed-large
    litellm_params:
      model: openai/text-embedding-3-large
      api_key: os.environ/OPENAI_API_KEY

# Routing
router_settings:
  routing_strategy: "least-busy"
  num_retries: 3
  retry_after: 5
  timeout: 60
  allowed_fails: 3
  cooldown_time: 60
  fallbacks:
    - gpt-4o-mini: [claude-haiku]
    - claude-sonnet: [gpt-4o-mini]

# Budget controls
general_settings:
  master_key: sk-my-secret-master-key
  max_budget: 500.0           # monthly budget USD
  budget_duration: "30d"

# Logging
litellm_settings:
  success_callback: ["langfuse"]
  failure_callback: ["langfuse"]
  langfuse_public_key: os.environ/LANGFUSE_PUBLIC_KEY
  langfuse_secret_key: os.environ/LANGFUSE_SECRET_KEY
```

### Using the Proxy

```python
from openai import OpenAI

# Point any OpenAI-compatible client at the proxy
client = OpenAI(
    base_url="http://localhost:4000/v1",
    api_key="sk-my-secret-master-key",
)

# Chat completion (routes through gateway)
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "What is vector search?"}],
    temperature=0.0,
)
print(response.choices[0].message.content)

# Embedding (routes through gateway)
response = client.embeddings.create(
    model="embed-small",
    input=["What is semantic search?"],
)
print(f"Embedding dimensions: {len(response.data[0].embedding)}")

# If gpt-4o-mini fails, gateway automatically falls back to claude-haiku
```

---

## Fallback Chain Implementation

### Custom Fallback with Portkey

```python
from portkey_ai import Portkey, Config

# Define a three-tier fallback chain
config = Config(
    strategy="fallback",
    targets=[
        # Tier 1: Fastest and cheapest
        {
            "virtual_key": "openai-key",
            "override_params": {"model": "gpt-4o-mini"},
            "retry": {"attempts": 2, "on_status_codes": [429, 500, 502, 503]},
        },
        # Tier 2: If OpenAI is down
        {
            "virtual_key": "anthropic-key",
            "override_params": {"model": "claude-haiku-4-20250514"},
            "retry": {"attempts": 2, "on_status_codes": [429, 500, 502, 503]},
        },
        # Tier 3: Self-hosted fallback (always available)
        {
            "virtual_key": "ollama-key",
            "override_params": {"model": "llama3.1:8b"},
        },
    ],
)

client = Portkey(api_key="your-portkey-key", config=config)

# This call automatically tries all three tiers
response = client.chat.completions.create(
    messages=[{"role": "user", "content": "What is RAG?"}],
)
```

### Custom Fallback in Pure Python

```python
import openai
import anthropic
import time
import logging
from dataclasses import dataclass
from typing import Optional

logger = logging.getLogger("llm_gateway")


@dataclass
class ProviderConfig:
    name: str
    model: str
    client: object
    priority: int
    max_retries: int = 2
    timeout: float = 30.0
    cooldown_until: float = 0.0  # timestamp when cooldown expires


class FallbackGateway:
    """LLM gateway with fallback chain and circuit breaker."""

    def __init__(self, providers: list[ProviderConfig]):
        self.providers = sorted(providers, key=lambda p: p.priority)

    def chat(
        self,
        messages: list[dict],
        temperature: float = 0.0,
        max_tokens: int = 500,
    ) -> dict:
        """Try providers in priority order until one succeeds."""
        errors = []

        for provider in self.providers:
            # Skip providers in cooldown
            if time.time() < provider.cooldown_until:
                continue

            for attempt in range(provider.max_retries + 1):
                try:
                    result = self._call_provider(
                        provider, messages, temperature, max_tokens,
                    )
                    return {
                        "content": result,
                        "provider": provider.name,
                        "model": provider.model,
                        "attempt": attempt + 1,
                    }
                except Exception as e:
                    errors.append({
                        "provider": provider.name,
                        "attempt": attempt + 1,
                        "error": str(e),
                    })
                    logger.warning(
                        f"{provider.name} attempt {attempt + 1} failed: {e}"
                    )

                    if attempt < provider.max_retries:
                        time.sleep(min(2 ** attempt, 10))

            # All retries exhausted for this provider, set cooldown
            provider.cooldown_until = time.time() + 60
            logger.error(f"{provider.name} exhausted retries, cooling down 60s")

        raise RuntimeError(f"All providers failed: {errors}")

    def _call_provider(
        self,
        provider: ProviderConfig,
        messages: list[dict],
        temperature: float,
        max_tokens: int,
    ) -> str:
        """Call a single provider."""
        if provider.name.startswith("openai"):
            response = provider.client.chat.completions.create(
                model=provider.model,
                messages=messages,
                temperature=temperature,
                max_tokens=max_tokens,
                timeout=provider.timeout,
            )
            return response.choices[0].message.content

        elif provider.name.startswith("anthropic"):
            # Convert OpenAI messages to Anthropic format
            system = ""
            anthro_messages = []
            for msg in messages:
                if msg["role"] == "system":
                    system = msg["content"]
                else:
                    anthro_messages.append(msg)

            response = provider.client.messages.create(
                model=provider.model,
                messages=anthro_messages,
                system=system,
                temperature=temperature,
                max_tokens=max_tokens,
            )
            return response.content[0].text

        else:
            raise ValueError(f"Unknown provider type: {provider.name}")


# Setup
gateway = FallbackGateway([
    ProviderConfig(
        name="openai-primary",
        model="gpt-4o-mini",
        client=openai.OpenAI(),
        priority=1,
    ),
    ProviderConfig(
        name="anthropic-fallback",
        model="claude-haiku-4-20250514",
        client=anthropic.Anthropic(),
        priority=2,
    ),
])

# Use
result = gateway.chat(
    messages=[{"role": "user", "content": "What is RAG?"}],
)
print(f"Answer ({result['provider']}): {result['content']}")
```

---

## Semantic Caching

Semantic caching stores LLM responses and returns cached results for semantically similar (not just identical) queries. This reduces API costs by 30-70% for workloads with repetitive queries.

### Implementation with Redis and Embeddings

```python
import hashlib
import json
import time
import numpy as np
import redis
from typing import Optional
import openai


class SemanticCache:
    """LLM response cache based on semantic similarity."""

    def __init__(
        self,
        redis_url: str = "redis://localhost:6379",
        embedding_model: str = "text-embedding-3-small",
        similarity_threshold: float = 0.95,
        ttl_seconds: int = 3600,
        max_cache_size: int = 10000,
    ):
        self.redis = redis.from_url(redis_url)
        self.oai = openai.OpenAI()
        self.embedding_model = embedding_model
        self.threshold = similarity_threshold
        self.ttl = ttl_seconds
        self.max_cache_size = max_cache_size

    def get(self, messages: list[dict]) -> Optional[dict]:
        """Check cache for a semantically similar query."""
        query_text = self._messages_to_text(messages)
        query_embedding = self._embed(query_text)

        # Get all cached embeddings
        cache_keys = self.redis.keys("llm_cache:*")
        if not cache_keys:
            return None

        best_match = None
        best_score = 0.0

        for key in cache_keys[:self.max_cache_size]:
            cached = self.redis.hgetall(key)
            if not cached:
                continue

            cached_embedding = json.loads(cached[b"embedding"])
            similarity = self._cosine_similarity(query_embedding, cached_embedding)

            if similarity > best_score:
                best_score = similarity
                best_match = cached

        if best_match and best_score >= self.threshold:
            return {
                "content": best_match[b"response"].decode(),
                "cached": True,
                "similarity": best_score,
                "model": best_match[b"model"].decode(),
            }

        return None

    def set(
        self,
        messages: list[dict],
        response: str,
        model: str,
    ):
        """Cache a response with its semantic embedding."""
        query_text = self._messages_to_text(messages)
        query_embedding = self._embed(query_text)

        cache_key = f"llm_cache:{hashlib.sha256(query_text.encode()).hexdigest()[:16]}"

        self.redis.hset(cache_key, mapping={
            "query": query_text,
            "response": response,
            "model": model,
            "embedding": json.dumps(query_embedding),
            "created_at": str(time.time()),
        })
        self.redis.expire(cache_key, self.ttl)

    def _messages_to_text(self, messages: list[dict]) -> str:
        """Convert messages to a single text for embedding."""
        parts = []
        for msg in messages:
            if msg["role"] == "user":
                parts.append(msg["content"])
        return " ".join(parts)

    def _embed(self, text: str) -> list[float]:
        """Get embedding for text."""
        response = self.oai.embeddings.create(
            input=text,
            model=self.embedding_model,
        )
        return response.data[0].embedding

    def _cosine_similarity(self, a: list[float], b: list[float]) -> float:
        """Compute cosine similarity between two vectors."""
        a = np.array(a)
        b = np.array(b)
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))


# Integration with gateway
class CachedGateway:
    """LLM gateway with semantic caching."""

    def __init__(self, gateway: FallbackGateway, cache: SemanticCache):
        self.gateway = gateway
        self.cache = cache

    def chat(self, messages: list[dict], **kwargs) -> dict:
        # Check cache first
        cached = self.cache.get(messages)
        if cached:
            return cached

        # Cache miss: call the gateway
        result = self.gateway.chat(messages, **kwargs)

        # Store in cache
        self.cache.set(messages, result["content"], result["model"])

        result["cached"] = False
        return result


# Usage
cache = SemanticCache(
    similarity_threshold=0.95,
    ttl_seconds=3600,
)
cached_gateway = CachedGateway(gateway, cache)

# First call: cache miss, calls API
result1 = cached_gateway.chat(
    messages=[{"role": "user", "content": "What is vector search?"}]
)
print(f"Cached: {result1.get('cached')}")  # False

# Similar query: cache hit
result2 = cached_gateway.chat(
    messages=[{"role": "user", "content": "Explain vector search"}]
)
print(f"Cached: {result2.get('cached')}")  # True (if similarity > 0.95)
```

---

## RAG Pipeline Integration

```python
class GatewayRAGPipeline:
    """RAG pipeline using an LLM gateway for all API calls."""

    def __init__(
        self,
        gateway_url: str = "http://localhost:4000/v1",
        gateway_key: str = "sk-my-key",
        embedding_model: str = "embed-small",
        generation_model: str = "gpt-4o-mini",
    ):
        self.client = openai.OpenAI(
            base_url=gateway_url,
            api_key=gateway_key,
        )
        self.embedding_model = embedding_model
        self.generation_model = generation_model

    def embed_query(self, query: str) -> list[float]:
        """Embed a query through the gateway."""
        response = self.client.embeddings.create(
            model=self.embedding_model,
            input=query,
        )
        return response.data[0].embedding

    def generate(
        self,
        question: str,
        context: list[str],
        model: str = None,
    ) -> dict:
        """Generate answer through the gateway."""
        context_text = "\n\n".join(
            f"[{i+1}] {c}" for i, c in enumerate(context)
        )

        response = self.client.chat.completions.create(
            model=model or self.generation_model,
            messages=[
                {
                    "role": "system",
                    "content": "Answer using ONLY the provided context. Cite passage numbers.",
                },
                {
                    "role": "user",
                    "content": f"Context:\n{context_text}\n\nQuestion: {question}",
                },
            ],
            temperature=0.0,
        )

        return {
            "answer": response.choices[0].message.content,
            "model": response.model,
            "tokens": response.usage.total_tokens,
        }


# Usage
pipeline = GatewayRAGPipeline(
    gateway_url="http://localhost:4000/v1",
    gateway_key="sk-my-key",
)

# All API calls go through the gateway
# If OpenAI is down, gateway falls back to Anthropic
embedding = pipeline.embed_query("What is vector search?")
result = pipeline.generate(
    question="What is vector search?",
    context=["Vector search finds similar items using embeddings..."],
)
print(result["answer"])
```

---

## Common Pitfalls

1. **Caching with user-specific context**: If your RAG retrieves different documents per user, caching the final answer is dangerous -- it may return tenant A's data to tenant B. Only cache at the LLM generation level, not at the RAG pipeline level, and include tenant context in the cache key.

2. **Gateway timeout shorter than LLM timeout**: If the gateway times out at 30s but the LLM needs 45s for a long response, requests fail. Set gateway timeout >= max LLM generation time.

3. **Fallback to incompatible model**: If your prompt uses OpenAI-specific features (function calling, JSON mode) that the fallback model does not support, the fallback fails. Test your prompts on all fallback models.

4. **Not monitoring cache hit rate**: Without monitoring, you do not know if caching is effective. Track hit rate, miss rate, and false positive rate (cached wrong answers).

5. **Semantic cache threshold too low**: A threshold of 0.85 may return cached answers for quite different questions. Start with 0.95 and lower gradually while monitoring quality.

6. **Single gateway instance**: A single LiteLLM proxy is a single point of failure. Run at least two instances behind a load balancer.

---

## References

- LiteLLM documentation: https://docs.litellm.ai/
- Portkey documentation: https://portkey.ai/docs/
- Redis caching patterns: https://redis.io/docs/manual/patterns/
- OpenAI API reference: https://platform.openai.com/docs/api-reference/
