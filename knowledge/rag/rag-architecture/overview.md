# RAG Architecture

## Overview / TL;DR

Retrieval-Augmented Generation (RAG) augments LLM prompts with relevant documents fetched from an external knowledge base at query time, grounding the model's output in verifiable facts. RAG architectures exist on a maturity spectrum from Naive RAG (simple retrieve-and-generate) through Advanced RAG (query transformations, re-ranking, hybrid search) to Agentic RAG (autonomous agents that plan retrieval strategies, iterate, and self-correct). Choosing the right maturity level depends on accuracy requirements, latency budget, corpus complexity, and engineering capacity. This document provides a comprehensive architecture guide with component diagrams, maturity-level definitions, decision criteria, and a production-ready Python scaffold.

---

## When to Build RAG

RAG is the right pattern when:

1. **Knowledge changes frequently.** Fine-tuning is expensive to repeat; RAG uses a live index.
2. **Answers must cite sources.** RAG naturally produces traceable provenance (chunk + metadata).
3. **Corpus is large and specialized.** The LLM's parametric knowledge is insufficient or stale.
4. **Hallucination tolerance is low.** Grounding in retrieved text reduces confabulation.
5. **Multi-tenancy is required.** Different users can query different document collections with the same model.

RAG is NOT the right pattern when:

- The task is purely generative (creative writing, brainstorming) with no factual grounding needed.
- The entire knowledge base fits in the model's context window and updates infrequently.
- Sub-10ms latency is required (retrieval adds 50-500ms).
- The domain is well-covered by the model's training data and accuracy is non-critical.

---

## Component Diagram

```
                        RAG Pipeline Architecture
 
  User Query
      |
      v
  +-------------------+
  | Query Processing   |  <-- Query transformation, expansion, routing
  +-------------------+
      |
      v
  +-------------------+     +-------------------+
  | Retriever          | --> | Vector Database    |  <-- HNSW/IVF index
  | (Embedding Search) |     | (Pinecone/Qdrant/ |
  +-------------------+     |  Weaviate/Chroma)  |
      |                      +-------------------+
      |                              ^
      |                              |
      |                     +-------------------+
      |                     | Ingestion Pipeline |  <-- Chunking, embedding
      |                     | (offline/batch)    |      metadata extraction
      |                     +-------------------+
      |                              ^
      |                              |
      |                     +-------------------+
      |                     | Document Sources   |  <-- PDFs, APIs, DBs
      |                     +-------------------+
      v
  +-------------------+
  | Re-ranker          |  <-- Cross-encoder or LLM-based re-scoring
  +-------------------+
      |
      v
  +-------------------+
  | Context Assembly   |  <-- Token budget, dedup, ordering
  +-------------------+
      |
      v
  +-------------------+
  | LLM Generation     |  <-- System prompt + retrieved context + query
  +-------------------+
      |
      v
  +-------------------+
  | Post-processing    |  <-- Citation extraction, guardrails, caching
  +-------------------+
      |
      v
  Response + Sources
```

---

## Maturity Levels

### Level 1: Naive RAG

The simplest architecture. Embed the query, retrieve top-k chunks via cosine similarity, stuff them into the prompt, and generate.

**Pipeline**: Query --> Embed --> Retrieve Top-K --> Stuff into Prompt --> Generate

**Characteristics**:
- Single embedding model, single vector store
- No query transformation
- No re-ranking
- Fixed top-k (typically 3-5)
- Context stuffed in order of similarity score

**Typical accuracy**: 40-60% on complex QA benchmarks (varies by domain).

**When sufficient**: Internal FAQ bots, simple documentation search, prototypes, demos.

```python
"""Naive RAG -- minimal viable pipeline."""

import anthropic
from sentence_transformers import SentenceTransformer
import chromadb

# --- Ingestion (offline) ---
embed_model = SentenceTransformer("BAAI/bge-base-en-v1.5")
chroma = chromadb.PersistentClient(path="./chroma_db")
collection = chroma.get_or_create_collection(
    name="docs",
    metadata={"hnsw:space": "cosine"},
)


def ingest_documents(docs: list[dict]):
    """Ingest documents with metadata."""
    texts = [d["text"] for d in docs]
    embeddings = embed_model.encode(texts).tolist()
    ids = [d["id"] for d in docs]
    metadatas = [{"source": d["source"]} for d in docs]
    collection.add(
        documents=texts,
        embeddings=embeddings,
        ids=ids,
        metadatas=metadatas,
    )


# --- Query (online) ---
client = anthropic.Anthropic()


def naive_rag_query(question: str, top_k: int = 5) -> str:
    query_embedding = embed_model.encode(question).tolist()
    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=top_k,
    )
    context = "\n\n---\n\n".join(results["documents"][0])
    sources = results["metadatas"][0]

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        system="Answer the question using ONLY the provided context. "
               "If the context doesn't contain the answer, say so. "
               "Cite the source for each claim.",
        messages=[
            {
                "role": "user",
                "content": f"Context:\n{context}\n\nQuestion: {question}",
            }
        ],
    )
    return response.content[0].text
```

### Level 2: Advanced RAG

Adds pre-retrieval and post-retrieval optimizations around the same core retrieve-generate loop.

**Pipeline**: Query --> Transform --> Retrieve (Hybrid) --> Re-rank --> Filter --> Assemble Context --> Generate --> Validate

**Key additions over Naive RAG**:

| Stage | Technique | Impact |
|-------|-----------|--------|
| Pre-retrieval | Query rewriting, HyDE, multi-query | +10-25% recall |
| Retrieval | Hybrid search (vector + BM25) | +5-15% recall on keyword-heavy queries |
| Post-retrieval | Cross-encoder re-ranking | +5-15% precision |
| Post-retrieval | Contextual compression | Better token efficiency |
| Generation | Structured prompts with citations | Better faithfulness |

**Typical accuracy**: 65-85% on complex QA benchmarks.

**When needed**: Customer-facing products, compliance-sensitive domains, multi-document reasoning.

```python
"""Advanced RAG -- hybrid search + re-ranking + query transformation."""

import anthropic
from sentence_transformers import SentenceTransformer, CrossEncoder
import chromadb
from rank_bm25 import BM25Okapi
import numpy as np


class AdvancedRAG:
    def __init__(self):
        self.embed_model = SentenceTransformer("BAAI/bge-base-en-v1.5")
        self.reranker = CrossEncoder("BAAI/bge-reranker-v2-m3")
        self.client = anthropic.Anthropic()
        self.chroma = chromadb.PersistentClient(path="./chroma_db")
        self.collection = self.chroma.get_or_create_collection("docs")
        self.bm25 = None
        self.bm25_corpus = []

    def build_bm25_index(self, documents: list[str]):
        """Build BM25 index for keyword search."""
        tokenized = [doc.lower().split() for doc in documents]
        self.bm25 = BM25Okapi(tokenized)
        self.bm25_corpus = documents

    def rewrite_query(self, question: str) -> str:
        """Use LLM to produce a better search query."""
        response = self.client.messages.create(
            model="claude-haiku-4-20250514",
            max_tokens=200,
            messages=[
                {
                    "role": "user",
                    "content": (
                        f"Rewrite this question as a concise search query "
                        f"optimized for semantic search. Return ONLY the "
                        f"rewritten query, nothing else.\n\n"
                        f"Question: {question}"
                    ),
                }
            ],
        )
        return response.content[0].text.strip()

    def generate_multi_queries(self, question: str, n: int = 3) -> list[str]:
        """Generate multiple search queries for broader recall."""
        response = self.client.messages.create(
            model="claude-haiku-4-20250514",
            max_tokens=300,
            messages=[
                {
                    "role": "user",
                    "content": (
                        f"Generate {n} different search queries that would "
                        f"help answer this question from different angles. "
                        f"Return one query per line, nothing else.\n\n"
                        f"Question: {question}"
                    ),
                }
            ],
        )
        queries = response.content[0].text.strip().split("\n")
        return [q.strip() for q in queries if q.strip()][:n]

    def hybrid_retrieve(
        self,
        question: str,
        vector_k: int = 20,
        bm25_k: int = 20,
        alpha: float = 0.7,
    ) -> list[dict]:
        """Hybrid search: weighted combination of vector + BM25 scores."""
        # Vector search
        query_emb = self.embed_model.encode(question).tolist()
        vector_results = self.collection.query(
            query_embeddings=[query_emb],
            n_results=vector_k,
            include=["documents", "metadatas", "distances"],
        )

        # BM25 search
        bm25_scores = self.bm25.get_scores(question.lower().split())
        bm25_top = np.argsort(bm25_scores)[::-1][:bm25_k]

        # Normalize scores to [0, 1]
        vector_docs = {}
        for i, doc in enumerate(vector_results["documents"][0]):
            score = 1 - vector_results["distances"][0][i]  # cosine distance to similarity
            vector_docs[doc] = {
                "vector_score": score,
                "metadata": vector_results["metadatas"][0][i],
            }

        bm25_max = max(bm25_scores) if max(bm25_scores) > 0 else 1.0
        for idx in bm25_top:
            doc = self.bm25_corpus[idx]
            norm_score = bm25_scores[idx] / bm25_max
            if doc in vector_docs:
                vector_docs[doc]["bm25_score"] = norm_score
            else:
                vector_docs[doc] = {
                    "vector_score": 0.0,
                    "bm25_score": norm_score,
                    "metadata": {},
                }

        # Fuse scores
        results = []
        for doc, scores in vector_docs.items():
            fused = (
                alpha * scores.get("vector_score", 0.0)
                + (1 - alpha) * scores.get("bm25_score", 0.0)
            )
            results.append({
                "text": doc,
                "score": fused,
                "metadata": scores.get("metadata", {}),
            })

        results.sort(key=lambda x: x["score"], reverse=True)
        return results

    def rerank(
        self, question: str, candidates: list[dict], top_k: int = 5
    ) -> list[dict]:
        """Re-rank candidates using a cross-encoder."""
        pairs = [(question, c["text"]) for c in candidates]
        scores = self.reranker.predict(pairs)
        for i, candidate in enumerate(candidates):
            candidate["rerank_score"] = float(scores[i])
        candidates.sort(key=lambda x: x["rerank_score"], reverse=True)
        return candidates[:top_k]

    def query(self, question: str) -> dict:
        """Full Advanced RAG pipeline."""
        # 1. Query transformation
        rewritten = self.rewrite_query(question)

        # 2. Multi-query for broader recall
        queries = [rewritten] + self.generate_multi_queries(question, n=2)

        # 3. Hybrid retrieval across all queries
        all_candidates = {}
        for q in queries:
            results = self.hybrid_retrieve(q, vector_k=15, bm25_k=10)
            for r in results:
                key = r["text"][:100]  # dedup key
                if key not in all_candidates or r["score"] > all_candidates[key]["score"]:
                    all_candidates[key] = r

        candidates = list(all_candidates.values())

        # 4. Re-rank
        top_chunks = self.rerank(question, candidates, top_k=5)

        # 5. Assemble context with token budget
        context_parts = []
        for i, chunk in enumerate(top_chunks):
            source = chunk["metadata"].get("source", "unknown")
            context_parts.append(
                f"[Source {i+1}: {source}]\n{chunk['text']}"
            )
        context = "\n\n---\n\n".join(context_parts)

        # 6. Generate with citations
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1500,
            system=(
                "Answer the question using ONLY the provided context. "
                "For each claim, cite the source number in brackets like [Source 1]. "
                "If the context is insufficient, explicitly state what is missing."
            ),
            messages=[
                {
                    "role": "user",
                    "content": f"Context:\n{context}\n\nQuestion: {question}",
                }
            ],
        )

        return {
            "answer": response.content[0].text,
            "sources": [c["metadata"] for c in top_chunks],
            "rewritten_query": rewritten,
        }
```

### Level 3: Agentic RAG

The LLM becomes an autonomous agent that plans its retrieval strategy, executes multiple retrieval steps, evaluates intermediate results, and self-corrects. It may route queries to different data sources, reformulate queries based on initial results, and decide when it has enough information to answer.

**Pipeline**: Query --> Agent Plans Strategy --> [Retrieve --> Evaluate --> Decide: More retrieval? Route elsewhere? Refine query?] --> Generate --> Self-check --> Respond

**Key additions over Advanced RAG**:

| Capability | Description |
|------------|-------------|
| Planning | Agent decomposes complex queries into sub-questions |
| Tool use | Agent calls retrieval as a tool alongside SQL, web search, calculators |
| Self-reflection | Agent evaluates whether retrieved context is sufficient |
| Iterative retrieval | Agent performs multiple retrieval rounds, refining each time |
| Multi-source routing | Agent routes sub-queries to appropriate data sources |
| Adaptive strategy | Agent changes approach based on intermediate results |

**Typical accuracy**: 80-95% on complex QA benchmarks (with higher latency and cost).

**When needed**: Research assistants, complex multi-hop questions, heterogeneous data sources, mission-critical accuracy.

```python
"""Agentic RAG -- tool-using agent with iterative retrieval."""

import anthropic
import json
from typing import Any


class AgenticRAG:
    def __init__(self, rag_pipeline):
        """Wrap an existing RAG pipeline as an agent tool."""
        self.rag = rag_pipeline
        self.client = anthropic.Anthropic()
        self.tools = [
            {
                "name": "search_knowledge_base",
                "description": (
                    "Search the internal knowledge base for information. "
                    "Use specific, focused queries for best results."
                ),
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "query": {
                            "type": "string",
                            "description": "The search query",
                        },
                        "filter_source": {
                            "type": "string",
                            "description": "Optional: filter by source document type",
                        },
                    },
                    "required": ["query"],
                },
            },
            {
                "name": "evaluate_sufficiency",
                "description": (
                    "Evaluate whether the information gathered so far is "
                    "sufficient to answer the original question completely."
                ),
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "original_question": {"type": "string"},
                        "gathered_info": {"type": "string"},
                    },
                    "required": ["original_question", "gathered_info"],
                },
            },
        ]

    def _execute_tool(self, name: str, input_data: dict) -> str:
        if name == "search_knowledge_base":
            results = self.rag.hybrid_retrieve(
                input_data["query"], vector_k=10, bm25_k=5
            )
            top = results[:3]
            return json.dumps([
                {"text": r["text"][:500], "source": r["metadata"].get("source", "")}
                for r in top
            ])
        elif name == "evaluate_sufficiency":
            return json.dumps({
                "note": "Agent should use its own judgment based on gathered information."
            })
        return json.dumps({"error": f"Unknown tool: {name}"})

    def query(self, question: str, max_iterations: int = 5) -> dict:
        """Run the agent loop with tool-based retrieval."""
        messages = [
            {
                "role": "user",
                "content": (
                    f"Answer this question thoroughly using the search tools. "
                    f"Break complex questions into sub-queries. Search multiple "
                    f"times if needed. Cite sources.\n\n"
                    f"Question: {question}"
                ),
            }
        ]

        all_sources = []
        for iteration in range(max_iterations):
            response = self.client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=2000,
                system=(
                    "You are a research assistant with access to a knowledge base. "
                    "Use the search tool to find relevant information. "
                    "Search multiple times with different queries if needed. "
                    "When you have enough information, provide a comprehensive answer."
                ),
                tools=self.tools,
                messages=messages,
            )

            # Check if the agent wants to use tools
            if response.stop_reason == "tool_use":
                tool_results = []
                for block in response.content:
                    if block.type == "tool_use":
                        result = self._execute_tool(block.name, block.input)
                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": result,
                        })
                        if block.name == "search_knowledge_base":
                            all_sources.append(block.input["query"])

                messages.append({"role": "assistant", "content": response.content})
                messages.append({"role": "user", "content": tool_results})
            else:
                # Agent has finished -- extract the final answer
                answer = ""
                for block in response.content:
                    if hasattr(block, "text"):
                        answer += block.text
                return {
                    "answer": answer,
                    "iterations": iteration + 1,
                    "search_queries": all_sources,
                }

        return {
            "answer": "Max iterations reached. Partial answer may be available.",
            "iterations": max_iterations,
            "search_queries": all_sources,
        }
```

---

## RAG Maturity Assessment

Use this checklist to assess your current pipeline and identify the next improvement:

| # | Capability | Naive | Advanced | Agentic |
|---|-----------|-------|----------|---------|
| 1 | Basic vector retrieval | Yes | Yes | Yes |
| 2 | Metadata filtering | -- | Yes | Yes |
| 3 | Hybrid search (vector + keyword) | -- | Yes | Yes |
| 4 | Query rewriting/expansion | -- | Yes | Yes |
| 5 | Cross-encoder re-ranking | -- | Yes | Yes |
| 6 | Multi-query retrieval | -- | Yes | Yes |
| 7 | Citation generation | -- | Yes | Yes |
| 8 | Iterative retrieval | -- | -- | Yes |
| 9 | Query routing to data sources | -- | -- | Yes |
| 10 | Self-reflection and correction | -- | -- | Yes |
| 11 | Sub-question decomposition | -- | -- | Yes |
| 12 | Tool-augmented retrieval | -- | -- | Yes |

---

## Full Production Scaffold

Below is a production-grade scaffold that supports all three maturity levels via configuration.

```python
"""Production RAG scaffold with configurable maturity levels."""

from dataclasses import dataclass, field
from enum import Enum
from typing import Optional
import time
import logging

logger = logging.getLogger(__name__)


class RAGLevel(Enum):
    NAIVE = "naive"
    ADVANCED = "advanced"
    AGENTIC = "agentic"


@dataclass
class RAGConfig:
    """Configuration for the RAG pipeline."""
    level: RAGLevel = RAGLevel.ADVANCED
    embedding_model: str = "BAAI/bge-base-en-v1.5"
    generation_model: str = "claude-sonnet-4-20250514"
    reranker_model: str = "BAAI/bge-reranker-v2-m3"

    # Retrieval settings
    vector_top_k: int = 20
    bm25_top_k: int = 20
    final_top_k: int = 5
    hybrid_alpha: float = 0.7  # weight for vector vs BM25

    # Token budget
    max_context_tokens: int = 4000
    max_output_tokens: int = 1500

    # Agentic settings
    max_agent_iterations: int = 5

    # Performance
    enable_caching: bool = True
    cache_ttl_seconds: int = 3600

    # Evaluation
    enable_tracing: bool = True
    log_retrievals: bool = True


@dataclass
class RAGResult:
    """Structured result from the RAG pipeline."""
    answer: str
    sources: list[dict] = field(default_factory=list)
    latency_ms: float = 0.0
    tokens_used: int = 0
    retrieval_count: int = 0
    cache_hit: bool = False
    metadata: dict = field(default_factory=dict)


class RAGPipeline:
    """Unified RAG pipeline supporting Naive, Advanced, and Agentic levels."""

    def __init__(self, config: RAGConfig):
        self.config = config
        self._init_components()

    def _init_components(self):
        """Initialize components based on configuration level."""
        import anthropic
        from sentence_transformers import SentenceTransformer

        self.client = anthropic.Anthropic()
        self.embed_model = SentenceTransformer(self.config.embedding_model)

        if self.config.level in (RAGLevel.ADVANCED, RAGLevel.AGENTIC):
            from sentence_transformers import CrossEncoder
            self.reranker = CrossEncoder(self.config.reranker_model)

        if self.config.enable_caching:
            self._cache = {}  # Replace with Redis in production

    def query(self, question: str) -> RAGResult:
        """Route to the appropriate pipeline based on config level."""
        start = time.time()

        if self.config.level == RAGLevel.NAIVE:
            result = self._naive_pipeline(question)
        elif self.config.level == RAGLevel.ADVANCED:
            result = self._advanced_pipeline(question)
        else:
            result = self._agentic_pipeline(question)

        result.latency_ms = (time.time() - start) * 1000

        if self.config.log_retrievals:
            logger.info(
                "RAG query completed",
                extra={
                    "level": self.config.level.value,
                    "latency_ms": result.latency_ms,
                    "retrieval_count": result.retrieval_count,
                    "sources": len(result.sources),
                },
            )

        return result

    def _naive_pipeline(self, question: str) -> RAGResult:
        """Level 1: simple embed-retrieve-generate."""
        chunks = self._vector_search(question, k=self.config.final_top_k)
        answer = self._generate(question, chunks)
        return RAGResult(
            answer=answer,
            sources=[c["metadata"] for c in chunks],
            retrieval_count=1,
        )

    def _advanced_pipeline(self, question: str) -> RAGResult:
        """Level 2: transform, hybrid retrieve, re-rank, generate."""
        rewritten = self._rewrite_query(question)
        candidates = self._hybrid_search(rewritten)
        top_chunks = self._rerank(question, candidates)
        answer = self._generate(question, top_chunks)
        return RAGResult(
            answer=answer,
            sources=[c["metadata"] for c in top_chunks],
            retrieval_count=1,
            metadata={"rewritten_query": rewritten},
        )

    def _agentic_pipeline(self, question: str) -> RAGResult:
        """Level 3: agent loop with iterative retrieval."""
        # Implementation follows the AgenticRAG pattern above
        # Omitted for brevity -- see the AgenticRAG class
        raise NotImplementedError("See AgenticRAG class for full implementation")

    # --- Building blocks (implement per your vector DB) ---

    def _vector_search(self, query: str, k: int) -> list[dict]:
        """Override with your vector database client."""
        raise NotImplementedError

    def _hybrid_search(self, query: str) -> list[dict]:
        """Override with vector + BM25 fusion."""
        raise NotImplementedError

    def _rewrite_query(self, question: str) -> str:
        """LLM-based query rewriting."""
        response = self.client.messages.create(
            model="claude-haiku-4-20250514",
            max_tokens=200,
            messages=[{
                "role": "user",
                "content": (
                    f"Rewrite this question as a search query optimized for "
                    f"semantic similarity search. Return ONLY the query.\n\n"
                    f"Question: {question}"
                ),
            }],
        )
        return response.content[0].text.strip()

    def _rerank(self, question: str, candidates: list[dict]) -> list[dict]:
        """Cross-encoder re-ranking."""
        pairs = [(question, c["text"]) for c in candidates]
        scores = self.reranker.predict(pairs)
        for i, c in enumerate(candidates):
            c["rerank_score"] = float(scores[i])
        candidates.sort(key=lambda x: x["rerank_score"], reverse=True)
        return candidates[: self.config.final_top_k]

    def _generate(self, question: str, chunks: list[dict]) -> str:
        """Generate answer from retrieved context."""
        context_parts = []
        for i, chunk in enumerate(chunks):
            source = chunk.get("metadata", {}).get("source", "unknown")
            context_parts.append(f"[Source {i+1}: {source}]\n{chunk['text']}")
        context = "\n\n---\n\n".join(context_parts)

        response = self.client.messages.create(
            model=self.config.generation_model,
            max_tokens=self.config.max_output_tokens,
            system=(
                "Answer using ONLY the provided context. "
                "Cite sources as [Source N]. "
                "State explicitly if the context is insufficient."
            ),
            messages=[{
                "role": "user",
                "content": f"Context:\n{context}\n\nQuestion: {question}",
            }],
        )
        return response.content[0].text
```

---

## Choosing Between Maturity Levels

| Factor | Naive | Advanced | Agentic |
|--------|-------|----------|---------|
| **Accuracy** | 40-60% | 65-85% | 80-95% |
| **Latency (p50)** | 500ms-1s | 1-3s | 3-15s |
| **Cost per query** | $0.001-0.005 | $0.005-0.02 | $0.02-0.10 |
| **Eng effort to build** | 1-2 days | 1-2 weeks | 2-6 weeks |
| **Maintenance burden** | Low | Medium | High |
| **Best for** | Prototypes, simple QA | Production products | Research, complex reasoning |

Start with Naive RAG to validate the use case, then add Advanced RAG components one at a time (re-ranking first -- it gives the biggest lift for the least effort), and move to Agentic only when you have evidence that iterative retrieval is needed.

---

## Common Pitfalls

1. **Starting with Agentic RAG.** Over-engineering from day one adds latency, cost, and debugging complexity before you understand your data. Start naive, measure, then improve.
2. **Ignoring chunk quality.** Architecture cannot compensate for poor chunking. If the right information is not in a retrievable chunk, no amount of re-ranking or agent iteration will help.
3. **Not measuring retrieval separately from generation.** When answers are wrong, you need to know whether retrieval failed (wrong chunks) or generation failed (hallucinated despite good chunks). Evaluate both stages independently.
4. **Treating RAG as stateless.** Production RAG needs conversation history management, caching, and session-aware retrieval. A user's follow-up question depends on what was already discussed.
5. **Skipping hybrid search.** Pure vector search misses keyword-specific matches (product IDs, error codes, exact names). BM25 catches what embeddings miss.
6. **Using the same embedding model for all content types.** Code, legal text, and conversational text have different optimal embedding models. Consider domain-specific models or fine-tuned embeddings.

---

## References

- Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks" (2020) -- https://arxiv.org/abs/2005.11401
- Gao et al., "Retrieval-Augmented Generation for Large Language Models: A Survey" (2024) -- https://arxiv.org/abs/2312.10997
- LangChain RAG documentation -- https://python.langchain.com/docs/concepts/rag/
- LlamaIndex RAG guide -- https://docs.llamaindex.ai/en/stable/understanding/rag/
- Anthropic RAG cookbook -- https://docs.anthropic.com/en/docs/build-with-claude/retrieval-augmented-generation
