# LangSmith and Langfuse -- Setup, Trace Structure, Retrieval Spans, and Scoring

## TL;DR

LangSmith (by LangChain) and Langfuse are the two leading observability platforms for LLM applications, each offering trace collection, evaluation, and dataset management with first-class RAG support. LangSmith integrates natively with LangChain and provides built-in evaluators for faithfulness and relevance. Langfuse is open-source, self-hostable, and framework-agnostic with a decorator-based Python SDK. This article covers setup for both platforms, trace structure and how RAG stages map to spans, retrieval-specific instrumentation (logging scores, document IDs, reranking), and automated scoring pipelines that evaluate traces offline.

---

## LangSmith Setup and Configuration

### Installation and Authentication

```python
# pip install langsmith langchain langchain-anthropic langchain-openai

import os

# Set environment variables for LangSmith
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "ls-..."  # Your LangSmith API key
os.environ["LANGCHAIN_PROJECT"] = "rag-production"  # Project name
os.environ["LANGCHAIN_ENDPOINT"] = "https://api.smith.langchain.com"
```

### Automatic Tracing with LangChain

LangChain automatically sends traces to LangSmith when `LANGCHAIN_TRACING_V2` is enabled:

```python
from langchain_anthropic import ChatAnthropic
from langchain_openai import OpenAIEmbeddings
from langchain_chroma import Chroma
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain_core.prompts import ChatPromptTemplate


# All of these steps are automatically traced in LangSmith
llm = ChatAnthropic(model="claude-sonnet-4-20250514")
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma(embedding_function=embeddings)
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

prompt = ChatPromptTemplate.from_messages([
    ("system", "Answer based on context:\n{context}"),
    ("human", "{input}"),
])

chain = create_stuff_documents_chain(llm, prompt)
rag_chain = create_retrieval_chain(retriever, chain)

# This call generates a complete trace in LangSmith with:
# - Retriever span (query, retrieved docs, scores)
# - LLM span (prompt, response, tokens, latency)
# - Chain span (combining retriever + LLM)
result = rag_chain.invoke({"input": "What are the rate limits?"})
```

### Custom Tracing with @traceable

For non-LangChain components, use the `@traceable` decorator:

```python
from langsmith import traceable
from langsmith.run_helpers import get_current_run_tree


@traceable(name="rag_pipeline", run_type="chain")
def rag_query(query: str) -> dict:
    """Full RAG pipeline with custom tracing."""

    # Stage 1: Rewrite
    rewritten = rewrite_query(query)

    # Stage 2: Retrieve
    docs = retrieve_documents(rewritten)

    # Stage 3: Rerank
    reranked = rerank_documents(rewritten, docs)

    # Stage 4: Generate
    answer = generate_answer(query, reranked)

    return {
        "answer": answer,
        "sources": [d.metadata for d in reranked[:5]],
    }


@traceable(name="query_rewrite", run_type="llm")
def rewrite_query(query: str) -> str:
    """Rewrite query for better retrieval."""
    from anthropic import Anthropic
    client = Anthropic()
    response = client.messages.create(
        model="claude-haiku-4-20250514",
        max_tokens=200,
        messages=[{
            "role": "user",
            "content": f"Rewrite as a search query: {query}",
        }],
    )
    return response.content[0].text.strip()


@traceable(name="vector_search", run_type="retriever")
def retrieve_documents(query: str) -> list:
    """Retrieve documents from vector store."""
    docs = vectorstore.similarity_search_with_score(query, k=10)
    # LangSmith will capture the retriever inputs and outputs
    return [doc for doc, score in docs]


@traceable(name="rerank", run_type="chain")
def rerank_documents(query: str, docs: list) -> list:
    """Rerank retrieved documents."""
    # Custom reranking logic
    return docs[:5]


@traceable(name="llm_generation", run_type="llm")
def generate_answer(query: str, docs: list) -> str:
    """Generate answer from documents."""
    context = "\n\n".join(d.page_content for d in docs)
    llm = ChatAnthropic(model="claude-sonnet-4-20250514")
    response = llm.invoke(
        f"Context:\n{context}\n\nQuestion: {query}"
    )
    return response.content
```

### Adding Metadata and Feedback

```python
from langsmith import Client

ls_client = Client()


def add_feedback_to_trace(
    run_id: str,
    key: str,
    score: float,
    comment: str = "",
) -> None:
    """Add evaluation feedback to a LangSmith trace.

    Feedback can be user-provided (thumbs up/down) or
    automated (evaluation scores).
    """
    ls_client.create_feedback(
        run_id=run_id,
        key=key,
        score=score,
        comment=comment,
    )


def add_rag_evaluation_scores(
    run_id: str,
    faithfulness: float,
    relevance: float,
    context_precision: float,
) -> None:
    """Add multiple RAG evaluation scores to a trace."""
    scores = {
        "faithfulness": faithfulness,
        "answer_relevance": relevance,
        "context_precision": context_precision,
    }
    for key, score in scores.items():
        ls_client.create_feedback(
            run_id=run_id,
            key=key,
            score=score,
        )
```

### LangSmith Evaluation Datasets

```python
from langsmith import Client

client = Client()


def create_rag_eval_dataset(
    dataset_name: str,
    examples: list[dict],
) -> str:
    """Create a dataset in LangSmith for RAG evaluation.

    Each example has an input (query + context) and expected output.
    """
    dataset = client.create_dataset(
        dataset_name=dataset_name,
        description="RAG evaluation dataset",
    )

    for example in examples:
        client.create_example(
            inputs={
                "query": example["query"],
                "context": example.get("context", ""),
            },
            outputs={
                "answer": example["expected_answer"],
                "sources": example.get("expected_sources", []),
            },
            dataset_id=dataset.id,
        )

    return str(dataset.id)


def run_evaluation(
    dataset_name: str,
    rag_fn,
) -> dict:
    """Run RAG function against a LangSmith dataset and evaluate."""
    from langsmith.evaluation import evaluate

    def predict(inputs: dict) -> dict:
        result = rag_fn(inputs["query"])
        return {"answer": result["answer"]}

    def faithfulness_evaluator(run, example) -> dict:
        """Custom evaluator for faithfulness."""
        prediction = run.outputs["answer"]
        context = example.inputs.get("context", "")
        # Simplified scoring
        context_words = set(context.lower().split())
        answer_words = set(prediction.lower().split())
        overlap = len(answer_words & context_words) / max(
            len(answer_words), 1
        )
        return {"key": "faithfulness", "score": overlap}

    results = evaluate(
        predict,
        data=dataset_name,
        evaluators=[faithfulness_evaluator],
    )

    return results
```

---

## Langfuse Setup and Configuration

### Installation and Initialization

```python
# pip install langfuse

from langfuse import Langfuse

# Initialize Langfuse client
langfuse = Langfuse(
    public_key="pk-...",
    secret_key="sk-...",
    host="https://cloud.langfuse.com",  # or self-hosted URL
)
```

### Trace Structure with Decorators

```python
from langfuse.decorators import observe, langfuse_context


@observe()
def rag_pipeline(query: str) -> dict:
    """Full RAG pipeline traced by Langfuse.

    The @observe decorator creates a trace and automatically
    captures function inputs, outputs, and timing.
    """
    # Add metadata to the trace
    langfuse_context.update_current_trace(
        user_id="user-123",
        session_id="session-456",
        metadata={"pipeline_version": "v2.1"},
        tags=["production", "rag"],
    )

    rewritten = rewrite_query_lf(query)
    docs = retrieve_docs_lf(rewritten)
    reranked = rerank_docs_lf(rewritten, docs)
    answer = generate_answer_lf(query, reranked)

    # Score the trace
    langfuse_context.score_current_trace(
        name="user_satisfaction",
        value=1.0,  # Will be updated by user feedback
    )

    return {"answer": answer, "sources": reranked[:5]}


@observe(as_type="generation")
def rewrite_query_lf(query: str) -> str:
    """Rewrite query -- traced as a 'generation' span."""
    from anthropic import Anthropic
    client = Anthropic()

    langfuse_context.update_current_observation(
        model="claude-haiku-4-20250514",
        input={"query": query},
        metadata={"purpose": "query_rewrite"},
    )

    response = client.messages.create(
        model="claude-haiku-4-20250514",
        max_tokens=200,
        messages=[{
            "role": "user",
            "content": f"Rewrite for search: {query}",
        }],
    )

    result = response.content[0].text.strip()

    langfuse_context.update_current_observation(
        output={"rewritten_query": result},
        usage={
            "input": response.usage.input_tokens,
            "output": response.usage.output_tokens,
        },
    )

    return result


@observe(as_type="span")
def retrieve_docs_lf(query: str) -> list:
    """Retrieve documents -- traced as a generic span."""
    docs_with_scores = vectorstore.similarity_search_with_score(
        query, k=10
    )

    langfuse_context.update_current_observation(
        input={"query": query, "k": 10},
        output={
            "num_results": len(docs_with_scores),
            "doc_ids": [
                d.metadata.get("source", "?")
                for d, _ in docs_with_scores
            ],
            "scores": [float(s) for _, s in docs_with_scores],
        },
        metadata={"vector_store": "chroma"},
    )

    return [doc for doc, _ in docs_with_scores]


@observe(as_type="span")
def rerank_docs_lf(query: str, docs: list) -> list:
    """Rerank documents -- traced as a span with before/after ordering."""
    pre_order = [
        d.metadata.get("source", "?") for d in docs
    ]

    # Reranking logic here
    reranked = docs  # Placeholder

    post_order = [
        d.metadata.get("source", "?") for d in reranked
    ]

    langfuse_context.update_current_observation(
        input={"query": query, "num_docs": len(docs)},
        output={
            "pre_rerank_order": pre_order,
            "post_rerank_order": post_order,
        },
    )

    return reranked


@observe(as_type="generation")
def generate_answer_lf(query: str, docs: list) -> str:
    """Generate answer -- traced as a 'generation' span."""
    context = "\n\n".join(d.page_content for d in docs[:5])

    langfuse_context.update_current_observation(
        model="claude-sonnet-4-20250514",
        input={"query": query, "context_length": len(context)},
        model_parameters={"max_tokens": 2048, "temperature": 0.0},
    )

    from anthropic import Anthropic
    client = Anthropic()
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": f"Context:\n{context}\n\nQuestion: {query}",
        }],
    )

    answer = response.content[0].text

    langfuse_context.update_current_observation(
        output={"answer": answer},
        usage={
            "input": response.usage.input_tokens,
            "output": response.usage.output_tokens,
        },
    )

    return answer
```

### Langfuse Scoring and Evaluation

```python
from langfuse import Langfuse


def score_trace(
    langfuse_client: Langfuse,
    trace_id: str,
    scores: dict[str, float],
) -> None:
    """Add evaluation scores to a Langfuse trace.

    Scores can be added asynchronously after the request
    completes, enabling offline evaluation pipelines.
    """
    for name, value in scores.items():
        langfuse_client.score(
            trace_id=trace_id,
            name=name,
            value=value,
        )


def batch_evaluate_traces(
    langfuse_client: Langfuse,
    evaluator_fn,
    limit: int = 100,
) -> dict:
    """Evaluate recent traces that have not been scored yet.

    This runs as a batch job (e.g., hourly cron) to score
    traces that were collected during production traffic.
    """
    # Fetch recent traces without scores
    traces = langfuse_client.fetch_traces(
        limit=limit,
        order_by="timestamp",
    )

    scored = 0
    for trace in traces.data:
        if not trace.scores:  # Not yet scored
            # Extract inputs/outputs for evaluation
            query = trace.input.get("query", "")
            answer = trace.output.get("answer", "")
            context = trace.metadata.get("context", "")

            if query and answer:
                scores = evaluator_fn(query, answer, context)
                score_trace(
                    langfuse_client,
                    trace.id,
                    scores,
                )
                scored += 1

    langfuse_client.flush()
    return {"traces_evaluated": scored, "total_fetched": len(traces.data)}
```

### Langfuse Dataset Management

```python
def create_langfuse_eval_dataset(
    langfuse_client: Langfuse,
    name: str,
    items: list[dict],
) -> None:
    """Create an evaluation dataset in Langfuse.

    Datasets can be created from production traces (extracting
    interesting cases) or manually curated.
    """
    langfuse_client.create_dataset(name=name)

    for item in items:
        langfuse_client.create_dataset_item(
            dataset_name=name,
            input={"query": item["query"]},
            expected_output={"answer": item["expected_answer"]},
            metadata={
                "category": item.get("category", "general"),
                "difficulty": item.get("difficulty", "medium"),
            },
        )

    langfuse_client.flush()


def run_langfuse_eval(
    langfuse_client: Langfuse,
    dataset_name: str,
    rag_fn,
    run_name: str = "eval-run",
) -> dict:
    """Run RAG function against a Langfuse dataset."""
    dataset = langfuse_client.get_dataset(dataset_name)

    results = []
    for item in dataset.items:
        query = item.input["query"]

        # Run the RAG pipeline (traced by Langfuse)
        output = rag_fn(query)

        # Link this trace to the dataset item
        item.link(
            trace_id=langfuse_context.get_current_trace_id(),
            run_name=run_name,
        )

        results.append({
            "query": query,
            "expected": item.expected_output.get("answer", ""),
            "actual": output.get("answer", ""),
        })

    langfuse_client.flush()
    return {"results": results, "count": len(results)}
```

---

## Comparing LangSmith and Langfuse

| Feature | LangSmith | Langfuse |
|---------|-----------|---------|
| Hosting | Cloud only | Cloud + self-hosted (open source) |
| Framework integration | LangChain native | Framework-agnostic |
| Pricing | Per-trace pricing | Free tier + usage-based |
| Trace structure | Runs + child runs | Traces + observations |
| Evaluation | Built-in evaluators | Custom scoring + datasets |
| Python SDK | `langsmith` + `@traceable` | `langfuse` + `@observe` |
| Retrieval span support | Native retriever type | Manual span annotation |
| Dataset management | Built-in datasets | Built-in datasets |
| Prompt management | Yes (prompt hub) | Yes (prompt management) |
| User feedback | `create_feedback` | `score` on trace |

---

## Production Integration Pattern

```python
import os
from enum import Enum


class ObservabilityBackend(Enum):
    LANGSMITH = "langsmith"
    LANGFUSE = "langfuse"
    BOTH = "both"


def configure_observability(
    backend: ObservabilityBackend = ObservabilityBackend.LANGFUSE,
) -> dict:
    """Configure the observability backend based on environment."""
    config = {}

    if backend in (ObservabilityBackend.LANGSMITH, ObservabilityBackend.BOTH):
        os.environ["LANGCHAIN_TRACING_V2"] = "true"
        os.environ["LANGCHAIN_PROJECT"] = os.getenv(
            "LANGSMITH_PROJECT", "rag-production"
        )
        config["langsmith"] = True

    if backend in (ObservabilityBackend.LANGFUSE, ObservabilityBackend.BOTH):
        config["langfuse"] = Langfuse(
            public_key=os.getenv("LANGFUSE_PUBLIC_KEY"),
            secret_key=os.getenv("LANGFUSE_SECRET_KEY"),
            host=os.getenv(
                "LANGFUSE_HOST", "https://cloud.langfuse.com"
            ),
        )

    return config
```

---

## Common Pitfalls

1. **Not enabling tracing in production.** Many teams only trace during development. Production is where the unexpected failures happen. Enable tracing for 100% of production traffic (both LangSmith and Langfuse handle the volume).
2. **Not linking user feedback to traces.** If a user clicks "thumbs down," that signal must be connected to the specific trace. Without this link, you cannot improve systematically.
3. **Using LangSmith only for LangChain apps.** LangSmith's `@traceable` decorator works with any Python function, not just LangChain chains. Use it for custom pipelines too.
4. **Not flushing Langfuse in serverless environments.** Langfuse batches events. In serverless (Lambda, Cloud Functions), the process may terminate before events are flushed. Call `langfuse.flush()` before returning.
5. **Logging full document content in every trace.** This bloats trace storage. Log document IDs and scores instead; retrieve full content on demand when debugging a specific trace.
6. **Not setting up evaluation datasets early.** Start collecting interesting production queries (especially failures) into evaluation datasets from day one. You need these for regression testing when you change the pipeline.

---

## References

- LangSmith documentation: https://docs.smith.langchain.com/
- LangSmith evaluation: https://docs.smith.langchain.com/evaluation
- Langfuse documentation: https://langfuse.com/docs
- Langfuse Python SDK: https://langfuse.com/docs/sdk/python
- Langfuse self-hosting: https://langfuse.com/docs/deployment/self-host
