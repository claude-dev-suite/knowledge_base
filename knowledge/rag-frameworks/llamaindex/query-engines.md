# LlamaIndex Query Engines Taxonomy

## Overview

A query engine is LlamaIndex's interface for getting answers from your indexed data. It takes a natural language query, retrieves relevant context, and synthesizes a response using an LLM. LlamaIndex provides multiple query engine types, each designed for different use cases: simple vector retrieval, citation-tracked answers, multi-source decomposition, intent-based routing, SQL integration, and more. This guide covers each type with practical code.

---

## VectorStoreIndex Query Engine (Default)

The most common query engine. Retrieves the top-k most similar chunks and synthesizes an answer:

```python
from llama_index.core import VectorStoreIndex, Settings
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding

Settings.llm = OpenAI(model="gpt-4o-mini")
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")

index = VectorStoreIndex.from_documents(documents)

query_engine = index.as_query_engine(
    similarity_top_k=5,
    response_mode="compact",
)

response = query_engine.query("How do I configure HNSW parameters?")
print(response.response)

# Access source nodes
for node in response.source_nodes:
    print(f"Score: {node.score:.4f} | Source: {node.node.metadata.get('source', 'unknown')}")
    print(f"Text: {node.node.text[:100]}...")
```

### Response Modes

```python
# compact: stuff all context into one prompt (best for most cases)
engine = index.as_query_engine(response_mode="compact")

# tree_summarize: recursively summarize chunks in a tree
# Good when you have many relevant chunks that exceed context window
engine = index.as_query_engine(response_mode="tree_summarize")

# refine: iterate through chunks one by one, refining the answer
# Most thorough but slowest (one LLM call per chunk)
engine = index.as_query_engine(response_mode="refine")

# no_text: return retrieved nodes without LLM synthesis
# Useful when you handle generation yourself
engine = index.as_query_engine(response_mode="no_text")
response = engine.query("HNSW tuning")
# response.source_nodes contains the chunks, response.response is empty

# accumulate: generate a separate response for each chunk, then concatenate
engine = index.as_query_engine(response_mode="accumulate")
```

---

## CitationQueryEngine

Returns answers with inline citations pointing to specific source chunks. Essential for applications where users need to verify claims:

```python
from llama_index.core.query_engine import CitationQueryEngine

citation_engine = CitationQueryEngine.from_args(
    index,
    similarity_top_k=5,
    citation_chunk_size=512,
)

response = citation_engine.query("What distance functions does pgvector support?")
print(response.response)
# Output: "pgvector supports L2 (Euclidean) distance [1], cosine distance [2],
#          and inner product [3]..."

# Access cited sources
for i, node in enumerate(response.source_nodes, 1):
    print(f"[{i}] {node.node.metadata.get('source', 'unknown')}")
    print(f"    {node.node.text[:150]}...")
```

### Custom Citation Format

```python
from llama_index.core.query_engine import CitationQueryEngine
from llama_index.core.prompts import PromptTemplate

CITATION_QA_TEMPLATE = PromptTemplate(
    "Context information is below.\n"
    "---------------------\n"
    "{context_str}\n"
    "---------------------\n"
    "Given the context information and not prior knowledge, "
    "answer the query. Cite sources using [Source N] format "
    "where N is the source number.\n"
    "Query: {query_str}\n"
    "Answer: "
)

citation_engine = CitationQueryEngine.from_args(
    index,
    similarity_top_k=5,
    text_qa_template=CITATION_QA_TEMPLATE,
)
```

---

## SubQuestionQueryEngine

Decomposes a complex query into simpler sub-questions, routes each to the appropriate data source, and synthesizes the final answer. Essential for multi-source RAG:

```python
from llama_index.core.query_engine import SubQuestionQueryEngine
from llama_index.core.tools import QueryEngineTool, ToolMetadata

# Create separate indexes for different data sources
api_index = VectorStoreIndex.from_documents(api_docs)
tutorial_index = VectorStoreIndex.from_documents(tutorial_docs)
changelog_index = VectorStoreIndex.from_documents(changelog_docs)

# Wrap each as a tool with description
query_engine_tools = [
    QueryEngineTool(
        query_engine=api_index.as_query_engine(),
        metadata=ToolMetadata(
            name="api_reference",
            description="API reference documentation with function signatures, "
                        "parameters, and return types",
        ),
    ),
    QueryEngineTool(
        query_engine=tutorial_index.as_query_engine(),
        metadata=ToolMetadata(
            name="tutorials",
            description="Step-by-step tutorials and how-to guides",
        ),
    ),
    QueryEngineTool(
        query_engine=changelog_index.as_query_engine(),
        metadata=ToolMetadata(
            name="changelog",
            description="Version history, breaking changes, and migration guides",
        ),
    ),
]

# Create sub-question engine
sub_q_engine = SubQuestionQueryEngine.from_defaults(
    query_engine_tools=query_engine_tools,
)

# Complex query that spans multiple sources
response = sub_q_engine.query(
    "What changed in the pgvector HNSW API between v0.6 and v0.7, "
    "and how do I migrate my existing indexes?"
)
print(response.response)

# The engine internally:
# 1. Decomposes into: "What changed in pgvector HNSW between v0.6 and v0.7?" -> changelog
#                     "How to migrate HNSW indexes to new API?" -> tutorials
#                     "What is the current HNSW API?" -> api_reference
# 2. Queries each tool
# 3. Synthesizes a unified answer
```

---

## RouterQueryEngine

Routes queries to different underlying engines based on the query's intent. Unlike SubQuestionQueryEngine (which uses multiple engines for one query), RouterQueryEngine picks the single best engine:

```python
from llama_index.core.query_engine import RouterQueryEngine
from llama_index.core.selectors import LLMSingleSelector, LLMMultiSelector

# Different engines for different query types
vector_engine = index.as_query_engine(similarity_top_k=5)
summary_engine = summary_index.as_query_engine(response_mode="tree_summarize")
keyword_engine = keyword_index.as_query_engine()

# Router with LLM-based selection
router_engine = RouterQueryEngine(
    selector=LLMSingleSelector.from_defaults(),  # picks one engine
    query_engine_tools=[
        QueryEngineTool(
            query_engine=vector_engine,
            metadata=ToolMetadata(
                name="specific_questions",
                description="Use for specific technical questions about code, "
                            "configuration, or APIs",
            ),
        ),
        QueryEngineTool(
            query_engine=summary_engine,
            metadata=ToolMetadata(
                name="summarization",
                description="Use for broad questions asking for overviews or "
                            "summaries of topics",
            ),
        ),
        QueryEngineTool(
            query_engine=keyword_engine,
            metadata=ToolMetadata(
                name="keyword_lookup",
                description="Use for looking up specific terms, error codes, "
                            "or function names",
            ),
        ),
    ],
)

# The router decides which engine to use
response = router_engine.query("What is pgvector?")  # -> summary engine
response = router_engine.query("How to set ef_construction?")  # -> vector engine
response = router_engine.query("Error ORA-01017")  # -> keyword engine
```

### Multi-Selector Router

For queries that need multiple engines:

```python
# Multi-selector: can pick multiple engines and combine results
router_engine = RouterQueryEngine(
    selector=LLMMultiSelector.from_defaults(),
    query_engine_tools=query_engine_tools,
)
```

---

## SQL Query Engine

Converts natural language to SQL and executes against a database:

```python
# Install: pip install llama-index-llms-openai sqlalchemy
from llama_index.core import SQLDatabase
from llama_index.core.query_engine import NLSQLTableQueryEngine
from sqlalchemy import create_engine

# Connect to database
engine = create_engine("postgresql://user:pass@localhost/mydb")
sql_database = SQLDatabase(
    engine,
    include_tables=["products", "orders", "customers"],
)

# Natural language to SQL
sql_engine = NLSQLTableQueryEngine(
    sql_database=sql_database,
    tables=["products", "orders", "customers"],
)

response = sql_engine.query("What are the top 5 products by revenue this month?")
print(response.response)
print(f"SQL: {response.metadata['sql_query']}")
```

### Combined SQL + Vector Engine

For questions that need both structured data and unstructured knowledge:

```python
from llama_index.core.query_engine import SQLAutoVectorQueryEngine
from llama_index.core.tools import QueryEngineTool

sql_tool = QueryEngineTool.from_defaults(
    query_engine=sql_engine,
    description="Use for questions about product data, sales numbers, "
                "customer statistics, or anything involving database queries",
)

vector_tool = QueryEngineTool.from_defaults(
    query_engine=vector_engine,
    description="Use for questions about product documentation, "
                "technical specifications, or how-to guides",
)

# Combined engine
combined_engine = RouterQueryEngine(
    selector=LLMSingleSelector.from_defaults(),
    query_engine_tools=[sql_tool, vector_tool],
)

# Routes automatically
response = combined_engine.query("How many orders last month?")  # -> SQL
response = combined_engine.query("How to use the bulk order API?")  # -> vector
```

---

## PandasQueryEngine

For querying DataFrames with natural language:

```python
from llama_index.experimental.query_engine.pandas import PandasQueryEngine
import pandas as pd

df = pd.DataFrame({
    "product": ["Widget A", "Widget B", "Gadget C"],
    "price": [29.99, 49.99, 99.99],
    "units_sold": [1500, 800, 200],
    "category": ["widgets", "widgets", "gadgets"],
})

pandas_engine = PandasQueryEngine(
    df=df,
    verbose=True,
)

response = pandas_engine.query("What is the total revenue by category?")
print(response.response)
# Also prints the generated pandas code
```

---

## Custom Query Engines

Build your own query engine by subclassing `CustomQueryEngine`:

```python
from llama_index.core.query_engine import CustomQueryEngine
from llama_index.core.retrievers import BaseRetriever
from llama_index.llms.openai import OpenAI
from llama_index.core.schema import NodeWithScore
from llama_index.core.response_synthesizers import get_response_synthesizer

class HybridRAGQueryEngine(CustomQueryEngine):
    """Query engine that combines vector retrieval with BM25."""

    retriever: BaseRetriever
    llm: OpenAI

    def custom_query(self, query_str: str):
        # 1. Retrieve from vector index
        vector_nodes = self.retriever.retrieve(query_str)

        # 2. BM25 search (custom logic)
        bm25_nodes = self._bm25_search(query_str)

        # 3. Fuse results (RRF)
        fused_nodes = self._rrf_fuse(vector_nodes, bm25_nodes)

        # 4. Synthesize response
        synthesizer = get_response_synthesizer(
            llm=self.llm,
            response_mode="compact",
        )

        response = synthesizer.synthesize(
            query=query_str,
            nodes=fused_nodes,
        )

        return response

    def _bm25_search(self, query: str) -> list[NodeWithScore]:
        # Your BM25 implementation here
        ...

    def _rrf_fuse(self, list1, list2, k=60):
        # RRF fusion
        scores = {}
        for rank, node in enumerate(list1, 1):
            scores[node.node.node_id] = scores.get(node.node.node_id, 0) + 1/(k + rank)
            scores[node.node.node_id + "_node"] = node
        for rank, node in enumerate(list2, 1):
            scores[node.node.node_id] = scores.get(node.node.node_id, 0) + 1/(k + rank)
            scores[node.node.node_id + "_node"] = node

        # Sort by RRF score
        node_ids = [k for k in scores if not k.endswith("_node")]
        node_ids.sort(key=lambda x: scores[x], reverse=True)

        return [scores[nid + "_node"] for nid in node_ids[:10]]


# Usage
engine = HybridRAGQueryEngine(
    retriever=index.as_retriever(similarity_top_k=20),
    llm=OpenAI(model="gpt-4o-mini"),
)
response = engine.query("How to optimize database queries?")
```

---

## Query Engine with Post-Processing

Add post-retrieval processing (re-ranking, filtering, deduplication):

```python
from llama_index.core.postprocessor import (
    SimilarityPostprocessor,
    KeywordNodePostprocessor,
    MetadataReplacementPostProcessor,
    SentenceTransformerRerank,
)

# Similarity threshold (remove low-relevance chunks)
similarity_filter = SimilarityPostprocessor(similarity_cutoff=0.7)

# Keyword filter (require specific terms in results)
keyword_filter = KeywordNodePostprocessor(
    required_keywords=["pgvector"],
    exclude_keywords=["deprecated"],
)

# Cross-encoder re-ranker (most impactful quality improvement)
reranker = SentenceTransformerRerank(
    model="cross-encoder/ms-marco-MiniLM-L-6-v2",
    top_n=5,
)

# Metadata replacement (for sentence window retrieval)
# Replaces node text with the surrounding window for better context
window_replacer = MetadataReplacementPostProcessor(
    target_metadata_key="window",
)

# Combine in query engine
query_engine = index.as_query_engine(
    similarity_top_k=20,           # retrieve many candidates
    node_postprocessors=[
        similarity_filter,          # filter low-relevance
        keyword_filter,             # require keywords
        reranker,                   # re-rank with cross-encoder (returns top 5)
    ],
)

response = query_engine.query("How to configure HNSW in pgvector?")
```

---

## Streaming Responses

```python
query_engine = index.as_query_engine(
    streaming=True,
    similarity_top_k=5,
)

streaming_response = query_engine.query("Explain HNSW indexing")

# Stream token by token
for text in streaming_response.response_gen:
    print(text, end="", flush=True)
print()  # newline at end

# Async streaming
async_engine = index.as_query_engine(streaming=True)
streaming_response = await async_engine.aquery("Explain HNSW indexing")
async for text in streaming_response.async_response_gen():
    print(text, end="", flush=True)
```

---

## Query Engine Decision Guide

| Query Type | Engine | Why |
|---|---|---|
| Simple factual question | VectorStoreIndex.as_query_engine | Standard RAG, retrieve + synthesize |
| Needs source attribution | CitationQueryEngine | Inline citations with source references |
| Spans multiple data sources | SubQuestionQueryEngine | Decomposes into sub-questions per source |
| Different query types need different handling | RouterQueryEngine | Routes to specialized engines |
| Structured data questions | NLSQLTableQueryEngine | Natural language to SQL |
| DataFrame analysis | PandasQueryEngine | Natural language to pandas |
| Needs re-ranking | VectorStoreIndex + SentenceTransformerRerank | Two-stage retrieve + re-rank |
| Custom retrieval logic | CustomQueryEngine | Full control over pipeline |

---

## Common Pitfalls

1. **Using default similarity_top_k=2**: retrieving only 2 chunks often misses relevant context. Start with 5-10, then tune based on evaluation.

2. **Not adding a re-ranker**: a cross-encoder re-ranker is the single most impactful quality improvement. Retrieve 20, re-rank to 5.

3. **Using SubQuestionQueryEngine for simple queries**: the overhead of decomposition and multi-source querying adds latency. Use it only when queries genuinely span multiple sources.

4. **Not providing good tool descriptions for RouterQueryEngine**: the LLM selector relies on the description to route queries. Vague descriptions lead to wrong routing.

5. **Forgetting response_mode for large contexts**: if you retrieve many chunks, `compact` mode may exceed the context window. Use `tree_summarize` for large result sets.

6. **Not testing each query engine type on your data**: the best engine depends on your documents, queries, and quality requirements. Test at least VectorStoreIndex and CitationQueryEngine before choosing.

---

## References

- LlamaIndex Query Engines: https://docs.llamaindex.ai/en/stable/module_guides/deploying/query_engine/
- LlamaIndex Response Synthesizers: https://docs.llamaindex.ai/en/stable/module_guides/querying/response_synthesizers/
- LlamaIndex Node Postprocessors: https://docs.llamaindex.ai/en/stable/module_guides/querying/node_postprocessors/
