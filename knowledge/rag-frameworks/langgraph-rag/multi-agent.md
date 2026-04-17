# Multi-Agent RAG in LangGraph

## Overview

Multi-agent RAG uses a supervisor agent to orchestrate specialist sub-agents, each responsible for a different retrieval strategy or data source. Instead of a single retrieval pipeline, the supervisor analyzes the query, routes to the appropriate specialist (vector retriever, web searcher, SQL querier), collects results, and synthesizes a unified answer. LangGraph's graph structure naturally supports this pattern through nodes (agents), conditional edges (routing), and shared state (accumulated context).

---

## Architecture

```
User Query
    |
    v
[Supervisor Agent]
    |
    +-- "knowledge base query" --> [Retriever Agent]
    |                                    |
    |                              vector store search
    |                                    |
    +-- "current events query" ----> [Web Searcher Agent]
    |                                    |
    |                              Tavily / Google search
    |                                    |
    +-- "data/metrics query" ------> [SQL Querier Agent]
    |                                    |
    |                              database query
    |                                    |
    v
[Synthesizer Agent]
    |
    v
Final Answer with sources
```

---

## State Design for Multi-Agent

```python
from typing import TypedDict, Literal, Annotated
from operator import add

class MultiAgentRAGState(TypedDict):
    # Input
    question: str

    # Supervisor decisions
    agents_to_invoke: list[str]          # ["retriever", "web_searcher", "sql_querier"]
    current_agent: str                    # which agent is currently running

    # Per-agent results (accumulated)
    retriever_results: list[str]
    web_results: list[str]
    sql_results: list[str]

    # Merged context for synthesis
    all_context: list[str]

    # Final output
    generation: str
    sources_used: list[str]

    # Control
    agents_completed: list[str]
    retry_count: int
```

---

## Supervisor Agent

The supervisor analyzes the query and decides which specialist agents to invoke:

```python
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

class SupervisorDecision(BaseModel):
    """Supervisor's routing decision."""
    agents: list[str] = Field(
        description="List of agents to invoke. Options: 'retriever', 'web_searcher', 'sql_querier'"
    )
    reasoning: str = Field(description="Brief reasoning for the routing decision")

supervisor_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0).with_structured_output(
    SupervisorDecision
)

def supervisor(state: MultiAgentRAGState) -> dict:
    """Analyze the question and decide which agents to invoke."""
    question = state["question"]

    decision = supervisor_llm.invoke(
        f"You are a supervisor that routes questions to specialist agents.\n\n"
        f"Available agents:\n"
        f"- retriever: searches the internal knowledge base (documentation, guides, references)\n"
        f"- web_searcher: searches the web for current information, news, recent updates\n"
        f"- sql_querier: queries the database for metrics, statistics, and structured data\n\n"
        f"You can invoke multiple agents if the question spans multiple domains.\n\n"
        f"Question: {question}\n\n"
        f"Which agents should handle this question?"
    )

    return {
        "agents_to_invoke": decision.agents,
        "agents_completed": [],
    }
```

---

## Specialist Agents

### Retriever Agent

```python
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import Chroma

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma(
    embedding_function=embeddings,
    persist_directory="./chroma_db",
)

def retriever_agent(state: MultiAgentRAGState) -> dict:
    """Retrieve relevant documents from the knowledge base."""
    question = state["question"]

    # Retrieve with scores
    docs_with_scores = vectorstore.similarity_search_with_relevance_scores(
        question, k=5
    )

    # Filter by relevance threshold
    relevant_docs = [
        f"[KB] {doc.page_content}"
        for doc, score in docs_with_scores
        if score > 0.6
    ]

    if not relevant_docs:
        relevant_docs = [f"[KB] {doc.page_content}" for doc, _ in docs_with_scores[:3]]

    completed = state.get("agents_completed", []) + ["retriever"]

    return {
        "retriever_results": relevant_docs,
        "agents_completed": completed,
    }
```

### Web Searcher Agent

```python
from langchain_community.tools.tavily_search import TavilySearchResults

web_tool = TavilySearchResults(max_results=5)

def web_searcher_agent(state: MultiAgentRAGState) -> dict:
    """Search the web for current information."""
    question = state["question"]

    results = web_tool.invoke({"query": question})

    web_docs = []
    for r in results:
        if "content" in r:
            source = r.get("url", "web")
            web_docs.append(f"[Web: {source}] {r['content']}")

    completed = state.get("agents_completed", []) + ["web_searcher"]

    return {
        "web_results": web_docs,
        "agents_completed": completed,
    }
```

### SQL Querier Agent

```python
from langchain_community.utilities import SQLDatabase

db = SQLDatabase.from_uri("postgresql://user:pass@localhost/analytics")

def sql_querier_agent(state: MultiAgentRAGState) -> dict:
    """Query the database for metrics and structured data."""
    question = state["question"]
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

    # Generate SQL from natural language
    schema = db.get_table_info()
    sql_response = llm.invoke(
        f"Given the following database schema, write a SQL query to answer the question. "
        f"Return ONLY the SQL query, no explanation.\n\n"
        f"Schema:\n{schema}\n\n"
        f"Question: {question}\n\n"
        f"SQL:"
    )

    sql_query = sql_response.content.strip().strip("```sql").strip("```").strip()

    # Execute with safety check
    if not sql_query.upper().startswith("SELECT"):
        return {
            "sql_results": ["[SQL] Error: only SELECT queries are allowed"],
            "agents_completed": state.get("agents_completed", []) + ["sql_querier"],
        }

    try:
        result = db.run(sql_query)
        sql_docs = [f"[SQL Query: {sql_query}]\n[SQL Result]: {result}"]
    except Exception as e:
        sql_docs = [f"[SQL] Error executing query: {e}"]

    completed = state.get("agents_completed", []) + ["sql_querier"]

    return {
        "sql_results": sql_docs,
        "agents_completed": completed,
    }
```

---

## Synthesizer Agent

Merges results from all specialist agents into a coherent answer:

```python
def merge_results(state: MultiAgentRAGState) -> dict:
    """Merge results from all agents into a single context."""
    all_context = []

    if state.get("retriever_results"):
        all_context.extend(state["retriever_results"])
    if state.get("web_results"):
        all_context.extend(state["web_results"])
    if state.get("sql_results"):
        all_context.extend(state["sql_results"])

    sources = []
    if state.get("retriever_results"):
        sources.append("knowledge_base")
    if state.get("web_results"):
        sources.append("web_search")
    if state.get("sql_results"):
        sources.append("database")

    return {
        "all_context": all_context,
        "sources_used": sources,
    }


def synthesize(state: MultiAgentRAGState) -> dict:
    """Generate a final answer from all collected context."""
    question = state["question"]
    context = state["all_context"]
    sources = state.get("sources_used", [])

    context_text = "\n\n---\n\n".join(context)

    llm = ChatOpenAI(model="gpt-4o", temperature=0)

    response = llm.invoke(
        f"You are a helpful assistant answering questions using information "
        f"from multiple sources. Synthesize a comprehensive answer from the "
        f"context below. Cite sources using [KB], [Web], or [SQL] prefixes "
        f"as they appear in the context.\n\n"
        f"Sources used: {', '.join(sources)}\n\n"
        f"Context:\n{context_text}\n\n"
        f"Question: {question}\n\n"
        f"Answer:"
    )

    return {"generation": response.content}
```

---

## Building the Graph

### Sequential Agent Execution

The simplest approach: invoke agents one at a time based on the supervisor's decision:

```python
from langgraph.graph import StateGraph, START, END

def route_to_agent(state: MultiAgentRAGState) -> str:
    """Route to the next agent that has not been completed."""
    agents_to_invoke = state.get("agents_to_invoke", [])
    agents_completed = state.get("agents_completed", [])

    for agent in agents_to_invoke:
        if agent not in agents_completed:
            return agent

    return "merge"  # all agents done

graph = StateGraph(MultiAgentRAGState)

# Add nodes
graph.add_node("supervisor", supervisor)
graph.add_node("retriever", retriever_agent)
graph.add_node("web_searcher", web_searcher_agent)
graph.add_node("sql_querier", sql_querier_agent)
graph.add_node("merge", merge_results)
graph.add_node("synthesize", synthesize)

# Edges
graph.add_edge(START, "supervisor")

# After supervisor, route to first agent
graph.add_conditional_edges(
    "supervisor",
    route_to_agent,
    {
        "retriever": "retriever",
        "web_searcher": "web_searcher",
        "sql_querier": "sql_querier",
        "merge": "merge",
    },
)

# After each agent, route to next agent or merge
for agent_node in ["retriever", "web_searcher", "sql_querier"]:
    graph.add_conditional_edges(
        agent_node,
        route_to_agent,
        {
            "retriever": "retriever",
            "web_searcher": "web_searcher",
            "sql_querier": "sql_querier",
            "merge": "merge",
        },
    )

graph.add_edge("merge", "synthesize")
graph.add_edge("synthesize", END)

# Compile
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
app = graph.compile(checkpointer=memory)
```

### Running the Multi-Agent System

```python
# Knowledge base question
config = {"configurable": {"thread_id": "multi_1"}}
result = app.invoke(
    {"question": "How do I configure HNSW indexes in pgvector?"},
    config=config,
)
print(result["generation"])
print(f"Sources: {result['sources_used']}")

# Cross-domain question (routes to multiple agents)
config = {"configurable": {"thread_id": "multi_2"}}
result = app.invoke(
    {"question": "What is our current database query latency, and how does "
                 "pgvector recommend tuning HNSW for better performance?"},
    config=config,
)
print(result["generation"])
print(f"Sources: {result['sources_used']}")
# Sources: ['knowledge_base', 'database']
```

---

## Parallel Retrieval from Multiple Sources

For lower latency, invoke agents in parallel instead of sequentially:

```python
import asyncio
from langgraph.graph import StateGraph, START, END

# Parallel execution using Send API
from langgraph.types import Send

def route_to_agents_parallel(state: MultiAgentRAGState) -> list[Send]:
    """Send the question to all selected agents in parallel."""
    agents = state.get("agents_to_invoke", [])
    sends = []
    for agent in agents:
        sends.append(Send(agent, state))
    return sends

# Alternative: use asyncio for parallel execution within a single node
async def parallel_retrieval(state: MultiAgentRAGState) -> dict:
    """Execute all agents in parallel."""
    agents = state.get("agents_to_invoke", [])

    async def run_retriever():
        if "retriever" in agents:
            return retriever_agent(state)
        return {}

    async def run_web_searcher():
        if "web_searcher" in agents:
            return web_searcher_agent(state)
        return {}

    async def run_sql_querier():
        if "sql_querier" in agents:
            return sql_querier_agent(state)
        return {}

    results = await asyncio.gather(
        run_retriever(),
        run_web_searcher(),
        run_sql_querier(),
    )

    # Merge all results
    merged = {}
    for result in results:
        merged.update(result)

    return merged
```

---

## Handoff Patterns

### Explicit Handoff

Agent A explicitly passes control to Agent B with a message:

```python
class HandoffState(TypedDict):
    question: str
    documents: list[str]
    generation: str
    handoff_to: str             # next agent to invoke
    handoff_message: str        # context for the next agent

def retriever_with_handoff(state: HandoffState) -> dict:
    """Retrieve docs. If insufficient, hand off to web searcher."""
    docs = vectorstore.similarity_search(state["question"], k=5)

    if len(docs) < 2:  # insufficient results
        return {
            "documents": [doc.page_content for doc in docs],
            "handoff_to": "web_searcher",
            "handoff_message": f"Knowledge base had only {len(docs)} results. "
                               f"Need web search for: {state['question']}",
        }

    return {
        "documents": [doc.page_content for doc in docs],
        "handoff_to": "synthesize",
        "handoff_message": "",
    }
```

### Supervisor Re-Evaluation

After each agent completes, the supervisor re-evaluates whether more agents are needed:

```python
def supervisor_reevaluate(state: MultiAgentRAGState) -> dict:
    """Re-evaluate after an agent completes. Decide if more work is needed."""
    question = state["question"]
    completed = state.get("agents_completed", [])
    all_context = []
    if state.get("retriever_results"):
        all_context.extend(state["retriever_results"])
    if state.get("web_results"):
        all_context.extend(state["web_results"])

    if len(all_context) >= 3:
        # Enough context, proceed to synthesis
        return {"agents_to_invoke": []}

    # Not enough context, invoke more agents
    remaining = [a for a in ["retriever", "web_searcher", "sql_querier"]
                 if a not in completed]

    if remaining:
        return {"agents_to_invoke": remaining[:1]}  # invoke next available
    return {"agents_to_invoke": []}
```

---

## Shared State vs Per-Agent State

### Shared State (Simpler)

All agents read from and write to the same state object:

```python
# All agents see the same state
class SharedState(TypedDict):
    question: str
    documents: list[str]      # all agents append to the same list
    generation: str
```

**Pros**: simple, no merging needed.
**Cons**: agents can accidentally overwrite each other's results.

### Per-Agent State (Safer)

Each agent has its own output field:

```python
class PerAgentState(TypedDict):
    question: str
    retriever_results: list[str]   # only retriever writes here
    web_results: list[str]         # only web searcher writes here
    sql_results: list[str]         # only SQL querier writes here
    merged_context: list[str]      # merger combines all
    generation: str
```

**Pros**: no accidental overwrites, clear data lineage.
**Cons**: more state fields, need an explicit merge step.

Recommendation: use per-agent state for production systems with multiple agents.

---

## Error Handling

### Agent Failure Recovery

```python
def safe_agent_wrapper(agent_fn, agent_name):
    """Wrap an agent function with error handling."""
    def wrapped(state):
        try:
            return agent_fn(state)
        except Exception as e:
            print(f"Agent {agent_name} failed: {e}")
            # Mark as completed (with empty results) so the pipeline continues
            completed = state.get("agents_completed", []) + [agent_name]
            return {
                f"{agent_name}_results": [f"[{agent_name}] Error: {str(e)}"],
                "agents_completed": completed,
            }
    return wrapped

# Use in graph
graph.add_node("retriever", safe_agent_wrapper(retriever_agent, "retriever"))
graph.add_node("web_searcher", safe_agent_wrapper(web_searcher_agent, "web_searcher"))
graph.add_node("sql_querier", safe_agent_wrapper(sql_querier_agent, "sql_querier"))
```

### Timeout Handling

```python
import asyncio

async def agent_with_timeout(agent_fn, state, timeout_seconds=30):
    """Run an agent with a timeout."""
    try:
        result = await asyncio.wait_for(
            asyncio.to_thread(agent_fn, state),
            timeout=timeout_seconds,
        )
        return result
    except asyncio.TimeoutError:
        return {
            "agents_completed": state.get("agents_completed", []) + ["timed_out"],
        }
```

---

## Production Considerations

### Latency Budget

| Component | Typical Latency | Notes |
|---|---|---|
| Supervisor routing | 200-500ms | Single LLM call with structured output |
| Vector retrieval | 10-50ms | Depends on index type and size |
| Web search | 500-2000ms | External API call |
| SQL query | 50-500ms | Depends on query complexity |
| Synthesis | 500-2000ms | Single LLM call |
| **Total (sequential)** | **1.5-5 seconds** | |
| **Total (parallel retrieval)** | **1-3 seconds** | Retrieval in parallel |

### Cost Per Query

| Component | Calls | Est. Cost (GPT-4o-mini) |
|---|---|---|
| Supervisor routing | 1 | $0.0002 |
| SQL generation | 0-1 | $0.0003 |
| Synthesis (GPT-4o) | 1 | $0.003 |
| **Total** | 3-4 | **~$0.004/query** |

---

## Common Pitfalls

1. **Supervisor routing to all agents every time**: if the supervisor always invokes all 3 agents, you pay the latency and cost of all three. Train the supervisor to be selective.

2. **Not handling agent failures gracefully**: if the web search API is down, the entire pipeline should not fail. Wrap agents with error handling and continue with available results.

3. **Over-engineering for simple queries**: a question like "What is pgvector?" does not need multi-agent orchestration. Consider a "fast path" that skips the supervisor for simple queries.

4. **Shared state collisions**: if two agents run in parallel and both write to `state["documents"]`, one overwrites the other. Use per-agent result fields.

5. **Not logging which agents were invoked**: for debugging, always record which agents were called and what they returned. This is essential for understanding why a particular answer was produced.

6. **Ignoring the synthesis step**: just concatenating results from multiple agents produces a disjointed answer. The synthesizer must integrate information from all sources into a coherent narrative.

---

## References

- LangGraph Multi-Agent: https://langchain-ai.github.io/langgraph/concepts/multi_agent/
- LangGraph Supervisor pattern: https://langchain-ai.github.io/langgraph/tutorials/multi_agent/agent_supervisor/
- LangGraph Send API (parallel execution): https://langchain-ai.github.io/langgraph/how-tos/map-reduce/
- LangGraph Handoffs: https://langchain-ai.github.io/langgraph/concepts/multi_agent/#handoffs
