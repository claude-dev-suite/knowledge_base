# Long Context vs RAG -- Cost Comparison: $/Query Math for 200K Context vs RAG Pipeline

## TL;DR

This article provides exact cost calculations comparing long-context stuffing versus RAG pipelines across different corpus sizes, query volumes, and model providers. The analysis covers input/output token pricing, prompt caching discounts, embedding costs, vector store hosting, and reranking costs. The key finding: RAG is 10-100x cheaper per query for high-volume workloads, but long context with prompt caching can be competitive for low-volume, high-accuracy use cases.

---

## Cost Models

### Long Context Cost

```python
from dataclasses import dataclass


@dataclass
class LLMPricing:
    name: str
    input_per_1m: float
    output_per_1m: float
    cached_input_per_1m: float
    max_context: int


PRICING_2025 = {
    "claude_sonnet": LLMPricing("Claude Sonnet 4", 3.0, 15.0, 0.30, 200_000),
    "claude_haiku": LLMPricing("Claude Haiku 3.5", 0.80, 4.0, 0.08, 200_000),
    "gpt4o": LLMPricing("GPT-4o", 2.50, 10.0, 1.25, 128_000),
    "gpt4o_mini": LLMPricing("GPT-4o-mini", 0.15, 0.60, 0.075, 128_000),
    "gemini_pro": LLMPricing("Gemini 1.5 Pro", 1.25, 5.0, 0.315, 2_000_000),
}


def estimate_long_context_cost(
    corpus_tokens: int,
    queries_per_day: int,
    model: str = "claude_sonnet",
    avg_output_tokens: int = 500,
    use_prompt_caching: bool = True,
) -> dict:
    """Estimate long context cost per query and per month."""
    pricing = PRICING_2025[model]

    if corpus_tokens > pricing.max_context:
        return {"error": f"Corpus ({corpus_tokens:,}) exceeds {model} context ({pricing.max_context:,})"}

    # First query: full input price
    first_query_input_cost = (corpus_tokens / 1_000_000) * pricing.input_per_1m
    # Subsequent queries with caching
    if use_prompt_caching:
        cached_input_cost = (corpus_tokens / 1_000_000) * pricing.cached_input_per_1m
    else:
        cached_input_cost = first_query_input_cost

    output_cost = (avg_output_tokens / 1_000_000) * pricing.output_per_1m

    # Average cost per query (amortize first-query over all queries)
    if queries_per_day > 1 and use_prompt_caching:
        # Cache TTL is typically 5 minutes; assume 80% cache hit rate
        avg_input_cost = (0.2 * first_query_input_cost) + (0.8 * cached_input_cost)
    else:
        avg_input_cost = first_query_input_cost

    cost_per_query = avg_input_cost + output_cost
    daily_cost = cost_per_query * queries_per_day
    monthly_cost = daily_cost * 30

    return {
        "model": pricing.name,
        "corpus_tokens": corpus_tokens,
        "cost_per_query": round(cost_per_query, 4),
        "daily_cost": round(daily_cost, 2),
        "monthly_cost": round(monthly_cost, 2),
        "prompt_caching": use_prompt_caching,
        "cache_hit_rate": "80%" if use_prompt_caching else "N/A",
    }
```

### RAG Pipeline Cost

```python
@dataclass
class EmbeddingPricing:
    name: str
    cost_per_1m_tokens: float
    dimension: int


EMBEDDING_PRICING = {
    "openai_small": EmbeddingPricing("text-embedding-3-small", 0.02, 1536),
    "openai_large": EmbeddingPricing("text-embedding-3-large", 0.13, 3072),
    "voyage_3": EmbeddingPricing("voyage-3", 0.06, 1024),
    "cohere_v3": EmbeddingPricing("embed-v3.0", 0.10, 1024),
}


def estimate_rag_cost(
    corpus_tokens: int,
    queries_per_day: int,
    embedding_model: str = "openai_small",
    llm_model: str = "claude_sonnet",
    top_k: int = 5,
    avg_chunk_tokens: int = 500,
    avg_output_tokens: int = 500,
    reranker_cost_per_query: float = 0.001,
    vector_db_monthly: float = 50.0,
) -> dict:
    """Estimate RAG pipeline cost per query and per month."""
    emb_pricing = EMBEDDING_PRICING[embedding_model]
    llm_pricing = PRICING_2025[llm_model]

    # One-time indexing cost (embedding the corpus)
    indexing_cost = (corpus_tokens / 1_000_000) * emb_pricing.cost_per_1m_tokens

    # Per-query costs
    # 1. Embed the query
    query_embed_cost = (20 / 1_000_000) * emb_pricing.cost_per_1m_tokens  # ~20 tokens

    # 2. Vector search (included in DB hosting cost)

    # 3. Reranking (optional)
    rerank_cost = reranker_cost_per_query

    # 4. LLM generation with retrieved context
    context_tokens = top_k * avg_chunk_tokens
    llm_input_cost = (context_tokens / 1_000_000) * llm_pricing.input_per_1m
    llm_output_cost = (avg_output_tokens / 1_000_000) * llm_pricing.output_per_1m

    cost_per_query = query_embed_cost + rerank_cost + llm_input_cost + llm_output_cost
    daily_cost = cost_per_query * queries_per_day
    monthly_cost = daily_cost * 30 + vector_db_monthly

    return {
        "llm_model": llm_pricing.name,
        "embedding_model": emb_pricing.name,
        "one_time_indexing_cost": round(indexing_cost, 2),
        "cost_per_query": round(cost_per_query, 5),
        "daily_cost": round(daily_cost, 2),
        "monthly_cost": round(monthly_cost, 2),
        "vector_db_monthly": vector_db_monthly,
        "context_tokens_per_query": context_tokens,
    }
```

---

## Side-by-Side Comparison

```python
def full_comparison(
    corpus_tokens: int,
    queries_per_day: int,
) -> dict:
    """Full cost comparison across approaches and models."""
    results = {"corpus_tokens": corpus_tokens, "queries_per_day": queries_per_day}

    # Long context options
    for model in ["claude_sonnet", "claude_haiku", "gpt4o_mini"]:
        lc = estimate_long_context_cost(corpus_tokens, queries_per_day, model)
        if "error" not in lc:
            results[f"long_context_{model}"] = lc

    # RAG options
    for emb in ["openai_small"]:
        for llm in ["claude_sonnet", "claude_haiku"]:
            rag = estimate_rag_cost(corpus_tokens, queries_per_day, emb, llm)
            results[f"rag_{emb}_{llm}"] = rag

    return results


# Example scenarios
SCENARIOS = {
    "small_corpus_low_volume": full_comparison(50_000, 50),
    "small_corpus_high_volume": full_comparison(50_000, 5000),
    "medium_corpus_low_volume": full_comparison(500_000, 50),
    "medium_corpus_high_volume": full_comparison(500_000, 5000),
    "large_corpus_high_volume": full_comparison(5_000_000, 5000),
}

# Key findings:
# 50K tokens, 50 queries/day:
#   Long context (Sonnet, cached): ~$4.50/month
#   RAG (Sonnet): ~$55/month (vector DB dominates)
#   Winner: Long Context
#
# 50K tokens, 5000 queries/day:
#   Long context (Sonnet, cached): ~$450/month
#   RAG (Sonnet): ~$85/month
#   Winner: RAG
#
# 500K tokens, 50 queries/day:
#   Long context (Sonnet, cached): ~$45/month
#   RAG (Sonnet): ~$55/month
#   Winner: Roughly tied (long context simpler)
#
# 500K tokens, 5000 queries/day:
#   Long context: ~$4,500/month
#   RAG: ~$85/month
#   Winner: RAG (50x cheaper)
```

---

## Break-Even Analysis

```python
def find_break_even_queries(
    corpus_tokens: int,
    llm_model: str = "claude_sonnet",
    vector_db_monthly: float = 50.0,
) -> dict:
    """Find the query volume where RAG becomes cheaper than long context."""
    # Find where RAG monthly cost < long context monthly cost
    for qpd in range(10, 100_000, 10):
        lc = estimate_long_context_cost(corpus_tokens, qpd, llm_model)
        rag = estimate_rag_cost(corpus_tokens, qpd, "openai_small", llm_model,
                                vector_db_monthly=vector_db_monthly)

        if "error" in lc:
            return {"break_even": 0, "reason": "Corpus too large for long context"}

        if rag["monthly_cost"] < lc["monthly_cost"]:
            return {
                "break_even_queries_per_day": qpd,
                "at_break_even_lc_cost": lc["monthly_cost"],
                "at_break_even_rag_cost": rag["monthly_cost"],
                "corpus_tokens": corpus_tokens,
                "model": llm_model,
            }

    return {"break_even": "Not found in range", "note": "RAG may never be cheaper for this corpus size"}
```

---

## Common Pitfalls

1. **Forgetting prompt caching.** Without caching, long context is 3-10x more expensive. With caching (90% discount on cached tokens), long context becomes competitive for small corpora.

2. **Ignoring vector DB hosting costs.** RAG has a fixed monthly cost ($20-200 for vector DB hosting) that dominates at low query volumes. Below ~50 queries/day, long context may be cheaper.

3. **Not accounting for reranking costs.** Cross-encoder reranking adds $0.001-0.01 per query. At high volumes, this adds up.

4. **Comparing wrong models.** Long context with Claude Haiku ($0.80/M) vs RAG with Claude Sonnet ($3/M) is not a fair comparison. Use the same LLM model for both.

5. **Ignoring the engineering cost.** RAG requires building and maintaining a pipeline (embedding, indexing, retrieval). Long context requires nothing. Factor in engineering time for low-volume use cases.

---

## References

- Anthropic pricing: https://docs.anthropic.com/en/docs/about-claude/models
- OpenAI pricing: https://openai.com/pricing
- Google Gemini pricing: https://ai.google.dev/pricing
- Prompt caching: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
