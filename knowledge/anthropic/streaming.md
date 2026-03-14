# Anthropic Python SDK - Streaming

> Official Documentation: https://platform.claude.com/docs/en/api/messages-streaming

## Overview

Streaming returns Claude's response incrementally via Server-Sent Events (SSE) instead of waiting for the full response. This reduces perceived latency and is required by the SDK for large responses to avoid HTTP timeouts.

---

## Table of Contents

1. [Basic Streaming (sync)](#basic-streaming-sync)
2. [Async Streaming](#async-streaming)
3. [Getting the Final Message](#getting-the-final-message)
4. [Stream Events Reference](#stream-events-reference)
5. [Raw SSE Streaming](#raw-sse-streaming)
6. [Streaming with Tool Use](#streaming-with-tool-use)
7. [Error Handling During Streaming](#error-handling-during-streaming)
8. [Event Handlers](#event-handlers)

---

## Basic Streaming (sync)

The simplest way to stream text:

```python
import anthropic

client = anthropic.Anthropic()

with client.messages.stream(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Write a poem about the ocean."}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)

print()  # newline at end
```

The `stream.text_stream` iterator yields text chunks as they arrive.

### With System Prompt

```python
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=2048,
    system="You are a creative writing assistant.",
    messages=[{"role": "user", "content": "Tell me a short story about a robot."}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

---

## Async Streaming

Use `AsyncAnthropic` with `async with` and `async for`:

```python
import asyncio
import anthropic

async def stream_response():
    client = anthropic.AsyncAnthropic()

    async with client.messages.stream(
        model="claude-opus-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": "Explain async programming in Python."}],
    ) as stream:
        async for text in stream.text_stream:
            print(text, end="", flush=True)

asyncio.run(stream_response())
```

### Async with Multiple Streams Concurrently

```python
import asyncio
import anthropic

async def stream_one(client, prompt: str, label: str):
    async with client.messages.stream(
        model="claude-haiku-4-5",
        max_tokens=512,
        messages=[{"role": "user", "content": prompt}],
    ) as stream:
        result = ""
        async for text in stream.text_stream:
            result += text
        print(f"\n[{label}]: {result}")

async def main():
    client = anthropic.AsyncAnthropic()
    await asyncio.gather(
        stream_one(client, "What is Python?", "Q1"),
        stream_one(client, "What is Rust?", "Q2"),
        stream_one(client, "What is Go?", "Q3"),
    )

asyncio.run(main())
```

---

## Getting the Final Message

When you don't need to process tokens as they arrive but still want streaming (e.g., to avoid HTTP timeout on long responses), use `get_final_message()`:

```python
import anthropic

client = anthropic.Anthropic()

with client.messages.stream(
    model="claude-opus-4-6",
    max_tokens=128000,
    messages=[{"role": "user", "content": "Write a detailed technical specification."}],
) as stream:
    # Streams under the hood, returns full Message object
    message = stream.get_final_message()

print(message.content[0].text)
print(f"Input tokens: {message.usage.input_tokens}")
print(f"Output tokens: {message.usage.output_tokens}")
```

### Get Final Text String

```python
with client.messages.stream(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Summarize quantum computing."}],
) as stream:
    full_text = stream.get_final_text()

print(full_text)
```

---

## Stream Events Reference

### Event Sequence

A complete streaming session produces these events in order:

1. `message_start` — Start of message (includes model, usage scaffold)
2. `content_block_start` — Start of a content block (type: text, tool_use, thinking)
3. `content_block_delta` — Incremental chunk within a content block
4. `content_block_stop` — End of a content block
5. `message_delta` — Final message metadata (stop_reason, stop_sequence, output tokens)
6. `message_stop` — End of the entire message

### SSE Wire Format

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_01...","type":"message","role":"assistant","content":[],"model":"claude-opus-4-6","stop_reason":null,"stop_sequence":null,"usage":{"input_tokens":25,"output_tokens":1}}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"! How"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn","stop_sequence":null},"usage":{"output_tokens":15}}

event: message_stop
data: {"type":"message_stop"}
```

### Delta Types

| Delta Type | Content | When |
|------------|---------|------|
| `text_delta` | `{"type": "text_delta", "text": "..."}` | Text content blocks |
| `input_json_delta` | `{"type": "input_json_delta", "partial_json": "..."}` | Tool input (JSON fragments) |
| `thinking_delta` | `{"type": "thinking_delta", "thinking": "..."}` | Extended thinking blocks |
| `signature_delta` | `{"type": "signature_delta", "signature": "..."}` | Thinking block signature |

---

## Raw SSE Streaming

For lower-level control, use `stream=True` directly:

```python
import anthropic

client = anthropic.Anthropic()

with client.messages.stream(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
) as stream:
    for event in stream:
        # Access raw events
        print(event.type)
        if event.type == "content_block_delta":
            print(event.delta)
```

### Using create() with stream=True (lower level)

```python
with client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    stream=True,
    messages=[{"role": "user", "content": "Hello"}],
) as stream:
    for event in stream:
        if event.type == "content_block_delta" and event.delta.type == "text_delta":
            print(event.delta.text, end="", flush=True)
```

---

## Streaming with Tool Use

When streaming with tools, tool input arrives as `input_json_delta` events:

```python
import anthropic
import json

client = anthropic.Anthropic()

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

with client.messages.stream(
    model="claude-opus-4-6",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "What's the weather in Paris?"}],
) as stream:
    for event in stream:
        if event.type == "content_block_start":
            if event.content_block.type == "tool_use":
                print(f"\nTool call: {event.content_block.name}")

        elif event.type == "content_block_delta":
            if event.delta.type == "input_json_delta":
                print(event.delta.partial_json, end="", flush=True)

    # Get the final message to check stop_reason and tool inputs
    final = stream.get_final_message()
    if final.stop_reason == "tool_use":
        tool_block = next(b for b in final.content if b.type == "tool_use")
        print(f"\nFull input: {tool_block.input}")
```

---

## Error Handling During Streaming

Errors can occur mid-stream. The SDK surfaces them as exceptions:

```python
import anthropic

client = anthropic.Anthropic()

try:
    with client.messages.stream(
        model="claude-opus-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": "Tell me a long story."}],
    ) as stream:
        for text in stream.text_stream:
            print(text, end="", flush=True)
except anthropic.APIConnectionError:
    print("\nConnection lost during streaming")
except anthropic.RateLimitError:
    print("\nRate limit hit during streaming")
except anthropic.APIStatusError as e:
    print(f"\nAPI error {e.status_code}: {e.message}")
```

The SDK raises errors as soon as they occur during iteration, so partial output is preserved up to that point.

---

## Event Handlers

The streaming context manager exposes event-handler callbacks:

```python
import anthropic

client = anthropic.Anthropic()

# Sync event handlers via on()
stream = client.messages.stream(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
)

# You can attach handlers before entering context
# (for TypeScript-style event-driven approach in Python):
with stream as s:
    # Access accumulated text at any point
    for text_chunk in s.text_stream:
        pass  # process incrementally

    # After stream ends, get the accumulated message
    final = s.get_final_message()
    print(f"Stop reason: {final.stop_reason}")
    print(f"Total tokens: {final.usage.input_tokens + final.usage.output_tokens}")
```

### Async Event-Driven Pattern

```python
import asyncio
import anthropic

async def process_stream():
    client = anthropic.AsyncAnthropic()

    full_text = ""
    input_tokens = 0
    output_tokens = 0

    async with client.messages.stream(
        model="claude-opus-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": "Explain generators in Python."}],
    ) as stream:
        async for event in stream:
            if event.type == "message_start":
                input_tokens = event.message.usage.input_tokens

            elif event.type == "content_block_delta":
                if event.delta.type == "text_delta":
                    full_text += event.delta.text
                    print(event.delta.text, end="", flush=True)

            elif event.type == "message_delta":
                output_tokens = event.usage.output_tokens

    print(f"\n\nTokens: {input_tokens} in / {output_tokens} out")
    return full_text

asyncio.run(process_stream())
```
