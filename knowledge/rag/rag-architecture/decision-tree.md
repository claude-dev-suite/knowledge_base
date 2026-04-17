# RAG vs Fine-Tuning vs Long-Context: Decision Tree

## Overview / TL;DR

When you need an LLM to use custom knowledge, the four main approaches are RAG, fine-tuning, long-context prompting, and hybrid combinations. The right choice depends on five concrete factors: corpus size, update frequency, accuracy requirements, latency budget, and cost constraints. This document provides a structured decision framework with quantified criteria, a text-based decision flowchart, and head-to-head comparisons across real scenarios.

---

## The Four Approaches

### RAG (Retrieval-Augmented Generation)

Retrieve relevant chunks from an external knowledge base at query time and inject them into the prompt.

**Strengths**: Scales to millions of documents, supports real-time updates, provides source attribution, no model retraining needed.

**Weaknesses**: Retrieval latency (50-500ms added), retrieval can miss relevant context, requires vector DB infrastructure, chunk boundaries can split important information.

### Fine-Tuning

Train the model's weights on your domain data so knowledge is encoded parametrically.

**Strengths**: No retrieval latency, can learn domain-specific language patterns, style, and reasoning. Works well for structured output formats.

**Weaknesses**: Expensive to train ($50-$5000+ per run), knowledge goes stale (requires retraining), no source attribution, hallucination risk for specific facts, limited by training data size.

### Long-Context Prompting

Stuff the entire knowledge base (or large portions) directly into the prompt using models with 100K-1M+ token context windows.

**Strengths**: Simplest to implement (no infrastructure), model sees all information simultaneously, works well for small corpora.

**Weaknesses**: Cost scales linearly with context size, latency increases with prompt length (TTFT), "lost in the middle" effect degrades accuracy for information in the center of long contexts, impractical beyond ~500 pages.

### Hybrid: RAG + Fine-Tuning

Fine-tune the model to be better at using retrieved context, to follow domain conventions, or to learn retrieval-resistant knowledge (common patterns, abbreviations, jargon).

**Strengths**: Best of both worlds -- parametric domain knowledge plus real-time retrieval. Reduced hallucination on domain terminology.

**Weaknesses**: Most complex to build and maintain. Two systems to keep in sync. Highest cost.

---

## Decision Criteria Matrix

| Criterion | RAG | Fine-Tuning | Long-Context | RAG + Fine-Tune |
|-----------|-----|-------------|--------------|-----------------|
| **Corpus size** | 10K-10M+ docs | 100-100K examples | Up to ~500 pages | Any size |
| **Update frequency** | Real-time to daily | Weekly to monthly | Real-time | Daily + periodic retrain |
| **Accuracy (factual)** | High (with reranking) | Medium | High (small corpus) | Highest |
| **Accuracy (style)** | Medium | High | Medium | High |
| **Latency added** | 50-500ms | 0ms | 0ms (but slow TTFT) | 50-500ms |
| **Cost (setup)** | Medium ($100-5K) | High ($500-50K) | Low ($0-50) | Highest |
| **Cost (per query)** | $0.001-0.02 | Same as base model | $0.01-0.50 | $0.005-0.03 |
| **Source attribution** | Native | Not possible | Possible but noisy | Native |
| **Infrastructure** | Vector DB + embeddings | Training pipeline | None | Both |
| **Maintenance** | Index updates | Periodic retraining | Keep docs current | Both |

---

## Decision Flowchart

Follow this text-based flowchart from top to bottom. At each decision point, take the branch that matches your situation.

```
START: You need an LLM to use custom knowledge
  |
  +---> Is your corpus < 200 pages AND rarely changes?
  |     |
  |     YES --> Can you tolerate $0.05-0.50/query cost?
  |     |       |
  |     |       YES --> LONG-CONTEXT PROMPTING
  |     |       |       (Simplest. Just stuff it in the prompt.)
  |     |       |
  |     |       NO --> RAG
  |     |               (Retrieve only what's needed to save tokens.)
  |     |
  |     NO --> Does knowledge change more than weekly?
  |            |
  |            YES --> Do you need source citations?
  |            |       |
  |            |       YES --> RAG
  |            |       |       (Only RAG gives real-time updates + citations.)
  |            |       |
  |            |       NO --> Is the primary goal style/format compliance?
  |            |               |
  |            |               YES --> RAG + FINE-TUNING
  |            |               |       (Fine-tune for style, RAG for facts.)
  |            |               |
  |            |               NO --> RAG
  |            |                       (Cheaper and simpler than hybrid.)
  |            |
  |            NO --> Is the primary need factual recall or style/behavior?
  |                   |
  |                   FACTUAL --> Is corpus > 10K documents?
  |                   |          |
  |                   |          YES --> RAG
  |                   |          |       (Fine-tuning can't absorb this volume.)
  |                   |          |
  |                   |          NO --> Do you need source attribution?
  |                   |                  |
  |                   |                  YES --> RAG
  |                   |                  |
  |                   |                  NO --> FINE-TUNING
  |                   |                          (Parametric knowledge for stable facts.)
  |                   |
  |                   STYLE --> Are there < 5000 style examples?
  |                             |
  |                             YES --> FINE-TUNING
  |                             |       (Few examples are enough for style transfer.)
  |                             |
  |                             NO --> Do you also need factual grounding?
  |                                    |
  |                                    YES --> RAG + FINE-TUNING
  |                                    |
  |                                    NO --> FINE-TUNING
```

---

## Detailed Comparison by Scenario

### Scenario 1: Internal Documentation Bot

**Context**: 5,000 Confluence pages, updated daily, employees ask questions.

| Factor | Value |
|--------|-------|
| Corpus size | 5,000 pages (~2.5M tokens) |
| Update frequency | Daily |
| Accuracy need | High (employees trust the bot) |
| Latency budget | < 3 seconds |
| Citation need | Yes (link to source page) |

**Best approach**: **RAG (Advanced)**

Reasoning: Corpus is too large for long-context (2.5M tokens exceeds even 1M context windows). Daily updates rule out fine-tuning. Citations are required, which RAG provides natively. Advanced RAG with hybrid search handles the mix of technical terms (BM25) and conceptual queries (vector search).

### Scenario 2: Legal Contract Review

**Context**: 50 contract templates (stable), need to flag deviations from standard clauses.

| Factor | Value |
|--------|-------|
| Corpus size | 50 documents (~100 pages) |
| Update frequency | Monthly |
| Accuracy need | Very high (legal liability) |
| Latency budget | < 10 seconds (not user-facing) |
| Citation need | Yes (specific clause references) |

**Best approach**: **RAG + Long-Context hybrid**

Reasoning: The corpus is small enough for long-context, but clause-level precision benefits from RAG's targeted retrieval. Use RAG to find the most relevant standard clauses, then include the full contract in context. The combination provides both precision (RAG) and completeness (full document context).

### Scenario 3: Customer Support Chatbot (Brand Voice)

**Context**: 200 product pages plus a specific brand voice and response format.

| Factor | Value |
|--------|-------|
| Corpus size | 200 pages |
| Update frequency | Weekly product updates |
| Accuracy need | High (customer-facing) |
| Latency budget | < 2 seconds |
| Style requirement | Strict brand voice compliance |

**Best approach**: **RAG + Fine-Tuning**

Reasoning: RAG handles the factual product knowledge with weekly updates. Fine-tuning encodes the brand voice, response format, and escalation patterns that are hard to capture in a system prompt alone. The fine-tuned model generates better brand-consistent responses from retrieved context.

### Scenario 4: Research Paper Q&A

**Context**: User uploads 1-5 papers and asks questions about them.

| Factor | Value |
|--------|-------|
| Corpus size | 1-5 papers (~50-200 pages per session) |
| Update frequency | Per session (ephemeral) |
| Accuracy need | High |
| Latency budget | < 5 seconds |
| Citation need | Yes (page/section references) |

**Best approach**: **Long-Context Prompting**

Reasoning: The corpus fits comfortably in a 200K token context window. Building a RAG pipeline per session adds unnecessary complexity. The model can reason across the entire paper simultaneously, which is better for cross-reference questions. Use a model with strong long-context performance (Claude 3.5 Sonnet, GPT-4 Turbo).

### Scenario 5: Enterprise Knowledge Graph (100K+ Documents)

**Context**: 500,000 documents across engineering, sales, legal, HR. Multiple teams.

| Factor | Value |
|--------|-------|
| Corpus size | 500,000 documents |
| Update frequency | Continuous (real-time ingestion) |
| Accuracy need | High with multi-hop reasoning |
| Latency budget | < 5 seconds |
| Multi-tenancy | Yes (access control per team) |

**Best approach**: **Agentic RAG**

Reasoning: The corpus is far too large for long-context or fine-tuning. Multi-hop questions require iterative retrieval. The agent can route queries to the appropriate index (engineering docs vs HR policies vs legal). Access control filters are applied at the retrieval layer per user's permissions. The agent can decompose complex queries like "What engineering decisions were influenced by the Q3 legal review?" into sub-queries across indexes.

---

## Cost Comparison Table

Assumptions: 10,000 queries/month, average 500-token query, average 1000-token response.

| Approach | Setup Cost | Monthly Infra | Per-Query Cost | Monthly Total |
|----------|-----------|---------------|----------------|--------------|
| **Long-Context (200K)** | $0 | $0 | ~$0.15 (input tokens) | ~$1,500 |
| **Long-Context (50K)** | $0 | $0 | ~$0.04 | ~$400 |
| **Naive RAG** | ~$200 | ~$50 (Chroma/Qdrant) | ~$0.003 | ~$80 |
| **Advanced RAG** | ~$500 | ~$100 (DB + reranker) | ~$0.01 | ~$200 |
| **Agentic RAG** | ~$2,000 | ~$150 | ~$0.05 | ~$650 |
| **Fine-Tuning (GPT-4o)** | ~$500/run | ~$0 | ~$0.006 | ~$60 + retrain |
| **Fine-Tuning (OSS)** | ~$200/run | ~$300 (GPU hosting) | ~$0.001 | ~$310 + retrain |
| **RAG + Fine-Tune** | ~$1,000 | ~$400 | ~$0.008 | ~$480 |

Note: Costs are approximate and vary significantly by provider, model, and implementation. Fine-tuning costs exclude the initial training run amortized over months.

---

## Hybrid Strategies in Detail

### RAG + Fine-Tuning: What to Put Where

The key insight is that RAG and fine-tuning serve different knowledge types:

| Knowledge Type | RAG | Fine-Tuning |
|---------------|-----|-------------|
| Specific facts (dates, numbers, names) | Best | Poor (memorization is unreliable) |
| Procedures and processes | Good | Good |
| Domain jargon and abbreviations | Poor (may not be in chunks) | Best |
| Response format/style | Poor | Best |
| Rapidly changing information | Best | Cannot handle |
| Reasoning patterns | Moderate | Good |
| Edge case handling | Best (if documented) | Good (if in training data) |

**Implementation pattern**:

```python
"""Hybrid RAG + Fine-Tuned model."""

import anthropic

# The fine-tuned model knows domain jargon, response format,
# and common reasoning patterns.
# RAG provides specific, up-to-date facts.

FINE_TUNED_MODEL = "ft:claude-sonnet:my-org:domain-v3:abc123"

client = anthropic.Anthropic()


def hybrid_query(question: str, retrieved_chunks: list[str]) -> str:
    context = "\n\n".join(retrieved_chunks)
    response = client.messages.create(
        model=FINE_TUNED_MODEL,  # Fine-tuned for domain style
        max_tokens=1500,
        system=(
            "You are a domain expert assistant. Use the retrieved context "
            "for specific facts. Apply your training for terminology, format, "
            "and reasoning patterns. Always cite retrieved sources."
        ),
        messages=[{
            "role": "user",
            "content": f"Context:\n{context}\n\nQuestion: {question}",
        }],
    )
    return response.content[0].text
```

### RAG + Long-Context: Belt and Suspenders

Use RAG to find the most relevant sections, then include them alongside a broader document context:

```python
def rag_plus_context(question: str, full_document: str, rag_chunks: list[str]) -> str:
    """Combine targeted RAG chunks with broader document context."""
    # RAG chunks go first (most relevant)
    rag_section = "MOST RELEVANT SECTIONS:\n" + "\n---\n".join(rag_chunks)

    # Full document provides broader context
    # Truncate if needed to fit token budget
    doc_section = f"FULL DOCUMENT:\n{full_document[:50000]}"

    context = f"{rag_section}\n\n{'='*50}\n\n{doc_section}"

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1500,
        system=(
            "Answer using the provided information. The MOST RELEVANT SECTIONS "
            "contain the best matches for the question. The FULL DOCUMENT "
            "provides additional context if needed."
        ),
        messages=[{
            "role": "user",
            "content": f"{context}\n\nQuestion: {question}",
        }],
    )
    return response.content[0].text
```

---

## When Each Approach Fails

### RAG Failure Modes

- **Query-document vocabulary mismatch**: The user asks about "auth" but documents say "authentication." Mitigation: query expansion, synonyms in metadata.
- **Multi-hop reasoning**: "What is the policy for employees in departments that exceeded their Q3 budget?" requires joining across multiple documents. Mitigation: agentic RAG.
- **Aggregation queries**: "How many products have a warranty longer than 2 years?" requires scanning all documents. Mitigation: structured index or SQL.
- **Temporal reasoning**: "What changed between v2.1 and v2.3?" Mitigation: versioned indexes with diff-aware retrieval.

### Fine-Tuning Failure Modes

- **Factual recall of specific details**: Fine-tuned models memorize patterns, not specific facts reliably. They may hallucinate plausible but wrong numbers.
- **Knowledge cutoff**: The model only knows what it was trained on. New information requires retraining.
- **Catastrophic forgetting**: Aggressive fine-tuning can degrade the model's general capabilities.
- **No attribution**: There is no way to trace an answer back to a specific training example.

### Long-Context Failure Modes

- **Lost in the middle**: Models perform best on information at the start and end of the context. Information in the middle (positions 30-70% of context) gets lower attention.
- **Cost explosion**: 200K tokens of input at $3/M tokens = $0.60 per query. At 10K queries/day, that is $6,000/day.
- **Latency**: Time-to-first-token scales with context length. A 200K token prompt may take 5-15 seconds for TTFT.
- **Context window exhaustion**: When your corpus grows beyond the window, you need RAG anyway.

---

## Common Pitfalls

1. **Choosing fine-tuning for factual recall.** Fine-tuning is for behavior, not facts. Use RAG for facts, fine-tuning for style.
2. **Using long-context as a permanent solution.** It works today at 200 pages. What happens at 2,000 pages? Build RAG infrastructure early if your corpus will grow.
3. **Not testing the "lost in the middle" effect.** If you use long-context, specifically test retrieval of information placed in the middle third of the prompt.
4. **Assuming fine-tuning replaces RAG.** Even OpenAI and Anthropic recommend RAG as the first approach for knowledge-grounded applications.
5. **Ignoring the maintenance cost of fine-tuning.** Each retraining run costs money, takes hours, and requires evaluation. If your knowledge changes weekly, fine-tuning becomes expensive.
6. **Building Agentic RAG before measuring Naive RAG.** Most applications do not need autonomous agents. Start simple, measure failure modes, add complexity only where evidence justifies it.

---

## References

- Anthropic, "Long context prompting tips" -- https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- Mallen et al., "When Not to Trust Language Models: Investigating Effectiveness of Parametric and Non-Parametric Memories" (2023) -- https://arxiv.org/abs/2212.10511
- Liu et al., "Lost in the Middle: How Language Models Use Long Contexts" (2024) -- https://arxiv.org/abs/2307.03172
- OpenAI, "Fine-tuning guide" -- https://platform.openai.com/docs/guides/fine-tuning
- LangChain, "RAG vs Fine-Tuning" -- https://blog.langchain.dev/rag-vs-finetuning/
