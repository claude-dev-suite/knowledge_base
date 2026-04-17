# RankGPT -- LLM-as-Reranker

## Overview

RankGPT is a paradigm for using large language models (LLMs) as document rerankers. Instead of training a specialized cross-encoder or learning a scoring function, RankGPT treats reranking as a **text generation task**: the LLM receives a query and a list of candidate documents, then outputs a permutation (reordered ranking) of those documents. The approach was introduced by Sun et al. (2023) and has spawned a family of techniques that leverage the reasoning capabilities of instruction-tuned LLMs for relevance judgment.

The core insight is that modern LLMs already understand relevance -- they can read a query and a passage, then determine which passage better answers the query, without any task-specific fine-tuning. This makes RankGPT a zero-shot or few-shot reranker that works across domains out of the box.

---

## The Three Approaches to LLM Reranking

### Pointwise Scoring

The simplest approach: ask the LLM to score each document independently.

```
System: You are a relevance assessor. Given a query and a passage,
rate the relevance of the passage to the query on a scale of 0-10.
Output only the numeric score.

User:
Query: "how to handle Python memory leaks"
Passage: "Python's garbage collector uses reference counting as its
primary mechanism. When an object's reference count drops to zero,
it is immediately deallocated. However, circular references require
the cyclic garbage collector to detect and clean up."
```

**Pros**: simple, parallelizable (each doc scored independently), gives a numeric score.

**Cons**: scores are poorly calibrated -- the LLM might give "8/10" to everything or nothing. The model has no sense of relative relevance because it never sees other candidates.

### Pairwise Comparison

Ask the LLM to compare two documents and pick the more relevant one:

```
System: Given a query and two passages (A and B), determine which
passage is more relevant to the query. Output only "A" or "B".

User:
Query: "how to handle Python memory leaks"
Passage A: "Python's garbage collector uses reference counting..."
Passage B: "Memory leaks in Python commonly occur with circular
references, unclosed file handles, and C extension objects that..."
```

**Pros**: more reliable than pointwise (relative comparison is easier for LLMs than absolute scoring). Better calibrated.

**Cons**: O(n^2) comparisons for n documents (need to compare all pairs), or O(n log n) with a sorting algorithm. Expensive for large candidate lists.

### Listwise Ranking (RankGPT)

The RankGPT approach: give the LLM the query and ALL candidate documents at once, ask it to output the complete ranking:

```
System: You are RankGPT, an intelligent assistant that can rank
passages based on their relevancy to the query.

User:
I will provide you with {n} passages, each indicated by a numeric
identifier [1] to [{n}]. Rank the passages based on their relevance
to the query: "{query}"

[1] {passage_1}
[2] {passage_2}
...
[n] {passage_n}

Rank the {n} passages above based on their relevance to the query.
The passages should be listed in descending order using identifiers.
The most relevant passage should be listed first. The output format
should be [] > [] > ... > [], e.g., [1] > [2] > ... > [{n}].
Only output the ranking.
```

**Pros**: best quality -- the model sees all candidates and can make holistic comparisons. Single inference call for the entire ranking.

**Cons**: limited by context window (fitting 20+ passages in one prompt is challenging), order bias (documents listed first may be favored), and the output format requires careful parsing.

---

## The RankGPT Paper: Key Contributions

Sun et al. (2023) "Is ChatGPT Good at Search? Investigating Large Language Models as Re-Ranking Agents" introduced three main ideas:

### 1. Listwise Reranking is Superior

The paper systematically compared pointwise, pairwise, and listwise approaches across multiple benchmarks. Listwise consistently outperformed the others, often by significant margins. The hypothesis is that seeing all candidates simultaneously lets the model make more nuanced distinctions.

### 2. Sliding Window for Long Candidate Lists

When the candidate list exceeds what fits in the context window, the paper proposes a **sliding window** approach:

```
Candidate list: [d1, d2, d3, d4, ..., d20]

Step 1: Rank [d11, d12, ..., d20]  (last 10)
        Result: [d15, d18, d11, d20, d13, d17, d14, d12, d16, d19]

Step 2: Rank [d6, d7, ..., d10, d15, d18, d11, d20, d13]  (next window)
        The top results from step 1 "bubble up" if they are truly relevant

Step 3: Rank [d1, d2, ..., d5, <top from step 2>]
        Final result: best documents have bubbled to the top

This is analogous to a bubble sort where each "comparison" is a
full listwise ranking of a window.
```

The window slides from the bottom of the initial ranking to the top, allowing relevant documents to "bubble up" through successive windows.

### 3. Permutation Distillation

The paper showed that rankings from GPT-4 can be distilled into smaller models (e.g., a fine-tuned FLAN-T5 or LLaMA) by training on the input/output pairs of the listwise ranking task. This creates fast, cheap rerankers that approximate GPT-4 quality.

---

## Why Listwise Ranking Works

### Calibration-Free Relevance

Pointwise scoring requires the model to produce calibrated scores on a fixed scale. This is inherently difficult -- what does "7 out of 10" mean for relevance? Different queries and domains have different baselines.

Listwise ranking avoids this entirely: the model only needs to determine the relative ordering. "Is passage A more relevant than passage B?" is a much easier judgment than "Is passage A a 7 or an 8?"

### Holistic Comparison

When the model sees all candidates simultaneously, it can identify:
- Which passage has the most specific answer (not just topically related)
- Which passage would be redundant given other results
- Which passage answers the actual intent behind the query

Consider a query "Python async database connections":
- Passage A: general Python async tutorial
- Passage B: specific asyncpg connection pooling guide
- Passage C: comparison of sync vs. async database drivers

A pointwise scorer might rate all three as "highly relevant." A listwise ranker recognizes that B is the most specific and useful, C provides complementary information, and A is too general.

### Position Bias and Mitigation

LLMs exhibit position bias -- documents listed earlier in the prompt tend to receive higher rankings. The sliding window approach partially mitigates this by processing documents in overlapping windows. Additional mitigation strategies:

1. **Reverse initial ordering**: present documents in reverse BM25/initial ranking order so that truly relevant documents must overcome position bias to rank highly
2. **Multiple passes with shuffled order**: rank the same set multiple times with different orderings, then aggregate (expensive but effective)
3. **Explicit instructions**: "Do not let the position of a passage in the list influence your ranking"

---

## Architecture of a RankGPT Reranking Pipeline

A typical production pipeline using RankGPT:

```
Query
  |
  v
First-Stage Retriever (BM25, dense, SPLADE)
  |  returns top-100 candidates
  v
Optional Pre-Filter (remove duplicates, apply metadata filters)
  |  reduces to top-20-50 candidates
  v
RankGPT Reranker (LLM listwise ranking)
  |  reranks to produce final top-10
  v
Application (RAG, search results page, etc.)
```

The key constraint is that the LLM reranker is expensive (time and cost), so it should operate on a small candidate set (10-50 documents). The first-stage retriever handles the heavy lifting of narrowing millions of documents to tens of candidates.

---

## Practical Considerations

### Context Window Limits

Each candidate document consumes tokens in the prompt. A 200-word passage is approximately 250-300 tokens. For a window of 20 passages:

```
System prompt:        ~100 tokens
20 passages:          ~5,000-6,000 tokens
Query + formatting:   ~100 tokens
Total input:          ~5,200-6,200 tokens
Output (ranking):     ~100 tokens
```

This fits comfortably in models with 8K+ context windows. For longer passages or more candidates, you need either larger context windows (100K+) or the sliding window approach.

### Output Parsing

The model outputs a permutation like `[3] > [1] > [5] > [2] > [4]`. Robust parsing must handle:

1. Variations in format: `[3]>[1]>[5]` vs `3 > 1 > 5` vs `3, 1, 5`
2. Missing identifiers: the model might forget to include some documents
3. Invalid identifiers: the model might hallucinate document IDs
4. Duplicates: the model might list a document twice

### Cost and Latency

For a typical reranking of 20 passages with a 200-word average:

| Model | Input Cost | Output Cost | Latency | Total Cost (per query) |
|---|---|---|---|---|
| Claude Sonnet | ~6K input tokens | ~50 output tokens | ~1.5s | ~$0.02 |
| Claude Haiku | ~6K input tokens | ~50 output tokens | ~0.5s | ~$0.005 |
| GPT-4o | ~6K input tokens | ~50 output tokens | ~2s | ~$0.02 |
| GPT-4o-mini | ~6K input tokens | ~50 output tokens | ~0.8s | ~$0.002 |

At 1000 queries/day, this ranges from $2/day (Haiku/4o-mini) to $20/day (Sonnet/GPT-4o). For high-volume search (millions of queries/day), LLM reranking is prohibitively expensive without distillation.

---

## Comparison with Other Reranking Approaches

| Property | LLM Reranker (RankGPT) | Cross-Encoder | Cohere Rerank | BM25 (no reranking) |
|---|---|---|---|---|
| Quality (BEIR avg) | Excellent | Very good | Good-Very good | Baseline |
| Zero-shot domain transfer | Excellent | Good (varies) | Good | N/A |
| Latency (20 docs) | 500ms-3s | 20-50ms | 50-200ms | 0ms |
| Cost per query | $0.002-0.02 | ~$0 (self-hosted) | $0.001-0.002 | $0 |
| Customizable | Prompt engineering | Fine-tuning | No | Parameter tuning |
| Explainable | Can output reasoning | No | No | No |
| Max throughput | ~10-100 QPS | ~500-2000 QPS | ~100-500 QPS | Unlimited |

---

## When to Use RankGPT

### Use LLM reranking when:

1. **Quality is paramount and volume is low**: internal tools, research, high-value searches where each query matters
2. **Domain transfer without training data**: you are searching a new domain and have no labeled relevance data for fine-tuning a cross-encoder
3. **Explainability is needed**: you can ask the LLM to explain its ranking decisions (add "briefly explain your ranking" to the prompt)
4. **Rapid prototyping**: LLM reranking works out of the box with zero training, making it ideal for prototypes and MVPs

### Do not use LLM reranking when:

1. **High throughput is required**: >100 QPS requires dedicated infrastructure that is cheaper to serve with cross-encoders
2. **Cost is critical**: at millions of queries/day, even cheap models add up
3. **Latency budget is tight**: <100ms reranking rules out most LLM approaches
4. **You have labeled training data**: if you can fine-tune a cross-encoder on your domain, it will be faster and cheaper while approaching LLM quality

---

## Key Results from the Literature

### Sun et al. (2023) -- Original RankGPT Paper

Evaluated on TREC-DL 2019 and TREC-DL 2020 with BM25 as the first-stage retriever (top 100 candidates):

| Method | TREC-DL19 NDCG@10 | TREC-DL20 NDCG@10 |
|---|---|---|
| BM25 (no reranking) | 0.506 | 0.480 |
| monoT5-3B (pointwise) | 0.711 | 0.674 |
| RankGPT (GPT-3.5, listwise) | 0.658 | 0.622 |
| RankGPT (GPT-4, listwise) | 0.729 | 0.694 |

### Subsequent Work

| Method | Avg BEIR NDCG@10 (reranking BM25 top-100) | Notes |
|---|---|---|
| No reranking (BM25) | 0.432 | Baseline |
| Cross-encoder (MiniLM-L-12) | 0.501 | Fast, cheap |
| Cohere Rerank v3 | 0.510 | API-based |
| RankGPT (Claude Sonnet) | 0.525 | High quality, expensive |
| RankGPT (GPT-4o) | 0.530 | Highest quality, most expensive |
| RankVicuna (distilled 7B) | 0.495 | Distilled from GPT-4 rankings |

---

## Common Pitfalls

1. **Reranking too many documents**: LLM reranking of 100 documents is expensive and slow. Keep the candidate list to 10-30 for cost efficiency. Use a good first-stage retriever to minimize the number of candidates.

2. **Ignoring position bias**: documents listed first in the prompt are favored. Always reverse the initial ranking or shuffle before presenting to the LLM.

3. **Not validating the output permutation**: the LLM may output invalid rankings (missing IDs, duplicates, hallucinated IDs). Always validate and fall back to the original ranking for missing documents.

4. **Using pointwise scoring when listwise is available**: pointwise scoring is tempting for its simplicity but consistently underperforms listwise ranking.

5. **Not considering distillation**: if you are running RankGPT in production, distill the LLM rankings into a smaller model (FLAN-T5, Mistral 7B) for 10-100x cost and latency reduction.

6. **Treating all passages equally**: truncate long passages to ~200 words. Including full 2000-word documents wastes tokens and may degrade ranking quality as the model struggles with too much text.

---

## References

- Sun, W. et al. "Is ChatGPT Good at Search? Investigating Large Language Models as Re-Ranking Agents." arXiv 2023 (RankGPT paper).
- Ma, X. et al. "Zero-Shot Listwise Document Reranking with a Large Language Model." arXiv 2023.
- Pradeep, R. et al. "RankVicuna: Zero-Shot Listwise Document Reranking with Open-Source Large Language Models." arXiv 2023.
- Qin, Z. et al. "Large Language Models are Effective Text Rankers with Pairwise Ranking Prompting." arXiv 2023.
- Zhuang, S. et al. "Beyond Yes and No: Improving Zero-Shot LLM Rankers via Scoring Fine-Grained Relevance Labels." arXiv 2023.
- Tang, Y. et al. "Found in the Middle: Permutation Self-Consistency Improves Listwise Ranking in Large Language Models." arXiv 2023.
