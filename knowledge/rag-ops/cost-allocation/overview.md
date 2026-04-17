# Cost Allocation -- Token Counting and Structured Logging

## Overview

Cost allocation for RAG systems tracks every token consumed across embedding, retrieval, and generation to attribute costs to teams, features, customers, or projects. Without cost allocation, LLM spending becomes an opaque line item that grows unchecked. With it, you know exactly which tenant costs $500/month, which feature consumes 80% of tokens, and where optimization will have the most impact.

This guide covers the foundations: token counting, structured logging of LLM calls, and the data model for cost attribution.

---

## Why Cost Allocation Matters

### The Problem

A typical RAG system makes multiple LLM calls per user query:

```
User query "How do I deploy to production?"
  -> Embedding API call (query embedding):     ~50 tokens
  -> Vector search:                            ~0 tokens (compute cost, not token)
  -> Reranking API call:                       ~2,000 tokens (query + 10 passages)
  -> Generation API call:                      ~3,000 input + ~500 output tokens
  -> Total per query:                          ~5,550 tokens
```

At 100,000 queries/month with GPT-4o-mini, that is approximately $0.15/1M input + $0.60/1M output:
- Input: 500M tokens * $0.15/1M = $75
- Output: 50M tokens * $0.60/1M = $30
- Embedding: 5M tokens * $0.02/1M = $0.10
- **Total: ~$105/month**

Without allocation, you cannot tell which of your 50 tenants drives most of this cost.

---

## Token Counting

### Counting Tokens Before API Calls

```python
import tiktoken


def count_tokens(text: str, model: str = "gpt-4o-mini") -> int:
    """Count tokens for a given text and model."""
    try:
        encoding = tiktoken.encoding_for_model(model)
    except KeyError:
        encoding = tiktoken.get_encoding("cl100k_base")
    return len(encoding.encode(text))


def count_chat_tokens(
    messages: list[dict],
    model: str = "gpt-4o-mini",
) -> int:
    """Count tokens for a chat completion request."""
    encoding = tiktoken.encoding_for_model(model)

    # Every message has overhead tokens for role/formatting
    tokens_per_message = 3  # <|start|>role<|end|>
    tokens_per_name = 1

    total = 0
    for message in messages:
        total += tokens_per_message
        for key, value in message.items():
            total += len(encoding.encode(str(value)))
            if key == "name":
                total += tokens_per_name

    total += 3  # <|start|>assistant<|message|> priming
    return total


def count_embedding_tokens(texts: list[str], model: str = "text-embedding-3-small") -> int:
    """Count tokens for embedding requests."""
    encoding = tiktoken.get_encoding("cl100k_base")
    return sum(len(encoding.encode(text)) for text in texts)


# Usage
messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "What is vector search?"},
]

input_tokens = count_chat_tokens(messages)
print(f"Input tokens: {input_tokens}")
```

### Token Counting for Anthropic

```python
import anthropic


def count_anthropic_tokens(
    messages: list[dict],
    model: str = "claude-sonnet-4-20250514",
    system: str = "",
) -> int:
    """Count tokens for an Anthropic message request."""
    client = anthropic.Anthropic()

    result = client.messages.count_tokens(
        model=model,
        messages=messages,
        system=system,
    )

    return result.input_tokens
```

---

## Structured Logging

### Log Schema

Every LLM API call should produce a structured log entry:

```python
from dataclasses import dataclass, field, asdict
from datetime import datetime
from typing import Optional
import json
import uuid


@dataclass
class LLMCallLog:
    """Structured log entry for an LLM API call."""
    # Identity
    call_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    timestamp: str = field(default_factory=lambda: datetime.utcnow().isoformat() + "Z")

    # Attribution
    tenant_id: str = ""
    user_id: str = ""
    project_id: str = ""
    feature: str = ""           # "search", "generation", "evaluation", "embedding"
    pipeline_step: str = ""     # "query_embed", "rerank", "generate", "judge"

    # Request
    provider: str = ""          # "openai", "anthropic", "voyage"
    model: str = ""
    endpoint: str = ""          # "/v1/embeddings", "/v1/chat/completions"

    # Tokens
    input_tokens: int = 0
    output_tokens: int = 0
    total_tokens: int = 0

    # Cost (in USD)
    input_cost: float = 0.0
    output_cost: float = 0.0
    total_cost: float = 0.0

    # Performance
    latency_ms: float = 0.0
    status: str = "success"     # "success", "error", "timeout"
    error_message: Optional[str] = None

    # Context
    request_metadata: dict = field(default_factory=dict)

    def to_json(self) -> str:
        return json.dumps(asdict(self))
```

### Cost Calculator

```python
# Pricing lookup (update as prices change)
PRICING = {
    "gpt-4o-mini": {"input": 0.15, "output": 0.60},
    "gpt-4o": {"input": 2.50, "output": 10.00},
    "claude-sonnet-4-20250514": {"input": 3.00, "output": 15.00},
    "claude-haiku-4-20250514": {"input": 0.80, "output": 4.00},
    "text-embedding-3-small": {"input": 0.02, "output": 0.0},
    "text-embedding-3-large": {"input": 0.13, "output": 0.0},
    "voyage-3": {"input": 0.06, "output": 0.0},
}


def calculate_cost(model: str, input_tokens: int, output_tokens: int) -> dict:
    """Calculate cost for a single API call."""
    if model not in PRICING:
        return {"input_cost": 0, "output_cost": 0, "total_cost": 0}

    p = PRICING[model]
    input_cost = (input_tokens / 1_000_000) * p["input"]
    output_cost = (output_tokens / 1_000_000) * p["output"]

    return {
        "input_cost": input_cost,
        "output_cost": output_cost,
        "total_cost": input_cost + output_cost,
    }
```

### Logging Middleware for OpenAI

```python
import openai
import time
import logging

logger = logging.getLogger("llm_cost")


class CostTrackingOpenAI:
    """OpenAI client wrapper that logs all calls with cost attribution."""

    def __init__(self, tenant_id: str = "", project_id: str = ""):
        self.client = openai.OpenAI()
        self.tenant_id = tenant_id
        self.project_id = project_id

    def embed(
        self,
        texts: list[str],
        model: str = "text-embedding-3-small",
        feature: str = "embedding",
        pipeline_step: str = "embed",
    ) -> tuple[list[list[float]], LLMCallLog]:
        """Embed texts with cost tracking."""
        start = time.perf_counter()

        try:
            response = self.client.embeddings.create(
                input=texts,
                model=model,
            )

            latency = (time.perf_counter() - start) * 1000
            usage = response.usage

            costs = calculate_cost(model, usage.total_tokens, 0)

            log = LLMCallLog(
                tenant_id=self.tenant_id,
                project_id=self.project_id,
                feature=feature,
                pipeline_step=pipeline_step,
                provider="openai",
                model=model,
                endpoint="/v1/embeddings",
                input_tokens=usage.total_tokens,
                output_tokens=0,
                total_tokens=usage.total_tokens,
                latency_ms=latency,
                **costs,
            )

            logger.info(log.to_json())

            embeddings = [d.embedding for d in response.data]
            return embeddings, log

        except Exception as e:
            latency = (time.perf_counter() - start) * 1000
            log = LLMCallLog(
                tenant_id=self.tenant_id,
                project_id=self.project_id,
                feature=feature,
                pipeline_step=pipeline_step,
                provider="openai",
                model=model,
                endpoint="/v1/embeddings",
                latency_ms=latency,
                status="error",
                error_message=str(e),
            )
            logger.error(log.to_json())
            raise

    def chat(
        self,
        messages: list[dict],
        model: str = "gpt-4o-mini",
        feature: str = "generation",
        pipeline_step: str = "generate",
        **kwargs,
    ) -> tuple[str, LLMCallLog]:
        """Chat completion with cost tracking."""
        start = time.perf_counter()

        try:
            response = self.client.chat.completions.create(
                model=model,
                messages=messages,
                **kwargs,
            )

            latency = (time.perf_counter() - start) * 1000
            usage = response.usage

            costs = calculate_cost(model, usage.prompt_tokens, usage.completion_tokens)

            log = LLMCallLog(
                tenant_id=self.tenant_id,
                project_id=self.project_id,
                feature=feature,
                pipeline_step=pipeline_step,
                provider="openai",
                model=model,
                endpoint="/v1/chat/completions",
                input_tokens=usage.prompt_tokens,
                output_tokens=usage.completion_tokens,
                total_tokens=usage.total_tokens,
                latency_ms=latency,
                **costs,
            )

            logger.info(log.to_json())

            content = response.choices[0].message.content
            return content, log

        except Exception as e:
            latency = (time.perf_counter() - start) * 1000
            log = LLMCallLog(
                tenant_id=self.tenant_id,
                project_id=self.project_id,
                feature=feature,
                pipeline_step=pipeline_step,
                provider="openai",
                model=model,
                endpoint="/v1/chat/completions",
                latency_ms=latency,
                status="error",
                error_message=str(e),
            )
            logger.error(log.to_json())
            raise


# Usage
client = CostTrackingOpenAI(tenant_id="acme-corp", project_id="knowledge-base")

embeddings, embed_log = client.embed(
    ["What is vector search?"],
    feature="search",
    pipeline_step="query_embed",
)

answer, gen_log = client.chat(
    messages=[{"role": "user", "content": "What is vector search?"}],
    feature="search",
    pipeline_step="generate",
)

print(f"Embed cost: ${embed_log.total_cost:.6f}")
print(f"Generate cost: ${gen_log.total_cost:.6f}")
print(f"Total cost: ${embed_log.total_cost + gen_log.total_cost:.6f}")
```

---

## Log Output Format

### JSON Log Lines (for ingestion into analytics systems)

```json
{
  "call_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "timestamp": "2025-04-15T14:30:00.000Z",
  "tenant_id": "acme-corp",
  "user_id": "user-123",
  "project_id": "knowledge-base",
  "feature": "search",
  "pipeline_step": "generate",
  "provider": "openai",
  "model": "gpt-4o-mini",
  "endpoint": "/v1/chat/completions",
  "input_tokens": 2500,
  "output_tokens": 350,
  "total_tokens": 2850,
  "input_cost": 0.000375,
  "output_cost": 0.000210,
  "total_cost": 0.000585,
  "latency_ms": 1250.5,
  "status": "success"
}
```

### Logging Destinations

| Destination | Use Case | Latency |
|-------------|----------|---------|
| stdout/JSON | Container logging (CloudWatch, Stackdriver) | None |
| File | Local development, log rotation | None |
| BigQuery | Analytics, BI dashboards | Batch (streaming insert) |
| ClickHouse | Real-time analytics, high cardinality | Low |
| Kafka | Event streaming, multiple consumers | Low |
| Langfuse | LLM-specific observability | Low |

---

## Common Pitfalls

1. **Counting tokens after the API call only**: API response usage may not include system prompt tokens or be delayed. Count tokens both before (for estimation) and after (for accuracy).

2. **Not tracking embedding costs**: Embedding tokens are cheap per call but add up. A RAG system making 100K queries/day at 50 tokens each = 5M tokens/day = $0.10/day just for query embeddings.

3. **Missing pipeline step attribution**: Logging only the model and tokens is insufficient. You need to know which step (embedding, reranking, generation) consumed the tokens to optimize the right component.

4. **Not updating pricing tables**: LLM pricing changes frequently. Centralize pricing in a config file and update it when providers change prices.

5. **Logging in the hot path synchronously**: If log ingestion is slow, it adds latency to every API call. Use async logging or buffer logs and flush periodically.

6. **Not normalizing across providers**: OpenAI, Anthropic, and Voyage count tokens differently. Normalize to a common unit (e.g., OpenAI cl100k_base tokens) for consistent cost comparison.

---

## References

- tiktoken: https://github.com/openai/tiktoken
- OpenAI token counting: https://platform.openai.com/docs/guides/text-generation/managing-tokens
- Anthropic token counting: https://docs.anthropic.com/en/docs/build-with-claude/token-counting
- Langfuse: https://langfuse.com/docs/model-usage-and-cost
