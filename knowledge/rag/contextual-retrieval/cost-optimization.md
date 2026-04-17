# Contextual Retrieval: Cost Optimization

## Overview / TL;DR

Contextual retrieval requires an LLM call for every chunk during ingestion, which can be expensive at scale. The primary cost driver is sending the full document text with every chunk's prompt. Anthropic's prompt caching reduces this cost by up to 90%, making the technique economically viable even for million-document corpora. This document covers the exact cost math, prompt caching mechanics, batch processing strategies, and decision criteria for when the additional ingestion cost is justified by retrieval quality gains.

---

## Cost Breakdown Without Optimization

### Per-Chunk Cost (No Caching)

Each chunk requires sending the full document + the chunk to Claude Haiku:

```
Input tokens per call = document_tokens + chunk_tokens + prompt_tokens
                      = D + C + ~100 (prompt template overhead)

Output tokens per call = ~80 (typical context preamble length)

Cost per call (Haiku) = (D + C + 100) * $0.80/M + 80 * $4.00/M
```

For a typical 10,000-token document with 20 chunks of 512 tokens each:

```
Cost per chunk  = (10,000 + 512 + 100) * $0.80/1M + 80 * $4.00/1M
                = $0.00849 + $0.00032
                = $0.00881

Cost per document = 20 * $0.00881 = $0.176

Cost for 1,000 documents = $176
Cost for 10,000 documents = $1,760
Cost for 100,000 documents = $17,600
```

This is prohibitively expensive for large corpora.

---

## Prompt Caching: How It Works

Anthropic's prompt caching allows you to mark portions of the prompt for caching. When the same cached content appears in subsequent requests, you pay a reduced rate.

### Pricing Tiers (Claude Haiku)

| Token Category | Price per M tokens |
|---------------|-------------------|
| Normal input tokens | $0.80 |
| Cache write tokens (first request) | $1.00 (25% premium) |
| Cache read tokens (subsequent requests) | $0.08 (90% discount) |
| Output tokens | $4.00 |

### Cache Behavior

1. **First chunk from a document**: Full document is sent and cached. You pay the cache write premium ($1.00/M instead of $0.80/M) on the document tokens.
2. **Subsequent chunks from the same document**: Document tokens hit the cache. You pay the cache read rate ($0.08/M) on those tokens.
3. **Cache TTL**: 5 minutes. All chunks from a document must be processed within this window.
4. **Minimum cacheable size**: 1,024 tokens for Claude Haiku, 2,048 for Sonnet/Opus.

---

## Cost with Prompt Caching

### Per-Document Cost (With Caching)

For a 10,000-token document with 20 chunks:

```
Chunk 1 (cache write):
  Input = 10,000 (cache write) + 612 (chunk + template) = 10,612 tokens
  Cost  = 10,000 * $1.00/M + 612 * $0.80/M + 80 * $4.00/M
        = $0.01000 + $0.00049 + $0.00032
        = $0.01081

Chunks 2-20 (cache read, 19 chunks):
  Input = 10,000 (cache read) + 612 (chunk + template) = 10,612 tokens per chunk
  Cost per chunk = 10,000 * $0.08/M + 612 * $0.80/M + 80 * $4.00/M
                 = $0.00080 + $0.00049 + $0.00032
                 = $0.00161

Total for document = $0.01081 + 19 * $0.00161 = $0.04140

Savings vs no caching: $0.176 -> $0.041 = 76.7% savings
```

### Scaling to Corpus Size

| Corpus Size | Avg Doc Size | Chunks/Doc | No Caching | With Caching | Savings |
|------------|-------------|-----------|-----------|-------------|---------|
| 100 docs | 10K tokens | 20 | $17.60 | $4.14 | 76% |
| 1,000 docs | 10K tokens | 20 | $176 | $41.40 | 76% |
| 10,000 docs | 10K tokens | 20 | $1,760 | $414 | 76% |
| 100,000 docs | 10K tokens | 20 | $17,600 | $4,140 | 76% |
| 1,000 docs | 50K tokens | 100 | $4,280 | $221 | 94.8% |
| 10,000 docs | 2K tokens | 4 | $352 | $159 | 54.8% |

**Key insight**: The savings percentage increases with document size and chunks per document. Longer documents amortize the cache write cost over more chunks.

### Break-Even: Minimum Chunks Per Document

Caching has a 25% write premium on the first call. It pays off when:

```
cache_write_premium < (N-1) * cache_read_savings

0.25 * D < (N-1) * 0.90 * D
(where D = document tokens, N = number of chunks)

0.25 < (N-1) * 0.90
N > 1.28

Caching is beneficial when a document has 2 or more chunks.
```

For single-chunk documents, caching is slightly more expensive (25% premium with no subsequent savings). Skip caching for very short documents.

---

## Implementation: Optimal Batch Processing

```python
import anthropic
import asyncio
import logging
from dataclasses import dataclass, field
from typing import AsyncIterator

logger = logging.getLogger(__name__)


@dataclass
class IngestionStats:
    documents_processed: int = 0
    chunks_processed: int = 0
    total_input_tokens: int = 0
    total_output_tokens: int = 0
    cache_write_tokens: int = 0
    cache_read_tokens: int = 0
    estimated_cost: float = 0.0

    def update_from_response(self, response):
        usage = response.usage
        self.total_input_tokens += usage.input_tokens
        self.total_output_tokens += usage.output_tokens
        if hasattr(usage, "cache_creation_input_tokens"):
            self.cache_write_tokens += usage.cache_creation_input_tokens
        if hasattr(usage, "cache_read_input_tokens"):
            self.cache_read_tokens += usage.cache_read_input_tokens

    @property
    def cache_hit_rate(self) -> float:
        total = self.cache_write_tokens + self.cache_read_tokens
        if total == 0:
            return 0.0
        return self.cache_read_tokens / total

    def summary(self) -> str:
        return (
            f"Documents: {self.documents_processed}, "
            f"Chunks: {self.chunks_processed}, "
            f"Cache hit rate: {self.cache_hit_rate:.1%}, "
            f"Input tokens: {self.total_input_tokens:,}, "
            f"Output tokens: {self.total_output_tokens:,}, "
            f"Est. cost: ${self.estimated_cost:.2f}"
        )


class CostOptimizedContextualizer:
    """Contextualize chunks with maximum cost efficiency."""

    def __init__(
        self,
        model: str = "claude-haiku-4-20250514",
        max_concurrent: int = 10,
        min_doc_tokens_for_caching: int = 1024,
    ):
        self.client = anthropic.AsyncAnthropic()
        self.model = model
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.min_doc_tokens_for_caching = min_doc_tokens_for_caching
        self.stats = IngestionStats()

    async def process_corpus(
        self,
        documents: list[dict],
    ) -> AsyncIterator[list[dict]]:
        """Process entire corpus, yielding contextualized chunks per document.

        Documents are sorted by size (largest first) to maximize cache value.
        """
        # Sort by estimated token count (descending)
        sorted_docs = sorted(
            documents,
            key=lambda d: len(d["text"]),
            reverse=True,
        )

        for doc in sorted_docs:
            chunks = doc["chunks"]
            if not chunks:
                continue

            doc_tokens = len(doc["text"].split()) * 1.3  # Rough token estimate

            # Skip caching for very short documents (cache write premium not worth it)
            use_caching = doc_tokens >= self.min_doc_tokens_for_caching and len(chunks) >= 2

            results = await self._process_document(
                doc["text"], chunks, use_caching
            )
            self.stats.documents_processed += 1
            self.stats.chunks_processed += len(results)

            yield results

    async def _process_document(
        self,
        document_text: str,
        chunks: list[dict],
        use_caching: bool,
    ) -> list[dict]:
        """Process all chunks from a single document."""
        tasks = [
            self._process_chunk(document_text, chunk, use_caching)
            for chunk in chunks
        ]
        results = await asyncio.gather(*tasks, return_exceptions=True)

        # Handle any failures gracefully
        processed = []
        for i, result in enumerate(results):
            if isinstance(result, Exception):
                logger.warning(f"Failed to contextualize chunk {i}: {result}")
                # Fall back to original text without context
                processed.append({
                    "text": chunks[i]["text"],
                    "original": chunks[i]["text"],
                    "context": "",
                    "metadata": chunks[i].get("metadata", {}),
                })
            else:
                processed.append(result)

        return processed

    async def _process_chunk(
        self,
        document_text: str,
        chunk: dict,
        use_caching: bool,
    ) -> dict:
        """Process a single chunk with optional caching."""
        async with self.semaphore:
            if use_caching:
                response = await self.client.messages.create(
                    model=self.model,
                    max_tokens=200,
                    temperature=0,
                    system=[{
                        "type": "text",
                        "text": f"<document>\n{document_text}\n</document>",
                        "cache_control": {"type": "ephemeral"},
                    }],
                    messages=[{
                        "role": "user",
                        "content": (
                            f"Here is the chunk we want to situate within the "
                            f"whole document:\n<chunk>\n{chunk['text']}\n</chunk>\n"
                            f"Please give a short succinct context to situate "
                            f"this chunk within the overall document for the "
                            f"purposes of improving search retrieval of the chunk. "
                            f"Answer only with the succinct context and nothing else."
                        ),
                    }],
                )
            else:
                response = await self.client.messages.create(
                    model=self.model,
                    max_tokens=200,
                    temperature=0,
                    messages=[{
                        "role": "user",
                        "content": CONTEXTUAL_RETRIEVAL_PROMPT.format(
                            WHOLE_DOCUMENT=document_text,
                            CHUNK_CONTENT=chunk["text"],
                        ),
                    }],
                )

            self.stats.update_from_response(response)
            context = response.content[0].text.strip()

            return {
                "text": f"{context}\n\n{chunk['text']}",
                "original": chunk["text"],
                "context": context,
                "metadata": chunk.get("metadata", {}),
            }
```

---

## When Is the Extra Cost Worth It?

### Decision Framework

```
Is your retrieval failure rate unacceptable?
  |
  NO --> Don't add contextual retrieval. Focus on other improvements first.
  |
  YES --> Have you already implemented hybrid search + reranking?
          |
          NO --> Implement those first. They're cheaper and may solve the problem.
          |
          YES --> Are your chunks self-contained (FAQ, product cards, etc.)?
                  |
                  YES --> Contextual retrieval won't help much. Chunks already
                          contain their own context.
                  |
                  NO --> Are chunks from long documents missing key context?
                         |
                         YES --> Contextual retrieval is a good fit.
                         |       Calculate the ingestion cost and compare to
                         |       the business value of improved accuracy.
                         |
                         NO --> Investigate other issues (chunking strategy,
                                embedding model, query transformation).
```

### ROI Calculation

```python
def calculate_contextual_retrieval_roi(
    num_documents: int,
    avg_doc_tokens: int,
    avg_chunks_per_doc: int,
    queries_per_month: int,
    current_failure_rate: float,
    cost_per_failed_query: float,  # Business cost of a wrong answer
) -> dict:
    """Calculate whether contextual retrieval is worth the investment."""

    # Ingestion cost (one-time, with caching)
    # First chunk per doc: cache write premium
    first_chunk_cost = num_documents * (
        avg_doc_tokens * 1.00 / 1_000_000  # cache write
        + 600 * 0.80 / 1_000_000           # chunk + template
        + 80 * 4.00 / 1_000_000            # output
    )
    # Subsequent chunks: cache read discount
    subsequent_chunks = num_documents * (avg_chunks_per_doc - 1)
    subsequent_cost = subsequent_chunks * (
        avg_doc_tokens * 0.08 / 1_000_000  # cache read
        + 600 * 0.80 / 1_000_000           # chunk + template
        + 80 * 4.00 / 1_000_000            # output
    )
    total_ingestion_cost = first_chunk_cost + subsequent_cost

    # Expected improvement (based on Anthropic's published numbers)
    # Contextual embeddings + BM25 + reranking: 67% failure rate reduction
    new_failure_rate = current_failure_rate * 0.33
    failure_reduction = current_failure_rate - new_failure_rate

    # Monthly savings from fewer failed queries
    monthly_savings = queries_per_month * failure_reduction * cost_per_failed_query

    # Payback period
    payback_months = total_ingestion_cost / monthly_savings if monthly_savings > 0 else float("inf")

    return {
        "ingestion_cost": total_ingestion_cost,
        "current_failure_rate": current_failure_rate,
        "expected_failure_rate": new_failure_rate,
        "monthly_savings": monthly_savings,
        "payback_months": payback_months,
    }


# Example: customer support bot
roi = calculate_contextual_retrieval_roi(
    num_documents=5000,
    avg_doc_tokens=8000,
    avg_chunks_per_doc=16,
    queries_per_month=50000,
    current_failure_rate=0.20,   # 20% of queries get wrong answers
    cost_per_failed_query=2.00,  # Support escalation cost
)
# ingestion_cost: ~$150
# monthly_savings: ~$13,400
# payback_months: ~0.01 (pays for itself almost immediately)
```

---

## Diminishing Returns Analysis

The three layers of contextual retrieval have decreasing marginal returns:

| Layer Added | Cumulative Failure Rate Reduction | Marginal Improvement | Marginal Cost |
|------------|----------------------------------|---------------------|---------------|
| Contextual Embeddings | 35% | 35% | LLM ingestion cost (major) |
| + Contextual BM25 | 49% | 14% | BM25 index (negligible) |
| + Reranking | 67% | 18% | Reranker calls (minor) |

**Implication**: If cost is a hard constraint:

1. **Minimum viable**: Contextual embeddings only. 35% improvement for the full ingestion cost.
2. **Best value**: Add BM25 (essentially free). 49% improvement for no additional cost.
3. **Maximum quality**: Add reranking. 67% improvement for a small per-query cost.

The decision to skip any layer should never be about BM25 (it is free) or reranking (it is cheap per-query). The only real cost decision is whether to do contextual retrieval at all (the ingestion cost).

---

## Cost-Saving Alternatives

If the ingestion cost of contextual retrieval is prohibitive, consider these cheaper alternatives that capture some of the same benefit:

### Alternative 1: Metadata Prepending (Free)

Instead of generating context with an LLM, prepend the document title and section headers mechanically.

```python
def cheap_contextualize(chunk_text: str, metadata: dict) -> str:
    """Free alternative: prepend metadata as context."""
    parts = []
    if metadata.get("title"):
        parts.append(f"Document: {metadata['title']}")
    if metadata.get("h1"):
        parts.append(f"Section: {metadata['h1']}")
    if metadata.get("h2"):
        parts.append(f"Subsection: {metadata['h2']}")
    context = ". ".join(parts)
    return f"{context}\n\n{chunk_text}" if context else chunk_text
```

**Effectiveness**: Captures ~40-60% of the benefit of full contextual retrieval for structured documents (those with good headings). Much less effective for unstructured documents.

### Alternative 2: Document Summary Prepending (Cheap)

Generate a single summary per document (not per chunk) and prepend it to all chunks.

```python
async def summarize_once_prepend_many(
    document_text: str,
    chunks: list[dict],
) -> list[dict]:
    """Generate one summary per document, prepend to all chunks.
    Cost: 1 LLM call per document instead of N calls per chunk.
    """
    response = await client.messages.create(
        model="claude-haiku-4-20250514",
        max_tokens=150,
        messages=[{
            "role": "user",
            "content": (
                f"Summarize this document in 2-3 sentences, focusing on "
                f"the main topic and key entities mentioned:\n\n{document_text[:5000]}"
            ),
        }],
    )
    summary = response.content[0].text.strip()

    return [
        {
            "text": f"{summary}\n\n{chunk['text']}",
            "original": chunk["text"],
            "context": summary,
            "metadata": chunk.get("metadata", {}),
        }
        for chunk in chunks
    ]
```

**Cost**: 1/N of full contextual retrieval (where N = chunks per document). For a 20-chunk document, this is 95% cheaper.

**Effectiveness**: Captures ~50-70% of the benefit. All chunks get the same context, so chunk-specific keywords are not added. But the document-level context (topic, entities) is present.

---

## Common Pitfalls

1. **Not monitoring cache hit rate.** If your cache hit rate is below 70%, something is wrong with your processing order. Check that you are processing all chunks from one document before moving to the next.
2. **Using Sonnet instead of Haiku for context generation.** Haiku is 3-4x cheaper and produces equally good context preambles for this task. The context only needs to be a few sentences.
3. **Contextualizing single-chunk documents with caching enabled.** The cache write premium (25%) makes caching counterproductive for documents with only one chunk.
4. **Not accounting for re-contextualization costs.** When you re-chunk (change chunk size or strategy), you must re-generate all contexts. Factor this into the total cost of experimentation.
5. **Over-generating context.** Set `max_tokens=200` and `temperature=0`. Longer contexts do not help and increase both ingestion cost and embedding noise.
6. **Ignoring the free alternatives.** For structured documents with good headings, metadata prepending captures much of the benefit at zero cost. Try it first before committing to full contextual retrieval.

---

## References

- Anthropic, "Introducing Contextual Retrieval" -- https://www.anthropic.com/news/contextual-retrieval
- Anthropic prompt caching pricing -- https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- Anthropic API usage tracking -- https://docs.anthropic.com/en/api/messages
