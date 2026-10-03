# Evaluation Harness: Can a Decision Model Replace Your LLM Classifier? - Deep Reference

> Official Documentation: https://docs.typesafe.ai/api
> Official Documentation: https://docs.typesafe.ai/models
> Official Documentation: https://docs.typesafe.ai/confidence
> SDK: https://pypi.org/project/typesafe-sdk/
> LLM adapter: https://github.com/typesafe-ai/system-one-adapter-python (PyPI: `system-one-adapter`)
> Last verified: 2026-10-03
> Verified against: Python 3.12.10, typesafe-sdk 0.7.2 (httpx2 2.13.1, tenacity 9.1.4, pydantic 2.13.5), system-one-adapter 0.2.1 (PyPI; repository commit `e1d4cc9`, 2026-09-22), numpy 2.5.3, scipy 1.18.1, scikit-learn 1.9.1; TypeSafe docs for `jev-1.13.0`

## Overview

This page is a complete, runnable harness for one decision: **can a typed decision model (TypeSafe Jev, or anything that returns class probabilities) take over a classification job that an LLM does today, on your own data?** It answers with numbers from your labelled corpus, never from a vendor benchmark: accuracy with a confidence interval, calibration against its noise floor, option-order stability, the bar each question needs given what its mistakes cost, the hand-off rate of a Jev-first cascade, and the price per 1,000 items.

The LLM baseline goes through **the same calling code** as Jev. TypeSafe publishes `system-one-adapter`, a "drop-in replacement for `typesafe_sdk`'s `system_one` evaluation API, backed by LLM APIs instead of TypeSafe" (README, https://github.com/typesafe-ai/system-one-adapter-python, commit `e1d4cc9`). It takes the same `Choice`/`Noul` objects and returns a `typesafe_sdk.SystemOneResponse` subclass, so the runner, the cache and the report do not know which model answered.

Every statistic is computed by the functions of `kb_calib.py`, the companion module of the sibling pages `calibration.md`, `order-bias.md` and `conformal.md`; the harness imports that file unchanged (same SHA-256 as the one those pages were verified with), and the functions it uses are reproduced verbatim in the [appendix](#appendix-kb_calib-functions-used-verbatim). Nothing is reimplemented. The full module is not published in this knowledge base. Save the appendix block as `kb_calib.py` next to the harness files and you have everything the harness and `sample_size.py` import. This was checked: a folder whose `kb_calib.py` was only the appendix block reproduced the same report.

**How this page was verified.** All eight Python files below were executed end to end on 2026-10-03, offline: Jev was a mock `httpx2` transport plugged into the real `AsyncTypeSafeClient` (with injected 429 and 529 responses), and the LLM was a fake provider plugged into the real `AsyncSystemOneAdapterClient` (with injected malformed output). The report in [The report it produced](#the-report-it-produced) is the verbatim output of that run. No real API was called and no API key was used; the paths that need one are marked where they appear. The models in the run are **simulations with known behaviour**, so the numbers illustrate what the harness detects; they are not measurements of Jev or any LLM.

Sibling pages: `calibration.md` (estimators, fits, sample sizes), `order-bias.md` (rotations, PriDe), `conformal.md` (prediction sets), `provider-logprobs.md` (which LLM APIs expose probabilities at all), and `../typesafe-jev/python-sdk.md` (the SDK in depth, including a request/token rate limiter).

---

## Table of Contents

1. [What "replace" means: the decision criteria](#what-replace-means-the-decision-criteria)
2. [Layout and data flow](#layout-and-data-flow)
3. [Corpus and question spec](#corpus-and-question-spec)
4. [Backends: Jev and an LLM behind one call](#backends-jev-and-an-llm-behind-one-call)
5. [Runner: bounded concurrency, retries, cache](#runner-bounded-concurrency-retries-cache)
6. [Report: metrics, fits, order, bars, cascade, cost](#report-metrics-fits-order-bars-cascade-cost)
7. [The CLI](#the-cli)
8. [Offline end-to-end run](#offline-end-to-end-run)
9. [The report it produced](#the-report-it-produced)
10. [Reading the report: what the simulation shows](#reading-the-report-what-the-simulation-shows)
11. [Running it for real](#running-it-for-real)
12. [Limits and what was not verified](#limits-and-what-was-not-verified)
13. [Appendix: kb_calib functions used (verbatim)](#appendix-kb_calib-functions-used-verbatim)
14. [Sources](#sources)

---

## What "replace" means: the decision criteria

A replacement decision is per question, not per model: Jev can be good enough to route tickets and not good enough to flag outages. The report gives, for each question, the five inputs below. The thresholds are the harness's suggestion, not an industry standard; set them before you look at the numbers.

| # | Criterion | Report line | Suggested pass rule |
|---|---|---|---|
| 1 | **Accuracy is not worse** | `difference ... -> non-inferior at margin` | Lower end of the paired 95% bootstrap CI of acc(Jev) - acc(LLM) above `-margin` (default 2 points). This is a one-sided non-inferiority test at 2.5%. |
| 2 | **Probabilities can be trusted after a fit** | `... on cal; test: ... ECE x (noise floor ..., 95th ...: inside)` | Test-split ECE after the cal-split fit at or below the 95th percentile of the noise floor. Raw miscalibration is fine if one parameter (temperature) or two (Platt) fixes it. |
| 3 | **The answer does not depend on option order** | `flip rate` | Flip rate small enough for your use, or budget the rotations: on Jev they cost question tokens only, because all orders go in one request. |
| 4 | **The cheapest safe policy costs less** | `Policies` table | Jev's lowest `cost / item` at least as low as the LLM's lowest. This is where asymmetric costs enter. |
| 5 | **The money** | `Cost ... per 1,000 items` | Your budget. |

Criterion 4 is the one that decides most real cases. Two models with equal accuracy are not equally useful if one of them puts its errors at confidence 0.95 and the other at 0.6: the second can be gated, the first cannot.

---

## Layout and data flow

```text
corpus.jsonl ─┐                      ┌─ JevBackend ──── AsyncTypeSafeClient ─────────┐
questions.py ─┴─> run_backend() ─────┤                                               ├─> responses.jsonl (cache)
                  (semaphore, keys)  └─ AdapterBackend ─ AsyncSystemOneAdapterClient ─┘            │
                                                                                                   v
                                                                     build_report() ── kb_calib ──> report.md
```

| File | Role | Needs the output of |
|---|---|---|
| `kb_calib.py` | All statistics (unchanged copy) | - |
| `harness_spec.py` | `QuestionSpec`, corpus loader, hashed split | `kb_calib.rotations` |
| `harness_backends.py` | `JevBackend`, `AdapterBackend` | - |
| `harness_runner.py` | `ResponseCache`, `run_backend` | spec, backends |
| `harness_report.py` | Collect from cache, fit, bars, cascade, cost | the cache only: it never calls a model |
| `questions.py` | Your questions and costs | spec |
| `run_eval.py` | CLI wiring | all of the above |
| `offline_demo.py` | Synthetic corpus, mock transport, fake provider | `run_eval.amain` |

The two backends do not depend on each other: they read the same corpus and write to the same cache under different keys, and `run_eval.py` runs them one after the other only so that each backend's rate limit is easy to reason about. The report depends on **both** only for the comparison and the cascade section; with only the Jev run it prints the per-backend sections. Because the report reads the cache and nothing else, you can re-run it with a different margin, split or cost matrix for free.

---

## Corpus and question spec

The corpus is JSONL, one item per line: an `id`, the `state` exactly as you would send it (string, JSON object or array, the three shapes `SystemOneRequest.state` accepts in the OpenAPI spec), and `labels`, one gold label per question name. Two lines of the synthetic corpus the run used:

```json
{"id": "t0000", "state": "[t0000] The export job fails with error 502 since this morning.", "labels": {"department": "technical", "urgent": true}}
{"id": "t0001", "state": "[t0001] I was charged twice for the March invoice.", "labels": {"department": "sales", "urgent": false}}
```

An item may leave a question unlabelled (`null` or absent): the runner does not ask it and the report does not count it. Labels must come from people who know what the right action is, not from another model; a corpus labelled by a model measures agreement (see the `decision-model-calibration` skill, section 1).

The split is a hash of the item id, not a random draw: it is stable across runs and machines, and when the corpus grows, old items stay on their side. The fit (temperature or Platt) and every bar are chosen on the **cal** side and every number in the report except the flip rate is measured on the **test** side.

`harness_spec.py` (executed):

```python
"""Question specs, the labelled corpus, and the cal/test split."""

from __future__ import annotations

import hashlib
import json
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any, Literal

import numpy as np
from typesafe_sdk import Choice, Noul

import kb_calib as kc


@dataclass(frozen=True)
class QuestionSpec:
    """One decision the classifier must make, plus what its mistakes cost.

    costs[(true, predicted)] is the cost of ACTING on `predicted` when the truth is
    `true`; unlisted off-diagonal pairs cost `default_error_cost`, the diagonal 0.
    `handoff_cost` is what it costs to send the item to the fallback (an LLM, a
    person) instead of acting. All costs share one unit (cents, minutes, ...).
    """

    name: str
    kind: Literal["choice", "noul"]
    instructions: str
    options: dict[str, str | None] | None = None  # choice: label -> description, canonical order
    noul_criteria: dict[str, str] | None = None  # noul: optional {"true": ..., "false": ...}
    costs: dict[tuple[str, str], float] = field(default_factory=dict)
    default_error_cost: float = 1.0
    handoff_cost: float = 0.2
    target_error: float = 0.05  # for kb_calib.lowest_bar_for
    n_rotations: int | None = None  # choice only: None = all k cyclic orders, 1 = no order check

    @property
    def labels(self) -> list[str]:
        """Column order of every probability matrix for this question."""
        return list(self.options or {}) if self.kind == "choice" else ["no", "yes"]

    def cost_matrix(self) -> np.ndarray:
        """C[i, j] = cost of predicting labels[j] when the truth is labels[i]."""
        labs = self.labels
        c = np.full((len(labs), len(labs)), float(self.default_error_cost))
        np.fill_diagonal(c, 0.0)
        for (t, p), v in self.costs.items():
            c[labs.index(t), labs.index(p)] = v
        return c

    def variants(self) -> list[tuple[str, tuple[str, ...], Any]]:
        """(question key, option order, SDK question) for every order to ask."""
        if self.kind == "noul":
            q = Noul(instructions=self.instructions, criteria=self.noul_criteria)
            return [(self.name, ("no", "yes"), q)]
        out = []
        for r, crit in enumerate(kc.rotations(self.options, self.n_rotations)):
            q = Choice(instructions=self.instructions, criteria=crit)
            out.append((f"{self.name}__r{r}", tuple(crit), q))
        return out


@dataclass(frozen=True)
class Item:
    id: str
    state: Any  # str, JSON object or array: whatever you send as `state`
    labels: dict[str, Any]  # question name -> label (choice: option name; noul: bool or "yes"/"no")


def load_corpus(path: str | Path) -> list[Item]:
    """JSONL, one item per line: {"id": ..., "state": ..., "labels": {question: label}}."""
    items, seen = [], set()
    for n, line in enumerate(Path(path).read_text(encoding="utf-8").splitlines(), 1):
        if not line.strip():
            continue
        rec = json.loads(line)
        if rec["id"] in seen:
            raise ValueError(f"line {n}: duplicate id {rec['id']!r}")
        seen.add(rec["id"])
        items.append(Item(str(rec["id"]), rec["state"], dict(rec["labels"])))
    return items


def label_index(spec: QuestionSpec, value: Any) -> int | None:
    """Column index of a gold label; None when the item is unlabelled for this question."""
    if value is None:
        return None
    if spec.kind == "noul":
        if isinstance(value, str):
            value = {"yes": True, "no": False, "true": True, "false": False}[value.lower()]
        return int(bool(value))
    return spec.labels.index(value)


def is_calibration(item_id: str, cal_fraction: float = 0.5, salt: str = "split-v1") -> bool:
    """Deterministic split by hashed id: stable across runs, machines and corpus growth."""
    h = int.from_bytes(hashlib.sha256(f"{salt}:{item_id}".encode()).digest()[:8], "big")
    return h / 2**64 < cal_fraction
```

Design notes:

- **Noul is a two-column matrix** `[1 - p, p]` over `["no", "yes"]`, so one code path handles both kinds; `kb_calib.top_label` then gives `max(p, 1 - p)` as the confidence, as `calibration.md` prescribes for Noul.
- **Rotations come from `kb_calib.rotations`** (cyclic shifts, k orders rather than k!), and each order is a separate question key, `department__r0` ... `department__r3`. `n_rotations=1` turns the order check off; `None` asks all k orders.
- **Score questions are not supported** by this harness. A Score could be treated as an ordered Choice over levels `0..k-1`, but the `score` field is an expected value and accuracy is not the right metric for it; that extension was not built or tested.
- **Costs** are a matrix over (true, predicted) plus one hand-off cost per question, all in the same unit. The diagonal is zero; anything you do not list costs `default_error_cost`.

The question spec used in the run (`questions.py`, executed):

```python
"""The question spec: edit this file for your project. Costs are in cents per ticket."""

from harness_spec import QuestionSpec

SPECS = [
    QuestionSpec(
        name="department",
        kind="choice",
        instructions="Which team should handle this support ticket?",
        options={
            "billing": "Charges, refunds, invoices, payment methods",
            "technical": "Errors, outages, bugs, integrations",
            "sales": "Pricing questions, upgrades, new accounts",
            "account": "Login, profile, permissions, account deletion",
        },
        # A billing ticket sent to sales loses a refund window; the reverse only costs a hop.
        costs={("billing", "sales"): 5.0, ("billing", "technical"): 3.0, ("account", "sales"): 3.0},
        default_error_cost=1.0,
        handoff_cost=0.4,  # what the fallback (LLM call or a person) costs per ticket
        target_error=0.05,
    ),
    QuestionSpec(
        name="urgent",
        kind="noul",
        instructions="The customer cannot use the product at all right now.",
        # Missing a real outage is ten times worse than paging someone for nothing.
        costs={("yes", "no"): 10.0, ("no", "yes"): 1.0},
        handoff_cost=0.4,
        target_error=0.05,
    ),
]
```

---

## Backends: Jev and an LLM behind one call

Facts this file relies on, each checked in the installed package source or the docs on 2026-10-03:

- `AsyncTypeSafeClient(*, api_key=None, model=None, retry=None, timeout=None, headers=None, transport=None, http_client=None, base_url=None)` and `system_one(state, questions, *, model=None, retry=None, timeout=None, extra_headers=None, extra_body=None, response_model=None)` (typesafe-sdk 0.7.2, `inspect.signature`). The key comes from `TYPESAFE_API_KEY` when `api_key` is not passed (`typesafe_sdk.constants.API_KEY_ENV`); the default model is `jev-latest` and the default per-operation timeout 10.0 s (same module).
- The response's `raw_http_response` is the `httpx2.Response` it was parsed from; on a response that was not built from HTTP it raises `TypeSafeError("The response was not created from a raw HTTP response.")` (source `typesafe_sdk/_core/schemas/base.py`, and executed against an adapter response). That is why the Jev backend caches the raw body and the adapter backend caches `model_dump(mode="json")`.
- Jev pricing: "\$42 / \$0.042" per Btok / per Mtok, "Charged per input token. Output tokens are free" (https://docs.typesafe.ai/models, fetched 2026-10-03, model `jev-1.13.0`).
- Jev "ingests the `state` once and evaluates every question against it in parallel" (same page). That is the reason all rotations of a question share one Jev request: the extra orders cost their question tokens and no extra read of the state.
- `AsyncSystemOneAdapterClient(*, structured_outputs, llm_answer_mode, normalize_probabilities=False, n_retry_malformed_structure=0, retry=None, provider=None, model=None)`; `system_one(state, questions, *, provider=None, model=None, retry=None)`; `model` may be a provider instance, whose `model_name` names it (source `system_one_adapter/_client.py`, 0.2.1).
- The adapter's response usage adds `input_tokens_total` / `output_tokens_total` "across retries", `n_retries`, `n_retries_malformed_structure` and `latency` (README). The harness bills the `*_total` counts, since corrective retries are paid for. A count the provider did not report is `None` (README); the cost code treats it as 0, which under-counts, so check for it on a real run.
- **The adapter does not retry transient failures by default**: `retry` defaults to `RetryPolicy(max_retries=0)` (source, and printed by the executed check), and the built-in OpenAI and Anthropic providers construct their SDK clients with `max_retries=0` (source `providers/openai.py`, `providers/anthropic.py`). Pass a `RetryPolicy` or a 429 from the LLM provider fails the item. `typesafe_sdk`'s own default is `RetryPolicy()` = 2 retries on 408, 429 and 5xx, honouring `retry-after` / `retry-after-ms`, with a 30 s total budget (source `typesafe_sdk/_core/retry.py`).
- In `"probabilities"` mode the adapter asks the LLM for a number per Noul and an object mapping every label to a probability per Choice/Score, through a JSON Schema whose property names are the question keys (source `_client.py` system prompt; schema printed by the executed probe). The LLM therefore **verbalises** probabilities; they are not token logprobs. Whether verbalised numbers calibrate as well as logprobs is exactly what this harness measures; see `provider-logprobs.md` for the logprob route.
- An LLM reads the whole prompt, so two orders of the same question in one prompt can see each other. The adapter backend therefore sends each rotation in its own request (`rotations_in_one_request = False`).

`harness_backends.py` (executed):

```python
"""Two backends behind one call signature: Jev through typesafe-sdk, an LLM through
system-one-adapter. Both take the SAME typesafe_sdk question objects and return the
same raw JSON shape, so everything downstream is shared."""

from __future__ import annotations

from collections.abc import Mapping
from dataclasses import dataclass
from typing import Any

from typesafe_sdk import AsyncTypeSafeClient


@dataclass
class JevBackend:
    client: AsyncTypeSafeClient
    model: str  # pin a versioned id (e.g. "jev-1.13.0"): it is part of the cache key
    name: str = "jev"
    price_in_per_mtok: float = 0.042  # docs.typesafe.ai/models, jev-1.13.0, read 2026-10-03
    price_out_per_mtok: float = 0.0  # "Output tokens are free" (same page)
    # The models page: Jev "ingests the state once and evaluates every question against
    # it in parallel", so all option orders of a question can share one request.
    rotations_in_one_request: bool = True

    async def call(self, state: Any, questions: Mapping[str, Any]) -> dict[str, Any]:
        resp = await self.client.system_one(state, questions, model=self.model)
        raw = resp.raw_http_response.json()  # the bytes the API sent, not a re-serialisation
        return {
            "model": raw["model"],
            "answers": raw["answers"],
            "usage": {"input_tokens": resp.usage.input_tokens, "output_tokens": resp.usage.output_tokens},
            "request_id": resp.raw_http_response.headers.get("x-typesafe-request-id"),
        }


@dataclass
class AdapterBackend:
    """An LLM answering the same questions via typesafe-ai/system-one-adapter-python."""

    client: Any  # system_one_adapter.AsyncSystemOneAdapterClient
    model: Any  # model name, or a provider instance (then its .model_name is the cache key)
    provider: str | None = None  # "openai" | "anthropic" | "gemini"; None with a provider instance
    name: str = "llm"
    price_in_per_mtok: float = 0.0  # set from YOUR provider's price page
    price_out_per_mtok: float = 0.0
    # An LLM reads the whole prompt: two orders of one question in the same prompt
    # can see each other, so each rotation goes in its own request.
    rotations_in_one_request: bool = False

    @property
    def model_key(self) -> str:
        return self.model if isinstance(self.model, str) else self.model.model_name

    async def call(self, state: Any, questions: Mapping[str, Any]) -> dict[str, Any]:
        resp = await self.client.system_one(state, questions, provider=self.provider, model=self.model)
        dumped = resp.model_dump(mode="json")  # no HTTP response to keep: the adapter builds it
        u = resp.usage
        return {
            "model": dumped["model"],
            "answers": dumped["answers"],
            # *_total include corrective and transient retries: that is what you pay for.
            "usage": {"input_tokens": u.input_tokens_total, "output_tokens": u.output_tokens_total},
            "n_retries": u.n_retries,
            "n_retries_malformed_structure": u.n_retries_malformed_structure,
            "latency": u.latency,
        }


def cache_model_key(backend: Any) -> str:
    return backend.model_key if isinstance(backend, AdapterBackend) else backend.model
```

---

## Runner: bounded concurrency, retries, cache

**Cache key.** Every answer is stored under `(item id, question name, requested model, option order)`. That gives four properties for free:

1. An interrupted run resumes: only the missing keys are asked.
2. Adding a question asks only that question; adding items asks only those items.
3. A new model (or a new pinned Jev version) is a new key, so old answers are kept for comparison, never overwritten.
4. Each rotation is a separate entry, so the order audit costs nothing on a re-run.

The key uses the **requested** model, because that is all the runner knows before calling. The served model (the response's `model` field) is stored in every record and printed in the report. Pin a versioned ID such as `jev-1.13.0`: TypeSafe's models page says an alias "moves when a new release ships, so the answers behind it can change without a change on your side", and "If you have tuned confidence thresholds against a specific version, pin that version's ID instead of the alias" (https://docs.typesafe.ai/models, fetched 2026-10-03). `run_eval.py` warns when you request `jev-latest` or `jev-preview`.

**Failures.** The runner catches `TypeSafeError` (the base class of every SDK error; the adapter raises the same classes) **after** the client's retry policy is exhausted, writes an error record and moves on. Error records are never cache hits, so the next run asks those items again, and the report counts them as `missing/failed`. In the run below, the mock answered item `t0013` with `529 Overloaded` forever: the SDK made 5 attempts (1 + `max_retries=4`), raised `TypeSafeInternalServerError`, and the item was excluded and counted.

**Concurrency is not rate.** The semaphore bounds requests in flight, which bounds memory and sockets. It does not bound requests per second: by Little's law, rate = in-flight / latency, so 8 slots at 50 ms latency is 160 req/s, above Jev's published 80 req/s (https://docs.typesafe.ai/models, 2026-10-03, which also says the limits "can change without notice"). The SDK's retry policy absorbs occasional 429s; for a large run, add the sliding-window limiter from `../typesafe-jev/python-sdk.md` ("Production Pattern: Bounded Concurrency Under the Rate Limit") in front of `backend.call`.

One record as written to the cache (from the run; pretty-printed here, one line on disk):

```json
{
 "backend": "jev",
 "item_id": "t0001",
 "requested_model": "jev-1.13.0",
 "questions": {
  "department__r0": {
   "question": "department",
   "order": [
    "billing",
    "technical",
    "sales",
    "account"
   ]
  },
  "department__r1": {
   "question": "department",
   "order": [
    "technical",
    "sales",
    "account",
    "billing"
   ]
  },
  "department__r2": {
   "question": "department",
   "order": [
    "sales",
    "account",
    "billing",
    "technical"
   ]
  },
  "department__r3": {
   "question": "department",
   "order": [
    "account",
    "billing",
    "technical",
    "sales"
   ]
  },
  "urgent": {
   "question": "urgent",
   "order": [
    "no",
    "yes"
   ]
  }
 },
 "raw": {
  "model": "jev-1.13.0",
  "answers": {
   "department__r0": {
    "type": "choice",
    "choice": "billing",
    "probabilities": {
     "billing": 0.97,
     "technical": 0.0,
     "sales": 0.03,
     "account": 0.0
    },
    "confidence": 0.96
   },
   "department__r1": {
    "type": "choice",
    "choice": "billing",
    "probabilities": {
     "technical": 0.0,
     "sales": 0.05,
     "account": 0.0,
     "billing": 0.95
    },
    "confidence": 0.93
   },
   "department__r2": {
    "type": "choice",
    "choice": "billing",
    "probabilities": {
     "sales": 0.06,
     "account": 0.0,
     "billing": 0.94,
     "technical": 0.0
    },
    "confidence": 0.92
   },
   "department__r3": {
    "type": "choice",
    "choice": "billing",
    "probabilities": {
     "account": 0.0,
     "billing": 0.96,
     "technical": 0.0,
     "sales": 0.04
    },
    "confidence": 0.95
   },
   "urgent": {
    "type": "noul",
    "noul": 0.04
   }
  },
  "usage": {
   "input_tokens": 375,
   "output_tokens": 50
  },
  "request_id": "req-15"
 },
 "elapsed_s": 0.0243
}
```

`request_id` and `elapsed_s` differ on every run: concurrent requests are scheduled in a different order each time, and the mock numbers its responses in arrival order.

And a failed request:

```json
{
 "error": "TypeSafeInternalServerError",
 "message": "POST https://api.typesafe.ai/v1/systemone: 529 Overloaded",
 "backend": "jev",
 "item_id": "t0013",
 "requested_model": "jev-1.13.0",
 "questions": [
  "department__r0",
  "department__r1",
  "department__r2",
  "department__r3",
  "urgent"
 ]
}
```

`harness_runner.py` (executed):

```python
"""Async runner with bounded concurrency and an append-only JSONL response cache.

Cache key: (item id, question name, model, option order). A rerun only asks what is
missing, so an interrupted run resumes, adding a question re-asks only that
question, and changing the model asks everything again under the new key.
"""

from __future__ import annotations

import asyncio
import json
import time
from collections import defaultdict
from pathlib import Path
from typing import Any

from typesafe_sdk import TypeSafeError

from harness_backends import cache_model_key
from harness_spec import Item, QuestionSpec


class ResponseCache:
    def __init__(self, path: str | Path) -> None:
        self.path = Path(path)
        self.index: dict[tuple[str, str, str, tuple[str, ...]], dict[str, Any]] = {}
        self.records: list[dict[str, Any]] = []
        self.errors: list[dict[str, Any]] = []
        if self.path.exists():
            for line in self.path.read_text(encoding="utf-8").splitlines():
                if line.strip():
                    self._index(json.loads(line))

    def _index(self, rec: dict[str, Any]) -> None:
        if "error" in rec:
            self.errors.append(rec)  # kept for the report, never a cache hit: retried next run
            return
        self.records.append(rec)
        for qkey, meta in rec["questions"].items():
            k = (rec["item_id"], meta["question"], rec["requested_model"], tuple(meta["order"]))
            self.index[k] = {"answer": rec["raw"]["answers"][qkey], "served_model": rec["raw"]["model"]}

    def get(self, item_id: str, question: str, model: str, order: tuple[str, ...]) -> dict[str, Any] | None:
        return self.index.get((item_id, question, model, tuple(order)))

    def append(self, rec: dict[str, Any]) -> None:
        with self.path.open("a", encoding="utf-8") as f:  # one line per request, flushed at once
            f.write(json.dumps(rec, ensure_ascii=False) + "\n")
        self._index(rec)


async def run_backend(
    backend: Any,
    corpus: list[Item],
    specs: list[QuestionSpec],
    cache: ResponseCache,
    *,
    concurrency: int = 8,
) -> dict[str, Any]:
    """Ask every (item, question, order) the cache does not hold yet."""
    model = cache_model_key(backend)
    jobs: list[tuple[Item, dict[str, tuple[str, tuple[str, ...], Any]]]] = []
    for item in corpus:
        groups: dict[int, dict[str, tuple[str, tuple[str, ...], Any]]] = defaultdict(dict)
        for spec in specs:
            if item.labels.get(spec.name) is None:
                continue  # unlabelled for this question: asking it would cost and teach nothing
            for r, (qkey, order, q) in enumerate(spec.variants()):
                if cache.get(item.id, spec.name, model, order) is None:
                    groups[0 if backend.rotations_in_one_request else r][qkey] = (spec.name, order, q)
        jobs += [(item, g) for _, g in sorted(groups.items())]

    sem = asyncio.Semaphore(concurrency)
    stats = {"backend": backend.name, "requests": len(jobs), "ok": 0, "errors": 0}

    async def one(item: Item, group: dict[str, tuple[str, tuple[str, ...], Any]]) -> None:
        async with sem:
            t0 = time.perf_counter()
            try:
                out = await backend.call(item.state, {k: v[2] for k, v in group.items()})
            except TypeSafeError as e:  # retries are exhausted by now (SDK / adapter RetryPolicy)
                stats["errors"] += 1
                cache.append({"error": type(e).__name__, "message": str(e)[:300], "backend": backend.name,
                              "item_id": item.id, "requested_model": model, "questions": sorted(group)})
                return
            stats["ok"] += 1
            cache.append({
                "backend": backend.name,
                "item_id": item.id,
                "requested_model": model,
                "questions": {k: {"question": v[0], "order": list(v[1])} for k, v in group.items()},
                "raw": out,
                "elapsed_s": round(time.perf_counter() - t0, 4),
            })

    t0 = time.perf_counter()
    await asyncio.gather(*(one(i, g) for i, g in jobs))
    stats["wall_s"] = round(time.perf_counter() - t0, 2)
    return stats
```

---

## Report: metrics, fits, order, bars, cascade, cost

What each section of the report computes, and with which `kb_calib` function:

| Report item | Computation | kb_calib |
|---|---|---|
| accuracy `[lo, hi]` | mean of correct on test, 2,000-resample percentile bootstrap | `top_label`, `bootstrap_ci` |
| ECE / noise floor | equal-mass ECE, 10 bins; floor = median and 95th percentile of the ECE a perfectly calibrated model would show **at these confidences and this n** (2,000 simulations) | `ece`, `ece_noise_floor` |
| Brier, NLL | multiclass Brier (range 0-2); log loss with the 0.005 floor for Jev's rounded zeros | `brier`, `nll` |
| fit | Choice: one temperature on order-0 probabilities. Noul: Platt on `p_yes`. Fitted on cal, applied to all | `fit_temperature`/`apply_temperature`, `fit_platt`/`apply_platt` |
| flip rate | share of items whose pick differs across the k cyclic orders, Wilson CI; on **all** items (no fit involved) | `rotations`, `flip_rate` |
| risk-coverage | coverage and error among kept items at each bar on the calibrated top probability; the table shows the first observed value at or above each displayed bar | `risk_coverage` |
| bars | see below | `risk_coverage`, `lowest_bar_for` |
| paired comparison | CI of acc(primary) - acc(fallback) on the same test items | `paired_bootstrap_diff` |
| cascade hand-off | share of items with any question below its bar, Wilson CI | `wilson_interval`, `bootstrap_ci` |

**Why order-0 probabilities and not the rotation average.** Production will most likely send one order per question. The report therefore fits and gates on what a single call returns, and prints the rotation-averaged accuracy next to it so you can see whether paying for rotations would change the answer.

**Three bars per question, all chosen on cal and measured on test.** With costs `C[true, predicted]` and hand-off cost `h`:

1. **Top-probability bar at minimum cost.** Every observed calibrated top probability on cal is a candidate (plus "hand off everything"); for each, the cost per item is `C[y, argmax]` for kept items and `h` for the rest; the cheapest wins. This is the rule most code ships (`if p_max >= bar: act`).
2. **Top-probability bar for a target error.** `lowest_bar_for(target_error, ...)`: the lowest bar whose kept items err at most `target_error` on cal, with at least 30 kept. It ignores costs; it is there because many teams state requirements as an error rate.
3. **Bayes rule on calibrated probabilities.** For each item, the expected cost of acting on label j is `sum_i p_i * C[i, j]`; act on the cheapest label if its expected cost is at most `h`, otherwise hand off. This is the cost-minimising decision **if** the probabilities are calibrated, which is why it runs on the fitted ones. Unlike a top-probability bar, it can act on a label that is not the most likely one. With the `urgent` costs below (missing an outage 10, a false page 1, hand-off 0.4), acting "yes" costs `1 - p` in expectation and acting "no" costs `10 p`: "yes" is the cheaper action from p = 1/11 on, and it is taken once `1 - p <= 0.4`, that is from p = 0.6, while "no" is taken only up to p = 0.04. A top-probability bar of 0.6 would instead act "no" at p = 0.35.

**Cascade.** The primary answers every item. For each question, the bar is the primary's min-cost top-probability bar. One fallback call answers all questions of an item, so the **item** is handed off when any question is below its bar, and that is what you pay for; each question then keeps the primary's answer where it cleared its own bar and takes the fallback's where it did not. The fallback's answer is its raw order-0 argmax.

**Cost.** Per item, from the cached `usage` of every request for that item, at the backend's configured prices. Two figures: all requests (what the evaluation spent, rotations included) and the request(s) holding order 0 (what a single-order deployment would send). For the LLM those differ by the number of rotations. For Jev they are equal, because the rotations ride in the same request; the Jev figure is therefore an **upper bound** on single-order production cost, inflated by the rotation questions' tokens.

`harness_report.py` (executed):

```python
"""Turn cached answers into the decision report. Every statistic comes from kb_calib."""

from __future__ import annotations

from collections import defaultdict
from typing import Any

import numpy as np

import kb_calib as kc
from harness_runner import ResponseCache
from harness_spec import Item, QuestionSpec, is_calibration, label_index

DISPLAY_BARS = (0.5, 0.6, 0.7, 0.8, 0.9, 0.95, 0.99)


# ------------------------------------------------------------------ gather
def collect(cache: ResponseCache, model: str, corpus: list[Item], spec: QuestionSpec) -> dict[str, Any]:
    """Probabilities in CANONICAL option order for every order asked: probs (n, R, k)."""
    labs, variants = spec.labels, spec.variants()
    ids, ys, probs, wins, served, missing = [], [], [], [], set(), 0
    for item in corpus:
        y = label_index(spec, item.labels.get(spec.name))
        if y is None:
            continue
        hits = [cache.get(item.id, spec.name, model, order) for _, order, _ in variants]
        if any(h is None for h in hits):
            missing += 1  # failed or not yet asked: excluded, and counted in the report
            continue
        rows, w = [], []
        for h in hits:
            a = h["answer"]
            served.add(h["served_model"])
            if spec.kind == "noul":
                rows.append([1.0 - float(a["noul"]), float(a["noul"])])
                w.append(int(float(a["noul"]) >= 0.5))
            else:
                rows.append([float(a["probabilities"].get(lab, 0.0)) for lab in labs])
                w.append(labs.index(a["choice"]))
        ids.append(item.id)
        ys.append(y)
        probs.append(rows)
        wins.append(w)
    return {
        "ids": np.array(ids),
        "y": np.array(ys, int),
        "probs": np.array(probs, float).reshape(len(ids), len(variants), len(labs)),
        "winners": np.array(wins, int).reshape(len(ids), len(variants)),
        "served": sorted(served),
        "missing": missing,
    }


def item_costs(cache: ResponseCache, backend: Any, model: str) -> dict[str, dict[str, float]]:
    """Per item: tokens and dollars over all requests, and over the request(s) holding
    order 0 (what a single-order deployment would send)."""
    agg: dict[str, dict[str, float]] = defaultdict(lambda: defaultdict(float))
    for rec in cache.records:
        if rec["backend"] != backend.name or rec["requested_model"] != model:
            continue
        u = rec["raw"]["usage"]
        tin, tout = u.get("input_tokens") or 0, u.get("output_tokens") or 0
        usd = (tin * backend.price_in_per_mtok + tout * backend.price_out_per_mtok) / 1e6
        a = agg[rec["item_id"]]
        a["in"] += tin
        a["out"] += tout
        a["usd"] += usd
        if any(k.endswith("__r0") or "__r" not in k for k in rec["questions"]):
            a["usd_order0"] += usd
    return agg


# ------------------------------------------------------------------ metrics
def metrics(P: np.ndarray, y: np.ndarray) -> dict[str, float]:
    conf, pred = kc.top_label(P)
    correct = (pred == y).astype(float)
    acc, lo, hi = kc.bootstrap_ci(np.mean, correct)
    f50, f95 = kc.ece_noise_floor(conf)
    return {"n": len(y), "acc": acc, "acc_lo": lo, "acc_hi": hi, "ece": kc.ece(conf, correct),
            "floor50": f50, "floor95": f95, "brier": kc.brier(P, y), "nll": kc.nll(P, y)}


def calibrate(spec: QuestionSpec, P: np.ndarray, y: np.ndarray, cal: np.ndarray) -> tuple[np.ndarray, str]:
    """Fit on the cal split only, apply everywhere: temperature (choice) or Platt (noul)."""
    if spec.kind == "choice":
        t = kc.fit_temperature(P[cal], y[cal])
        return kc.apply_temperature(P, t), f"temperature T = {t:.2f}"
    m = kc.fit_platt(P[cal, 1], y[cal])
    p = kc.apply_platt(m, P[:, 1])
    return np.column_stack([1.0 - p, p]), f"Platt a = {m.coef_[0][0]:.2f}, b = {m.intercept_[0]:.2f}"


def policy_cost(C, h, y, pred, auto) -> tuple[float, float, float]:
    """(coverage, error among automated, mean cost per item) of one gating policy."""
    cost = np.where(auto, C[y, pred], h)
    err = float((pred[auto] != y[auto]).mean()) if auto.any() else float("nan")
    return float(auto.mean()), err, float(cost.mean())


def bars(spec: QuestionSpec, Pc: np.ndarray, y: np.ndarray, cal: np.ndarray) -> dict[str, Any]:
    """Pick bars on the cal split, report them on the test split."""
    C, h = spec.cost_matrix(), spec.handoff_cost
    conf, pred = kc.top_label(Pc)
    correct = pred == y
    test = ~cal
    # 1. cost-optimal bar on the top probability, searched over every observed value
    cands = [r[0] for r in kc.risk_coverage(conf[cal], correct[cal])] + [np.inf]
    best = min(cands, key=lambda b: policy_cost(C, h, y[cal], pred[cal], conf[cal] >= b)[2])
    # 2. lowest bar meeting the target error on the cal split (kb_calib.lowest_bar_for)
    tgt = kc.lowest_bar_for(spec.target_error, conf[cal], correct[cal])
    # 3. Bayes rule on calibrated probabilities: act on argmin expected cost if <= h
    E = Pc @ C
    a, r = E.argmin(axis=1), E.min(axis=1)
    rows = {
        "automate all": policy_cost(C, h, y[test], pred[test], np.ones(test.sum(), bool)),
        "hand off all": policy_cost(C, h, y[test], pred[test], np.zeros(test.sum(), bool)),
        f"top-prob bar {best:.2f} (min cost on cal)": policy_cost(C, h, y[test], pred[test], conf[test] >= best),
    }
    if tgt is not None:
        rows[f"top-prob bar {tgt[0]:.2f} (error <= {spec.target_error:.0%} on cal)"] = policy_cost(
            C, h, y[test], pred[test], conf[test] >= tgt[0])
    rows["Bayes: act iff expected cost <= hand-off"] = policy_cost(C, h, y[test], a[test], r[test] <= h)
    rc = kc.risk_coverage(conf[test], correct[test])
    table = []
    for b in DISPLAY_BARS:
        hit = next((row for row in rc if row[0] >= b), None)
        if hit is not None:
            table.append((b, hit[0], hit[1], hit[2], int(round(hit[1] * test.sum()))))
    return {"best_bar": best, "policies": rows, "rc_table": table}


# ------------------------------------------------------------------ report
def fmt_m(m: dict[str, float]) -> str:
    noise = "inside" if m["ece"] <= m["floor95"] else "ABOVE"
    return (f"acc {m['acc']:.3f} [{m['acc_lo']:.3f}, {m['acc_hi']:.3f}] | ECE {m['ece']:.3f} "
            f"(noise floor {m['floor50']:.3f}, 95th {m['floor95']:.3f}: {noise}) | "
            f"Brier {m['brier']:.3f} | NLL {m['nll']:.3f}")


def build_report(corpus, specs, cache, backends, run_stats, *, primary="jev", fallback="llm",
                 margin=0.02, cal_fraction=0.5) -> str:
    out: list[str] = []
    w = out.append
    by_name = {b.name: b for b in backends}
    from harness_backends import cache_model_key

    w("# Decision-model evaluation report\n")
    w(f"Corpus: {len(corpus)} items. Split: hashed id, cal fraction {cal_fraction}.")
    for s in run_stats:
        w(f"Run {s['backend']}: {s['requests']} requests sent ({s['ok']} ok, {s['errors']} failed "
          f"after retries), {s['wall_s']} s.")
    w(f"Cache: {len(cache.records)} responses, {len(cache.errors)} error records.\n")

    data, cal_p = {}, {}
    for b in backends:
        model = cache_model_key(b)
        w(f"## Backend `{b.name}` (requested `{model}`)\n")
        for spec in specs:
            d = collect(cache, model, corpus, spec)
            data[(b.name, spec.name)] = d
            if len(d["y"]) == 0:
                w(f"### {spec.name}: no answered items\n")
                continue
            cal = np.array([is_calibration(i, cal_fraction) for i in d["ids"]])
            test = ~cal
            P0 = d["probs"][:, 0, :]
            Pc, fit = calibrate(spec, P0, d["y"], cal)
            cal_p[(b.name, spec.name)] = (Pc, cal)
            w(f"### {spec.name} ({spec.kind}, {len(spec.labels)} labels)\n")
            w(f"- served model(s): {', '.join(d['served'])}; items: {len(d['y'])} "
              f"(cal {cal.sum()}, test {test.sum()}); missing/failed: {d['missing']}")
            w(f"- raw, test:        {fmt_m(metrics(P0[test], d['y'][test]))}")
            w(f"- {fit} on cal; test: {fmt_m(metrics(Pc[test], d['y'][test]))}")
            if d["probs"].shape[1] > 1:
                rate, (lo, hi) = kc.flip_rate(d["winners"])
                avg_acc = float((d["probs"].mean(axis=1).argmax(axis=1) == d["y"])[test].mean())
                w(f"- option order: {d['probs'].shape[1]} cyclic rotations; flip rate {rate:.3f} "
                  f"[{lo:.3f}, {hi:.3f}] (all items); test acc order-0 "
                  f"{float((P0.argmax(1) == d['y'])[test].mean()):.3f} vs rotation-averaged {avg_acc:.3f}")
            bt = bars(spec, Pc, d["y"], cal)
            w("\nRisk-coverage, calibrated top probability, test split:\n")
            w("| bar asked | bar used | coverage | error kept | n kept |")
            w("|---|---|---|---|---|")
            for b_ask, b_used, cov, err, nk in bt["rc_table"]:
                w(f"| {b_ask:.2f} | {b_used:.3f} | {cov:.3f} | {err:.3f} | {nk} |")
            w(f"\nPolicies (bars chosen on cal, measured on test; hand-off cost {spec.handoff_cost}):\n")
            w("| policy | coverage | error kept | cost / item |")
            w("|---|---|---|---|")
            for name, (cov, err, cost) in bt["policies"].items():
                w(f"| {name} | {cov:.3f} | {err:.3f} | {cost:.4f} |")
            w("")
            cal_p[(b.name, spec.name, "bar")] = bt["best_bar"]

        costs = item_costs(cache, b, model)
        if costs:
            n = len(costs)
            tin = sum(c["in"] for c in costs.values()) / n
            tout = sum(c["out"] for c in costs.values()) / n
            usd = sum(c["usd"] for c in costs.values())
            usd0 = sum(c["usd_order0"] for c in costs.values()) / n
            w(f"Cost `{b.name}`: {n} items, mean {tin:.0f} input / {tout:.0f} output tokens per item "
              f"(all orders), eval spend ${usd:.4f}; per 1,000 items: all orders ${usd / n * 1000:.4f}, "
              f"request(s) holding order 0 ${usd0 * 1000:.4f} "
              f"(price in ${b.price_in_per_mtok}/Mtok, out ${b.price_out_per_mtok}/Mtok)\n")

    if primary in by_name and fallback in by_name:
        w(f"## `{primary}` vs `{fallback}`, and the cascade `{primary}` -> `{fallback}`\n")
        common = None
        for spec in specs:
            ids = set(data[(primary, spec.name)]["ids"]) & set(data[(fallback, spec.name)]["ids"])
            common = ids if common is None else common & ids
        test_ids = sorted(i for i in common if not is_calibration(i, cal_fraction))
        if not test_ids:  # e.g. every fallback request failed (wrong key on a pilot)
            w("No test item was answered by both backends: comparison and cascade skipped.")
            return "\n".join(out) + "\n"
        handoff = np.zeros(len(test_ids), bool)
        picks = {}
        for spec in specs:
            dp, df = data[(primary, spec.name)], data[(fallback, spec.name)]
            Pc, _ = cal_p[(primary, spec.name)]
            ip = {i: k for k, i in enumerate(dp["ids"])}
            jf = {i: k for k, i in enumerate(df["ids"])}
            rp = np.array([ip[i] for i in test_ids])
            rf = np.array([jf[i] for i in test_ids])
            y = dp["y"][rp]
            conf_p, pred_p = kc.top_label(Pc[rp])
            pred_f = df["probs"][rf, 0, :].argmax(axis=1)
            below = conf_p < cal_p[(primary, spec.name, "bar")]
            handoff |= below
            picks[spec.name] = (y, pred_p, pred_f, below)
            cp, cf = (pred_p == y).astype(float), (pred_f == y).astype(float)
            d, lo, hi = kc.paired_bootstrap_diff(np.mean, [cf], [cp])
            verdict = "non-inferior" if lo > -margin else "NOT shown non-inferior"
            w(f"- {spec.name}: acc {primary} {cp.mean():.3f} vs {fallback} {cf.mean():.3f}; "
              f"difference {d:+.3f} [{lo:+.3f}, {hi:+.3f}] -> {primary} {verdict} at margin {margin:.0%}")
        k = int(handoff.sum())
        rate, (lo, hi) = k / len(test_ids), kc.wilson_interval(k, len(test_ids))
        w(f"\nCascade on {len(test_ids)} common test items (bar per question = min-cost top-prob bar "
          f"from cal). One `{fallback}` call answers every question, so an item is handed off when ANY "
          f"question is below its bar; each question keeps `{primary}`'s answer where it cleared its bar:")
        w(f"- item hand-off rate {rate:.3f} [{lo:.3f}, {hi:.3f}]")
        for name, (y, pp, pf, below) in picks.items():
            final = np.where(below, pf, pp)
            acc, alo, ahi = kc.bootstrap_ci(np.mean, (final == y).astype(float))
            w(f"- {name}: below bar {below.mean():.3f}; cascade acc {acc:.3f} [{alo:.3f}, {ahi:.3f}] "
              f"(vs {primary} alone {(pp == y).mean():.3f}, {fallback} alone {(pf == y).mean():.3f})")
        cp_ = item_costs(cache, by_name[primary], cache_model_key(by_name[primary]))
        cf_ = item_costs(cache, by_name[fallback], cache_model_key(by_name[fallback]))
        up = np.mean([c["usd_order0"] for c in cp_.values()])
        uf = np.mean([c["usd_order0"] for c in cf_.values()])
        w(f"- cost per 1,000 items: {primary} alone ${up * 1000:.4f}, {fallback} alone ${uf * 1000:.4f}, "
          f"cascade ${(up + rate * uf) * 1000:.4f} ({primary} on every item + {fallback} on hand-offs)")
    return "\n".join(out) + "\n"
```

---

## The CLI

`run_eval.py` (executed offline through `amain`, with the arguments `offline_demo.py` passes):

```python
"""CLI: python run_eval.py --corpus corpus.jsonl --llm-provider anthropic --llm-model <id> ..."""

from __future__ import annotations

import argparse
import asyncio
import sys
from pathlib import Path
from typing import Any

from typesafe_sdk import AsyncTypeSafeClient, RetryPolicy

from harness_backends import AdapterBackend, JevBackend
from harness_report import build_report
from harness_runner import ResponseCache, run_backend
from harness_spec import load_corpus
from questions import SPECS


def parse(argv: list[str] | None = None) -> argparse.Namespace:
    p = argparse.ArgumentParser()
    p.add_argument("--corpus", required=True)
    p.add_argument("--cache", default="responses.jsonl")
    p.add_argument("--report", default="report.md")
    p.add_argument("--jev-model", default="jev-1.13.0")
    p.add_argument("--llm-provider", choices=["openai", "anthropic", "gemini"])
    p.add_argument("--llm-model")
    p.add_argument("--llm-price-in", type=float, default=0.0, help="USD per million input tokens")
    p.add_argument("--llm-price-out", type=float, default=0.0, help="USD per million output tokens")
    p.add_argument("--concurrency", type=int, default=8)
    p.add_argument("--max-retries", type=int, default=4)
    p.add_argument("--margin", type=float, default=0.02, help="non-inferiority margin on accuracy")
    p.add_argument("--limit", type=int, help="only the first N corpus items (pilot run)")
    return p.parse_args(argv)


async def amain(args: argparse.Namespace, *, jev_client_kwargs: dict[str, Any] | None = None,
                llm_model: Any = None) -> str:
    corpus = load_corpus(args.corpus)[: args.limit]
    cache = ResponseCache(args.cache)
    retry = RetryPolicy(max_retries=args.max_retries, timeout=120.0)
    if args.jev_model in ("jev-latest", "jev-preview"):
        print(f"warning: {args.jev_model} is an alias; cached answers will be keyed by it, "
              "not by the version that served them", file=sys.stderr)
    backends, stats = [], []
    async with AsyncTypeSafeClient(retry=retry, **(jev_client_kwargs or {})) as jc:
        jev = JevBackend(jc, args.jev_model)
        backends.append(jev)
        stats.append(await run_backend(jev, corpus, SPECS, cache, concurrency=args.concurrency))
    if args.llm_model or llm_model is not None:
        from system_one_adapter import AsyncSystemOneAdapterClient

        async with AsyncSystemOneAdapterClient(
            structured_outputs=True,
            llm_answer_mode="probabilities",
            normalize_probabilities=True,
            n_retry_malformed_structure=2,
            retry=retry,  # the adapter's own default is RetryPolicy(max_retries=0)
        ) as lc:
            llm = AdapterBackend(lc, llm_model if llm_model is not None else args.llm_model,
                                 provider=None if llm_model is not None else args.llm_provider,
                                 price_in_per_mtok=args.llm_price_in, price_out_per_mtok=args.llm_price_out)
            backends.append(llm)
            stats.append(await run_backend(llm, corpus, SPECS, cache, concurrency=args.concurrency))
    report = build_report(corpus, SPECS, cache, backends, stats, margin=args.margin)
    Path(args.report).write_text(report, encoding="utf-8")
    return report


if __name__ == "__main__":
    print(asyncio.run(amain(parse())))
```

The adapter client is configured with `structured_outputs=True` (the provider's native structured output), `llm_answer_mode="probabilities"`, `normalize_probabilities=True` (rescale distributions that do not sum to 1; README "Options") and two corrective retries for malformed output. All four option names and meanings are from the README's options table and `_client.py`.

---

## Offline end-to-end run

The simulation, so the report can be read against a known truth:

- 800 items, 4 departments and a yes/no "urgent". Each item has calibrated evidence `z` per department, normal with mean 0 and standard deviation 2.4, and the gold department is **drawn** from `softmax(z)`, so even a perfect model cannot be right every time. Urgency: `u` normal with mean -1 and standard deviation 2.5, gold drawn with probability `expit(u)`.
- **Jev mock** (an `httpx2.MockTransport` behind the real `AsyncTypeSafeClient`): sees `z` plus noise (sd 0.6), reports `softmax(1.6 * evidence + position bias)` with +0.35 on the **first** option shown, rounded to 0.01; Noul `expit(0.6 u + 0.2)` (compressed toward 0.5), rounded to 0.01. It answers the documented response shape, including `confidence` computed with the formula from https://docs.typesafe.ai/confidence. It returns `429` with `retry-after-ms: 5` on some first attempts and `529` forever for one item. Token counts are `len(body) // 4`: **not** a real tokenizer.
- **LLM mock** (a fake `AsyncProvider` behind the real `AsyncSystemOneAdapterClient`): sees `z` with less noise (sd 0.3), reports `softmax(3.0 * evidence + bias)` with +0.25 on the **third** option, rounded to 0.05 the way a verbalised probability is; Noul `expit(2u + 0.5)` (over-confident). It returns schema-violating output on the first attempt for 10 items, which the adapter repairs with a corrective retry.
- Prices: Jev at the published \$0.042/Mtok input, \$0 output. The LLM at **\$1.00 / \$5.00 per Mtok, invented for the demo**: not any provider's price.

The run is executed twice; the second must be served from the cache.

`offline_demo.py` (executed):

```python
"""Run the whole harness offline: synthetic corpus, mock Jev HTTP transport, fake LLM provider.

Nothing here touches a network. The 'models' are simulations with KNOWN behaviour, so
the report can be checked against the truth: Jev-mock is over-confident (logit scale
1.6), favours the first option shown, compresses Noul toward 0.5, and rounds to 0.01;
the LLM-mock is sharper on the evidence, more over-confident (scale 3.0), favours the
third option, and rounds to 0.05 like a verbalised probability.
"""

from __future__ import annotations

import asyncio
import json
import sys
import zlib
from pathlib import Path

import httpx2
import numpy as np
from scipy.special import expit, softmax
from system_one_adapter.providers import Message, ProviderResult

from questions import SPECS
from run_eval import amain, parse

HERE = Path(__file__).parent
N_ITEMS = 800
DEPT = list(SPECS[0].options)  # canonical order
TEXT = {
    "billing": "I was charged twice for the March invoice.",
    "technical": "The export job fails with error 502 since this morning.",
    "sales": "What would the enterprise tier cost for 40 seats?",
    "account": "I can no longer log in after changing my email.",
}


def make_corpus(path: Path) -> dict[str, dict]:
    rng = np.random.default_rng(7)
    latents, lines = {}, []
    for i in range(N_ITEMS):
        z = rng.normal(0.0, 2.4, 4)  # calibrated evidence for the 4 departments
        dept = DEPT[rng.choice(4, p=softmax(z))]  # truth drawn from the calibrated posterior
        u = rng.normal(-1.0, 2.5)
        urgent = bool(rng.random() < expit(u))
        state = f"[t{i:04d}] {TEXT[DEPT[int(np.argmax(z))]]}"
        latents[state] = {"z": z, "u": u, "jev_noise": rng.normal(0, 0.6, 4), "llm_noise": rng.normal(0, 0.3, 4)}
        lines.append(json.dumps({"id": f"t{i:04d}", "state": state, "labels": {"department": dept, "urgent": urgent}}))
    path.write_text("\n".join(lines) + "\n", encoding="utf-8")
    return latents


def make_jev_transport(latents: dict[str, dict], counters: dict[str, int]) -> httpx2.MockTransport:
    attempts: dict[str, int] = {}

    async def handler(request: httpx2.Request) -> httpx2.Response:
        await asyncio.sleep(0.005)
        body = json.loads(request.content)
        state, item = body["state"], latents[body["state"]]
        counters["http"] += 1
        n = attempts[state] = attempts.get(state, 0) + 1
        if state.startswith("[t0013]"):  # one item the service never answers
            return httpx2.Response(529, headers={"retry-after-ms": "5"}, json={"detail": "Overloaded"})
        if n == 1 and zlib.crc32(state.encode()) % 17 == 0:  # occasional rate limit on a first attempt
            counters["429"] += 1
            return httpx2.Response(429, headers={"retry-after-ms": "5"}, json={"detail": "Too Many Requests"})
        answers = {}
        for key, q in body["questions"].items():
            if q["type"] == "noul":
                answers[key] = {"type": "noul", "noul": round(float(expit(0.6 * item["u"] + 0.2)), 2)}
                continue
            shown = list(q["criteria"])  # the order this request shows
            idx = [DEPT.index(lab) for lab in shown]
            bias = np.array([0.35, 0.0, 0.0, -0.1])  # by POSITION
            p = softmax(1.6 * (item["z"] + item["jev_noise"])[idx] + bias)
            p = np.round(p, 2)
            peak = float(p.max())
            answers[key] = {
                "type": "choice",
                "choice": shown[int(p.argmax())],
                "probabilities": {lab: float(v) for lab, v in zip(shown, p)},
                "confidence": round((4 * peak - 1) / 3, 2),  # docs.typesafe.ai/confidence formula
            }
        usage = {"input_tokens": len(request.content) // 4, "output_tokens": 10 * len(answers)}
        return httpx2.Response(200, headers={"x-typesafe-request-id": f"req-{counters['http']}"},
                               json={"model": "jev-1.13.0", "answers": answers, "usage": usage})

    return httpx2.MockTransport(handler)


class FakeLLMProvider:
    """Satisfies system_one_adapter.providers.AsyncProvider without a network."""

    model_name = "fake-llm-1"

    def __init__(self, latents: dict[str, dict], counters: dict[str, int]) -> None:
        self.latents, self.counters = latents, counters

    async def request(self, messages: list[Message], *, schema: dict, structured: bool) -> ProviderResult:
        await asyncio.sleep(0.005)
        self.counters["llm"] += 1
        doc = messages[1].content.removeprefix("<document>\n").removesuffix("\n</document>")
        state = json.loads(doc)
        item = self.latents[state]
        props = schema["$defs"]["TypeSafeAnswers"]["properties"]
        out = {}
        for key, prop in props.items():
            if "$ref" not in prop:  # Noul: a bare number
                out[key] = round(float(expit(2.0 * item["u"] + 0.5)) * 20) / 20
                continue
            shown = list(schema["$defs"][prop["$ref"].split("/")[-1]]["properties"])
            idx = [DEPT.index(lab) for lab in shown]
            bias = np.array([0.0, 0.0, 0.25, 0.0])
            p = softmax(3.0 * (item["z"] + item["llm_noise"])[idx] + bias)
            out[key] = {lab: round(float(v) * 20) / 20 for lab, v in zip(shown, p)}
        text = json.dumps({"answers": out})
        if len(messages) == 2 and state.startswith("[t00") and state[5] == "7":
            text = "```json\n" + json.dumps({"answers": {}}) + "\n```"  # malformed: forces a corrective retry
        tokens_in = sum(len(m.content) for m in messages) // 4 + len(json.dumps(schema)) // 4
        return ProviderResult(text=text, input_tokens=tokens_in, output_tokens=len(text) // 4)

    def translate_error(self, error: Exception):  # never called: request() does not raise
        raise error


def main() -> None:
    work = HERE / "demo"
    work.mkdir(exist_ok=True)
    for f in ("responses.jsonl", "report.md"):
        (work / f).unlink(missing_ok=True)
    latents = make_corpus(work / "corpus.jsonl")
    argv = ["--corpus", str(work / "corpus.jsonl"), "--cache", str(work / "responses.jsonl"),
            "--report", str(work / "report.md"), "--concurrency", "16",
            "--llm-price-in", "1.0", "--llm-price-out", "5.0"]  # ILLUSTRATIVE prices, not a quote
    reports = []
    for run in (1, 2):  # run 2 must be served from the cache
        counters = {"http": 0, "429": 0, "llm": 0}
        kwargs = {"api_key": "offline", "transport": make_jev_transport(latents, counters)}
        reports.append(asyncio.run(amain(parse(argv), jev_client_kwargs=kwargs, llm_model=FakeLLMProvider(latents, counters))))
        print(f"== run {run}: HTTP attempts to Jev mock {counters['http']} (429s {counters['429']}), "
              f"LLM provider calls {counters['llm']}", file=sys.stderr)
    print(reports[0])
    print("== run 2 header ==")
    print("\n".join(reports[1].splitlines()[2:6]))


if __name__ == "__main__":
    main()
```

Console output of the two runs (stderr):

```text
== run 1: HTTP attempts to Jev mock 848 (429s 44), LLM provider calls 3240
== run 2: HTTP attempts to Jev mock 5 (429s 0), LLM provider calls 0
```

Run 1 sent 800 Jev requests (one per item, all rotations inside) and the transport saw 848 attempts: 800, plus 44 retried 429s, plus 4 retries of the item that always failed. The adapter made 3,240 provider calls: 800 items x 4 rotation requests, plus 40 corrective retries (10 malformed items x 4 requests). Run 2 sent one request, for the failed item only, and made zero LLM calls.

---

## The report it produced

Verbatim `report.md` of run 1 (on 2026-10-03; the wall-clock seconds in the header change from run to run, everything else is deterministic):

```text
# Decision-model evaluation report

Corpus: 800 items. Split: hashed id, cal fraction 0.5.
Run jev: 800 requests sent (799 ok, 1 failed after retries), 0.79 s.
Run llm: 3200 requests sent (3200 ok, 0 failed after retries), 8.67 s.
Cache: 3999 responses, 1 error records.

## Backend `jev` (requested `jev-1.13.0`)

### department (choice, 4 labels)

- served model(s): jev-1.13.0; items: 799 (cal 418, test 381); missing/failed: 1
- raw, test:        acc 0.701 [0.654, 0.745] | ECE 0.123 (noise floor 0.037, 95th 0.056: ABOVE) | Brier 0.439 | NLL 0.803
- temperature T = 1.56 on cal; test: acc 0.701 [0.654, 0.745] | ECE 0.064 (noise floor 0.052, 95th 0.076: inside) | Brier 0.411 | NLL 0.739
- option order: 4 cyclic rotations; flip rate 0.098 [0.079, 0.120] (all items); test acc order-0 0.701 vs rotation-averaged 0.688

Risk-coverage, calibrated top probability, test split:

| bar asked | bar used | coverage | error kept | n kept |
|---|---|---|---|---|
| 0.50 | 0.502 | 0.874 | 0.270 | 333 |
| 0.60 | 0.601 | 0.701 | 0.217 | 267 |
| 0.70 | 0.701 | 0.564 | 0.163 | 215 |
| 0.80 | 0.801 | 0.446 | 0.129 | 170 |
| 0.90 | 0.908 | 0.150 | 0.070 | 57 |

Policies (bars chosen on cal, measured on test; hand-off cost 0.4):

| policy | coverage | error kept | cost / item |
|---|---|---|---|
| automate all | 1.000 | 0.299 | 0.5197 |
| hand off all | 0.000 | nan | 0.4000 |
| top-prob bar 0.82 (min cost on cal) | 0.386 | 0.116 | 0.3270 |
| top-prob bar 0.86 (error <= 5% on cal) | 0.270 | 0.068 | 0.3207 |
| Bayes: act iff expected cost <= hand-off | 0.572 | 0.188 | 0.2945 |

### urgent (noul, 2 labels)

- served model(s): jev-1.13.0; items: 799 (cal 418, test 381); missing/failed: 1
- raw, test:        acc 0.801 [0.761, 0.840] | ECE 0.076 (noise floor 0.052, 95th 0.076: ABOVE) | Brier 0.273 | NLL 0.427
- Platt a = 1.36, b = -0.18 on cal; test: acc 0.801 [0.759, 0.840] | ECE 0.060 (noise floor 0.046, 95th 0.068: inside) | Brier 0.262 | NLL 0.401

Risk-coverage, calibrated top probability, test split:

| bar asked | bar used | coverage | error kept | n kept |
|---|---|---|---|---|
| 0.50 | 0.504 | 1.000 | 0.199 | 381 |
| 0.60 | 0.605 | 0.843 | 0.134 | 321 |
| 0.70 | 0.711 | 0.696 | 0.113 | 265 |
| 0.80 | 0.801 | 0.541 | 0.063 | 206 |
| 0.90 | 0.903 | 0.307 | 0.009 | 117 |
| 0.95 | 0.951 | 0.165 | 0.000 | 63 |
| 0.99 | 0.993 | 0.010 | 0.000 | 4 |

Policies (bars chosen on cal, measured on test; hand-off cost 0.4):

| policy | coverage | error kept | cost / item |
|---|---|---|---|
| automate all | 1.000 | 0.199 | 1.2388 |
| hand off all | 0.000 | nan | 0.4000 |
| top-prob bar 0.94 (min cost on cal) | 0.186 | 0.000 | 0.3255 |
| top-prob bar 0.93 (error <= 5% on cal) | 0.234 | 0.000 | 0.3066 |
| Bayes: act iff expected cost <= hand-off | 0.391 | 0.107 | 0.2856 |

Cost `jev`: 799 items, mean 376 input / 50 output tokens per item (all orders), eval spend $0.0126; per 1,000 items: all orders $0.0158, request(s) holding order 0 $0.0158 (price in $0.042/Mtok, out $0.0/Mtok)

## Backend `llm` (requested `fake-llm-1`)

### department (choice, 4 labels)

- served model(s): fake-llm-1; items: 800 (cal 419, test 381); missing/failed: 0
- raw, test:        acc 0.685 [0.640, 0.730] | ECE 0.226 (noise floor 0.020, 95th 0.035: ABOVE) | Brier 0.502 | NLL 1.037
- temperature T = 1.86 on cal; test: acc 0.685 [0.640, 0.730] | ECE 0.094 (noise floor 0.052, 95th 0.075: ABOVE) | Brier 0.427 | NLL 0.797
- option order: 4 cyclic rotations; flip rate 0.037 [0.026, 0.053] (all items); test acc order-0 0.685 vs rotation-averaged 0.680

Risk-coverage, calibrated top probability, test split:

| bar asked | bar used | coverage | error kept | n kept |
|---|---|---|---|---|
| 0.50 | 0.500 | 0.919 | 0.277 | 350 |
| 0.60 | 0.604 | 0.837 | 0.260 | 319 |
| 0.70 | 0.751 | 0.685 | 0.195 | 261 |
| 0.80 | 0.851 | 0.559 | 0.136 | 213 |

Policies (bars chosen on cal, measured on test; hand-off cost 0.4):

| policy | coverage | error kept | cost / item |
|---|---|---|---|
| automate all | 1.000 | 0.315 | 0.5459 |
| hand off all | 0.000 | nan | 0.4000 |
| top-prob bar 0.85 (min cost on cal) | 0.559 | 0.136 | 0.3312 |
| Bayes: act iff expected cost <= hand-off | 0.625 | 0.231 | 0.3312 |

### urgent (noul, 2 labels)

- served model(s): fake-llm-1; items: 800 (cal 419, test 381); missing/failed: 0
- raw, test:        acc 0.798 [0.758, 0.837] | ECE 0.104 (noise floor 0.023, 95th 0.039: ABOVE) | Brier 0.294 | NLL 0.487
- Platt a = 0.45, b = -0.10 on cal; test: acc 0.798 [0.756, 0.837] | ECE 0.065 (noise floor 0.047, 95th 0.068: inside) | Brier 0.265 | NLL 0.412

Risk-coverage, calibrated top probability, test split:

| bar asked | bar used | coverage | error kept | n kept |
|---|---|---|---|---|
| 0.50 | 0.503 | 1.000 | 0.202 | 381 |
| 0.60 | 0.619 | 0.843 | 0.134 | 321 |
| 0.70 | 0.708 | 0.766 | 0.127 | 292 |
| 0.80 | 0.808 | 0.577 | 0.073 | 220 |
| 0.90 | 0.909 | 0.488 | 0.038 | 186 |

Policies (bars chosen on cal, measured on test; hand-off cost 0.4):

| policy | coverage | error kept | cost / item |
|---|---|---|---|
| automate all | 1.000 | 0.202 | 1.2651 |
| hand off all | 0.000 | nan | 0.4000 |
| top-prob bar inf (min cost on cal) | 0.000 | nan | 0.4000 |
| Bayes: act iff expected cost <= hand-off | 0.299 | 0.140 | 0.3223 |

Cost `llm`: 800 items, mean 1906 input / 100 output tokens per item (all orders), eval spend $1.9258; per 1,000 items: all orders $2.4073, request(s) holding order 0 $0.6650 (price in $1.0/Mtok, out $5.0/Mtok)

## `jev` vs `llm`, and the cascade `jev` -> `llm`

- department: acc jev 0.701 vs llm 0.685; difference +0.016 [-0.016, +0.047] -> jev non-inferior at margin 2%
- urgent: acc jev 0.801 vs llm 0.798; difference +0.003 [-0.010, +0.016] -> jev non-inferior at margin 2%

Cascade on 381 common test items (bar per question = min-cost top-prob bar from cal). One `llm` call answers every question, so an item is handed off when ANY question is below its bar; each question keeps `jev`'s answer where it cleared its bar:
- item hand-off rate 0.929 [0.899, 0.951]
- department: below bar 0.614; cascade acc 0.682 [0.635, 0.727] (vs jev alone 0.701, llm alone 0.685)
- urgent: below bar 0.814; cascade acc 0.798 [0.758, 0.837] (vs jev alone 0.801, llm alone 0.798)
- cost per 1,000 items: jev alone $0.0158, llm alone $0.6650, cascade $0.6337 (jev on every item + llm on hand-offs)
```

The header of run 2, from the cache:

```text
Corpus: 800 items. Split: hashed id, cal fraction 0.5.
Run jev: 1 requests sent (0 ok, 1 failed after retries), 0.14 s.
Run llm: 0 requests sent (0 ok, 0 failed after retries), 0.0 s.
Cache: 3999 responses, 2 error records.
```

---

## Reading the report: what the simulation shows

All numbers in this section are from the simulated run above.

**Raw calibration fails, one parameter fixes most of it.** Every raw ECE is above its noise floor. After the cal-split fit, three of four questions land inside the floor's 95th percentile on test. The fitted parameters point the right way: Jev's department temperature T = 1.56 (the mock is over-confident by a logit factor of 1.6), Jev's urgent Platt slope 1.36 (the mock compresses toward 0.5, so the fit stretches), and the LLM's urgent Platt slope 0.45 (over-confident, so the fit compresses). The LLM's department ECE stays above the floor after one temperature (0.094 vs 0.075). On real data, that is the signal to try a richer map (`calibration.md`, "Recalibration maps") or to stop trusting that model's probabilities for gating.

**What the temperatures lose to rounding.** The fitted temperatures fall short of the mocks' true scales, which are 1.6 for Jev and 3.0 for the LLM. The temperature is fitted on `log(clip(p, 0.005))`, so a probability that was rounded to 0, or that sits below the 0.005 floor, carries no logit information. A sharp model has many such probabilities. Side checks run during verification on the same seeds (harness unchanged except for the noted edits; not shown):

| Mock change | LLM department after the fit | Jev department after the fit |
|---|---|---|
| none (the run above) | T = 1.86, ECE 0.094, ABOVE | T = 1.56, ECE 0.064, inside |
| unrounded, still with the 0.005 floor | T = 1.82, ECE 0.105, ABOVE | - |
| no position bias | T = 1.85, ECE 0.094, ABOVE | - |
| unrounded, with a `floor=1e-9` passed to the fit | T = 2.97, ECE 0.063, inside | T = 1.83, ECE 0.051, inside |

The residual LLM miscalibration therefore does not come from the 0.05 rounding alone, and it does not come from the position bias. It comes from the lost tail: everything below the floor. For Jev the noise-free optimum lies above 1.6, because the mock's evidence is noisy, and the floor pulls the fit below it. Only providers that return unrounded probabilities can use a smaller floor. Rounded output cannot, because its exact zeros would become huge negative logits (`calibration.md`, "The rounding floor"). A larger floor did not help either: 0.025, which is half the LLM's 0.05 quantum, gave T = 1.36 and ECE 0.115. For such a model, accept the residual or move to a map with more parameters.

**The noise floor is not a constant.** The raw LLM department floor is 0.020, the calibrated one 0.052: the floor depends on the confidences themselves, so it must be recomputed for every set of probabilities you judge, which `metrics()` does.

**Option order.** The flip rate catches the injected position bias: 0.098 [0.079, 0.120] for Jev-mock against 0.037 [0.026, 0.053] for the sharper LLM-mock. Averaging the four rotations did **not** raise accuracy here (0.688 vs 0.701 for Jev-mock); flip rate tells you answers are unstable, not that averaging will make them more accurate. Measure both before paying for rotations.

**Asymmetric costs change the bar, and the rule.** For `urgent`, missing a real outage costs 10 and a false page 1. Both top-probability bars end up high (0.93-0.94) and automate under a quarter of the items. The Bayes rule automates 39% at a lower cost per item (0.2856 vs 0.3066 and 0.3255), because its gate is asymmetric: it acts "yes" from p = 0.6 but "no" only at or below p = 0.04 (the arithmetic is in the previous section), whereas a top-probability bar demands the same confidence for both answers. On the LLM-mock's `urgent`, the min-cost top-probability bar is `inf`: no bar beat handing everything off on cal, while the Bayes rule still found a cheaper policy (0.3223). If your costs are asymmetric, gate on expected cost, not on `p_max`.

**Accuracy alone would have said "equal".** Jev-mock is non-inferior on both questions at a 2-point margin (department +0.016 [-0.016, +0.047]). The CIs are wide at 381 test items, which is the point of printing them: see [sample size](#sample-size).

**This cascade does not pay.** 92.9% of items are handed off and the cascade's accuracy (0.682 / 0.798) is no better than either model alone, at almost the LLM-only price. Two reasons, both general:

- Hand-offs compound across questions: the item goes to the fallback if **any** question is below its bar, and the `urgent` bar alone sends 81% of items.
- A cascade only pays when the fallback is **better on the items the primary is unsure about**. Here both mocks see the same evidence, so the hard items are hard for both. In the spec, the hand-off cost (0.4) is one flat figure for "the fallback" and leaves out the fallback's own mistakes. When the fallback is an LLM, set `handoff_cost` to the price of an LLM call plus the expected cost of that LLM's errors. In this run that error term alone is about 0.55 per department item: the LLM's "automate all" cost. That raises the hand-off cost and lowers the bar.

**Cost.** Jev-mock's \$0.0158 per 1,000 items comes from the mock's `len(body) // 4` token count at the real Jev price; the LLM figures use invented prices. Neither is an estimate of a real bill; see the next section for how to get one.

---

## Running it for real

### 1. Install and authenticate

Install commands verbatim from the adapter README (https://github.com/typesafe-ai/system-one-adapter-python, commit `e1d4cc9`):

```bash
pip install 'system-one-adapter[openai]'      # OpenAI-compatible providers
pip install 'system-one-adapter[anthropic]'   # native Anthropic
pip install 'system-one-adapter[gemini]'      # native Gemini
```

The adapter 0.2.1 depends on `typesafe-sdk>=0.7.0` (its `pyproject.toml`), so this also installs the Jev SDK. The harness additionally needs `numpy`, `scipy` and `scikit-learn` (for `kb_calib`). Pin what you verified, for example `typesafe-sdk==0.7.2 system-one-adapter==0.2.1`.

Credentials, from the sources named: Jev reads `TYPESAFE_API_KEY` (`typesafe_sdk.constants`). The adapter's OpenAI provider "Defaults to the SDK's own resolution, including `OPENAI_API_KEY`" and the Gemini provider to "`GEMINI_API_KEY` and `GOOGLE_API_KEY`" (provider docstrings, 0.2.1); the Anthropic provider constructs `anthropic.AsyncAnthropic(max_retries=0)` with no key argument, so it uses the Anthropic SDK's own resolution of `ANTHROPIC_API_KEY` (see `../anthropic/basics.md`). For larger Anthropic evaluations, the README says to raise the provider's output limit (default 4,096 tokens) by passing `AnthropicProvider(..., max_tokens=...)` as the model. It adds that "`AsyncAnthropicProvider` accepts the same option". The harness uses the async client, so it needs `AsyncAnthropicProvider`. A sync provider passed to `AsyncSystemOneAdapterClient` fails with `TypeError: object ProviderResult can't be used in 'await' expression` (checked offline with a fake sync provider). That error is not a `TypeSafeError`, so it stops the run instead of producing error records. The CLI has no flag for a provider instance. Pass the instance through `amain(args, llm_model=AsyncAnthropicProvider("<model-id>", max_tokens=...))`, the same way `offline_demo.py` passes its fake provider.

### 2. Run a pilot, then the full corpus

Not executed (no API key on the verification machine); the same argument parser and `amain` were executed offline with the demo's arguments:

```bash
export TYPESAFE_API_KEY=...        # and the LLM provider's key
python run_eval.py --corpus corpus.jsonl --limit 50 \
  --jev-model jev-1.13.0 --llm-provider anthropic --llm-model <model-id> \
  --llm-price-in <USD per Mtok> --llm-price-out <USD per Mtok>
# inspect report.md and responses.jsonl, then the full run reuses those 50:
python run_eval.py --corpus corpus.jsonl \
  --jev-model jev-1.13.0 --llm-provider anthropic --llm-model <model-id> \
  --llm-price-in <USD per Mtok> --llm-price-out <USD per Mtok>
```

The pilot checks the three things that waste money on a full run: the questions are what you meant (read 10 answers in `responses.jsonl`), the token counts per item are what you budgeted, and the LLM's output parses (look for `n_retries_malformed_structure` > 0 in the LLM records). Keep `responses.jsonl`: it is the evidence behind the decision, and the baseline for the next model version.

### Sample size

The report's CIs are honest, so a small corpus produces a report that cannot decide. Half-widths for the two quantities that matter (`sample_size.py`, executed):

```python
"""How many labelled TEST items a decision needs (normal approximations)."""
import math

import kb_calib as kc

print("accuracy 95% CI half-width (Wilson), by test n and true accuracy")
for n in (100, 200, 400, 800, 1600):
    row = []
    for acc in (0.80, 0.90, 0.95):
        lo, hi = kc.wilson_interval(round(acc * n), n)
        row.append(f"{(hi - lo) / 2:.3f}")
    print(n, *row)

print("paired accuracy-difference 95% half-width ~ 1.96*sqrt(d/n), d = share of items where exactly one model is right")
for n in (100, 200, 400, 800, 1600):
    print(n, *(f"{1.96 * math.sqrt(d / n):.3f}" for d in (0.05, 0.10, 0.20)))

print("test n for the paired half-width to fall under a 2-point margin")
for d in (0.05, 0.10, 0.20):
    print(d, math.ceil(d * (1.96 / 0.02) ** 2))
```

Output:

```text
accuracy 95% CI half-width (Wilson), by test n and true accuracy
100 0.078 0.060 0.045
200 0.055 0.042 0.031
400 0.039 0.030 0.022
800 0.028 0.021 0.015
1600 0.020 0.015 0.011
paired accuracy-difference 95% half-width ~ 1.96*sqrt(d/n), d = share of items where exactly one model is right
100 0.044 0.062 0.088
200 0.031 0.044 0.062
400 0.022 0.031 0.044
800 0.015 0.022 0.031
1600 0.011 0.015 0.022
test n for the paired half-width to fall under a 2-point margin
0.05 481
0.1 961
0.2 1921
```

Read it as: a **test** split of 400 items pins accuracy to about +-3-4 points; showing non-inferiority within 2 points needs roughly 500-1,900 test items depending on how often the two models disagree in correctness (`d`, which the run measures; a normal approximation, `1.96 * sqrt(d / n)`, valid when the accuracy difference is small). With the default 50/50 split, double those numbers for the corpus. The fits need far less: about 50 labelled items per question for a temperature or Platt fit, fewer than about 30 can make calibration worse, and isotonic needs about 1,000 (`calibration.md`, "Sample-size guidance", and the `decision-model-calibration` skill). Rare, expensive classes need enough examples **of that class** for their bar.

### Cost estimate before you run

Jev bills input tokens only, at \$0.042 per million for `jev-1.13.0` (https://docs.typesafe.ai/models, 2026-10-03). Per item, one request carries the state once plus every question in every order. Worked example, arithmetic only: 1,600 items, 500-token states, a 4-option Choice of about 60 tokens asked in 4 orders and a Noul of about 60 tokens is about 500 + 4 x 60 + 60 = 800 tokens per item, 1.28 M tokens, **about \$0.05** for the whole evaluation. Measure the real count on the pilot: the `usage.input_tokens` in each cached Jev record is what was billed.

The LLM is the expensive side. Per item it costs `rotations x (input_tokens x price_in + output_tokens x price_out) / 1e6`, where the input includes the adapter's system prompt and the JSON Schema of all questions on every request, and the output is a JSON object with one probability per label. Take the per-request token counts from the pilot's LLM records (`usage.input_tokens` / `output_tokens`, which are the `*_total` counts including retries) and your provider's current price page. To cut the LLM bill, ask fewer orders. The code as shown cannot do that for the LLM alone. `run_eval.py` passes one `SPECS` list to every backend, and `collect()` counts an item as missing unless every order of the spec is in the cache. An LLM-only `n_rotations=1` therefore needs per-backend spec lists passed to both `run_backend` and `build_report`, which is not implemented here. The route that works unchanged is `n_rotations=1` for both backends on the full corpus, plus a separate audit run with the default `n_rotations` on a subset (`--limit`, with its own `--report`).

Context limits for the Jev side: "64k tokens per request; 32k tokens for `state` plus the longest question" (https://docs.typesafe.ai/models, 2026-10-03). Rotations add questions, not state, so they consume the 64k budget, not the 32k one.

### Rate limits

`jev-1.13.0`: "100K tokens per second / 80 requests per second", and "the limits above can change without notice" (same page). With `--concurrency 8`, the run goes over 80 requests per second whenever latency is under 100 ms (Little's law, runner section). The CLI's `RetryPolicy(max_retries=4)` honours `retry-after` and absorbs occasional 429s, and an item that still fails is asked again on the next run. Whether that is enough for a given corpus size was not measured. For a sustained run, add the rate limiter mentioned in the runner section. The LLM provider has its own limits, and the adapter only retries if you pass a `RetryPolicy` (the CLI does).

---

## Limits and what was not verified

- **No real API was called.** The Jev side is the real SDK over a mock transport, the LLM side the real adapter client over a fake provider. The adapter's built-in OpenAI, Anthropic and Gemini providers were read in source but not executed (their SDKs are not installed here and there was no key).
- **Token counts in the run are fake** (`len // 4`); the published Jev price is real. The LLM prices are invented.
- **The simulated models are not Jev or any LLM.** They were built to have the faults the harness should catch; the report proves the harness catches them, not that any product has them.
- **Score questions** are not handled.
- **Verbalised LLM probabilities vs logprobs.** The adapter asks the LLM to write probabilities. An LLM classifier you run today with logprobs (`provider-logprobs.md`) may calibrate differently from the same model through the adapter, so the baseline measured here is "the LLM via the adapter", not necessarily your current production classifier. If you have that classifier's outputs, write them into the cache format under their own backend name and the report will compare them.
- **The bar search** uses every observed confidence on cal as a candidate, which slightly over-fits the cal split at small n; the test-split numbers in the policy table are the honest ones.
- **Rotations under the adapter** are sent one per request; whether a given LLM shows order effects when two orders share a prompt was not measured.
- **The flip rate is not a pure order effect.** It counts every item whose pick differs across orders. Jev's probabilities "move by a few hundredths between identical calls" (`decision-model-calibration` skill, section 5), so a near-tie can flip with no order effect at all. The harness has no same-order control, so it cannot separate the two. The simulated mocks are deterministic.
- **Only `TypeSafeError` is caught per item.** Any other exception raised inside `backend.call` stops `asyncio.gather` and ends the run. One example is the `TypeError` from passing a sync provider to the async adapter client. The cache keeps every request that finished, so a re-run resumes.

---

## Appendix: kb_calib functions used (verbatim)

The harness imports `kb_calib.py` unchanged (SHA-256 `d4059ca70132084c9c2eb499b264b0a5eeb12428ca51de7963a96bd226919461`). These are the functions it calls, extracted with `inspect.getsource` from that file, preceded by the module's imports and constant:

```python
from __future__ import annotations

import math
from collections import Counter

import numpy as np
from scipy import stats
from scipy.optimize import minimize, minimize_scalar
from scipy.special import expit, log_softmax, logit, softmax

# Jev rounds probabilities to 0.01 and returns exact zeros. log(0) is -inf, and a
# tiny floor (1e-6) turns every zero into a huge negative logit that inflates the
# fitted temperature. Half the rounding quantum is the honest floor.
FLOOR = 0.005


def rotations(criteria, limit=None):
    """Cyclic rotations of a Choice's options (Zheng et al., ICLR 2024): n orders, not n!."""
    items = list(criteria.items())
    n = len(items)
    count = n if limit is None else min(limit, n)
    shifts = sorted({round(j * n / count) % n for j in range(count)})  # evenly spread, first order kept
    return [dict(items[i:] + items[:i]) for i in shifts]


def top_label(probs):
    """(confidence, predicted index) for each row of an (n, k) matrix."""
    probs = np.asarray(probs, float)
    return probs.max(axis=1), probs.argmax(axis=1)


def wilson_interval(k, n, z=1.959964):
    """Wilson score interval for a binomial proportion k/n."""
    if n == 0:
        return (0.0, 1.0)
    phat = k / n
    denom = 1 + z * z / n
    centre = (phat + z * z / (2 * n)) / denom
    half = z * math.sqrt(phat * (1 - phat) / n + z * z / (4 * n * n)) / denom
    return (max(0.0, centre - half), min(1.0, centre + half))


def bootstrap_ci(statistic, *arrays, n_boot=2000, alpha=0.05, seed=0):
    """Percentile bootstrap CI of statistic(*arrays) resampling rows (items) jointly."""
    rng = np.random.default_rng(seed)
    arrays = [np.asarray(a) for a in arrays]
    n = len(arrays[0])
    vals = []
    for _ in range(n_boot):
        idx = rng.integers(0, n, n)
        vals.append(statistic(*[a[idx] for a in arrays]))
    lo, hi = np.percentile(vals, [100 * alpha / 2, 100 * (1 - alpha / 2)])
    return float(statistic(*arrays)), float(lo), float(hi)


def paired_bootstrap_diff(statistic, a_arrays, b_arrays, n_boot=2000, alpha=0.05, seed=0):
    """CI of statistic(b) - statistic(a) on the SAME items (e.g. ECE before/after a fit)."""
    rng = np.random.default_rng(seed)
    a_arrays = [np.asarray(x) for x in a_arrays]
    b_arrays = [np.asarray(x) for x in b_arrays]
    n = len(a_arrays[0])
    diffs = []
    for _ in range(n_boot):
        idx = rng.integers(0, n, n)
        diffs.append(statistic(*[x[idx] for x in b_arrays]) - statistic(*[x[idx] for x in a_arrays]))
    lo, hi = np.percentile(diffs, [100 * alpha / 2, 100 * (1 - alpha / 2)])
    point = statistic(*b_arrays) - statistic(*a_arrays)
    return float(point), float(lo), float(hi)


def ece(confidence, correct, n_bins=10):
    """Expected calibration error with equal-mass bins (less biased than equal-width)."""
    confidence = np.asarray(confidence, float)
    correct = np.asarray(correct, float)
    order = np.argsort(confidence, kind="stable")
    total = len(confidence)
    return sum(
        len(b) / total * abs(correct[b].mean() - confidence[b].mean())
        for b in np.array_split(order, n_bins)
        if len(b)
    )


def ece_noise_floor(confidence, n_bins=10, n_sims=2000, seed=0, estimator=None):
    """ECE a *perfectly calibrated* model would show on this many items with these
    confidences. A measured ECE inside this band is indistinguishable from noise.
    Returns the (median, 95th percentile)."""
    estimator = estimator or ece
    rng = np.random.default_rng(seed)
    confidence = np.asarray(confidence, float)
    sims = [estimator(confidence, rng.random(confidence.size) < confidence, n_bins) for _ in range(n_sims)]
    return np.percentile(sims, [50, 95])


def brier(probs, labels):
    """Multiclass Brier score; probs is (n, k), labels are column indices. Range [0, 2]."""
    probs = np.asarray(probs, float)
    onehot = np.eye(probs.shape[1])[labels]
    return np.mean(np.sum((probs - onehot) ** 2, axis=1))


def nll(probs, labels, floor=FLOOR):
    """Mean negative log-likelihood of the true class (log loss), with a floor for zeros."""
    probs = np.asarray(probs, float)
    p = np.clip(probs[np.arange(len(labels)), labels], floor, 1.0)
    return float(-np.mean(np.log(p)))


def fit_temperature(probs, labels, floor=FLOOR):
    logits = np.log(np.clip(probs, floor, 1.0))
    labels = np.asarray(labels)

    def nll_t(t):
        p = softmax(logits / t, axis=1)
        return -np.mean(np.log(p[np.arange(len(labels)), labels]))

    return minimize_scalar(nll_t, bounds=(0.05, 20.0), method="bounded").x


def apply_temperature(probs, t, floor=FLOOR):
    return softmax(np.log(np.clip(probs, floor, 1.0)) / t, axis=1)


def fit_platt(p_yes, y, floor=FLOOR):
    from sklearn.linear_model import LogisticRegression

    x = logit(np.clip(np.asarray(p_yes, float), floor, 1 - floor)).reshape(-1, 1)
    return LogisticRegression(C=1e6).fit(x, y)  # effectively unregularised: two parameters


def apply_platt(model, p_yes, floor=FLOOR):
    x = logit(np.clip(np.asarray(p_yes, float), floor, 1 - floor)).reshape(-1, 1)
    return model.predict_proba(x)[:, 1]


def risk_coverage(confidence, correct):
    """For each candidate bar: the share of items kept and the error rate among them."""
    confidence = np.asarray(confidence, float)
    correct = np.asarray(correct, bool)
    rows = []
    for bar in np.unique(confidence):
        kept = confidence >= bar
        rows.append((bar, kept.mean(), 1 - correct[kept].mean()))
    return rows


def lowest_bar_for(target_error, confidence, correct, min_kept=30):
    """Lowest bar whose kept items err at most target_error, with at least min_kept items."""
    for bar, coverage, error in risk_coverage(confidence, correct):
        if error <= target_error and coverage * len(confidence) >= min_kept:
            return bar, coverage, error
    return None


def flip_rate(winners_by_order):
    """winners_by_order: (n_items, n_orders) array of the option each order picked.
    Share of items whose pick is not the same under every order, with a Wilson CI."""
    w = np.asarray(winners_by_order)
    flipped = ~np.all(w == w[:, :1], axis=1)
    k, n = int(flipped.sum()), len(flipped)
    return k / n, wilson_interval(k, n)
```

---

## Sources

- TypeSafe API reference (request/response shapes, errors 401/422/429/529): https://docs.typesafe.ai/api (fetched 2026-10-03)
- TypeSafe models page (jev-1.13.0 price, rate limits, context, aliases, pinning advice): https://docs.typesafe.ai/models (fetched 2026-10-03)
- TypeSafe Confidence page (the `confidence` formula the mock reproduces): https://docs.typesafe.ai/confidence
- typesafe-sdk 0.7.2 installed package source: `AsyncTypeSafeClient`, `RetryPolicy` (`_core/retry.py`), constants, `_core/schemas/base.py`, `_core/response_types.py`
- system-one-adapter README, `pyproject.toml`, `_client.py`, `providers/*.py`, changelog: https://github.com/typesafe-ai/system-one-adapter-python (commit `e1d4cc938204b22fc5a3c3aca7044072fe3f712d`, 2026-09-22); PyPI `system-one-adapter` 0.2.1
- OpenAPI spec of the TypeSafe API (`SystemOneRequest`), version 0.2.0
- Sibling pages: `calibration.md`, `order-bias.md`, `conformal.md`, `provider-logprobs.md`, `../typesafe-jev/python-sdk.md`
