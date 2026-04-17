# Batch Inference -- Cost Analysis: 50% Savings Math and Latency Tradeoffs

## Overview

Batch inference offers 50% cost reduction on major LLM and embedding APIs. This guide quantifies the savings across different workload types, analyzes the latency tradeoffs, and provides frameworks for deciding when batch processing is worth the architectural complexity.

---

## Pricing Tables (as of 2025)

### Embedding Models

| Provider | Model | Standard (per 1M tokens) | Batch (per 1M tokens) | Savings |
|----------|-------|--------------------------|----------------------|---------|
| OpenAI | `text-embedding-3-small` | $0.020 | $0.010 | 50% |
| OpenAI | `text-embedding-3-large` | $0.130 | $0.065 | 50% |
| Voyage | `voyage-3` | $0.060 | $0.030 | 50% |
| Voyage | `voyage-3-lite` | $0.020 | $0.010 | 50% |

### Generation Models

| Provider | Model | Standard Input/Output | Batch Input/Output | Savings |
|----------|-------|----------------------|--------------------|---------| 
| OpenAI | `gpt-4o-mini` | $0.15 / $0.60 | $0.075 / $0.30 | 50% |
| OpenAI | `gpt-4o` | $2.50 / $10.00 | $1.25 / $5.00 | 50% |
| Anthropic | `claude-sonnet-4-20250514` | $3.00 / $15.00 | $1.50 / $7.50 | 50% |
| Anthropic | `claude-haiku-4-20250514` | $0.80 / $4.00 | $0.40 / $2.00 | 50% |
| Anthropic | `claude-opus-4-20250514` | $15.00 / $75.00 | $7.50 / $37.50 | 50% |

---

## Cost Calculations by Workload Type

### Workload 1: Document Embedding (Index Building)

```python
def embedding_cost(
    num_documents: int,
    avg_tokens_per_doc: int,
    model: str = "text-embedding-3-small",
    use_batch: bool = False,
) -> dict:
    """Calculate embedding cost for a document collection."""
    pricing = {
        "text-embedding-3-small": {"standard": 0.020, "batch": 0.010},
        "text-embedding-3-large": {"standard": 0.130, "batch": 0.065},
        "voyage-3": {"standard": 0.060, "batch": 0.030},
    }

    p = pricing[model]
    total_tokens = num_documents * avg_tokens_per_doc
    rate = p["batch"] if use_batch else p["standard"]
    cost = (total_tokens / 1_000_000) * rate

    return {
        "num_documents": num_documents,
        "total_tokens": total_tokens,
        "cost": cost,
        "model": model,
        "batch": use_batch,
    }


# Example: embedding 1M documents at 256 tokens each
for model in ["text-embedding-3-small", "text-embedding-3-large", "voyage-3"]:
    standard = embedding_cost(1_000_000, 256, model, use_batch=False)
    batch = embedding_cost(1_000_000, 256, model, use_batch=True)
    print(f"{model}:")
    print(f"  Standard: ${standard['cost']:,.2f}")
    print(f"  Batch:    ${batch['cost']:,.2f}")
    print(f"  Savings:  ${standard['cost'] - batch['cost']:,.2f}")
    print()
```

Output:
```
text-embedding-3-small:
  Standard: $5.12
  Batch:    $2.56
  Savings:  $2.56

text-embedding-3-large:
  Standard: $33.28
  Batch:    $16.64
  Savings:  $16.64

voyage-3:
  Standard: $15.36
  Batch:    $7.68
  Savings:  $7.68
```

### Workload 2: RAG Evaluation

```python
def evaluation_cost(
    num_evaluations: int,
    avg_context_tokens: int = 2000,
    avg_question_tokens: int = 50,
    avg_response_tokens: int = 200,
    avg_judge_output_tokens: int = 100,
    model: str = "gpt-4o-mini",
    use_batch: bool = False,
) -> dict:
    """Calculate cost for RAG evaluation pipeline."""
    pricing = {
        "gpt-4o-mini": {
            "input_standard": 0.15, "output_standard": 0.60,
            "input_batch": 0.075, "output_batch": 0.30,
        },
        "gpt-4o": {
            "input_standard": 2.50, "output_standard": 10.00,
            "input_batch": 1.25, "output_batch": 5.00,
        },
        "claude-sonnet-4-20250514": {
            "input_standard": 3.00, "output_standard": 15.00,
            "input_batch": 1.50, "output_batch": 7.50,
        },
    }

    p = pricing[model]
    total_input = num_evaluations * (avg_context_tokens + avg_question_tokens + avg_response_tokens)
    total_output = num_evaluations * avg_judge_output_tokens

    if use_batch:
        cost = (total_input / 1e6) * p["input_batch"] + (total_output / 1e6) * p["output_batch"]
    else:
        cost = (total_input / 1e6) * p["input_standard"] + (total_output / 1e6) * p["output_standard"]

    return {"cost": cost, "total_input_tokens": total_input, "total_output_tokens": total_output}


# Evaluate 10,000 RAG responses
for model in ["gpt-4o-mini", "claude-sonnet-4-20250514"]:
    std = evaluation_cost(10_000, model=model, use_batch=False)
    bat = evaluation_cost(10_000, model=model, use_batch=True)
    print(f"{model} (10K evaluations):")
    print(f"  Standard: ${std['cost']:,.2f}")
    print(f"  Batch:    ${bat['cost']:,.2f}")
    print(f"  Savings:  ${std['cost'] - bat['cost']:,.2f}")
    print()
```

### Workload 3: Bulk Content Generation

```python
def generation_cost(
    num_requests: int,
    avg_input_tokens: int = 500,
    avg_output_tokens: int = 1000,
    model: str = "gpt-4o-mini",
    use_batch: bool = False,
) -> dict:
    """Calculate cost for bulk generation."""
    pricing = {
        "gpt-4o-mini": {
            "input_standard": 0.15, "output_standard": 0.60,
            "input_batch": 0.075, "output_batch": 0.30,
        },
        "gpt-4o": {
            "input_standard": 2.50, "output_standard": 10.00,
            "input_batch": 1.25, "output_batch": 5.00,
        },
    }

    p = pricing[model]
    total_input = num_requests * avg_input_tokens
    total_output = num_requests * avg_output_tokens

    if use_batch:
        cost = (total_input / 1e6) * p["input_batch"] + (total_output / 1e6) * p["output_batch"]
    else:
        cost = (total_input / 1e6) * p["input_standard"] + (total_output / 1e6) * p["output_standard"]

    return {"cost": cost}


# Generate summaries for 50K documents
for model in ["gpt-4o-mini", "gpt-4o"]:
    std = generation_cost(50_000, model=model, use_batch=False)
    bat = generation_cost(50_000, model=model, use_batch=True)
    savings = std["cost"] - bat["cost"]
    print(f"{model} (50K generations): Standard ${std['cost']:,.2f} -> Batch ${bat['cost']:,.2f} (save ${savings:,.2f})")
```

---

## Latency Tradeoffs

### Latency Comparison

| Approach | P50 Latency | P99 Latency | Throughput |
|----------|-------------|-------------|------------|
| Standard API (single) | 80-200ms | 500ms-2s | 10-50 req/s |
| Standard API (concurrent) | 80-200ms | 1-5s | 100-500 req/s |
| Batch API | 1-24 hours | 24 hours | Unlimited |
| Self-hosted (GPU) | 5-15ms | 50ms | 2000-5000 req/s |

### Latency vs. Cost Decision Matrix

| Volume (per month) | Latency Need | Best Approach | Monthly Cost (embedding) |
|---------------------|-------------|---------------|--------------------------|
| <1M embeddings | Real-time | Standard API | <$1 |
| 1-10M embeddings | Real-time | Standard API | $1-$10 |
| 10-50M embeddings | Real-time | Self-hosted GPU | $300-$500 |
| 10-50M embeddings | Can wait hours | Batch API | $5-$25 |
| 50-500M embeddings | Real-time | Self-hosted GPU | $500-$2,000 |
| 50-500M embeddings | Can wait hours | Batch API | $25-$250 |
| >500M embeddings | Any | Self-hosted GPU | $2,000+ (amortized) |

### Hybrid Architecture: Real-Time + Batch

```python
class HybridEmbeddingService:
    """
    Real-time embeddings for queries + batch embeddings for indexing.
    Saves ~40% compared to all-real-time.
    """

    def __init__(self):
        self.realtime_client = openai.OpenAI()
        self.batch_pipeline = BatchEmbeddingPipeline()

    def embed_query(self, query: str) -> list[float]:
        """Real-time embedding for search queries (latency-sensitive)."""
        response = self.realtime_client.embeddings.create(
            input=query,
            model="text-embedding-3-small",
        )
        return response.data[0].embedding

    def embed_documents_batch(self, documents: list[dict]) -> dict:
        """Batch embedding for document indexing (cost-sensitive)."""
        return self.batch_pipeline.embed_documents(documents)

    def cost_estimate(self, query_volume: int, doc_volume: int, avg_tokens: int = 128):
        """Estimate monthly cost for hybrid approach."""
        query_cost = (query_volume * avg_tokens / 1e6) * 0.020  # standard rate
        doc_cost = (doc_volume * avg_tokens / 1e6) * 0.010     # batch rate (50% off)
        total = query_cost + doc_cost

        # Compare with all-standard
        all_standard = ((query_volume + doc_volume) * avg_tokens / 1e6) * 0.020
        savings = all_standard - total

        return {
            "query_cost": query_cost,
            "doc_cost": doc_cost,
            "total": total,
            "all_standard": all_standard,
            "savings": savings,
            "savings_pct": (savings / all_standard * 100) if all_standard > 0 else 0,
        }
```

---

## Annual Cost Projections

```python
def annual_projection(
    monthly_embeddings: int,
    monthly_generations: int,
    embedding_model: str = "text-embedding-3-small",
    generation_model: str = "gpt-4o-mini",
    batch_pct: float = 0.8,  # % of workload that can be batched
):
    """Project annual costs with different batch adoption rates."""
    embed_pricing = {
        "text-embedding-3-small": {"standard": 0.020, "batch": 0.010},
    }
    gen_pricing = {
        "gpt-4o-mini": {
            "input_standard": 0.15, "output_standard": 0.60,
            "input_batch": 0.075, "output_batch": 0.30,
        },
    }

    ep = embed_pricing[embedding_model]
    gp = gen_pricing[generation_model]

    # Embedding costs (128 avg tokens)
    embed_tokens = monthly_embeddings * 128
    embed_standard = (embed_tokens / 1e6) * ep["standard"] * 12
    embed_hybrid = (
        (embed_tokens * (1 - batch_pct) / 1e6) * ep["standard"]
        + (embed_tokens * batch_pct / 1e6) * ep["batch"]
    ) * 12

    # Generation costs (500 input, 200 output avg)
    gen_input = monthly_generations * 500
    gen_output = monthly_generations * 200
    gen_standard = (
        (gen_input / 1e6) * gp["input_standard"]
        + (gen_output / 1e6) * gp["output_standard"]
    ) * 12
    gen_hybrid = (
        (gen_input * (1 - batch_pct) / 1e6) * gp["input_standard"]
        + (gen_input * batch_pct / 1e6) * gp["input_batch"]
        + (gen_output * (1 - batch_pct) / 1e6) * gp["output_standard"]
        + (gen_output * batch_pct / 1e6) * gp["output_batch"]
    ) * 12

    total_standard = embed_standard + gen_standard
    total_hybrid = embed_hybrid + gen_hybrid

    return {
        "annual_standard": total_standard,
        "annual_hybrid": total_hybrid,
        "annual_savings": total_standard - total_hybrid,
    }


# Project for different scales
for emb, gen in [(10_000_000, 100_000), (100_000_000, 1_000_000), (500_000_000, 5_000_000)]:
    result = annual_projection(emb, gen)
    print(f"{emb/1e6:.0f}M emb + {gen/1e3:.0f}K gen/month:")
    print(f"  Standard: ${result['annual_standard']:>10,.2f}/year")
    print(f"  Hybrid:   ${result['annual_hybrid']:>10,.2f}/year")
    print(f"  Savings:  ${result['annual_savings']:>10,.2f}/year")
    print()
```

---

## Decision Framework

### Should You Use Batch?

```
Is latency critical for this workload?
  YES -> Use standard API or self-hosted
  NO  -> Continue

Is volume > 10K requests per batch?
  YES -> Batch saves meaningful money
  NO  -> Overhead not worth it, use standard

Can you tolerate up to 24h delay?
  YES -> Use batch API (50% savings)
  NO  -> Use standard API with concurrency

Is volume > 50M embeddings/month?
  YES -> Consider self-hosted (80-95% savings)
  NO  -> Batch API is the sweet spot
```

---

## Common Pitfalls

1. **Assuming batch is always cheaper**: For very low volumes (<1000 requests), the engineering overhead of batch pipeline code exceeds the cost savings. Use standard API for small jobs.

2. **Not accounting for developer time**: Building and maintaining batch pipelines takes engineering hours. Factor in development cost when calculating ROI.

3. **Ignoring the 24-hour SLA**: OpenAI guarantees batch completion within 24 hours, not 1 hour. Plan your pipeline accordingly. Some batches complete in 30 minutes, but do not depend on it.

4. **Double-spending on retries**: If you retry a batch that actually completed (but you missed the completion), you pay twice. Implement idempotent tracking of batch jobs.

5. **Not monitoring batch failure rates**: Track what percentage of requests fail in each batch. Rising failure rates indicate input quality issues or API problems.

6. **Mixing batch and real-time models**: Ensure the batch model matches your real-time model. Using `text-embedding-3-small` in batch but `text-embedding-3-large` in real-time produces incompatible embeddings.

---

## References

- OpenAI Batch API: https://platform.openai.com/docs/guides/batch
- OpenAI pricing: https://openai.com/pricing
- Anthropic Message Batches: https://docs.anthropic.com/en/docs/build-with-claude/message-batches
- Anthropic pricing: https://docs.anthropic.com/en/docs/about-claude/pricing
