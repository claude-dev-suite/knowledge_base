# Self-RAG Implementation in LangGraph

## Overview

Self-RAG (Self-Reflective Retrieval-Augmented Generation) adds reflection loops to RAG: after retrieval, the system grades document relevance; after generation, it grades faithfulness and answer quality. If any check fails, the system retries with a different strategy (web search, query rewriting, regeneration). This document provides a complete, runnable Self-RAG implementation in LangGraph with all nodes, edges, and grading logic.

---

## Architecture

```
        START
          |
     [Route Question]
      /            \
  retrieve     web_search
      |              |
 [Grade Documents]   |
   /        \        |
 relevant  not_relevant
   |          |      |
   |     web_search  |
   |          |      |
   +----+-----+------+
        |
    [Generate]
        |
  [Grade: Hallucination?]
      /          \
  grounded    not_grounded
      |              |
  [Grade: Useful?]  [Regenerate] ---> (max 3 retries)
      /       \
  useful    not_useful
      |          |
    END     web_search + retry
```

---

## Complete Implementation

### State Definition

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END

class SelfRAGState(TypedDict):
    question: str
    documents: list[str]
    generation: str
    web_search: Literal["yes", "no"]
    generation_count: int
```

### LLM and Tool Setup

```python
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain_community.tools.tavily_search import TavilySearchResults
from pydantic import BaseModel, Field

# LLM
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Embeddings and vector store
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma(
    embedding_function=embeddings,
    persist_directory="./chroma_db",
)
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

# Web search tool
web_search_tool = TavilySearchResults(max_results=3)


# --- Structured output schemas for grading ---

class RouteQuery(BaseModel):
    """Route a user query to the most relevant datasource."""
    datasource: Literal["vectorstore", "web_search"] = Field(
        description="Route to 'vectorstore' for questions about indexed documentation, "
                    "or 'web_search' for current events and recent information."
    )

class GradeDocuments(BaseModel):
    """Grade whether a retrieved document is relevant to the question."""
    binary_score: Literal["yes", "no"] = Field(
        description="Is the document relevant to the question? 'yes' or 'no'"
    )

class GradeHallucination(BaseModel):
    """Grade whether the generation is grounded in the documents."""
    binary_score: Literal["yes", "no"] = Field(
        description="Is the answer grounded in / supported by the documents? 'yes' or 'no'"
    )

class GradeAnswer(BaseModel):
    """Grade whether the answer is useful for the question."""
    binary_score: Literal["yes", "no"] = Field(
        description="Does the answer address the question? 'yes' or 'no'"
    )

# Structured LLM wrappers
route_llm = llm.with_structured_output(RouteQuery)
doc_grader = llm.with_structured_output(GradeDocuments)
hallucination_grader = llm.with_structured_output(GradeHallucination)
answer_grader = llm.with_structured_output(GradeAnswer)
```

### Node Functions

```python
def route_question(state: SelfRAGState) -> dict:
    """Route the question to vectorstore or web search."""
    question = state["question"]

    result = route_llm.invoke(
        f"Route this question. If it is about indexed technical documentation "
        f"(databases, programming, frameworks), route to 'vectorstore'. "
        f"If it requires current information or is about recent events, "
        f"route to 'web_search'.\n\n"
        f"Question: {question}"
    )

    if result.datasource == "web_search":
        return {"web_search": "yes"}
    return {"web_search": "no"}


def retrieve(state: SelfRAGState) -> dict:
    """Retrieve documents from the vector store."""
    question = state["question"]
    docs = retriever.invoke(question)
    return {"documents": [doc.page_content for doc in docs]}


def web_search(state: SelfRAGState) -> dict:
    """Search the web for relevant information."""
    question = state["question"]
    results = web_search_tool.invoke({"query": question})

    web_docs = [r["content"] for r in results if "content" in r]
    # Append to existing documents (if any from previous retrieval)
    existing = state.get("documents", [])
    return {"documents": existing + web_docs}


def grade_documents(state: SelfRAGState) -> dict:
    """Grade retrieved documents for relevance. Filter out irrelevant ones."""
    question = state["question"]
    documents = state["documents"]

    relevant_docs = []
    web_search_needed = False

    for doc in documents:
        result = doc_grader.invoke(
            f"Is this document relevant to the question?\n\n"
            f"Question: {question}\n\n"
            f"Document: {doc[:500]}"  # truncate for grading efficiency
        )

        if result.binary_score == "yes":
            relevant_docs.append(doc)

    # If fewer than 2 relevant documents, trigger web search
    if len(relevant_docs) < 2:
        web_search_needed = True

    return {
        "documents": relevant_docs,
        "web_search": "yes" if web_search_needed else "no",
    }


def generate(state: SelfRAGState) -> dict:
    """Generate an answer using the retrieved documents."""
    question = state["question"]
    documents = state["documents"]
    generation_count = state.get("generation_count", 0)

    context = "\n\n---\n\n".join(documents)

    response = llm.invoke(
        f"You are an assistant for question-answering tasks. "
        f"Use the following pieces of retrieved context to answer the question. "
        f"If you don't know the answer, say that you don't know. "
        f"Keep the answer concise but thorough.\n\n"
        f"Context:\n{context}\n\n"
        f"Question: {question}\n\n"
        f"Answer:"
    )

    return {
        "generation": response.content,
        "generation_count": generation_count + 1,
    }


def grade_generation_vs_documents(state: SelfRAGState) -> dict:
    """Check if the generation is grounded in the documents (hallucination check)."""
    documents = state["documents"]
    generation = state["generation"]

    context = "\n\n".join(documents[:3])  # use top 3 for efficiency

    result = hallucination_grader.invoke(
        f"Assess whether the following answer is grounded in / supported by "
        f"the provided documents. Consider whether the key claims in the answer "
        f"can be traced back to the documents.\n\n"
        f"Documents:\n{context}\n\n"
        f"Answer: {generation}"
    )

    return {}  # grading result is used in conditional edge


def grade_generation_vs_question(state: SelfRAGState) -> dict:
    """Check if the generation is useful for answering the question."""
    question = state["question"]
    generation = state["generation"]

    result = answer_grader.invoke(
        f"Does this answer address the question? "
        f"Consider completeness and accuracy.\n\n"
        f"Question: {question}\n\n"
        f"Answer: {generation}"
    )

    return {}  # grading result is used in conditional edge
```

### Routing Functions

```python
def route_after_question(state: SelfRAGState) -> str:
    """Route after initial question analysis."""
    if state.get("web_search") == "yes":
        return "web_search"
    return "retrieve"


def route_after_grading(state: SelfRAGState) -> str:
    """Route after document grading."""
    if state.get("web_search") == "yes":
        return "web_search"
    return "generate"


def route_after_hallucination_check(state: SelfRAGState) -> str:
    """Route after checking if generation is grounded."""
    documents = state["documents"]
    generation = state["generation"]

    # Check hallucination
    context = "\n\n".join(documents[:3])
    result = hallucination_grader.invoke(
        f"Is this answer grounded in the documents?\n\n"
        f"Documents:\n{context}\n\n"
        f"Answer: {generation}"
    )

    if result.binary_score == "yes":
        # Grounded -- now check if it is useful
        return "check_usefulness"

    # Not grounded -- regenerate (up to max retries)
    if state.get("generation_count", 0) >= 3:
        return "finish"  # give up
    return "regenerate"


def route_after_usefulness_check(state: SelfRAGState) -> str:
    """Route after checking if the answer is useful."""
    question = state["question"]
    generation = state["generation"]

    result = answer_grader.invoke(
        f"Does this answer address the question?\n\n"
        f"Question: {question}\n\n"
        f"Answer: {generation}"
    )

    if result.binary_score == "yes":
        return "finish"

    # Not useful -- try web search for more context
    if state.get("generation_count", 0) >= 3:
        return "finish"
    return "web_search_retry"
```

### Build the Graph

```python
# Create graph
graph = StateGraph(SelfRAGState)

# Add nodes
graph.add_node("route_question", route_question)
graph.add_node("retrieve", retrieve)
graph.add_node("web_search", web_search)
graph.add_node("grade_documents", grade_documents)
graph.add_node("generate", generate)

# Connect nodes with edges
graph.add_edge(START, "route_question")

# Route from question analysis
graph.add_conditional_edges(
    "route_question",
    route_after_question,
    {
        "retrieve": "retrieve",
        "web_search": "web_search",
    },
)

# After retrieval, grade documents
graph.add_edge("retrieve", "grade_documents")

# After grading, either generate or web search
graph.add_conditional_edges(
    "grade_documents",
    route_after_grading,
    {
        "generate": "generate",
        "web_search": "web_search",
    },
)

# After web search, generate
graph.add_edge("web_search", "generate")

# After generation, check hallucination then usefulness
graph.add_conditional_edges(
    "generate",
    route_after_hallucination_check,
    {
        "check_usefulness": "check_usefulness",
        "regenerate": "generate",   # retry generation
        "finish": END,
    },
)

# Usefulness check node (just a passthrough -- routing function does the work)
graph.add_node("check_usefulness", lambda state: {})

graph.add_conditional_edges(
    "check_usefulness",
    route_after_usefulness_check,
    {
        "finish": END,
        "web_search_retry": "web_search",
    },
)

# Compile
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
app = graph.compile(checkpointer=memory)
```

### Run the Self-RAG System

```python
# Simple question (routes to vectorstore)
config = {"configurable": {"thread_id": "test_1"}}
result = app.invoke(
    {"question": "How do I create an HNSW index in pgvector?"},
    config=config,
)
print(result["generation"])

# Current events question (routes to web search)
config = {"configurable": {"thread_id": "test_2"}}
result = app.invoke(
    {"question": "What are the latest pgvector release features?"},
    config=config,
)
print(result["generation"])
```

---

## Tracing and Debugging

### Verbose Execution

```python
# Stream events to see each step
config = {"configurable": {"thread_id": "debug_1"}}

for event in app.stream(
    {"question": "What are HNSW index parameters?"},
    config=config,
    stream_mode="updates",
):
    for node_name, output in event.items():
        print(f"\n--- Node: {node_name} ---")
        for key, value in output.items():
            if isinstance(value, list):
                print(f"  {key}: [{len(value)} items]")
            elif isinstance(value, str) and len(value) > 100:
                print(f"  {key}: {value[:100]}...")
            else:
                print(f"  {key}: {value}")
```

### State Inspection

```python
# Inspect state at any point
state = app.get_state(config)
print(f"Current state: {state.values}")
print(f"Next node: {state.next}")

# State history
for state_snapshot in app.get_state_history(config):
    print(f"Node: {state_snapshot.metadata.get('source', 'unknown')}")
    print(f"  question: {state_snapshot.values.get('question', '')[:50]}")
    print(f"  documents: {len(state_snapshot.values.get('documents', []))} docs")
    print(f"  generation: {state_snapshot.values.get('generation', '')[:50]}")
```

---

## Optimizing Self-RAG

### Reduce LLM Calls

The grading nodes each make an LLM call. For 5 documents, that is 5 grading calls + 1 hallucination check + 1 usefulness check = 7 additional LLM calls per query. To reduce costs:

```python
# Batch document grading into a single call
def grade_documents_batch(state: SelfRAGState) -> dict:
    """Grade all documents in a single LLM call."""
    question = state["question"]
    documents = state["documents"]

    docs_text = "\n\n".join(
        f"[Doc {i+1}]: {doc[:300]}" for i, doc in enumerate(documents)
    )

    response = llm.invoke(
        f"Grade each document for relevance to the question. "
        f"Return a comma-separated list of 'yes' or 'no' for each document.\n\n"
        f"Question: {question}\n\n"
        f"Documents:\n{docs_text}\n\n"
        f"Grades (comma-separated):"
    )

    grades = [g.strip().lower() for g in response.content.split(",")]
    relevant_docs = [
        doc for doc, grade in zip(documents, grades) if grade == "yes"
    ]

    return {
        "documents": relevant_docs,
        "web_search": "yes" if len(relevant_docs) < 2 else "no",
    }
```

### Use a Smaller Model for Grading

```python
# Use a cheaper model for grading decisions
grading_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Use a stronger model for generation
generation_llm = ChatOpenAI(model="gpt-4o", temperature=0)
```

### Skip Grading for High-Confidence Retrievals

```python
def grade_documents_with_threshold(state: SelfRAGState) -> dict:
    """Only grade documents below a similarity threshold."""
    question = state["question"]
    # Re-retrieve with scores
    docs_with_scores = vectorstore.similarity_search_with_relevance_scores(
        question, k=5
    )

    # High-confidence documents (score > 0.8) skip grading
    relevant_docs = []
    needs_grading = []

    for doc, score in docs_with_scores:
        if score > 0.8:
            relevant_docs.append(doc.page_content)
        else:
            needs_grading.append(doc.page_content)

    # Grade only borderline documents
    for doc in needs_grading:
        result = doc_grader.invoke(
            f"Is this document relevant?\n\n"
            f"Question: {question}\n\nDocument: {doc[:500]}"
        )
        if result.binary_score == "yes":
            relevant_docs.append(doc)

    return {
        "documents": relevant_docs,
        "web_search": "yes" if len(relevant_docs) < 2 else "no",
    }
```

---

## Metrics and Evaluation

### Tracking Self-RAG Decisions

```python
class SelfRAGMetrics:
    def __init__(self):
        self.total_queries = 0
        self.web_search_triggered = 0
        self.docs_filtered = 0
        self.hallucinations_caught = 0
        self.regenerations = 0
        self.avg_generation_count = 0

    def record(self, final_state):
        self.total_queries += 1
        if "web_search" in str(final_state.get("route_taken", [])):
            self.web_search_triggered += 1
        self.avg_generation_count = (
            (self.avg_generation_count * (self.total_queries - 1) +
             final_state.get("generation_count", 1)) / self.total_queries
        )

    def report(self):
        print(f"Total queries: {self.total_queries}")
        print(f"Web search triggered: {self.web_search_triggered} "
              f"({self.web_search_triggered/max(self.total_queries,1)*100:.1f}%)")
        print(f"Avg generations per query: {self.avg_generation_count:.2f}")
```

---

## Common Pitfalls

1. **Not capping retry loops**: without a generation_count limit, hallucination detection + regeneration can loop forever. Always check `generation_count >= 3` in routing.

2. **Grading every document individually**: 5 documents = 5 LLM calls just for grading. Batch grading into a single call to reduce latency and cost.

3. **Using the same prompt for grading and generation**: the generation prompt should focus on answering; the grading prompt should focus on binary yes/no assessment. Different prompts for different purposes.

4. **Not handling empty retrieval**: if the vector store returns no results, grade_documents receives an empty list and should immediately route to web search.

5. **Overly strict grading**: if your hallucination grader is too strict, every answer triggers regeneration. Calibrate the grading prompt and test with sample outputs.

6. **Not logging the full trace**: in production, log every node's input/output for debugging. When a user reports a bad answer, you need to trace which node made the wrong decision.

---

## References

- Self-RAG paper: Asai, A. et al. "Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection." ICLR 2024. https://arxiv.org/abs/2310.11511
- Corrective RAG: Yan, S. et al. "Corrective Retrieval Augmented Generation." 2024. https://arxiv.org/abs/2401.15884
- LangGraph Self-RAG tutorial: https://langchain-ai.github.io/langgraph/tutorials/rag/langgraph_self_rag/
