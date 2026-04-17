# Designing RAG State in LangGraph

## Overview

State design is the foundation of a LangGraph RAG application. The state object carries all data between nodes -- the question, retrieved documents, generated text, grading results, and control flags. Poor state design leads to bugs, lost context, and difficulty adding new features. This guide covers practical patterns for state schema design, conditional edges, checkpointing for persistence, and human-in-the-loop approval nodes.

---

## State Schema with TypedDict

The simplest approach uses Python's TypedDict:

```python
from typing import TypedDict, Annotated
from operator import add

class RAGState(TypedDict):
    # Input
    question: str

    # Retrieval
    documents: list[str]
    document_scores: list[float]

    # Generation
    generation: str

    # Grading / routing
    web_search_needed: bool
    documents_relevant: bool
    generation_grounded: bool
    answer_useful: bool

    # Control flow
    retry_count: int
    route: str  # "retrieve" | "web_search" | "generate" | "done"
```

### Annotated State for Accumulation

By default, each node's return value **replaces** the state field. To **accumulate** values (e.g., appending to a list), use Annotated with a reducer:

```python
from typing import TypedDict, Annotated
from operator import add

class RAGState(TypedDict):
    question: str
    # Each node's "documents" return is APPENDED to the list
    documents: Annotated[list[str], add]
    # Each node's "messages" return is APPENDED
    messages: Annotated[list[dict], add]
    generation: str  # replaced each time (no annotation)
```

```python
# Node returns are accumulated
def retrieve(state: RAGState) -> dict:
    docs = vectorstore.similarity_search(state["question"], k=5)
    return {"documents": [doc.page_content for doc in docs]}

def web_search(state: RAGState) -> dict:
    results = web_search_tool(state["question"])
    # These are APPENDED to existing documents, not replacing them
    return {"documents": [r["content"] for r in results]}
```

### Pydantic State (Validated)

For production, use Pydantic models for validation:

```python
from pydantic import BaseModel, Field
from typing import Literal

class RAGState(BaseModel):
    question: str = Field(description="User's original question")
    documents: list[str] = Field(default_factory=list)
    generation: str = Field(default="")
    web_search_needed: bool = Field(default=False)
    retry_count: int = Field(default=0, ge=0, le=5)
    route: Literal["retrieve", "web_search", "generate", "done"] = Field(default="retrieve")
```

---

## Practical State Patterns

### Pattern 1: Minimal RAG State

For simple retrieve-and-generate:

```python
class SimpleRAGState(TypedDict):
    question: str
    documents: list[str]
    generation: str
```

### Pattern 2: Self-RAG State

For Self-RAG with document grading and generation grading:

```python
class SelfRAGState(TypedDict):
    question: str
    documents: list[str]
    generation: str
    web_search: str             # "yes" or "no"
    documents_relevant: str     # "yes" or "no"
    generation_grounded: str    # "yes" or "no"
    answer_useful: str          # "yes" or "no"
    retry_count: int
```

### Pattern 3: Multi-Source RAG State

For systems that query multiple knowledge sources:

```python
class MultiSourceRAGState(TypedDict):
    question: str
    # Per-source retrieval results
    kb_documents: list[str]
    web_documents: list[str]
    sql_results: list[str]
    # Merged documents after fusion
    merged_documents: list[str]
    # Generation
    generation: str
    # Source attribution
    sources_used: list[str]     # ["kb", "web", "sql"]
```

### Pattern 4: Conversational RAG State

For multi-turn conversations:

```python
class ConversationalRAGState(TypedDict):
    # Current question
    question: str
    # Chat history for context
    chat_history: list[dict]    # [{"role": "user", "content": "..."}, ...]
    # Condensed question (rewritten with chat context)
    condensed_question: str
    # Retrieval and generation
    documents: list[str]
    generation: str
```

```python
def condense_question(state: ConversationalRAGState) -> dict:
    """Rewrite the question incorporating chat history context."""
    if not state.get("chat_history"):
        return {"condensed_question": state["question"]}

    history_text = "\n".join(
        f"{msg['role']}: {msg['content']}" for msg in state["chat_history"][-6:]
    )

    response = llm.invoke(
        f"Given the chat history and follow-up question, "
        f"rewrite the question to be standalone.\n\n"
        f"Chat History:\n{history_text}\n\n"
        f"Follow-up: {state['question']}\n\n"
        f"Standalone question:"
    )

    return {"condensed_question": response.content}
```

---

## Conditional Edges

### Basic Routing

```python
from langgraph.graph import StateGraph, START, END

def decide_to_retrieve_or_search(state: RAGState) -> str:
    """Route based on whether the question needs web search."""
    question = state["question"].lower()

    # Simple heuristic routing
    web_triggers = ["latest", "recent", "current", "today", "2024", "2025"]
    if any(trigger in question for trigger in web_triggers):
        return "web_search"
    return "retrieve"

graph = StateGraph(RAGState)
graph.add_node("retrieve", retrieve)
graph.add_node("web_search", web_search)
graph.add_node("generate", generate)

# Conditional edge from START
graph.add_conditional_edges(
    START,
    decide_to_retrieve_or_search,
    {
        "retrieve": "retrieve",
        "web_search": "web_search",
    },
)
graph.add_edge("retrieve", "generate")
graph.add_edge("web_search", "generate")
graph.add_edge("generate", END)
```

### LLM-Based Routing

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field

class RouteDecision(BaseModel):
    route: str = Field(description="Route: 'vectorstore' or 'web_search'")

route_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0).with_structured_output(
    RouteDecision
)

def route_question(state: RAGState) -> str:
    """Use LLM to decide retrieval strategy."""
    question = state["question"]

    result = route_llm.invoke(
        f"Route this question to either 'vectorstore' (for questions about "
        f"your indexed documentation) or 'web_search' (for current events, "
        f"recent updates, or topics not in your docs).\n\n"
        f"Question: {question}"
    )

    return result.route

graph.add_conditional_edges(
    START,
    route_question,
    {
        "vectorstore": "retrieve",
        "web_search": "web_search",
    },
)
```

### Multi-Step Conditional Flow

```python
def should_retry_or_finish(state: RAGState) -> str:
    """Decide whether to retry generation or return the answer."""
    if state.get("generation_grounded") == "yes" and state.get("answer_useful") == "yes":
        return "finish"

    if state.get("retry_count", 0) >= 3:
        return "finish"  # give up after 3 retries

    if state.get("generation_grounded") != "yes":
        return "regenerate"  # hallucination detected, try again

    if state.get("answer_useful") != "yes":
        return "web_search"  # answer not useful, try web search

    return "finish"

graph.add_conditional_edges(
    "grade_generation",
    should_retry_or_finish,
    {
        "finish": END,
        "regenerate": "generate",
        "web_search": "web_search",
    },
)
```

---

## Checkpointing for Persistence

Checkpointing saves the state at each node, enabling:
- Resuming interrupted workflows
- Time-travel debugging (inspect state at any point)
- Human-in-the-loop (pause, wait for approval, resume)

### In-Memory Checkpointing

```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
app = graph.compile(checkpointer=memory)

# Run with a thread ID (identifies the conversation)
config = {"configurable": {"thread_id": "user_123_session_1"}}
result = app.invoke({"question": "What is pgvector?"}, config=config)

# Later: inspect the state history
states = list(app.get_state_history(config))
for state in states:
    print(f"Node: {state.metadata.get('source', 'unknown')}")
    print(f"State: {state.values}")
```

### SQLite Checkpointing (Persistent)

```python
from langgraph.checkpoint.sqlite import SqliteSaver

# Persistent checkpoint storage
with SqliteSaver.from_conn_string("checkpoints.db") as checkpointer:
    app = graph.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "user_123"}}
    result = app.invoke({"question": "What is HNSW?"}, config=config)

    # This state survives process restart
    # Resume with the same thread_id to continue the conversation
```

### PostgreSQL Checkpointing

```python
from langgraph.checkpoint.postgres import PostgresSaver

with PostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost/langgraph"
) as checkpointer:
    app = graph.compile(checkpointer=checkpointer)
```

---

## Human-in-the-Loop Approval Nodes

LangGraph can pause execution at a node, wait for human input, then resume:

```python
from langgraph.graph import StateGraph, START, END

class ApprovalRAGState(TypedDict):
    question: str
    documents: list[str]
    generation: str
    approved: bool

def generate(state: ApprovalRAGState) -> dict:
    context = "\n\n".join(state["documents"])
    response = llm.invoke(
        f"Context: {context}\n\nQuestion: {state['question']}\n\nAnswer:"
    )
    return {"generation": response.content}

def check_approval(state: ApprovalRAGState) -> str:
    if state.get("approved"):
        return "end"
    return "regenerate"

graph = StateGraph(ApprovalRAGState)
graph.add_node("retrieve", retrieve)
graph.add_node("generate", generate)
graph.add_node("human_review", lambda state: {})  # no-op, just a pause point

graph.add_edge(START, "retrieve")
graph.add_edge("retrieve", "generate")
graph.add_edge("generate", "human_review")
graph.add_conditional_edges(
    "human_review",
    check_approval,
    {"end": END, "regenerate": "generate"},
)

# Compile with interrupt_before the human review node
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
app = graph.compile(
    checkpointer=memory,
    interrupt_before=["human_review"],  # pause before this node
)

# Run -- will pause at human_review
config = {"configurable": {"thread_id": "review_session"}}
result = app.invoke({"question": "Explain HNSW indexing"}, config=config)

# The graph is paused. Show the generation to a human:
current_state = app.get_state(config)
print(f"Generated answer: {current_state.values['generation']}")
print("Approve? (y/n)")

# Human approves:
app.update_state(config, {"approved": True})

# Resume execution
result = app.invoke(None, config=config)
print(f"Final answer: {result['generation']}")
```

### Practical Human-in-the-Loop Patterns

| Pattern | Implementation |
|---|---|
| Review before sending | `interrupt_before=["send_response"]` |
| Edit generation | `update_state(config, {"generation": "edited text"})` |
| Approve/reject | `update_state(config, {"approved": True/False})` |
| Add context | `update_state(config, {"documents": extra_docs})` |
| Redirect | `update_state(config, {"route": "web_search"})` |

---

## State Design Best Practices

### 1. Keep State Flat

Avoid deeply nested state. Flat state is easier to update from nodes:

```python
# Good: flat state
class RAGState(TypedDict):
    question: str
    documents: list[str]
    generation: str
    retry_count: int

# Avoid: nested state
class RAGState(TypedDict):
    input: dict       # {"question": "...", "metadata": {...}}
    retrieval: dict    # {"documents": [...], "scores": [...]}
    output: dict       # {"generation": "...", "sources": [...]}
```

### 2. Use Explicit Control Fields

Do not derive routing decisions from data fields. Use explicit control fields:

```python
class RAGState(TypedDict):
    question: str
    documents: list[str]
    generation: str
    # Explicit control fields
    should_web_search: bool    # set by grade_documents node
    is_grounded: bool          # set by grade_generation node
    is_useful: bool            # set by grade_answer node
    retry_count: int
```

### 3. Include Retry Limits

Always cap retries to prevent infinite loops:

```python
def increment_retry(state: RAGState) -> dict:
    return {"retry_count": state.get("retry_count", 0) + 1}

def should_retry(state: RAGState) -> str:
    if state.get("retry_count", 0) >= 3:
        return "give_up"
    return "retry"
```

### 4. Track Provenance

Include fields that track where data came from:

```python
class RAGState(TypedDict):
    question: str
    documents: list[str]
    document_sources: list[str]   # ["kb", "kb", "web", "web", "kb"]
    generation: str
    generation_model: str          # which model generated this
    route_taken: list[str]         # ["retrieve", "grade", "web_search", "generate"]
```

---

## Common Pitfalls

1. **Forgetting to initialize optional state fields**: if a node reads `state["web_search_needed"]` but no previous node set it, you get a KeyError. Use `state.get("field", default)` or initialize all fields in the input.

2. **Using Annotated[list, add] when you want replacement**: if you add documents from retrieval and then from web search with `Annotated[list, add]`, both lists are concatenated. If you want web search results to replace retrieval results, do not annotate the field.

3. **Not testing conditional edges independently**: test your routing functions with different state values before wiring them into the graph. A routing function that returns an unexpected string causes a runtime error.

4. **State explosion**: adding every intermediate value to the state makes it large and hard to debug. Only put values in the state that other nodes need. Keep intermediate calculations local to the node function.

5. **Checkpointing with large state**: if your state contains large documents (full page content), checkpointing to SQLite/Postgres stores all of it. Consider storing document IDs instead of full text, and fetching content when needed.

---

## References

- LangGraph State: https://langchain-ai.github.io/langgraph/concepts/low_level/#state
- LangGraph Checkpointing: https://langchain-ai.github.io/langgraph/concepts/persistence/
- LangGraph Human-in-the-Loop: https://langchain-ai.github.io/langgraph/concepts/human_in_the_loop/
- LangGraph Conditional Edges: https://langchain-ai.github.io/langgraph/how-tos/branching/
