# Anthropic Python SDK - Advanced Features

> Official Documentation: https://platform.claude.com/docs/en/api/getting-started
> SDK Repository: https://github.com/anthropics/anthropic-sdk-python

## Overview

Advanced features including the AsyncAnthropic client, Message Batches API, token counting, prompt caching, rate limits, and retry/error handling.

---

## Table of Contents

1. [Async Client (AsyncAnthropic)](#async-client-asyncanthropic)
2. [Message Batches API](#message-batches-api)
3. [Token Counting](#token-counting)
4. [Prompt Caching](#prompt-caching)
5. [Rate Limits](#rate-limits)
6. [Retry Logic](#retry-logic)
7. [Timeout Configuration](#timeout-configuration)
8. [Extended Thinking](#extended-thinking)

---

## Async Client (AsyncAnthropic)

Use `AsyncAnthropic` for async/await codebases (FastAPI, async scripts, etc.):

```python
import asyncio
import anthropic

async def main():
    client = anthropic.AsyncAnthropic()

    message = await client.messages.create(
        model="claude-opus-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": "Hello, Claude"}],
    )
    print(message.content[0].text)

asyncio.run(main())
```

### Async Streaming

```python
import asyncio
import anthropic

async def stream():
    client = anthropic.AsyncAnthropic()

    async with client.messages.stream(
        model="claude-opus-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": "Write a short poem."}],
    ) as stream:
        async for text in stream.text_stream:
            print(text, end="", flush=True)

asyncio.run(stream())
```

### Async with FastAPI

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import anthropic

app = FastAPI()
client = anthropic.AsyncAnthropic()

@app.post("/chat")
async def chat(prompt: str):
    message = await client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}],
    )
    return {"response": message.content[0].text}

@app.post("/chat/stream")
async def chat_stream(prompt: str):
    async def generate():
        async with client.messages.stream(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            messages=[{"role": "user", "content": prompt}],
        ) as stream:
            async for text in stream.text_stream:
                yield text

    return StreamingResponse(generate(), media_type="text/plain")
```

### Concurrent Requests

```python
import asyncio
import anthropic

async def process_all(prompts: list[str]) -> list[str]:
    client = anthropic.AsyncAnthropic()

    async def process_one(prompt: str) -> str:
        msg = await client.messages.create(
            model="claude-haiku-4-5",
            max_tokens=512,
            messages=[{"role": "user", "content": prompt}],
        )
        return msg.content[0].text

    results = await asyncio.gather(*[process_one(p) for p in prompts])
    return results

prompts = ["Summarize Python.", "Summarize Rust.", "Summarize Go."]
results = asyncio.run(process_all(prompts))
for r in results:
    print(r)
```

---

## Message Batches API

Process large volumes of requests asynchronously at 50% cost reduction. Most batches finish within 1 hour.

### Creating a Batch

```python
import anthropic
from anthropic.types.message_create_params import MessageCreateParamsNonStreaming
from anthropic.types.messages.batch_create_params import Request

client = anthropic.Anthropic()

batch = client.messages.batches.create(
    requests=[
        Request(
            custom_id="req-001",
            params=MessageCreateParamsNonStreaming(
                model="claude-opus-4-6",
                max_tokens=1024,
                messages=[{"role": "user", "content": "Summarize the French Revolution."}],
            ),
        ),
        Request(
            custom_id="req-002",
            params=MessageCreateParamsNonStreaming(
                model="claude-opus-4-6",
                max_tokens=1024,
                messages=[{"role": "user", "content": "Summarize the Industrial Revolution."}],
            ),
        ),
    ]
)

print(batch.id)           # "msgbatch_01..."
print(batch.processing_status)  # "in_progress"
```

### Polling for Completion

```python
import time
import anthropic

client = anthropic.Anthropic()
batch_id = "msgbatch_01..."

# Poll until complete
while True:
    batch = client.messages.batches.retrieve(batch_id)
    print(f"Status: {batch.processing_status}")

    if batch.processing_status == "ended":
        break

    time.sleep(30)  # Check every 30 seconds

print(f"Request counts: {batch.request_counts}")
# request_counts.processing, .succeeded, .errored, .canceled, .expired
```

### Retrieving Results

```python
# Iterate over results as they stream
for result in client.messages.batches.results(batch_id):
    print(f"ID: {result.custom_id}")
    print(f"Type: {result.result.type}")  # "succeeded" | "errored" | "canceled" | "expired"

    if result.result.type == "succeeded":
        msg = result.result.message
        print(f"Response: {msg.content[0].text}")
    elif result.result.type == "errored":
        print(f"Error: {result.result.error}")
```

### Complete Batch Processing Workflow

```python
import time
import anthropic
from anthropic.types.message_create_params import MessageCreateParamsNonStreaming
from anthropic.types.messages.batch_create_params import Request

client = anthropic.Anthropic()

# 1. Prepare requests
texts_to_summarize = [
    ("doc-001", "Long article about climate change..."),
    ("doc-002", "Long article about AI safety..."),
    ("doc-003", "Long article about quantum computing..."),
]

requests = [
    Request(
        custom_id=doc_id,
        params=MessageCreateParamsNonStreaming(
            model="claude-haiku-4-5",
            max_tokens=512,
            messages=[{"role": "user", "content": f"Summarize: {text}"}],
        ),
    )
    for doc_id, text in texts_to_summarize
]

# 2. Create batch
batch = client.messages.batches.create(requests=requests)
print(f"Batch created: {batch.id}")

# 3. Poll for completion
while (batch := client.messages.batches.retrieve(batch.id)).processing_status != "ended":
    print(f"Status: {batch.processing_status}, sleeping 60s...")
    time.sleep(60)

# 4. Collect results
summaries = {}
for result in client.messages.batches.results(batch.id):
    if result.result.type == "succeeded":
        summaries[result.custom_id] = result.result.message.content[0].text

print(summaries)
```

### Batch Limits

| Limit | Value |
|-------|-------|
| Max requests per batch | 100,000 |
| Max batch size | 256 MB |
| Processing timeout | 24 hours |
| Results available for | 29 days |

### Batch Pricing (50% discount)

| Model | Input | Output |
|-------|-------|--------|
| Claude Opus 4.6 | $2.50/MTok | $12.50/MTok |
| Claude Sonnet 4.6 | $1.50/MTok | $7.50/MTok |
| Claude Haiku 4.5 | $0.50/MTok | $2.50/MTok |

---

## Token Counting

Count tokens before sending a request to manage costs and avoid hitting limits:

```python
import anthropic

client = anthropic.Anthropic()

# Count tokens for a simple message
response = client.messages.count_tokens(
    model="claude-opus-4-6",
    messages=[{"role": "user", "content": "Hello, Claude. How are you today?"}],
)
print(f"Input tokens: {response.input_tokens}")

# Count tokens including system prompt
response = client.messages.count_tokens(
    model="claude-opus-4-6",
    system="You are a helpful assistant.",
    messages=[
        {"role": "user", "content": "What is the capital of France?"},
        {"role": "assistant", "content": "The capital of France is Paris."},
        {"role": "user", "content": "And what is its population?"},
    ],
)
print(f"Total input tokens: {response.input_tokens}")

# Count tokens including tool definitions
tools = [
    {
        "name": "get_weather",
        "description": "Get current weather",
        "input_schema": {
            "type": "object",
            "properties": {"location": {"type": "string"}},
            "required": ["location"],
        },
    }
]

response = client.messages.count_tokens(
    model="claude-opus-4-6",
    tools=tools,
    messages=[{"role": "user", "content": "What's the weather in Paris?"}],
)
print(f"Tokens with tools: {response.input_tokens}")
```

Use token counting to:
- Pre-validate requests before sending
- Estimate costs
- Implement context window management (trim old messages when approaching limits)

---

## Prompt Caching

Cache large prompt prefixes to reduce cost and latency on repeated calls.

### How It Works

Add `cache_control` to content blocks you want cached. Subsequent requests with the same prefix will read from cache instead of reprocessing.

- **Cache TTL**: 5 minutes (default) or 1 hour (`"ttl": "1h"`)
- **Pricing**: Cache writes cost 1.25x base; cache reads cost 0.1x base
- **Minimum cacheable tokens**: 2,048 (Sonnet) or 4,096 (Opus/Haiku)

### Caching a Large System Prompt

```python
import anthropic

client = anthropic.Anthropic()

# First call: writes to cache
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "You are a legal document analyzer with expertise in contract law.",
        },
        {
            "type": "text",
            "text": "[Full text of a 50-page contract - 10,000+ tokens...]",
            "cache_control": {"type": "ephemeral"},
        },
    ],
    messages=[{"role": "user", "content": "What are the termination clauses?"}],
)

print(f"Cache created: {response.usage.cache_creation_input_tokens}")
print(f"Cache read: {response.usage.cache_read_input_tokens}")

# Second call: reads from cache (0.1x cost)
response2 = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "You are a legal document analyzer with expertise in contract law.",
        },
        {
            "type": "text",
            "text": "[Same 50-page contract text...]",
            "cache_control": {"type": "ephemeral"},
        },
    ],
    messages=[{"role": "user", "content": "What are the payment terms?"}],
)

print(f"Cache created: {response2.usage.cache_creation_input_tokens}")  # 0
print(f"Cache read: {response2.usage.cache_read_input_tokens}")          # ~10,000
```

### 1-Hour Cache TTL

For prompts used less frequently than every 5 minutes:

```python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "[Large knowledge base...]",
            "cache_control": {"type": "ephemeral", "ttl": "1h"},
        }
    ],
    messages=[{"role": "user", "content": "Answer based on the knowledge base."}],
)
```

### Caching Tool Definitions

```python
tools = [
    {"name": "search", "description": "...", "input_schema": {...}},
    {"name": "retrieve", "description": "...", "input_schema": {...}},
    {
        "name": "analyze",
        "description": "...",
        "input_schema": {...},
        "cache_control": {"type": "ephemeral"},  # Cache up to this point
    },
]
```

### Automatic Caching for Multi-turn Conversations

```python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    cache_control={"type": "ephemeral"},  # Top-level: auto cache management
    system="You are a helpful assistant.",
    messages=[
        {"role": "user", "content": "Hi, I'm working on a Python project."},
        {"role": "assistant", "content": "Great! What are you building?"},
        {"role": "user", "content": "A web scraper. How do I handle rate limits?"},
    ],
)
```

### Cache Hit Monitoring

```python
usage = response.usage
print(f"Input tokens (uncached): {usage.input_tokens}")
print(f"Cache writes: {usage.cache_creation_input_tokens}")
print(f"Cache reads: {usage.cache_read_input_tokens}")
total = usage.input_tokens + usage.cache_creation_input_tokens + usage.cache_read_input_tokens
print(f"Total input tokens: {total}")
```

### Pricing

| Model | Base Input | 5m Cache Write | 1h Cache Write | Cache Read |
|-------|-----------|----------------|----------------|------------|
| Opus 4.6 | $5/MTok | $6.25/MTok | $10/MTok | $0.50/MTok |
| Sonnet 4.6 | $3/MTok | $3.75/MTok | $6/MTok | $0.30/MTok |
| Haiku 4.5 | $1/MTok | $1.25/MTok | $2/MTok | $0.10/MTok |

---

## Rate Limits

The API uses a token bucket algorithm. Limits are per-model and per-organization.

### Rate Limit Tiers (Messages API)

| Tier | Model Class | RPM | ITPM | OTPM |
|------|-------------|-----|------|------|
| 1 | Opus 4.x | 50 | 30,000 | 8,000 |
| 1 | Sonnet 4.x | 50 | 30,000 | 8,000 |
| 1 | Haiku 4.5 | 50 | 50,000 | 10,000 |
| 2 | Opus 4.x | 1,000 | 450,000 | 90,000 |
| 2 | Sonnet 4.x | 1,000 | 450,000 | 90,000 |
| 3 | Opus 4.x | 2,000 | 800,000 | 160,000 |
| 4 | Opus 4.x | 4,000 | 2,000,000 | 400,000 |

Note: **Cache reads do NOT count toward ITPM** for current models, making prompt caching an effective way to increase throughput.

### Rate Limit Response Headers

```
anthropic-ratelimit-requests-limit: 50
anthropic-ratelimit-requests-remaining: 49
anthropic-ratelimit-requests-reset: 2025-01-01T00:01:00Z
anthropic-ratelimit-tokens-limit: 40000
anthropic-ratelimit-tokens-remaining: 38500
anthropic-ratelimit-tokens-reset: 2025-01-01T00:00:30Z
retry-after: 30
```

### Handling Rate Limit Errors

```python
import time
import anthropic

client = anthropic.Anthropic()

def call_with_backoff(prompt: str, max_retries: int = 5) -> str:
    for attempt in range(max_retries):
        try:
            response = client.messages.create(
                model="claude-opus-4-6",
                max_tokens=1024,
                messages=[{"role": "user", "content": prompt}],
            )
            return response.content[0].text
        except anthropic.RateLimitError as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt  # Exponential backoff: 1, 2, 4, 8, 16s
            print(f"Rate limited. Waiting {wait_time}s...")
            time.sleep(wait_time)

result = call_with_backoff("What is the meaning of life?")
```

---

## Retry Logic

The SDK has built-in automatic retry with exponential backoff for transient errors:

```python
import anthropic

# Configure max retries (default: 2)
client = anthropic.Anthropic(
    max_retries=4,  # Retry up to 4 times
)

# Disable retries
client_no_retry = anthropic.Anthropic(max_retries=0)

# Per-request override
response = client.messages.with_options(max_retries=0).create(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
)
```

The SDK retries on:
- Connection errors
- HTTP 408 (Request Timeout)
- HTTP 429 (Rate Limit) — uses `retry-after` header
- HTTP 500, 502, 503, 529 (Server Errors)

It does NOT retry on 400, 401, 403, 404, 422 (client errors).

---

## Timeout Configuration

```python
import anthropic
import httpx

# Global timeout (seconds)
client = anthropic.Anthropic(timeout=60.0)

# Granular timeout control
client = anthropic.Anthropic(
    timeout=httpx.Timeout(
        connect=5.0,    # Connection timeout
        read=300.0,     # Read timeout (important for long responses)
        write=10.0,     # Write timeout
        pool=5.0,       # Pool timeout
    )
)

# Per-request timeout override
response = client.messages.with_options(timeout=120.0).create(
    model="claude-opus-4-6",
    max_tokens=4096,
    messages=[{"role": "user", "content": "Write a 2000-word essay."}],
)
```

For very long responses, use streaming to avoid timeouts:

```python
# Stream for long responses - avoids read timeout
with client.messages.stream(
    model="claude-opus-4-6",
    max_tokens=128000,
    messages=[{"role": "user", "content": "Write a very long document..."}],
) as stream:
    message = stream.get_final_message()
```

---

## Extended Thinking

Enable Claude's chain-of-thought reasoning for complex problems:

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000,  # Min 1024, must be < max_tokens
    },
    messages=[
        {"role": "user", "content": "Solve: If a train travels at 60 mph for 2.5 hours, then at 80 mph for 1.5 hours, what is the total distance and average speed?"}
    ],
)

for block in response.content:
    if block.type == "thinking":
        print(f"[Thinking]: {block.thinking[:200]}...")
    elif block.type == "text":
        print(f"[Answer]: {block.text}")
```

Thinking options:
- `{"type": "enabled", "budget_tokens": N}` — Enable with N token budget
- `{"type": "disabled"}` — Disable explicitly
- `{"type": "adaptive"}` — Let the model decide

The thinking budget counts toward `max_tokens`. Thinking blocks are visible in the response but their content is not factored into prompt caching costs for subsequent turns.
