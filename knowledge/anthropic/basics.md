# Anthropic Python SDK - Basics

> Official Documentation: https://platform.claude.com/docs/en/api/getting-started
> SDK Repository: https://github.com/anthropics/anthropic-sdk-python

## Overview

The Anthropic Python SDK provides programmatic access to Claude models via the Messages API. It handles authentication, request formatting, error handling, retry logic, and streaming. Requires Python 3.9+ and is installable via pip.

---

## Table of Contents

1. [Installation](#installation)
2. [Authentication](#authentication)
3. [Creating the Client](#creating-the-client)
4. [Messages API - Basic Usage](#messages-api---basic-usage)
5. [Request Parameters](#request-parameters)
6. [Response Object](#response-object)
7. [Available Models](#available-models)
8. [Error Handling](#error-handling)
9. [HTTP Request Headers](#http-request-headers)

---

## Installation

```bash
pip install anthropic
```

Latest version as of early 2026: `v0.84.0`

For development environments:
```bash
pip install anthropic[dev]
```

---

## Authentication

The SDK reads the API key from the `ANTHROPIC_API_KEY` environment variable by default.

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

Alternatively, pass it explicitly when constructing the client:

```python
from anthropic import Anthropic

client = Anthropic(api_key="sk-ant-...")
```

API keys are created in the [Anthropic Console](https://platform.claude.com/settings/keys).

---

## Creating the Client

### Synchronous Client

```python
from anthropic import Anthropic

# Reads ANTHROPIC_API_KEY from environment automatically
client = Anthropic()

# Explicit API key
client = Anthropic(api_key="sk-ant-...")

# Custom timeout (default: 600s for sync)
client = Anthropic(timeout=30.0)

# Custom base URL (for proxies)
client = Anthropic(base_url="https://my-proxy.example.com")
```

### Async Client

```python
from anthropic import AsyncAnthropic

client = AsyncAnthropic()
```

The async client is covered in detail in `advanced.md`.

---

## Messages API - Basic Usage

The primary endpoint is `POST /v1/messages`. In Python:

```python
import anthropic

client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Hello, Claude"}
    ],
)

print(message.content[0].text)
```

### With System Prompt

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system="You are a helpful Python expert.",
    messages=[
        {"role": "user", "content": "What is a list comprehension?"}
    ],
)
print(message.content[0].text)
```

### Multi-turn Conversation

```python
message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "My name is Mario."},
        {"role": "assistant", "content": "Nice to meet you, Mario!"},
        {"role": "user", "content": "What is my name?"},
    ],
)
print(message.content[0].text)  # "Your name is Mario."
```

### Raw HTTP (curl) Equivalent

```bash
curl https://api.anthropic.com/v1/messages \
  --header "x-api-key: $ANTHROPIC_API_KEY" \
  --header "anthropic-version: 2023-06-01" \
  --header "content-type: application/json" \
  --data '{
    "model": "claude-opus-4-6",
    "max_tokens": 1024,
    "messages": [
      {"role": "user", "content": "Hello, Claude"}
    ]
  }'
```

---

## Request Parameters

### Required

| Parameter | Type | Description |
|-----------|------|-------------|
| `model` | string | Model ID (e.g., `"claude-opus-4-6"`) |
| `max_tokens` | integer | Maximum tokens to generate before stopping |
| `messages` | array | List of message objects with `role` and `content` |

### Optional

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `system` | string or array | — | System prompt providing context/instructions |
| `temperature` | float | 1.0 | Randomness: 0.0 (deterministic) to 1.0 (creative) |
| `top_p` | float | — | Nucleus sampling; mutually exclusive with `top_k` |
| `top_k` | integer | — | Sample from top K token options |
| `stop_sequences` | list[str] | — | Custom strings that stop generation |
| `stream` | bool | false | Enable server-sent event streaming |
| `tools` | array | — | Tool definitions for function calling |
| `tool_choice` | object | — | Control which tool Claude uses |
| `metadata` | object | — | User-level metadata (e.g., `user_id`) |

### Message Format

```python
# Simple string content
{"role": "user", "content": "Hello"}

# Array of content blocks (for multi-modal)
{"role": "user", "content": [
    {"type": "text", "text": "What's in this image?"},
    {"type": "image", "source": {"type": "url", "url": "https://..."}}
]}
```

---

## Response Object

```python
message = client.messages.create(...)

# Top-level fields
message.id           # "msg_01XFDUDYJgAACzvnptvVoYEL"
message.type         # "message"
message.role         # "assistant"
message.model        # "claude-opus-4-6"
message.stop_reason  # "end_turn" | "stop_sequence" | "max_tokens" | "tool_use"
message.stop_sequence  # The matched stop sequence (if stop_reason == "stop_sequence")

# Content blocks
message.content      # list of content blocks
message.content[0].type  # "text" | "tool_use" | "thinking"
message.content[0].text  # text content (when type == "text")

# Token usage
message.usage.input_tokens   # tokens consumed from input
message.usage.output_tokens  # tokens generated
message.usage.cache_creation_input_tokens  # tokens written to prompt cache
message.usage.cache_read_input_tokens      # tokens read from prompt cache
```

### Stop Reasons

| Value | Meaning |
|-------|---------|
| `"end_turn"` | Model finished naturally |
| `"stop_sequence"` | Hit a custom stop sequence |
| `"max_tokens"` | Reached `max_tokens` limit |
| `"tool_use"` | Model is invoking a tool |

### JSON Response Example

```json
{
  "id": "msg_01XFDUDYJgAACzvnptvVoYEL",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "Hello! How can I assist you today?"
    }
  ],
  "model": "claude-opus-4-6",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 12,
    "output_tokens": 8
  }
}
```

---

## Available Models

| Model ID | Description |
|----------|-------------|
| `claude-opus-4-6` | Most intelligent; best for complex reasoning, agents, coding |
| `claude-sonnet-4-6` | Best speed/intelligence balance; high-performance |
| `claude-haiku-4-5` | Fastest; best for high-throughput, cost-sensitive tasks |
| `claude-opus-4-5` | Premium intelligence |
| `claude-sonnet-4-5` | High-performance agents and coding |
| `claude-opus-4-1` | Specialized complex tasks |
| `claude-sonnet-4-0` | Extended thinking capable |
| `claude-3-haiku-20240307` | Fast and cost-effective (older generation) |

Use `claude-opus-4-6` for complex tasks requiring deep reasoning, `claude-sonnet-4-6` for everyday production use, and `claude-haiku-4-5` for fast/cheap tasks.

List all available models programmatically:

```python
models = client.models.list()
for model in models:
    print(model.id)
```

---

## Error Handling

The SDK raises typed exceptions:

```python
import anthropic

client = anthropic.Anthropic()

try:
    message = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": "Hello"}],
    )
except anthropic.APIConnectionError as e:
    print("Server could not be reached:", e.__cause__)
except anthropic.RateLimitError as e:
    print("Rate limit exceeded:", e.status_code)
except anthropic.AuthenticationError as e:
    print("Invalid API key:", e.status_code)
except anthropic.APIStatusError as e:
    print("API error:", e.status_code, e.response)
```

### Common Error Types

| Exception | HTTP Status | Cause |
|-----------|-------------|-------|
| `AuthenticationError` | 401 | Invalid or missing API key |
| `PermissionDeniedError` | 403 | Insufficient permissions |
| `NotFoundError` | 404 | Resource not found |
| `UnprocessableEntityError` | 422 | Invalid request parameters |
| `RateLimitError` | 429 | Rate or spend limit exceeded |
| `InternalServerError` | 500 | Anthropic server error |
| `APIConnectionError` | — | Network/connection failure |

---

## HTTP Request Headers

Every request must include these headers (handled automatically by the SDK):

| Header | Value |
|--------|-------|
| `x-api-key` | Your API key |
| `anthropic-version` | `"2023-06-01"` |
| `content-type` | `"application/json"` |

### Available APIs

| API | Endpoint | Description |
|-----|----------|-------------|
| Messages | `POST /v1/messages` | Core conversational API |
| Message Batches | `POST /v1/messages/batches` | Async batch processing (50% cost reduction) |
| Token Counting | `POST /v1/messages/count_tokens` | Count tokens before sending |
| Models | `GET /v1/models` | List available models |
| Files (beta) | `POST /v1/files` | Upload files for reuse across calls |

### Request Size Limits

| Endpoint | Max Size |
|----------|----------|
| Messages, Token Counting | 32 MB |
| Batch API | 256 MB |
| Files API | 500 MB |
