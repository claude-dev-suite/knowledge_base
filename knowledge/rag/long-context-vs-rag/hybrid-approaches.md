# Long Context vs RAG -- Hybrid Approaches: RAG Narrows, Long Context Reads

## TL;DR

The most effective production systems combine RAG and long context rather than choosing one exclusively. The hybrid pattern is: RAG narrows the corpus to a relevant subset (20-50 chunks), then the LLM reads that full subset in its long context window for deep understanding and synthesis. This avoids both the cost of stuffing the entire corpus into context and the accuracy loss of working with only 3-5 retrieved chunks. This article covers three hybrid architectures, implementation patterns, and benchmarks showing when hybrid outperforms either approach alone.

---

## Hybrid Architecture Patterns

### Pattern 1: RAG Pre-Filter + Long Context Read

```python
class HybridRAGLongContext:
    """RAG retrieves broadly, long context reads deeply.

    Flow:
    1. RAG retrieves top-50 relevant chunks
    2. Reranker selects top-20 (high-quality context)
    3. All 20 chunks are placed in the LLM context window
    4. LLM reads ALL 20 chunks and generates a comprehensive answer

    This is better than standard RAG (top-5 chunks) because:
    - More context means fewer missed relevant passages
    - LLM can cross-reference and synthesize across all 20 chunks
    - Cheaper than stuffing the entire corpus (only 20 chunks vs 10K+)
    """

    def __init__(self, retriever, reranker, llm, embedding_model):
        self.retriever = retriever
        self.reranker = reranker
        self.llm = llm
        self.embedder = embedding_model

    def query(self, question: str, broad_k: int = 50, deep_k: int = 20) -> dict:
        """Execute hybrid query."""
        # Phase 1: Broad RAG retrieval
        query_embedding = self.embedder.embed([question])[0]
        broad_results = self.retriever.query(query_embedding, top_k=broad_k)

        # Phase 2: Rerank and select top-K for deep reading
        reranked = self.reranker.rerank(
            question,
            [r.get("content", r.get("text", "")) for r in broad_results],
            top_k=deep_k,
        )

        # Phase 3: Build rich context for long-context reading
        context_chunks = []
        total_tokens = 0
        for doc, score in reranked:
            chunk_tokens = len(doc.split()) * 1.3  # rough token estimate
            if total_tokens + chunk_tokens > 50000:  # cap at 50K tokens
                break
            context_chunks.append(doc)
            total_tokens += chunk_tokens

        context = "\n\n---\n\n".join(
            f"[Source {i+1}]\n{chunk}" for i, chunk in enumerate(context_chunks)
        )

        # Phase 4: Long-context generation
        response = self.llm.invoke(
            f"You have access to {len(context_chunks)} relevant document excerpts.\n"
            f"Read ALL of them carefully before answering.\n\n"
            f"## Document Excerpts\n{context}\n\n"
            f"## Question\n{question}\n\n"
            f"## Instructions\n"
            f"1. Synthesize information across all relevant excerpts\n"
            f"2. Cite source numbers [Source N] for key claims\n"
            f"3. If excerpts contain conflicting information, note the conflict\n"
            f"4. If no excerpt addresses the question, say so"
        )

        return {
            "answer": response.content,
            "chunks_retrieved": len(broad_results),
            "chunks_used": len(context_chunks),
            "context_tokens": int(total_tokens),
        }
```

### Pattern 2: Tiered Context Strategy

```python
class TieredContextStrategy:
    """Use different context strategies based on query type.

    Tier 1 (Simple factual): RAG with top-5 chunks (cheapest)
    Tier 2 (Analytical): RAG with top-20 chunks (balanced)
    Tier 3 (Summarization): Long context with full corpus (most expensive)

    The query classifier routes to the appropriate tier.
    """

    def __init__(self, retriever, reranker, llm, embedding_model, corpus_text: str):
        self.retriever = retriever
        self.reranker = reranker
        self.llm = llm
        self.embedder = embedding_model
        self.corpus_text = corpus_text
        self.corpus_tokens = len(corpus_text.split()) * 1.3

    def query(self, question: str) -> dict:
        """Route to appropriate tier and execute."""
        tier = self._classify_query(question)

        if tier == 1:
            return self._tier1_rag(question)
        elif tier == 2:
            return self._tier2_hybrid(question)
        elif tier == 3:
            return self._tier3_long_context(question)
        else:
            return self._tier2_hybrid(question)  # default

    def _classify_query(self, question: str) -> int:
        """Classify query complexity to determine tier."""
        response = self.llm.invoke(
            "Classify this question's complexity:\n\n"
            f"Question: {question}\n\n"
            "1 = Simple factual (who, what, when, single fact)\n"
            "2 = Analytical (compare, explain, how does X affect Y)\n"
            "3 = Summarization (summarize, overview, main themes)\n\n"
            "Respond with just the number: 1, 2, or 3"
        )
        try:
            return int(response.content.strip())
        except ValueError:
            return 2

    def _tier1_rag(self, question: str) -> dict:
        """Standard RAG: top-5 chunks."""
        query_emb = self.embedder.embed([question])[0]
        results = self.retriever.query(query_emb, top_k=5)
        context = "\n\n".join(r.get("content", "") for r in results)
        response = self.llm.invoke(
            f"Answer based on the context:\n\n{context}\n\nQuestion: {question}"
        )
        return {"answer": response.content, "tier": 1, "chunks_used": 5}

    def _tier2_hybrid(self, question: str) -> dict:
        """Hybrid: retrieve broadly, read deeply."""
        query_emb = self.embedder.embed([question])[0]
        broad = self.retriever.query(query_emb, top_k=50)
        reranked = self.reranker.rerank(
            question, [r.get("content", "") for r in broad], top_k=20
        )
        context = "\n\n---\n\n".join(doc for doc, _ in reranked)
        response = self.llm.invoke(
            f"Read all excerpts and answer:\n\n{context}\n\nQuestion: {question}"
        )
        return {"answer": response.content, "tier": 2, "chunks_used": len(reranked)}

    def _tier3_long_context(self, question: str) -> dict:
        """Full long context: stuff entire corpus."""
        if self.corpus_tokens > 180_000:
            return self._tier2_hybrid(question)  # fallback if too large

        response = self.llm.invoke(
            f"Read the following document collection and answer the question.\n\n"
            f"## Documents\n{self.corpus_text}\n\n"
            f"## Question\n{question}"
        )
        return {
            "answer": response.content,
            "tier": 3,
            "context_tokens": int(self.corpus_tokens),
        }
```

### Pattern 3: Map-Reduce with Long Context Windows

```python
class MapReduceHybrid:
    """For very large corpora: RAG retrieves sections, map-reduce over them.

    When the corpus is too large for even a single context window but
    the query requires broad coverage:

    1. RAG retrieves top-100 chunks (covering many topics)
    2. Group chunks by subtopic
    3. MAP: Generate partial answers from each group (long context per group)
    4. REDUCE: Synthesize partial answers into final answer

    This is the same pattern as GraphRAG's global search but using
    RAG retrieval instead of community summaries.
    """

    def __init__(self, retriever, reranker, llm, embedding_model):
        self.retriever = retriever
        self.reranker = reranker
        self.llm = llm
        self.embedder = embedding_model

    def query(
        self, question: str, broad_k: int = 100, groups: int = 5
    ) -> dict:
        """Execute map-reduce hybrid query."""
        # Retrieve broadly
        query_emb = self.embedder.embed([question])[0]
        results = self.retriever.query(query_emb, top_k=broad_k)

        # Group chunks by similarity clustering
        chunk_groups = self._cluster_chunks(results, groups)

        # MAP: generate partial answer from each group
        partial_answers = []
        for i, group in enumerate(chunk_groups):
            context = "\n\n".join(r.get("content", "") for r in group)
            partial = self.llm.invoke(
                f"Extract information relevant to this question from the excerpts.\n\n"
                f"Excerpts:\n{context}\n\n"
                f"Question: {question}\n\n"
                f"If no relevant information, respond with NO_RELEVANT_INFO."
            )
            if "NO_RELEVANT_INFO" not in partial.content:
                partial_answers.append(partial.content)

        if not partial_answers:
            return {"answer": "No relevant information found.", "tier": "map_reduce"}

        # REDUCE: synthesize
        partials_text = "\n\n---\n\n".join(
            f"[Partial {i+1}]\n{pa}" for i, pa in enumerate(partial_answers)
        )
        final = self.llm.invoke(
            f"Synthesize these partial answers into a comprehensive response.\n\n"
            f"{partials_text}\n\n"
            f"Question: {question}"
        )

        return {
            "answer": final.content,
            "tier": "map_reduce",
            "chunks_retrieved": len(results),
            "groups": len(chunk_groups),
            "relevant_groups": len(partial_answers),
        }

    def _cluster_chunks(
        self, results: list[dict], n_groups: int
    ) -> list[list[dict]]:
        """Cluster chunks into groups by similarity."""
        # Simple round-robin grouping (use K-means on embeddings in production)
        groups: list[list[dict]] = [[] for _ in range(n_groups)]
        for i, result in enumerate(results):
            groups[i % n_groups].append(result)
        return groups
```

---

## Cost Comparison: Standard RAG vs Hybrid

```python
def compare_hybrid_cost(
    corpus_tokens: int,
    queries_per_day: int,
) -> dict:
    """Compare costs of standard RAG, hybrid, and full long context."""
    # Standard RAG: 5 chunks * 500 tokens = 2,500 context tokens
    rag_context = 2500
    # Hybrid: 20 chunks * 500 tokens = 10,000 context tokens
    hybrid_context = 10000
    # Long context: full corpus
    lc_context = corpus_tokens

    sonnet = PRICING_2025["claude_sonnet"]

    rag_cost_per_query = (rag_context / 1e6) * sonnet.input_per_1m + (500 / 1e6) * sonnet.output_per_1m
    hybrid_cost_per_query = (hybrid_context / 1e6) * sonnet.input_per_1m + (500 / 1e6) * sonnet.output_per_1m
    lc_cost_per_query = (lc_context / 1e6) * sonnet.cached_input_per_1m + (500 / 1e6) * sonnet.output_per_1m

    return {
        "standard_rag": {
            "context_tokens": rag_context,
            "cost_per_query": round(rag_cost_per_query, 5),
            "monthly": round(rag_cost_per_query * queries_per_day * 30 + 50, 2),
        },
        "hybrid": {
            "context_tokens": hybrid_context,
            "cost_per_query": round(hybrid_cost_per_query, 5),
            "monthly": round(hybrid_cost_per_query * queries_per_day * 30 + 50, 2),
        },
        "long_context_cached": {
            "context_tokens": lc_context,
            "cost_per_query": round(lc_cost_per_query, 5),
            "monthly": round(lc_cost_per_query * queries_per_day * 30, 2),
        },
    }
```

---

## When to Use Each Hybrid Pattern

| Pattern | Best For | Context Budget | Cost |
|---------|---------|---------------|------|
| RAG Pre-Filter + Long Read | Analytical queries on large corpora | 10K-50K tokens | Medium |
| Tiered Context | Mixed query types with varying complexity | Varies by tier | Optimized |
| Map-Reduce | Global/summary queries on huge corpora | Multiple 20K windows | High |

---

## Common Pitfalls

1. **Using too many chunks in hybrid.** 20 high-quality chunks beat 100 mediocre ones. The reranker is critical -- it ensures only relevant chunks enter the long context.

2. **Not accounting for reranker latency.** Reranking 50 chunks adds 200-500ms. Include this in latency budgets.

3. **Skipping the query classifier.** Simple factual queries do not need 20 chunks. Route them to standard RAG (top-5) to save cost.

4. **Map-reduce with too many groups.** Each group requires a separate LLM call. 5 groups means 5 MAP calls + 1 REDUCE call = 6 total. Keep groups under 10.

5. **Not using prompt caching for repeated corpus sections.** If many queries hit the same document sections, prompt caching reduces hybrid costs by 50-80%.

---

## References

- Anthropic prompt caching: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- Lost in the Middle: Liu et al. (2023)
- GraphRAG global search (map-reduce pattern): Edge et al. (2024)
