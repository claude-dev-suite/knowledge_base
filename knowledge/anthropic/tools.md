# Anthropic Python SDK - Tool Use (Function Calling)

> Official Documentation: https://platform.claude.com/docs/en/docs/build-with-claude/tool-use/overview

## Overview

Tool use (function calling) lets Claude interact with external systems. You define a set of tools with names, descriptions, and JSON schemas. Claude decides when and how to call them. The model generates a `tool_use` block; your code executes the function and returns a `tool_result`.

---

## Table of Contents

1. [Tool Definition Format](#tool-definition-format)
2. [Basic Tool Use Flow](#basic-tool-use-flow)
3. [Complete Multi-step Example](#complete-multi-step-example)
4. [tool_result Block](#tool_result-block)
5. [Parallel Tool Use](#parallel-tool-use)
6. [Tool Choice](#tool-choice)
7. [Strict Tool Use (Schema Validation)](#strict-tool-use-schema-validation)
8. [Server-side Tools](#server-side-tools)
9. [Computer Use](#computer-use)
10. [Best Practices](#best-practices)

---

## Tool Definition Format

Each tool requires three fields: `name`, `description`, and `input_schema`.

```python
tool = {
    "name": "get_weather",                          # unique snake_case name
    "description": "Get the current weather in a given location. "
                   "Returns temperature (°F), conditions, and humidity.",
    "input_schema": {
        "type": "object",
        "properties": {
            "location": {
                "type": "string",
                "description": "The city and state, e.g. 'San Francisco, CA'",
            },
            "unit": {
                "type": "string",
                "enum": ["celsius", "fahrenheit"],
                "description": "Temperature unit. Defaults to fahrenheit.",
            },
        },
        "required": ["location"],
    },
}
```

The `input_schema` follows [JSON Schema](https://json-schema.org/) syntax.

### Field Guidelines

- **`name`**: Use descriptive snake_case names. Must match `[a-zA-Z0-9_-]{1,64}`.
- **`description`**: Be specific. Include what the tool does, when to use it, and what it returns. Better descriptions = more reliable tool use.
- **`input_schema`**: Define every parameter clearly with types and descriptions. Mark required fields.

---

## Basic Tool Use Flow

The flow for client tools has four steps:

1. Send request with tool definitions
2. Claude returns `stop_reason: "tool_use"` with a `tool_use` content block
3. Execute the tool and send back a `tool_result`
4. Claude generates its final response

```python
import anthropic

client = anthropic.Anthropic()

# Step 1: Define tools and send first request
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    tools=[
        {
            "name": "get_weather",
            "description": "Get the current weather for a location",
            "input_schema": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "City and state, e.g. 'San Francisco, CA'",
                    }
                },
                "required": ["location"],
            },
        }
    ],
    messages=[{"role": "user", "content": "What's the weather like in San Francisco?"}],
)

# Step 2: Claude decides to use the tool
print(response.stop_reason)  # "tool_use"
tool_use_block = response.content[0]
print(tool_use_block.type)   # "tool_use"
print(tool_use_block.name)   # "get_weather"
print(tool_use_block.input)  # {"location": "San Francisco, CA"}
print(tool_use_block.id)     # "toolu_01A09q90qw90lq917835lq9"
```

---

## Complete Multi-step Example

```python
import anthropic
import json

client = anthropic.Anthropic()

tools = [
    {
        "name": "get_weather",
        "description": "Get current weather for a location",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "City and state"},
            },
            "required": ["location"],
        },
    }
]

def get_weather(location: str) -> str:
    """Mock weather function"""
    return f"The weather in {location} is 68°F and partly cloudy."

messages = [{"role": "user", "content": "What's the weather in Boston?"}]

# First API call
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    tools=tools,
    messages=messages,
)

# Process tool calls in a loop
while response.stop_reason == "tool_use":
    # Collect all tool use blocks
    tool_results = []
    for block in response.content:
        if block.type == "tool_use":
            # Execute the tool
            if block.name == "get_weather":
                result = get_weather(block.input["location"])
            else:
                result = f"Unknown tool: {block.name}"

            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": result,
            })

    # Append assistant response and tool results to history
    messages.append({"role": "assistant", "content": response.content})
    messages.append({"role": "user", "content": tool_results})

    # Get next response
    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=1024,
        tools=tools,
        messages=messages,
    )

# Final text response
print(response.content[0].text)
# "The weather in Boston is 68°F and partly cloudy."
```

---

## tool_result Block

After executing a tool, send back results in a `tool_result` content block within a user turn:

```python
# Successful result (string)
{
    "type": "tool_result",
    "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
    "content": "The weather is 68°F and sunny.",
}

# Successful result (structured content blocks)
{
    "type": "tool_result",
    "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
    "content": [
        {"type": "text", "text": "Temperature: 68°F"},
        {"type": "text", "text": "Conditions: Partly cloudy"},
    ],
}

# Error result
{
    "type": "tool_result",
    "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
    "content": "Error: Location 'XYZ' not found",
    "is_error": True,
}
```

When `is_error=True`, Claude will acknowledge the error and may try a different approach.

---

## Parallel Tool Use

Claude can request multiple tools simultaneously when they are independent. The response contains multiple `tool_use` blocks:

```python
import anthropic

client = anthropic.Anthropic()

tools = [
    {
        "name": "get_weather",
        "description": "Get weather for a location",
        "input_schema": {
            "type": "object",
            "properties": {"location": {"type": "string"}},
            "required": ["location"],
        },
    },
    {
        "name": "get_time",
        "description": "Get current local time for a timezone",
        "input_schema": {
            "type": "object",
            "properties": {"timezone": {"type": "string", "description": "e.g. 'America/New_York'"}},
            "required": ["timezone"],
        },
    },
]

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "What's the weather and time in New York?"}],
)

# Claude may return multiple tool_use blocks
for block in response.content:
    if block.type == "tool_use":
        print(f"Tool: {block.name}, Input: {block.input}, ID: {block.id}")

# Execute all tools in parallel, then return all results
tool_results = []
for block in response.content:
    if block.type == "tool_use":
        if block.name == "get_weather":
            result = "72°F, sunny"
        elif block.name == "get_time":
            result = "3:45 PM EST"
        tool_results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": result,
        })

# Send all results in a single user message
messages = [
    {"role": "user", "content": "What's the weather and time in New York?"},
    {"role": "assistant", "content": response.content},
    {"role": "user", "content": tool_results},
]

final = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    tools=tools,
    messages=messages,
)
print(final.content[0].text)
```

To disable parallel tool use:

```python
tool_choice={"type": "auto", "disable_parallel_tool_use": True}
```

---

## Tool Choice

Control which tool Claude must use:

```python
# Auto (default): Claude decides whether to use a tool and which one
tool_choice={"type": "auto"}

# Any: Claude must use at least one tool (but chooses which)
tool_choice={"type": "any"}

# Force a specific tool
tool_choice={"type": "tool", "name": "get_weather"}

# None: Claude cannot use any tools (text only)
tool_choice={"type": "none"}
```

Example forcing a specific tool:

```python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    tools=tools,
    tool_choice={"type": "tool", "name": "get_weather"},
    messages=[{"role": "user", "content": "Tell me about Paris."}],
)
# Claude will call get_weather even for a general question
```

---

## Strict Tool Use (Schema Validation)

Add `"strict": True` to guarantee Claude's tool inputs always match your schema exactly:

```python
tool = {
    "name": "create_user",
    "description": "Create a new user account",
    "input_schema": {
        "type": "object",
        "properties": {
            "username": {"type": "string"},
            "email": {"type": "string", "format": "email"},
            "age": {"type": "integer", "minimum": 18},
        },
        "required": ["username", "email"],
        "additionalProperties": False,
    },
    "strict": True,  # Enforce schema validation
}
```

Use strict mode in production agents where invalid tool parameters would cause failures.

---

## Server-side Tools

Server tools execute on Anthropic's servers — no client-side implementation needed. They use versioned type identifiers:

### Web Search

```python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    tools=[
        {
            "type": "web_search_20260209",
            "name": "web_search",
        }
    ],
    messages=[{"role": "user", "content": "What happened in tech news today?"}],
)
print(response.content[0].text)
```

### Web Fetch

```python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=2048,
    tools=[
        {
            "type": "web_fetch_20260209",
            "name": "web_fetch",
        }
    ],
    messages=[{"role": "user", "content": "Fetch and summarize https://example.com/article"}],
)
```

### Code Execution

```python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=2048,
    tools=[
        {
            "type": "code_execution_20250825",
            "name": "code_execution",
        }
    ],
    messages=[{"role": "user", "content": "Calculate the first 20 fibonacci numbers."}],
)
```

### Bash Tool (client-side)

```python
# The bash tool requires client-side implementation
tools=[
    {
        "type": "bash_20250124",
        "name": "bash",
    }
]
```

---

## Computer Use

Computer use tools allow Claude to interact with a desktop environment. These are client-side tools requiring implementation:

```python
tools = [
    {
        "type": "computer_20241022",
        "name": "computer",
        "display_width_px": 1024,
        "display_height_px": 768,
        "display_number": 1,
    },
    {
        "type": "text_editor_20250728",
        "name": "str_replace_based_edit_tool",
    },
    {
        "type": "bash_20250124",
        "name": "bash",
    },
]

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=4096,
    tools=tools,
    messages=[{"role": "user", "content": "Take a screenshot of the desktop."}],
    betas=["computer-use-2024-10-22"],
)
```

Computer use is a beta feature requiring the `betas` header. See the computer use documentation for full implementation details including screenshot capture and mouse/keyboard action execution.

---

## Best Practices

1. **Write detailed descriptions**: Describe not just what the tool does but when Claude should use it, what inputs are valid, and what the output looks like.

2. **Use clear parameter names**: `user_id` is better than `id`; `start_date_iso` is better than `date`.

3. **Mark required fields explicitly**: Only fields that are truly required should be in `required`.

4. **Handle errors gracefully**: Return `is_error: True` with a clear message so Claude can adapt.

5. **Use strict mode in production**: Prevents type mismatches from breaking your application.

6. **Cache tool definitions**: If your tools rarely change, add `cache_control` to the last tool definition to save tokens on repeated calls:

```python
tools[-1]["cache_control"] = {"type": "ephemeral"}
```

7. **Limit tool count**: Fewer tools = more reliable selection. Expose only what's needed for the task.
