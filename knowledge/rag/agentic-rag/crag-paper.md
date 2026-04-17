# Corrective RAG (CRAG)

## TL;DR

Corrective RAG (Yan et al., 2024) adds a retrieval evaluation step to the RAG pipeline: after retrieving documents, a lightweight evaluator scores retrieval confidence. If confidence is high, the retrieved documents are used directly. If confidence is low, the system falls back to web search. If confidence is ambiguous, both sources are combined. CRAG also includes a knowledge refinement step that extracts only the relevant portions of retrieved documents, reducing noise in the generation context. This article covers the paper's methodology, the confidence evaluation mechanism, the knowledge refinement process, and a complete LangGraph implementation.

---

## The CRAG Framework

### Architecture Overview

```
Query
  |
  v
[Retriever] --> top-K documents
  |
  v
[Retrieval Evaluator] --> confidence score per document
  |
  +---> Confidence = CORRECT (score >= upper threshold)
  |     |
  |     v
  |   [Knowledge Refinement] --> extract relevant spans
  |     |
  |     v
  |   [Generate with refined knowledge]
  |
  +---> Confidence = INCORRECT (score <= lower threshold)
  |     |
  |     v
  |   [Web Search] --> web results
  |     |
  |     v
  |   [Knowledge Refinement] --> extract relevant spans
  |     |
  |     v
  |   [Generate with web knowledge]
  |
  +---> Confidence = AMBIGUOUS (between thresholds)
        |
        v
      [Knowledge Refinement on retrieved docs]
      + [Web Search]
        |
        v
      [Generate with combined knowledge]
```

### Three Confidence Actions

| Action | Trigger | Behavior |
|--------|---------|----------|
| CORRECT | At least one document scores above the upper threshold | Use retrieved documents (after refinement) |
| INCORRECT | All documents score below the lower threshold | Discard retrieved docs, use web search |
| AMBIGUOUS | Some documents score between thresholds | Combine refined retrieved docs with web search |

---

## Retrieval Evaluator

### How It Works

The retrieval evaluator is a lightweight model (the paper uses a fine-tuned T5-large) that takes a (query, document) pair and outputs a relevance score.

In practice, you can approximate this with:
1. A cross-encoder reranker (scores naturally serve as confidence)
2. An LLM-as-judge (cheaper but slower)
3. A fine-tuned classifier on your domain data

### Cross-Encoder as Confidence Evaluator

```python
from sentence_transformers import CrossEncoder
import numpy as np


class RetrievalConfidenceEvaluator:
    """Evaluate retrieval confidence using a cross-encoder."""

    def __init__(
        self,
        model_name: str = "cross-encoder/ms-marco-MiniLM-L-6-v2",
        correct_threshold: float = 0.7,
        incorrect_threshold: float = 0.3,
    ):
        self.model = CrossEncoder(model_name)
        self.correct_threshold = correct_threshold
        self.incorrect_threshold = incorrect_threshold

    def evaluate(
        self, query: str, documents: list[str]
    ) -> tuple[str, list[float]]:
        """
        Evaluate retrieval confidence.

        Returns:
            action: "correct", "incorrect", or "ambiguous"
            scores: per-document confidence scores
        """
        pairs = [(query, doc) for doc in documents]
        raw_scores = self.model.predict(pairs)

        # Normalize to [0, 1] using sigmoid
        scores = 1.0 / (1.0 + np.exp(-raw_scores))

        max_score = max(scores)
        if max_score >= self.correct_threshold:
            return "correct", scores.tolist()
        elif max_score <= self.incorrect_threshold:
            return "incorrect", scores.tolist()
        else:
            return "ambiguous", scores.tolist()


# Usage
evaluator = RetrievalConfidenceEvaluator()
action, scores = evaluator.evaluate(
    "How to configure RLS in PostgreSQL?",
    [
        "PostgreSQL RLS enables row-level access control...",
        "MySQL user management uses GRANT statements...",
    ],
)
print(f"Action: {action}, Scores: {scores}")
```

### LLM-as-Judge Evaluator

```python
import anthropic

client = anthropic.Anthropic()


def llm_confidence_evaluator(
    query: str, documents: list[str]
) -> tuple[str, list[float]]:
    """Evaluate retrieval confidence using Claude."""
    doc_list = "\n\n".join(
        f"Document {i+1}:\n{doc[:500]}" for i, doc in enumerate(documents)
    )

    response = client.messages.create(
        model="claude-3-5-haiku-20241022",
        max_tokens=200,
        messages=[{
            "role": "user",
            "content": (
                f"Rate how well each document answers this query on a scale of 0-10.\n"
                f"Query: {query}\n\n{doc_list}\n\n"
                f"Respond in format: doc1_score,doc2_score,...\n"
                f"Then on a new line, write CORRECT if any doc scores >= 7, "
                f"INCORRECT if all score <= 3, or AMBIGUOUS otherwise."
            ),
        }],
    )

    lines = response.content[0].text.strip().split("\n")
    scores = [float(s.strip()) / 10.0 for s in lines[0].split(",")]
    action = lines[-1].strip().lower()
    if action not in ("correct", "incorrect", "ambiguous"):
        action = "ambiguous"  # Default to safe option
    return action, scores
```

---

## Knowledge Refinement

### The Problem

Even when retrieved documents are relevant, they contain irrelevant passages, boilerplate, and noise. Stuffing full documents into the LLM context dilutes the signal.

### CRAG's Solution: Decompose-then-Filter

1. **Decompose** each document into fine-grained knowledge strips (sentences or small passages)
2. **Filter** strips by relevance to the query
3. **Recompose** filtered strips into a clean context

```python
import re


def decompose_document(document: str) -> list[str]:
    """Split document into sentence-level knowledge strips."""
    # Split on sentence boundaries
    sentences = re.split(r'(?<=[.!?])\s+', document)
    # Filter out very short fragments
    return [s.strip() for s in sentences if len(s.strip()) > 20]


def filter_strips(
    query: str,
    strips: list[str],
    model,
    relevance_threshold: float = 0.5,
) -> list[str]:
    """Filter knowledge strips by relevance to the query."""
    if not strips:
        return []

    pairs = [(query, strip) for strip in strips]
    scores = model.predict(pairs)

    # Normalize scores
    import numpy as np
    normalized = 1.0 / (1.0 + np.exp(-scores))

    relevant = [
        strip for strip, score in zip(strips, normalized)
        if score >= relevance_threshold
    ]
    return relevant


def refine_knowledge(
    query: str,
    documents: list[str],
    confidence_scores: list[float],
    model,
    min_doc_confidence: float = 0.3,
) -> str:
    """
    CRAG knowledge refinement: decompose documents into strips,
    filter by relevance, recompose.
    """
    all_relevant_strips = []

    for doc, conf in zip(documents, confidence_scores):
        if conf < min_doc_confidence:
            continue  # Skip very low-confidence documents

        strips = decompose_document(doc)
        relevant = filter_strips(query, strips, model)
        all_relevant_strips.extend(relevant)

    if not all_relevant_strips:
        return ""

    return "\n".join(all_relevant_strips)
```

### LLM-Based Refinement

For higher quality (at higher cost), use an LLM to extract relevant information:

```python
def llm_refine_knowledge(query: str, document: str) -> str:
    """Use LLM to extract query-relevant information from a document."""
    response = client.messages.create(
        model="claude-3-5-haiku-20241022",
        max_tokens=500,
        messages=[{
            "role": "user",
            "content": (
                f"Extract ONLY the sentences from this document that are directly "
                f"relevant to answering the query. If nothing is relevant, respond "
                f"with 'NONE'.\n\n"
                f"Query: {query}\n\n"
                f"Document:\n{document[:2000]}"
            ),
        }],
    )
    text = response.content[0].text.strip()
    return "" if text == "NONE" else text
```

---

## Web Search Fallback

When retrieval confidence is low, CRAG falls back to web search. The web results go through the same knowledge refinement process.

```python
def web_search_fallback(query: str, num_results: int = 5) -> list[str]:
    """Search the web using Tavily (or similar API)."""
    from tavily import TavilyClient

    tavily = TavilyClient(api_key="your-tavily-key")
    results = tavily.search(
        query=query,
        max_results=num_results,
        search_depth="advanced",
        include_raw_content=True,
    )

    return [
        result.get("raw_content") or result.get("content", "")
        for result in results.get("results", [])
    ]


# Alternative: using Serper
def serper_search(query: str, num_results: int = 5) -> list[str]:
    """Search the web using Serper API."""
    import requests

    response = requests.post(
        "https://google.serper.dev/search",
        headers={"X-API-KEY": "your-serper-key"},
        json={"q": query, "num": num_results},
    )
    data = response.json()
    return [
        item.get("snippet", "") for item in data.get("organic", [])
    ]
```

---

## Complete LangGraph CRAG Implementation

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Literal
from sentence_transformers import CrossEncoder
import numpy as np


# --- State ---
class CRAGState(TypedDict):
    query: str
    documents: list[dict]          # Retrieved documents
    confidence_scores: list[float]
    action: str                     # "correct", "incorrect", "ambiguous"
    refined_knowledge: str          # After knowledge refinement
    web_results: list[str]          # Web search results
    generation: str                 # Final answer


# --- Shared resources ---
cross_encoder = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")
CORRECT_THRESHOLD = 0.7
INCORRECT_THRESHOLD = 0.3


# --- Nodes ---
def retrieve(state: CRAGState) -> CRAGState:
    """Retrieve documents from the knowledge base."""
    docs = retriever.invoke(state["query"])
    return {
        **state,
        "documents": [
            {"content": d.page_content, "metadata": d.metadata}
            for d in docs
        ],
    }


def evaluate_confidence(state: CRAGState) -> CRAGState:
    """Evaluate retrieval confidence using cross-encoder."""
    doc_texts = [d["content"] for d in state["documents"]]
    pairs = [(state["query"], text) for text in doc_texts]
    raw_scores = cross_encoder.predict(pairs)
    scores = (1.0 / (1.0 + np.exp(-raw_scores))).tolist()

    max_score = max(scores) if scores else 0.0
    if max_score >= CORRECT_THRESHOLD:
        action = "correct"
    elif max_score <= INCORRECT_THRESHOLD:
        action = "incorrect"
    else:
        action = "ambiguous"

    return {**state, "confidence_scores": scores, "action": action}


def refine_knowledge_node(state: CRAGState) -> CRAGState:
    """Refine retrieved documents by extracting relevant strips."""
    doc_texts = [d["content"] for d in state["documents"]]
    refined = refine_knowledge(
        state["query"],
        doc_texts,
        state["confidence_scores"],
        cross_encoder,
    )
    return {**state, "refined_knowledge": refined}


def web_search_node(state: CRAGState) -> CRAGState:
    """Perform web search as fallback."""
    results = web_search_fallback(state["query"], num_results=5)
    return {**state, "web_results": results}


def combine_sources(state: CRAGState) -> CRAGState:
    """Combine refined KB knowledge with web results (ambiguous case)."""
    # Refine web results too
    web_text = "\n\n".join(state.get("web_results", []))
    combined = state.get("refined_knowledge", "") + "\n\n" + web_text
    return {**state, "refined_knowledge": combined.strip()}


def generate_answer(state: CRAGState) -> CRAGState:
    """Generate the final answer."""
    context = state.get("refined_knowledge", "")
    if not context:
        context = "\n\n".join(state.get("web_results", []))

    response = llm.invoke(
        f"Answer the query using the provided context. "
        f"If the context is insufficient, say so.\n\n"
        f"Context:\n{context[:4000]}\n\n"
        f"Query: {state['query']}"
    )
    return {**state, "generation": response.content}


# --- Routing ---
def route_by_confidence(state: CRAGState) -> str:
    action = state["action"]
    if action == "correct":
        return "refine"
    elif action == "incorrect":
        return "web_search"
    else:
        return "ambiguous"


# --- Build Graph ---
graph = StateGraph(CRAGState)

# Add nodes
graph.add_node("retrieve", retrieve)
graph.add_node("evaluate", evaluate_confidence)
graph.add_node("refine", refine_knowledge_node)
graph.add_node("web_search", web_search_node)
graph.add_node("combine", combine_sources)
graph.add_node("generate", generate_answer)

# Set entry
graph.set_entry_point("retrieve")

# Add edges
graph.add_edge("retrieve", "evaluate")

graph.add_conditional_edges(
    "evaluate",
    route_by_confidence,
    {
        "refine": "refine",         # High confidence -> refine KB docs
        "web_search": "web_search",  # Low confidence -> web search
        "ambiguous": "refine",       # Medium -> refine first, then web
    },
)

# Correct path: refine -> generate
graph.add_edge("refine", "generate")

# For ambiguous: after refine, also do web search and combine
# (In practice, handle this with a conditional in refine or a separate path)

# Incorrect path: web search -> generate
graph.add_edge("web_search", "generate")

graph.add_edge("generate", END)

crag_app = graph.compile()


# --- Run ---
result = crag_app.invoke({
    "query": "What are the new features in PostgreSQL 17?",
    "documents": [],
    "confidence_scores": [],
    "action": "",
    "refined_knowledge": "",
    "web_results": [],
    "generation": "",
})
print(result["generation"])
```

### Handling the Ambiguous Path

The ambiguous case needs both refinement and web search. Here is a more complete handling:

```python
def route_by_confidence_v2(state: CRAGState) -> str:
    return state["action"]


# Build graph with full ambiguous handling
graph_v2 = StateGraph(CRAGState)

graph_v2.add_node("retrieve", retrieve)
graph_v2.add_node("evaluate", evaluate_confidence)
graph_v2.add_node("refine", refine_knowledge_node)
graph_v2.add_node("web_search", web_search_node)
graph_v2.add_node("combine", combine_sources)
graph_v2.add_node("generate_from_kb", generate_answer)
graph_v2.add_node("generate_from_web", generate_answer)
graph_v2.add_node("generate_combined", generate_answer)

graph_v2.set_entry_point("retrieve")
graph_v2.add_edge("retrieve", "evaluate")

graph_v2.add_conditional_edges(
    "evaluate",
    route_by_confidence_v2,
    {
        "correct": "refine",
        "incorrect": "web_search",
        "ambiguous": "refine",
    },
)

# Correct: refine -> generate
graph_v2.add_edge("refine", "generate_from_kb")
graph_v2.add_edge("generate_from_kb", END)

# Incorrect: web -> generate
graph_v2.add_edge("web_search", "generate_from_web")
graph_v2.add_edge("generate_from_web", END)
```

---

## Threshold Tuning

### How to Set Correct/Incorrect Thresholds

The thresholds depend on your cross-encoder's score distribution and your tolerance for false positives/negatives.

```python
def tune_crag_thresholds(
    eval_queries: list[dict],
    retriever,
    cross_encoder,
    threshold_pairs: list[tuple[float, float]],
) -> dict:
    """
    Find optimal (correct_threshold, incorrect_threshold) pair.

    eval_queries: [{"query": str, "has_answer_in_kb": bool, "expected_answer": str}]
    """
    results = {}

    for correct_t, incorrect_t in threshold_pairs:
        correct_count = 0
        false_web_search = 0  # KB had answer but we went to web
        missed_web_search = 0  # KB had no answer but we used KB

        for eq in eval_queries:
            docs = retriever.invoke(eq["query"])
            doc_texts = [d.page_content for d in docs]
            pairs = [(eq["query"], t) for t in doc_texts]
            scores = 1.0 / (1.0 + np.exp(-cross_encoder.predict(pairs)))
            max_score = max(scores) if len(scores) > 0 else 0.0

            predicted_correct = max_score >= correct_t
            predicted_incorrect = max_score <= incorrect_t

            if eq["has_answer_in_kb"]:
                if predicted_correct:
                    correct_count += 1
                elif predicted_incorrect:
                    false_web_search += 1  # Unnecessary fallback
            else:
                if not predicted_incorrect:
                    missed_web_search += 1  # Should have used web search

        n = len(eval_queries)
        results[(correct_t, incorrect_t)] = {
            "accuracy": correct_count / n,
            "false_web_rate": false_web_search / n,
            "missed_web_rate": missed_web_search / n,
        }
        print(
            f"thresholds=({correct_t:.2f}, {incorrect_t:.2f}): "
            f"accuracy={correct_count/n:.3f} "
            f"false_web={false_web_search/n:.3f} "
            f"missed_web={missed_web_search/n:.3f}"
        )

    return results


# Example threshold pairs to test
pairs = [
    (0.5, 0.2), (0.6, 0.2), (0.7, 0.3),
    (0.7, 0.2), (0.8, 0.3), (0.8, 0.4),
]
tune_crag_thresholds(eval_queries, retriever, cross_encoder, pairs)
```

---

## When to Use CRAG

### Good Fit

1. **Knowledge base has incomplete coverage** -- some queries will need external information
2. **Users ask about current events** that are not in the static KB
3. **High-stakes applications** where returning "I don't know" is better than hallucinating
4. **Mixed query distribution** where some queries are well-covered by the KB and others are not

### Poor Fit

1. **Air-gapped environments** where web search is not available
2. **Highly specialized domains** where web search results are unreliable
3. **Very low latency requirements** -- the confidence evaluation and potential web search add 200-1000ms

---

## Common Pitfalls

1. **Setting thresholds without calibration.** The "correct" and "incorrect" thresholds must be tuned on an eval set for your specific cross-encoder and domain. Default values (0.7/0.3) are a starting point, not a final answer.
2. **Trusting web search results blindly.** Web results can be outdated, incorrect, or manipulated. Apply the same knowledge refinement process to web results.
3. **Not applying knowledge refinement.** Skipping the refinement step and passing full documents to the LLM introduces noise and degrades generation quality.
4. **Using a slow evaluator.** If the confidence evaluation takes 500ms and triggers a web search that takes another 1000ms, total latency can exceed 2 seconds. Use a fast cross-encoder (MiniLM) for confidence evaluation.
5. **Binary thresholding without the ambiguous zone.** Having only "correct" and "incorrect" (no ambiguous) forces a hard decision. The ambiguous zone allows combining both sources, which is safer for borderline cases.
6. **Not rate-limiting web search.** In production, web search APIs have rate limits and costs. Implement caching, deduplication, and rate limiting for the fallback path.

---

## References

- Yan et al. "Corrective Retrieval Augmented Generation" (2024). https://arxiv.org/abs/2401.15884
- LangGraph CRAG tutorial: https://langchain-ai.github.io/langgraph/tutorials/rag/langgraph_crag/
- Tavily search API: https://tavily.com
