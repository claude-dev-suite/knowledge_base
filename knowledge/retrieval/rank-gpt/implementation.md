# RankGPT -- Implementation Guide

## Overview

This guide covers practical implementation of LLM-based reranking using the Anthropic SDK (Claude) and OpenAI-compatible APIs. It includes the sliding window algorithm, structured output parsing, prompt engineering for ranking, and cost/latency analysis for production deployments.

---

## Basic Reranking with Claude

### Single-Window Listwise Reranking

The simplest implementation: pass all candidates in one prompt and ask for a ranking.

```python
import anthropic
import re
from dataclasses import dataclass


@dataclass
class Passage:
    id: str
    text: str
    score: float = 0.0
    metadata: dict | None = None


def rerank_with_claude(
    query: str,
    passages: list[Passage],
    model: str = "claude-sonnet-4-20250514",
    max_tokens: int = 256,
    client: anthropic.Anthropic | None = None,
) -> list[Passage]:
    """
    Rerank passages using Claude's listwise ranking capability.
    Returns passages in descending relevance order.

    Handles up to ~20 passages of ~200 words each in a single call.
    For more passages, use the sliding window approach below.
    """
    if client is None:
        client = anthropic.Anthropic()

    # Build the passage list
    passage_text = ""
    for i, p in enumerate(passages, 1):
        # Truncate long passages to ~200 words
        truncated = " ".join(p.text.split()[:200])
        passage_text += f"[{i}] {truncated}\n\n"

    n = len(passages)

    response = client.messages.create(
        model=model,
        max_tokens=max_tokens,
        system=(
            "You are a passage ranking assistant. Given a query and a "
            "numbered list of passages, rank ALL passages by relevance "
            "to the query. Output ONLY the ranking as bracket-enclosed "
            "numbers separated by ' > ', from most to least relevant. "
            "Example: [2] > [5] > [1] > [3] > [4]"
        ),
        messages=[
            {
                "role": "user",
                "content": (
                    f"Query: {query}\n\n"
                    f"Passages:\n{passage_text}"
                    f"Rank all {n} passages by relevance to the query. "
                    f"Output format: [most relevant] > ... > [least relevant]"
                ),
            },
        ],
    )

    ranking_text = response.content[0].text.strip()
    ranked_indices = parse_ranking(ranking_text, n)

    # Reorder passages according to the LLM ranking
    ranked_passages = []
    for idx in ranked_indices:
        p = passages[idx]
        ranked_passages.append(p)

    return ranked_passages


def parse_ranking(ranking_text: str, n: int) -> list[int]:
    """
    Parse a ranking string like '[3] > [1] > [5] > [2] > [4]'
    into a list of 0-based indices: [2, 0, 4, 1, 3].

    Handles various formats and gracefully fills missing indices.
    """
    # Extract all numbers from bracket-enclosed identifiers
    matches = re.findall(r"\[(\d+)\]", ranking_text)

    if not matches:
        # Fallback: try plain numbers separated by > or ,
        matches = re.findall(r"(\d+)", ranking_text)

    seen = set()
    indices = []
    for m in matches:
        idx = int(m) - 1  # Convert 1-based to 0-based
        if 0 <= idx < n and idx not in seen:
            indices.append(idx)
            seen.add(idx)

    # Fill in any missing indices at the end (in original order)
    for i in range(n):
        if i not in seen:
            indices.append(i)

    return indices
```

### Structured Output for Reliable Parsing

Using Claude's tool_use for structured ranking output:

```python
def rerank_structured(
    query: str,
    passages: list[Passage],
    model: str = "claude-sonnet-4-20250514",
    client: anthropic.Anthropic | None = None,
) -> list[Passage]:
    """
    Rerank using Claude's tool_use for structured JSON output.
    More reliable parsing than free-text ranking strings.
    """
    if client is None:
        client = anthropic.Anthropic()

    n = len(passages)
    passage_text = ""
    for i, p in enumerate(passages, 1):
        truncated = " ".join(p.text.split()[:200])
        passage_text += f"[{i}] {truncated}\n\n"

    # Define a tool that accepts the ranking
    tools = [
        {
            "name": "submit_ranking",
            "description": (
                "Submit the passage ranking from most relevant to least relevant."
            ),
            "input_schema": {
                "type": "object",
                "properties": {
                    "ranking": {
                        "type": "array",
                        "items": {"type": "integer"},
                        "description": (
                            "Ordered list of passage numbers (1-indexed) from "
                            "most relevant to least relevant. Must contain all "
                            f"numbers from 1 to {n} exactly once."
                        ),
                    },
                    "reasoning": {
                        "type": "string",
                        "description": "Brief explanation of the ranking rationale.",
                    },
                },
                "required": ["ranking"],
            },
        },
    ]

    response = client.messages.create(
        model=model,
        max_tokens=1024,
        system=(
            f"Rank the {n} passages by relevance to the query. "
            "Use the submit_ranking tool to provide your ranking."
        ),
        tools=tools,
        tool_choice={"type": "tool", "name": "submit_ranking"},
        messages=[
            {
                "role": "user",
                "content": f"Query: {query}\n\nPassages:\n{passage_text}",
            },
        ],
    )

    # Extract the tool use result
    for block in response.content:
        if block.type == "tool_use" and block.name == "submit_ranking":
            ranking = block.input.get("ranking", [])
            # Convert 1-indexed to 0-indexed
            indices = [r - 1 for r in ranking if 1 <= r <= n]

            # Fill missing
            seen = set(indices)
            for i in range(n):
                if i not in seen:
                    indices.append(i)

            return [passages[i] for i in indices]

    # Fallback: return original order
    return passages
```

---

## Sliding Window Algorithm

When the candidate list is too large for a single context window, use the sliding window (bubble sort) approach from the RankGPT paper.

```python
def rerank_sliding_window(
    query: str,
    passages: list[Passage],
    window_size: int = 20,
    step_size: int = 10,
    model: str = "claude-sonnet-4-20250514",
    client: anthropic.Anthropic | None = None,
) -> list[Passage]:
    """
    Rerank a large list of passages using a sliding window approach.

    The algorithm processes overlapping windows from the bottom of the
    initial ranking to the top. Relevant documents "bubble up" through
    successive windows.

    Parameters:
    - window_size: number of passages per LLM call (default: 20)
    - step_size: how far the window moves each step (default: 10)
      A smaller step_size gives more overlap and better quality but costs more.

    Example with 50 passages, window=20, step=10:
      Step 1: rank passages [30:50]  -> reorder positions 30-49
      Step 2: rank passages [20:40]  -> top from step 1 can bubble up
      Step 3: rank passages [10:30]  -> continued bubbling
      Step 4: rank passages [0:20]   -> final top positions settled

    Total LLM calls: ceil((50 - 20) / 10) + 1 = 4
    """
    if client is None:
        client = anthropic.Anthropic()

    if len(passages) <= window_size:
        return rerank_with_claude(query, passages, model=model, client=client)

    # Work with a mutable copy
    current = list(passages)
    n = len(current)

    # Calculate window positions (from bottom to top)
    # Start from the end of the list and slide toward the beginning
    start_positions = []
    pos = n - window_size
    while pos >= 0:
        start_positions.append(pos)
        pos -= step_size
    if start_positions[-1] != 0:
        start_positions.append(0)

    total_calls = len(start_positions)
    print(f"Sliding window: {n} passages, {total_calls} LLM calls "
          f"(window={window_size}, step={step_size})")

    for call_num, start in enumerate(start_positions, 1):
        end = min(start + window_size, n)
        window = current[start:end]

        print(f"  Call {call_num}/{total_calls}: ranking positions [{start}:{end}]")

        # Rerank the window
        ranked_window = rerank_with_claude(
            query, window, model=model, client=client,
        )

        # Place the reranked window back
        current[start:end] = ranked_window

    return current


def estimate_sliding_window_cost(
    num_passages: int,
    avg_passage_words: int = 150,
    window_size: int = 20,
    step_size: int = 10,
    model: str = "claude-sonnet-4-20250514",
) -> dict:
    """
    Estimate the cost and latency of sliding window reranking.
    """
    import math

    num_calls = math.ceil((num_passages - window_size) / step_size) + 1
    if num_passages <= window_size:
        num_calls = 1

    tokens_per_passage = int(avg_passage_words * 1.3)  # ~1.3 tokens per word
    input_tokens_per_call = (
        100  # system prompt
        + 50  # query + formatting
        + window_size * tokens_per_passage
    )
    output_tokens_per_call = 100  # ranking string

    total_input = num_calls * input_tokens_per_call
    total_output = num_calls * output_tokens_per_call

    # Pricing (approximate, per million tokens)
    pricing = {
        "claude-sonnet-4-20250514": {"input": 3.0, "output": 15.0},
        "claude-haiku-3-20250414": {"input": 0.25, "output": 1.25},
        "claude-opus-4-20250514": {"input": 15.0, "output": 75.0},
    }

    prices = pricing.get(model, pricing["claude-sonnet-4-20250514"])
    cost = (
        total_input / 1_000_000 * prices["input"]
        + total_output / 1_000_000 * prices["output"]
    )

    # Latency estimate (sequential calls)
    latency_per_call_s = 1.5 if "sonnet" in model else (0.5 if "haiku" in model else 3.0)
    total_latency_s = num_calls * latency_per_call_s

    return {
        "num_llm_calls": num_calls,
        "total_input_tokens": total_input,
        "total_output_tokens": total_output,
        "estimated_cost_usd": round(cost, 4),
        "estimated_latency_seconds": round(total_latency_s, 1),
        "model": model,
    }
```

---

## Batch Reranking Pipeline

For processing multiple queries efficiently:

```python
import asyncio
from typing import AsyncIterator


async def rerank_batch_async(
    queries_and_passages: list[tuple[str, list[Passage]]],
    model: str = "claude-sonnet-4-20250514",
    max_concurrent: int = 5,
    window_size: int = 20,
) -> list[list[Passage]]:
    """
    Rerank multiple query-passage lists concurrently.
    Uses asyncio with a semaphore to limit concurrent API calls.
    """
    client = anthropic.AsyncAnthropic()
    semaphore = asyncio.Semaphore(max_concurrent)
    results = [None] * len(queries_and_passages)

    async def process_one(idx: int, query: str, passages: list[Passage]):
        async with semaphore:
            ranked = await _async_rerank(
                client, query, passages, model, window_size,
            )
            results[idx] = ranked

    tasks = [
        process_one(i, query, passages)
        for i, (query, passages) in enumerate(queries_and_passages)
    ]
    await asyncio.gather(*tasks)

    return results


async def _async_rerank(
    client: anthropic.AsyncAnthropic,
    query: str,
    passages: list[Passage],
    model: str,
    window_size: int,
) -> list[Passage]:
    """Single query async reranking."""
    n = len(passages)
    passage_text = ""
    for i, p in enumerate(passages, 1):
        truncated = " ".join(p.text.split()[:200])
        passage_text += f"[{i}] {truncated}\n\n"

    response = await client.messages.create(
        model=model,
        max_tokens=256,
        system=(
            "You are a passage ranking assistant. Rank ALL passages by "
            "relevance to the query. Output ONLY bracket-enclosed numbers "
            "separated by ' > ', most to least relevant."
        ),
        messages=[
            {
                "role": "user",
                "content": (
                    f"Query: {query}\n\nPassages:\n{passage_text}"
                    f"Rank all {n} passages."
                ),
            },
        ],
    )

    ranking_text = response.content[0].text.strip()
    indices = parse_ranking(ranking_text, n)
    return [passages[i] for i in indices]
```

---

## Production-Grade Reranker Class

A complete reranker with retries, caching, and telemetry:

```python
import hashlib
import json
import time
from functools import lru_cache
from typing import Literal

import anthropic


class RankGPTReranker:
    """
    Production-grade LLM reranker using Claude.

    Features:
    - Automatic sliding window for large candidate lists
    - Structured output parsing with fallback
    - Response caching (optional)
    - Cost and latency tracking
    - Retry with exponential backoff
    """

    def __init__(
        self,
        model: str = "claude-sonnet-4-20250514",
        window_size: int = 20,
        step_size: int = 10,
        max_passage_words: int = 200,
        max_retries: int = 3,
        enable_cache: bool = True,
        api_key: str | None = None,
    ):
        self.model = model
        self.window_size = window_size
        self.step_size = step_size
        self.max_passage_words = max_passage_words
        self.max_retries = max_retries
        self.enable_cache = enable_cache

        kwargs = {}
        if api_key:
            kwargs["api_key"] = api_key
        self.client = anthropic.Anthropic(**kwargs)

        # Telemetry
        self.total_queries = 0
        self.total_llm_calls = 0
        self.total_input_tokens = 0
        self.total_output_tokens = 0
        self.total_latency_ms = 0
        self._cache: dict[str, list[int]] = {}

    def rerank(
        self,
        query: str,
        passages: list[Passage],
        top_k: int | None = None,
    ) -> list[Passage]:
        """
        Rerank passages by relevance to the query.
        Returns top_k passages in descending relevance order.
        """
        self.total_queries += 1
        start = time.monotonic()

        if len(passages) <= self.window_size:
            ranked = self._rerank_single_window(query, passages)
        else:
            ranked = self._rerank_sliding_window(query, passages)

        elapsed_ms = (time.monotonic() - start) * 1000
        self.total_latency_ms += elapsed_ms

        if top_k:
            ranked = ranked[:top_k]

        return ranked

    def _rerank_single_window(
        self,
        query: str,
        passages: list[Passage],
    ) -> list[Passage]:
        """Rerank a small list in a single LLM call."""
        indices = self._call_llm(query, passages)
        return [passages[i] for i in indices]

    def _rerank_sliding_window(
        self,
        query: str,
        passages: list[Passage],
    ) -> list[Passage]:
        """Rerank a large list using sliding windows."""
        current = list(passages)
        n = len(current)

        pos = n - self.window_size
        while pos >= 0:
            end = min(pos + self.window_size, n)
            window = current[pos:end]

            indices = self._call_llm(query, window)
            ranked_window = [window[i] for i in indices]
            current[pos:end] = ranked_window

            pos -= self.step_size

        # Final window from position 0
        if pos + self.step_size > 0:
            end = min(self.window_size, n)
            window = current[0:end]
            indices = self._call_llm(query, window)
            ranked_window = [window[i] for i in indices]
            current[0:end] = ranked_window

        return current

    def _call_llm(
        self,
        query: str,
        passages: list[Passage],
    ) -> list[int]:
        """
        Make a single LLM call to rank passages.
        Includes caching, retries, and telemetry.
        """
        n = len(passages)

        # Check cache
        if self.enable_cache:
            cache_key = self._cache_key(query, passages)
            if cache_key in self._cache:
                return self._cache[cache_key]

        passage_text = self._format_passages(passages)

        for attempt in range(self.max_retries):
            try:
                response = self.client.messages.create(
                    model=self.model,
                    max_tokens=256,
                    system=(
                        "You are a passage ranking assistant. Rank ALL passages "
                        "by relevance to the query. Output ONLY the ranking as "
                        "bracket-enclosed numbers separated by ' > '. "
                        "Example: [2] > [5] > [1] > [3] > [4]"
                    ),
                    messages=[
                        {
                            "role": "user",
                            "content": (
                                f"Query: {query}\n\n"
                                f"Passages:\n{passage_text}"
                                f"Rank all {n} passages by relevance."
                            ),
                        },
                    ],
                )

                # Track tokens
                self.total_llm_calls += 1
                self.total_input_tokens += response.usage.input_tokens
                self.total_output_tokens += response.usage.output_tokens

                ranking_text = response.content[0].text.strip()
                indices = parse_ranking(ranking_text, n)

                # Cache the result
                if self.enable_cache:
                    self._cache[cache_key] = indices

                return indices

            except anthropic.RateLimitError:
                wait = 2 ** attempt
                time.sleep(wait)
            except anthropic.APIError as e:
                if attempt == self.max_retries - 1:
                    # Return original order on final failure
                    return list(range(n))
                time.sleep(1)

        return list(range(n))

    def _format_passages(self, passages: list[Passage]) -> str:
        """Format passages for the prompt, truncating to max_passage_words."""
        parts = []
        for i, p in enumerate(passages, 1):
            words = p.text.split()
            truncated = " ".join(words[: self.max_passage_words])
            parts.append(f"[{i}] {truncated}\n")
        return "\n".join(parts)

    def _cache_key(self, query: str, passages: list[Passage]) -> str:
        """Generate a deterministic cache key."""
        content = query + "|" + "|".join(p.id for p in passages)
        return hashlib.sha256(content.encode()).hexdigest()[:16]

    def get_stats(self) -> dict:
        """Return cumulative telemetry statistics."""
        avg_latency = (
            self.total_latency_ms / self.total_queries
            if self.total_queries > 0
            else 0
        )
        return {
            "total_queries": self.total_queries,
            "total_llm_calls": self.total_llm_calls,
            "total_input_tokens": self.total_input_tokens,
            "total_output_tokens": self.total_output_tokens,
            "avg_latency_ms": round(avg_latency, 1),
            "cache_size": len(self._cache),
        }
```

---

## Cost and Latency Analysis

### Per-Query Cost Breakdown

```python
def analyze_reranking_cost(
    num_passages: int = 20,
    avg_passage_words: int = 150,
    queries_per_day: int = 1000,
) -> None:
    """
    Print a cost and latency analysis for different models.
    """
    tokens_per_passage = int(avg_passage_words * 1.3)
    system_tokens = 100
    query_tokens = 50
    input_tokens = system_tokens + query_tokens + num_passages * tokens_per_passage
    output_tokens = 80  # ranking output

    models = {
        "claude-haiku-3-20250414": {
            "input_per_mtok": 0.25,
            "output_per_mtok": 1.25,
            "latency_ms": 500,
        },
        "claude-sonnet-4-20250514": {
            "input_per_mtok": 3.0,
            "output_per_mtok": 15.0,
            "latency_ms": 1500,
        },
        "claude-opus-4-20250514": {
            "input_per_mtok": 15.0,
            "output_per_mtok": 75.0,
            "latency_ms": 3000,
        },
    }

    print(f"Configuration: {num_passages} passages, ~{avg_passage_words} words each")
    print(f"Input tokens per query: ~{input_tokens}")
    print(f"Queries per day: {queries_per_day}")
    print()

    for model_name, specs in models.items():
        cost_per_query = (
            input_tokens / 1_000_000 * specs["input_per_mtok"]
            + output_tokens / 1_000_000 * specs["output_per_mtok"]
        )
        daily_cost = cost_per_query * queries_per_day
        monthly_cost = daily_cost * 30

        print(f"{model_name}:")
        print(f"  Cost per query:  ${cost_per_query:.4f}")
        print(f"  Daily cost:      ${daily_cost:.2f}")
        print(f"  Monthly cost:    ${monthly_cost:.2f}")
        print(f"  Latency:         ~{specs['latency_ms']}ms")
        print(f"  Max QPS:         ~{1000 / specs['latency_ms']:.1f} (single thread)")
        print()


# Example output:
# Configuration: 20 passages, ~150 words each
# Input tokens per query: ~4050
# Queries per day: 1000
#
# claude-haiku-3-20250414:
#   Cost per query:  $0.0011
#   Daily cost:      $1.11
#   Monthly cost:    $33.30
#   Latency:         ~500ms
#   Max QPS:         ~2.0 (single thread)
#
# claude-sonnet-4-20250514:
#   Cost per query:  $0.0133
#   Daily cost:      $13.35
#   Monthly cost:    $400.50
#   Latency:         ~1500ms
#   Max QPS:         ~0.7 (single thread)
```

### Cost Optimization Strategies

```python
def optimize_reranking_cost(
    passages: list[Passage],
    query: str,
    budget_per_query_usd: float = 0.005,
) -> dict:
    """
    Recommend cost optimization strategies based on budget.
    """
    strategies = []

    # Strategy 1: Reduce candidate count with pre-filtering
    if len(passages) > 20:
        strategies.append({
            "strategy": "Pre-filter with BM25/dense score threshold",
            "detail": f"Reduce from {len(passages)} to 15-20 candidates",
            "cost_reduction": f"~{(1 - 20/len(passages))*100:.0f}%",
        })

    # Strategy 2: Use cheaper model for easy cases
    strategies.append({
        "strategy": "Two-tier model selection",
        "detail": (
            "Use Haiku for queries where BM25 top-1 score > threshold "
            "(likely easy), Sonnet for harder queries"
        ),
        "cost_reduction": "~60-70% (if 80% of queries are easy)",
    })

    # Strategy 3: Truncate passages more aggressively
    avg_words = sum(len(p.text.split()) for p in passages) / len(passages)
    if avg_words > 100:
        strategies.append({
            "strategy": "Truncate passages to first 100 words",
            "detail": f"Current avg: {avg_words:.0f} words -> 100 words",
            "cost_reduction": f"~{(1 - 100/avg_words)*50:.0f}% of input tokens",
        })

    # Strategy 4: Cache repeated queries
    strategies.append({
        "strategy": "Cache rankings for repeated/similar queries",
        "detail": "Hash query + passage IDs for cache key",
        "cost_reduction": "Varies (20-80% for search with query clusters)",
    })

    # Strategy 5: Distill to a smaller model
    strategies.append({
        "strategy": "Distill LLM rankings to a fine-tuned cross-encoder",
        "detail": "Generate 10K ranking examples, fine-tune MiniLM or DistilBERT",
        "cost_reduction": "~99% (one-time cost, then free inference)",
    })

    return {
        "current_passages": len(passages),
        "budget_per_query": budget_per_query_usd,
        "strategies": strategies,
    }
```

---

## Integration with Retrieval Pipelines

### RAG Pipeline with RankGPT

```python
def rag_with_reranking(
    query: str,
    retriever,  # Any first-stage retriever
    reranker: RankGPTReranker,
    generator_client: anthropic.Anthropic,
    retrieval_top_k: int = 50,
    rerank_top_k: int = 5,
    model: str = "claude-sonnet-4-20250514",
) -> str:
    """
    Full RAG pipeline: retrieve -> rerank -> generate.
    """
    # Step 1: First-stage retrieval
    candidates = retriever.search(query, k=retrieval_top_k)
    passages = [
        Passage(id=str(c["id"]), text=c["text"], score=c["score"])
        for c in candidates
    ]

    # Step 2: LLM reranking
    ranked = reranker.rerank(query, passages, top_k=rerank_top_k)

    # Step 3: Generate answer from top passages
    context = "\n\n---\n\n".join(
        f"Source {i+1}: {p.text}" for i, p in enumerate(ranked)
    )

    response = generator_client.messages.create(
        model=model,
        max_tokens=1024,
        messages=[
            {
                "role": "user",
                "content": (
                    f"Answer the following question using ONLY the provided sources. "
                    f"If the sources do not contain enough information, say so.\n\n"
                    f"Question: {query}\n\n"
                    f"Sources:\n{context}"
                ),
            },
        ],
    )

    return response.content[0].text
```

---

## Common Pitfalls

1. **Not handling API rate limits**: production reranking can easily hit rate limits. Always implement exponential backoff and concurrency limits.

2. **Ignoring output validation**: the LLM may return incomplete or malformed rankings. Always validate that the output contains all expected passage IDs and fill in missing ones from the original order.

3. **Using the wrong model tier**: Haiku handles simple reranking well for ~10x less cost than Sonnet. Only use Sonnet/Opus for complex queries where Haiku underperforms.

4. **Not truncating passages**: sending full 1000-word passages wastes tokens and does not improve ranking quality. 150-200 words captures the essential content.

5. **Sequential processing when async is possible**: use asyncio for batch reranking to process multiple queries in parallel within rate limits.

6. **Not caching**: many search systems see repeated or very similar queries. A simple hash-based cache can eliminate 20-80% of LLM calls.

---

## References

- Anthropic Claude API documentation: https://docs.anthropic.com/en/api
- Sun, W. et al. "Is ChatGPT Good at Search? Investigating Large Language Models as Re-Ranking Agents." arXiv 2023.
- Pradeep, R. et al. "RankVicuna: Zero-Shot Listwise Document Reranking with Open-Source Large Language Models." arXiv 2023.
