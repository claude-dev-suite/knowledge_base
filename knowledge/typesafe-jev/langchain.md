# TypeSafe Jev in LangChain (`langchain-typesafe` / `@langchain/typesafe`)

> Official Documentation: https://docs.langchain.com/oss/python/integrations/providers/typesafe
> Official Documentation (JS): https://docs.langchain.com/oss/javascript/integrations/providers/typesafe
> API Reference: https://reference.langchain.com/python/langchain-typesafe
> Source (Python): https://github.com/langchain-ai/langchain (package `langchain-typesafe`)
> Source (JS): https://github.com/langchain-ai/langchainjs/tree/main/libs/providers/langchain-typesafe
> TypeSafe documentation: https://docs.typesafe.ai/
> Last verified: 2026-10-03

**Versions verified against (2026-10-03):**

| Package | Version | Notes |
|---|---|---|
| `langchain-typesafe` (PyPI) | `0.0.1a3` | uploaded 2026-09-20; earlier `0.0.1a1`/`0.0.1a2` both 2026-09-17 (PyPI JSON API) |
| `langchain-core` (Python) | `1.6.6` | package requires `>=1.6.2,<2.0.0` |
| `langchain` (Python, `[experimental]` extra) | `1.4.3` (+ `langgraph` `1.2.12`) | extra requires `langchain>=1.3.15,<2.0.0` |
| `httpx2` | `2.13.1` | the package's only other runtime dependency |
| `@langchain/typesafe` (npm) | `0.0.2` | npm `time.modified` 2026-10-01; `0.0.1` and `0.0.2` are the only versions |
| `@langchain/core` (JS) | `1.2.14` | peer `^1.0.0` |
| `langchain` (JS) | `1.5.15` | **optional** peer `^1.0.0`, needed only for `@langchain/typesafe/middleware` |
| TypeScript used for type-checking | `5.9.3` | `strict`, `NodeNext` |

All Python examples marked "executed" were run offline on Python 3.12.10 with an `httpx2.MockTransport`
standing in for the API; all JS examples marked "executed" were compiled with `tsc` and run on Node 22.22.0
with a fake `fetch`. No API key was used and the real API was never called. The model names in the
verbatim LangChain examples (`openai:gpt-5.6-terra`, `openai:gpt-6-astra`) are copied from the docs page and
were not checked against OpenAI.

## Overview

`TypeSafeClassifier` wraps TypeSafe's System One endpoint (`POST /v1/systemone`) as a LangChain `Runnable`.
You send a `state` (text, JSON, or LangChain messages) and a map of typed questions (`Noul`, `Choice`,
`Score`); you get back typed answers with calibrated probabilities instead of generated text. Because it is
a `Runnable` it composes with `|`, batches, streams through callbacks, and is traced by LangSmith like any
other LangChain component.

The Python and JavaScript packages share a name and an idea, **but not an API**:

- **Python** (`0.0.1a3`): the classifier is configured with transport settings only; **`state` and
  `questions` both go into `invoke`**. Messages are sent as role/content JSON. No retries. Experimental
  middleware lives at `langchain_typesafe.experimental.middleware`.
- **JavaScript** (`0.0.2`): **questions are a constructor setting**; `invoke` takes only the state.
  Messages are rendered to one transcript line each. Retries through LangChain's `AsyncCaller`
  (`maxRetries` default 2). Middleware lives at `@langchain/typesafe/middleware` — added in `0.0.2` and
  **not mentioned on the JS docs page** as of 2026-10-03.

This document goes below the docs page: what each class actually sends on the wire, what is and is not
validated, the error hierarchy, the retry behaviour that differs between the two languages, the
middleware internals (fixed thresholds, what state is classified, the Python default-criteria gap), and how
to test all of it offline.

---

## Table of Contents

1. [Installation and Configuration](#installation-and-configuration)
2. [Quickstart (Python)](#quickstart-python)
3. [Question and Answer Types](#question-and-answer-types)
4. [The Classifier as a Runnable](#the-classifier-as-a-runnable)
5. [State Forms and Message Serialization](#state-forms-and-message-serialization)
6. [The Breaking Change: Questions Moved to `invoke`](#the-breaking-change-questions-moved-to-invoke)
7. [Errors and Retries](#errors-and-retries)
8. [LangSmith Tracing](#langsmith-tracing)
9. [Experimental Middleware (Python)](#experimental-middleware-python)
10. [Custom Middleware](#custom-middleware)
11. [JavaScript: `@langchain/typesafe`](#javascript-langchaintypesafe)
12. [Python vs JavaScript Differences](#python-vs-javascript-differences)
13. [Gotchas Checklist](#gotchas-checklist)

---

## Installation and Configuration

Verbatim from https://docs.langchain.com/oss/python/integrations/providers/typesafe:

```bash
pip install langchain-typesafe
```

```bash
export TYPESAFE_API_KEY=...
```

The experimental middleware needs the `experimental` extra (it pulls in `langchain`, and with it
`langgraph`). Without it, importing `langchain_typesafe.experimental.middleware` raises
`ImportError: AutoModeMiddleware requires the LangChain agent framework. Install it with
`pip install 'langchain-typesafe[experimental]'`.` (observed on 0.0.1a3 in an environment without
`langchain`).

### Constructor fields (Python, `TypeSafeClassifier`)

From the installed source of `langchain_typesafe/classifier.py` (0.0.1a3). The model is a Pydantic model
with `extra="forbid"`, so any unknown keyword is a `ValidationError`.

| Field | Type | Default | Behaviour |
|---|---|---|---|
| `model` | `str` | `"jev-latest"` | stripped of whitespace; empty string rejected. Pin a concrete model (e.g. `jev-1.13`) for reproducibility |
| `api_key` | `SecretStr \| str` | `TYPESAFE_API_KEY` env var, read **at construction** | blank/missing raises `ValidationError` ("TypeSafe API key is required...") at construction, not at first call |
| `base_url` | `str` | `TYPESAFE_BASE_URL` env var, then `https://api.typesafe.ai` | requests go to `{base_url.rstrip('/')}/v1/systemone` |
| `timeout` | `float` (> 0) | `30.0` seconds | applied **only** to clients the classifier creates itself |
| `client` | `httpx2.Client \| None` | created with `timeout` | used by `invoke` / `batch`; never closed by the classifier |
| `async_client` | `httpx2.AsyncClient \| None` | created with `timeout` | used by `ainvoke` / `abatch`; never closed by the classifier |

Details that matter in practice:

- The HTTP library is **`httpx2`**, not `httpx`, and the package does **not** depend on `typesafe-sdk`.
  An `httpx.Client` will not satisfy the `client` field.
- An injected client keeps **its own** timeout. `httpx2.Client()` defaults to 5 s, so a test or proxy
  client silently replaces the 30 s default (observed: `TypeSafeAPITimeoutError: Request timed out
  (timeout=Timeout(timeout=5.0)).` with an injected default client).
- There is no `close()`/context manager. Keep one long-lived classifier for connection pooling; close
  `classifier.client` / `classifier.async_client` yourself if you need deterministic cleanup.
- The class is decorated with LangChain's `@beta()`: constructing it emits `LangChainBetaWarning`.
- Every request carries `Authorization: Bearer <key>`, `Content-Type: application/json` and
  `User-Agent: langchain-typesafe/0.0.1a3`.

---

## Quickstart (Python)

Verbatim from https://docs.langchain.com/oss/python/integrations/providers/typesafe:

```python
from langchain_typesafe import Choice, Noul, Score, TypeSafeClassifier

classifier = TypeSafeClassifier()

response = classifier.invoke(
    {
        "state": (
            "The deploy failed twice and customers are seeing 500s. "
            "Can someone look now?"
        ),
        "questions": {
            "urgent": Noul(instructions="Does this need attention right now?"),
            "team": Choice(
                instructions="Which team should pick this up?",
                criteria={
                    "infra": "Deploys, availability, and on-call incidents.",
                    "billing": "Payments, invoices, and subscriptions.",
                },
            ),
            "severity": Score(
                instructions="How severe is the impact?",
                criteria=["Cosmetic.", "Degraded for some users.", "Full outage."],
            ),
        },
    }
)

print(response.nouls["urgent"].noul)
print(response.choices["team"].choice, response.choices["team"].confidence)
print(response.scores["severity"].score)
```

Questions that share state are evaluated independently, in one HTTP request. One request is one
`invoke`; the request body is exactly `{"state": ..., "model": ..., "questions": {...}}` (verified by
capturing it, see below).

---

## Question and Answer Types

Everything below is exported from `langchain_typesafe` (`__all__` in 0.0.1a3): `Answer`, `Choice`,
`ChoiceAnswer`, `ClassifierRequest`, `ClassifierResponse`, `Noul`, `NoulAnswer`, `NoulCriteria`,
`Question`, `Score`, `ScoreAnswer`, `State`, `TypeSafeClassifier`, `Usage`, `__version__`. The error classes
are **not** re-exported; import them from `langchain_typesafe.client`.

### Questions (Pydantic models, serialized with `model_dump(mode="json", exclude_none=True)`)

| Class | `type` | Required | Validation (0.0.1a3) |
|---|---|---|---|
| `Noul` | `"noul"` | `instructions` | `criteria: NoulCriteria \| None = None`; `NoulCriteria(true=..., false=...)`, both any JSON, default `None`. **`instructions` is required** — `Noul(criteria=...)` alone is a `ValidationError` (the API and the JS package accept criteria-only) |
| `Choice` | `"choice"` | `instructions`, `criteria` | `criteria: dict[str, JsonValue]` with `min_length=1`; values may be text, JSON or `None` |
| `Score` | `"score"` | `instructions`, `criteria` | `criteria: list[JsonValue]` with `min_length=2`, ordered low to high, index = level |

`instructions` is required on all three Python classes. In the API's OpenAPI schema (`NoulQuestion`,
`ChoiceQuestion`, `ScoreQuestion`), `instructions` is optional for every type: only `type` is required,
plus `criteria` for `choice` and `score`. The JS package matches the API here. Python 0.0.1a3 is
therefore stricter than both.

`instructions` may be a string, a JSON object, or a JSON array. Because `exclude_none=True`, a `Noul`
without criteria is sent with no `criteria` key at all, and a `NoulCriteria(true="x")` is sent as
`{"true": "x"}`.

`Question` is the discriminated union `Annotated[Noul | Choice | Score, Field(discriminator="type")]`.

### Answers

| Class | Fields | Notes |
|---|---|---|
| `NoulAnswer` | `type`, `noul: float` in [0, 1] | P(yes). No `confidence` — the probability is the answer |
| `ChoiceAnswer` | `type`, `choice: str`, `probabilities: dict[str, float]`, `confidence: float` in [0, 1] | `confidence` describes the shape of the distribution; it is **not** the winning label's probability |
| `ScoreAnswer` | `type`, `score: float`, `legend: dict[int, JsonValue]`, `probabilities: dict[int, float]`, `confidence` | `score` is an expected value and may be fractional; `legend`/`probabilities` keys are **ints** after validation (JSON sends strings) |

### `ClassifierResponse`

| Attribute | Type | Source |
|---|---|---|
| `model` | `str` | the model that actually answered (e.g. `jev-1.13` when you asked for `jev-latest`) |
| `answers` | `dict[str, Answer]` | every answer keyed by your question id |
| `usage` | `Usage(input_tokens: int \| None, output_tokens: int \| None)` | defaults to an empty `Usage` when the API omits it |
| `request_id` | `str \| None` | copied from the `x-typesafe-request-id` response header |
| `nouls` / `choices` / `scores` | `@property` → `dict[str, NoulAnswer]` etc. | filtered views of `answers`; recomputed on each access |

For what `confidence` means and how to pick thresholds, see the TypeSafe confidence guide
(https://docs.typesafe.ai/confidence) and the `decision-calibration` KB documents; this package passes
the numbers through unchanged.

---

## The Classifier as a Runnable

`TypeSafeClassifier` is a `RunnableSerializable[ClassifierRequest, ClassifierResponse]`. It overrides
`invoke` and `ainvoke` only. Consequences:

- **`batch` / `abatch` are the inherited defaults**: one HTTP request per input, sync batch on a thread
  pool, bounded by `config={"max_concurrency": N}`. There is no multi-state batch endpoint underneath.
  To ask many questions about **one** state, put them in one `questions` map instead — that *is* one
  request.
- **`stream` yields one complete result**, there is nothing incremental.
- `invoke(input, config=None, **_)` ignores extra keyword arguments.
- Each call is traced with `run_type="llm"` (see [LangSmith Tracing](#langsmith-tracing)).

### Offline test double

The constructor's `client` / `async_client` fields are the supported seam for tests. Save this helper as
`lc_fake.py`; every Python example below imports it (executed, `langchain-typesafe` 0.0.1a3):

```python
"""Offline stand-in for POST /v1/systemone (documented response shape). No API key, no network."""
import json

import httpx2
from langchain_typesafe import TypeSafeClassifier

SENT: list[dict] = []  # every request body the classifier sent


def fake_api(request: httpx2.Request) -> httpx2.Response:
    body = json.loads(request.content)
    SENT.append(body)
    answers = {}
    for qid, q in body["questions"].items():
        if q["type"] == "noul":
            answers[qid] = {"type": "noul", "noul": 0.91}
        elif q["type"] == "choice":
            first, *rest = q["criteria"]
            probs = {first: 0.8, **{label: 0.2 / len(rest) for label in rest}}
            answers[qid] = {"type": "choice", "choice": first, "probabilities": probs, "confidence": 0.62}
        else:
            n = len(q["criteria"])
            answers[qid] = {
                "type": "score", "score": 1.35, "confidence": 0.4,
                "legend": {str(i): c for i, c in enumerate(q["criteria"])},
                "probabilities": {str(i): 1 / n for i in range(n)},
            }
    return httpx2.Response(
        200,
        json={"model": "jev-1.13", "answers": answers, "usage": {"input_tokens": 120, "output_tokens": 0}},
        headers={"x-typesafe-request-id": "req_123"},
    )


async def fake_api_async(request: httpx2.Request) -> httpx2.Response:
    return fake_api(request)


def make_classifier(handler=fake_api, async_handler=fake_api_async) -> TypeSafeClassifier:
    return TypeSafeClassifier(
        api_key="ts-test",  # any non-blank string; validated, never checked by the mock
        client=httpx2.Client(transport=httpx2.MockTransport(handler)),
        async_client=httpx2.AsyncClient(transport=httpx2.MockTransport(async_handler)),
    )
```

### invoke, batch, ainvoke, compose

Executed (outputs shown as comments are the actual outputs):

```python
import asyncio

from langchain_core.runnables import RunnableLambda
from langchain_typesafe import Choice, Noul, Score

from lc_fake import make_classifier

classifier = make_classifier()
QUESTIONS = {
    "urgent": Noul(instructions="Does this need attention right now?"),
    "team": Choice(
        instructions="Which team should pick this up?",
        criteria={"infra": "Deploys and incidents.", "billing": "Payments and invoices."},
    ),
    "severity": Score(instructions="How severe is the impact?", criteria=["Cosmetic.", "Degraded.", "Outage."]),
}

# invoke
response = classifier.invoke({"state": "Deploy failed; customers see 500s.", "questions": QUESTIONS})
print(response.model, response.request_id, response.usage)
# -> jev-1.13 req_123 input_tokens=120 output_tokens=0
print(response.nouls["urgent"].noul, response.choices["team"].choice, response.scores["severity"].legend)
# -> 0.91 infra {0: 'Cosmetic.', 1: 'Degraded.', 2: 'Outage.'}   (legend keys are ints)

# batch: the inherited Runnable.batch, i.e. one HTTP request per input, run in a thread pool
responses = classifier.batch(
    [{"state": t, "questions": {"urgent": QUESTIONS["urgent"]}} for t in ("ticket A", "ticket B")],
    config={"max_concurrency": 2},
)
print([r.nouls["urgent"].noul for r in responses])  # -> [0.91, 0.91]

# ainvoke uses async_client, never client
print(asyncio.run(classifier.ainvoke({"state": "x", "questions": {"urgent": QUESTIONS["urgent"]}})).nouls)
# -> {'urgent': NoulAnswer(type='noul', noul=0.91)}

# compose: text in, routing decision out
route = (
    RunnableLambda(lambda text: {"state": text, "questions": QUESTIONS})
    | classifier
    | RunnableLambda(lambda r: "page-oncall" if r.nouls["urgent"].noul >= 0.8 else "queue")
)
print(route.invoke("Production is down"))  # -> page-oncall
```

---

## State Forms and Message Serialization

`State` is `str | BaseMessage | Sequence[...] | dict[str, ...]`. Serialization
(`langchain_typesafe/_state.py`) walks the value recursively:

- A `BaseMessage` anywhere (root or nested) is converted with `langchain_core.messages.convert_to_openai_messages`
  — OpenAI-style `{"role", "content"}` dicts, tool calls as `{"type": "function", "id", "function": {"name",
  "arguments"}}` with **arguments as a JSON string**, tool results with `tool_call_id`. Message ids are dropped.
- Dict keys must be strings (`TypeError: TypeSafe state object keys must be strings.`).
- Scalars (`int`, `float`, `bool`, `None`) are fine **nested** but rejected **at the root**.
- Anything else (a `datetime`, a Pydantic model, a set) raises `TypeError: Unsupported TypeSafe state
  value: <type>.` — convert it yourself first.

Executed:

```python
import json

from langchain_core.messages import AIMessage, HumanMessage, SystemMessage, ToolMessage
from langchain_typesafe import Noul

from lc_fake import SENT, make_classifier

classifier = make_classifier()
question = {"urgent": Noul(instructions="Does the customer need a human now?")}

conversation = [
    SystemMessage("You are a support agent."),
    HumanMessage("Refund the duplicate charge", id="msg-1"),
    AIMessage("", tool_calls=[{"name": "issue_refund", "args": {"amount": 250}, "id": "call-1"}]),
    ToolMessage("done", tool_call_id="call-1"),
]
# Messages may sit at the root or anywhere inside a JSON object/array.
classifier.invoke({"state": {"ticket_id": 42, "messages": conversation}, "questions": question})
print(json.dumps(SENT[-1]["state"], indent=1))

for bad_root in (42, None, True):
    try:
        classifier.invoke({"state": bad_root, "questions": question})
    except TypeError as error:
        print(repr(bad_root), "->", error)
```

Captured output:

```text
{
 "ticket_id": 42,
 "messages": [
  {
   "role": "system",
   "content": "You are a support agent."
  },
  {
   "role": "user",
   "content": "Refund the duplicate charge"
  },
  {
   "role": "assistant",
   "tool_calls": [
    {
     "type": "function",
     "id": "call-1",
     "function": {
      "name": "issue_refund",
      "arguments": "{\"amount\": 250}"
     }
    }
   ],
   "content": ""
  },
  {
   "role": "tool",
   "tool_call_id": "call-1",
   "content": "done"
  }
 ]
}
42 -> TypeSafe state must be a string, object, array, BaseMessage, or sequence of BaseMessage objects.
None -> TypeSafe state must be a string, object, array, BaseMessage, or sequence of BaseMessage objects.
True -> TypeSafe state must be a string, object, array, BaseMessage, or sequence of BaseMessage objects.
```

Two consequences worth designing around:

1. **Everything in `state` is sent to TypeSafe**, including tool-call arguments and system prompts. Strip
   secrets before classifying agent state.
2. The tool-call arguments travel as a *string inside JSON*. That is what `convert_to_openai_messages`
   produces; the JS package renders the same conversation very differently (see
   [Python vs JavaScript Differences](#python-vs-javascript-differences)), so the two languages do not send
   byte-identical state for the same conversation.

---

## The Breaking Change: Questions Moved to `invoke`

From the docs page (verbatim note): "Earlier versions configured `questions` on `TypeSafeClassifier` and
passed only the state to `invoke`. Pass both `state` and `questions` to `invoke` now."

The page does not name the version that changed. Which of `0.0.1a1`/`0.0.1a2`/`0.0.1a3` introduced it was
**not verified**; on `0.0.1a3` the old shapes fail as follows. The source's own rationale (docstring of
`ClassifierRequest`): keeping state and questions in the Runnable input "ensures both values participate in
composition, batching, and tracing".

Executed:

```python
from pydantic import ValidationError
from langchain_typesafe import Noul, TypeSafeClassifier

from lc_fake import make_classifier

questions = {"urgent": Noul(instructions="Is this urgent?")}

# Old shape (questions on the constructor) is rejected: the model forbids extra fields.
try:
    TypeSafeClassifier(api_key="k", questions=questions)
except ValidationError as error:
    print(error.errors()[0]["type"], error.errors()[0]["loc"])     # extra_forbidden ('questions',)

classifier = make_classifier()
# Old call shape (bare state) fails inside the classifier, not at a type boundary.
try:
    classifier.invoke("Production is down")
except TypeError as error:
    print("TypeError:", error)

# New shape: state and questions travel together in the Runnable input.
print(classifier.invoke({"state": "Production is down", "questions": questions}).nouls["urgent"].noul)

# Keep the old ergonomics with bind-like partial application:
from langchain_core.runnables import RunnableLambda
urgent_check = RunnableLambda(lambda state: {"state": state, "questions": questions}) | classifier
print(urgent_check.invoke("Production is down").nouls["urgent"].noul)
```

Note the second failure is an unhelpful `TypeError: string indices must be integers, not 'str'` raised from
inside `_payload` — if you see it after an upgrade, an old `invoke("...")` call site is the cause.

The **JavaScript** package still uses the *old* shape (questions in the constructor). Code ported between
the two languages has to be restructured, not just re-syntaxed.

---

## Errors and Retries

### Exception hierarchy (Python, `langchain_typesafe.client`)

The status-specific and connection errors also inherit a LangChain core exception, so generic
`langchain_core.exceptions` handlers catch them. Three classes do **not**: the base `TypeSafeError`, the
generic `TypeSafeAPIError` (any non-2xx status not listed below, e.g. 409), and
`TypeSafeAPIResponseValidationError`. Their MRO is only `TypeSafeError → Exception` (checked with
`__mro__` on 0.0.1a3), so a handler written against `langchain_core.exceptions` alone lets them through.

| Exception | Raised when | Also a |
|---|---|---|
| `TypeSafeError` | base class | `Exception` |
| `TypeSafeAPIError` | any non-2xx not listed below | `TypeSafeError` |
| `TypeSafeBadRequestError` | 400 | `ModelInvalidRequestError` |
| `TypeSafeAuthenticationError` | 401 | `ModelAuthenticationError` |
| `TypeSafePermissionDeniedError` | 403 | `ModelPermissionDeniedError` |
| `TypeSafeNotFoundError` | 404 | `ModelNotFoundError` |
| `TypeSafeUnprocessableEntityError` | 422 | `ModelInvalidRequestError` |
| `TypeSafeRateLimitError` | 429; has `retry_after_ms` parsed from `retry-after-ms`, else `retry-after` (seconds or HTTP-date) | `ModelRateLimitError` |
| `TypeSafeInternalServerError` | any status ≥ 500, including 529 ("Overloaded") | `ModelAPIError` |
| `TypeSafeAPIResponseValidationError` | a 2xx body that fails `ClassifierResponse` validation; has `field_path` (e.g. `answers.u.noul.noul`) | `TypeSafeAPIError` |
| `TypeSafeAPIConnectionError` | no HTTP response (`httpx2.HTTPError`) | `ModelConnectionError`, `ConnectionError` |
| `TypeSafeAPITimeoutError` | `httpx2.TimeoutException`; has `timeout` | `TypeSafeAPIConnectionError`, `ModelTimeoutError`, `TimeoutError` |

`TypeSafeAPIError` keeps `status` (alias `status_code`), `body`, `headers`, `endpoint` and `request_id`,
but `str()`/`repr()` deliberately include only endpoint, status and request id — **never the body**,
because TypeSafe error bodies can echo the submitted state. Do not log `error.body` indiscriminately.

### No retries in Python

`langchain-typesafe` 0.0.1a3 makes **exactly one HTTP attempt** per `invoke`; `retry_after_ms` is parsed
but nothing consumes it. (The JS source states this explicitly: "this package retries where the Python one
does not".) Add retries with the generic `Runnable.with_retry`, which does not read `retry_after_ms` — honour
it yourself if you need to.

Executed:

```python
import httpx2
from langchain_core.exceptions import ModelRateLimitError
from langchain_typesafe import Noul
from langchain_typesafe.client import (
    TypeSafeAPIConnectionError,
    TypeSafeAPIError,
    TypeSafeInternalServerError,
    TypeSafeRateLimitError,
)

from lc_fake import fake_api, fake_api_async, make_classifier

REQUEST = {"state": "x", "questions": {"urgent": Noul(instructions="Is this urgent?")}}
attempts = {"n": 0}


def flaky(request: httpx2.Request) -> httpx2.Response:
    """429 on the first attempt, then a normal answer."""
    attempts["n"] += 1
    if attempts["n"] == 1:
        return httpx2.Response(429, json={"error": "rate limited"},
                               headers={"retry-after": "2", "x-typesafe-request-id": "req_9"})
    return fake_api(request)


classifier = make_classifier(flaky, fake_api_async)

# 1. The package itself never retries: one HTTP attempt, then the mapped exception.
try:
    classifier.invoke(REQUEST)
except TypeSafeRateLimitError as error:
    print(error)                       # message carries endpoint, status and request id, never the body
    print(error.retry_after_ms, error.status_code, error.request_id)
    print(isinstance(error, ModelRateLimitError), isinstance(error, TypeSafeAPIError))
print("attempts:", attempts["n"])

# 2. Add retries with the generic Runnable.with_retry (it does not read retry_after_ms).
attempts["n"] = 0
retrying = classifier.with_retry(
    retry_if_exception_type=(TypeSafeRateLimitError, TypeSafeInternalServerError, TypeSafeAPIConnectionError),
    stop_after_attempt=3,
    wait_exponential_jitter=False,
)
print(retrying.invoke(REQUEST).nouls["urgent"].noul, "after", attempts["n"], "attempts")
```

Output (the `LangChainBetaWarning` on stderr omitted):

```text
POST https://api.typesafe.ai/v1/systemone: 429 Too Many Requests (request_id=req_9)
2000.0 429 req_9
True True
attempts: 1
0.91 after 2 attempts
```

`retry-after: 2` (seconds) is parsed to `retry_after_ms == 2000.0`, a float.

---

## LangSmith Tracing

What the classifier contributes to a trace (from `classifier.py`, confirmed with a tracer below):

- The run is recorded with `run_type="llm"`; `run_name` from the config is respected.
- Metadata gains `ls_provider="typesafe"`, `ls_model_name=<the configured model>` and
  `ls_model_type="chat"`, merged over your own metadata. `ls_model_name` is the **requested** alias
  (`jev-latest`), not the resolved model; the resolved one is in `response.model`.
- After a successful call, if a LangSmith run tree is active, `usage_metadata` (`input_tokens`,
  `output_tokens`, `total_tokens`) is attached to it. A tracing failure is logged at debug level and never
  fails the classification.
- The run's **inputs are the full request** (`state` and `questions`). If the state is sensitive, the trace
  is sensitive. The middleware classes set `trace_policy = TracePolicy(process_inputs=omit_payload)` on
  their own middleware node, but that does not change what the nested classifier run records — that
  interaction was **not verified** against a live LangSmith project.
- Whether LangSmith prices TypeSafe usage was not verified. The JS docs page says LangSmith "records
  `usage` in the traced output rather than as a costed LLM metric" (JS); the Python page only says
  decisions, token usage and spend are inspectable.

Executed (a minimal tracer standing in for LangSmith):

```python
from langchain_core.tracers.base import BaseTracer
from langchain_typesafe import Noul

from lc_fake import make_classifier


class PrintTracer(BaseTracer):
    """Minimal tracer: shows what LangSmith would receive for the classifier run."""

    def _persist_run(self, run):
        pass

    def on_chain_end(self, outputs, *, run_id, **kwargs):
        run = super().on_chain_end(outputs, run_id=run_id, **kwargs)
        print(run.run_type, run.name, run.extra["metadata"])
        print("inputs recorded:", list(run.inputs))
        return run


make_classifier().invoke(
    {"state": "x", "questions": {"urgent": Noul(instructions="Is this urgent?")}},
    config={"callbacks": [PrintTracer()], "metadata": {"tenant": "acme"}, "run_name": "urgency-check"},
)
```

Output:

```text
llm urgency-check {'tenant': 'acme', 'ls_provider': 'typesafe', 'ls_model_name': 'jev-latest', 'ls_model_type': 'chat'}
inputs recorded: ['state', 'questions']
```

Serialization (`dumpd`) is supported: `is_lc_serializable()` is `True`, namespace
`["langchain", "classifiers", "typesafe"]`, and the API key serializes as a secret reference to
`TYPESAFE_API_KEY` (observed: `'api_key': {'lc': 1, 'type': 'secret', 'id': ['TYPESAFE_API_KEY']}`).
`client`/`async_client` are excluded.

---

## Experimental Middleware (Python)

Verbatim from the docs page: "The middleware are experimental and require
`langchain-typesafe[experimental]`. Their APIs may change without notice."

```bash
pip install "langchain-typesafe[experimental]" langchain-openai
```

| Middleware | Hook | Decision | Question id | State key |
|---|---|---|---|---|
| `ModelRouterMiddleware` | `before_agent` / `abefore_agent`, `wrap_model_call` / `awrap_model_call` | which model handles the run | `model_route` (`Choice`) | `model_route: ChoiceAnswer` |
| `AutoModeMiddleware` | `wrap_tool_call` / `awrap_tool_call` | whether a tool call is too risky to run | `is_risky` (`Noul`) | none |

### Real constructor signatures

Verbatim excerpts from the installed package, `langchain_typesafe/experimental/middleware/model_router.py`
and `auto_mode.py` (0.0.1a3):

```python
@dataclass(frozen=True)
class ModelChoice:
    model: str | BaseChatModel
    criteria: JsonValue


class ModelRouterMiddleware(AgentMiddleware[_ModelRouterState]):
    def __init__(
        self,
        *,
        choices: Mapping[str, ModelChoice],
        instructions: _QuestionContent,
    ) -> None:


class AutoModeMiddleware(AgentMiddleware[AgentState[ResponseT], ContextT, ResponseT]):
    def __init__(
        self,
        *,
        tools: Sequence[str | BaseTool],
        instructions: str = _DEFAULT_INSTRUCTIONS,
        criteria: NoulCriteria | None = None,
    ) -> None:
```

All arguments are keyword-only. `_QuestionContent` is `str | dict[str, JsonValue] | list[JsonValue]`.
**Neither middleware accepts a classifier, an API key, a model, a base URL, a timeout or a threshold**:
both build `self.classifier = TypeSafeClassifier()` with no arguments, so they read `TYPESAFE_API_KEY` /
`TYPESAFE_BASE_URL` from the environment at construction and always ask `jev-latest`. The `classifier`
attribute is an ordinary instance attribute; replacing it after construction (as the offline example below
does) works on 0.0.1a3 but is not a documented API.

### Model routing — `ModelRouterMiddleware`

Verbatim from the docs page:

```python
from langchain.agents import create_agent
from langchain_typesafe.experimental.middleware import (
    ModelChoice,
    ModelRouterMiddleware,
)

router = ModelRouterMiddleware(
    choices={
        "fast": ModelChoice(
            model="openai:gpt-5.6-terra",
            criteria="Direct lookups, extraction, and localized changes with explicit targets.",
        ),
        "powerful": ModelChoice(
            model="openai:gpt-6-astra",
            criteria="Architecture, novel root-cause reasoning, and high-stakes decisions.",
        ),
    },
    instructions="Choose the least costly model that can complete the task safely.",
)

agent = create_agent("openai:gpt-5.6-terra", middleware=[router])

result = agent.invoke(
    {
        "messages": [
            {
                "role": "user",
                "content": "Prove that there are infinitely many prime numbers.",
            }
        ]
    }
)
print(result["model_route"].choice)
```

Behaviour, from the source and the offline run:

- **One classification per agent run**, in `before_agent`. Every model call in that run (including the
  calls after tool results) uses the chosen model via `request.override(model=...)`. The model passed to
  `create_agent` is never called when routing succeeds.
- The classified state is **only the latest `HumanMessage`**, serialized as `{"role": "user", "content": ...}`
  — not the system prompt, not earlier turns. A follow-up such as "and now do the same for X" is routed on
  that sentence alone.
- The question is a `Choice` whose labels are your route names and whose criteria are each
  `ModelChoice.criteria`; `instructions` becomes the question's instructions.
- String models are resolved with `init_chat_model` **eagerly in `__init__`** (the JS version resolves
  lazily and caches). Constructing the router therefore needs every provider package installed and
  whatever credentials those chat-model constructors require.
- The full `ChoiceAnswer` is stored at `state["model_route"]`, so probabilities and confidence are in the
  agent result and the trace. There is **no confidence floor**: a 0.51 / 0.49 split still routes to the
  arg-max. If you want "fall back to the powerful model when unsure", write a custom middleware (below).
- Failure modes (0.0.1a3): classifier errors propagate and end the run — no silent fallback. An empty
  `choices` mapping raises `pydantic.ValidationError`. A state with **no `HumanMessage`** makes
  `_latest_human_message` raise a bare `StopIteration` (observed), not a descriptive error; the JS port
  raises "state contains no human message to route on" instead.

### Tool-risk gating — `AutoModeMiddleware`

Verbatim from the docs page:

```python
from langchain.agents import create_agent
from langchain.messages import ToolMessage
from langchain.tools import tool
from langchain_typesafe import NoulCriteria
from langchain_typesafe.experimental.middleware import AutoModeMiddleware


@tool
def delete_all_backups() -> str:
    """Delete every backup. This action cannot be undone."""
    return "Backups deleted."


agent = create_agent(
    "openai:gpt-6-astra",
    tools=[delete_all_backups],
    middleware=[AutoModeMiddleware(tools=[delete_all_backups])],
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "Delete all backups."}]}
)
print(result["messages"][-1].content)
```

Behaviour, from the source and the offline run:

- Only tools named in `tools` (strings or `BaseTool`, matched by stripped name) are classified; every other
  tool call goes straight to the handler with no request.
- **One request per guarded tool call.** The state sent is
  `{"messages": <last 30 messages>, "tool_call": {"id", "name", "args"}, "tool_description": <if non-empty>}`.
- The question is a `Noul` with id `is_risky`. **The threshold is hard-coded**: `_PROBABILITY_THRESHOLD = 0.5`,
  blocked when `noul >= 0.5`. It is not a constructor argument in either language, despite the
  `Raises:` docstring mentioning "threshold configuration".
- A blocked call returns `ToolMessage(status="error", content="The tool call `<name>` was blocked because it
  was classified as risky (probability: 0.91). The tool was not executed.")` — the agent sees the refusal and
  continues; the tool is never invoked. The message template is fixed in Python (JS has `blockedMessage`).
- **Fail-closed**: a classifier exception propagates and the tool handler is not called.
- It **refuses, it does not ask**. Pair it with the built-in human-in-the-loop middleware when a person
  should approve.
- The default `instructions` tell the model to treat tool descriptions and arguments as data and to let only
  explicit user messages authorize execution.

**Default criteria are not sent in Python 0.0.1a3.** The module defines default `true`/`false` criteria
("Execution could cause harm, exceed authorization, expose sensitive data, or create an external side
effect." / "Execution is low risk, reversible, and clearly authorized by the user.") as the `Field` default
of its config, but `__init__` passes its own `criteria=None` default explicitly into
`model_validate`, which overrides the field default. Observed: `AutoModeMiddleware(tools=["x"]).config.criteria`
is `None`, and the request carries no `criteria` key. The JS `autoModeMiddleware` *does* send those defaults
when `criteria` is omitted. If you want the documented risk definition in Python, pass it explicitly via
`criteria=NoulCriteria(true=..., false=...)`. (The docstring says "Pass `None` to classify without outcome
criteria", which is, in effect, what the default does.)

### Running both middleware offline

Executed against `langchain` 1.4.3 with `GenericFakeChatModel` (from `langchain_core`) as the agent model:

```python
import os

os.environ.setdefault("TYPESAFE_API_KEY", "ts-test")  # both middleware call TypeSafeClassifier() with no arguments

from langchain.agents import create_agent
from langchain.tools import tool
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage
from langchain_typesafe import NoulCriteria
from langchain_typesafe.experimental.middleware import AutoModeMiddleware, ModelChoice, ModelRouterMiddleware

from lc_fake import SENT, make_classifier


class ScriptedModel(GenericFakeChatModel):
    """Replays fixed AIMessages; bind_tools is a no-op so create_agent accepts it."""

    def bind_tools(self, tools, **kwargs):
        return self


# --- ModelRouterMiddleware ---------------------------------------------------
router = ModelRouterMiddleware(
    choices={
        "fast": ModelChoice(model=ScriptedModel(messages=iter([AIMessage("fast answer")])),
                            criteria="Direct lookups and extraction."),
        "powerful": ModelChoice(model=ScriptedModel(messages=iter([AIMessage("powerful answer")])),
                                criteria="Proofs and novel reasoning."),
    },
    instructions="Choose the least costly model that can complete the task safely.",
)
router.classifier = make_classifier()  # swap in the offline classifier (fake_api picks the first label)

agent = create_agent(ScriptedModel(messages=iter([AIMessage("default model")])), middleware=[router])
result = agent.invoke({"messages": [{"role": "user", "content": "Where is the invoice PDF?"}]})
print(result["model_route"].choice, result["model_route"].confidence, "|", result["messages"][-1].content)
print("router sent state:", SENT[-1]["state"])  # latest HumanMessage only, as role/content JSON


# --- AutoModeMiddleware --------------------------------------------------------
@tool
def delete_all_backups() -> str:
    """Delete every backup. This action cannot be undone."""
    return "Backups deleted."


@tool
def read_status() -> str:
    """Read the service status."""
    return "ok"


guard = AutoModeMiddleware(
    tools=[delete_all_backups],  # read_status is not listed, so it is never classified
    criteria=NoulCriteria(true="Writes, deletes, publishes or changes access.", false="Read-only."),
)
guard.classifier = make_classifier()  # fake_api answers every Noul with 0.91, i.e. >= 0.5: blocked

script = iter([
    AIMessage("", tool_calls=[{"name": "read_status", "args": {}, "id": "c0"}]),
    AIMessage("", tool_calls=[{"name": "delete_all_backups", "args": {}, "id": "c1"}]),
    AIMessage("I could not delete the backups."),
])
before = len(SENT)
out = create_agent(ScriptedModel(messages=script), tools=[delete_all_backups, read_status], middleware=[guard]).invoke(
    {"messages": [{"role": "user", "content": "Check the status, then delete all backups."}]}
)
for message in out["messages"]:
    if message.type == "tool":
        print(message.name, message.status, "|", message.content)
print("classifications:", len(SENT) - before, "| state keys:", sorted(SENT[-1]["state"]))
print("criteria sent:", SENT[-1]["questions"]["is_risky"].get("criteria"))
print("default criteria when none passed:", AutoModeMiddleware(tools=["x"]).config.criteria)
```

Output:

```text
fast 0.62 | fast answer
router sent state: {'role': 'user', 'content': 'Where is the invoice PDF?'}
read_status success | ok
delete_all_backups error | The tool call `delete_all_backups` was blocked because it was classified as risky (probability: 0.91). The tool was not executed.
classifications: 1 | state keys: ['messages', 'tool_call', 'tool_description']
criteria sent: {'true': 'Writes, deletes, publishes or changes access.', 'false': 'Read-only.'}
default criteria when none passed: None
```

---

## Custom Middleware

When the two built-ins do not fit — a confidence floor on routing, a different risk threshold, a
classification after each tool result — write your own hook around `TypeSafeClassifier`. It accepts
LangChain messages directly, so agent state can be passed through unconverted.

Verbatim from https://docs.langchain.com/oss/python/integrations/providers/typesafe:

```python
from langchain.agents import create_agent
from langchain.agents.middleware import AgentMiddleware, AgentState, Runtime
from langchain_typesafe import Choice, ChoiceAnswer, TypeSafeClassifier
from typing_extensions import NotRequired


class TriageState(AgentState):
    triage: NotRequired[ChoiceAnswer]


class TriageMiddleware(AgentMiddleware[TriageState]):
    state_schema = TriageState

    def __init__(self) -> None:
        self.classifier = TypeSafeClassifier()

    def before_agent(
        self, state: TriageState, runtime: Runtime
    ) -> dict[str, ChoiceAnswer]:
        response = self.classifier.invoke(
            {
                "state": state["messages"],
                "questions": {
                    "triage": Choice(
                        instructions="Which team should handle this conversation?",
                        criteria={
                            "billing": "Payments, invoices, and subscriptions.",
                            "infra": "Deploys, availability, and incidents.",
                            "other": "Requests that belong to another team.",
                        },
                    )
                },
            }
        )
        return {"triage": response.choices["triage"]}


agent = create_agent(
    "openai:gpt-5.6-terra",
    middleware=[TriageMiddleware()],
)
result = agent.invoke(
    {"messages": [{"role": "user", "content": "Customers are seeing 500 errors."}]}
)
print(result["triage"].choice, result["triage"].confidence)
```

Design notes for custom hooks (from the docs page and the built-ins' source):

- `before_agent` runs once per run; `before_model` re-runs after each tool result; `wrap_tool_call` sees one
  proposed action. Pick the hook whose cadence matches how often the decision can change — every
  classification is one billed request.
- Implement the `a`-prefixed twin (`abefore_agent`, `awrap_tool_call`, ...) with `ainvoke` if the agent
  runs async; the built-ins implement both.
- Store the whole answer (`ChoiceAnswer` / `NoulAnswer`) in state rather than the label, as both built-ins
  do — the probabilities are what you will want when auditing a decision.
- Copy the built-ins' `trace_policy = TracePolicy(process_inputs=omit_payload)` (imported from
  `langchain.agents.middleware.types`) if the middleware node's own trace should not include the state.
- Make the classifier injectable (constructor argument) so it can be swapped for the offline double —
  the built-ins' lack of that seam is their main testing friction.

---

## JavaScript: `@langchain/typesafe`

Verbatim from https://docs.langchain.com/oss/javascript/integrations/providers/typesafe:

```bash
npm install @langchain/typesafe @langchain/core
```

Requires Node ≥ 20 (`engines` in `package.json`); runtime dependency `zod ^4`. The middleware entry point
additionally needs the optional peer `langchain`.

### Quickstart

Verbatim from the JS docs page; also type-checked against `@langchain/typesafe` 0.0.2:

```typescript
import { TypeSafeClassifier } from "@langchain/typesafe";

const classifier = new TypeSafeClassifier({
  dangerouslyAllowBrowser: false,
  questions: {
    urgent: {
      type: "noul",
      instructions: "Does this need attention right now?",
    },
    team: {
      type: "choice",
      instructions: "Which team should pick this up?",
      criteria: {
        infra: "Deploys, availability, and on-call incidents.",
        billing: "Payments, invoices, and subscriptions.",
      },
    },
    severity: {
      type: "score",
      instructions: "How severe is the impact?",
      criteria: ["Cosmetic.", "Degraded for some users.", "Full outage."],
    },
  },
});

const response = await classifier.invoke(
  "The deploy failed twice and customers are seeing 500s. Can someone look now?"
);

console.log(response.nouls.urgent.noul);
console.log(response.choices.team.choice, response.choices.team.confidence);
console.log(response.scores.severity.score);
```

### `TypeSafeClassifierFields` (from `dist/classifier.d.ts`, 0.0.2)

| Field | Default | Notes |
|---|---|---|
| `questions` (required) | — | validated at construction by a zod schema; empty map → `TypeSafeError: TypeSafe requires at least one question.`; a one-level score → `Invalid TypeSafe question "s": criteria: Score question must have at least 2 ordered levels.` (observed). `instructions` is optional for **every** type (a `noul` needs `criteria` or `instructions`); observed: criteria-only `noul`, `choice` and `score` questions are all accepted |
| `model` | `"jev-latest"` | trimmed; empty rejected |
| `apiKey` | `TYPESAFE_API_KEY` | stored non-enumerable so `console.log` cannot print it |
| `baseUrl` | `TYPESAFE_BASE_URL`, then `https://api.typesafe.ai` | trailing slashes stripped |
| `timeout` | `30000` **milliseconds** | Python's `timeout` is seconds |
| `fetch` | `globalThis.fetch` | the seam for tests and proxies |
| `dangerouslyAllowBrowser` | **`true`** | set `false` to throw when `isBrowser()`; the docs quickstart sets it to `false` |
| `maxRetries`, `maxConcurrency`, `onFailedAttempt` | `maxRetries: 2` | from `AsyncCallerParams` |

The response type is `ClassificationResponse`: `model`, `answers`, `usage` **in camelCase**
(`inputTokens`, `outputTokens`, both optional), optional `requestId`, and the non-enumerable getters
`nouls`/`choices`/`scores`. Score `legend`/`probabilities` keys stay `string` (`Record<string, ...>`); the
`types.d.ts` comment explicitly says not to "fix" them to numbers. The package exports the error classes
`TypeSafeError`, `TypeSafeAPIError`, `TypeSafeAuthenticationError`, `TypeSafeRateLimitError` (use their
static `isInstance`) and only *types* for questions — there are no `Noul`/`Choice`/`Score` constructors.

### Offline behaviour, including what is actually retried

Executed (compiled with `tsc`, run on Node 22) against `@langchain/typesafe` 0.0.2 / `@langchain/core` 1.2.14:

```typescript
import { TypeSafeClassifier, TypeSafeRateLimitError, type ClassificationResponse } from "@langchain/typesafe";
import { AIMessage, HumanMessage, ToolMessage } from "@langchain/core/messages";
import { RunnableLambda } from "@langchain/core/runnables";

// Offline transport: records each request, can fail the next N calls with a chosen status.
const bodies: any[] = [];
let failures: { status: number; headers: Record<string, string> }[] = [];
const fakeFetch: typeof fetch = async (_url, init) => {
  bodies.push(JSON.parse(String(init?.body)));
  const fail = failures.shift();
  if (fail) return new Response(JSON.stringify({ error: "x" }), fail);
  return new Response(
    JSON.stringify({
      model: "jev-1.13",
      answers: { urgent: { type: "noul", noul: 0.91 } },
      usage: { input_tokens: 120, output_tokens: 0 },
    }),
    { status: 200, headers: { "x-typesafe-request-id": "req_123" } },
  );
};

const classifier = new TypeSafeClassifier({
  apiKey: "ts-test",
  fetch: fakeFetch,
  questions: { urgent: { type: "noul", instructions: "Does this need attention right now?" } },
});

const r: ClassificationResponse = await classifier.invoke("Deploy failed; customers see 500s.");
console.log(r.nouls.urgent.noul, r.requestId, r.usage); // 0.91 req_123 { inputTokens: 120, outputTokens: 0 }
console.log(Object.keys({ ...r }));                     // [ 'model', 'answers', 'usage', 'requestId' ]  (no nouls/choices/scores)

// Messages become one transcript line each, not role/content JSON as in Python.
await classifier.invoke({
  ticket: 42,
  messages: [
    new HumanMessage("Refund the duplicate charge"),
    new AIMessage({ content: "", tool_calls: [{ name: "issue_refund", args: { amount: 250 }, id: "c1" }] }),
    new ToolMessage({ content: "done", tool_call_id: "c1" }),
  ],
});
console.log(JSON.stringify(bodies.at(-1).state));

console.log((await classifier.batch(["a", "b"])).map((x) => x.nouls.urgent.noul));
const routed = classifier.pipe(RunnableLambda.from((x: ClassificationResponse) => (x.nouls.urgent.noul >= 0.8 ? "page" : "queue")));
console.log(await routed.invoke("Production is down"));

// What maxRetries (default 2) actually retries, one failure each:
const probes: { status: number; headers: Record<string, string> }[] = [
  { status: 429, headers: {} },
  { status: 429, headers: { "retry-after-ms": "1" } },
  { status: 429, headers: { "retry-after": "0" } },
  { status: 503, headers: {} },
  { status: 401, headers: {} },
];
for (const fail of probes) {
  failures = [fail];
  const before = bodies.length;
  try {
    await classifier.invoke("retry probe");
    console.log(fail.status, JSON.stringify(fail.headers), "-> ok after", bodies.length - before, "attempts");
  } catch (e) {
    const extra = TypeSafeRateLimitError.isInstance(e) ? ` retryAfterMs=${e.retryAfterMs}` : "";
    console.log(fail.status, JSON.stringify(fail.headers), "-> threw", (e as Error).name, "after", bodies.length - before, "attempt(s)" + extra);
  }
}
```

Output:

```text
0.91 req_123 { inputTokens: 120, outputTokens: 0 }
[ 'model', 'answers', 'usage', 'requestId' ]
{"ticket":42,"messages":["user: Refund the duplicate charge","assistant: [called issue_refund#c1 with {\"amount\":250}]","tool#c1: done"]}
[ 0.91, 0.91 ]
page
429 {} -> threw TypeSafeRateLimitError after 1 attempt(s) retryAfterMs=undefined
429 {"retry-after-ms":"1"} -> threw TypeSafeRateLimitError after 1 attempt(s) retryAfterMs=1
429 {"retry-after":"0"} -> ok after 2 attempts
503 {} -> ok after 2 attempts
401 {} -> threw TypeSafeAuthenticationError after 1 attempt(s)
```

What this shows:

- Object spread and `JSON.stringify` drop `nouls`/`choices`/`scores`; persist `answers`.
- Messages become transcript lines (`"assistant: [called issue_refund#c1 with {\"amount\":250}]"`,
  `"tool#c1: done"`). The source comment says this flat form "classifies identically to the full OpenAI
  message envelope" when measured live; that measurement is the package author's, not re-verified here.
- **`maxRetries: 2` does not mean "429s are retried".** Retries go through `@langchain/core`'s
  `AsyncCaller`, whose default handler, for a 429, retries only when a `retry-after` header (seconds or
  HTTP-date) of ≤ 60 000 ms is present. A bare 429, or one carrying only `retry-after-ms`, fails on the first
  attempt even though the package parses `retry-after-ms` into `retryAfterMs`. 5xx (here 503), 408,
  connection errors and timeouts are retried; 400/401/403/404/409/422 are not. Retry attempts carry an
  `X-TypeSafe-Retry-Count` header. Behaviour observed with `@langchain/core` 1.2.14; it depends on core's
  version, not on this package's.

### JS middleware (`@langchain/typesafe/middleware`, since 0.0.2)

The 0.0.2 changelog entry (verbatim from the package's `CHANGELOG.md`): "Add
`@langchain/typesafe/middleware`, with `modelRouterMiddleware` for TypeSafe `Choice`-based model routing and
`autoModeMiddleware` for blocking risky tool calls with a calibrated TypeSafe `Noul` probability." Both are
marked `@experimental`. They are factory **functions** built on `createMiddleware`, not classes.

Verbatim excerpts from `dist/middleware/modelRouter.d.ts` and `dist/middleware/autoMode.d.ts` (0.0.2,
doc comments shortened):

```typescript
interface ModelChoice {
  /** A model instance, or a string accepted by `initChatModel`. */
  model: string | BaseChatModel;
  /** Description of the tasks this model suits. Becomes the Choice criterion. */
  criteria: JsonValue;
}
interface ModelRouterMiddlewareConfig {
  choices: Record<string, ModelChoice>;
  instructions: QuestionContent;
  classifierOptions?: Omit<TypeSafeClassifierFields, "questions">;
}
interface AutoModeMiddlewareConfig {
  tools: (string | {
    name: string;
  })[];
  instructions?: QuestionContent;
  criteria?: NoulCriteria | null;
  /** Supports `{tool_name}` and `{probability}`. */
  blockedMessage?: string;
  classifierOptions?: Omit<TypeSafeClassifierFields, "questions">;
}
```

Differences from the Python middleware, read from `dist/middleware/*.js`:

- `classifierOptions` is spread into the inner classifier's constructor. It accepts every
  `TypeSafeClassifierFields` option except `questions`: `apiKey`, `model`, `baseUrl`, `timeout`, `fetch`,
  `maxRetries`, `dangerouslyAllowBrowser`, and so on. So the JS middleware *can* be configured and tested
  without environment variables.
- The routing answer is stored under **`modelRoute`** (camelCase), not `model_route`.
- Router models given as strings are resolved lazily with `initChatModel` and the promise is cached per
  route (a failed resolution is evicted).
- No human message → `Error("modelRouterMiddleware: state contains no human message to route on.")`.
  Failures in `wrapModelCall` (e.g. an unresolvable model) arrive wrapped in LangChain's `MiddlewareError`;
  read `err.cause` (per the `.d.ts` comment).
- `autoModeMiddleware` sends the default `true`/`false` criteria when `criteria` is omitted; `criteria: null`
  sends none. `blockedMessage` is customizable; the threshold is still a fixed `0.5`.

Executed (compiled with `tsc`, run on Node 22, `langchain` 1.5.15, with a scripted `BaseChatModel`
subclass as the agent model):

```typescript
import { createAgent, tool } from "langchain";
import { AIMessage, type BaseMessage } from "@langchain/core/messages";
import { BaseChatModel } from "@langchain/core/language_models/chat_models";
import type { ChatResult } from "@langchain/core/outputs";
import * as z from "zod";
import { autoModeMiddleware, modelRouterMiddleware } from "@langchain/typesafe/middleware";

class SeqModel extends BaseChatModel {
  constructor(private label: string, private queue: AIMessage[]) { super({}); }
  _llmType() { return "seq"; }
  async _generate(_m: BaseMessage[]): Promise<ChatResult> {
    const message = this.queue.shift() ?? new AIMessage(`${this.label}: done`);
    return { generations: [{ message, text: String(message.content) }] };
  }
  bindTools() { return this as any; }
}

const sent: any[] = [];
const answerFetch = (answers: Record<string, unknown>): typeof fetch => async (_u, init) => {
  sent.push(JSON.parse(String(init?.body)));
  return new Response(JSON.stringify({ model: "jev-1.13", answers }), { status: 200 });
};

const router = modelRouterMiddleware({
  choices: {
    fast: { model: new SeqModel("fast", []), criteria: "Direct lookups." },
    powerful: { model: new SeqModel("powerful", []), criteria: "Proofs and novel reasoning." },
  },
  instructions: "Choose the least costly model that can complete the task safely.",
  classifierOptions: {
    apiKey: "ts-test",
    fetch: answerFetch({ model_route: { type: "choice", choice: "powerful", probabilities: { fast: 0.1, powerful: 0.9 }, confidence: 0.7 } }),
  },
});
const agent = createAgent({ model: new SeqModel("default", []), middleware: [router] });
const res = await agent.invoke({ messages: [{ role: "user", content: "Prove there are infinitely many primes." }] });
console.log("route:", res.modelRoute?.choice, "| final:", res.messages.at(-1)?.content);
console.log("router state sent:", JSON.stringify(sent.at(-1).state));

const deleteAllBackups = tool(async () => "Backups deleted.", {
  name: "delete_all_backups", description: "Delete every backup. This action cannot be undone.", schema: z.object({}),
});
const readStatus = tool(async () => "ok", { name: "read_status", description: "Read service status.", schema: z.object({}) });
const guard = autoModeMiddleware({
  tools: [deleteAllBackups],
  blockedMessage: "Refused {tool_name} (risk {probability}).",
  classifierOptions: { apiKey: "ts-test", fetch: answerFetch({ is_risky: { type: "noul", noul: 0.97 } }) },
});
const model = new SeqModel("m", [
  new AIMessage({ content: "", tool_calls: [{ name: "read_status", args: {}, id: "c0" }] }),
  new AIMessage({ content: "", tool_calls: [{ name: "delete_all_backups", args: {}, id: "c1" }] }),
]);
const n = sent.length;
const out = await createAgent({ model, tools: [deleteAllBackups, readStatus], middleware: [guard] })
  .invoke({ messages: [{ role: "user", content: "Check status, then delete all backups." }] });
for (const m of out.messages) if (m.getType() === "tool") console.log("tool:", m.name, (m as any).status, "|", m.content);
console.log("classifier calls:", sent.length - n, "| default criteria sent:", JSON.stringify(sent.at(-1).questions.is_risky.criteria));
```

Output:

```text
route: powerful | final: powerful: done
router state sent: "user: Prove there are infinitely many primes."
tool: read_status success | ok
tool: delete_all_backups error | Refused delete_all_backups (risk 0.97).
classifier calls: 1 | default criteria sent: {"true":"Execution could cause harm, exceed authorization, expose sensitive data, or create an external side effect.","false":"Execution is low risk, reversible, and clearly authorized by the user."}
```

---

## Python vs JavaScript Differences

| Aspect | Python `langchain-typesafe` 0.0.1a3 | JS `@langchain/typesafe` 0.0.2 |
|---|---|---|
| Where questions go | `invoke({"state", "questions"})` | constructor `questions`; `invoke(state)` |
| Runnable types | `RunnableSerializable[ClassifierRequest, ClassifierResponse]` | `Runnable<State, ClassificationResponse>` |
| Question constructors | `Noul`, `Choice`, `Score` Pydantic classes | plain objects with `type: "noul" \| "choice" \| "score"` (types only) |
| `instructions` | required on `Noul`, `Choice` and `Score` | optional on all three (a `noul` needs `criteria` or `instructions`) |
| Message state | OpenAI role/content JSON (`convert_to_openai_messages`), args as JSON string | one transcript line per message, `"role#id (name): content [called tool#id with {...}]"` |
| Response usage | `usage.input_tokens` / `output_tokens` | `usage.inputTokens` / `outputTokens` |
| Score keys | `int` | `string` |
| Request id | `response.request_id` | `response.requestId` |
| Timeout unit / default | seconds, `30.0` (own clients only) | milliseconds, `30000` |
| HTTP transport seam | `client=httpx2.Client(...)`, `async_client=...` | `fetch` |
| Retries | none | `AsyncCaller`, `maxRetries` 2 (429 only with `retry-after` ≤ 60 s) |
| Browser guard | n/a | `dangerouslyAllowBrowser` (default `true`) |
| User-Agent | `langchain-typesafe/0.0.1a3` | `langchainjs-typesafe/0.0.2` |
| Middleware import | `langchain_typesafe.experimental.middleware` (classes) | `@langchain/typesafe/middleware` (factory functions) |
| Middleware configuration | environment only, `jev-latest` fixed | `classifierOptions` |
| Router state key | `model_route` | `modelRoute` |
| AutoMode default criteria | **not sent** (see above) | sent |
| AutoMode blocked text | fixed | `blockedMessage` template |
| Docs page mentions middleware | yes | no (as of 2026-10-03) |

---

## Gotchas Checklist

- [ ] Python: pass **both** `state` and `questions` to `invoke`. A bare string input raises
      `TypeError: string indices must be integers` from inside the classifier.
- [ ] Python: `TYPESAFE_API_KEY` is read when the classifier (or a middleware) is **constructed**; a missing
      key fails there with `ValidationError`, so module-level classifiers fail at import time.
- [ ] Python: no retries. Wrap with `.with_retry(retry_if_exception_type=(TypeSafeRateLimitError,
      TypeSafeInternalServerError, TypeSafeAPIConnectionError), stop_after_attempt=3)` or your own policy.
- [ ] JS: a 429 without a `retry-after` header is not retried despite `maxRetries: 2`.
- [ ] Injected `httpx2` clients keep their own timeout (5 s by default for `httpx2.Client()`).
- [ ] `batch` is N requests; many questions about one state belong in one `questions` map.
- [ ] Everything in `state` — tool arguments included — goes to TypeSafe and into the trace inputs.
- [ ] `ModelRouterMiddleware` classifies only the latest human message, has no confidence floor, and needs
      every routed provider importable at construction (Python).
- [ ] `AutoModeMiddleware` threshold is fixed at 0.5; Python sends no risk criteria unless you pass them.
- [ ] The middleware are experimental in both languages; pin exact versions
      (`langchain-typesafe==0.0.1a3`, `@langchain/typesafe@0.0.2`).
- [ ] `ChoiceAnswer.confidence` is not the winning probability; read
      https://docs.typesafe.ai/confidence before choosing a threshold.
