# Model Gateway and LLM Routing Patterns

## Overview

A model gateway is a single control point between applications and many models or
providers. It turns "which model should this request use?" and the cross-cutting
concerns of LLM access into infrastructure rather than per-app code.

## What a gateway centralizes

- **Routing:** select a model per request by cost, quality, latency, context
  length, or capability; run A/B tests and canary new models.
- **Fallback and resilience:** retry and fail over across providers on error or
  rate limit; circuit-break a failing provider.
- **Cost control:** per-team and per-app budgets and quotas, cost attribution,
  and routing cheap queries to cheap models.
- **Caching:** exact and **semantic** caching to skip duplicate or near-duplicate
  calls.
- **Security and governance:** central API-key custody, PII redaction, audit
  logs, and policy over which teams may call which models.
- **Observability:** latency, token, cost, and error metrics in one place.

Common implementations are LiteLLM, Envoy AI Gateway, cloud AI gateways, or a
custom service.

## Design decisions

- **Routing policy:** static rules versus a learned/heuristic router that
  predicts difficulty. Keep it explainable and mind the added hop latency.
- **Streaming:** the gateway must pass through token streaming with low overhead.
- **Statelessness:** keep the gateway stateless and horizontally scalable, with
  state (cache, budgets) pushed to fast external stores.
- **Failure semantics:** define what happens when all providers fail.

## Guidance

Build or adopt a gateway when there are multiple models or providers, multiple
teams, and real cost or governance needs. For a single model, single team, or a
prototype, a gateway is premature — call the model directly and introduce the
gateway when the second model, provider, or team appears.
