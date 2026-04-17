# LlamaIndex Agents

## Overview

LlamaIndex agents extend query engines with tool-calling capabilities. Instead of a fixed retrieve-then-synthesize pipeline, an agent can reason about which tools to use, call them in sequence, interpret results, and decide whether to call more tools or return a final answer. LlamaIndex 0.12+ provides two main agent types: FunctionAgent (direct function calling via LLM tool-calling APIs) and ReActAgent (explicit reason-then-act loop). Both can use query engines, API functions, and custom tools.

---

## FunctionAgent

FunctionAgent uses the LLM's native function-calling capability (OpenAI tool calls, Anthropic tool use) to select and invoke tools. It is the recommended agent type for models that support function calling.

```python
from llama_index.core.agent import FunctionAgent
from llama_index.core.tools import FunctionTool, QueryEngineTool
from llama_index.llms.openai import OpenAI

# Define tools
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    # In production, call a weather API
    return f"Weather in {city}: 22C, partly cloudy"

def calculate(expression: str) -> str:
    """Evaluate a mathematical expression."""
    try:
        result = eval(expression)  # use a safe evaluator in production
        return str(result)
    except Exception as e:
        return f"Error: {e}"

weather_tool = FunctionTool.from_defaults(fn=get_weather)
calc_tool = FunctionTool.from_defaults(fn=calculate)

# Create agent
agent = FunctionAgent(
    tools=[weather_tool, calc_tool],
    llm=OpenAI(model="gpt-4o-mini"),
    system_prompt="You are a helpful assistant. Use tools when needed.",
)

# Chat with the agent
response = await agent.achat("What is the weather in Rome? Also, what is 42 * 17?")
print(response.response)
# The agent calls get_weather("Rome") and calculate("42 * 17"),
# then synthesizes: "The weather in Rome is 22C, partly cloudy. And 42 * 17 = 714."
```

---

## ReActAgent

ReActAgent implements the Reason-Act loop: it explicitly thinks about what to do, calls a tool, observes the result, and repeats until it has enough information to answer. This is more transparent (you can see the reasoning) but slower than FunctionAgent.

```python
from llama_index.core.agent import ReActAgent
from llama_index.llms.openai import OpenAI

agent = ReActAgent.from_tools(
    tools=[weather_tool, calc_tool],
    llm=OpenAI(model="gpt-4o-mini"),
    verbose=True,    # prints the Thought/Action/Observation loop
    max_iterations=10,
)

response = agent.chat("What is the weather in Rome?")
# Thought: I need to find the weather in Rome. I'll use the get_weather tool.
# Action: get_weather(city="Rome")
# Observation: Weather in Rome: 22C, partly cloudy
# Thought: I have the answer.
# Answer: The weather in Rome is 22C and partly cloudy.
```

### ReAct vs Function Calling

| Aspect | FunctionAgent | ReActAgent |
|---|---|---|
| Mechanism | LLM native tool calling | Prompt-based reason-act loop |
| Speed | Faster (parallel tool calls possible) | Slower (sequential reasoning) |
| Transparency | Low (tool calls in API response) | High (visible Thought/Action/Observation) |
| Multi-step | Handled by LLM | Explicit iteration loop |
| Model support | Requires function-calling models | Any LLM (prompt-based) |
| Reliability | Higher (structured output) | Lower (depends on prompt following) |

---

## Tools from Query Engines

The most powerful pattern: wrap your indexed knowledge bases as tools that agents can query:

```python
from llama_index.core import VectorStoreIndex
from llama_index.core.tools import QueryEngineTool, ToolMetadata

# Create indexes for different knowledge domains
pgvector_index = VectorStoreIndex.from_documents(pgvector_docs)
python_index = VectorStoreIndex.from_documents(python_docs)
devops_index = VectorStoreIndex.from_documents(devops_docs)

# Wrap as tools
pgvector_tool = QueryEngineTool(
    query_engine=pgvector_index.as_query_engine(similarity_top_k=5),
    metadata=ToolMetadata(
        name="pgvector_docs",
        description="Use this tool to answer questions about pgvector: "
                    "installation, configuration, index types (HNSW, IVFFlat), "
                    "distance functions, and query optimization.",
    ),
)

python_tool = QueryEngineTool(
    query_engine=python_index.as_query_engine(similarity_top_k=5),
    metadata=ToolMetadata(
        name="python_docs",
        description="Use this tool for Python programming questions: "
                    "language features, standard library, best practices, "
                    "async programming, and common patterns.",
    ),
)

devops_tool = QueryEngineTool(
    query_engine=devops_index.as_query_engine(similarity_top_k=5),
    metadata=ToolMetadata(
        name="devops_docs",
        description="Use this tool for DevOps questions: Docker, Kubernetes, "
                    "CI/CD, monitoring, deployment strategies.",
    ),
)

# Agent with RAG tools
agent = FunctionAgent(
    tools=[pgvector_tool, python_tool, devops_tool],
    llm=OpenAI(model="gpt-4o"),
    system_prompt=(
        "You are a technical assistant. Use the available knowledge base tools "
        "to answer questions accurately. Always cite which tool you used."
    ),
)

# The agent routes to the right knowledge base automatically
response = await agent.achat("How do I create an HNSW index with pgvector in Python?")
# Agent calls pgvector_docs and python_docs tools, combines the answers
```

---

## Custom Tool Creation

### From Functions

```python
from llama_index.core.tools import FunctionTool

# Simple function tool
def search_jira(query: str, project: str = "BACKEND") -> str:
    """Search JIRA tickets by query text and project key."""
    # Call JIRA API
    import requests
    response = requests.get(
        "https://jira.example.com/rest/api/2/search",
        params={"jql": f'project = {project} AND text ~ "{query}"', "maxResults": 5},
        headers={"Authorization": "Bearer ..."},
    )
    issues = response.json()["issues"]
    return "\n".join(f"{i['key']}: {i['fields']['summary']}" for i in issues)

jira_tool = FunctionTool.from_defaults(
    fn=search_jira,
    name="search_jira",
    description="Search JIRA tickets. Use for questions about bugs, features, or tasks.",
)
```

### Async Function Tools

```python
from llama_index.core.tools import FunctionTool

async def fetch_api_status(service_name: str) -> str:
    """Check the health status of a microservice."""
    import aiohttp
    async with aiohttp.ClientSession() as session:
        async with session.get(f"https://api.example.com/{service_name}/health") as resp:
            data = await resp.json()
            return f"{service_name}: {data['status']} (uptime: {data['uptime']})"

status_tool = FunctionTool.from_defaults(
    async_fn=fetch_api_status,
    name="check_service_status",
    description="Check the health status of a microservice by name.",
)
```

### Tool with Pydantic Schema

For complex input/output schemas:

```python
from llama_index.core.tools import FunctionTool
from pydantic import BaseModel, Field

class DatabaseQueryInput(BaseModel):
    query: str = Field(description="SQL query to execute")
    database: str = Field(description="Target database name", default="production")
    limit: int = Field(description="Maximum rows to return", default=10, ge=1, le=100)

def execute_query(query: str, database: str = "production", limit: int = 10) -> str:
    """Execute a read-only SQL query against a database."""
    # Validate it is read-only
    if not query.strip().upper().startswith("SELECT"):
        return "Error: only SELECT queries are allowed"
    # Execute
    import psycopg2
    conn = psycopg2.connect(f"dbname={database}")
    cur = conn.cursor()
    cur.execute(query + f" LIMIT {limit}")
    rows = cur.fetchall()
    return str(rows)

db_tool = FunctionTool.from_defaults(
    fn=execute_query,
    name="database_query",
    description="Execute a read-only SQL query against the production database. "
                "Use for questions about data, statistics, or records.",
    fn_schema=DatabaseQueryInput,
)
```

---

## Multi-Agent Orchestration

### Agent as Tool Pattern

Create specialized agents and expose them as tools to a coordinator agent:

```python
from llama_index.core.agent import FunctionAgent
from llama_index.core.tools import FunctionTool, QueryEngineTool

# Specialist agent 1: Database expert
db_agent = FunctionAgent(
    tools=[pgvector_tool, db_query_tool],
    llm=OpenAI(model="gpt-4o-mini"),
    system_prompt="You are a database expert. Answer questions about "
                  "databases, SQL, pgvector, and data modeling.",
)

# Specialist agent 2: DevOps expert
devops_agent = FunctionAgent(
    tools=[devops_tool, kubernetes_tool, docker_tool],
    llm=OpenAI(model="gpt-4o-mini"),
    system_prompt="You are a DevOps expert. Answer questions about "
                  "deployment, containers, CI/CD, and infrastructure.",
)

# Wrap agents as tools
async def query_db_expert(question: str) -> str:
    """Ask the database expert a question about databases, SQL, or pgvector."""
    response = await db_agent.achat(question)
    return response.response

async def query_devops_expert(question: str) -> str:
    """Ask the DevOps expert a question about deployment or infrastructure."""
    response = await devops_agent.achat(question)
    return response.response

db_expert_tool = FunctionTool.from_defaults(async_fn=query_db_expert)
devops_expert_tool = FunctionTool.from_defaults(async_fn=query_devops_expert)

# Coordinator agent
coordinator = FunctionAgent(
    tools=[db_expert_tool, devops_expert_tool],
    llm=OpenAI(model="gpt-4o"),
    system_prompt=(
        "You are a technical coordinator. Analyze the user's question and "
        "route it to the appropriate specialist. For questions that span "
        "multiple domains, consult multiple specialists."
    ),
)

# The coordinator routes to the right specialist
response = await coordinator.achat(
    "How do I deploy pgvector on Kubernetes with HNSW indexing?"
)
```

---

## AgentRunner with Streaming

For production applications that need streaming and step-by-step execution:

```python
from llama_index.core.agent import FunctionAgent

agent = FunctionAgent(
    tools=[pgvector_tool, python_tool],
    llm=OpenAI(model="gpt-4o-mini"),
)

# Streaming chat
response = await agent.astream_chat("How to use pgvector with Python?")
async for token in response.async_response_gen():
    print(token, end="", flush=True)
print()
```

### Step-by-Step Execution

```python
# For debugging: see each step the agent takes
from llama_index.core.agent import FunctionAgent

agent = FunctionAgent(
    tools=[pgvector_tool, calc_tool, weather_tool],
    llm=OpenAI(model="gpt-4o-mini"),
)

# Get the full task with steps
handler = agent.run(
    "Compare pgvector HNSW with IVFFlat performance, "
    "and calculate the storage difference for 1M vectors"
)

# Access intermediate steps
response = await handler
for step in handler.tool_calls:
    print(f"Tool: {step.tool_name}")
    print(f"Input: {step.tool_kwargs}")
    print(f"Output: {step.output}")
    print("---")
```

---

## Combining RAG Tools with API Tools

The most practical agent pattern: knowledge base tools for documentation + API tools for live data:

```python
from llama_index.core.agent import FunctionAgent
from llama_index.core.tools import FunctionTool, QueryEngineTool

# RAG tool: documentation knowledge base
docs_tool = QueryEngineTool(
    query_engine=docs_index.as_query_engine(similarity_top_k=5),
    metadata=ToolMetadata(
        name="documentation",
        description="Search technical documentation for how-to guides, "
                    "API references, and best practices.",
    ),
)

# API tool: live system data
async def get_system_metrics(service: str, metric: str = "latency") -> str:
    """Get real-time metrics for a service. Metrics: latency, error_rate, qps."""
    import aiohttp
    async with aiohttp.ClientSession() as session:
        url = f"https://monitoring.internal/api/v1/metrics/{service}/{metric}"
        async with session.get(url) as resp:
            data = await resp.json()
            return f"{service} {metric}: {data['value']} {data['unit']} (last 5 min)"

metrics_tool = FunctionTool.from_defaults(async_fn=get_system_metrics)

# API tool: create JIRA ticket
async def create_ticket(title: str, description: str, priority: str = "Medium") -> str:
    """Create a JIRA ticket for tracking an issue or task."""
    # JIRA API call
    return f"Created ticket PROJ-1234: {title} (priority: {priority})"

ticket_tool = FunctionTool.from_defaults(async_fn=create_ticket)

# Agent with mixed tools
agent = FunctionAgent(
    tools=[docs_tool, metrics_tool, ticket_tool],
    llm=OpenAI(model="gpt-4o"),
    system_prompt=(
        "You are a platform engineering assistant. You can:\n"
        "1. Search documentation for technical answers\n"
        "2. Check real-time system metrics\n"
        "3. Create JIRA tickets for issues\n"
        "Answer questions using the appropriate tools. If you detect "
        "a problem, proactively suggest creating a ticket."
    ),
)

# Complex interaction
response = await agent.achat(
    "Our pgvector search is slow. What is the current latency, "
    "and what does the documentation say about HNSW tuning?"
)
# Agent:
# 1. Calls get_system_metrics("pgvector-search", "latency")
# 2. Calls documentation("pgvector HNSW tuning for performance")
# 3. Synthesizes answer with both live data and documentation
# 4. May suggest creating a ticket if latency is abnormal
```

---

## Agent Memory and Context

### Chat History

Agents automatically maintain chat history within a session:

```python
agent = FunctionAgent(
    tools=[docs_tool],
    llm=OpenAI(model="gpt-4o-mini"),
)

# Multi-turn conversation
r1 = await agent.achat("What is pgvector?")
r2 = await agent.achat("How do I install it?")  # "it" refers to pgvector
r3 = await agent.achat("What about HNSW tuning?")  # builds on previous context
```

### Resetting Context

```python
# Reset chat history
agent.reset()
```

---

## Common Pitfalls

1. **Vague tool descriptions**: agents select tools based on descriptions. "Database tool" is too vague. "Search pgvector documentation for HNSW index configuration, distance functions, and query optimization" is specific enough for accurate routing.

2. **Too many tools**: agents with >10 tools struggle to select the right one. Group related functionality into fewer, broader tools, or use a multi-agent architecture with specialized sub-agents.

3. **Not setting max_iterations**: ReActAgent can loop indefinitely if the tool output is ambiguous. Always set `max_iterations=10` or similar.

4. **Sync tools in async agents**: using synchronous tools with `achat()` blocks the event loop. Use `async_fn` for async agents.

5. **Not handling tool errors**: if a tool raises an exception, the agent may get confused. Wrap tool functions in try/except and return error messages as strings.

6. **Expecting deterministic behavior**: agents are non-deterministic. The same query may use different tools or produce different answers across runs. For deterministic pipelines, use query engines instead.

---

## References

- LlamaIndex Agents: https://docs.llamaindex.ai/en/stable/module_guides/deploying/agents/
- LlamaIndex Tools: https://docs.llamaindex.ai/en/stable/module_guides/deploying/agents/tools/
- ReAct paper: https://arxiv.org/abs/2210.03629
- LlamaIndex Multi-Agent: https://docs.llamaindex.ai/en/stable/examples/agent/multi_agent/
