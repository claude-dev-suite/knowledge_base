# LLM Gateway -- Portkey, OpenRouter, LiteLLM, and Kong AI

## Overview

An LLM Gateway is a proxy layer between your application and LLM providers. It unifies the API interface, adds reliability features (fallbacks, retries, load balancing), enables observability (cost tracking, latency monitoring), and provides governance controls (rate limiting, access control). Instead of calling OpenAI, Anthropic, and Cohere directly with different SDKs, your application calls one gateway endpoint that routes to the right provider.

This guide covers the four major LLM gateways, their architectures, and when to use each.

---

## Why Use an LLM Gateway

### Without a Gateway

```
Your Application
  |--- OpenAI SDK -----> OpenAI API
  |--- Anthropic SDK ---> Anthropic API
  |--- Cohere SDK ------> Cohere API
  |--- Custom HTTP -----> Self-hosted model

Problems:
- Multiple SDKs to maintain
- No unified error handling
- No automatic fallback
- Cost tracking per-provider
- Rate limiting per-provider
```

### With a Gateway

```
Your Application
  |
  v
+------------------+
|   LLM Gateway    |
|  (single API)    |
+--------+---------+
         |
    +----+----+----+----+
    |    |    |    |    |
    v    v    v    v    v
  OpenAI  Anthropic  Cohere  Self-hosted  Azure
```

### Core Gateway Features

| Feature | Description |
|---------|-------------|
| **Unified API** | One endpoint, one SDK, multiple providers |
| **Fallback chains** | If provider A fails, automatically try provider B |
| **Load balancing** | Distribute requests across providers or API keys |
| **Retries** | Automatic retry with exponential backoff |
| **Rate limiting** | Per-user, per-team, or per-model rate limits |
| **Cost tracking** | Centralized token counting and cost attribution |
| **Caching** | Cache repeated prompts/responses |
| **Semantic caching** | Cache based on semantic similarity, not exact match |
| **Logging** | Structured logs of all requests/responses |
| **Guardrails** | Content filtering, PII detection, prompt injection defense |

---

## Portkey

### Architecture

Portkey is a managed LLM gateway with an AI Gateway (open-source proxy) and a cloud platform:

```python
from portkey_ai import Portkey

# Initialize Portkey client
client = Portkey(
    api_key="your-portkey-api-key",
    # Optional: default provider config
    virtual_key="openai-virtual-key",  # maps to your OpenAI API key stored in Portkey
)

# Use like OpenAI SDK (unified interface)
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "What is RAG?"}],
    temperature=0.0,
)
print(response.choices[0].message.content)
```

### Portkey Features

```python
from portkey_ai import Portkey, Config, RetrySettings, CacheType

# Fallback configuration: try OpenAI first, then Anthropic
client = Portkey(
    api_key="your-portkey-api-key",
    config=Config(
        strategy="fallback",
        targets=[
            {"virtual_key": "openai-key", "override_params": {"model": "gpt-4o-mini"}},
            {"virtual_key": "anthropic-key", "override_params": {"model": "claude-haiku-4-20250514"}},
        ],
        retry=RetrySettings(attempts=3, on_status_codes=[429, 500, 502, 503]),
        cache=CacheType.SEMANTIC,  # cache similar prompts
    ),
)

# Load balancing across multiple API keys
client = Portkey(
    api_key="your-portkey-api-key",
    config=Config(
        strategy="loadbalance",
        targets=[
            {"virtual_key": "openai-key-1", "weight": 0.5},
            {"virtual_key": "openai-key-2", "weight": 0.3},
            {"virtual_key": "openai-key-3", "weight": 0.2},
        ],
    ),
)

# Request with metadata for cost tracking
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Explain vector search"}],
    metadata={
        "tenant_id": "acme-corp",
        "feature": "search",
        "environment": "production",
    },
)
```

---

## OpenRouter

### Architecture

OpenRouter is a unified API that routes to 100+ models from OpenAI, Anthropic, Google, Meta, Mistral, and more:

```python
from openai import OpenAI

# OpenRouter uses OpenAI-compatible API
client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="your-openrouter-api-key",
)

# Access any model through one endpoint
response = client.chat.completions.create(
    model="anthropic/claude-sonnet-4-20250514",
    messages=[{"role": "user", "content": "What is RAG?"}],
)
print(response.choices[0].message.content)

# Try different models easily
for model in [
    "openai/gpt-4o-mini",
    "anthropic/claude-haiku-4-20250514",
    "google/gemini-2.0-flash",
    "meta-llama/llama-3.1-70b-instruct",
]:
    response = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": "Explain RAG in one sentence."}],
        max_tokens=100,
    )
    print(f"{model}: {response.choices[0].message.content}")
```

### OpenRouter Features

```python
# Automatic routing: let OpenRouter pick the best model
response = client.chat.completions.create(
    model="openrouter/auto",  # automatic model selection
    messages=[{"role": "user", "content": "Complex analysis question..."}],
)

# With site identification (for analytics)
response = client.chat.completions.create(
    model="openai/gpt-4o-mini",
    messages=[{"role": "user", "content": "Hello"}],
    extra_headers={
        "HTTP-Referer": "https://myapp.com",
        "X-Title": "My RAG Application",
    },
)
```

---

## LiteLLM

### Architecture

LiteLLM is an open-source Python library and proxy server that provides a unified interface for 100+ LLM providers:

```python
import litellm

# Unified completion interface
response = litellm.completion(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "What is RAG?"}],
)
print(response.choices[0].message.content)

# Same interface for Anthropic
response = litellm.completion(
    model="claude-sonnet-4-20250514",
    messages=[{"role": "user", "content": "What is RAG?"}],
)

# Embedding (unified)
response = litellm.embedding(
    model="text-embedding-3-small",
    input=["What is semantic search?"],
)
```

### LiteLLM Proxy Server

```yaml
# litellm_config.yaml
model_list:
  - model_name: gpt-4o-mini
    litellm_params:
      model: openai/gpt-4o-mini
      api_key: sk-...

  - model_name: claude-sonnet
    litellm_params:
      model: anthropic/claude-sonnet-4-20250514
      api_key: sk-ant-...

  - model_name: local-llama
    litellm_params:
      model: ollama/llama3.1:8b
      api_base: http://localhost:11434

# Fallback configuration
router_settings:
  routing_strategy: "simple-shuffle"  # or "least-busy", "latency-based"
  num_retries: 3
  retry_after: 5
  fallbacks:
    - gpt-4o-mini: [claude-sonnet, local-llama]

# Budget controls
general_settings:
  master_key: sk-my-master-key
  max_budget: 100.0  # monthly budget in USD
```

```bash
# Start the proxy
litellm --config litellm_config.yaml --port 4000

# Use with any OpenAI-compatible client
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-my-master-key" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4o-mini", "messages": [{"role": "user", "content": "Hello"}]}'
```

### LiteLLM Fallback and Load Balancing

```python
from litellm import Router

# Router with fallback chains
router = Router(
    model_list=[
        {
            "model_name": "primary-gpt4",
            "litellm_params": {
                "model": "openai/gpt-4o-mini",
                "api_key": "sk-key1",
            },
        },
        {
            "model_name": "primary-gpt4",  # same name = load balance
            "litellm_params": {
                "model": "openai/gpt-4o-mini",
                "api_key": "sk-key2",  # different API key
            },
        },
        {
            "model_name": "fallback-claude",
            "litellm_params": {
                "model": "anthropic/claude-haiku-4-20250514",
                "api_key": "sk-ant-...",
            },
        },
    ],
    fallbacks=[
        {"primary-gpt4": ["fallback-claude"]},
    ],
    routing_strategy="least-busy",
    num_retries=3,
)

# Router automatically handles failover
response = await router.acompletion(
    model="primary-gpt4",
    messages=[{"role": "user", "content": "What is RAG?"}],
)
```

---

## Kong AI Gateway

### Architecture

Kong AI Gateway is an enterprise API gateway with AI-specific plugins:

```yaml
# kong.yml
services:
  - name: llm-service
    url: https://api.openai.com/v1
    routes:
      - name: llm-route
        paths:
          - /v1/chat/completions
        methods:
          - POST
    plugins:
      - name: ai-proxy
        config:
          route_type: llm/v1/chat
          model:
            provider: openai
            name: gpt-4o-mini
          auth:
            header_name: Authorization
            header_value: "Bearer sk-..."

      - name: rate-limiting-advanced
        config:
          limit: [100]
          window_size: [60]
          identifier: consumer

      - name: ai-request-transformer
        config:
          prompt: "You are a helpful RAG assistant. {user_prompt}"

      - name: ai-response-transformer
        config:
          max_tokens: 500
```

---

## Comparison Matrix

| Feature | Portkey | OpenRouter | LiteLLM | Kong AI |
|---------|---------|------------|---------|---------|
| Deployment | Managed + OSS | Managed | Self-hosted + managed | Self-hosted + managed |
| Providers supported | 15+ | 100+ | 100+ | 10+ |
| Fallback chains | Yes | No (single model) | Yes | Yes (plugin) |
| Load balancing | Yes | No | Yes | Yes |
| Semantic caching | Yes | No | No | No |
| Cost tracking | Yes (dashboard) | Yes (dashboard) | Yes (basic) | Yes (plugin) |
| Rate limiting | Yes | Account-level | Yes | Yes (enterprise) |
| Self-hosted option | AI Gateway (OSS) | No | Yes (proxy) | Yes |
| OpenAI API compatible | Yes | Yes | Yes | Yes |
| Guardrails | Yes | No | No | Yes |
| Price | Free tier + paid | Pay per token (markup) | Free (OSS) | Enterprise |

---

## When to Use Each

| Scenario | Best Choice |
|----------|------------|
| Quick prototype, try many models | OpenRouter |
| Production with fallbacks + caching | Portkey |
| Self-hosted, full control | LiteLLM |
| Enterprise with existing Kong | Kong AI Gateway |
| Need semantic caching | Portkey |
| Budget-constrained, many models | OpenRouter (pay as you go) |
| Multi-team governance | LiteLLM Proxy or Portkey |

---

## Common Pitfalls

1. **Gateway as single point of failure**: If the gateway goes down, all LLM calls fail. Run the gateway with redundancy (multiple instances, health checks, auto-restart).

2. **Caching sensitive data**: Semantic caching stores prompts and responses. Do not cache requests containing PII, credentials, or sensitive business data.

3. **OpenRouter markup**: OpenRouter adds a markup on top of provider prices. For high-volume production use, direct provider API keys are cheaper.

4. **LiteLLM version compatibility**: LiteLLM updates frequently and sometimes breaks backward compatibility. Pin your version and test before upgrading.

5. **Not testing fallback chains**: Configure fallbacks but never test them. Inject failures in staging to verify that fallback chains work correctly.

6. **Ignoring latency overhead**: Every gateway adds latency (5-50ms for managed, 1-5ms for self-hosted). For latency-sensitive applications, measure the gateway overhead.

---

## References

- Portkey: https://portkey.ai/docs/
- OpenRouter: https://openrouter.ai/docs
- LiteLLM: https://docs.litellm.ai/
- Kong AI Gateway: https://docs.konghq.com/gateway/latest/ai-gateway/
