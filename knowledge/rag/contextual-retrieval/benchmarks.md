# Contextual Retrieval: Benchmarks and Evaluation

## Overview / TL;DR

Anthropic published benchmark results showing that contextual retrieval with BM25 and reranking reduces retrieval failure rates by 67% across multiple knowledge domains. This document covers Anthropic's published numbers in detail, explains how to replicate the evaluation on your own corpus, and provides a framework for measuring whether contextual retrieval is working for your specific use case. The key insight: the 67% number is an average -- your mileage will vary by document type, and measuring on your own data is essential.

---

## Anthropic's Published Numbers

### Top-Level Results

Tested across multiple knowledge domains (codebases, scientific papers, fiction), measuring top-20 retrieval failure rate (percentage of queries where the correct chunk is NOT in the top-20 results).

| Configuration | Avg Failure Rate Reduction |
|--------------|--------------------------|
| Traditional Embeddings (baseline) | 0% |
| Contextual Embeddings | 35% |
| Contextual Embeddings + Contextual BM25 | 49% |
| Contextual Embeddings + Contextual BM25 + Reranking | **67%** |

### What "67% Reduction" Means Concretely

If your baseline system has a 30% failure rate (30 out of 100 queries fail to retrieve the correct chunk in top-20):

```
After contextual retrieval:
  New failure rate = 30% * (1 - 0.67) = 9.9%
  Failed queries reduced from 30 to ~10 out of 100
```

If your baseline is 15% failure rate:
```
  New failure rate = 15% * (1 - 0.67) = 4.95%
  Failed queries reduced from 15 to ~5 out of 100
```

### Breakdown by Technique Combination

| # | Technique | Failure Rate (absolute) | vs Baseline |
|---|-----------|------------------------|-------------|
| 1 | Standard embeddings | ~20% | Baseline |
| 2 | Standard embeddings + BM25 | ~17% | -15% |
| 3 | Standard embeddings + BM25 + reranking | ~14% | -30% |
| 4 | Contextual embeddings | ~13% | -35% |
| 5 | Contextual embeddings + BM25 | ~10.2% | -49% |
| 6 | Contextual embeddings + contextual BM25 | ~9.5% | -52.5% |
| 7 | Contextual embeddings + contextual BM25 + reranking | ~6.6% | -67% |

Note: The exact absolute numbers vary by domain. The percentage reductions are what Anthropic reported as averages across their test suite.

### Key Observations from Anthropic's Results

1. **BM25 adds value even without contextualization** (line 2 vs 1). Hybrid search is always better than vector-only.
2. **Contextual BM25 outperforms standard BM25** (line 6 vs 5). Adding context to the BM25 index provides a meaningful 3.5% additional improvement.
3. **Reranking provides the final push** (line 7 vs 6). The cross-encoder reranker is most effective when the candidate pool is already high-quality.
4. **The techniques stack multiplicatively, not additively.** Each layer reduces the remaining failure rate, not the total.

---

## How to Replicate on Your Own Corpus

### Step 1: Create an Evaluation Dataset

You need a set of (question, correct_chunk_id) pairs. There are three ways to create these:

**Method A: Manual curation (highest quality)**

```python
eval_set = [
    {
        "question": "What is the default timeout for API requests?",
        "correct_chunk_ids": ["doc-api-ref-chunk-42"],
        "difficulty": "easy",
        "type": "factual",
    },
    {
        "question": "How do I configure rate limiting for the payment endpoint?",
        "correct_chunk_ids": ["doc-payment-chunk-15", "doc-rate-limit-chunk-8"],
        "difficulty": "medium",
        "type": "multi-hop",
    },
]
```

**Method B: Synthetic generation from chunks**

```python
import anthropic

client = anthropic.Anthropic()


def generate_eval_questions(chunk_text: str, chunk_id: str, n: int = 3) -> list[dict]:
    """Generate questions that this chunk should answer."""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=500,
        messages=[{
            "role": "user",
            "content": (
                f"Given the following text passage, generate {n} questions that "
                f"can ONLY be answered using information in this passage. "
                f"Questions should be specific and diverse in difficulty.\n\n"
                f"Passage:\n{chunk_text}\n\n"
                f"Return as JSON: [{{'question': '...', 'difficulty': 'easy|medium|hard'}}]"
            ),
        }],
    )
    import json
    questions = json.loads(response.content[0].text)
    for q in questions:
        q["correct_chunk_ids"] = [chunk_id]
    return questions
```

**Method C: RAGAS synthetic test generation**

```python
from ragas.testset.generator import TestsetGenerator
from ragas.testset.evolutions import simple, reasoning, multi_context
from langchain_openai import ChatOpenAI, OpenAIEmbeddings

generator_llm = ChatOpenAI(model="gpt-4o")
critic_llm = ChatOpenAI(model="gpt-4o")
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

generator = TestsetGenerator.from_langchain(
    generator_llm=generator_llm,
    critic_llm=critic_llm,
    embeddings=embeddings,
)

testset = generator.generate_with_langchain_docs(
    documents=documents,
    test_size=100,
    distributions={simple: 0.4, reasoning: 0.3, multi_context: 0.3},
)
```

### Step 2: Measure Baseline Retrieval

```python
import numpy as np
from sentence_transformers import SentenceTransformer


def measure_retrieval_quality(
    index,  # Your vector index
    eval_set: list[dict],
    top_k_values: list[int] = [5, 10, 20],
) -> dict:
    """Measure hit rate and MRR at various top-k values."""
    results = {k: {"hits": 0, "mrr_sum": 0} for k in top_k_values}
    total = len(eval_set)

    for item in eval_set:
        query = item["question"]
        correct_ids = set(item["correct_chunk_ids"])

        # Retrieve
        retrieved = index.search(query, top_k=max(top_k_values))
        retrieved_ids = [r["id"] for r in retrieved]

        for k in top_k_values:
            top_k_ids = retrieved_ids[:k]

            # Hit rate (recall@k)
            if any(rid in correct_ids for rid in top_k_ids):
                results[k]["hits"] += 1

            # MRR (Mean Reciprocal Rank)
            for rank, rid in enumerate(top_k_ids, 1):
                if rid in correct_ids:
                    results[k]["mrr_sum"] += 1 / rank
                    break

    metrics = {}
    for k in top_k_values:
        metrics[f"hit_rate@{k}"] = results[k]["hits"] / total
        metrics[f"mrr@{k}"] = results[k]["mrr_sum"] / total
        metrics[f"failure_rate@{k}"] = 1 - (results[k]["hits"] / total)

    return metrics
```

### Step 3: Measure with Contextual Retrieval

Run the same evaluation after contextualizing your chunks:

```python
async def run_ab_comparison(
    documents: list[dict],
    eval_set: list[dict],
):
    """Compare baseline vs contextual retrieval."""
    from contextual_retrieval import ContextualRetrievalPipeline, DualIndex

    # --- Baseline ---
    print("Building baseline index...")
    baseline_index = build_standard_index(documents)  # Your standard RAG index
    baseline_metrics = measure_retrieval_quality(baseline_index, eval_set)
    print(f"Baseline: {baseline_metrics}")

    # --- Contextual ---
    print("Building contextual index...")
    ctx_pipeline = CostOptimizedContextualizer()
    ctx_index = DualIndex()

    async for contextualized_chunks in ctx_pipeline.process_corpus(documents):
        ctx_index.add_chunks(contextualized_chunks)

    ctx_metrics = measure_retrieval_quality(ctx_index, eval_set)
    print(f"Contextual: {ctx_metrics}")

    # --- Comparison ---
    print("\n--- Comparison ---")
    for metric in baseline_metrics:
        baseline_val = baseline_metrics[metric]
        ctx_val = ctx_metrics[metric]
        if "failure" in metric:
            improvement = (baseline_val - ctx_val) / baseline_val * 100
            print(f"{metric}: {baseline_val:.3f} -> {ctx_val:.3f} ({improvement:+.1f}% reduction)")
        else:
            improvement = (ctx_val - baseline_val) / baseline_val * 100
            print(f"{metric}: {baseline_val:.3f} -> {ctx_val:.3f} ({improvement:+.1f}% improvement)")
```

### Step 4: Measure Each Layer Independently

To understand which layer contributes most for your data:

```python
async def ablation_study(documents, eval_set):
    """Test each layer independently to find the most impactful."""
    configs = [
        {"name": "Standard Embeddings", "contextual": False, "bm25": False, "rerank": False},
        {"name": "Standard + BM25", "contextual": False, "bm25": True, "rerank": False},
        {"name": "Standard + BM25 + Rerank", "contextual": False, "bm25": True, "rerank": True},
        {"name": "Contextual Embeddings", "contextual": True, "bm25": False, "rerank": False},
        {"name": "Contextual + BM25", "contextual": True, "bm25": True, "rerank": False},
        {"name": "Contextual + Ctx BM25", "contextual": True, "bm25": True, "rerank": False},
        {"name": "Contextual + Ctx BM25 + Rerank", "contextual": True, "bm25": True, "rerank": True},
    ]

    results = []
    for config in configs:
        # Build index according to config
        index = build_index_with_config(documents, config)
        metrics = measure_retrieval_quality(index, eval_set)
        results.append({"config": config["name"], **metrics})
        print(f"{config['name']}: failure@20 = {metrics['failure_rate@20']:.3f}")

    return results
```

---

## Expected Results by Document Type

Based on the general patterns in Anthropic's results and community replications:

| Document Type | Baseline Failure@20 | After Contextual+BM25+Rerank | Improvement |
|--------------|--------------------|-----------------------------|-------------|
| Technical docs (well-structured) | 15-20% | 5-8% | 55-65% |
| API references | 10-15% | 3-6% | 55-65% |
| Research papers | 25-35% | 8-12% | 60-70% |
| Legal documents | 20-30% | 7-12% | 55-65% |
| Code repositories | 20-30% | 6-10% | 65-70% |
| Conversational transcripts | 30-40% | 12-18% | 55-60% |
| Product catalogs | 10-15% | 5-8% | 40-50% |

**Why product catalogs benefit less**: Product descriptions are typically self-contained (each product card has its own name, category, and specs). There is less context lost during chunking.

**Why code benefits more**: Code chunks frequently reference variables, classes, and functions defined elsewhere in the file. Contextual retrieval adds the file name, module, and class context that makes chunks retrievable.

---

## Building a Custom Eval Pipeline

```python
"""Complete evaluation pipeline for contextual retrieval."""

import json
import time
from pathlib import Path
from dataclasses import dataclass, field


@dataclass
class EvalResult:
    config_name: str
    hit_rate_5: float
    hit_rate_10: float
    hit_rate_20: float
    mrr_20: float
    failure_rate_20: float
    avg_latency_ms: float
    ingestion_cost: float
    num_chunks: int
    timestamp: str = ""


def run_full_eval(
    eval_set: list[dict],
    index,
    config_name: str,
    ingestion_cost: float = 0.0,
) -> EvalResult:
    """Run evaluation and return structured results."""
    latencies = []
    hits = {5: 0, 10: 0, 20: 0}
    mrr_sum = 0.0
    total = len(eval_set)

    for item in eval_set:
        start = time.time()
        retrieved = index.search(item["question"], top_k=20)
        elapsed = (time.time() - start) * 1000
        latencies.append(elapsed)

        retrieved_ids = [r.get("id", r.get("metadata", {}).get("chunk_id", "")) for r in retrieved]
        correct_ids = set(item["correct_chunk_ids"])

        for k in [5, 10, 20]:
            if any(rid in correct_ids for rid in retrieved_ids[:k]):
                hits[k] += 1

        for rank, rid in enumerate(retrieved_ids[:20], 1):
            if rid in correct_ids:
                mrr_sum += 1 / rank
                break

    return EvalResult(
        config_name=config_name,
        hit_rate_5=hits[5] / total,
        hit_rate_10=hits[10] / total,
        hit_rate_20=hits[20] / total,
        mrr_20=mrr_sum / total,
        failure_rate_20=1 - (hits[20] / total),
        avg_latency_ms=sum(latencies) / len(latencies),
        ingestion_cost=ingestion_cost,
        num_chunks=len(index.chunks),
        timestamp=time.strftime("%Y-%m-%d %H:%M:%S"),
    )


def compare_results(results: list[EvalResult]) -> str:
    """Generate a comparison table."""
    header = f"{'Config':<40} {'HR@5':>6} {'HR@10':>6} {'HR@20':>6} {'MRR@20':>7} {'Fail@20':>8} {'Latency':>8} {'Cost':>8}"
    lines = [header, "-" * len(header)]

    baseline = results[0].failure_rate_20

    for r in results:
        improvement = ((baseline - r.failure_rate_20) / baseline * 100) if baseline > 0 else 0
        lines.append(
            f"{r.config_name:<40} "
            f"{r.hit_rate_5:>6.3f} "
            f"{r.hit_rate_10:>6.3f} "
            f"{r.hit_rate_20:>6.3f} "
            f"{r.mrr_20:>7.3f} "
            f"{r.failure_rate_20:>7.3f} "
            f"{r.avg_latency_ms:>7.1f}ms "
            f"${r.ingestion_cost:>6.2f}"
        )

    return "\n".join(lines)
```

---

## Evaluation Dataset Size Guidelines

| Purpose | Minimum Questions | Recommended | Notes |
|---------|------------------|-------------|-------|
| Smoke test | 10 | 20 | Quick sanity check |
| A/B comparison | 50 | 100 | Statistically meaningful |
| Production validation | 100 | 200-500 | Per document category |
| Comprehensive benchmark | 500 | 1000+ | For published results |

**Statistical significance**: With 100 eval questions, a 10% improvement (e.g., 20% -> 10% failure rate) is statistically significant at p < 0.05 using McNemar's test. With 50 questions, you need a larger difference (~15%) for significance.

```python
from scipy.stats import chi2

def mcnemar_test(n_improved: int, n_degraded: int) -> float:
    """McNemar's test for paired proportions.
    n_improved: queries that failed in baseline but succeeded in new system
    n_degraded: queries that succeeded in baseline but failed in new system
    """
    if n_improved + n_degraded == 0:
        return 1.0
    chi2_stat = (abs(n_improved - n_degraded) - 1) ** 2 / (n_improved + n_degraded)
    p_value = 1 - chi2.cdf(chi2_stat, df=1)
    return p_value
```

---

## Monitoring Contextual Retrieval in Production

After deploying contextual retrieval, monitor these metrics continuously:

```python
import logging

logger = logging.getLogger(__name__)


def log_retrieval_quality(
    query: str,
    retrieved_chunks: list[dict],
    user_feedback: str = None,  # "helpful", "not_helpful", None
):
    """Log retrieval signals for ongoing monitoring."""
    logger.info(
        "retrieval_event",
        extra={
            "query_length": len(query.split()),
            "num_retrieved": len(retrieved_chunks),
            "top_score": retrieved_chunks[0]["score"] if retrieved_chunks else 0,
            "score_spread": (
                retrieved_chunks[0]["score"] - retrieved_chunks[-1]["score"]
                if len(retrieved_chunks) > 1 else 0
            ),
            "has_bm25_match": any(c.get("bm25_score", 0) > 0 for c in retrieved_chunks),
            "user_feedback": user_feedback,
        },
    )
```

**Key signals to track**:

| Signal | Healthy Range | Action If Out of Range |
|--------|-------------|----------------------|
| Top-1 rerank score | > 0.5 | Check query quality or index freshness |
| Score spread (top1 - top5) | > 0.2 | Low spread = no clear winner = uncertain retrieval |
| BM25 contribution | > 20% of top-5 results | Check BM25 index rebuild |
| User "not helpful" rate | < 15% | Investigate failure patterns |
| Context preamble quality | N/A | Spot-check 50 preambles quarterly |

---

## Common Pitfalls

1. **Evaluating on too few questions.** With 20 questions, random variance can show a 15% improvement that is pure noise. Use at least 50 questions for meaningful comparisons.
2. **Not controlling for re-ranking when comparing.** If you add contextual retrieval AND re-ranking simultaneously, you cannot attribute improvement to either. Test each layer independently.
3. **Using the same LLM for eval generation and answer generation.** The eval LLM may generate questions that are biased toward how your generation model phrases answers, inflating scores.
4. **Not measuring per-query improvements.** Average metrics can hide that contextual retrieval helps 40% of queries and hurts 5%. Look at per-query deltas to find regression patterns.
5. **Assuming Anthropic's 67% applies to your data.** The improvement varies dramatically by document type (see table above). Always measure on your own corpus.
6. **Forgetting to re-evaluate after corpus updates.** When you add new documents, the contextual retrieval quality may shift. Run periodic evaluations (monthly) on a fixed eval set.

---

## References

- Anthropic, "Introducing Contextual Retrieval" -- https://www.anthropic.com/news/contextual-retrieval
- RAGAS evaluation framework -- https://docs.ragas.io/
- McNemar's test for paired comparisons -- https://en.wikipedia.org/wiki/McNemar%27s_test
- BEIR benchmark for information retrieval -- https://github.com/beir-cellar/beir
