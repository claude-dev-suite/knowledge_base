# TypeSafe Jev in DSPy (`dspy.experimental` decision types, `TypeSafe`, `ReAnchor`)

> Official Documentation: https://dspy.ai/current/tutorials/jev_decisions/
> DSPy Repository: https://github.com/stanfordnlp/dspy
> TypeSafe documentation: https://docs.typesafe.ai/
> Last verified: 2026-10-03

**Versions verified against (2026-10-03):**

| Package | Version | Notes |
|---|---|---|
| `dspy` | `3.4.0` | uploaded to PyPI 2026-09-25 (PyPI JSON API) |
| `typesafe-sdk` | `0.7.2` | pulled in by the `typesafe` extra: `typesafe-sdk>=0.6.0,<1.0.0` (dspy 3.4.0 `METADATA`) |
| Python | `3.12.10` | |

Every "executed" example below was run offline: the TypeSafe backend through an `httpx2.MockTransport`
patched underneath `typesafe_sdk`, generative LMs through `dspy.utils.dummies.DummyLM`. No API key was
used and the real API was never called; the probabilities in the outputs are the mock's, chosen to make the
mechanics visible, not real Jev answers. The tutorial's `dspy.LM("openai/gpt-4o-mini")` examples are
copied verbatim and were not run against OpenAI.

## Overview

DSPy 3.4.0 ships three **experimental decision types** — `Noul` (probability-backed boolean), `Score`
(ordered rubric, continuous value) and `Choice` (one of N typed options) — plus a System One client,
`TypeSafe`, and an optimizer, `ReAnchor`. All five are imported from `dspy.experimental`.

The design idea that the tutorial states only briefly: **the backend supplies probability evidence, and DSPy
derives the final value locally.** Whether the evidence comes from TypeSafe's Jev (native probabilities) or
from a generative LM (asked to emit probabilities as JSON), the same per-predictor parameters turn it into a
decision:

| Type | Evidence used | Local decision rule | Parameter (`predict.fields[name]`) | Default |
|---|---|---|---|---|
| `Noul` | `noul` = P(True) | `value = p >= threshold` | `threshold` in [0, 1] | `0.5` |
| `Score` | `probabilities` over level indices | `value` = probability-weighted mean index; `level` = number of `cuts` ≤ value | `cuts`, N−1 increasing boundaries strictly inside (0, N−1) | `[0.5, 1.5, ..., N−1.5]` |
| `Choice` | `probabilities` per label | `value` = arg-max of `probability × weight` | `weights`, label → finite non-negative multiplier | `1.0` each |

`ReAnchor` fits those parameters against your metric on a labelled set. It does **not** change the
probabilities, so it fixes decision boundaries, not calibration: measure calibration separately (see the
`decision-calibration` KB documents).

This document covers what the tutorial does not: the exact request DSPy sends to TypeSafe, which API
fields DSPy keeps and which it discards, how `confidence` is computed per type (it differs), the
validation rules, the `TypeSafe` client's caching/timeout/serialization behaviour, the ReAnchor algorithm
and report, and how to run all of it offline.

---

## Table of Contents

1. [Installation and Import Paths](#installation-and-import-paths)
2. [Declaring Decision Types](#declaring-decision-types)
3. [Signatures and Results](#signatures-and-results)
4. [Two Backends, One Program](#two-backends-one-program)
5. [The `TypeSafe` Client](#the-typesafe-client)
6. [Decision Parameters: `fields`](#decision-parameters-fields)
7. [Criteria: `set_criteria` / `get_criteria`](#criteria-set_criteria--get_criteria)
8. [Native Annotations](#native-annotations)
9. [ReAnchor](#reanchor)
10. [Save and Load](#save-and-load)
11. [Errors and Validation](#errors-and-validation)
12. [Testing Offline](#testing-offline)
13. [Gotchas Checklist](#gotchas-checklist)

---

## Installation and Import Paths

Verbatim from https://dspy.ai/current/tutorials/jev_decisions/:

```bash
pip install dspy
```

```bash
pip install "dspy[typesafe]"
```

The extra only adds `typesafe-sdk`. Decision types and `ReAnchor` work without it; `TypeSafe` raises
`ImportError: Install TypeSafe support with `pip install "dspy[typesafe]"`.` at the **first call**, not at
construction (the SDK is imported lazily inside `__call__`/`acall`).

Import paths confirmed against the installed package (`dspy/experimental/__init__.py`):

| Public name | Defined in |
|---|---|
| `dspy.experimental.Noul`, `Score`, `Choice` | `dspy.adapters.types.decision` |
| `dspy.experimental.TypeSafe` | `dspy.clients.typesafe` |
| `dspy.experimental.ReAnchor` | `dspy.teleprompt.reanchor` (also importable from there) |
| (also re-exported) `Citations`, `Document` | unrelated to Jev |

All decision classes, `TypeSafe` and `ReAnchor` carry DSPy's `@experimental` marker. The tutorial's
warning applies: "Noul, Score, Choice, TypeSafe, and ReAnchor are experimental and may change without
warning."

---

## Declaring Decision Types

Verbatim from the tutorial:

```python
import dspy
import os
from dspy.experimental import Noul, Score, Choice

os.environ["OPENAI_API_KEY"] = "{your_openai_api_key}"

# A boolean decision: "Is this urgent?"
Urgent = Noul[
    (True,  "The customer cannot use the product at all"),
    (False, "The customer has a workaround or minor inconvenience"),
]

# A scored rubric: rate from 0 (low) to 2 (high), labels ordered lowest to highest
Severity = Score["Cosmetic issue", "Degrades workflow", "Blocks critical path"]

# A categorical choice: each tuple is (value, description)
Category = Choice[
    ("billing",   "Payment, invoice, or subscription problem"),
    ("technical", "Bug, crash, or performance issue"),
    ("account",   "Login, permissions, or profile issue"),
]
```

What the bracket syntax actually does (from `dspy/adapters/types/decision.py`, 3.4.0):

- Each subscription builds a new Pydantic subclass with `create_model` behind an `lru_cache(maxsize=256)`,
  named after the declaration (e.g. `Noul[(True, 'Completely blocked'), (False, 'Has workaround')]`). The
  same declaration returns **the same class object** — and for `Noul` the pairs are sorted, so
  `Noul[(False, "b"), (True, "a")] is Noul[(True, "a"), (False, "b")]` (observed: `True`).
- Descriptions must be JSON: `str`, `dict`, `list` or `None`, validated strictly. Rich descriptions
  (`{"definition": ..., "examples": [...]}`) are allowed directly in the declaration.
- `T.criteria()` returns an independent copy of the declared criteria: `{'true': ..., 'false': ...}` for
  `Noul`, a list for `Score`, `{str(value): description}` for `Choice`.

| Constructor | Rules (violations raise `ValueError` at declaration) |
|---|---|
| `Noul[(True, d1), (False, d2)]` | values must be real `bool`s and distinct; one pair is allowed (`Noul[(True, "Blocked")]` → criteria `{'true': 'Blocked'}`); bare `Noul` (no brackets) is valid and has no criteria |
| `Score[d0, d1, ...]` | at least two levels, lowest first. The generated class constrains `value` to [0, N−1] and `level` to [0, N−1]. **More than 10 levels declares fine but fails at call time** (see [Errors and Validation](#errors-and-validation)) |
| `Choice[(v1, d1), (v2, d2), ...]` | values must be `str`, `int`, `bool` or `None`, with distinct `str()` labels (`1` and `"1"` together are rejected). Values keep their Python type in results; probability keys are their string labels |

---

## Signatures and Results

Verbatim from the tutorial:

```python
class TriageTicket(dspy.Signature):
    """Assess a customer support ticket for routing and prioritization."""

    ticket: str = dspy.InputField(desc="The customer's support message.")
    urgent: Urgent = dspy.OutputField(desc="Is the customer completely blocked?")
    severity: Severity = dspy.OutputField(desc="How severely is the customer affected?")
    category: Category = dspy.OutputField(desc="What kind of issue is this?")
```

The tutorial's rule — "Every decision output field must have a `desc=` or an explicit `instructions`
entry in `predict.fields`" — is enforced before any request: the `desc` **becomes the question's
`instructions`**, so write it as the full question, not as a label. (Observed error: `ValueError: Decision
output 'urgent' requires an OutputField(desc=...) or explicit instructions.`)

### Result objects

| Attribute | `Noul` | `Score` | `Choice` |
|---|---|---|---|
| `.value` | `bool` (`p >= threshold`) | `float`, mean level index | the option, original Python type |
| `.probability` | P(True) from the backend | — | — |
| `.probabilities` | — | `dict[int, float]` | `dict[str, float]` |
| `.level` | — | `int`, from `cuts` | — |
| `.confidence` | **computed by DSPy**: `abs(p − threshold) / max(threshold, 1 − threshold)` | from the backend | from the backend |
| conversions | `bool(result)` → `.value` | `float(result)` → `.value` | — |

Three consequences that are easy to miss:

1. **`Noul.confidence` is a distance from the decision boundary, not a probability** (the class docstring
   says so: "a distance from the decision boundary, not a calibrated probability"). With the default
   threshold, P(True) = 0.58 gives confidence 0.16; raise the threshold and the same probability changes
   confidence. Use `.probability` for anything probabilistic.
2. **`Score.value` is recomputed locally** from `probabilities`; with TypeSafe, the API's own `score` field is
   ignored. The client copies `noul`, `score`, `choice`, `confidence` and `probabilities` from the SDK
   response, but `DecisionState._decode` reads only `probabilities` (and `confidence`) for a Score. The
   `legend` is not copied at all — use `Severity.criteria()`.
3. **`Choice.value` is DSPy's arg-max after weights**, not the API's `choice` field. With default weights they
   coincide; after ReAnchor fits weights they may not, by design.

---

## Two Backends, One Program

### Generative LM (no TypeSafe key)

Verbatim from the tutorial:

```python
dspy.configure(lm=dspy.LM("openai/gpt-4o-mini"))

triage = dspy.Predict(TriageTicket)
result = triage(ticket="I can't log in to my account. The password reset page crashes.")

print(f"Urgent:   {result.urgent}")           # Noul object (bool-like)
print(f"  value:  {result.urgent.value}")      # True or False
print(f"  prob:   {result.urgent.probability}") # P(True), e.g. 0.72

print(f"Severity: {result.severity}")
print(f"  value:  {result.severity.value}")    # e.g. 1.4 (continuous)
print(f"  level:  {result.severity.level}")    # e.g. 1 (discrete)

print(f"Category: {result.category}")
print(f"  value:  {result.category.value}")    # e.g. "account"
```

What happens underneath (`dspy/adapters/decision.py`, observed in the prompt DSPy built for `DummyLM`):

- Each decision output's type is swapped for a closed evidence schema — `NoulEvidence {noul}`,
  `ScoreEvidence {probabilities: {"0": .., "1": ..}, confidence}`, `ChoiceEvidence {probabilities: {label: ..},
  confidence}` — with `additionalProperties: false`, and the field description is replaced by the same
  question JSON (`instructions`, `type`, `criteria`) that would be sent to TypeSafe.
- Demos are not turned into fake evidence; they are appended to the instructions as "Task examples (labels
  or evidence)".
- Non-decision outputs (`summary: str`) are generated normally, so mixed signatures work.
- Decision types nested inside other output types (e.g. a `list[Noul[...]]`) are **not** decoded: DSPy warns
  ("Decision evidence decoding is not implemented for nested output ...") and the LM generates values
  directly, without thresholds/cuts/weights.
- Streaming is rejected: `NotImplementedError("Streaming decision evidence is not supported.")`.

Executed with `DummyLM` returning evidence JSON:

```python
import json
import os

os.environ.setdefault("DSPY_CACHEDIR", ".dspy_cache_gen")

import dspy
from dspy.experimental import Choice, Noul, Score
from dspy.utils.dummies import DummyLM

Urgent = Noul[(True, "Completely blocked"), (False, "Has workaround")]
Severity = Score["Minor", "Moderate", "Critical"]
Category = Choice[("billing", "Payment"), ("technical", "Product bug"), ("account", "Access")]


class Triage(dspy.Signature):
    """Route and prioritize support tickets."""

    ticket: str = dspy.InputField(desc="Customer message.")
    urgent: Urgent = dspy.OutputField(desc="Is the customer blocked?")
    severity: Severity = dspy.OutputField(desc="Impact level.")
    category: Category = dspy.OutputField(desc="Issue type.")
    summary: str = dspy.OutputField(desc="One-line summary.")  # allowed on a generative LM only


# A generative LM is asked for *evidence* JSON per decision field (NoulEvidence, ScoreEvidence,
# ChoiceEvidence), never for the label. DummyLM stands in for that LM here.
lm = DummyLM([{
    "urgent": json.dumps({"noul": 0.72}),
    "severity": json.dumps({"probabilities": {"0": 0.1, "1": 0.4, "2": 0.5}, "confidence": 0.3}),
    "category": json.dumps({"probabilities": {"billing": 0.1, "technical": 0.3, "account": 0.6}, "confidence": 0.4}),
    "summary": "Password reset page crashes.",
}] * 5)
dspy.configure(lm=lm)

triage = dspy.Predict(Triage)
ticket = "I can't log in to my account. The password reset page crashes."
r = triage(ticket=ticket)
print(r.urgent.value, r.urgent.probability, round(r.urgent.confidence, 2))  # True 0.72 0.44
print(r.severity.value, r.severity.level)                                  # 1.4 1  (0*.1 + 1*.4 + 2*.5)
print(r.category.value, "|", r.summary)                                    # account | Password reset page crashes.

# Per-field decision parameters live on the predictor, not on the type:
triage.fields["urgent"] = {"threshold": 0.8}
triage.fields["severity"] = {"cuts": [1.5, 1.9]}
triage.fields["category"] = {"weights": {"technical": 2.5}}  # unlisted labels keep weight 1.0
r = triage(ticket=ticket)
print(r.urgent.value, r.severity.level, r.category.value)  # False 0 technical
```

### TypeSafe System One backend

Verbatim from the tutorial:

```python
from dspy.experimental import TypeSafe

os.environ["TYPESAFE_API_KEY"] = "your-api-key"

jev_client = TypeSafe("jev-latest")
triage.set_lm(jev_client)

result = triage(ticket="My dashboard is blank after the update")
```

The request DSPy builds for TypeSafe (`DecisionAdapter._prepare`, confirmed by capturing it):

- `state` is an object: `{"instructions": <signature docstring>, "input_fields": <rendered input field
  descriptions>, "inputs": {<input values>}}`, plus `"demos": [...]` when the predictor has demos. Your
  signature docstring is therefore part of what Jev reads — keep it task-descriptive.
- `questions` has one entry per decision output, keyed by **field name**: `{"instructions": desc,
  "type": "noul" | "score" | "choice", "criteria": ...}`.
- **Every** output must be a decision type or a native equivalent; otherwise `ValueError: Unsupported System
  One output 'summary'; use a decision type or native equivalent.`
- Any generation setting is rejected: `ValueError: Unsupported TypeSafe generation settings: ['temperature'].`
- One predictor call = one `system_one` request carrying all of its decision outputs.

### Mixing backends

The tutorial: "You can even mix backends within a single program — use Jev for critical decisions and a
generative LM for free-text outputs." Mechanically that means binding different clients to different
predictors. `Module.set_lm` sets the LM on **every** predictor in the module, so bind the TypeSafe predictor
after it. Executed:

```python
from ds_fake import SENT  # offline TypeSafe backend; remove in production

import dspy
from dspy.experimental import Noul, TypeSafe
from dspy.utils.dummies import DummyLM

Urgent = Noul[(True, "Completely blocked"), (False, "Has workaround")]


class Decide(dspy.Signature):
    """Decide whether a ticket must be escalated."""

    ticket: str = dspy.InputField(desc="Customer message.")
    urgent: Urgent = dspy.OutputField(desc="Is the customer blocked?")


class Support(dspy.Module):
    def __init__(self):
        super().__init__()
        self.decide = dspy.Predict(Decide)               # every output is a decision type -> TypeSafe-compatible
        self.reply = dspy.Predict("ticket, urgent: bool -> reply: str")

    def forward(self, ticket):
        urgent = self.decide(ticket=ticket).urgent
        return dspy.Prediction(urgent=urgent, reply=self.reply(ticket=ticket, urgent=urgent.value).reply)


program = Support()
program.set_lm(DummyLM([{"reply": "We are on it."}] * 3))  # Module.set_lm sets EVERY predictor...
program.decide.set_lm(TypeSafe("jev-latest"))             # ...so bind the decision predictor afterwards
out = program(ticket="Locked out of my account")
print(out.urgent.value, out.urgent.probability, "|", out.reply, "| TypeSafe requests:", len(SENT))
```

Output: `True 0.66 | We are on it. | TypeSafe requests: 1`

---

## The `TypeSafe` Client

Verbatim excerpt of the constructor in the installed `dspy/clients/typesafe.py` (3.4.0):

```python
class TypeSafe:
    supports_decision_requests = True

    def __init__(
        self,
        model: str | None = None,
        *,
        api_key: str | None = None,
        base_url: str | None = None,
        cache: bool = True,
        timeout: float = 10.0,
        callbacks=None,
    ):
```

| Setting | Resolution | Notes |
|---|---|---|
| `model` | argument → `TYPESAFE_DEFAULT_MODEL` env → `"jev-latest"` | the same env var the TypeSafe Python SDK reads (`typesafe_sdk.constants.DEFAULT_MODEL_ENV`), so one setting changes both; pin e.g. `"jev-1.13"` |
| `api_key` | argument → (inside `typesafe_sdk`) `TYPESAFE_API_KEY` | never written to `dump_state()` or history |
| `base_url` | argument → `TYPESAFE_BASE_URL` → `https://api.typesafe.ai` | trailing `/` stripped |
| `cache` | `True` | DSPy's shared request cache (`dspy.cache`) |
| `timeout` | `10.0` s | passed to the SDK client per call — note the SDK's own `RetryPolicy` defaults to a 30 s total budget, see the Python SDK doc |
| `callbacks` | `[]` | DSPy callbacks (`with_callbacks` wraps `__call__` and `acall`) |

Behaviour, from the source:

- `TypeSafe` is **not** a `dspy.BaseLM`. `Predict` accepts it because it sets
  `supports_decision_requests = True`. It works with `predictor.set_lm(...)`, `module.set_lm(...)`,
  `dspy.configure(lm=TypeSafe(...))` (observed) and `predictor(..., lm=...)`.
- Each uncached call opens a fresh `typesafe_sdk.TypeSafeClient(model=..., api_key=..., base_url=...,
  timeout=...)` in a `with` block (or `AsyncTypeSafeClient` for `acall`). No connection pooling across calls,
  and no `retry` argument is passed, so the SDK's default `RetryPolicy` applies (`max_retries=2`, statuses
  408/429/5xx, `respect_retry_after=True` in `typesafe-sdk` 0.7.2 — see `python-sdk.md` in this folder).
- **Cache key** = `{"provider": "typesafe", "model", "base_url", "state", "questions"}` — the API key is not
  part of it. Changing a threshold/cut/weight does not change the request, which is what makes ReAnchor's
  candidate sweeps free after the first pass.
- Cache hits record empty usage and `cache_hit: True` in `lm.history`; misses add usage to DSPy's usage
  tracker as `{"prompt_tokens": input_tokens, "completion_tokens": output_tokens}`.
- `lm.inspect_history(n)` prints the stored `state`/`questions` prompt and the answers JSON.
- `copy(**kwargs)` keeps the key and callbacks and starts with empty history.

---

## Decision Parameters: `fields`

`predict.fields` is a plain dict on each `Predict`, keyed by output name. Verbatim from the tutorial:

```python
triage.fields["urgent"] = {"threshold": 0.7}
triage.fields["severity"] = {"cuts": [0.8, 1.5]}
triage.fields["category"] = {"weights": {"billing": 2.0, "technical": 1.0, "account": 1.0}}
```

Validation (`DecisionState`, run on every call and on `save`/`load`):

- Allowed keys per entry: the type's parameter plus optional `instructions` and `criteria`; anything else is a
  `ValueError`. Entries are merged over the defaults, so `{"criteria": ...}` alone keeps the default
  threshold.
- `threshold`: `int`/`float` in [0, 1].
- `cuts`: list of exactly N−1 strictly increasing numbers, each strictly between 0 and N−1; the Score must
  have 2–10 levels.
- `weights`: subset of the declared labels (missing labels count as 1.0), finite and ≥ 0, at least one
  effective weight > 0. Weighted ties prefer the raw winner, then declaration order.
- `instructions`: overrides the field `desc` for this predictor (a JSON string, object, array or `null`;
  a bare number or boolean is rejected).
- A `fields` entry for an output that is not a decision type raises "Decision configuration refers to
  unsupported outputs".
- A plain `bool` output (or a `Literal[...]`) with a `fields` entry is **opted in** to evidence decoding — the
  tutorial's FAQ: "You can opt a bare bool into evidence decoding by adding it to `predict.fields`."

`fields` belong to the predictor, not the type. `threshold`, `cuts` and `weights` are not part of the
request, so the same answer can be re-decoded under different parameters without another API call.
`instructions` and `criteria` entries are different: they change the question that is sent, so they produce
new requests (and cache misses).

---

## Criteria: `set_criteria` / `get_criteria`

Verbatim from the tutorial:

```python
triage.set_criteria("urgent", {
    "true": {
        "definition": "Customer is completely unable to use the product",
        "examples": ["App crashes on launch", "Login page returns 500"],
        "not": "Intermittent issues with workarounds",
    },
    "false": {
        "definition": "Customer can still use the product, possibly with degradation",
        "examples": ["Slow page load", "Minor UI glitch"],
    },
})
```

From `Predict.set_criteria` / `DecisionState` (3.4.0):

- `set_criteria(field, criteria)` validates against the declared type and stores a copy at
  `predict.fields[field]["criteria"]`. It is a per-predictor override; the type's declared criteria are
  untouched.
- Shape rules: `Noul` → `None` or a dict with keys ⊆ `{"true", "false"}`; `Choice` → a dict with **exactly**
  the declared labels; `Score` → a list with exactly N entries (2–10).
- `get_criteria(field)` returns the effective criteria: the override if present, else the type's. For a
  `Choice` declared with empty descriptions (e.g. derived from a bare `Literal`), empty strings are reported
  as `None`.
- Criteria are part of the request, so changing them **does** produce new requests (and cache misses).

---

## Native Annotations

Verbatim from the tutorial:

```python
from typing import Annotated, Literal

class SimpleTriageTicket(dspy.Signature):
    """Quick triage returning native Python types."""

    ticket: str = dspy.InputField(desc="Customer message.")
    urgent: Annotated[bool, Urgent] = dspy.OutputField(desc="Is the customer blocked?")
    category: Annotated[
        Literal["billing", "technical", "account"], Category
    ] = dspy.OutputField(desc="Issue type.")
```

Rules from the source:

- `Annotated[bool, Noul[...]]` returns a plain `bool`; `Annotated[Literal[...], Choice[...]]` returns the plain
  member. Thresholds/weights still apply; the probabilities are simply not returned (they are still visible to
  ReAnchor through its evidence recorder).
- The `Choice[...]` metadata must contain exactly the `Literal` members **with the same Python types**, or
  `ValueError: Choice criteria must match the Literal members, including their Python types.`
- `Annotated[float, Score[...]]` is rejected: "Use Score[...] directly as the field type, not
  Annotated[float, Score[...]]." There is no native form for `Score`.
- A bare `Literal[...]` output is decoded as a `Choice` with empty descriptions **on TypeSafe** (System One
  must decode every output) and on a generative LM only when opted in via `fields`.

Executed (generative stand-in; also exercises `set_criteria` and validation):

```python
import json
import os
from typing import Annotated, Literal

os.environ.setdefault("DSPY_CACHEDIR", ".dspy_cache_native")

import dspy
from dspy.experimental import Choice, Noul, Score
from dspy.utils.dummies import DummyLM

Urgent = Noul[(True, "Completely blocked"), (False, "Has workaround")]
Category = Choice[("billing", "Payment"), ("technical", "Product bug"), ("account", "Access")]


class SimpleTriage(dspy.Signature):
    """Quick triage returning native Python types."""

    ticket: str = dspy.InputField(desc="Customer message.")
    urgent: Annotated[bool, Urgent] = dspy.OutputField(desc="Is the customer blocked?")
    category: Annotated[Literal["billing", "technical", "account"], Category] = dspy.OutputField(desc="Issue type.")


lm = DummyLM([{
    "urgent": json.dumps({"noul": 0.31}),
    "category": json.dumps({"probabilities": {"billing": 0.7, "technical": 0.2, "account": 0.1}, "confidence": 0.5}),
}] * 3)
predict = dspy.Predict(SimpleTriage)
out = predict(ticket="The invoice total is wrong", lm=lm)
print(type(out.urgent).__name__, out.urgent, type(out.category).__name__, out.category)  # bool False str billing

# Rich criteria replace the type's descriptions for this predictor only.
predict.set_criteria("urgent", {
    "true": {"definition": "Customer cannot use the product", "examples": ["App crashes on launch"]},
    "false": {"definition": "Customer can still use the product"},
})
print(predict.get_criteria("urgent")["true"]["definition"])
print(predict.fields)  # criteria are stored next to (optional) threshold/cuts/weights

# Validation happens at declaration or at call time, before any LM request:
checks = {
    "Score with one level": lambda: Score["only"],
    "ambiguous Choice labels": lambda: Choice[(1, "one"), ("1", "string one")],
    "Choice criteria with a missing label": lambda: predict.set_criteria("category", {"billing": "x"}),
}
for name, check in checks.items():
    try:
        check()
    except ValueError as error:
        print(f"{name}: {error}")
```

Output:

```text
bool False str billing
Customer cannot use the product
{'urgent': {'criteria': {'true': {'definition': 'Customer cannot use the product', 'examples': ['App crashes on launch']}, 'false': {'definition': 'Customer can still use the product'}}}}
Score with one level: Score requires at least two ordered level descriptions, e.g. Score['poor', 'excellent'].
ambiguous Choice labels: Choice values must have distinct string labels (e.g. 1 and '1' are ambiguous).
Choice criteria with a missing label: Invalid criteria for 'category': must match the declared decision type and options.
```

---

## ReAnchor

`ReAnchor(metric, *, num_threads=None, log_dir=None, require_cache=True)` and
`compile(student, *, trainset, valset=None)` (signatures from `dspy/teleprompt/reanchor/reanchor.py`, 3.4.0).

Verbatim from the tutorial:

```python
optimizer = ReAnchor(metric)
calibrated_triage = optimizer.compile(triage, trainset=trainset)

# The original `triage` is unchanged. `calibrated_triage` has fitted parameters.
print(calibrated_triage.fields)
# e.g. {'urgent': {'threshold': 0.62}, 'severity': {'cuts': [0.45, 1.55]}, ...}
# If defaults already score well, fields may stay empty — that's normal.
```

### What `compile` does (from `reanchor.py` and `calibrate.py`)

1. Deep-copies the student; the original is never modified.
2. Finds every `Predict` with a decision output (`named_parameters`), so a multi-predictor `dspy.Module` is
   calibrated as a whole against one end-to-end metric. None found → `ValueError`.
3. With `require_cache=True`, checks that each predictor's client (bound, or global `dspy.settings.lm`) has
   caching enabled; otherwise refuses with "do not cache responses, so every candidate setting would send
   new requests". With no client bound or configured at all it refuses too — pass `require_cache=False` for
   runtime client selection.
4. Scores the program on `trainset` (and `valset` if given).
5. For each predictor, for each decision output, in order:
   - records the evidence the output is decoded from on every training call (one pass);
   - builds candidates at the **midpoints of the gaps** between observed values — between P(True) values for
     a threshold, between mean level indices for each cut (one cut at a time, kept between its neighbours),
     between the flip points for each Choice weight (geometric midpoints, range 1e‑3 to 1e3, one label at a
     time). More than 40 distinct values are thinned to quantiles (`MAX_CANDIDATES = 40`);
   - re-runs the whole program on the training set for every candidate (cached requests make this cheap);
   - keeps a candidate only if it scores **strictly better** and passes a **fold check**: the training set is
     split into `min(5, n)` parts (`FOLDS = 5`, deterministic shuffle with seed 0); for each part a
     candidate is picked on the other parts and scored on the held-out one, and the summed held-out score
     must beat the current setting's. Ties go
     to the candidate in the widest gap;
   - finally compares the fitted behaviour against the **original** configuration and restores the original
     (including removing an entry that did not exist) unless the fitted one wins.
6. Returns the copy with `fields` set and stores `optimizer.report`:
   `train_score_before`, `train_score`, `fitted` (one row per output: `predictor`, `field`, `parameter`, `value`
   or `skipped`, `observed`, `fold_check`, scores), plus `val_score_before`/`val_score` with a `valset`.
   `log_dir` writes the same report to `report.json`.

Metric contract: per-example, like `dspy.Evaluate`; may return a number or a `dspy.Prediction` with
`.score`; any non-finite value aborts. Any program or metric exception fails the pass
(`ParallelExecutor(max_errors=1)`). An empty `trainset` raises `ValueError`.

Because outputs are fitted sequentially and each candidate re-runs the full program, changing an upstream
decision can change downstream requests — the docstring warns that such requests are new, uncached calls.

### End to end on the TypeSafe backend

Executed (offline backend from [Testing Offline](#testing-offline); eight unique tickets → eight HTTP
requests for the whole calibration, every candidate sweep served from cache):

```python
import json

from ds_fake import SENT  # offline backend; remove in production

import dspy
from dspy.experimental import Choice, Noul, ReAnchor, Score, TypeSafe

Urgent = Noul[(True, "Completely blocked"), (False, "Has workaround")]
Severity = Score["Minor", "Moderate", "Critical"]
Category = Choice[("billing", "Payment"), ("technical", "Product bug"), ("account", "Access")]


class Triage(dspy.Signature):
    """Route and prioritize support tickets."""

    ticket: str = dspy.InputField(desc="Customer message.")
    urgent: Urgent = dspy.OutputField(desc="Is the customer blocked?")
    severity: Severity = dspy.OutputField(desc="Impact level.")
    category: Category = dspy.OutputField(desc="Issue type.")


triage = dspy.Predict(Triage)
triage.set_lm(TypeSafe("jev-latest"))  # TYPESAFE_API_KEY from the environment

result = triage(ticket="My dashboard is blank after the update")
print(result.urgent.value, result.urgent.probability, round(result.urgent.confidence, 2))  # True 0.58 0.16
print(result.severity.value, result.severity.level, result.severity.probabilities)
print(result.category.value, result.category.probabilities)
print(json.dumps(SENT[-1]["state"]))  # what DSPy actually sends as `state`

trainset = [
    dspy.Example(ticket="Payment failed, order stuck", urgent=True, severity=2, category="billing").with_inputs("ticket"),
    dspy.Example(ticket="Typo in the help page", urgent=False, severity=0, category="technical").with_inputs("ticket"),
    dspy.Example(ticket="Can't reset my password, locked out", urgent=True, severity=2, category="account").with_inputs("ticket"),
    dspy.Example(ticket="Invoice date is wrong", urgent=False, severity=0, category="billing").with_inputs("ticket"),
    dspy.Example(ticket="App freezes in settings", urgent=False, severity=1, category="technical").with_inputs("ticket"),
    dspy.Example(ticket="Checkout crash on submit", urgent=True, severity=2, category="technical").with_inputs("ticket"),
    dspy.Example(ticket="Receipt shows old address", urgent=False, severity=0, category="billing").with_inputs("ticket"),
    dspy.Example(ticket="Dark mode colours look off", urgent=False, severity=0, category="technical").with_inputs("ticket"),
]


def metric(example, pred, trace=None):
    s = float(pred.urgent.value == example.urgent) + float(pred.category.value == example.category)
    s += float(pred.severity.level == example.severity)
    return s / 3


calls_before = len(SENT)
optimizer = ReAnchor(metric, num_threads=1)
calibrated = optimizer.compile(triage, trainset=trainset)
print("HTTP calls during compile:", len(SENT) - calls_before)
print("student untouched:", triage.fields)
print("fitted:", calibrated.fields)
print("score:", optimizer.report["train_score_before"], "->", optimizer.report["train_score"])
for row in optimizer.report["fitted"]:
    print(" ", row["field"], row["parameter"], row.get("value", row.get("skipped")), row["fold_check"])

calibrated.save("triage_calibrated.json")
saved = json.load(open("triage_calibrated.json"))
print("saved lm:", saved["lm"])
loaded = dspy.Predict(Triage)
loaded.load("triage_calibrated.json")  # warns: base_url dropped unless allow_unsafe_lm_state=True
print("loaded:", loaded.fields, type(loaded.lm).__name__, loaded.lm.base_url)
```

Output (log lines come from stderr and are interleaved; their timestamps and the `tqdm` progress bar of
the evaluation pass are removed here):

```text
INFO dspy.teleprompt.reanchor.reanchor: answering 8 training examples
True 0.58 0.16
0.4 0 {0: 0.8, 1: 0.0, 2: 0.2}
technical {'billing': 0.15, 'technical': 0.7, 'account': 0.15}
{"instructions": "Route and prioritize support tickets.", "input_fields": "1. `ticket` (str): Customer message.", "inputs": {"ticket": "My dashboard is blank after the update"}}
INFO dspy.teleprompt.reanchor.reanchor: fitting thresholds, cuts, and weights
INFO dspy.teleprompt.reanchor.reanchor: calibrated: train 0.625 -> 0.9583
WARNING dspy.predict.predict: Ignoring unsafe LM config key(s) during state load: ['base_url']. Pass allow_unsafe_lm_state=True to preserve these keys for trusted files.
HTTP calls during compile: 8
student untouched: {}
fitted: {'urgent': {'threshold': 0.6}, 'severity': {'cuts': [0.5, 0.9]}}
score: 0.625 -> 0.9583
  urgent threshold 0.6 {'passed': 1, 'failed': 0}
  severity cuts [0.5, 0.9] {'passed': 1, 'failed': 0}
  category weights fitted behavior did not beat the original {'passed': 0, 'failed': 0}
saved lm: {'_dspy_lm_class': 'dspy.clients.typesafe.TypeSafe', 'model': 'jev-latest', 'base_url': 'https://api.typesafe.ai', 'cache': True, 'timeout': 10.0}
loaded: {'urgent': {'threshold': 0.6}, 'severity': {'cuts': [0.5, 0.9]}} TypeSafe https://api.typesafe.ai
```

Read the report, not just the fields: here `category` weights were tried (6 candidates) and **rejected** —
"fitted behavior did not beat the original" — so `category` is absent from `calibrated.fields`. The
tutorial's FAQ on dataset size: "ReAnchor requires a nonempty trainset and uses up to 5-fold
cross-validation (fewer folds for smaller sets)." With few examples the fold check will refuse most changes;
that is the intended protection, not a bug.

---

## Save and Load

Verbatim from the tutorial:

```python
calibrated_triage.save("triage_calibrated.json")

# Later:
loaded_triage = dspy.Predict(TriageTicket)
loaded_triage.load("triage_calibrated.json")
print(loaded_triage.fields)  # Fitted parameters restored
```

From `Predict.dump_state` / `load_state` (3.4.0) and the run above:

- The JSON contains `fields` (thresholds, cuts, weights **and** any `set_criteria` overrides), validated
  against the signature on both save and load.
- A bound `TypeSafe` client is saved as
  `{"_dspy_lm_class": "dspy.clients.typesafe.TypeSafe", "model", "base_url", "cache", "timeout"}` — no API key —
  and restored as a `TypeSafe` instance (it is on `BaseLM.load_state`'s allowlist, no
  `allow_custom_lm_class` needed).
- **`base_url` is dropped on load** unless you pass `allow_unsafe_lm_state=True` (observed warning:
  "Ignoring unsafe LM config key(s) during state load: ['base_url']"). The loaded client then falls back to
  `TYPESAFE_BASE_URL` or the public endpoint — a program saved against a gateway or private deployment
  silently points back at `https://api.typesafe.ai` unless the env var is set.
- Load into a `Predict` built from the same signature. Saved `fields` that name outputs the loading
  signature does not have as decision outputs raise "Decision configuration refers to unsupported outputs".
  Separately, passing a different `signature=` at call time that changes a configured output's answer space
  raises "Signature override must preserve the answer space of configured output" (from `DecisionState`).

---

## Errors and Validation

| When | What | Message |
|---|---|---|
| declaration | bad `Noul`/`Score`/`Choice` brackets | `Score requires at least two ordered level descriptions, e.g. Score['poor', 'excellent'].` |
| first call without the extra | SDK missing | `Install TypeSafe support with pip install "dspy[typesafe]".` |
| call, any backend | decision output without `desc`/`instructions` | `Decision output 'urgent' requires an OutputField(desc=...) or explicit instructions.` |
| call, any backend | invalid `fields` | `Invalid cuts for 'severity': require ordered boundaries inside the Score index range.` |
| call, TypeSafe | non-decision output | `Unsupported System One output 'summary'; use a decision type or native equivalent.` |
| call, TypeSafe | `config={"temperature": ...}` or other LM kwargs | `Unsupported TypeSafe generation settings: ['temperature'].` |
| call, TypeSafe | HTTP error | the `typesafe_sdk` exception, unchanged (e.g. `typesafe_sdk.TypeSafeAuthenticationError`) |
| call, any backend | malformed evidence (Score keys ≠ levels, Choice labels ≠ options, zero mass) | `Invalid Score distribution for 'severity'.` / `Invalid Choice answer for 'category'.` |
| compile | no client / uncached client / empty trainset | see [ReAnchor](#reanchor) |

Messages in rows exercised by this document's executed examples were observed; the "SDK missing" and "malformed evidence" messages are quoted from the 3.4.0 source.

A `Score` with **more than 10 levels** is accepted at declaration and then rejected on every call with the
misleading "Invalid cuts" message, because the default cuts are validated against a 2–10 level range.

Executed:

```python
import os

os.environ.setdefault("DSPY_CACHEDIR", ".dspy_cache_err")
os.environ.setdefault("TYPESAFE_API_KEY", "ts-wrong")

import httpx2
import typesafe_sdk

import dspy
from dspy.experimental import Noul, Score, TypeSafe


class _Rejecting(typesafe_sdk.TypeSafeClient):  # offline: every request gets a 401
    def __init__(self, **kwargs):
        super().__init__(transport=httpx2.MockTransport(
            lambda request: httpx2.Response(401, json={"error": {"message": "bad key"}})), **kwargs)


typesafe_sdk.TypeSafeClient = _Rejecting


class Check(dspy.Signature):
    """Check a ticket."""

    ticket: str = dspy.InputField(desc="Customer message.")
    urgent: Noul[(True, "Blocked"), (False, "Not blocked")] = dspy.OutputField(desc="Is the customer blocked?")


predict = dspy.Predict(Check)
predict.set_lm(TypeSafe(cache=False))
try:
    predict(ticket="hi")
except typesafe_sdk.TypeSafeAuthenticationError as error:  # SDK exceptions propagate unchanged
    print(type(error).__name__)

# Score with more than 10 levels: declared fine, rejected at call time by the cuts validator.
Wide = Score[tuple(f"level {i}" for i in range(11))]


class Rate(dspy.Signature):
    """Rate a ticket."""

    ticket: str = dspy.InputField(desc="Customer message.")
    level: Wide = dspy.OutputField(desc="How bad is it?")


try:
    rate = dspy.Predict(Rate)
    rate.set_lm(TypeSafe(cache=False))
    rate(ticket="hi")
except ValueError as error:
    print(error)
```

Output:

```text
TypeSafeAuthenticationError
Invalid cuts for 'level': require ordered boundaries inside the Score index range.
```

---

## Testing Offline

`TypeSafe` has no transport parameter, but it imports `TypeSafeClient` / `AsyncTypeSafeClient` from
`typesafe_sdk` **inside each call**, so replacing those module attributes with subclasses that pass an
`httpx2.MockTransport` routes every request offline while keeping the real SDK's request building and
response parsing. Save this helper as `ds_fake.py`; the TypeSafe examples above import it (executed). It
answers in the documented `POST /v1/systemone` response shape and records each request body in `SENT`:

```python
"""Offline TypeSafe backend for dspy.experimental.TypeSafe.

dspy imports TypeSafeClient / AsyncTypeSafeClient from typesafe_sdk lazily, inside each call,
so replacing the module attributes routes every request through an httpx2.MockTransport.
Import this module BEFORE the first prediction. No API key, no network.
"""
import json
import os
import tempfile

# A fresh cache directory per process: a persistent one would answer from disk on the next run and
# leave SENT empty.
os.environ.setdefault("DSPY_CACHEDIR", tempfile.mkdtemp(prefix="dspy_cache_"))
os.environ.setdefault("TYPESAFE_API_KEY", "ts-test")

import httpx2
import typesafe_sdk

SENT: list[dict] = []


def _answer(body: dict) -> dict:
    text = json.dumps(body["state"]).lower()
    blocked = any(w in text for w in ("can't", "locked", "crash", "stuck"))
    answers = {}
    for qid, q in body["questions"].items():
        if q["type"] == "noul":
            answers[qid] = {"type": "noul", "noul": 0.66 if blocked else 0.58}
        elif q["type"] == "score":
            n, hi = len(q["criteria"]), (0.7 if blocked else 0.2)
            probs = {str(i): 0.0 for i in range(n)}
            probs["0"], probs[str(n - 1)] = round(1 - hi, 2), hi
            answers[qid] = {"type": "score", "score": hi * (n - 1), "confidence": 0.5, "probabilities": probs,
                            "legend": {str(i): c for i, c in enumerate(q["criteria"])}}
        else:
            labels = list(q["criteria"])
            pick = ("billing" if any(w in text for w in ("invoice", "payment", "receipt"))
                    else "account" if any(w in text for w in ("password", "locked")) else "technical")
            pick = pick if pick in labels else labels[0]
            probs = {label: 0.7 if label == pick else 0.3 / (len(labels) - 1) for label in labels}
            answers[qid] = {"type": "choice", "choice": pick, "probabilities": probs, "confidence": 0.55}
    return answers


def _handler(request: httpx2.Request) -> httpx2.Response:
    body = json.loads(request.content)
    SENT.append(body)
    return httpx2.Response(200, json={"model": "jev-1.13", "answers": _answer(body),
                                      "usage": {"input_tokens": 300, "output_tokens": 0}})


async def _ahandler(request: httpx2.Request) -> httpx2.Response:
    return _handler(request)


class _Client(typesafe_sdk.TypeSafeClient):
    def __init__(self, **kwargs):
        super().__init__(transport=httpx2.MockTransport(_handler), **kwargs)


class _AsyncClient(typesafe_sdk.AsyncTypeSafeClient):
    def __init__(self, **kwargs):
        super().__init__(transport=httpx2.MockTransport(_ahandler), **kwargs)


typesafe_sdk.TypeSafeClient = _Client
typesafe_sdk.AsyncTypeSafeClient = _AsyncClient
```

Notes:

- Set `DSPY_CACHEDIR` to a fresh temporary directory in tests, as the helper does. DSPy's cache persists to
  disk, so a cached answer from an earlier run hides a change in your mock. With a fixed directory, a
  second run of the ReAnchor example answers everything from disk, leaves `SENT` empty and fails at
  `SENT[-1]` (observed). `TypeSafe(cache=False)` skips the cache entirely (but
  then ReAnchor needs `require_cache=False`).
- For generative-LM tests, `dspy.utils.dummies.DummyLM` with evidence JSON per decision field (as in
  the generative-LM example above) exercises the same decoder.
- The patch must happen before the first prediction; it is process-global, so keep it in test fixtures.

---

## Gotchas Checklist

- [ ] The `desc=` of a decision output is sent as the question's `instructions`; write the full question.
- [ ] On TypeSafe, **every** output must be a decision type (or `Annotated` bool/`Literal`), and no
      `temperature`/LM kwargs may be passed.
- [ ] `Noul.confidence` is distance-from-threshold, not probability; `Score.value` is DSPy's mean index;
      `Choice.value` is DSPy's weighted arg-max. Read `.probability` / `.probabilities` for evidence.
- [ ] `Module.set_lm` overwrites every predictor; bind TypeSafe predictors afterwards when mixing backends.
- [ ] `TypeSafe` timeout defaults to 10 s; `TYPESAFE_DEFAULT_MODEL` changes the default model silently.
- [ ] Keep the cache on for ReAnchor; set `DSPY_CACHEDIR` in tests.
- [ ] ReAnchor fits decision boundaries against your metric; it does not calibrate probabilities. Check the
      `report["fitted"]` rows for `skipped` entries before trusting `fields`.
- [ ] `load()` drops `base_url` unless `allow_unsafe_lm_state=True`.
- [ ] Score: 2–10 levels in practice.
- [ ] Everything here is `@experimental` in DSPy 3.4.0 — pin `dspy==3.4.0`.
