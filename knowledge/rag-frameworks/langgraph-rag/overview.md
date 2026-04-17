# LangGraph for RAG

## Overview

LangGraph is a library for building stateful, multi-step LLM applications as directed graphs. For RAG, LangGraph replaces simple retrieve-then-generate chains with state machines that can route queries, evaluate retrieval quality, retry with different strategies, and incorporate human feedback. The key insight: RAG is not a single step but a decision process where the system must judge whether retrieved context is sufficient, whether the generated answer is faithful, and what to do when it is not.

---

## Why State Machines Improve on Simple Chains

### The Problem with Chains

A typical RAG chain is linear:

```
Query -> Retrieve -> Generate -> Answer
```

This works for simple questions with clean data. It fails when:
- Retrieved documents are irrelevant (the system confidently generates a wrong answer)
- The question requires information from multiple sources
- The question should be routed to web search instead of the knowledge base
- The generated answer hallucinates despite having good context
- The user's query is ambiguous and needs clarification

### The State Machine Approach

A LangGraph RAG system models these decisions explicitly:

```
Query -> [Route: KB or Web?]
              |
    KB ------+------ Web
    |                  |
 Retrieve         Web Search
    |                  |
 Grade Docs       Grade Results
    |                  |
 [Relevant?]      [Relevant?]
    |                  |
  Yes: Generate    Yes: Generate
  No:  Web Search  No:  Rephrase + Retry
    |                  |
 Grade Generation     |
    |                  |
 [Faithful?]          |
    |                  |
  Yes: Return         |
  No: Regenerate      |
```

Each box is a **node** (function). Each arrow is an **edge** (transition). Conditional edges implement decision points.

---

## Graph vs Chain Mental Model

| Aspect | Chain (LangChain LCEL) | Graph (LangGraph) |
|---|---|---|
| Structure | Linear pipeline | Directed graph with cycles |
| Control flow | Fixed: A -> B -> C | Conditional: A -> B if X, A -> C if Y |
| State | Passed through pipe | Explicit state object, accumulated across nodes |
| Retries | Must be coded manually | Natural: edge loops back to a previous node |
| Human-in-the-loop | Difficult | Built-in: pause at a node, wait for approval |
| Parallelism | Via RunnableParallel | Multiple edges from one node |
| Debugging | Trace the pipe | Visualize the graph, inspect state at each node |

### When to Use LangGraph vs Simple Chains

**Use a simple chain when:**
- Your RAG is straightforward retrieve-and-generate
- No quality gates needed
- Single data source
- Latency is critical (every node adds latency)

**Use LangGraph when:**
- You need quality checks (relevance grading, hallucination detection)
- Multiple retrieval strategies (KB + web + SQL)
- Complex routing logic (different query types need different handling)
- Human-in-the-loop approval
- You want persistent state (resumable workflows)

---

## Core Concepts

### StateGraph

The main building block. You define a state schema, add nodes, and connect them with edges:

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
from operator import add

class RAGState(TypedDict):
    question: str
    documents: list[str]
    generation: str
    web_search_needed: bool

# Create graph
graph = StateGraph(RAGState)

# Add nodes
graph.add_node("retrieve", retrieve_node)
graph.add_node("generate", generate_node)

# Add edges
graph.add_edge(START, "retrieve")
graph.add_edge("retrieve", "generate")
graph.add_edge("generate", END)

# Compile
app = graph.compile()

# Run
result = app.invoke({"question": "What is pgvector?"})
```

### Nodes

Nodes are Python functions that take the current state and return state updates:

```python
def retrieve_node(state: RAGState) -> dict:
    """Retrieve documents relevant to the question."""
    question = state["question"]
    docs = vectorstore.similarity_search(question, k=5)
    return {"documents": [doc.page_content for doc in docs]}

def generate_node(state: RAGState) -> dict:
    """Generate answer from retrieved documents."""
    question = state["question"]
    context = "\n\n".join(state["documents"])

    response = llm.invoke(
        f"Answer the question based on the context.\n\n"
        f"Context: {context}\n\n"
        f"Question: {question}\n\n"
        f"Answer:"
    )

    return {"generation": response.content}
```

### Conditional Edges

Conditional edges route to different nodes based on the current state:

```python
def route_after_grading(state: RAGState) -> str:
    """Route based on document relevance."""
    if state.get("web_search_needed"):
        return "web_search"
    return "generate"

graph.add_conditional_edges(
    "grade_documents",     # source node
    route_after_grading,   # routing function
    {
        "web_search": "web_search",  # if returns "web_search" -> web_search node
        "generate": "generate",       # if returns "generate" -> generate node
    },
)
```

---

## Minimal RAG Graph

```python
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from typing import TypedDict

# State
class RAGState(TypedDict):
    question: str
    documents: list[str]
    generation: str

# Components
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma(embedding_function=embeddings, persist_directory="./chroma_db")

# Nodes
def retrieve(state: RAGState) -> dict:
    docs = vectorstore.similarity_search(state["question"], k=5)
    return {"documents": [doc.page_content for doc in docs]}

def generate(state: RAGState) -> dict:
    context = "\n\n".join(state["documents"])
    response = llm.invoke(
        f"Answer based on context only.\n\nContext: {context}\n\n"
        f"Question: {state['question']}\n\nAnswer:"
    )
    return {"generation": response.content}

# Build graph
graph = StateGraph(RAGState)
graph.add_node("retrieve", retrieve)
graph.add_node("generate", generate)
graph.add_edge(START, "retrieve")
graph.add_edge("retrieve", "generate")
graph.add_edge("generate", END)

app = graph.compile()

# Run
result = app.invoke({"question": "How do I create an HNSW index?"})
print(result["generation"])
```

---

## Visualization

```python
# Print the graph structure as ASCII
print(app.get_graph().draw_ascii())

# Or as a Mermaid diagram
print(app.get_graph().draw_mermaid())

# Save as PNG (requires graphviz)
from IPython.display import Image
Image(app.get_graph().draw_mermaid_png())
```

---

## Adding Quality Gates

The real power of LangGraph for RAG is adding quality checks. Here is the pattern:

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class RAGState(TypedDict):
    question: str
    documents: list[str]
    generation: str
    documents_relevant: bool
    generation_faithful: bool
    retry_count: int

def grade_documents(state: RAGState) -> dict:
    """Check if retrieved documents are relevant to the question."""
    question = state["question"]
    docs = state["documents"]

    prompt = (
        f"Are these documents relevant to the question?\n\n"
        f"Question: {question}\n\n"
        f"Documents: {docs[:3]}\n\n"
        f"Answer YES or NO."
    )
    response = llm.invoke(prompt)
    relevant = "YES" in response.content.upper()

    return {"documents_relevant": relevant}

def grade_generation(state: RAGState) -> dict:
    """Check if the generation is faithful to the documents."""
    docs = state["documents"]
    generation = state["generation"]

    prompt = (
        f"Is this answer supported by the provided documents? "
        f"Answer YES or NO.\n\n"
        f"Documents: {docs[:3]}\n\n"
        f"Answer: {generation}"
    )
    response = llm.invoke(prompt)
    faithful = "YES" in response.content.upper()

    return {"generation_faithful": faithful}

def route_after_doc_grading(state: RAGState) -> str:
    if state["documents_relevant"]:
        return "generate"
    return "web_search"

def route_after_gen_grading(state: RAGState) -> str:
    if state["generation_faithful"]:
        return "end"
    if state.get("retry_count", 0) >= 2:
        return "end"  # give up after 2 retries
    return "regenerate"

# Build graph with quality gates
graph = StateGraph(RAGState)
graph.add_node("retrieve", retrieve)
graph.add_node("grade_documents", grade_documents)
graph.add_node("generate", generate)
graph.add_node("grade_generation", grade_generation)
graph.add_node("web_search", web_search)
graph.add_node("regenerate", regenerate)

graph.add_edge(START, "retrieve")
graph.add_edge("retrieve", "grade_documents")
graph.add_conditional_edges("grade_documents", route_after_doc_grading,
                            {"generate": "generate", "web_search": "web_search"})
graph.add_edge("web_search", "generate")
graph.add_edge("generate", "grade_generation")
graph.add_conditional_edges("grade_generation", route_after_gen_grading,
                            {"end": END, "regenerate": "regenerate"})
graph.add_edge("regenerate", "grade_generation")

app = graph.compile()
```

See `self-rag-impl.md` for a complete, runnable Self-RAG implementation.

---

## Installation

```bash
pip install langgraph langchain-openai langchain-community
```

LangGraph is part of the LangChain ecosystem but is a separate package. It works with any LangChain-compatible LLM and vector store.

---

## Common Pitfalls

1. **Infinite loops**: conditional edges can create cycles. Always add a retry counter and a maximum iteration limit to prevent infinite loops.

2. **Over-engineering simple RAG**: if your use case is straightforward retrieval + generation with no quality concerns, a simple LangChain chain is sufficient. LangGraph adds latency from extra LLM calls (grading, routing).

3. **Not defining state types properly**: use TypedDict or Pydantic models for state. Untyped state leads to runtime errors when nodes expect keys that were not set.

4. **Grading with the same model that generates**: if the generation model hallucinates, using the same model to grade faithfulness may not catch it. Consider using a different model or a specialized NLI classifier for grading.

5. **Not testing each node independently**: test retrieve, grade, and generate functions in isolation before assembling the graph. Debugging a multi-node graph is harder than debugging individual functions.

---

## References

- LangGraph documentation: https://langchain-ai.github.io/langgraph/
- LangGraph tutorials: https://langchain-ai.github.io/langgraph/tutorials/
- LangGraph RAG examples: https://langchain-ai.github.io/langgraph/tutorials/rag/
- Self-RAG paper: https://arxiv.org/abs/2310.11511
- Corrective RAG paper: https://arxiv.org/abs/2401.15884
