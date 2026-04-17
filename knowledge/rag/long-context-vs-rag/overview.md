# Long Context vs RAG -- Decision Framework

## TL;DR

The expansion of LLM context windows to 200K+ tokens (Claude), 1M+ tokens (Gemini), and 128K tokens (GPT-4o) raises a critical architecture question: when should you stuff all documents into the context window versus building a RAG pipeline? The answer depends on corpus size, query patterns, cost constraints, latency requirements, and accuracy needs. This overview provides a systematic decision framework with quantitative cost comparisons, identifies the sweet spots for each approach, and explains when hybrid approaches (RAG narrows, long context reads) deliver the best results.

---

## The Decision Space

```python
from dataclasses import dataclass
from enum import Enum


class Approach(Enum):
    LONG_CONTEXT = "long_context"
    RAG = "rag"
    HYBRID = "hybrid"


@dataclass
class WorkloadProfile:
    corpus_size_tokens: int
    queries_per_day: int
    corpus_update_frequency: str  # "static", "daily", "hourly", "realtime"
    query_type: str  # "factual", "analytical", "multi_hop", "summarization"
    latency_budget_ms: int
    accuracy_requirement: str  # "high", "medium", "low"
    monthly_budget_usd: float


def recommend_approach(profile: WorkloadProfile) -> dict:
    """Recommend long context, RAG, or hybrid based on workload profile."""

    # Rule 1: If corpus fits in context window, long context is simplest
    if profile.corpus_size_tokens < 100_000:
        cost = estimate_long_context_cost(
            profile.corpus_size_tokens, profile.queries_per_day
        )
        if cost["monthly_cost"] <= profile.monthly_budget_usd:
            return {
                "approach": Approach.LONG_CONTEXT,
                "reasoning": (
                    f"Corpus ({profile.corpus_size_tokens:,} tokens) fits in a single "
                    "context window. Long context is simpler and avoids retrieval errors."
                ),
                "monthly_cost": cost["monthly_cost"],
                "caveats": ["Cost scales linearly with queries/day"],
            }

    # Rule 2: If corpus is very large, RAG is necessary
    if profile.corpus_size_tokens > 1_000_000:
        return {
            "approach": Approach.RAG,
            "reasoning": (
                f"Corpus ({profile.corpus_size_tokens:,} tokens) exceeds even the largest "
                "context windows. RAG is required."
            ),
            "monthly_cost": estimate_rag_cost(
                profile.corpus_size_tokens, profile.queries_per_day
            )["monthly_cost"],
        }

    # Rule 3: For summarization queries, long context wins
    if profile.query_type == "summarization":
        return {
            "approach": Approach.LONG_CONTEXT if profile.corpus_size_tokens < 500_000 else Approach.HYBRID,
            "reasoning": (
                "Summarization requires seeing all content. RAG retrieval may "
                "miss relevant passages. Long context or hybrid is preferred."
            ),
        }

    # Rule 4: For high-volume, factual queries, RAG wins on cost
    if profile.queries_per_day > 1000 and profile.query_type == "factual":
        return {
            "approach": Approach.RAG,
            "reasoning": (
                f"At {profile.queries_per_day} queries/day, long context costs "
                "scale linearly while RAG cost per query stays low."
            ),
        }

    # Rule 5: For multi-hop queries, hybrid works best
    if profile.query_type == "multi_hop":
        return {
            "approach": Approach.HYBRID,
            "reasoning": (
                "Multi-hop queries need broad retrieval (RAG) followed by "
                "deep reading (long context) of retrieved passages."
            ),
        }

    # Default: hybrid for medium corpora
    return {
        "approach": Approach.HYBRID,
        "reasoning": "Medium corpus with mixed query types benefits from hybrid approach.",
    }
```

---

## Strengths and Weaknesses

| Dimension | Long Context | RAG |
|-----------|-------------|-----|
| Corpus size limit | 200K-1M tokens | Unlimited |
| Setup complexity | None (just prompt) | High (embed, index, retrieve) |
| Accuracy on factual | Excellent (sees everything) | Good (may miss relevant chunks) |
| Accuracy on summarization | Excellent | Poor (partial view) |
| Cost per query | High (scales with corpus) | Low (fixed retrieval cost) |
| Cost at scale | Very high | Low |
| Latency | High (slow for large context) | Low-medium |
| Freshness | Always current (no index) | Depends on ingestion lag |
| Hallucination risk | Lower (answer is in context) | Higher (retrieval may miss) |
| Multi-hop reasoning | Good | Requires iterative retrieval |
| Needle-in-haystack | Degrades with size | Consistent (retrieval-dependent) |

---

## When Long Context Wins

```python
LONG_CONTEXT_SWEET_SPOTS = [
    {
        "use_case": "Single document Q&A",
        "corpus": "One document, 10-50 pages",
        "example": "Analyzing a contract, reading a paper",
        "why": "Entire document fits. No chunking artifacts.",
    },
    {
        "use_case": "Small corpus summarization",
        "corpus": "10-50 short documents",
        "example": "Summarize all customer feedback from last week",
        "why": "Summarization needs the full picture. RAG fragments it.",
    },
    {
        "use_case": "Code review / analysis",
        "corpus": "A codebase or module (<200K tokens)",
        "example": "Review this PR, explain this module",
        "why": "Code understanding requires seeing imports, types, and flow together.",
    },
    {
        "use_case": "Low-volume, high-stakes queries",
        "corpus": "Any size fitting context window",
        "example": "Legal document review (5 queries/day)",
        "why": "Cost is acceptable, and missing a clause is not.",
    },
]
```

---

## When RAG Wins

```python
RAG_SWEET_SPOTS = [
    {
        "use_case": "Large corpus factual Q&A",
        "corpus": "1M+ tokens (1000+ documents)",
        "example": "Enterprise knowledge base, documentation",
        "why": "Corpus does not fit in any context window. RAG required.",
    },
    {
        "use_case": "High-volume queries",
        "corpus": "Any size, 1000+ queries/day",
        "example": "Customer support chatbot",
        "why": "Long context at 10K queries/day costs $100K+/month.",
    },
    {
        "use_case": "Real-time updated corpus",
        "corpus": "Documents changing hourly/daily",
        "example": "Product catalog, live documentation",
        "why": "RAG index updates incrementally. Long context needs full reload.",
    },
    {
        "use_case": "Multi-tenant applications",
        "corpus": "Different corpus per user/tenant",
        "example": "SaaS RAG product",
        "why": "RAG namespaces per tenant. Long context cannot efficiently switch.",
    },
]
```

---

## The Lost-in-the-Middle Problem

```python
def explain_lost_in_middle() -> dict:
    """The key weakness of long context for RAG-like tasks."""
    return {
        "problem": (
            "LLMs attend more to the beginning and end of the context, "
            "with reduced attention to the middle. For a 100K token context, "
            "information at position 40K-60K is most likely to be missed."
        ),
        "research": "Liu et al. 'Lost in the Middle' (2023)",
        "impact_by_context_length": {
            "10K tokens": "Minimal (<5% accuracy drop)",
            "50K tokens": "Moderate (10-15% accuracy drop in middle)",
            "100K tokens": "Significant (20-30% accuracy drop in middle)",
            "200K tokens": "Severe for needle-in-haystack tasks",
        },
        "mitigation_strategies": [
            "Place critical information at start and end of context",
            "Use RAG to pre-filter, then long context for deep reading",
            "Repeat key facts multiple times in the context",
            "Use structured context (headers, sections) for better navigation",
        ],
    }
```

---

## Common Pitfalls

1. **Assuming bigger context is always better.** The lost-in-the-middle effect means 200K tokens of context is not 10x better than 20K. Quality degrades non-linearly.

2. **Ignoring cost scaling.** Long context cost is O(queries * corpus_size). RAG cost is O(queries * top_k). At 10K queries/day, this difference is enormous.

3. **Using RAG for summarization tasks.** RAG retrieves fragments. Summarization needs the whole picture. Use long context or hybrid for summary queries.

4. **Not measuring accuracy for your specific task.** Benchmark both approaches on your actual queries. The general guidance may not apply to your domain.

5. **Forgetting about prompt caching.** With Anthropic prompt caching, the repeated corpus tokens are cached at 90% discount, making long context much more affordable for repeated queries.

---

## References

- Liu et al. "Lost in the Middle: How Language Models Use Long Contexts" (2023)
- Anthropic prompt caching: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- Gemini 1M context: https://ai.google.dev/gemini-api/docs/long-context
