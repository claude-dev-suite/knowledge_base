# Agentic RAG -- Landscape of Patterns

## TL;DR

Agentic RAG extends traditional retrieve-then-generate pipelines with decision-making capabilities: the system can decide whether to retrieve, what to retrieve, whether the retrieved context is sufficient, and whether to iterate. Instead of a fixed linear pipeline (query -> retrieve -> generate), agentic RAG uses an LLM as a controller that orchestrates retrieval, evaluation, and generation steps dynamically. The three foundational papers are Self-RAG (self-reflection tokens), Corrective RAG (confidence-based web search fallback), and Adaptive RAG (routing by query complexity). This overview maps the landscape and connects the patterns.

---

## Why Standard RAG Falls Short

### The Fixed Pipeline Problem

Standard RAG follows a rigid sequence:

```
Query -> Embed -> Retrieve top-K -> Stuff into prompt -> Generate answer
```

This fails in predictable ways:

1. **Over-retrieval**: simple factual queries ("What year was Python released?") do not need retrieval -- the LLM already knows the answer. Retrieval adds latency and sometimes introduces noise.

2. **Under-retrieval**: complex multi-hop questions ("How does the PostgreSQL RLS implementation compare to MySQL's approach, and which is better for multi-tenant SaaS?") need multiple retrieval steps across different topics.

3. **Irrelevant retrieval**: the top-K documents may not actually answer the query. Standard RAG has no mechanism to detect this and try again.

4. **Hallucination despite retrieval**: the LLM may ignore retrieved context or generate claims not supported by it. Standard RAG has no verification step.

5. **No decomposition**: compound queries are sent as-is to the retriever, which may fail to find documents relevant to all sub-questions.

### What Agentic RAG Adds

| Capability | Standard RAG | Agentic RAG |
|-----------|-------------|-------------|
| Decide whether to retrieve | No (always retrieves) | Yes (skip for easy queries) |
| Evaluate retrieval quality | No | Yes (relevance scoring) |
| Retry with different query | No | Yes (query rewriting) |
| Fall back to web search | No | Yes (CRAG pattern) |
| Multi-step retrieval | No | Yes (decompose + iterate) |
| Verify answer against sources | No | Yes (Self-RAG reflection) |
| Route by query complexity | No | Yes (Adaptive RAG) |

---

## The Three Foundational Patterns

### 1. Self-RAG (Self-Reflective RAG)

**Paper**: Asai et al. "Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection" (ICLR 2024)

**Core idea**: train the LLM to emit special reflection tokens that control retrieval and assess generation quality.

```
Input: "What is the capital of France?"

LLM reasoning:
  [Retrieve] = No        (LLM knows this, no retrieval needed)
  [Generate] "Paris is the capital of France."
  [IsUse] = 5            (answer is useful)

Input: "What were the key findings of the 2024 PostgreSQL performance report?"

LLM reasoning:
  [Retrieve] = Yes       (needs external knowledge)
  [Retrieved context]: "The 2024 PostgreSQL report found..."
  [IsRel] = Relevant     (context is relevant to query)
  [Generate] "According to the 2024 report..."
  [IsSup] = Fully Supported  (answer is supported by context)
  [IsUse] = 5            (answer is useful)
```

**When to use**: when you need verifiable answers with source attribution and want the system to self-assess answer quality.

See `self-rag-paper.md` for full details.

### 2. Corrective RAG (CRAG)

**Paper**: Yan et al. "Corrective Retrieval Augmented Generation" (2024)

**Core idea**: evaluate retrieval confidence and fall back to web search when the retrieved documents are not confident enough.

```
Query -> Retrieve -> Evaluate confidence
  |
  +--> High confidence:  Use retrieved docs directly
  +--> Medium confidence: Combine retrieved + web search
  +--> Low confidence:   Discard retrieved, use web search only
```

**When to use**: when your knowledge base may not cover all queries and you need a fallback for out-of-domain questions.

See `crag-paper.md` for full details.

### 3. Adaptive RAG

**Paper**: Jeong et al. "Adaptive-RAG: Learning to Adapt Retrieval-Augmented Large Language Models through Question Complexity" (NAACL 2024)

**Core idea**: classify query complexity first, then route to the appropriate retrieval strategy.

```
Query -> Complexity Classifier
  |
  +--> Simple:  No retrieval (LLM generates directly)
  +--> Moderate: Single retrieval step
  +--> Complex:  Multi-step retrieval with iterative refinement
```

**When to use**: when your query distribution spans simple factual to complex multi-hop questions and you want to optimize both latency and quality.

See `adaptive-rag.md` for full details.

---

## How the Patterns Compose

The three patterns are not mutually exclusive. They address different aspects of the RAG pipeline and can be combined:

```
Query
  |
  v
[Adaptive RAG Classifier]
  |
  +--> Simple: LLM generates directly (no retrieval)
  |
  +--> Moderate:
  |     |
  |     v
  |   [Retrieve from KB]
  |     |
  |     v
  |   [CRAG Confidence Check]
  |     |
  |     +--> High: Generate with context
  |     +--> Low:  Web search fallback
  |     |
  |     v
  |   [Self-RAG Reflection]
  |     |
  |     +--> Supported: Return answer
  |     +--> Unsupported: Retry with rewritten query
  |
  +--> Complex:
        |
        v
      [Decompose into sub-questions]
        |
        v
      [For each sub-question: Retrieve + CRAG check]
        |
        v
      [Synthesize sub-answers]
        |
        v
      [Self-RAG Reflection on final answer]
```

---

## Implementation with LangGraph

LangGraph is the standard framework for implementing agentic RAG because its graph-based execution model maps naturally to the decision trees above.

### Basic Agentic RAG Graph

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Literal
import operator


class AgenticRAGState(TypedDict):
    query: str
    documents: list[str]
    generation: str
    retrieval_confidence: float
    is_relevant: bool
    is_supported: bool
    route: str
    steps: list[str]


def classify_query(state: AgenticRAGState) -> AgenticRAGState:
    """Determine query complexity and routing."""
    query = state["query"]
    # Use LLM to classify
    response = llm.invoke(
        f"Classify this query as 'simple', 'moderate', or 'complex':\n{query}"
    )
    route = response.content.strip().lower()
    return {**state, "route": route, "steps": ["classified"]}


def retrieve(state: AgenticRAGState) -> AgenticRAGState:
    """Retrieve documents from the knowledge base."""
    docs = retriever.invoke(state["query"])
    doc_texts = [doc.page_content for doc in docs]
    return {**state, "documents": doc_texts, "steps": state["steps"] + ["retrieved"]}


def evaluate_relevance(state: AgenticRAGState) -> AgenticRAGState:
    """Check if retrieved documents are relevant to the query."""
    # Score relevance using LLM
    response = llm.invoke(
        f"Are these documents relevant to the query?\n"
        f"Query: {state['query']}\n"
        f"Documents: {state['documents'][:3]}\n"
        f"Answer 'relevant' or 'irrelevant'."
    )
    is_relevant = "relevant" in response.content.lower()
    return {**state, "is_relevant": is_relevant}


def generate(state: AgenticRAGState) -> AgenticRAGState:
    """Generate answer from retrieved context."""
    context = "\n\n".join(state["documents"][:5])
    response = llm.invoke(
        f"Answer based on the context:\n{context}\n\nQuery: {state['query']}"
    )
    return {**state, "generation": response.content}


def direct_generate(state: AgenticRAGState) -> AgenticRAGState:
    """Generate answer without retrieval (simple queries)."""
    response = llm.invoke(f"Answer this question: {state['query']}")
    return {**state, "generation": response.content}


def web_search_fallback(state: AgenticRAGState) -> AgenticRAGState:
    """Fall back to web search when KB retrieval fails."""
    # Use Tavily, Serper, or similar web search API
    results = web_search.invoke(state["query"])
    doc_texts = [r["content"] for r in results]
    return {**state, "documents": doc_texts, "steps": state["steps"] + ["web_search"]}


def verify_answer(state: AgenticRAGState) -> AgenticRAGState:
    """Verify that the answer is supported by the retrieved context."""
    response = llm.invoke(
        f"Is this answer fully supported by the provided context?\n"
        f"Context: {state['documents'][:3]}\n"
        f"Answer: {state['generation']}\n"
        f"Respond 'supported' or 'unsupported'."
    )
    is_supported = "supported" in response.content.lower()
    return {**state, "is_supported": is_supported}


# Routing functions
def route_by_complexity(state: AgenticRAGState) -> str:
    if state["route"] == "simple":
        return "direct_generate"
    else:
        return "retrieve"


def route_by_relevance(state: AgenticRAGState) -> str:
    if state["is_relevant"]:
        return "generate"
    else:
        return "web_search_fallback"


def route_by_verification(state: AgenticRAGState) -> str:
    if state["is_supported"]:
        return END
    else:
        return "web_search_fallback"


# Build the graph
graph = StateGraph(AgenticRAGState)

graph.add_node("classify", classify_query)
graph.add_node("retrieve", retrieve)
graph.add_node("evaluate_relevance", evaluate_relevance)
graph.add_node("generate", generate)
graph.add_node("direct_generate", direct_generate)
graph.add_node("web_search_fallback", web_search_fallback)
graph.add_node("verify", verify_answer)

graph.set_entry_point("classify")

graph.add_conditional_edges("classify", route_by_complexity, {
    "direct_generate": "direct_generate",
    "retrieve": "retrieve",
})
graph.add_edge("retrieve", "evaluate_relevance")
graph.add_conditional_edges("evaluate_relevance", route_by_relevance, {
    "generate": "generate",
    "web_search_fallback": "web_search_fallback",
})
graph.add_edge("web_search_fallback", "generate")
graph.add_edge("generate", "verify")
graph.add_conditional_edges("verify", route_by_verification, {
    END: END,
    "web_search_fallback": "web_search_fallback",
})
graph.add_edge("direct_generate", END)

app = graph.compile()
```

---

## Other Agentic RAG Patterns

### Tool-Use RAG

The LLM decides which retrieval tools to invoke (multiple vector stores, SQL databases, APIs):

```python
tools = [
    Tool(name="search_docs", func=doc_retriever.invoke, description="Search internal docs"),
    Tool(name="search_code", func=code_retriever.invoke, description="Search codebase"),
    Tool(name="query_db", func=sql_tool.invoke, description="Query the database"),
    Tool(name="web_search", func=web_search.invoke, description="Search the web"),
]

agent = create_react_agent(llm, tools)
result = agent.invoke({"input": "What is the current user count and how does our auth work?"})
# Agent decides: first query_db for user count, then search_docs for auth documentation
```

### Iterative Retrieval-Generation

The LLM generates a partial answer, identifies knowledge gaps, retrieves more, and continues:

```python
def iterative_rag(query: str, max_iterations: int = 3) -> str:
    context_docs = []
    current_query = query

    for i in range(max_iterations):
        # Retrieve
        new_docs = retriever.invoke(current_query)
        context_docs.extend(new_docs)

        # Generate with accumulated context
        context = "\n\n".join([d.page_content for d in context_docs])
        response = llm.invoke(
            f"Context:\n{context}\n\n"
            f"Query: {query}\n\n"
            f"If you have enough information to fully answer, provide the answer. "
            f"If you need more information, respond with NEED_MORE: <specific follow-up query>."
        )

        if "NEED_MORE:" not in response.content:
            return response.content

        # Extract follow-up query for next iteration
        current_query = response.content.split("NEED_MORE:")[1].strip()

    # Max iterations reached, generate best-effort answer
    context = "\n\n".join([d.page_content for d in context_docs])
    return llm.invoke(f"Context:\n{context}\n\nQuery: {query}").content
```

### Multi-Agent RAG

Different specialized agents handle different aspects of the query:

```
User Query
    |
    v
[Router Agent] --> determines which specialist(s) to invoke
    |
    +---> [Code Agent] (searches codebase, reads code)
    +---> [Docs Agent] (searches documentation)
    +---> [Data Agent] (queries databases, APIs)
    |
    v
[Synthesizer Agent] --> combines specialist outputs into final answer
```

---

## Choosing the Right Pattern

| Query Distribution | Recommended Pattern |
|-------------------|-------------------|
| Mostly simple, some complex | Adaptive RAG (skip retrieval for simple) |
| KB coverage is incomplete | CRAG (web search fallback) |
| High-stakes, need verification | Self-RAG (reflection tokens) |
| Multiple data sources (docs + DB + code) | Tool-use RAG |
| Multi-hop reasoning required | Iterative retrieval-generation |
| All of the above | Compose patterns with LangGraph |

### Complexity vs. Benefit

```
Complexity:    Low -------- Medium -------- High
               |            |               |
Patterns:    Standard    CRAG/Self-RAG    Full Adaptive +
             RAG         (single check)   Multi-agent +
                                          Iterative
```

**Start simple**: standard RAG -> add CRAG confidence check -> add adaptive routing -> add Self-RAG verification -> add multi-step iteration. Only add complexity when evaluation shows the simpler approach is insufficient.

---

## Evaluation Considerations

Agentic RAG introduces new failure modes that standard RAG evaluation does not cover:

1. **Routing accuracy**: does the classifier correctly identify simple vs. complex queries?
2. **Retrieval decision quality**: when the system skips retrieval, is that correct?
3. **Fallback trigger accuracy**: does the confidence threshold trigger web search at the right time?
4. **Iteration efficiency**: does iterative retrieval converge in reasonable steps?
5. **Answer verification accuracy**: does Self-RAG reflection correctly identify unsupported claims?

```python
# Extended evaluation metrics for agentic RAG
def evaluate_agentic_rag(eval_set, pipeline):
    results = {
        "routing_accuracy": [],
        "retrieval_decision_accuracy": [],
        "answer_quality": [],
        "avg_retrieval_steps": [],
        "avg_latency_ms": [],
    }

    for example in eval_set:
        output = pipeline.invoke({"query": example["query"]})

        # Check routing
        expected_route = example["expected_route"]
        actual_route = output["route"]
        results["routing_accuracy"].append(actual_route == expected_route)

        # Check answer quality (standard RAGAS)
        results["answer_quality"].append(
            compute_answer_relevancy(output["generation"], example["query"])
        )

        # Track efficiency
        results["avg_retrieval_steps"].append(
            sum(1 for s in output["steps"] if "retriev" in s)
        )
    return {k: sum(v) / len(v) for k, v in results.items()}
```

---

## Common Pitfalls

1. **Over-engineering simple use cases.** If 90% of your queries are straightforward factual questions over a well-curated KB, standard RAG with reranking will outperform a complex agentic setup. Agentic patterns add latency, cost, and failure modes.
2. **Not evaluating routing accuracy.** A misrouted query (complex treated as simple, or vice versa) is worse than no routing at all. Measure routing accuracy separately.
3. **Infinite loops in iterative retrieval.** Always set a max iteration count. If the system cannot find sufficient information in 3-5 iterations, generate a best-effort answer with an uncertainty disclaimer.
4. **LLM-as-judge calibration.** Using an LLM to evaluate relevance or support is powerful but imperfect. Calibrate your confidence thresholds on an eval set, not intuitively.
5. **Ignoring latency.** Each agentic decision point adds 200-500ms (LLM call). A 3-stage agentic pipeline can easily exceed 5 seconds. Budget your latency carefully.
6. **Not providing a fallback.** Every agentic decision point should have a default fallback. If the classifier fails, default to "retrieve". If confidence check fails, default to "use retrieved docs". Never leave the system in a stuck state.

---

## References

- Asai et al. "Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection" (ICLR 2024)
- Yan et al. "Corrective Retrieval Augmented Generation" (2024)
- Jeong et al. "Adaptive-RAG: Learning to Adapt Retrieval-Augmented Large Language Models through Question Complexity" (NAACL 2024)
- LangGraph: https://langchain-ai.github.io/langgraph/
- Gao et al. "Retrieval-Augmented Generation for Large Language Models: A Survey" (2024)
