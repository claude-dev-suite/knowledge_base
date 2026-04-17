# LLM Gateway -- Monitoring: Cost Dashboards, Latency Tracking, and Model Comparison

## Overview

This guide covers monitoring an LLM gateway in production: building cost dashboards, tracking latency across providers, comparing model performance, setting up alerts, and implementing A/B testing for model selection. Effective monitoring turns an LLM gateway from a routing layer into a decision-making tool.

---

## Key Metrics to Track

### Cost Metrics

| Metric | Description | Aggregation |
|--------|-------------|-------------|
| Total cost | Sum of all API costs | Daily, weekly, monthly |
| Cost per request | Average cost per API call | By model, by feature |
| Cost per tenant | Spending by customer/team | Monthly, with trending |
| Cost by model | Spending per model | Daily for budget planning |
| Cache savings | Cost avoided through caching | Daily |

### Latency Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|----------------|
| P50 latency | Median response time | Varies by model |
| P95 latency | 95th percentile response time | > 2x P50 |
| P99 latency | 99th percentile response time | > 5x P50 |
| Time to first token (TTFT) | For streaming responses | > 2 seconds |
| Tokens per second (TPS) | Generation speed | < 20 TPS |

### Reliability Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|----------------|
| Error rate | % of failed requests | > 1% |
| Fallback rate | % using non-primary provider | > 5% |
| Timeout rate | % of requests timing out | > 0.5% |
| Rate limit hits | 429 responses from providers | > 10/minute |
| Cache hit rate | % of requests served from cache | Monitor trending |

---

## Structured Logging for Monitoring

```python
import json
import time
import logging
from dataclasses import dataclass, asdict
from datetime import datetime
from typing import Optional

logger = logging.getLogger("gateway_monitor")


@dataclass
class GatewayMetric:
    """Structured metric for gateway monitoring."""
    timestamp: str
    request_id: str

    # Routing
    requested_model: str
    actual_model: str
    provider: str
    was_fallback: bool
    fallback_chain: list[str]

    # Performance
    total_latency_ms: float
    ttft_ms: Optional[float]      # time to first token (streaming)
    tokens_per_second: Optional[float]

    # Tokens and cost
    input_tokens: int
    output_tokens: int
    total_cost_usd: float

    # Cache
    cache_hit: bool
    cache_similarity: Optional[float]

    # Status
    status: str                    # success, error, timeout, rate_limited
    error_code: Optional[str]
    retry_count: int

    # Attribution
    tenant_id: str
    feature: str

    def to_json_line(self) -> str:
        return json.dumps(asdict(self))


class GatewayMonitor:
    """Collect and emit gateway metrics."""

    def __init__(self, emit_fn=None):
        self._emit = emit_fn or (lambda m: logger.info(m.to_json_line()))
        self._metrics_buffer: list[GatewayMetric] = []

    def record(self, metric: GatewayMetric):
        """Record a metric."""
        self._emit(metric)
        self._metrics_buffer.append(metric)

    def get_summary(self, last_n_minutes: int = 60) -> dict:
        """Get summary statistics for recent metrics."""
        cutoff = time.time() - (last_n_minutes * 60)
        recent = [
            m for m in self._metrics_buffer
            if datetime.fromisoformat(m.timestamp.rstrip("Z")).timestamp() > cutoff
        ]

        if not recent:
            return {"total_requests": 0}

        latencies = [m.total_latency_ms for m in recent if m.status == "success"]
        costs = [m.total_cost_usd for m in recent]

        return {
            "total_requests": len(recent),
            "success_rate": sum(1 for m in recent if m.status == "success") / len(recent),
            "error_rate": sum(1 for m in recent if m.status == "error") / len(recent),
            "fallback_rate": sum(1 for m in recent if m.was_fallback) / len(recent),
            "cache_hit_rate": sum(1 for m in recent if m.cache_hit) / len(recent),
            "total_cost": sum(costs),
            "avg_cost_per_request": sum(costs) / len(costs),
            "p50_latency_ms": sorted(latencies)[len(latencies) // 2] if latencies else 0,
            "p95_latency_ms": sorted(latencies)[int(len(latencies) * 0.95)] if latencies else 0,
            "p99_latency_ms": sorted(latencies)[int(len(latencies) * 0.99)] if latencies else 0,
            "by_model": self._group_by(recent, "actual_model"),
            "by_tenant": self._group_by(recent, "tenant_id"),
        }

    def _group_by(self, metrics: list, field: str) -> dict:
        groups = {}
        for m in metrics:
            key = getattr(m, field)
            if key not in groups:
                groups[key] = {"count": 0, "cost": 0.0, "errors": 0}
            groups[key]["count"] += 1
            groups[key]["cost"] += m.total_cost_usd
            if m.status != "success":
                groups[key]["errors"] += 1
        return groups
```

---

## Prometheus Metrics

```python
from prometheus_client import Counter, Histogram, Gauge, start_http_server

# Request counters
gateway_requests_total = Counter(
    "gateway_requests_total",
    "Total gateway requests",
    ["model", "provider", "status", "tenant_id", "feature"],
)

gateway_fallbacks_total = Counter(
    "gateway_fallbacks_total",
    "Total fallback events",
    ["requested_model", "actual_model"],
)

gateway_cache_hits_total = Counter(
    "gateway_cache_hits_total",
    "Total cache hits",
    ["cache_type"],  # "exact", "semantic"
)

# Latency histograms
gateway_latency_seconds = Histogram(
    "gateway_latency_seconds",
    "Request latency in seconds",
    ["model", "provider"],
    buckets=[0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0, 30.0, 60.0],
)

gateway_ttft_seconds = Histogram(
    "gateway_ttft_seconds",
    "Time to first token in seconds",
    ["model"],
    buckets=[0.05, 0.1, 0.25, 0.5, 1.0, 2.0, 5.0],
)

# Cost tracking
gateway_cost_usd = Counter(
    "gateway_cost_usd_total",
    "Total cost in USD",
    ["model", "provider", "tenant_id"],
)

gateway_tokens_total = Counter(
    "gateway_tokens_total",
    "Total tokens processed",
    ["model", "direction"],  # direction: "input" or "output"
)

# Gauges
gateway_cache_hit_rate = Gauge(
    "gateway_cache_hit_rate",
    "Cache hit rate (rolling 5 minutes)",
)


def record_prometheus_metrics(metric: GatewayMetric):
    """Record a gateway metric to Prometheus."""
    # Request counter
    gateway_requests_total.labels(
        model=metric.actual_model,
        provider=metric.provider,
        status=metric.status,
        tenant_id=metric.tenant_id,
        feature=metric.feature,
    ).inc()

    # Fallback counter
    if metric.was_fallback:
        gateway_fallbacks_total.labels(
            requested_model=metric.requested_model,
            actual_model=metric.actual_model,
        ).inc()

    # Cache counter
    if metric.cache_hit:
        gateway_cache_hits_total.labels(cache_type="semantic").inc()

    # Latency
    gateway_latency_seconds.labels(
        model=metric.actual_model,
        provider=metric.provider,
    ).observe(metric.total_latency_ms / 1000.0)

    if metric.ttft_ms:
        gateway_ttft_seconds.labels(model=metric.actual_model).observe(
            metric.ttft_ms / 1000.0
        )

    # Cost
    gateway_cost_usd.labels(
        model=metric.actual_model,
        provider=metric.provider,
        tenant_id=metric.tenant_id,
    ).inc(metric.total_cost_usd)

    # Tokens
    gateway_tokens_total.labels(
        model=metric.actual_model, direction="input"
    ).inc(metric.input_tokens)
    gateway_tokens_total.labels(
        model=metric.actual_model, direction="output"
    ).inc(metric.output_tokens)


# Start Prometheus metrics server
# start_http_server(9090)
```

---

## Grafana Dashboard Panels

### Panel: Cost Over Time by Model

```json
{
  "title": "Daily Cost by Model",
  "type": "timeseries",
  "targets": [{
    "expr": "sum(rate(gateway_cost_usd_total[24h])) by (model) * 86400",
    "legendFormat": "{{ model }}"
  }]
}
```

### Panel: Latency Distribution

```json
{
  "title": "Request Latency P50/P95/P99",
  "type": "timeseries",
  "targets": [
    {
      "expr": "histogram_quantile(0.50, rate(gateway_latency_seconds_bucket[5m]))",
      "legendFormat": "P50"
    },
    {
      "expr": "histogram_quantile(0.95, rate(gateway_latency_seconds_bucket[5m]))",
      "legendFormat": "P95"
    },
    {
      "expr": "histogram_quantile(0.99, rate(gateway_latency_seconds_bucket[5m]))",
      "legendFormat": "P99"
    }
  ]
}
```

### Panel: Error and Fallback Rates

```json
{
  "title": "Error & Fallback Rate",
  "type": "timeseries",
  "targets": [
    {
      "expr": "sum(rate(gateway_requests_total{status='error'}[5m])) / sum(rate(gateway_requests_total[5m])) * 100",
      "legendFormat": "Error Rate %"
    },
    {
      "expr": "sum(rate(gateway_fallbacks_total[5m])) / sum(rate(gateway_requests_total[5m])) * 100",
      "legendFormat": "Fallback Rate %"
    }
  ]
}
```

### Panel: Cache Effectiveness

```json
{
  "title": "Cache Hit Rate & Savings",
  "type": "stat",
  "targets": [{
    "expr": "sum(rate(gateway_cache_hits_total[1h])) / sum(rate(gateway_requests_total[1h])) * 100",
    "legendFormat": "Cache Hit Rate %"
  }]
}
```

---

## Model Comparison and A/B Testing

### Compare Models Side by Side

```python
import asyncio
import time
import openai


async def compare_models(
    gateway_url: str,
    gateway_key: str,
    models: list[str],
    test_prompts: list[dict],
) -> dict:
    """Run the same prompts through multiple models and compare."""
    client = openai.AsyncOpenAI(base_url=gateway_url, api_key=gateway_key)
    results = {model: {"latencies": [], "costs": [], "responses": []} for model in models}

    for prompt in test_prompts:
        for model in models:
            start = time.perf_counter()
            try:
                response = await client.chat.completions.create(
                    model=model,
                    messages=prompt["messages"],
                    temperature=0.0,
                    max_tokens=500,
                )
                latency = (time.perf_counter() - start) * 1000
                usage = response.usage

                results[model]["latencies"].append(latency)
                results[model]["costs"].append(
                    usage.prompt_tokens * 0.00015 + usage.completion_tokens * 0.0006
                )
                results[model]["responses"].append(
                    response.choices[0].message.content
                )
            except Exception as e:
                results[model]["latencies"].append(None)
                results[model]["costs"].append(None)
                results[model]["responses"].append(f"ERROR: {e}")

    # Summary
    summary = {}
    for model, data in results.items():
        valid_latencies = [l for l in data["latencies"] if l is not None]
        valid_costs = [c for c in data["costs"] if c is not None]

        summary[model] = {
            "avg_latency_ms": sum(valid_latencies) / len(valid_latencies) if valid_latencies else 0,
            "p95_latency_ms": sorted(valid_latencies)[int(len(valid_latencies) * 0.95)] if valid_latencies else 0,
            "total_cost": sum(valid_costs),
            "error_rate": sum(1 for l in data["latencies"] if l is None) / len(data["latencies"]),
        }

    return summary
```

---

## Alerting Rules

### Prometheus Alert Rules

```yaml
# alerts.yml
groups:
  - name: llm_gateway
    rules:
      - alert: HighErrorRate
        expr: sum(rate(gateway_requests_total{status="error"}[5m])) / sum(rate(gateway_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "LLM Gateway error rate > 5%"

      - alert: HighFallbackRate
        expr: sum(rate(gateway_fallbacks_total[5m])) / sum(rate(gateway_requests_total[5m])) > 0.1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "LLM Gateway fallback rate > 10% (primary provider may be degraded)"

      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(gateway_latency_seconds_bucket[5m])) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "LLM Gateway P95 latency > 10 seconds"

      - alert: BudgetAlert
        expr: sum(gateway_cost_usd_total) > 400
        labels:
          severity: warning
        annotations:
          summary: "Monthly LLM spending approaching $500 budget"

      - alert: CacheHitRateDrop
        expr: sum(rate(gateway_cache_hits_total[1h])) / sum(rate(gateway_requests_total[1h])) < 0.1
        for: 30m
        labels:
          severity: info
        annotations:
          summary: "Cache hit rate dropped below 10% (new query patterns or cache issue)"
```

---

## Common Pitfalls

1. **Monitoring only aggregate latency**: P50 looks fine but P99 is terrible. Always track P50, P95, and P99 separately. A P99 spike often indicates a provider issue that affects 1% of users.

2. **Not tracking cost per feature**: Knowing total spend is useful, but knowing that "document summarization" costs 60% of the budget while "search" costs 5% drives optimization decisions.

3. **Alert fatigue**: Too many alerts cause teams to ignore them. Start with 3-5 critical alerts and add more based on actual incidents, not hypothetical scenarios.

4. **Not correlating gateway metrics with application metrics**: A spike in gateway latency may not affect user experience if the gateway is used for background tasks. Correlate gateway metrics with user-facing latency.

5. **Missing tenant attribution on cached responses**: If a cache hit does not record the tenant_id, cost attribution becomes incorrect. Always attribute cached responses to the requesting tenant.

6. **Not comparing model quality alongside cost**: A model that is 50% cheaper but produces 20% worse answers may actually cost more in user frustration and rework. Track quality metrics (user ratings, follow-up question rate) alongside cost.

---

## References

- Prometheus documentation: https://prometheus.io/docs/
- Grafana dashboard: https://grafana.com/docs/
- LiteLLM monitoring: https://docs.litellm.ai/docs/proxy/logging
- Portkey analytics: https://portkey.ai/docs/product/observability
