# RAG Observability -- What to Trace in RAG Systems

## TL;DR

RAG pipelines have more failure modes than standard LLM applications because retrieval, ranking, and generation are separate stages that each can silently degrade. Observability for RAG means instrumenting every stage -- query processing, embedding, retrieval, reranking, context assembly, and generation -- with structured traces, metrics, and logs that let you answer: "Why did this query produce a bad answer?" The three pillars are: tracing (capturing the full request lifecycle with timing and inputs/outputs at each stage), metrics (quantitative signals like latency, relevance scores, token usage, and cache hit rates), and evaluation (automated quality scoring on faithfulness, relevance, and completeness). This overview covers what to instrument, the trace data model, key metrics, and integration patterns. Subsequent articles cover LangSmith/Langfuse and OpenTelemetry implementations in depth.

---

## Why RAG Needs Specialized Observability

### Failure Modes by Stage

| Stage | Failure Mode | Symptom | What to Log |
|-------|-------------|---------|-------------|
| Query processing | Bad rewrite loses intent | Wrong docs retrieved | Original query, rewritten query, rewrite model |
| Embedding | Model drift, wrong model version | Gradually worse retrieval | Embedding model, dimension, latency |
| Retrieval | Wrong k, stale index, bad filter | Irrelevant docs returned | Query vector, top-k, scores, doc IDs |
| Reranking | Reranker promotes wrong docs | Good docs demoted | Pre-rerank order, post-rerank order, scores |
| Context assembly | Truncation drops key info | Answer misses obvious info | Token count, docs included/excluded |
| Generation | Hallucination, refusal, verbosity | Bad user experience | Full prompt, response, usage, latency |

### The "Why Was This Answer Bad?" Problem

```
User: "What are the rate limits for the v2 API?"
Answer: "The rate limit is 100 requests per second."  (WRONG: it is 1000/min)

Without observability:
  -> No idea which stage failed
  -> Was the right doc retrieved? Was it truncated? Did the LLM misread it?

With observability:
  -> Trace shows: retriever returned doc about v1 API (wrong version)
  -> Root cause: metadata filter for api_version was not applied
  -> Fix: add version filter to retrieval query
```

---

## The RAG Trace Data Model

### Trace Structure

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Any
import uuid


@dataclass
class SpanData:
    """A single span in the RAG trace -- one stage of the pipeline."""

    span_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    name: str = ""
    span_type: str = ""  # "retrieval", "llm", "embedding", "rerank", "tool"
    parent_span_id: str | None = None
    start_time: datetime = field(default_factory=datetime.utcnow)
    end_time: datetime | None = None
    duration_ms: float = 0.0

    # Input/Output
    input_data: dict = field(default_factory=dict)
    output_data: dict = field(default_factory=dict)

    # Metadata
    metadata: dict = field(default_factory=dict)
    status: str = "ok"  # "ok", "error"
    error: str | None = None

    # Metrics specific to this span
    metrics: dict = field(default_factory=dict)


@dataclass
class RAGTrace:
    """Complete trace for one RAG request."""

    trace_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    session_id: str | None = None
    user_id: str | None = None
    start_time: datetime = field(default_factory=datetime.utcnow)
    end_time: datetime | None = None

    # The request
    query: str = ""
    rewritten_query: str | None = None

    # Spans
    spans: list[SpanData] = field(default_factory=list)

    # Final output
    answer: str = ""
    sources: list[dict] = field(default_factory=list)

    # Quality scores (filled by evaluators)
    scores: dict = field(default_factory=dict)

    # Aggregate metrics
    total_latency_ms: float = 0.0
    total_tokens: int = 0
    total_cost_usd: float = 0.0

    def add_span(self, span: SpanData) -> None:
        self.spans.append(span)

    def get_spans_by_type(self, span_type: str) -> list[SpanData]:
        return [s for s in self.spans if s.span_type == span_type]

    def to_dict(self) -> dict:
        return {
            "trace_id": self.trace_id,
            "session_id": self.session_id,
            "user_id": self.user_id,
            "query": self.query,
            "rewritten_query": self.rewritten_query,
            "answer": self.answer,
            "sources": self.sources,
            "scores": self.scores,
            "total_latency_ms": self.total_latency_ms,
            "total_tokens": self.total_tokens,
            "total_cost_usd": self.total_cost_usd,
            "spans": [
                {
                    "span_id": s.span_id,
                    "name": s.name,
                    "span_type": s.span_type,
                    "duration_ms": s.duration_ms,
                    "input_data": s.input_data,
                    "output_data": s.output_data,
                    "metadata": s.metadata,
                    "status": s.status,
                    "metrics": s.metrics,
                }
                for s in self.spans
            ],
        }
```

### Instrumented RAG Pipeline

```python
import time
from contextlib import contextmanager


class InstrumentedRAGPipeline:
    """RAG pipeline with full observability instrumentation.

    Every stage emits a span with timing, inputs, outputs,
    and stage-specific metrics.
    """

    def __init__(self, retriever, reranker, llm, embedder):
        self.retriever = retriever
        self.reranker = reranker
        self.llm = llm
        self.embedder = embedder

    @contextmanager
    def _span(self, trace: RAGTrace, name: str, span_type: str):
        """Context manager that creates and records a span."""
        span = SpanData(name=name, span_type=span_type)
        start = time.perf_counter()
        try:
            yield span
            span.status = "ok"
        except Exception as e:
            span.status = "error"
            span.error = str(e)
            raise
        finally:
            span.duration_ms = (time.perf_counter() - start) * 1000
            span.end_time = datetime.utcnow()
            trace.add_span(span)

    def query(self, user_query: str, user_id: str = None) -> dict:
        trace = RAGTrace(query=user_query, user_id=user_id)
        pipeline_start = time.perf_counter()

        # Stage 1: Query rewriting
        with self._span(trace, "query_rewrite", "llm") as span:
            span.input_data = {"original_query": user_query}
            rewritten = self._rewrite_query(user_query)
            span.output_data = {"rewritten_query": rewritten}
            trace.rewritten_query = rewritten

        # Stage 2: Embedding
        with self._span(trace, "embed_query", "embedding") as span:
            span.input_data = {"text": rewritten}
            query_vector = self.embedder.embed_query(rewritten)
            span.output_data = {"dimension": len(query_vector)}
            span.metrics = {"embedding_dimension": len(query_vector)}

        # Stage 3: Retrieval
        with self._span(trace, "vector_search", "retrieval") as span:
            span.input_data = {"query": rewritten, "k": 10}
            raw_docs = self.retriever.invoke(rewritten)
            span.output_data = {
                "num_docs": len(raw_docs),
                "doc_ids": [
                    d.metadata.get("source", "unknown") for d in raw_docs
                ],
                "scores": [
                    d.metadata.get("score", None) for d in raw_docs
                ],
            }
            span.metrics = {"num_retrieved": len(raw_docs)}

        # Stage 4: Reranking
        with self._span(trace, "rerank", "rerank") as span:
            span.input_data = {
                "num_docs": len(raw_docs),
                "query": rewritten,
            }
            reranked = self.reranker.rerank(rewritten, raw_docs)
            span.output_data = {
                "reranked_order": [
                    d.metadata.get("source", "unknown") for d in reranked
                ],
                "reranked_scores": [
                    d.metadata.get("rerank_score", None)
                    for d in reranked
                ],
            }

        # Stage 5: Context assembly
        with self._span(trace, "context_assembly", "tool") as span:
            top_docs = reranked[:5]
            context = "\n\n".join(d.page_content for d in top_docs)
            context_tokens = len(context) // 4  # rough estimate
            span.input_data = {"num_docs": len(reranked)}
            span.output_data = {
                "docs_used": len(top_docs),
                "context_chars": len(context),
                "estimated_tokens": context_tokens,
            }
            span.metrics = {"context_token_estimate": context_tokens}

        # Stage 6: Generation
        with self._span(trace, "llm_generation", "llm") as span:
            span.input_data = {
                "context_length": len(context),
                "query": user_query,
            }
            response = self.llm.invoke(
                f"Context:\n{context}\n\nQuestion: {user_query}"
            )
            answer = response.content
            span.output_data = {"answer_length": len(answer)}
            span.metrics = {
                "input_tokens": getattr(
                    response, "usage", {}
                ).get("input_tokens", 0),
                "output_tokens": getattr(
                    response, "usage", {}
                ).get("output_tokens", 0),
            }

        # Finalize trace
        trace.answer = answer
        trace.sources = [d.metadata for d in top_docs]
        trace.total_latency_ms = (
            (time.perf_counter() - pipeline_start) * 1000
        )

        # Compute aggregate metrics
        trace.total_tokens = sum(
            s.metrics.get("input_tokens", 0)
            + s.metrics.get("output_tokens", 0)
            for s in trace.spans
        )

        return {
            "answer": answer,
            "trace": trace,
        }

    def _rewrite_query(self, query: str) -> str:
        """Placeholder for query rewriting logic."""
        return query
```

---

## Key Metrics to Track

### Latency Metrics

```python
@dataclass
class RAGLatencyMetrics:
    """Latency breakdown for a RAG request."""

    query_rewrite_ms: float = 0.0
    embedding_ms: float = 0.0
    retrieval_ms: float = 0.0
    reranking_ms: float = 0.0
    context_assembly_ms: float = 0.0
    generation_ms: float = 0.0
    total_ms: float = 0.0

    @classmethod
    def from_trace(cls, trace: RAGTrace) -> "RAGLatencyMetrics":
        """Extract latency metrics from a trace."""
        metrics = cls()
        for span in trace.spans:
            if span.name == "query_rewrite":
                metrics.query_rewrite_ms = span.duration_ms
            elif span.name == "embed_query":
                metrics.embedding_ms = span.duration_ms
            elif span.name == "vector_search":
                metrics.retrieval_ms = span.duration_ms
            elif span.name == "rerank":
                metrics.reranking_ms = span.duration_ms
            elif span.name == "context_assembly":
                metrics.context_assembly_ms = span.duration_ms
            elif span.name == "llm_generation":
                metrics.generation_ms = span.duration_ms
        metrics.total_ms = trace.total_latency_ms
        return metrics
```

### Quality Metrics

| Metric | What It Measures | Collection Method |
|--------|-----------------|-------------------|
| Faithfulness | Answer grounded in context | LLM-as-judge or NLI |
| Answer relevancy | Answer addresses the query | Embedding similarity |
| Context precision | Retrieved docs are relevant | LLM-as-judge |
| Context recall | Retrieved docs contain the answer | LLM-as-judge |
| TTFT | Time to first token | Streaming timestamp |
| Retrieval latency p99 | Worst-case retrieval time | Span duration |
| Cache hit rate | Semantic/prompt cache effectiveness | Cache instrumentation |
| User satisfaction | Thumbs up/down, follow-up rate | UI feedback |

### Operational Metrics

```python
from collections import defaultdict
import statistics


class RAGMetricsAggregator:
    """Aggregate RAG metrics over a time window for dashboards."""

    def __init__(self):
        self.latencies: list[float] = []
        self.stage_latencies: dict[str, list[float]] = defaultdict(list)
        self.token_counts: list[int] = []
        self.faithfulness_scores: list[float] = []
        self.error_count: int = 0
        self.total_count: int = 0

    def record_trace(self, trace: RAGTrace) -> None:
        """Record metrics from a completed trace."""
        self.total_count += 1
        self.latencies.append(trace.total_latency_ms)
        self.token_counts.append(trace.total_tokens)

        for span in trace.spans:
            self.stage_latencies[span.name].append(span.duration_ms)
            if span.status == "error":
                self.error_count += 1

        if "faithfulness" in trace.scores:
            self.faithfulness_scores.append(
                trace.scores["faithfulness"]
            )

    def summary(self) -> dict:
        """Generate summary statistics."""
        return {
            "total_requests": self.total_count,
            "error_rate": (
                self.error_count / self.total_count
                if self.total_count > 0
                else 0
            ),
            "latency": {
                "p50_ms": self._percentile(self.latencies, 50),
                "p95_ms": self._percentile(self.latencies, 95),
                "p99_ms": self._percentile(self.latencies, 99),
                "mean_ms": (
                    statistics.mean(self.latencies)
                    if self.latencies
                    else 0
                ),
            },
            "stage_latencies": {
                name: {
                    "p50_ms": self._percentile(vals, 50),
                    "p95_ms": self._percentile(vals, 95),
                }
                for name, vals in self.stage_latencies.items()
            },
            "tokens": {
                "mean": (
                    statistics.mean(self.token_counts)
                    if self.token_counts
                    else 0
                ),
                "total": sum(self.token_counts),
            },
            "faithfulness": {
                "mean": (
                    statistics.mean(self.faithfulness_scores)
                    if self.faithfulness_scores
                    else None
                ),
                "below_threshold": sum(
                    1 for s in self.faithfulness_scores if s < 0.7
                ),
            },
        }

    @staticmethod
    def _percentile(data: list[float], pct: int) -> float:
        if not data:
            return 0.0
        sorted_data = sorted(data)
        idx = int(len(sorted_data) * pct / 100)
        return sorted_data[min(idx, len(sorted_data) - 1)]
```

---

## Alerting Rules

### What to Alert On

```python
ALERT_RULES = {
    "latency_p99_above_5s": {
        "condition": lambda m: m["latency"]["p99_ms"] > 5000,
        "severity": "warning",
        "message": "RAG p99 latency exceeds 5 seconds",
    },
    "latency_p99_above_10s": {
        "condition": lambda m: m["latency"]["p99_ms"] > 10000,
        "severity": "critical",
        "message": "RAG p99 latency exceeds 10 seconds",
    },
    "error_rate_above_5pct": {
        "condition": lambda m: m["error_rate"] > 0.05,
        "severity": "critical",
        "message": "RAG error rate exceeds 5%",
    },
    "faithfulness_drop": {
        "condition": lambda m: (
            m["faithfulness"]["mean"] is not None
            and m["faithfulness"]["mean"] < 0.7
        ),
        "severity": "warning",
        "message": "Mean faithfulness score dropped below 0.7",
    },
    "retrieval_latency_spike": {
        "condition": lambda m: (
            m["stage_latencies"]
            .get("vector_search", {})
            .get("p95_ms", 0)
            > 500
        ),
        "severity": "warning",
        "message": "Retrieval p95 latency exceeds 500ms",
    },
}


def check_alerts(metrics: dict) -> list[dict]:
    """Evaluate alert rules against current metrics."""
    triggered = []
    for rule_name, rule in ALERT_RULES.items():
        try:
            if rule["condition"](metrics):
                triggered.append({
                    "rule": rule_name,
                    "severity": rule["severity"],
                    "message": rule["message"],
                })
        except (KeyError, TypeError):
            pass
    return triggered
```

---

## Common Pitfalls

1. **Only tracing the LLM call.** LLM generation is just one stage. If retrieval returns the wrong documents, the LLM will produce a confident wrong answer. Trace every stage.
2. **Not capturing retrieval scores.** Without similarity scores, you cannot tell if the retriever is returning marginally relevant documents (score 0.3) or highly relevant ones (score 0.9). Log all scores.
3. **Logging full document content in every trace.** At scale, this generates terabytes of trace data. Log document IDs and scores; store full content separately in the vector store where it can be looked up on demand.
4. **Not correlating traces with user feedback.** A trace without a quality label is useful for debugging but not for systematic improvement. Connect user thumbs-up/thumbs-down signals to their corresponding traces.
5. **Sampling too aggressively.** In low-traffic RAG systems, sampling 1% of traces means you miss rare failure patterns. For systems under 1000 QPS, trace 100% of requests. Sample only when trace storage becomes a bottleneck.
6. **Not alerting on slow degradation.** A sudden spike in latency triggers alerts, but a 5% weekly decline in faithfulness scores goes unnoticed. Set trend-based alerts that compare weekly averages.

---

## References

- LangSmith documentation: https://docs.smith.langchain.com/
- Langfuse documentation: https://langfuse.com/docs
- OpenTelemetry GenAI semantic conventions: https://opentelemetry.io/docs/specs/semconv/gen-ai/
- Brundage et al. "Tracing and Debugging LLM Applications" (2024)
