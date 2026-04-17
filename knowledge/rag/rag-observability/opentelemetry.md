# OpenTelemetry for RAG -- GenAI Semantic Conventions, Custom Spans, and Alerting

## TL;DR

OpenTelemetry (OTel) is the vendor-neutral observability standard for distributed systems, and its GenAI semantic conventions (introduced in 2024) define standardized span attributes for LLM calls, embeddings, and retrieval operations. For RAG pipelines, OTel provides a portable instrumentation layer that works with any backend (Jaeger, Grafana Tempo, Datadog, Honeycomb) and avoids vendor lock-in to LangSmith or Langfuse. This article covers the GenAI semantic conventions, custom span instrumentation for RAG-specific stages (retrieval, reranking, context assembly), metric collection via OTel Metrics API, and alerting pipelines using Prometheus and Grafana.

---

## GenAI Semantic Conventions

### What They Standardize

The OpenTelemetry GenAI semantic conventions define standard attribute names for LLM operations. This means traces from different frameworks and providers use the same attribute keys, enabling cross-tool analysis.

### Key Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `gen_ai.system` | string | LLM provider ("anthropic", "openai") |
| `gen_ai.request.model` | string | Model name ("claude-sonnet-4-20250514") |
| `gen_ai.request.max_tokens` | int | Max output tokens |
| `gen_ai.request.temperature` | float | Temperature setting |
| `gen_ai.response.model` | string | Actual model used (may differ from request) |
| `gen_ai.usage.input_tokens` | int | Input token count |
| `gen_ai.usage.output_tokens` | int | Output token count |
| `gen_ai.response.finish_reasons` | string[] | Why generation stopped |

### Setting Up OTel for RAG

```python
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import (
    BatchSpanProcessor,
    ConsoleSpanExporter,
)
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import (
    PeriodicExportingMetricReader,
    ConsoleMetricExporter,
)
from opentelemetry.sdk.resources import Resource
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import (
    OTLPSpanExporter,
)
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import (
    OTLPMetricExporter,
)


def setup_otel(
    service_name: str = "rag-service",
    otlp_endpoint: str = "http://localhost:4317",
) -> tuple[trace.Tracer, metrics.Meter]:
    """Configure OpenTelemetry with OTLP export for traces and metrics."""
    resource = Resource.create({
        "service.name": service_name,
        "service.version": "2.1.0",
        "deployment.environment": "production",
    })

    # Traces
    trace_provider = TracerProvider(resource=resource)
    trace_provider.add_span_processor(
        BatchSpanProcessor(
            OTLPSpanExporter(endpoint=otlp_endpoint)
        )
    )
    # Also export to console for debugging
    trace_provider.add_span_processor(
        BatchSpanProcessor(ConsoleSpanExporter())
    )
    trace.set_tracer_provider(trace_provider)

    # Metrics
    metric_reader = PeriodicExportingMetricReader(
        OTLPMetricExporter(endpoint=otlp_endpoint),
        export_interval_millis=30000,
    )
    meter_provider = MeterProvider(
        resource=resource,
        metric_readers=[metric_reader],
    )
    metrics.set_meter_provider(meter_provider)

    tracer = trace.get_tracer("rag-pipeline", "2.1.0")
    meter = metrics.get_meter("rag-pipeline", "2.1.0")

    return tracer, meter
```

---

## Custom RAG Spans

### Instrumenting Each Pipeline Stage

```python
from opentelemetry import trace
from opentelemetry.trace import StatusCode
import time


class OTelRAGPipeline:
    """RAG pipeline instrumented with OpenTelemetry custom spans.

    Each stage creates a child span with RAG-specific attributes
    following the GenAI semantic conventions where applicable.
    """

    def __init__(self, retriever, reranker, llm, embedder):
        self.retriever = retriever
        self.reranker = reranker
        self.llm = llm
        self.embedder = embedder
        self.tracer = trace.get_tracer("rag-pipeline")

    def query(self, user_query: str) -> dict:
        """Execute RAG query with full OTel tracing."""
        with self.tracer.start_as_current_span(
            "rag.query",
            attributes={
                "rag.query.original": user_query,
                "rag.pipeline.version": "v2.1",
            },
        ) as root_span:
            try:
                # Stage 1: Query rewriting
                rewritten = self._rewrite_query(user_query)
                root_span.set_attribute(
                    "rag.query.rewritten", rewritten
                )

                # Stage 2: Embedding
                query_vector = self._embed_query(rewritten)

                # Stage 3: Retrieval
                docs = self._retrieve(rewritten)

                # Stage 4: Reranking
                reranked = self._rerank(rewritten, docs)

                # Stage 5: Context assembly
                context, used_docs = self._assemble_context(reranked)

                # Stage 6: Generation
                answer = self._generate(user_query, context)

                root_span.set_attribute("rag.answer.length", len(answer))
                root_span.set_attribute(
                    "rag.sources.count", len(used_docs)
                )
                root_span.set_status(StatusCode.OK)

                return {
                    "answer": answer,
                    "sources": [d.metadata for d in used_docs],
                }

            except Exception as e:
                root_span.set_status(StatusCode.ERROR, str(e))
                root_span.record_exception(e)
                raise

    def _rewrite_query(self, query: str) -> str:
        """Rewrite query with OTel span."""
        with self.tracer.start_as_current_span(
            "rag.query_rewrite",
            attributes={
                "gen_ai.system": "anthropic",
                "gen_ai.request.model": "claude-haiku-4-20250514",
                "gen_ai.operation.name": "query_rewrite",
            },
        ) as span:
            response = self.llm.invoke(
                f"Rewrite as search query: {query}"
            )
            rewritten = response.content.strip()

            span.set_attribute("rag.rewrite.changed", query != rewritten)
            span.set_attribute(
                "gen_ai.usage.input_tokens",
                getattr(response, "usage", {}).get("input_tokens", 0),
            )
            span.set_attribute(
                "gen_ai.usage.output_tokens",
                getattr(response, "usage", {}).get("output_tokens", 0),
            )

            return rewritten

    def _embed_query(self, text: str) -> list[float]:
        """Embed query with OTel span."""
        with self.tracer.start_as_current_span(
            "rag.embed_query",
            attributes={
                "gen_ai.system": "openai",
                "gen_ai.request.model": "text-embedding-3-small",
                "gen_ai.operation.name": "embeddings",
                "rag.embedding.input_length": len(text),
            },
        ) as span:
            vector = self.embedder.embed_query(text)
            span.set_attribute(
                "rag.embedding.dimension", len(vector)
            )
            return vector

    def _retrieve(self, query: str) -> list:
        """Retrieve documents with OTel span."""
        with self.tracer.start_as_current_span(
            "rag.retrieval",
            attributes={
                "rag.retrieval.query": query,
                "rag.retrieval.top_k": 10,
                "rag.retrieval.vector_store": "chroma",
            },
        ) as span:
            docs = self.retriever.invoke(query)

            span.set_attribute(
                "rag.retrieval.num_results", len(docs)
            )
            # Log top scores
            scores = [
                d.metadata.get("score", 0.0) for d in docs[:5]
            ]
            span.set_attribute(
                "rag.retrieval.top_scores", str(scores)
            )
            span.set_attribute(
                "rag.retrieval.doc_ids",
                str([
                    d.metadata.get("source", "?") for d in docs[:5]
                ]),
            )

            return docs

    def _rerank(self, query: str, docs: list) -> list:
        """Rerank with OTel span."""
        with self.tracer.start_as_current_span(
            "rag.rerank",
            attributes={
                "rag.rerank.input_count": len(docs),
                "rag.rerank.model": "cross-encoder/ms-marco",
            },
        ) as span:
            reranked = self.reranker.rerank(query, docs)

            span.set_attribute(
                "rag.rerank.output_count", len(reranked)
            )
            # Log reranking score changes
            pre_ids = [
                d.metadata.get("source", "?") for d in docs[:5]
            ]
            post_ids = [
                d.metadata.get("source", "?") for d in reranked[:5]
            ]
            span.set_attribute(
                "rag.rerank.order_changed",
                pre_ids != post_ids,
            )

            return reranked

    def _assemble_context(self, docs: list) -> tuple[str, list]:
        """Assemble context with OTel span."""
        with self.tracer.start_as_current_span(
            "rag.context_assembly",
            attributes={
                "rag.context.input_docs": len(docs),
                "rag.context.max_docs": 5,
            },
        ) as span:
            used = docs[:5]
            context = "\n\n".join(d.page_content for d in used)
            estimated_tokens = len(context) // 4

            span.set_attribute(
                "rag.context.used_docs", len(used)
            )
            span.set_attribute(
                "rag.context.estimated_tokens", estimated_tokens
            )
            span.set_attribute(
                "rag.context.char_length", len(context)
            )

            return context, used

    def _generate(self, query: str, context: str) -> str:
        """Generate answer with OTel span following GenAI conventions."""
        with self.tracer.start_as_current_span(
            "rag.generation",
            attributes={
                "gen_ai.system": "anthropic",
                "gen_ai.request.model": "claude-sonnet-4-20250514",
                "gen_ai.request.max_tokens": 2048,
                "gen_ai.request.temperature": 0.0,
                "gen_ai.operation.name": "chat",
                "rag.generation.context_tokens": len(context) // 4,
            },
        ) as span:
            from anthropic import Anthropic
            client = Anthropic()
            response = client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=2048,
                messages=[{
                    "role": "user",
                    "content": (
                        f"Context:\n{context}\n\nQuestion: {query}"
                    ),
                }],
            )

            answer = response.content[0].text

            span.set_attribute(
                "gen_ai.usage.input_tokens",
                response.usage.input_tokens,
            )
            span.set_attribute(
                "gen_ai.usage.output_tokens",
                response.usage.output_tokens,
            )
            span.set_attribute(
                "gen_ai.response.finish_reasons",
                [response.stop_reason],
            )

            return answer
```

---

## OTel Metrics for RAG

### Defining RAG-Specific Metrics

```python
from opentelemetry import metrics


def create_rag_metrics(
    meter: metrics.Meter,
) -> dict:
    """Create OpenTelemetry metrics instruments for RAG monitoring.

    These metrics are exported to Prometheus/Grafana for
    dashboarding and alerting.
    """
    return {
        # Latency histograms
        "request_duration": meter.create_histogram(
            name="rag.request.duration",
            description="Total RAG request duration in milliseconds",
            unit="ms",
        ),
        "retrieval_duration": meter.create_histogram(
            name="rag.retrieval.duration",
            description="Vector search duration in milliseconds",
            unit="ms",
        ),
        "generation_duration": meter.create_histogram(
            name="rag.generation.duration",
            description="LLM generation duration in milliseconds",
            unit="ms",
        ),
        "rerank_duration": meter.create_histogram(
            name="rag.rerank.duration",
            description="Reranking duration in milliseconds",
            unit="ms",
        ),

        # Counters
        "request_count": meter.create_counter(
            name="rag.request.count",
            description="Total RAG requests",
        ),
        "error_count": meter.create_counter(
            name="rag.error.count",
            description="RAG request errors",
        ),
        "cache_hits": meter.create_counter(
            name="rag.cache.hits",
            description="Semantic cache hits",
        ),
        "cache_misses": meter.create_counter(
            name="rag.cache.misses",
            description="Semantic cache misses",
        ),

        # Token usage
        "input_tokens": meter.create_histogram(
            name="rag.tokens.input",
            description="Input tokens per request",
            unit="tokens",
        ),
        "output_tokens": meter.create_histogram(
            name="rag.tokens.output",
            description="Output tokens per request",
            unit="tokens",
        ),

        # Quality
        "faithfulness_score": meter.create_histogram(
            name="rag.quality.faithfulness",
            description="Faithfulness score per request",
        ),
        "retrieval_score": meter.create_histogram(
            name="rag.quality.retrieval_score",
            description="Top retrieval similarity score",
        ),

        # Gauges
        "active_requests": meter.create_up_down_counter(
            name="rag.requests.active",
            description="Currently processing RAG requests",
        ),
    }


class RAGMetricsRecorder:
    """Record metrics from RAG traces into OTel metrics."""

    def __init__(self, meter: metrics.Meter):
        self.metrics = create_rag_metrics(meter)

    def record_request(
        self,
        duration_ms: float,
        stage_durations: dict[str, float],
        input_tokens: int,
        output_tokens: int,
        cache_hit: bool,
        faithfulness_score: float | None = None,
        top_retrieval_score: float | None = None,
        error: bool = False,
        labels: dict | None = None,
    ) -> None:
        """Record all metrics for a single RAG request."""
        attrs = labels or {}

        self.metrics["request_count"].add(1, attrs)
        self.metrics["request_duration"].record(duration_ms, attrs)

        if error:
            self.metrics["error_count"].add(1, attrs)

        # Stage durations
        if "retrieval" in stage_durations:
            self.metrics["retrieval_duration"].record(
                stage_durations["retrieval"], attrs
            )
        if "generation" in stage_durations:
            self.metrics["generation_duration"].record(
                stage_durations["generation"], attrs
            )
        if "rerank" in stage_durations:
            self.metrics["rerank_duration"].record(
                stage_durations["rerank"], attrs
            )

        # Tokens
        self.metrics["input_tokens"].record(input_tokens, attrs)
        self.metrics["output_tokens"].record(output_tokens, attrs)

        # Cache
        if cache_hit:
            self.metrics["cache_hits"].add(1, attrs)
        else:
            self.metrics["cache_misses"].add(1, attrs)

        # Quality scores
        if faithfulness_score is not None:
            self.metrics["faithfulness_score"].record(
                faithfulness_score, attrs
            )
        if top_retrieval_score is not None:
            self.metrics["retrieval_score"].record(
                top_retrieval_score, attrs
            )
```

---

## Prometheus Alerting Rules

### Alert Definitions

```yaml
# prometheus/rag_alerts.yml
groups:
  - name: rag_alerts
    rules:
      # Latency alerts
      - alert: RAGHighLatencyP99
        expr: |
          histogram_quantile(0.99,
            rate(rag_request_duration_bucket[5m])
          ) > 5000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "RAG p99 latency exceeds 5 seconds"
          description: |
            The 99th percentile latency for RAG requests has exceeded
            5 seconds for more than 5 minutes.

      - alert: RAGRetrievalSlow
        expr: |
          histogram_quantile(0.95,
            rate(rag_retrieval_duration_bucket[5m])
          ) > 500
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Vector search p95 latency exceeds 500ms"

      # Error rate alerts
      - alert: RAGHighErrorRate
        expr: |
          rate(rag_error_count_total[5m])
          / rate(rag_request_count_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "RAG error rate exceeds 5%"

      # Quality alerts
      - alert: RAGFaithfulnessDrop
        expr: |
          histogram_quantile(0.50,
            rate(rag_quality_faithfulness_bucket[1h])
          ) < 0.7
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Median faithfulness score dropped below 0.7"

      # Cache alerts
      - alert: RAGCacheHitRateLow
        expr: |
          rate(rag_cache_hits_total[1h])
          / (rate(rag_cache_hits_total[1h]) + rate(rag_cache_misses_total[1h]))
          < 0.1
        for: 30m
        labels:
          severity: info
        annotations:
          summary: "Semantic cache hit rate below 10%"

      # Token usage alerts
      - alert: RAGHighTokenUsage
        expr: |
          sum(rate(rag_tokens_input_sum[1h])) > 1000000
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "RAG input token usage exceeds 1M tokens/hour"
```

### Implementing Alerting in Python

```python
import logging
from dataclasses import dataclass

logger = logging.getLogger(__name__)


@dataclass
class AlertRule:
    name: str
    condition_fn: callable
    severity: str
    message: str
    cooldown_seconds: int = 300


class RAGAlertManager:
    """Simple alert manager that evaluates rules against
    OTel metrics summaries and fires notifications."""

    def __init__(self, notification_fn=None):
        self.rules: list[AlertRule] = []
        self.last_fired: dict[str, float] = {}
        self.notify = notification_fn or self._default_notify

    def add_rule(self, rule: AlertRule) -> None:
        self.rules.append(rule)

    def evaluate(self, metrics_summary: dict) -> list[dict]:
        """Evaluate all rules against current metrics."""
        import time

        triggered = []
        now = time.time()

        for rule in self.rules:
            try:
                if rule.condition_fn(metrics_summary):
                    last = self.last_fired.get(rule.name, 0)
                    if now - last > rule.cooldown_seconds:
                        alert = {
                            "name": rule.name,
                            "severity": rule.severity,
                            "message": rule.message,
                            "timestamp": now,
                        }
                        triggered.append(alert)
                        self.last_fired[rule.name] = now
                        self.notify(alert)
            except Exception as e:
                logger.warning(
                    "Alert rule %s evaluation failed: %s",
                    rule.name,
                    str(e),
                )

        return triggered

    @staticmethod
    def _default_notify(alert: dict) -> None:
        logger.warning(
            "[%s] %s: %s",
            alert["severity"].upper(),
            alert["name"],
            alert["message"],
        )


# Configure alerts
alert_manager = RAGAlertManager()
alert_manager.add_rule(AlertRule(
    name="high_latency_p99",
    condition_fn=lambda m: m.get("latency_p99_ms", 0) > 5000,
    severity="warning",
    message="RAG p99 latency exceeds 5 seconds",
))
alert_manager.add_rule(AlertRule(
    name="high_error_rate",
    condition_fn=lambda m: m.get("error_rate", 0) > 0.05,
    severity="critical",
    message="RAG error rate exceeds 5%",
))
alert_manager.add_rule(AlertRule(
    name="faithfulness_drop",
    condition_fn=lambda m: (
        m.get("mean_faithfulness") is not None
        and m["mean_faithfulness"] < 0.7
    ),
    severity="warning",
    message="Mean faithfulness score below 0.7",
))
```

---

## Grafana Dashboard Configuration

### Key Panels for RAG Dashboard

```python
# Dashboard configuration as code (Grafana JSON model)
RAG_DASHBOARD_PANELS = {
    "row_1_latency": [
        {
            "title": "RAG Request Latency (p50/p95/p99)",
            "type": "timeseries",
            "queries": [
                'histogram_quantile(0.50, rate(rag_request_duration_bucket[5m]))',
                'histogram_quantile(0.95, rate(rag_request_duration_bucket[5m]))',
                'histogram_quantile(0.99, rate(rag_request_duration_bucket[5m]))',
            ],
        },
        {
            "title": "Latency by Stage",
            "type": "timeseries",
            "queries": [
                'histogram_quantile(0.95, rate(rag_retrieval_duration_bucket[5m]))',
                'histogram_quantile(0.95, rate(rag_generation_duration_bucket[5m]))',
                'histogram_quantile(0.95, rate(rag_rerank_duration_bucket[5m]))',
            ],
        },
    ],
    "row_2_throughput": [
        {
            "title": "Request Rate",
            "type": "stat",
            "query": 'rate(rag_request_count_total[5m])',
        },
        {
            "title": "Error Rate",
            "type": "gauge",
            "query": (
                'rate(rag_error_count_total[5m]) '
                '/ rate(rag_request_count_total[5m])'
            ),
            "thresholds": {"green": 0, "yellow": 0.01, "red": 0.05},
        },
    ],
    "row_3_quality": [
        {
            "title": "Faithfulness Score Distribution",
            "type": "histogram",
            "query": 'rag_quality_faithfulness_bucket',
        },
        {
            "title": "Cache Hit Rate",
            "type": "gauge",
            "query": (
                'rate(rag_cache_hits_total[1h]) '
                '/ (rate(rag_cache_hits_total[1h]) '
                '+ rate(rag_cache_misses_total[1h]))'
            ),
        },
    ],
    "row_4_tokens": [
        {
            "title": "Token Usage per Request",
            "type": "timeseries",
            "queries": [
                'rate(rag_tokens_input_sum[5m]) / rate(rag_tokens_input_count[5m])',
                'rate(rag_tokens_output_sum[5m]) / rate(rag_tokens_output_count[5m])',
            ],
        },
        {
            "title": "Estimated Cost per Hour",
            "type": "stat",
            "query": (
                '(rate(rag_tokens_input_sum[1h]) * 3 '
                '+ rate(rag_tokens_output_sum[1h]) * 15) / 1000000'
            ),
        },
    ],
}
```

---

## Common Pitfalls

1. **Not using semantic conventions.** Custom attribute names like `my_app.tokens` prevent cross-tool analysis. Use the GenAI semantic conventions (`gen_ai.usage.input_tokens`) so dashboards and alerts are portable.
2. **Forgetting to propagate trace context.** If your RAG pipeline calls external services (reranking API, embedding API), propagate the trace context via HTTP headers so child spans appear in the same trace.
3. **Not recording retrieval scores as span attributes.** Without scores, you cannot tell if retrieval quality degraded. Always log the similarity/relevance scores for the top-k results.
4. **Using synchronous span export in the request path.** Synchronous OTLP export adds latency to every request. Always use `BatchSpanProcessor` which exports asynchronously in the background.
5. **Not setting up metric-based alerts.** Traces are great for debugging individual requests, but metrics are what trigger alerts. Define Prometheus alert rules for latency, error rate, and quality thresholds.
6. **Ignoring the cardinality of metric labels.** Adding high-cardinality labels (like user_id or query text) to OTel metrics creates cardinality explosion in Prometheus. Use labels with bounded cardinality (model name, pipeline version, error type).

---

## References

- OpenTelemetry GenAI semantic conventions: https://opentelemetry.io/docs/specs/semconv/gen-ai/
- OpenTelemetry Python SDK: https://opentelemetry.io/docs/languages/python/
- Prometheus alerting rules: https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- Grafana dashboards: https://grafana.com/docs/grafana/latest/dashboards/
- OpenLLMetry (OTel for LLMs): https://github.com/traceloop/openllmetry
