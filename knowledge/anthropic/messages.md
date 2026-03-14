# Anthropic Python SDK - Messages API

> Official Documentation: https://platform.claude.com/docs/en/api/messages

## Overview

The Messages API (`POST /v1/messages`) is the core of Claude. It supports structured multi-turn conversations, system prompts, multi-modal inputs (text, image, document), and precise control over generation parameters.

---

## Table of Contents

1. [System Prompt](#system-prompt)
2. [User and Assistant Turns](#user-and-assistant-turns)
3. [Multi-turn Conversations](#multi-turn-conversations)
4. [Content Blocks](#content-blocks)
5. [Vision - Image Input](#vision---image-input)
6. [Document Input](#document-input)
7. [Stop Sequences](#stop-sequences)
8. [Temperature and Sampling](#temperature-and-sampling)
9. [Constrained Output Prefill](#constrained-output-prefill)

---

## System Prompt

The system prompt sets context, persona, and instructions that apply to the entire conversation. It is not part of the `messages` array; it is a separate top-level parameter.

### String System Prompt

```python
import anthropic

client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    system="You are an expert Python developer who writes clean, well-documented code. Always include type hints.",
    messages=[
        {"role": "user", "content": "Write a function to flatten a nested list."}
    ],
)
print(message.content[0].text)
```

### Array System Prompt (with cache control)

The system can be an array of text blocks, useful for prompt caching:

```python
message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "You are a legal document analyzer.",
        },
        {
            "type": "text",
            "text": "[Full text of a 50-page legal agreement...]",
            "cache_control": {"type": "ephemeral"},
        },
    ],
    messages=[
        {"role": "user", "content": "Summarize the key obligations in section 3."}
    ],
)
```

---

## User and Assistant Turns

Messages alternate between `"user"` and `"assistant"` roles. Each message has a `role` and `content`.

```python
messages = [
    {"role": "user", "content": "What is the capital of France?"},
    {"role": "assistant", "content": "The capital of France is Paris."},
    {"role": "user", "content": "What is its population?"},
]
```

Rules:
- The first message must be a `"user"` turn.
- Roles must alternate: user, assistant, user, assistant...
- Consecutive same-role messages are automatically merged.
- Maximum 100,000 messages per request.

---

## Multi-turn Conversations

Implement a conversation loop by accumulating messages:

```python
import anthropic

client = anthropic.Anthropic()

conversation_history = []

def chat(user_message: str) -> str:
    conversation_history.append({"role": "user", "content": user_message})

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        system="You are a helpful assistant.",
        messages=conversation_history,
    )

    assistant_message = response.content[0].text
    conversation_history.append({"role": "assistant", "content": assistant_message})
    return assistant_message

print(chat("My name is Mario."))
print(chat("What is my name?"))   # "Your name is Mario."
print(chat("What did I tell you first?"))
```

---

## Content Blocks

Message content can be a string shorthand or an array of typed blocks.

### Text Block

```python
{"type": "text", "text": "Hello, Claude"}
```

### Image Block

```python
{
    "type": "image",
    "source": {
        "type": "base64",
        "media_type": "image/jpeg",  # image/jpeg, image/png, image/gif, image/webp
        "data": "<base64-encoded-bytes>",
    },
}

# Or via URL:
{
    "type": "image",
    "source": {
        "type": "url",
        "url": "https://example.com/photo.jpg",
    },
}
```

### Document Block

```python
{
    "type": "document",
    "source": {
        "type": "base64",
        "media_type": "application/pdf",
        "data": "<base64-encoded-pdf>",
    },
    "title": "Annual Report 2024",  # optional
    "context": "This is a financial document.",  # optional
    "citations": {"enabled": True},  # optional: enable citations from document
}

# Plain text document:
{
    "type": "document",
    "source": {
        "type": "text",
        "media_type": "text/plain",
        "data": "Full document text here...",
    },
}

# URL PDF:
{
    "type": "document",
    "source": {
        "type": "url",
        "url": "https://example.com/report.pdf",
    },
}
```

### Tool Use Block (in assistant messages)

```python
{
    "type": "tool_use",
    "id": "toolu_01A09q90qw90lq917835lq9",
    "name": "get_weather",
    "input": {"location": "San Francisco, CA"},
}
```

### Tool Result Block (in user messages, after tool use)

```python
{
    "type": "tool_result",
    "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
    "content": "The weather in San Francisco is 65°F and sunny.",
    "is_error": False,  # set True if the tool execution failed
}
```

---

## Vision - Image Input

Claude can analyze images passed as content blocks in user messages.

### From URL

```python
import anthropic

client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": "What objects do you see in this image?",
                },
                {
                    "type": "image",
                    "source": {
                        "type": "url",
                        "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/4/47/PNG_transparency_demonstration_1.png/280px-PNG_transparency_demonstration_1.png",
                    },
                },
            ],
        }
    ],
)
print(message.content[0].text)
```

### From Base64

```python
import base64
import anthropic

client = anthropic.Anthropic()

with open("photo.jpg", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode("utf-8")

message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "Describe this image in detail."},
                {
                    "type": "image",
                    "source": {
                        "type": "base64",
                        "media_type": "image/jpeg",
                        "data": image_data,
                    },
                },
            ],
        }
    ],
)
print(message.content[0].text)
```

### Multiple Images

```python
message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "Compare these two charts:"},
                {"type": "image", "source": {"type": "url", "url": "https://example.com/chart1.png"}},
                {"type": "text", "text": "vs"},
                {"type": "image", "source": {"type": "url", "url": "https://example.com/chart2.png"}},
            ],
        }
    ],
)
```

Supported image types: `image/jpeg`, `image/png`, `image/gif`, `image/webp`.

---

## Document Input

For PDF and plain-text documents:

```python
import base64
import anthropic

client = anthropic.Anthropic()

with open("report.pdf", "rb") as f:
    pdf_data = base64.standard_b64encode(f.read()).decode("utf-8")

message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=2048,
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "document",
                    "source": {
                        "type": "base64",
                        "media_type": "application/pdf",
                        "data": pdf_data,
                    },
                    "title": "Q4 Report",
                    "citations": {"enabled": True},
                },
                {"type": "text", "text": "What are the key financial highlights?"},
            ],
        }
    ],
)
print(message.content[0].text)
```

---

## Stop Sequences

Custom strings that halt generation. When hit, `stop_reason` becomes `"stop_sequence"` and `stop_sequence` contains the matched string.

```python
import anthropic

client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    stop_sequences=["END", "---", "\n\nHuman:"],
    messages=[
        {"role": "user", "content": "Write a short poem about Python. End with END."}
    ],
)

print(message.stop_reason)     # "stop_sequence"
print(message.stop_sequence)   # "END"
print(message.content[0].text)
```

Use cases:
- Delimiters for structured output parsing
- Stopping multi-section generation at section boundaries
- Simulating RLHF prompt formats

---

## Temperature and Sampling

### Temperature

Controls randomness of generation. Range: `0.0` to `1.0`.

```python
# Analytical / deterministic (e.g., code, math, factual Q&A)
message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    temperature=0.0,
    messages=[{"role": "user", "content": "What is 2 + 2?"}],
)

# Creative / generative (e.g., brainstorming, fiction)
message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    temperature=1.0,
    messages=[{"role": "user", "content": "Write a creative short story about a robot."}],
)
```

Note: Even at `temperature=0.0`, results are not fully deterministic due to hardware differences.

### top_p (Nucleus Sampling)

Only sample from tokens whose cumulative probability reaches `top_p`. Mutually exclusive with `top_k`.

```python
message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    top_p=0.9,  # Only consider tokens making up 90% of probability mass
    messages=[{"role": "user", "content": "Generate a tagline for a coffee shop."}],
)
```

### top_k

Sample only from the top K most probable tokens:

```python
message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    top_k=50,   # Consider only the 50 most likely next tokens
    messages=[{"role": "user", "content": "Describe autumn in one sentence."}],
)
```

Recommended: use `temperature` alone for most tasks. Use `top_p` or `top_k` only for advanced fine-grained control.

---

## Constrained Output Prefill

Prefill the assistant's response to guide output format:

```python
# Force a specific answer format
message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=16,
    messages=[
        {
            "role": "user",
            "content": "What's the Greek name for Sun? (A) Sol (B) Helios (C) Sun"
        },
        {
            "role": "assistant",
            "content": "The best answer is ("  # Claude will continue from here
        },
    ],
)
print(message.content[0].text)  # "B)"

# Force JSON output
message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=256,
    messages=[
        {"role": "user", "content": "Give me a user object with name and age fields."},
        {"role": "assistant", "content": "{"},
    ],
)
print("{" + message.content[0].text)
```

Note: The prefill text is not included in the response content; Claude continues from where you left off.
