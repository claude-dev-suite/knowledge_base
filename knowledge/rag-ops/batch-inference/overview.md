# Batch Inference -- OpenAI Batch, Anthropic Message Batches, Voyage Batch

## Overview

Batch inference APIs allow you to submit large volumes of embedding or generation requests at once, processed asynchronously with significant cost savings. Instead of making thousands of individual API calls, you upload a batch file and retrieve results when processing completes. The three major providers offering batch APIs are OpenAI (Batch API), Anthropic (Message Batches), and Voyage AI (batch embedding endpoints).

The cost savings are substantial: OpenAI offers 50% off batch requests, and Anthropic offers 50% off with Message Batches. The tradeoff is latency -- batch results are typically available within 24 hours rather than immediately.

This guide covers how each provider's batch API works, when batch processing is appropriate, and the architectural patterns for integrating batch inference into RAG pipelines.

---

## When to Use Batch Inference

### Good Fit

- **Bulk document embedding**: Embedding 100K+ documents for initial index building
- **Periodic re-indexing**: Nightly or weekly re-embedding of updated documents
- **Evaluation datasets**: Running LLM judges on hundreds of evaluation examples
- **Content generation**: Generating descriptions, summaries, or metadata for a document corpus
- **Quality audits**: Running quality checks on all responses in a RAG system

### Poor Fit

- **Real-time queries**: User-facing search and generation needs immediate responses
- **Interactive chat**: Multi-turn conversations require synchronous responses
- **Low volume**: <100 requests do not benefit from batch overhead
- **Time-sensitive updates**: If results are needed within minutes, not hours

---

## OpenAI Batch API

### How It Works

1. Create a JSONL file where each line is a complete API request
2. Upload the file to OpenAI
3. Create a batch job referencing the file
4. Poll for completion (typically 1-24 hours)
5. Download the results JSONL file

### Batch Embeddings

```python
import openai
import json
import time
from pathlib import Path


def create_embedding_batch(
    texts: list[str],
    model: str = "text-embedding-3-small",
    output_file: str = "batch_input.jsonl",
) -> str:
    """Create a JSONL batch file for embedding requests."""
    with open(output_file, "w") as f:
        for i, text in enumerate(texts):
            request = {
                "custom_id": f"embed-{i}",
                "method": "POST",
                "url": "/v1/embeddings",
                "body": {
                    "model": model,
                    "input": text,
                },
            }
            f.write(json.dumps(request) + "\n")

    print(f"Created batch file with {len(texts)} requests: {output_file}")
    return output_file


def submit_batch(input_file: str) -> str:
    """Upload file and create a batch job."""
    client = openai.OpenAI()

    # Upload the input file
    with open(input_file, "rb") as f:
        uploaded = client.files.create(file=f, purpose="batch")

    print(f"File uploaded: {uploaded.id}")

    # Create the batch
    batch = client.batches.create(
        input_file_id=uploaded.id,
        endpoint="/v1/embeddings",
        completion_window="24h",
        metadata={"description": "Bulk document embedding"},
    )

    print(f"Batch created: {batch.id}")
    print(f"Status: {batch.status}")
    return batch.id


def wait_for_batch(batch_id: str, poll_interval: int = 60) -> dict:
    """Poll until batch completes."""
    client = openai.OpenAI()

    while True:
        batch = client.batches.retrieve(batch_id)
        status = batch.status

        print(f"Status: {status} | "
              f"Completed: {batch.request_counts.completed}/{batch.request_counts.total} | "
              f"Failed: {batch.request_counts.failed}")

        if status == "completed":
            return batch
        elif status in ("failed", "expired", "cancelled"):
            raise RuntimeError(f"Batch {batch_id} {status}: {batch.errors}")

        time.sleep(poll_interval)


def download_results(batch) -> list[dict]:
    """Download and parse batch results."""
    client = openai.OpenAI()

    output_file_id = batch.output_file_id
    content = client.files.content(output_file_id)

    results = []
    for line in content.text.strip().split("\n"):
        result = json.loads(line)
        results.append({
            "custom_id": result["custom_id"],
            "embedding": result["response"]["body"]["data"][0]["embedding"],
            "usage": result["response"]["body"]["usage"],
        })

    return results


# Complete workflow
texts = [f"Document {i}: content about topic {i}" for i in range(1000)]

input_file = create_embedding_batch(texts, model="text-embedding-3-small")
batch_id = submit_batch(input_file)
batch = wait_for_batch(batch_id)
results = download_results(batch)

print(f"Got {len(results)} embeddings")
print(f"First embedding dimensions: {len(results[0]['embedding'])}")
```

### Batch Chat Completions (LLM Generation)

```python
def create_generation_batch(
    prompts: list[dict],
    model: str = "gpt-4o-mini",
    output_file: str = "batch_generation.jsonl",
) -> str:
    """Create a batch file for chat completion requests."""
    with open(output_file, "w") as f:
        for i, prompt in enumerate(prompts):
            request = {
                "custom_id": f"gen-{i}",
                "method": "POST",
                "url": "/v1/chat/completions",
                "body": {
                    "model": model,
                    "messages": prompt["messages"],
                    "temperature": prompt.get("temperature", 0.0),
                    "max_tokens": prompt.get("max_tokens", 500),
                },
            }
            f.write(json.dumps(request) + "\n")

    return output_file


# Example: batch RAG evaluation
eval_prompts = []
for question, context, reference in evaluation_dataset:
    eval_prompts.append({
        "messages": [
            {
                "role": "system",
                "content": "You are an evaluator. Rate the answer on a scale of 1-5.",
            },
            {
                "role": "user",
                "content": f"Question: {question}\nContext: {context}\n"
                           f"Answer to evaluate: {reference}\nScore (1-5):",
            },
        ],
        "temperature": 0.0,
        "max_tokens": 10,
    })

batch_file = create_generation_batch(eval_prompts)
batch_id = submit_batch(batch_file)
```

---

## Anthropic Message Batches

### How It Works

Anthropic's Message Batches API processes up to 100,000 messages per batch with 50% cost reduction. Results are available within 24 hours.

```python
import anthropic
import json
import time


def create_anthropic_batch(
    requests: list[dict],
) -> str:
    """Submit a batch of message requests to Anthropic."""
    client = anthropic.Anthropic()

    # Each request in the batch
    batch_requests = []
    for i, req in enumerate(requests):
        batch_requests.append({
            "custom_id": f"req-{i}",
            "params": {
                "model": req.get("model", "claude-sonnet-4-20250514"),
                "max_tokens": req.get("max_tokens", 500),
                "messages": req["messages"],
                "temperature": req.get("temperature", 0.0),
            },
        })

    # Create the batch
    batch = client.messages.batches.create(requests=batch_requests)

    print(f"Batch created: {batch.id}")
    print(f"Status: {batch.processing_status}")
    return batch.id


def wait_for_anthropic_batch(batch_id: str, poll_interval: int = 60) -> dict:
    """Poll until Anthropic batch completes."""
    client = anthropic.Anthropic()

    while True:
        batch = client.messages.batches.retrieve(batch_id)
        status = batch.processing_status

        counts = batch.request_counts
        print(f"Status: {status} | "
              f"Succeeded: {counts.succeeded}/{counts.total} | "
              f"Failed: {counts.errored}")

        if status == "ended":
            return batch
        time.sleep(poll_interval)


def download_anthropic_results(batch_id: str) -> list[dict]:
    """Download results from an Anthropic batch."""
    client = anthropic.Anthropic()

    results = []
    for result in client.messages.batches.results(batch_id):
        if result.result.type == "succeeded":
            results.append({
                "custom_id": result.custom_id,
                "content": result.result.message.content[0].text,
                "usage": {
                    "input_tokens": result.result.message.usage.input_tokens,
                    "output_tokens": result.result.message.usage.output_tokens,
                },
            })
        else:
            results.append({
                "custom_id": result.custom_id,
                "error": str(result.result),
            })

    return results


# Usage: batch evaluation with Claude
eval_requests = [
    {
        "model": "claude-sonnet-4-20250514",
        "max_tokens": 100,
        "messages": [
            {
                "role": "user",
                "content": f"Rate this RAG answer (1-5):\n"
                           f"Question: {q}\nAnswer: {a}\nScore:",
            },
        ],
    }
    for q, a in zip(questions, answers)
]

batch_id = create_anthropic_batch(eval_requests)
batch = wait_for_anthropic_batch(batch_id)
results = download_anthropic_results(batch_id)
```

---

## Voyage AI Batch Embedding

```python
import voyageai
import time


def batch_embed_voyage(
    texts: list[str],
    model: str = "voyage-3",
    batch_size: int = 128,
) -> list[list[float]]:
    """Embed texts in batches using Voyage AI."""
    client = voyageai.Client()
    all_embeddings = []

    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        result = client.embed(
            texts=batch,
            model=model,
            input_type="document",  # or "query" for search queries
        )
        all_embeddings.extend(result.embeddings)

        print(f"Embedded {min(i + batch_size, len(texts))}/{len(texts)}")

    return all_embeddings


# Usage
texts = [f"Document about topic {i}" for i in range(10000)]
embeddings = batch_embed_voyage(texts, model="voyage-3")
print(f"Got {len(embeddings)} embeddings of dim {len(embeddings[0])}")
```

---

## Cost Comparison

### Standard vs. Batch Pricing

| Provider | Model | Standard Price | Batch Price | Savings |
|----------|-------|---------------|-------------|---------|
| OpenAI | `text-embedding-3-small` | $0.02/1M tokens | $0.01/1M tokens | 50% |
| OpenAI | `text-embedding-3-large` | $0.13/1M tokens | $0.065/1M tokens | 50% |
| OpenAI | `gpt-4o-mini` | $0.15/$0.60 per 1M in/out | $0.075/$0.30 | 50% |
| OpenAI | `gpt-4o` | $2.50/$10 per 1M in/out | $1.25/$5.00 | 50% |
| Anthropic | `claude-sonnet-4-20250514` | $3/$15 per 1M in/out | $1.50/$7.50 | 50% |
| Anthropic | `claude-haiku-4-20250514` | $0.80/$4 per 1M in/out | $0.40/$2.00 | 50% |

### Cost Calculation Example

```python
def calculate_batch_savings(
    num_requests: int,
    avg_input_tokens: int,
    avg_output_tokens: int,
    model: str = "gpt-4o-mini",
) -> dict:
    """Calculate savings from batch API."""
    pricing = {
        "gpt-4o-mini": {"input": 0.15, "output": 0.60, "batch_discount": 0.5},
        "gpt-4o": {"input": 2.50, "output": 10.00, "batch_discount": 0.5},
        "claude-sonnet-4-20250514": {"input": 3.00, "output": 15.00, "batch_discount": 0.5},
        "text-embedding-3-small": {"input": 0.02, "output": 0, "batch_discount": 0.5},
    }

    p = pricing[model]
    total_input_tokens = num_requests * avg_input_tokens
    total_output_tokens = num_requests * avg_output_tokens

    standard_cost = (
        (total_input_tokens / 1_000_000) * p["input"]
        + (total_output_tokens / 1_000_000) * p["output"]
    )

    batch_cost = standard_cost * (1 - p["batch_discount"])
    savings = standard_cost - batch_cost

    return {
        "standard_cost": standard_cost,
        "batch_cost": batch_cost,
        "savings": savings,
        "savings_pct": (savings / standard_cost * 100) if standard_cost > 0 else 0,
    }


# Example: evaluating 10,000 RAG responses
result = calculate_batch_savings(
    num_requests=10_000,
    avg_input_tokens=500,
    avg_output_tokens=50,
    model="gpt-4o-mini",
)
print(f"Standard: ${result['standard_cost']:.2f}")
print(f"Batch:    ${result['batch_cost']:.2f}")
print(f"Savings:  ${result['savings']:.2f} ({result['savings_pct']:.0f}%)")
```

---

## Common Pitfalls

1. **Expecting real-time results**: Batch APIs are asynchronous. OpenAI guarantees completion within 24 hours, not minutes. Do not use batch for user-facing features.

2. **Not handling partial failures**: Some requests in a batch may fail while others succeed. Always check each result's status and implement retry logic for failed requests.

3. **Exceeding batch size limits**: OpenAI limits batches to 50,000 requests or 200MB file size. Split larger jobs into multiple batches.

4. **Forgetting to download results**: Batch output files are available for 24 hours after completion. Implement automatic download in your pipeline or you lose the results.

5. **Not tracking custom_ids**: The `custom_id` field is how you map results back to inputs. Use deterministic IDs (e.g., document IDs) so you can correlate results after download.

6. **Over-batching small jobs**: The overhead of file upload, batch creation, and polling is not worth it for fewer than 100 requests. Use standard API calls for small volumes.

---

## References

- OpenAI Batch API: https://platform.openai.com/docs/guides/batch
- Anthropic Message Batches: https://docs.anthropic.com/en/docs/build-with-claude/message-batches
- Voyage AI: https://docs.voyageai.com/
- OpenAI pricing: https://openai.com/pricing
