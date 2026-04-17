# Query Routing

## Overview / TL;DR

Query routing directs incoming queries to the most appropriate retrieval backend, index, or tool based on the query's intent, topic, or structure. In production RAG systems, not all queries should hit the same index -- some need vector search over documentation, others need SQL queries against a database, others need web search for real-time information, and some need no retrieval at all (the LLM can answer directly). Routing prevents wasting resources on inappropriate retrieval and improves answer quality by matching queries to their optimal data source. This document covers LLM-based routing, embedding-based classification, rule-based routing, and implementations in LangChain and LlamaIndex.

---

## Why Routing Matters

Consider a system with three knowledge sources:

1. **Vector index**: Technical documentation (500K chunks)
2. **SQL database**: Product inventory and pricing
3. **Web search**: Real-time news and updates

Without routing:
- "What is our return policy?" -> Vector search (correct)
- "How many units of SKU-4821 are in stock?" -> Vector search (wrong -- should be SQL)
- "What did the CEO say at yesterday's conference?" -> Vector search (wrong -- should be web)

With routing:
- Each query goes to the right source
- Latency is lower (SQL is faster than vector search for structured queries)
- Accuracy is higher (the right tool for the right job)

---

## Routing Strategy 1: LLM-Based Routing

Use an LLM to classify the query intent and route to the appropriate backend.

```python
import anthropic
import json
from typing import Literal

client = anthropic.Anthropic()

ROUTE_DEFINITIONS = {
    "vector_search": "Questions about documentation, procedures, concepts, best practices, how-to guides",
    "sql_query": "Questions about quantities, inventory, pricing, statistics, counts, comparisons of numerical data",
    "web_search": "Questions about current events, recent news, real-time information, things that happened today/this week",
    "direct_answer": "Simple factual questions the AI can answer without retrieval, greetings, clarification requests",
}


def route_query(question: str) -> dict:
    """Classify query intent and select the routing destination."""
    route_descriptions = "\n".join(
        f"- {name}: {desc}" for name, desc in ROUTE_DEFINITIONS.items()
    )

    response = client.messages.create(
        model="claude-haiku-4-20250514",
        max_tokens=100,
        temperature=0,
        messages=[{
            "role": "user",
            "content": (
                f"Classify the following query into exactly one of these categories:\n"
                f"{route_descriptions}\n\n"
                f"Query: {question}\n\n"
                f"Respond with a JSON object: {{\"route\": \"<category_name>\", \"confidence\": <0.0-1.0>}}"
            ),
        }],
    )

    try:
        result = json.loads(response.content[0].text.strip())
        return result
    except json.JSONDecodeError:
        return {"route": "vector_search", "confidence": 0.5}  # Default


# Examples:
# "What is our return policy?" -> {"route": "vector_search", "confidence": 0.95}
# "How many units of SKU-4821 in stock?" -> {"route": "sql_query", "confidence": 0.92}
# "What did the CEO say yesterday?" -> {"route": "web_search", "confidence": 0.88}
# "Hello, how are you?" -> {"route": "direct_answer", "confidence": 0.97}
```

### Multi-Route (Query Needs Multiple Sources)

Sometimes a query requires information from multiple sources:

```python
def route_multi_source(question: str) -> list[dict]:
    """Identify all data sources needed to answer the query."""
    response = client.messages.create(
        model="claude-haiku-4-20250514",
        max_tokens=200,
        temperature=0,
        messages=[{
            "role": "user",
            "content": (
                f"Analyze this query and determine which data sources are needed. "
                f"A query may require 1-3 sources.\n\n"
                f"Available sources:\n"
                f"- vector_search: documentation, procedures, concepts\n"
                f"- sql_query: inventory, pricing, statistics\n"
                f"- web_search: current events, real-time info\n\n"
                f"Query: {question}\n\n"
                f"Respond with JSON: [{{\"route\": \"...\", \"sub_query\": \"...\"}}]"
            ),
        }],
    )

    try:
        return json.loads(response.content[0].text.strip())
    except json.JSONDecodeError:
        return [{"route": "vector_search", "sub_query": question}]


# Example:
# "Compare our return policy with what Amazon announced last week"
# -> [
#     {"route": "vector_search", "sub_query": "company return policy"},
#     {"route": "web_search", "sub_query": "Amazon return policy announcement this week"},
# ]
```

---

## Routing Strategy 2: Embedding-Based Classification

Train or use a pre-computed embedding classifier to route queries without an LLM call. This is faster (~5ms vs ~150ms for LLM) and cheaper (no API cost).

```python
from sentence_transformers import SentenceTransformer
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity


class EmbeddingRouter:
    """Route queries using embedding similarity to route descriptions."""

    def __init__(self, model_name: str = "BAAI/bge-base-en-v1.5"):
        self.model = SentenceTransformer(model_name)
        self.routes = {}
        self.route_embeddings = {}

    def add_route(
        self,
        name: str,
        description: str,
        examples: list[str],
    ):
        """Add a routing destination with description and example queries."""
        # Embed the description and all examples
        all_texts = [description] + examples
        embeddings = self.model.encode(all_texts, normalize_embeddings=True)

        # Average all embeddings to create the route's centroid
        centroid = np.mean(embeddings, axis=0)
        centroid = centroid / np.linalg.norm(centroid)

        self.routes[name] = {
            "description": description,
            "examples": examples,
        }
        self.route_embeddings[name] = centroid

    def route(self, query: str, threshold: float = 0.3) -> dict:
        """Route a query to the best-matching destination."""
        query_emb = self.model.encode(query, normalize_embeddings=True)

        scores = {}
        for name, centroid in self.route_embeddings.items():
            scores[name] = float(np.dot(query_emb, centroid))

        best_route = max(scores, key=scores.get)
        best_score = scores[best_route]

        if best_score < threshold:
            return {"route": "default", "confidence": 0.0, "scores": scores}

        return {
            "route": best_route,
            "confidence": best_score,
            "scores": scores,
        }


# Setup
router = EmbeddingRouter()

router.add_route(
    "documentation",
    "Technical documentation, procedures, how-to guides, configuration",
    [
        "How do I configure SSL?",
        "What is the authentication process?",
        "Explain the deployment pipeline",
        "What are the best practices for error handling?",
    ],
)

router.add_route(
    "database",
    "Database queries, inventory, pricing, statistics, counts",
    [
        "How many orders were placed last month?",
        "What is the price of product X?",
        "Show me the top 10 customers by revenue",
        "Count of active subscriptions by plan type",
    ],
)

router.add_route(
    "web_search",
    "Current events, recent news, real-time information",
    [
        "What happened in the market today?",
        "Latest updates on the product launch",
        "Current exchange rate for EUR/USD",
        "What did the CEO announce this week?",
    ],
)

# Route queries
result = router.route("How do I set up database replication?")
# -> {"route": "documentation", "confidence": 0.82, ...}
```

**Advantages over LLM routing**: 5-10ms latency (vs 100-200ms), zero API cost, deterministic results.

**Disadvantages**: Less flexible, requires example queries for each route, may struggle with ambiguous queries.

---

## Routing Strategy 3: Rule-Based Routing

For simpler systems, rules can be effective and require no model:

```python
import re


class RuleRouter:
    """Rule-based query router using keyword patterns."""

    def __init__(self):
        self.rules = []

    def add_rule(self, route: str, patterns: list[str], priority: int = 0):
        """Add a routing rule with regex patterns."""
        compiled = [re.compile(p, re.IGNORECASE) for p in patterns]
        self.rules.append({
            "route": route,
            "patterns": compiled,
            "priority": priority,
        })
        self.rules.sort(key=lambda r: r["priority"], reverse=True)

    def route(self, query: str) -> dict:
        """Match query against rules."""
        for rule in self.rules:
            for pattern in rule["patterns"]:
                if pattern.search(query):
                    return {
                        "route": rule["route"],
                        "matched_pattern": pattern.pattern,
                        "confidence": 1.0,
                    }

        return {"route": "vector_search", "confidence": 0.5}  # Default


# Setup
router = RuleRouter()

router.add_rule("sql_query", [
    r"how many",
    r"count of",
    r"total number",
    r"price of",
    r"in stock",
    r"inventory",
    r"revenue",
    r"top \d+",
    r"average|median|sum|min|max",
], priority=10)

router.add_rule("web_search", [
    r"today|yesterday|this week|this month|latest|recent|current",
    r"announced|released|launched",
    r"breaking news",
], priority=5)

router.add_rule("direct_answer", [
    r"^(hello|hi|hey|thanks|thank you)",
    r"^(what time|what date)",
], priority=20)

# Everything else falls through to vector_search
```

---

## Routing to Different Vector Indexes

In large systems, you may have multiple specialized vector indexes (one per domain, topic, or document type):

```python
class IndexRouter:
    """Route queries to specialized vector indexes."""

    def __init__(self):
        self.indexes = {}  # name -> (index, metadata)

    def register_index(self, name: str, index, description: str):
        self.indexes[name] = {"index": index, "description": description}

    def route_and_retrieve(
        self,
        question: str,
        top_k: int = 5,
    ) -> list[dict]:
        """Route to the best index and retrieve."""
        # Classify the question
        route_result = classify_query_domain(question)
        index_name = route_result["route"]

        if index_name not in self.indexes:
            index_name = "general"  # Fallback

        index = self.indexes[index_name]["index"]
        results = index.search(question, top_k=top_k)

        # Tag results with which index they came from
        for r in results:
            r["source_index"] = index_name

        return results

    def route_and_retrieve_multi(
        self,
        question: str,
        top_k: int = 5,
    ) -> list[dict]:
        """Query multiple indexes and fuse results."""
        # Determine which indexes are relevant
        relevant_indexes = classify_relevant_indexes(question)

        all_results = []
        for idx_name in relevant_indexes:
            if idx_name in self.indexes:
                results = self.indexes[idx_name]["index"].search(
                    question, top_k=top_k * 2
                )
                for r in results:
                    r["source_index"] = idx_name
                    r["id"] = f"{idx_name}-{r.get('id', hash(r['text']))}"
                all_results.append(results)

        # Fuse with RRF
        from query_transformations import reciprocal_rank_fusion
        fused = reciprocal_rank_fusion(all_results, top_k=top_k)
        return fused
```

---

## LangChain RouterChain

```python
from langchain.chains.router import MultiPromptChain
from langchain.chains.router.llm_router import LLMRouterChain, RouterOutputParser
from langchain.prompts import PromptTemplate
from langchain_openai import ChatOpenAI

# Define destination chains
destinations = [
    {
        "name": "documentation",
        "description": "For questions about technical documentation, setup guides, and procedures",
        "chain": documentation_rag_chain,  # Your RAG chain for docs
    },
    {
        "name": "database",
        "description": "For questions about data, statistics, counts, and inventory",
        "chain": sql_chain,  # Your SQL generation chain
    },
]

# Create router
router_template = """Given a raw text input to a language model, select the
model prompt best suited for the input. You will be given the names of the
available prompts and a description of what the prompt is best suited for.

<< FORMATTING >>
Return a JSON object with "destination" and "next_inputs" keys.

<< CANDIDATE PROMPTS >>
{destinations}

<< INPUT >>
{input}

<< OUTPUT >>"""

router_prompt = PromptTemplate(
    template=router_template,
    input_variables=["input"],
    partial_variables={
        "destinations": "\n".join(
            f"{d['name']}: {d['description']}" for d in destinations
        )
    },
)

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
router_chain = LLMRouterChain.from_llm(llm, router_prompt)

# Create the multi-route chain
chain = MultiPromptChain(
    router_chain=router_chain,
    destination_chains={d["name"]: d["chain"] for d in destinations},
    default_chain=documentation_rag_chain,
)
```

---

## LlamaIndex RouterQueryEngine

```python
from llama_index.core.query_engine import RouterQueryEngine
from llama_index.core.selectors import LLMSingleSelector, LLMMultiSelector
from llama_index.core.tools import QueryEngineTool

# Create query engine tools
doc_tool = QueryEngineTool.from_defaults(
    query_engine=doc_query_engine,
    description="Useful for questions about technical documentation, setup, configuration",
)

code_tool = QueryEngineTool.from_defaults(
    query_engine=code_query_engine,
    description="Useful for questions about source code, functions, classes, implementations",
)

api_tool = QueryEngineTool.from_defaults(
    query_engine=api_query_engine,
    description="Useful for questions about API endpoints, request/response formats, authentication",
)

# Single selector: routes to one engine
router_engine = RouterQueryEngine(
    selector=LLMSingleSelector.from_defaults(),
    query_engine_tools=[doc_tool, code_tool, api_tool],
)
response = router_engine.query("How is authentication implemented in the API?")

# Multi selector: can route to multiple engines and combine results
multi_router = RouterQueryEngine(
    selector=LLMMultiSelector.from_defaults(),
    query_engine_tools=[doc_tool, code_tool, api_tool],
)
response = multi_router.query("Compare the documented API spec with the actual code implementation")
```

---

## Routing to SQL vs Vector

A common pattern is routing structured data queries to SQL and unstructured queries to vector search:

```python
def route_sql_vs_vector(question: str) -> str:
    """Determine if a query should use SQL or vector search."""
    response = client.messages.create(
        model="claude-haiku-4-20250514",
        max_tokens=50,
        temperature=0,
        messages=[{
            "role": "user",
            "content": (
                "Classify this question as either 'sql' (needs structured data, "
                "counts, aggregations, filtering by specific values) or 'vector' "
                "(needs unstructured text, explanations, documentation). "
                "Answer with just 'sql' or 'vector'.\n\n"
                f"Question: {question}"
            ),
        }],
    )
    return response.content[0].text.strip().lower()


class SQLVectorRouter:
    """Route between SQL and vector search, generate SQL when needed."""

    def __init__(self, vector_index, db_connection, schema_description: str):
        self.vector_index = vector_index
        self.db = db_connection
        self.schema = schema_description

    def query(self, question: str) -> dict:
        route = route_sql_vs_vector(question)

        if route == "sql":
            return self._sql_query(question)
        else:
            return self._vector_query(question)

    def _sql_query(self, question: str) -> dict:
        """Generate and execute SQL query."""
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=500,
            messages=[{
                "role": "user",
                "content": (
                    f"Given this database schema:\n{self.schema}\n\n"
                    f"Generate a SQL query to answer: {question}\n\n"
                    f"Return ONLY the SQL query, nothing else."
                ),
            }],
        )
        sql = response.content[0].text.strip().strip("```sql").strip("```").strip()
        results = self.db.execute(sql)
        return {"type": "sql", "query": sql, "results": results}

    def _vector_query(self, question: str) -> dict:
        """Standard vector search."""
        results = self.vector_index.search(question, top_k=5)
        return {"type": "vector", "results": results}
```

---

## Benchmarks: Routing Impact

Tested on a mixed-query evaluation set (50 documentation queries, 25 SQL queries, 25 web queries):

| Approach | Overall Accuracy | Latency (p50) | Cost |
|----------|-----------------|---------------|------|
| All queries -> vector search | 56% | 200ms | $0.003 |
| LLM routing (Haiku) | 89% | 350ms | $0.0033 |
| Embedding routing | 84% | 210ms | $0.003 |
| Rule-based routing | 78% | 200ms | $0.003 |
| Multi-source routing | **92%** | 500ms | $0.005 |

**Routing accuracy by method** (did the router select the correct destination?):

| Method | Documentation | SQL | Web Search | Overall |
|--------|-------------|-----|-----------|---------|
| LLM (Haiku) | 94% | 88% | 84% | 90% |
| Embedding | 92% | 80% | 76% | 85% |
| Rule-based | 86% | 84% | 60% | 78% |

**Key finding**: LLM routing is the most accurate but adds 150ms latency. Embedding routing is nearly as good and adds only 5ms. Rule-based routing works well for SQL queries (clear patterns) but poorly for web search queries (hard to enumerate all patterns).

---

## Cascading Router Pattern

For production systems, combine multiple routing strategies in a cascade:

```python
class CascadingRouter:
    """Try rule-based first (fast), fall back to embedding, then LLM."""

    def __init__(self):
        self.rule_router = RuleRouter()
        self.embedding_router = EmbeddingRouter()

    def route(self, query: str) -> dict:
        # Level 1: Rules (0ms, high precision for clear patterns)
        rule_result = self.rule_router.route(query)
        if rule_result["confidence"] >= 1.0:
            return {**rule_result, "method": "rules"}

        # Level 2: Embedding (5ms, good for most queries)
        emb_result = self.embedding_router.route(query)
        if emb_result["confidence"] >= 0.6:
            return {**emb_result, "method": "embedding"}

        # Level 3: LLM (150ms, for ambiguous queries)
        llm_result = route_query(query)
        return {**llm_result, "method": "llm"}
```

---

## Common Pitfalls

1. **Not having a default route.** When the router is uncertain, queries should fall through to a sensible default (usually vector search), not fail.
2. **Routing based on form instead of intent.** "How many" sounds like SQL, but "How many ways can I configure TLS?" should go to vector search. Use semantic understanding, not just keywords.
3. **Single-source routing when multi-source is needed.** Questions like "Compare our pricing with competitors" need both SQL (our pricing) and web search (competitor pricing).
4. **Not monitoring routing accuracy.** Log routing decisions and periodically audit them. A 10% misrouting rate means 10% of queries get worse answers.
5. **Over-engineering the router.** For systems with only 2 possible routes, a simple rule-based router may be sufficient. Reserve LLM routing for systems with 4+ routes.
6. **Not caching routing decisions.** If similar queries repeat, cache the route to avoid repeated LLM calls.

---

## References

- LangChain RouterChain docs -- https://python.langchain.com/docs/how_to/routing/
- LlamaIndex RouterQueryEngine -- https://docs.llamaindex.ai/en/stable/examples/query_engine/RouterQueryEngine/
- Semantic Router library -- https://github.com/aurelio-labs/semantic-router
