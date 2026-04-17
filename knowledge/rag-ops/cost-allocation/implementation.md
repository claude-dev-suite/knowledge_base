# Cost Allocation -- Implementation: Middleware, Analytics Schema, and Grafana Dashboard

## Overview

This guide covers implementing a complete cost allocation system: middleware that intercepts LLM calls, BigQuery/ClickHouse schemas for storing cost data, and Grafana dashboards for visualization. The system provides per-tenant, per-feature, and per-model cost breakdowns.

---

## Middleware Architecture

```
Application Code
       |
       v
+------------------+
| Cost Middleware   | -> Intercepts all LLM calls
| (pre/post hooks) | -> Counts tokens, calculates cost
+--------+---------+ -> Emits structured logs
         |
    +----v----+
    | Log     | -> JSON log lines
    | Buffer  |
    +----+----+
         |
    +----v----+     +----------+
    | Ingest  | --> | BigQuery | --> Grafana Dashboard
    | Service |     | or       |
    +---------+     | ClickHouse|
                    +----------+
```

### Python Middleware

```python
import time
import json
import logging
import threading
from typing import Callable, Optional
from collections import deque
from dataclasses import dataclass, field, asdict
from datetime import datetime
import uuid

logger = logging.getLogger("cost_allocation")


class CostAllocationMiddleware:
    """Middleware that wraps LLM clients with cost tracking and async log flushing."""

    def __init__(
        self,
        log_sink: Callable[[list[dict]], None] = None,
        flush_interval: int = 10,
        flush_size: int = 100,
    ):
        self._buffer: deque = deque(maxlen=10000)
        self._lock = threading.Lock()
        self._sink = log_sink or self._default_sink
        self._flush_interval = flush_interval
        self._flush_size = flush_size

        # Start background flush thread
        self._flush_thread = threading.Thread(target=self._flush_loop, daemon=True)
        self._flush_thread.start()

    def record(self, log_entry: dict):
        """Add a log entry to the buffer."""
        with self._lock:
            self._buffer.append(log_entry)

        if len(self._buffer) >= self._flush_size:
            self._flush()

    def _flush(self):
        """Flush buffered logs to the sink."""
        with self._lock:
            if not self._buffer:
                return
            entries = list(self._buffer)
            self._buffer.clear()

        try:
            self._sink(entries)
        except Exception as e:
            logger.error(f"Failed to flush logs: {e}")
            # Re-add to buffer on failure
            with self._lock:
                for entry in entries:
                    self._buffer.append(entry)

    def _flush_loop(self):
        """Background thread that periodically flushes logs."""
        while True:
            time.sleep(self._flush_interval)
            self._flush()

    @staticmethod
    def _default_sink(entries: list[dict]):
        """Default sink: write to stdout as JSON lines."""
        for entry in entries:
            print(json.dumps(entry))

    def wrap_openai(self, client, tenant_id: str, project_id: str = ""):
        """Return a wrapped OpenAI client that tracks costs."""
        return TrackedOpenAIClient(
            client=client,
            middleware=self,
            tenant_id=tenant_id,
            project_id=project_id,
        )


class TrackedOpenAIClient:
    """OpenAI client with automatic cost tracking."""

    PRICING = {
        "gpt-4o-mini": {"input": 0.15, "output": 0.60},
        "gpt-4o": {"input": 2.50, "output": 10.00},
        "text-embedding-3-small": {"input": 0.02, "output": 0.0},
        "text-embedding-3-large": {"input": 0.13, "output": 0.0},
    }

    def __init__(self, client, middleware, tenant_id, project_id):
        self._client = client
        self._middleware = middleware
        self._tenant_id = tenant_id
        self._project_id = project_id

    def embeddings_create(self, input, model="text-embedding-3-small", **kwargs):
        start = time.perf_counter()
        response = self._client.embeddings.create(input=input, model=model, **kwargs)
        latency = (time.perf_counter() - start) * 1000

        usage = response.usage
        costs = self._calculate_cost(model, usage.total_tokens, 0)

        self._middleware.record({
            "call_id": str(uuid.uuid4()),
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "tenant_id": self._tenant_id,
            "project_id": self._project_id,
            "provider": "openai",
            "model": model,
            "endpoint": "embeddings",
            "input_tokens": usage.total_tokens,
            "output_tokens": 0,
            "total_tokens": usage.total_tokens,
            "latency_ms": round(latency, 2),
            "status": "success",
            **costs,
        })

        return response

    def chat_completions_create(self, messages, model="gpt-4o-mini", **kwargs):
        start = time.perf_counter()
        response = self._client.chat.completions.create(
            messages=messages, model=model, **kwargs
        )
        latency = (time.perf_counter() - start) * 1000

        usage = response.usage
        costs = self._calculate_cost(model, usage.prompt_tokens, usage.completion_tokens)

        self._middleware.record({
            "call_id": str(uuid.uuid4()),
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "tenant_id": self._tenant_id,
            "project_id": self._project_id,
            "provider": "openai",
            "model": model,
            "endpoint": "chat_completions",
            "input_tokens": usage.prompt_tokens,
            "output_tokens": usage.completion_tokens,
            "total_tokens": usage.total_tokens,
            "latency_ms": round(latency, 2),
            "status": "success",
            **costs,
        })

        return response

    def _calculate_cost(self, model, input_tokens, output_tokens):
        p = self.PRICING.get(model, {"input": 0, "output": 0})
        input_cost = (input_tokens / 1_000_000) * p["input"]
        output_cost = (output_tokens / 1_000_000) * p["output"]
        return {
            "input_cost_usd": round(input_cost, 8),
            "output_cost_usd": round(output_cost, 8),
            "total_cost_usd": round(input_cost + output_cost, 8),
        }
```

---

## BigQuery Schema

```sql
-- BigQuery table for LLM cost tracking
CREATE TABLE IF NOT EXISTS `project.dataset.llm_costs` (
  call_id STRING NOT NULL,
  timestamp TIMESTAMP NOT NULL,

  -- Attribution
  tenant_id STRING NOT NULL,
  user_id STRING,
  project_id STRING,
  feature STRING,          -- search, generation, evaluation, embedding
  pipeline_step STRING,    -- query_embed, rerank, generate, judge

  -- Request details
  provider STRING NOT NULL,  -- openai, anthropic, voyage
  model STRING NOT NULL,
  endpoint STRING,

  -- Token usage
  input_tokens INT64 NOT NULL DEFAULT 0,
  output_tokens INT64 NOT NULL DEFAULT 0,
  total_tokens INT64 NOT NULL DEFAULT 0,

  -- Cost (USD)
  input_cost_usd FLOAT64 NOT NULL DEFAULT 0.0,
  output_cost_usd FLOAT64 NOT NULL DEFAULT 0.0,
  total_cost_usd FLOAT64 NOT NULL DEFAULT 0.0,

  -- Performance
  latency_ms FLOAT64,
  status STRING DEFAULT 'success',
  error_message STRING,

  -- Metadata
  request_metadata JSON
)
PARTITION BY DATE(timestamp)
CLUSTER BY tenant_id, model;
```

### BigQuery Sink

```python
from google.cloud import bigquery
import json


class BigQuerySink:
    """Flush cost logs to BigQuery."""

    def __init__(self, table_id: str = "project.dataset.llm_costs"):
        self.client = bigquery.Client()
        self.table_id = table_id

    def __call__(self, entries: list[dict]):
        """Insert log entries into BigQuery."""
        if not entries:
            return

        errors = self.client.insert_rows_json(self.table_id, entries)
        if errors:
            raise RuntimeError(f"BigQuery insert errors: {errors}")


# Usage
bq_sink = BigQuerySink("my-project.llm_costs.cost_logs")
middleware = CostAllocationMiddleware(log_sink=bq_sink)
```

---

## ClickHouse Schema

```sql
-- ClickHouse table (better for real-time analytics)
CREATE TABLE llm_costs (
    call_id UUID DEFAULT generateUUIDv4(),
    timestamp DateTime64(3) DEFAULT now64(3),

    -- Attribution
    tenant_id LowCardinality(String),
    user_id String DEFAULT '',
    project_id LowCardinality(String) DEFAULT '',
    feature LowCardinality(String) DEFAULT '',
    pipeline_step LowCardinality(String) DEFAULT '',

    -- Request
    provider LowCardinality(String),
    model LowCardinality(String),
    endpoint LowCardinality(String) DEFAULT '',

    -- Tokens
    input_tokens UInt32 DEFAULT 0,
    output_tokens UInt32 DEFAULT 0,
    total_tokens UInt32 DEFAULT 0,

    -- Cost
    input_cost_usd Float64 DEFAULT 0.0,
    output_cost_usd Float64 DEFAULT 0.0,
    total_cost_usd Float64 DEFAULT 0.0,

    -- Performance
    latency_ms Float32 DEFAULT 0.0,
    status LowCardinality(String) DEFAULT 'success',
    error_message String DEFAULT ''
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (tenant_id, timestamp)
TTL timestamp + INTERVAL 90 DAY;
```

---

## Analytics Queries

### Cost by Tenant (Monthly)

```sql
-- BigQuery
SELECT
    tenant_id,
    SUM(total_cost_usd) AS total_cost,
    SUM(total_tokens) AS total_tokens,
    COUNT(*) AS total_calls,
    AVG(latency_ms) AS avg_latency_ms
FROM `project.dataset.llm_costs`
WHERE timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY tenant_id
ORDER BY total_cost DESC;
```

### Cost by Feature and Model

```sql
SELECT
    feature,
    model,
    SUM(total_cost_usd) AS cost,
    SUM(input_tokens) AS input_tokens,
    SUM(output_tokens) AS output_tokens,
    COUNT(*) AS calls
FROM `project.dataset.llm_costs`
WHERE timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY feature, model
ORDER BY cost DESC;
```

### Daily Cost Trend

```sql
SELECT
    DATE(timestamp) AS date,
    SUM(total_cost_usd) AS daily_cost,
    SUM(total_tokens) AS daily_tokens,
    COUNT(*) AS daily_calls
FROM `project.dataset.llm_costs`
WHERE timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY date
ORDER BY date;
```

### Top Cost Drivers (Per-Request Analysis)

```sql
SELECT
    tenant_id,
    feature,
    model,
    total_cost_usd,
    total_tokens,
    latency_ms,
    timestamp
FROM `project.dataset.llm_costs`
WHERE timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
ORDER BY total_cost_usd DESC
LIMIT 100;
```

---

## Grafana Dashboard Configuration

### Panel 1: Total Cost Over Time

```json
{
  "title": "Daily LLM Cost",
  "type": "timeseries",
  "targets": [{
    "rawSql": "SELECT DATE(timestamp) as time, SUM(total_cost_usd) as cost FROM llm_costs WHERE $__timeFilter(timestamp) GROUP BY time ORDER BY time",
    "format": "time_series"
  }]
}
```

### Panel 2: Cost by Tenant (Pie Chart)

```json
{
  "title": "Cost by Tenant (Last 30 Days)",
  "type": "piechart",
  "targets": [{
    "rawSql": "SELECT tenant_id, SUM(total_cost_usd) as cost FROM llm_costs WHERE timestamp > now() - INTERVAL 30 DAY GROUP BY tenant_id ORDER BY cost DESC LIMIT 10"
  }]
}
```

### Panel 3: Token Usage by Model

```json
{
  "title": "Token Usage by Model",
  "type": "barchart",
  "targets": [{
    "rawSql": "SELECT model, SUM(input_tokens) as input_tokens, SUM(output_tokens) as output_tokens FROM llm_costs WHERE $__timeFilter(timestamp) GROUP BY model ORDER BY input_tokens + output_tokens DESC"
  }]
}
```

---

## Common Pitfalls

1. **Not partitioning by time**: Cost tables grow fast. Without time partitioning, queries over 30 days scan the entire table. Always partition by date.

2. **Logging synchronously in the hot path**: Sending logs to BigQuery synchronously adds 50-200ms per API call. Always buffer and flush asynchronously.

3. **Missing tenant_id on some calls**: If any API call lacks a tenant_id, those costs become unattributable. Make tenant_id required at the middleware level.

4. **Not accounting for failed requests**: Failed API calls still consume tokens (for the request parsing). Log failures with their token counts and mark status as "error".

5. **Stale pricing data**: LLM pricing changes. If your pricing table has outdated rates, cost calculations drift. Version your pricing data and backfill when prices change.

6. **Over-relying on provider-reported tokens**: Provider token counts may arrive late (streaming) or differ slightly from tiktoken estimates. Use tiktoken for pre-call estimation and provider counts for post-call accuracy.

---

## References

- BigQuery documentation: https://cloud.google.com/bigquery/docs
- ClickHouse documentation: https://clickhouse.com/docs
- Grafana documentation: https://grafana.com/docs/
- tiktoken: https://github.com/openai/tiktoken
