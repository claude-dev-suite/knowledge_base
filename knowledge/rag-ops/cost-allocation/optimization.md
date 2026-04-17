# Cost Allocation -- Optimization: Budget Alerts, Per-Tenant Caps, and Model Routing

## Overview

Once you have cost visibility (see `overview.md` and `implementation.md`), the next step is cost control: setting budget alerts before overspending, enforcing per-tenant cost caps, and routing requests to the most cost-effective model for each use case. This guide covers implementing these controls in a production RAG system.

---

## Budget Alerts

### Alert Thresholds

```python
from dataclasses import dataclass
from typing import Optional
import json
import logging

logger = logging.getLogger("cost_alerts")


@dataclass
class BudgetAlert:
    tenant_id: str
    alert_type: str        # "warning", "critical", "limit_reached"
    current_spend: float
    budget_limit: float
    period: str            # "daily", "weekly", "monthly"
    message: str


class BudgetAlertService:
    """Monitor spending and trigger alerts when thresholds are crossed."""

    def __init__(self, query_fn, notify_fn):
        """
        query_fn: callable that returns current spend for a tenant/period
        notify_fn: callable that sends alerts (email, Slack, PagerDuty)
        """
        self.query_fn = query_fn
        self.notify_fn = notify_fn

    def check_budgets(self, budgets: list[dict]) -> list[BudgetAlert]:
        """
        Check all tenant budgets and trigger alerts.

        budgets: [{"tenant_id": "acme", "monthly_limit": 500, "daily_limit": 25}]
        """
        alerts = []

        for budget in budgets:
            tenant = budget["tenant_id"]

            # Check monthly budget
            if "monthly_limit" in budget:
                monthly_spend = self.query_fn(tenant, "monthly")
                monthly_limit = budget["monthly_limit"]

                if monthly_spend >= monthly_limit:
                    alerts.append(BudgetAlert(
                        tenant_id=tenant,
                        alert_type="limit_reached",
                        current_spend=monthly_spend,
                        budget_limit=monthly_limit,
                        period="monthly",
                        message=f"Tenant {tenant} reached monthly budget: "
                                f"${monthly_spend:.2f} / ${monthly_limit:.2f}",
                    ))
                elif monthly_spend >= monthly_limit * 0.9:
                    alerts.append(BudgetAlert(
                        tenant_id=tenant,
                        alert_type="critical",
                        current_spend=monthly_spend,
                        budget_limit=monthly_limit,
                        period="monthly",
                        message=f"Tenant {tenant} at 90% of monthly budget: "
                                f"${monthly_spend:.2f} / ${monthly_limit:.2f}",
                    ))
                elif monthly_spend >= monthly_limit * 0.7:
                    alerts.append(BudgetAlert(
                        tenant_id=tenant,
                        alert_type="warning",
                        current_spend=monthly_spend,
                        budget_limit=monthly_limit,
                        period="monthly",
                        message=f"Tenant {tenant} at 70% of monthly budget: "
                                f"${monthly_spend:.2f} / ${monthly_limit:.2f}",
                    ))

            # Check daily budget (spike detection)
            if "daily_limit" in budget:
                daily_spend = self.query_fn(tenant, "daily")
                daily_limit = budget["daily_limit"]

                if daily_spend >= daily_limit:
                    alerts.append(BudgetAlert(
                        tenant_id=tenant,
                        alert_type="critical",
                        current_spend=daily_spend,
                        budget_limit=daily_limit,
                        period="daily",
                        message=f"Tenant {tenant} exceeded daily budget: "
                                f"${daily_spend:.2f} / ${daily_limit:.2f}",
                    ))

        # Send notifications
        for alert in alerts:
            self.notify_fn(alert)
            logger.warning(json.dumps({
                "event": "budget_alert",
                "tenant_id": alert.tenant_id,
                "alert_type": alert.alert_type,
                "current_spend": alert.current_spend,
                "budget_limit": alert.budget_limit,
                "period": alert.period,
            }))

        return alerts
```

### Slack Notification

```python
import httpx


def send_slack_alert(alert: BudgetAlert):
    """Send a budget alert to Slack."""
    color = {
        "warning": "#FFA500",
        "critical": "#FF0000",
        "limit_reached": "#8B0000",
    }.get(alert.alert_type, "#808080")

    payload = {
        "attachments": [{
            "color": color,
            "title": f"LLM Budget Alert: {alert.alert_type.upper()}",
            "text": alert.message,
            "fields": [
                {"title": "Tenant", "value": alert.tenant_id, "short": True},
                {"title": "Period", "value": alert.period, "short": True},
                {"title": "Current Spend", "value": f"${alert.current_spend:.2f}", "short": True},
                {"title": "Budget Limit", "value": f"${alert.budget_limit:.2f}", "short": True},
            ],
        }],
    }

    httpx.post(
        "https://hooks.slack.com/services/YOUR/WEBHOOK/URL",
        json=payload,
    )
```

---

## Per-Tenant Cost Caps

### Rate Limiting by Cost

```python
import time
import threading
from collections import defaultdict


class TenantCostLimiter:
    """Enforce per-tenant cost limits with automatic throttling."""

    def __init__(self, default_daily_limit: float = 50.0):
        self._daily_spend: dict[str, float] = defaultdict(float)
        self._daily_limit: dict[str, float] = {}
        self._default_daily_limit = default_daily_limit
        self._lock = threading.Lock()
        self._last_reset = time.time()

    def set_limit(self, tenant_id: str, daily_limit: float):
        """Set daily cost limit for a tenant."""
        self._daily_limit[tenant_id] = daily_limit

    def check_budget(self, tenant_id: str, estimated_cost: float) -> dict:
        """
        Check if a request can proceed given the tenant's budget.
        Returns: {"allowed": bool, "remaining": float, "reason": str}
        """
        self._maybe_reset()

        with self._lock:
            limit = self._daily_limit.get(tenant_id, self._default_daily_limit)
            current = self._daily_spend[tenant_id]
            remaining = limit - current

            if estimated_cost > remaining:
                return {
                    "allowed": False,
                    "remaining": remaining,
                    "reason": f"Daily budget exceeded: ${current:.4f} / ${limit:.2f}. "
                              f"Estimated cost ${estimated_cost:.4f} exceeds remaining ${remaining:.4f}",
                }

            return {
                "allowed": True,
                "remaining": remaining - estimated_cost,
                "reason": "ok",
            }

    def record_cost(self, tenant_id: str, actual_cost: float):
        """Record actual cost after a successful API call."""
        with self._lock:
            self._daily_spend[tenant_id] += actual_cost

    def get_spend(self, tenant_id: str) -> float:
        """Get current daily spend for a tenant."""
        return self._daily_spend.get(tenant_id, 0.0)

    def _maybe_reset(self):
        """Reset daily counters at midnight UTC."""
        now = time.time()
        # Reset every 24 hours
        if now - self._last_reset > 86400:
            with self._lock:
                self._daily_spend.clear()
                self._last_reset = now


# Usage in a RAG pipeline
limiter = TenantCostLimiter(default_daily_limit=25.0)
limiter.set_limit("enterprise-tenant", 200.0)
limiter.set_limit("free-tier-tenant", 5.0)


def rag_query_with_limits(tenant_id: str, question: str) -> dict:
    """RAG query with cost cap enforcement."""
    # Estimate cost before making the call
    estimated_cost = 0.001  # ~$0.001 per RAG query with gpt-4o-mini

    check = limiter.check_budget(tenant_id, estimated_cost)
    if not check["allowed"]:
        return {
            "error": "budget_exceeded",
            "message": check["reason"],
            "remaining_budget": check["remaining"],
        }

    # Proceed with RAG query
    # ... (embedding, retrieval, generation)

    actual_cost = 0.00085  # from API response usage
    limiter.record_cost(tenant_id, actual_cost)

    return {
        "answer": "...",
        "cost": actual_cost,
        "remaining_budget": check["remaining"] - actual_cost,
    }
```

### HTTP 429 Response for Budget Exceeded

```python
from fastapi import FastAPI, HTTPException, Depends, Header

app = FastAPI()
limiter = TenantCostLimiter()


async def check_tenant_budget(
    x_tenant_id: str = Header(...),
    estimated_cost: float = 0.001,
):
    check = limiter.check_budget(x_tenant_id, estimated_cost)
    if not check["allowed"]:
        raise HTTPException(
            status_code=429,
            detail={
                "error": "budget_exceeded",
                "message": check["reason"],
                "remaining_budget": check["remaining"],
                "retry_after": "tomorrow",
            },
            headers={"Retry-After": "86400"},
        )
    return x_tenant_id


@app.post("/query")
async def query(
    question: str,
    tenant_id: str = Depends(check_tenant_budget),
):
    # Process query...
    return {"answer": "..."}
```

---

## Cost-Based Model Routing

### Route Requests to the Cheapest Viable Model

```python
from dataclasses import dataclass


@dataclass
class ModelConfig:
    name: str
    provider: str
    input_cost_per_1m: float
    output_cost_per_1m: float
    quality_tier: str          # "high", "medium", "low"
    max_context_tokens: int
    supports_tools: bool = True


# Available models ranked by cost
MODELS = [
    ModelConfig("gpt-4o-mini", "openai", 0.15, 0.60, "medium", 128000),
    ModelConfig("claude-haiku-4-20250514", "anthropic", 0.80, 4.00, "medium", 200000),
    ModelConfig("gpt-4o", "openai", 2.50, 10.00, "high", 128000),
    ModelConfig("claude-sonnet-4-20250514", "anthropic", 3.00, 15.00, "high", 200000),
]


class CostBasedRouter:
    """Route requests to the most cost-effective model based on requirements."""

    def __init__(self, models: list[ModelConfig] = None):
        self.models = models or MODELS

    def select_model(
        self,
        quality_required: str = "medium",   # "low", "medium", "high"
        context_tokens: int = 4000,
        prefer_provider: str = None,
        max_cost_per_1m_input: float = None,
    ) -> ModelConfig:
        """Select the cheapest model that meets requirements."""
        candidates = []

        for model in self.models:
            # Filter by quality tier
            tier_order = {"low": 0, "medium": 1, "high": 2}
            if tier_order.get(model.quality_tier, 0) < tier_order.get(quality_required, 0):
                continue

            # Filter by context window
            if context_tokens > model.max_context_tokens:
                continue

            # Filter by cost limit
            if max_cost_per_1m_input and model.input_cost_per_1m > max_cost_per_1m_input:
                continue

            # Filter by provider preference
            if prefer_provider and model.provider != prefer_provider:
                continue

            candidates.append(model)

        if not candidates:
            # Fallback: return cheapest model regardless of quality
            return min(self.models, key=lambda m: m.input_cost_per_1m)

        # Return cheapest candidate
        return min(candidates, key=lambda m: m.input_cost_per_1m)


router = CostBasedRouter()

# Simple queries -> cheapest model
simple_model = router.select_model(quality_required="low")
print(f"Simple query: {simple_model.name} (${simple_model.input_cost_per_1m}/1M input)")

# Complex analysis -> high quality
complex_model = router.select_model(quality_required="high")
print(f"Complex query: {complex_model.name} (${complex_model.input_cost_per_1m}/1M input)")

# Budget-constrained tenant
budget_model = router.select_model(max_cost_per_1m_input=1.0)
print(f"Budget model: {budget_model.name}")
```

### Query Complexity Classification

```python
import tiktoken


def classify_query_complexity(question: str, context_length: int) -> str:
    """Classify query complexity to determine model routing."""
    encoding = tiktoken.get_encoding("cl100k_base")
    question_tokens = len(encoding.encode(question))

    # Simple: short question, small context
    if question_tokens < 30 and context_length < 2000:
        return "low"

    # Complex: long question with analysis keywords
    complexity_keywords = [
        "compare", "analyze", "explain why", "what are the tradeoffs",
        "step by step", "in detail", "comprehensive", "pros and cons",
    ]
    has_complexity_keyword = any(kw in question.lower() for kw in complexity_keywords)

    if has_complexity_keyword or context_length > 8000:
        return "high"

    return "medium"


# Usage in a RAG pipeline
def rag_with_routing(question: str, context: str) -> dict:
    complexity = classify_query_complexity(question, len(context.split()))
    model = router.select_model(quality_required=complexity)

    # Use the selected model for generation
    # ...
    return {"model_used": model.name, "complexity": complexity}
```

---

## Optimization Strategies Summary

| Strategy | Savings | Complexity | Best For |
|----------|---------|------------|----------|
| Batch API for offline work | 50% | Low | Document processing, evaluation |
| Model routing by complexity | 30-60% | Medium | Mixed-complexity query workloads |
| Per-tenant cost caps | Prevents overruns | Medium | Multi-tenant SaaS |
| Prompt compression | 20-40% | Medium | Long-context RAG |
| Caching frequent queries | 50-90% | Low | Repeated similar queries |
| Self-hosted embeddings | 80-95% | High | High-volume embedding workloads |

---

## Common Pitfalls

1. **Setting caps too low**: Overly aggressive cost caps cause legitimate queries to fail. Start with generous limits and tighten based on observed usage patterns.

2. **Not accounting for retries**: If a model routing decision sends a query to a cheap model that fails, the retry to a better model doubles the cost. Include retry costs in budget calculations.

3. **Complexity classification errors**: Routing a complex query to a cheap model produces poor answers, leading to user retries (more cost). Err on the side of higher quality when uncertain.

4. **Daily cap reset timing**: If your tenants are global, "daily" means different things in different timezones. Use UTC consistently and document this.

5. **Not alerting before the limit**: If alerts only fire at 100% budget, there is no time to react. Set alerts at 50%, 70%, and 90% to give teams time to adjust.

6. **Ignoring embedding costs in routing**: Embedding costs are typically 10-100x cheaper than generation costs. Focus routing optimization on the generation step, not embeddings.

---

## References

- OpenAI pricing: https://openai.com/pricing
- Anthropic pricing: https://docs.anthropic.com/en/docs/about-claude/pricing
- Grafana alerting: https://grafana.com/docs/grafana/latest/alerting/
- Rate limiting patterns: https://cloud.google.com/architecture/rate-limiting-strategies-techniques
