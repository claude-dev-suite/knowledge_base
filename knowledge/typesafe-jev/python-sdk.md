# TypeSafe Python SDK (`typesafe-sdk`) - Complete Reference

> Official Documentation: https://docs.typesafe.ai/sdk/python
> Usage guide: https://docs.typesafe.ai/sdk/python/usage
> API reference: https://docs.typesafe.ai/sdk/python/api (clients, types, retries, exceptions, constants)
> Changelog: https://docs.typesafe.ai/sdk/python/changelog
> Source: https://github.com/typesafe-ai/typesafe-sdk-python
> PyPI: https://pypi.org/project/typesafe-sdk/
> Last verified: 2026-10-03
> Verified against: `typesafe-sdk` 0.7.2 (latest on PyPI on 2026-10-03), `httpx2` 2.13.1, `pydantic` 2.13.5, `pydantic-core` 2.46.5, `tenacity` 9.1.4, CPython 3.12.10 (Windows). Model numbers refer to `jev-1.13.0`, the target of the `jev-latest` alias on 2026-10-03.

## Overview

`typesafe-sdk` is the official Python client for the TypeSafe API: `POST /v1/systemone` (ask named
Noul / Choice / Score questions about a `state`) and `GET /v1/models`. It ships a synchronous
`TypeSafeClient` and an asynchronous `AsyncTypeSafeClient` with the same surface, built on
`httpx2` (transport), `pydantic` v2 (request and response models) and `tenacity` (retries).

This document goes below the official usage guide. Every claim about behaviour was checked in two
ways: by reading the installed 0.7.2 source (`typesafe_sdk/_core/*.py`), and by running the code
offline against `httpx2.MockTransport`. No real API call was made, so nothing here says how the
live service behaves beyond what its documentation says.

How code blocks are labelled:

- **Verbatim** blocks have their source URL directly above them.
- **Executed** blocks were run offline on 2026-10-03. Where the output matters, it is shown.
- Anything that could not be verified says so explicitly.

Highlights that the official guide does not spell out (all executed):

| Behaviour | Consequence |
|---|---|
| `TypeSafeAPIConnectionError` / `TypeSafeAPITimeoutError` do **not** subclass `TypeSafeAPIError` | the guide's `except TypeSafeAPIError` lets timeouts escape |
| `RetryPolicy.timeout` (30 s) is a budget checked **between** attempts, not a deadline | wall clock can reach budget + one attempt timeout |
| A `Retry-After` larger than the remaining budget makes the SDK give up **immediately** | a `429` with `retry-after: 60` raises at once under defaults |
| `Retry-After` is not capped by `backoff_max` | with `timeout=None` the SDK sleeps as long as the server asks |
| Passing `http_client=` without `timeout=` inherits the client's timeout | `httpx2.AsyncClient(http2=True)` silently drops the per-attempt timeout from 10 s to 5 s |
| `extra_body` keys override `state` / `model` / `questions`; `extra_headers` **cannot** override `Authorization` | forward-compat is safe for auth, but not for the body |
| Unknown answer kinds are dropped with a `logging.warning` on a logger that has a `NullHandler` | invisible unless you configure logging |
| `response_model` as a plain `BaseModel` skips the unknown-kind filter | `dict[str, Answer]` breaks the day the API adds an answer kind |
| `.request_id` **raises** `TypeSafeError` if the header is missing | mock transports must send `x-typesafe-request-id` |
| `Noul`/`Choice`/`Score` use `extra="forbid"` | new API question fields need raw dicts, not kwargs |

---

## Table of Contents

1. [Installation and Versions](#installation-and-versions)
2. [Public Surface (every exported name)](#public-surface-every-exported-name)
3. [Clients: Construction and Configuration](#clients-construction-and-configuration)
4. [Environment Variables and API Key Validation](#environment-variables-and-api-key-validation)
5. [Calling `system_one`](#calling-system_one)
6. [Question Types and Raw Dicts](#question-types-and-raw-dicts)
7. [What Goes on the Wire](#what-goes-on-the-wire)
8. [Responses, Typed Views and Int-Keyed Score Maps](#responses-typed-views-and-int-keyed-score-maps)
9. [`response_model`: Both Forms and Their Edge Cases](#response_model-both-forms-and-their-edge-cases)
10. [Listing Models](#listing-models)
11. [Retries: `RetryPolicy` Semantics](#retries-retrypolicy-semantics)
12. [Exceptions: Hierarchy, Attributes, Mapping](#exceptions-hierarchy-attributes-mapping)
13. [HTTP/2 and Connection Pooling](#http2-and-connection-pooling)
14. [Logging and Redaction](#logging-and-redaction)
15. [Forward Compatibility: `extra_body`, `extra_headers`, Raw Dicts, Unknown Answers](#forward-compatibility-extra_body-extra_headers-raw-dicts-unknown-answers)
16. [Gateways](#gateways)
17. [Serialization and Picklability](#serialization-and-picklability)
18. [Production Pattern: Testing With a Mock Transport](#production-pattern-testing-with-a-mock-transport)
19. [Production Pattern: Bounded Concurrency Under the Rate Limit](#production-pattern-bounded-concurrency-under-the-rate-limit)
20. [Production Pattern: Packing Many Questions per State](#production-pattern-packing-many-questions-per-state)
21. [Production Pattern: Fallback Wrapper](#production-pattern-fallback-wrapper)
22. [Changelog and Breaking Changes](#changelog-and-breaking-changes)
23. [Unverified / Open Points](#unverified--open-points)

---

## Installation and Versions

Source: https://docs.typesafe.ai/sdk/python

```sh
uv add typesafe-sdk
```

Source: https://docs.typesafe.ai/sdk/python

```sh
pip install typesafe-sdk
```

The `http2` extra (`'typesafe-sdk[http2]'`, added in 0.7.2) adds `httpx2[http2]` (the `h2` package).

Package metadata of 0.7.2, read from the installed `METADATA` file:

| Field | Value |
|---|---|
| Python | `>=3.10` (classifiers list 3.10 to 3.14) |
| Runtime deps | `httpx2>=2.0.0`, `pydantic>=2.12.0`, `pydantic-core>=2.41.1`, `tenacity>=9.0.0`, `typing-extensions>=4.13.0` |
| Extra `http2` | `httpx2[http2]>=2.0.0` |
| License | MIT |
| Status classifier | `Development Status :: 5 - Production/Stable` |

PyPI release history (https://pypi.org/pypi/typesafe-sdk/json, read 2026-10-03): `0.0.1a0`
(2026-09-09), `0.5.7` (2026-09-11), `0.6.0` (2026-09-15), `0.7.0` (2026-09-18), `0.7.1`
(2026-09-21), `0.7.2` (2026-09-26). The changelog dates 0.5.7 to 2026-09-14, while PyPI shows an
upload on 2026-09-11. Both figures are given as their sources state them.

**Pin the minor version** (`typesafe-sdk~=0.7.2`). Breaking changes have shipped in two minor releases
within a week (see [Changelog](#changelog-and-breaking-changes)), even though the package declares
itself Production/Stable.

---

## Public Surface (every exported name)

`typesafe_sdk.__all__` in 0.7.2, grouped. The list comes from introspection.

| Group | Names |
|---|---|
| Clients | `TypeSafeClient`, `AsyncTypeSafeClient` |
| Model resources | `Models` (sync, `client.models`), `AsyncModels` (async) |
| Question objects | `Noul`, `Choice`, `Score`, `NoulCriteria` (TypedDict) |
| Question dict types | `NoulModel`, `ChoiceModel`, `ScoreModel` (closed TypedDicts), `QuestionModel` (their union) |
| Question aliases | `Question` (= `Noul \| Choice \| Score \| QuestionModel`), `Questions` (= `Mapping[str, Question]`) |
| Answers | `NoulAnswer`, `ChoiceAnswer`, `ScoreAnswer`, `Answer` (discriminated union on `type`) |
| Responses | `SystemOneResponse`, `Usage`, `ListModelsResponse`, `ModelMetadata` |
| JSON aliases | `JSONValue`, `JSONContent` |
| Retries | `RetryPolicy` |
| Exceptions | `TypeSafeError`, `TypeSafeAPIError`, `TypeSafeBadRequestError`, `TypeSafeAuthenticationError`, `TypeSafePermissionDeniedError`, `TypeSafeNotFoundError`, `TypeSafeUnprocessableEntityError`, `TypeSafeRateLimitError`, `TypeSafeInternalServerError`, `TypeSafeAPIResponseValidationError`, `TypeSafeAPIConnectionError`, `TypeSafeAPITimeoutError` |
| Module | `constants` |

`typesafe_sdk.__version__` also exists. It is read from installed metadata and is not in `__all__`.

The signatures below are condensed from `inspect.signature` on 0.7.2. They are reformatted for line
length and simplified in three ways, so the block is a reading aid, not runnable code and not raw output:
module prefixes are dropped and `Optional[X]` is written as `X | None`; `RetryPolicy`'s two
`<factory>` defaults are written out from the `default_factory` in `_core/retry.py`; and the
`NoulCriteria` line summarises the TypedDict's annotations and class keywords
(`__total__ == False`, `__closed__ == True`). That line is not valid Python.

```python
TypeSafeClient(*, api_key: str | None = None, model: str | None = None,
               retry: RetryPolicy | None = None, timeout: float | httpx2.Timeout | None = None,
               headers: Mapping[str, str] | None = None, transport: httpx2.BaseTransport | None = None,
               http_client: httpx2.Client | None = None, base_url: str | None = None) -> None
AsyncTypeSafeClient(...same..., transport: httpx2.AsyncBaseTransport | None = None,
                    http_client: httpx2.AsyncClient | None = None, ...) -> None

TypeSafeClient.system_one(self, state: JSONContent, questions: Mapping[str, Question], *,
    model: str | None = None, retry: RetryPolicy | None = None,
    timeout: float | httpx2.Timeout | None = None, extra_headers: Mapping[str, str] | None = None,
    extra_body: Mapping[str, JSONValue | None] | None = None,
    response_model: type[ResponseT] | None = None) -> SystemOneResponse | ResponseT
Models.list(self, *, retry: RetryPolicy | None = None, timeout: float | httpx2.Timeout | None = None,
            extra_headers: Mapping[str, str] | None = None) -> ListModelsResponse

RetryPolicy(max_retries: int = 2, backoff_initial: float = 0.5, backoff_max: float = 5.0,
            backoff_jitter: float = 0.25, http_statuses: set[int] = {408, 429, *range(500, 600)},
            respect_retry_after: bool = True, api_connection_error: bool = True,
            api_timeout_error: bool = True, exceptions: set[type[BaseException]] = set(),
            predicate: Callable[[BaseException], bool] | None = None, timeout: float | None = 30.0)

Noul(*, type: Literal['noul'] = 'noul', instructions: JSONContent | None = None,
     criteria: NoulCriteria | None = None)
Choice(*, type: Literal['choice'] = 'choice', instructions: JSONContent | None = None,
       criteria: Mapping[str, JSONContent | None])
Score(*, type: Literal['score'] = 'score', instructions: JSONContent | None = None,
      criteria: Sequence[JSONContent])
NoulCriteria = TypedDict(total=False, closed=True, true=JSONContent | None, false=JSONContent | None)
```

`system_one` has two `@overload`s plus the implementation. With `response_model=None` the return type is `SystemOneResponse`.
With `response_model=type[ResponseT]` it is `ResponseT`, where `ResponseT` is bound to `pydantic.BaseModel`.

Constants (`typesafe_sdk.constants`), quoted verbatim from https://docs.typesafe.ai/sdk/python/api/constants:

```python
API_KEY_ENV = 'TYPESAFE_API_KEY'
BASE_URL_ENV = 'TYPESAFE_BASE_URL'
DEFAULT_MODEL_ENV = 'TYPESAFE_DEFAULT_MODEL'
LOG_LEVEL_ENV = 'TYPESAFE_LOG_LEVEL'
DEFAULT_BASE_URL = 'https://api.typesafe.ai'
DEFAULT_MODEL = 'jev-latest'
DEFAULT_TIMEOUT = 10.0
```

These internal constants are not public API and may change. They were read from `_core/constants.py`
in 0.7.2: paths `/v1/systemone` and `/v1/models`; headers `X-TypeSafe-SDK`, `X-TypeSafe-Runtime`,
`X-TypeSafe-Retry-Count`, response header `x-typesafe-request-id`; when no message field can be
extracted from an error body, the raw body is truncated to 200 characters in the exception message.

---

## Clients: Construction and Configuration

| Argument | Meaning | Resolution |
|---|---|---|
| `api_key` | Bearer key | explicit > `TYPESAFE_API_KEY`; validated **at construction** |
| `model` | default model for every call | explicit > `TYPESAFE_DEFAULT_MODEL` > `jev-latest` |
| `base_url` | API root | explicit > `TYPESAFE_BASE_URL` > `https://api.typesafe.ai`; trailing `/` stripped |
| `timeout` | **per-attempt** HTTP timeout (float seconds or `httpx2.Timeout`) | explicit > `http_client.timeout` if `http_client` given > `10.0` |
| `retry` | client-wide `RetryPolicy` | `None` means `RetryPolicy()` defaults |
| `headers` | extra default headers on every request | cannot override `Authorization`, `Accept`, `User-Agent`, the `X-TypeSafe-*` headers, or (on `POST`) `Content-Type` (see wire section) |
| `transport` | custom `httpx2.(Async)BaseTransport`, e.g. `MockTransport` | closed with the client |
| `http_client` | your own `httpx2.Client` / `httpx2.AsyncClient` | closed with the client; exclusive with `transport` (`ValueError`) |

Construction-time errors, all executed:

- Missing, empty or whitespace-only key raises `TypeSafeError("No API key was provided. ...")`.
- Internal whitespace or non-ASCII raises `TypeSafeError("API key must contain only printable ASCII characters without whitespace.")`.
- `timeout=0`, a negative value or `inf` raises `TypeSafeError("timeout must be a positive, finite number of seconds.")`.
- `transport=` together with `http_client=` raises `ValueError("transport and http_client are mutually exclusive.")`.

Details read from the source:

- `client.models` is a `functools.cached_property`. The resource shares the client's config, HTTP client and retry policy.
- The SDK always sends the full URL (`base_url + path`). A supplied `http_client`'s own `base_url` is ignored.
- Lifecycle: the sync client has `close()` and `with`. The async client has `aclose()` and `async with`. Both close a supplied `http_client` too, so do not share one `httpx2` client between two SDK clients that close independently.
- One client is safe to reuse across many calls and, for the async client, across many concurrent tasks.
  The per-call state (attempt counter, timing) lives in a fresh `RequestState` per call.
  The retry controller is `.copy()`-ed per call.

---

## Environment Variables and API Key Validation

| Variable | Configures | Default |
|---|---|---|
| `TYPESAFE_API_KEY` | API key (required) | none |
| `TYPESAFE_BASE_URL` | API root | `https://api.typesafe.ai` |
| `TYPESAFE_DEFAULT_MODEL` | default model | `jev-latest` |
| `TYPESAFE_LOG_LEVEL` | level of the `typesafe_sdk` logger, applied **once at import** | unset |

Source: https://docs.typesafe.ai/sdk/python/usage#environment-variables. The verified rules:

- Explicit arguments win. Environment values are `.strip()`-ed, and empty or whitespace-only values count as unset.
- **An explicitly empty `api_key=""` does not fall back to the environment.** It raises even when `TYPESAFE_API_KEY` is set. This was executed.
- Leading and trailing whitespace, including a trailing newline from a key file, is stripped: `api_key="ok\n"` is accepted. This was executed.
- `TYPESAFE_LOG_LEVEL` accepts `debug`, `info`, `warn`, `warning`, `error` and `off`, case-insensitively. `off` sets the level above `CRITICAL`.
  Because it is read at import, setting it after `import typesafe_sdk` has no effect. Any other value
  (for example `critical`) is silently ignored and leaves the level unset.
- The key is kept in a dataclass field declared with `repr=False`, so `repr(client._config)` does not print it.
  Since 0.7.1 it is also scrubbed from connection-error messages (see [Logging](#logging-and-redaction)).

---

## Calling `system_one`

Source: https://docs.typesafe.ai/sdk/python/usage#calling-the-system-one-api (Sync tab)

```python
from typesafe_sdk import Choice, Noul, Score, TypeSafeClient

client = TypeSafeClient()
state = "I was charged twice. Please help ASAP."
questions = {
    "billing": Noul(instructions="Is this about billing?"),
    "tone": Choice(
        instructions="What is the tone?", criteria={"calm": None, "angry": None}
    ),
    "urgency": Score(
        instructions="How urgent is this?", criteria=["low", "medium", "high"]
    ),
}
result = client.system_one(state, questions)
print(
    result.nouls["billing"].noul,
    result.choices["tone"].choice,
    result.scores["urgency"].score,
)
```

Source: https://docs.typesafe.ai/sdk/python/usage#calling-the-system-one-api (Async tab)

```python
import asyncio

from typesafe_sdk import AsyncTypeSafeClient, Choice, Noul, Score


async def main() -> None:
    async with AsyncTypeSafeClient() as client:
        result = await client.system_one(
            "I was charged twice. Please help ASAP.",
            {
                "billing": Noul(instructions="Is this about billing?"),
                "tone": Choice(
                    instructions="What is the tone?",
                    criteria={"calm": None, "angry": None},
                ),
                "urgency": Score(
                    instructions="How urgent is this?",
                    criteria=["low", "medium", "high"],
                ),
            },
        )
        print(
            result.nouls["billing"].noul,
            result.choices["tone"].choice,
            result.scores["urgency"].score,
        )


asyncio.run(main())
```

Per-call keyword arguments, all optional, override the client for **this call only**:
`model`, `retry`, `timeout`, `extra_headers`, `extra_body`, `response_model`.

**Client-side validation** happens before any network I/O and raises a plain `TypeSafeError`, which
is never retried. All cases were executed:

| Input | Error message |
|---|---|
| `questions={}` | `At least one question is required.` |
| `Score(criteria=[])` (or raw score dict with `[]`) | `Score question "s" has no criteria; at least one score is required.` |
| raw `{"type": "choice"}` without `criteria` | `Question "c" requires "criteria".` |
| raw dict without a non-empty string `type` | `Question "x" must be a question object or a dictionary with a nonempty string "type".` |
| `state` containing an arbitrary object, or `bytes` that are not valid UTF-8 | `The request body could not be encoded as JSON` |

Valid UTF-8 `bytes` are **not** rejected: pydantic-core encodes them as a JSON string, so
`system_one(b"hi", ...)` sends `"state":"hi"`. This was executed.

The SDK does **not** enforce the API's documented domain limits:

- A Score "should have at least two levels; the API accepts up to 10".
- A Choice allows "a maximum of 255 options".

Source for both: https://docs.typesafe.ai/api, read 2026-10-03. Breaking either limit surfaces as a
server-side `422`, raised as `TypeSafeUnprocessableEntityError`. Validate these limits in your own code
if you build questions dynamically.

---

## Question Types and Raw Dicts

| Class | Required | Optional | Notes |
|---|---|---|---|
| `Noul` | none | `instructions`, `criteria: NoulCriteria` | `NoulCriteria(true=..., false=...)`: each value is text, an object or an array |
| `Choice` | `criteria: Mapping[str, JSONContent \| None]` | `instructions` | dict order is the order sent; `None` = label interpreted by its name |
| `Score` | `criteria: Sequence[JSONContent]` | `instructions` | position = level, starting at 0 |

These are pydantic models with `extra="forbid"`. Passing a field the SDK does not model, such as
`Noul(instructions="x", weight=2)`, raises `pydantic.ValidationError` when the object is built.
To send a new API field before the SDK models it, use a raw dict
(see [Forward Compatibility](#forward-compatibility-extra_body-extra_headers-raw-dicts-unknown-answers)).

Serialization drops fields left at `None`, so an unset `instructions` is omitted from the request.
A `None` placed inside `criteria` is kept, so `{"calm": None}` goes over the wire as `{"calm": null}`.
This was executed.

`instructions` and criteria values accept `JSONContent`: a string, a mapping, or a sequence.
Structured instructions are documented at https://docs.typesafe.ai/api and
https://docs.typesafe.ai/primitives/advanced.

`JSONValue` and `JSONContent` are recursive aliases built with `typing_extensions.TypeAliasType`:

- `JSONValue = str | int | float | bool | Sequence[JSONValue | None] | Mapping[str, JSONValue | None]`
- `JSONContent = str | Mapping[str, JSONValue | None] | Sequence[JSONValue | None]`

---

## What Goes on the Wire

The request below was captured from `httpx2.MockTransport` in an executed run. The call passed
`extra_body={"model": "override-model", "beam_width": 4}` and
`extra_headers={"Authorization": "Bearer evil", "X-Trace": "t1"}`. Header names are shown
lower-cased; the transport-level headers httpx2 adds itself (`Host`, `Accept-Encoding`, `Connection`,
`Content-Length`) are omitted.

```text
POST https://api.typesafe.ai/v1/systemone
x-trace: t1
authorization: Bearer test-key            <- extra_headers could NOT override it
accept: application/json
user-agent: typesafe-sdk/0.7.2
x-typesafe-sdk: typesafe-sdk/0.7.2
x-typesafe-runtime: python/3.12.10 (win32; AMD64)
content-type: application/json

{"state":"I was charged twice.","model":"override-model",
 "questions":{"billing":{"type":"noul","instructions":"Billing?"},
              "tone":{"type":"choice","instructions":"Tone?","criteria":{"calm":null,"angry":null}},
              "urgency":{"type":"score","instructions":"Urgent?","criteria":["low","medium","high"]}},
 "beam_width":4}
```

What the capture shows:

- Header precedence is: client `headers`, then per-call `extra_headers`, then SDK-protected headers.
  The protected headers are `Authorization`, `Accept`, `User-Agent`, `X-TypeSafe-SDK` and `X-TypeSafe-Runtime`; they are set last and always win.
  On a request with a body (`POST /v1/systemone`) `Content-Type: application/json` is also set after the
  merge, so a caller's `Content-Type` is overwritten too (executed).
- A user-supplied `X-TypeSafe-Retry-Count` is removed.
- Body precedence is the opposite. `extra_body` is shallow-merged **after** `state`, `model` and `questions`, so its keys win:
  `"model"` inside `extra_body` overrode the call's model. Do not put those three keys in `extra_body` by accident.
- On every retry the SDK adds `X-TypeSafe-Retry-Count: 1`, `2`, and so on. The first attempt does not carry the header.

---

## Responses, Typed Views and Int-Keyed Score Maps

`SystemOneResponse` is a pydantic model with `extra="ignore", frozen=True, strict=True`.

| Attribute | Type | Notes |
|---|---|---|
| `model` | `str` | versioned id that answered (e.g. `jev-1.13.0`), even when you sent an alias. Log it |
| `usage` | `Usage(input_tokens: int \| None, output_tokens: int \| None)` | `None` when the API omits a count |
| `answers` | `dict[str, Answer]` | every recognised answer, keyed by your question names |
| `nouls` / `choices` / `scores` | `dict[str, NoulAnswer]` / `...ChoiceAnswer` / `...ScoreAnswer` | `cached_property` filters over `answers` |
| `request_id` | `str` | the `x-typesafe-request-id` header. **Raises `TypeSafeError` if the header was absent** |
| `raw_http_response` | `httpx2.Response` | full status, headers and body. Not part of `model_dump()` |

Answer models (`extra="ignore", frozen=True, strict=True`):

| Class | Fields |
|---|---|
| `NoulAnswer` | `type="noul"`, `noul: float` (P(yes), 0 to 1). **No `confidence`** |
| `ChoiceAnswer` | `type="choice"`, `choice: str`, `confidence: float`, `probabilities: dict[str, float]` |
| `ScoreAnswer` | `type="score"`, `score: float` (expected level, may be fractional), `confidence: float`, `legend: dict[int, ...]`, `probabilities: dict[int, float]` |

**Int-keyed score maps.** On the wire, `legend` and `probabilities` have string keys (`"0"`, `"1"`,
and so on). The SDK declares them as `dict[int, ...]`, so the keys become integers.
`model_dump(mode="json")` turns them back into strings. Executed:

```python
import httpx2

from typesafe_sdk import Choice, Noul, Score, TypeSafeClient

WIRE = {
    "model": "jev-1.13.0",
    "answers": {
        "billing": {"type": "noul", "noul": 0.97},
        "tone": {"type": "choice", "choice": "angry", "confidence": 0.81, "probabilities": {"calm": 0.1, "angry": 0.9}},
        "urgency": {
            "type": "score", "score": 1.7, "confidence": 0.6,
            "legend": {"0": "low", "1": "medium", "2": "high"},
            "probabilities": {"0": 0.1, "1": 0.1, "2": 0.8},
        },
        "ranking": {"type": "ranking", "order": ["a", "b"]},  # an answer kind this SDK does not know
    },
    "usage": {"input_tokens": 296, "output_tokens": 20},
}
transport = httpx2.MockTransport(lambda request: httpx2.Response(200, json=WIRE, headers={"x-typesafe-request-id": "req_1"}))

with TypeSafeClient(api_key="test", transport=transport) as client:
    result = client.system_one(
        "I was charged twice. Please help ASAP.",
        {
            "billing": Noul(instructions="Is this about billing?"),
            "tone": Choice(instructions="What is the tone?", criteria={"calm": None, "angry": None}),
            "urgency": Score(instructions="How urgent is this?", criteria=["low", "medium", "high"]),
        },
    )

urgency = result.scores["urgency"]
assert urgency.probabilities == {0: 0.1, 1: 0.1, 2: 0.8}       # int keys, not "0"/"1"/"2"
assert urgency.legend[2] == "high"
assert max(urgency.probabilities, key=urgency.probabilities.get) == 2
assert result.choices["tone"].confidence == 0.81
assert not hasattr(result.nouls["billing"], "confidence")       # Noul answers carry only .noul
assert set(result.answers) == {"billing", "tone", "urgency"}    # "ranking" was logged and dropped
assert "ranking" in result.raw_http_response.json()["answers"]  # ...but is still in the raw body
assert (result.model, result.request_id, result.usage.input_tokens) == ("jev-1.13.0", "req_1", 296)
print(result.model_dump(mode="json")["answers"]["urgency"]["probabilities"])  # JSON dump turns keys back into strings
# -> {'0': 0.1, '1': 0.1, '2': 0.8}
```

Further behaviour, all executed:

- Answers are frozen. Assigning `result.answers["billing"].noul = 0.1` raises `pydantic.ValidationError`.
- `score` is the probability-weighted mean of the levels, not the most likely level. For a decision,
  use `max(probabilities, key=probabilities.get)` or a threshold on `probabilities[k]`. The API reference
  describes `score` as "probability-weighted ... can land between levels" (https://docs.typesafe.ai/api).
- Strict mode applies to answers. A 200 body with a missing or wrongly typed field raises
  `TypeSafeAPIResponseValidationError` with a dotted `field_path`. For example, a Noul answer without `noul`
  reports `field_path == "answers.spam.noul"`.

---

## `response_model`: Both Forms and Their Edge Cases

Source: https://docs.typesafe.ai/sdk/python/usage#typed-system_one-responses (Sync tab)

```python
from typesafe_sdk import Noul, NoulAnswer, SystemOneResponse, TypeSafeClient


class BillingResponse(SystemOneResponse):
    billing: NoulAnswer


with TypeSafeClient() as client:
    result = client.system_one(
        "I was charged twice.",
        {"billing": Noul(instructions="Is this about billing?")},
        response_model=BillingResponse,
    )
    assert 0 <= result.billing.noul <= 1
    assert result.billing == result.nouls["billing"]
    print(result.request_id)
```

Source: https://docs.typesafe.ai/sdk/python/usage#custom-response-types (Async tab)

```python
import asyncio

from pydantic import BaseModel

from typesafe_sdk import AsyncTypeSafeClient, Noul, NoulAnswer


class BillingAnswers(BaseModel):
    billing: NoulAnswer


class BillingResponse(BaseModel):
    answers: BillingAnswers


async def main() -> None:
    async with AsyncTypeSafeClient() as client:
        result = await client.system_one(
            "I was charged twice.",
            {"billing": Noul(instructions="Is this about billing?")},
            response_model=BillingResponse,
        )
        assert 0 <= result.answers.billing.noul <= 1


asyncio.run(main())
```

How the two forms differ (read from `_core/response_types.py` and `_core/schemas/base.py`, then executed):

| | Subclass of `SystemOneResponse` | Plain `BaseModel` with `answers` |
|---|---|---|
| How it decodes | declared extra fields are **lifted** from `answers[name]` to top-level attributes | `model_validate_json` on the raw body |
| Unknown answer kinds | dropped with a warning (forward-compatible) | **not filtered**: a `dict[str, Answer]` field fails with `field_path == "answers.<name>"` |
| `nouls` / `choices` / `scores` | yes | no |
| `request_id`, `raw_http_response` | yes | no |
| Missing declared answer | `TypeSafeAPIResponseValidationError` (status 200, `field_path == "<name>"`) | same, with path `answers.<name>` |
| Int-keyed score maps | yes | yes, when you use `ScoreAnswer` |

Executed demonstration:

```python
import httpx2
from pydantic import BaseModel

from typesafe_sdk import (
    Answer, Noul, NoulAnswer, ScoreAnswer, SystemOneResponse, TypeSafeAPIResponseValidationError, TypeSafeClient,
)

WIRE = {
    "model": "jev-1.13.0",
    "answers": {
        "billing": {"type": "noul", "noul": 0.97},
        "urgency": {"type": "score", "score": 1.7, "confidence": 0.6,
                    "legend": {"0": "low", "1": "medium", "2": "high"}, "probabilities": {"0": 0.1, "1": 0.1, "2": 0.8}},
        "ranking": {"type": "ranking", "order": ["a"]},
    },
    "usage": {"input_tokens": 296, "output_tokens": 20},
}
client = TypeSafeClient(api_key="test", transport=httpx2.MockTransport(
    lambda request: httpx2.Response(200, json=WIRE, headers={"x-typesafe-request-id": "req_1"})))
questions = {"billing": Noul(instructions="Is this about billing?")}


# Form 1: subclass SystemOneResponse -> answers are lifted to attributes, views and metadata kept.
class Triage(SystemOneResponse):
    billing: NoulAnswer
    urgency: ScoreAnswer

t = client.system_one("...", questions, response_model=Triage)
assert t.billing.noul == 0.97 and t.urgency.probabilities[2] == 0.8
assert t.billing == t.nouls["billing"] and t.request_id == "req_1"


# Form 2: any BaseModel with an `answers` field -> validated straight from the JSON body.
class TriageAnswers(BaseModel):
    billing: NoulAnswer
    urgency: ScoreAnswer

class TriagePlain(BaseModel):
    answers: TriageAnswers

p = client.system_one("...", questions, response_model=TriagePlain)
assert p.answers.urgency.probabilities[2] == 0.8
assert not hasattr(p, "request_id")  # no request_id / raw_http_response on plain models


# A declared answer the API did not return -> TypeSafeAPIResponseValidationError (on a 200).
class WantsSpam(SystemOneResponse):
    spam: NoulAnswer

try:
    client.system_one("...", questions, response_model=WantsSpam)
except TypeSafeAPIResponseValidationError as error:
    assert (error.status, error.field_path) == (200, "spam")


# Plain models lose the unknown-kind filter: dict[str, Answer] fails on "ranking".
class Everything(BaseModel):
    answers: dict[str, Answer]

try:
    client.system_one("...", questions, response_model=Everything)
except TypeSafeAPIResponseValidationError as error:
    print("plain model rejected unknown kind at", error.field_path)
# -> plain model rejected unknown kind at answers.ranking
```

Recommendation: **subclass `SystemOneResponse`**. You keep forward compatibility, the typed views and
the request id. Declare an attribute only for questions you always send. A declared field that is not
in the request raises on a 200, and the API call has already been billed by then.

---

## Listing Models

Source: https://docs.typesafe.ai/models#listing-models

```python
from typesafe_sdk import TypeSafeClient

with TypeSafeClient() as client:
    for model in client.models.list().models:
        print(model.name, model.release_date, model.description)
```

- `ListModelsResponse.models` is a **tuple** of `ModelMetadata(name, description, release_date)`, all strings.
  `release_date` is `YYYY-MM-DD` and is not parsed to a `date`.
- `models.list()` accepts `retry`, `timeout` and `extra_headers`, but no `model` or `extra_body`.
- On 2026-10-03 the endpoint lists aliases only. Versioned ids such as `jev-1.13.0` are accepted
  in `model=` "whether or not they appear in the list" (https://docs.typesafe.ai/models).
  Do not validate a pinned model id against `models.list()`.

---

## Retries: `RetryPolicy` Semantics

`RetryPolicy` is a frozen dataclass, so the same instance can be shared. Pass it as `retry=` on the
client, or per call (`system_one(..., retry=...)`, `models.list(retry=...)`). A per-call policy
**replaces** the client policy entirely; fields are not merged.

Source: https://docs.typesafe.ai/sdk/python/api/retries

```python
from typesafe_sdk import RetryPolicy, TypeSafeClient

client = TypeSafeClient(
    retry=RetryPolicy(
        max_retries=3, timeout=10.0, http_statuses={429, 500, 502, 503, 504}
    )
)
```

| Field | Default | Semantics (from `_core/retry.py`) |
|---|---|---|
| `max_retries` | `2` | retries **after** the first attempt, so 3 attempts in total. `0` disables retries |
| `backoff_initial` | `0.5` | delay before retry *n* is `initial * 2**(n-1)`, capped at `backoff_max`. `0` disables backoff |
| `backoff_max` | `5.0` | cap on the exponential delay. `0` disables backoff |
| `backoff_jitter` | `0.25` | up to this fraction is randomly **subtracted** from each delay. Must be in [0, 1] |
| `http_statuses` | `{408, 429, 500..599}` | statuses that are retried. 529 "Overloaded" is included |
| `respect_retry_after` | `True` | `retry-after-ms`, then `retry-after` (seconds or HTTP date), **replaces** the backoff, uncapped |
| `api_connection_error` | `True` | retry `TypeSafeAPIConnectionError` |
| `api_timeout_error` | `True` | retry `TypeSafeAPITimeoutError` |
| `exceptions` | `set()` | extra exception types to retry |
| `predicate` | `None` | `Callable[[BaseException], bool]`. `True` retries in addition to the rules above |
| `timeout` | `30.0` | **total retry budget per call**. `None` means no budget |

The default schedule without jitter (executed through the internal `_backoff` helper) is
0.5, 1.0, 2.0, 4.0, 5.0, 5.0, ... seconds.

Invalid values raise `TypeSafeError` in `__post_init__`. This covers a negative or non-int
`max_retries`, negative or infinite backoffs, jitter outside [0, 1], and a non-positive or infinite
`timeout`. The check is `isinstance(max_retries, int)`, so a `bool` slips through (`max_retries=True`
is accepted as 1); `http_statuses`, `exceptions` and `predicate` are not validated.

### Total budget vs per-attempt timeout

There are two independent clocks:

1. **Per-attempt timeout**: the client or call `timeout` (default `DEFAULT_TIMEOUT = 10.0` s).
   `httpx2` enforces it on each attempt, and when it fires it raises `TypeSafeAPITimeoutError`.
2. **Retry budget**: `RetryPolicy.timeout` (default 30 s). It is implemented as tenacity's
   `stop_before_delay(budget)`, combined with `stop_after_attempt(max_retries + 1)`.
   After a failed attempt the SDK checks whether *elapsed time + the next delay* would reach the budget.
   If it would, the SDK stops and re-raises the last error.

The budget **never interrupts an attempt in flight**. The worst-case wall clock of one call is
therefore about budget + one per-attempt timeout. Executed with 0.4 s attempts, a 0.05 s delay and a 0.5 s budget:

```python
import time

import httpx2

from typesafe_sdk import RetryPolicy, TypeSafeClient, TypeSafeInternalServerError, Noul

attempts = []

def slow_503(request: httpx2.Request) -> httpx2.Response:
    attempts.append(time.monotonic())
    time.sleep(0.4)  # each attempt takes 0.4 s
    return httpx2.Response(503)

policy = RetryPolicy(max_retries=5, backoff_initial=0.05, backoff_max=0.05, backoff_jitter=0, timeout=0.5)
start = time.monotonic()
with TypeSafeClient(api_key="test", transport=httpx2.MockTransport(slow_503), retry=policy) as client:
    try:
        client.system_one("x", {"q": Noul()})
    except TypeSafeInternalServerError:
        pass
print(f"attempts={len(attempts)} elapsed={time.monotonic() - start:.2f}s with a 0.5 s budget")
# -> attempts=2 elapsed=0.86s with a 0.5 s budget
```

With the defaults, a call against an endpoint that hangs makes 3 attempts of 10 s each, with 0.5 s and
1 s of backoff between them. That is roughly 31.5 s before the exception reaches you; this figure is derived,
not measured against the live API. On a latency-critical path, set **both** values. For example,
`timeout=0.8` per attempt with `RetryPolicy(max_retries=1, timeout=1.5)` allows two attempts and one
default 0.5 s backoff, about 0.8 + 0.5 + 0.8 = 2.1 s in the worst case. The general bound
"budget + one attempt" (1.5 + 0.8 = 2.3 s) is looser. Both figures are derived from the code, not measured.

### `Retry-After` interactions

These were executed with a mock that always returns `429`:

- `retry-after: 60` with the default 30 s budget: **1 attempt, raised immediately**. The 60 s delay
  exceeds the budget, so tenacity stops before sleeping. `error.retry_after_ms == 60000.0`.
- `retry-after-ms: 150` with `backoff_max=0.01`: 3 attempts and about 0.31 s elapsed. The server value
  replaced the backoff and was not capped by `backoff_max`.
- With `timeout=None` and `respect_retry_after=True`, the SDK sleeps for whatever the server asks.
  If you need a cap, set `respect_retry_after=False`, or keep a finite budget.

### Customising what is retried

These cases were executed:

- `RetryPolicy(http_statuses={503})`: a `429` is raised after a single attempt.
- `RetryPolicy(predicate=lambda e: isinstance(e, TypeSafeAPIError) and e.status == 409)`: a `409` is retried.
- Client-side `TypeSafeError`s (validation, encoding, bad key) are raised before the retry loop and are never retried.
- Response-validation errors (`TypeSafeAPIResponseValidationError`, status 200) are not retried by default,
  because 200 is not in `http_statuses`.

---

## Exceptions: Hierarchy, Attributes, Mapping

This hierarchy was taken from `__mro__` on 0.7.2:

```text
Exception
└── TypeSafeError
    ├── TypeSafeAPIError                     .status .body .headers .endpoint .request_id(property)
    │   ├── TypeSafeBadRequestError              400
    │   ├── TypeSafeAuthenticationError          401
    │   ├── TypeSafePermissionDeniedError        403
    │   ├── TypeSafeNotFoundError                404
    │   ├── TypeSafeUnprocessableEntityError     422
    │   ├── TypeSafeRateLimitError               429   + .retry_after_ms (float | None)
    │   ├── TypeSafeInternalServerError          >= 500 (incl. 503, 529)
    │   └── TypeSafeAPIResponseValidationError   2xx with a bad body; + .field_path
    └── TypeSafeAPIConnectionError (also ConnectionError -> OSError)
        └── TypeSafeAPITimeoutError (also TimeoutError)   + .timeout (float | httpx2.Timeout)
```

Status mapping, executed for each status:

- Any other non-2xx status, such as 402, 408 or 409, raises the **base** `TypeSafeAPIError`.
- 408 is still retried by default, because it is in `http_statuses`.

The `TypeSafeAPIError` attributes:

| Attribute | Content |
|---|---|
| `status` | HTTP status |
| `body` | parsed JSON, or text, or `None` for an empty body |
| `headers` | `httpx2.Headers` |
| `endpoint` | `"POST https://api.typesafe.ai/v1/systemone"`, with credentials, query and fragment stripped |
| `request_id` | `x-typesafe-request-id` or `None`. Unlike the response property, it does not raise |

`str(error)` has the form `"{endpoint}: {status} {message} (request_id=...)"`. The message is taken,
in this order, from a plain-text body, or the JSON body's `error` (string), `error.message`, `message`,
`detail` (string) or `detail.message`. A FastAPI-style 422 `detail` list is flattened into `path: msg`
(the leading `body` segment dropped, entries joined with `; `). If none of these is present, the raw body
is used, truncated to 200 characters. Executed example:

`POST https://api.typesafe.ai/v1/systemone: 422 questions.u.score.criteria: List should have at least 1 item`

`repr(error)` does not include the body or the headers.

Connection failures (executed with a mock raising `httpx2.ConnectError` and `httpx2.ReadTimeout`):

- `TypeSafeAPIConnectionError("Connection error: ...")`. `__cause__` is a redacted copy of the httpx2 error; `__context__` is `None`.
- `TypeSafeAPITimeoutError`. `str()` is `Request timed out (timeout=2.5).` and `.timeout` echoes the setting.
- Neither is a `TypeSafeAPIError`. `isinstance(e, ConnectionError)` is `True` for both, and
  `isinstance(e, TimeoutError)` is `True` for the timeout.

**The gotcha.** The usage guide's error-handling example catches only `TypeSafeAPIError`.
Source: https://docs.typesafe.ai/sdk/python/usage#error-handling (Sync tab)

```python
from typesafe_sdk import TypeSafeAPIError

try:
    client.system_one(state, questions)
except TypeSafeAPIError as error:
    print(error.status, error.request_id)
```

That handler does not catch a timeout, a refused connection, or a client-side validation error.
Catch `TypeSafeError` as well, or `TypeSafeAPIConnectionError` specifically. See
[Fallback Wrapper](#production-pattern-fallback-wrapper) for a complete, executed handler.

---

## HTTP/2 and Connection Pooling

Source: https://docs.typesafe.ai/sdk/python/usage#http2 (Async tab)

```python
import httpx2

from typesafe_sdk import AsyncTypeSafeClient

client = AsyncTypeSafeClient(http_client=httpx2.AsyncClient(http2=True))
```

**Caveat (executed).** When you pass `http_client=` without `timeout=`, the SDK inherits
`http_client.timeout`. The httpx2 default is `Timeout(5.0)`, so the snippet above changes the
per-attempt timeout from 10 s to 5 s. Pass the timeout explicitly:

```python
import httpx2

from typesafe_sdk import AsyncTypeSafeClient

# A supplied http_client's own timeout (httpx2 default: 5 s) replaces the SDK's 10 s
# per-attempt default unless you pass timeout= explicitly.
implicit = AsyncTypeSafeClient(api_key="test", http_client=httpx2.AsyncClient(http2=True))
assert implicit._config.timeout == httpx2.Timeout(5.0)  # private attribute, inspected only to prove the point

client = AsyncTypeSafeClient(
    api_key="test",
    timeout=10.0,
    http_client=httpx2.AsyncClient(
        http2=True,
        limits=httpx2.Limits(max_connections=100, max_keepalive_connections=100),
    ),
)
```

Notes:

- `http2=True` needs the `h2` package. Install it with `'typesafe-sdk[http2]'`. Construction succeeded
  in the verification environment, where `h2` 4.4.1 was present. ALPN negotiation against the real
  endpoint was **not** tested.
- `httpx2.AsyncClient`'s default `limits` are `Limits(max_connections=100, max_keepalive_connections=20,
  keepalive_expiry=5.0)`, from `inspect.signature` on httpx2 2.13.1. When you run more than 20
  concurrent HTTP/1.1 requests, raise `max_keepalive_connections` to avoid connection churn, or use HTTP/2.
- The SDK's own client (no `http_client`) is `httpx2.AsyncClient(timeout=..., transport=...)` with
  httpx2's default limits.

---

## Logging and Redaction

Source: https://docs.typesafe.ai/sdk/python/usage#logging

```python
import logging

logging.getLogger("typesafe_sdk").setLevel(logging.DEBUG)
```

What the SDK emits, read from `_core/transport.py` and `_core/logging.py`:

| Level | Message |
|---|---|
| INFO | `POST <url> <- 200 in 12ms (request <id>)` on every response; `POST <url> retry N` before each retry; `POST <url> <- ReadTimeout` on transport errors |
| DEBUG | the request and response headers **and bodies**, in both directions |
| WARNING | `Ignoring answer 'x' with unrecognized type 'y'` |

The logger has a `NullHandler` and a `SensitiveHeadersFilter`, and the SDK never configures handlers.
Two consequences:

1. **Without logging configuration nothing is printed, not even the WARNING for dropped answer kinds.**
   Python's last-resort handler is bypassed because a handler exists. Configure the `typesafe_sdk` logger
   if you want to see forward-compatibility drops.
2. Header redaction applies to `authorization`, `proxy-authorization`, `x-api-key`, `api-key`, `cookie`,
   `set-cookie`, and any header whose name contains `token` or `secret` (case-insensitive).
   **Bodies are never redacted.** Executed:

```python
import logging
import sys

import httpx2

from typesafe_sdk import Noul, TypeSafeClient

logging.basicConfig(stream=sys.stdout, format="%(levelname)s %(message)s")
logging.getLogger("typesafe_sdk").setLevel(logging.DEBUG)

ok = {"model": "jev-1.13.0", "answers": {"billing": {"type": "noul", "noul": 0.9}}, "usage": {"input_tokens": 3, "output_tokens": 1}}
with TypeSafeClient(
    api_key="sk-SECRET",
    headers={"X-Session-Token": "tok"},
    transport=httpx2.MockTransport(lambda r: httpx2.Response(200, json=ok, headers={"x-typesafe-request-id": "rq"})),
) as client:
    client.system_one({"customer_ssn": "123-45-6789"}, {"billing": Noul(instructions="Billing?")})
# DEBUG POST https://api.typesafe.ai/v1/systemone -> headers={'x-session-token': '***', 'authorization': '***', ...}
#       body=b'{"state":{"customer_ssn":"123-45-6789"},...}'      <- PII in the log
# INFO POST https://api.typesafe.ai/v1/systemone <- 200 in 0ms (request rq)
# DEBUG ... <- headers={'x-typesafe-request-id': 'rq', ...} body=b'{"model":"jev-1.13.0",...}'
```

In production use `INFO` at most. `DEBUG` writes every `state`, which is your users' content, into the logs.

Exception redaction was added in 0.7.1. When a transport error occurs, the SDK rebuilds the exception
chain and replaces every occurrence of a secret header value with `***`. The replacement covers the
raw value, its `repr`, its bytes `repr` and its JSON-escaped form, and also applies to `__notes__`.
It then drops the unredacted `__context__`. Executed: an `httpx2.ConnectError("dns failure Bearer k")`
surfaced as `Connection error: dns failure ***`.

---

## Forward Compatibility: `extra_body`, `extra_headers`, Raw Dicts, Unknown Answers

Source: https://docs.typesafe.ai/sdk/python/usage#extra-request-fields (Sync tab). The docs call the
`beam_width` field "illustrative"; it is not a documented API field.

```python
from typesafe_sdk import Noul, TypeSafeClient

with TypeSafeClient() as client:
    client.system_one(
        "I was charged twice.",
        {"billing": Noul(instructions="About billing?")},
        extra_body={"beam_width": 4},
    )
```

Source: https://docs.typesafe.ai/sdk/python/usage#raw-question-dictionaries (Sync tab). `weight` is
likewise not part of the OpenAPI `NoulQuestion` schema as of 2026-10-03; it illustrates passing an
unmodelled field.

```python
from typesafe_sdk import TypeSafeClient

with TypeSafeClient() as client:
    client.system_one(
        "I was charged twice.",
        {"billing": {"type": "noul", "instructions": "About billing?", "weight": 2}},
    )
```

Raw dicts and question objects can be mixed in one mapping. The SDK checks raw dicts only for a
non-empty string `type` and for `criteria` on `choice`/`score`. Any other key passes through
untouched, so a typo such as `"instruction"` reaches the server unnoticed.

Source: https://docs.typesafe.ai/sdk/python/usage#unknown-answer-kinds (Sync tab)

```python
from typesafe_sdk import Noul, TypeSafeClient

result = TypeSafeClient().system_one(
    "I was charged twice.",
    {"billing": Noul(instructions="Is this about billing?")},
)
raw_answers = result.raw_http_response.json()["answers"]
```

Unknown top-level response fields are ignored (`extra="ignore"`). An answer missing `type`, or whose
`type` is not a string, is **not** forward-compatible: it raises `TypeSafeAPIResponseValidationError`
with `field_path == "answers.<name>.type"`. This was read from the source.

---

## Gateways

Any service implementing the TypeSafe OpenAPI spec works through `base_url`. The documented gateways
(https://docs.typesafe.ai/sdk/python/usage#configuring-the-base-url, read 2026-10-03):

| Gateway | `base_url` | `model` | key env var |
|---|---|---|---|
| OpenRouter | `https://openrouter.ai/api` | `~typesafe/jev-latest` | `OPENROUTER_API_KEY` |
| Vercel AI Gateway | `https://ai-gateway.vercel.sh/typesafe` | `typesafe-ai/jev` | `AI_GATEWAY_API_KEY` |
| Pydantic AI Gateway | `https://gateway-us.pydantic.dev/proxy/typesafe` | `jev-latest` | `PYDANTIC_AI_GATEWAY_API_KEY` |

Source: https://docs.typesafe.ai/sdk/python/usage#configuring-the-base-url (OpenRouter, Sync client)

```python
import os

from typesafe_sdk import Noul, TypeSafeClient

with TypeSafeClient(
    api_key=os.environ["OPENROUTER_API_KEY"],
    base_url="https://openrouter.ai/api",
    model="~typesafe/jev-latest",
) as client:
    result = client.system_one(
        "I was charged twice.",
        {"billing": Noul(instructions="Is this about billing?")},
    )
    print(result.nouls["billing"].noul)
```

Gateway caveats, inferred from the source and **not tested against the gateways**:

- The SDK appends `/v1/systemone` to `base_url` and sends `Authorization: Bearer <key>`.
- `request_id` reads `x-typesafe-request-id`. If a gateway does not forward that header,
  `result.request_id` **raises `TypeSafeError`**. Read it defensively, for example
  `result.raw_http_response.headers.get("x-typesafe-request-id")`.
- `Retry-After` handling depends on what the gateway returns.

---

## Serialization and Picklability

Request encoding uses `pydantic_core.to_json` with a fallback that converts any `Mapping` to a `dict`,
any non-string `Sequence` to a `list`, and `str` subclasses to `str`. All rows were executed:

| Input | On the wire |
|---|---|
| `StrEnum` member as `state` or `instructions` | `"calm"`. The 0.7.0 fix: before it, `str` subclasses were serialized as lists of characters |
| `MappingProxyType`, tuples | object, array |
| `datetime.date` | `"2026-01-01"` (ISO) |
| `decimal.Decimal("9.90")` | `"9.90"` (**string**, not a number) |
| `bytes` (valid UTF-8) | a JSON string: `b"hi"` becomes `"hi"` (pydantic-core encodes bytes natively; the fallback is never called) |
| non-UTF-8 `bytes`, arbitrary objects | `TypeSafeError("The request body could not be encoded as JSON")` |

Pickling has worked since 0.6.0. Executed with `pickle.dumps` / `pickle.loads` on 0.7.2:

- `SystemOneResponse` round-trips and compares equal. `request_id` and even `raw_http_response` survive,
  because pydantic pickles `__dict__`.
- Individual answers round-trip.
- `TypeSafeRateLimitError` keeps `retry_after_ms`, `TypeSafeAPIResponseValidationError` keeps
  `field_path`, and `TypeSafeAPITimeoutError` keeps `timeout`.
- API errors raised by a real (mocked) call keep `endpoint`, `headers` and therefore `request_id`:
  `str()` is identical before and after a round trip, including the `(request_id=...)` suffix.
  This was executed for 401, 402, 422, 429 and 500 and for a response-validation error. The
  constructor `args` carry `(status, body, headers, message, endpoint)`, so nothing is lost.
  An error you construct by hand without `endpoint` or a request-id header simply has none to keep.

This makes results safe to return from `multiprocessing` / `concurrent.futures.ProcessPoolExecutor`
workers. A **client** is not picklable; build one per process.

---

## Production Pattern: Testing With a Mock Transport

The SDK's HTTP layer is `httpx2`, a separate package from `httpx`, so the right mock is
**`httpx2.MockTransport`**. Mocks that patch the `httpx` package, such as `respx` or
`httpx.MockTransport`, are not expected to intercept `httpx2`. That was not tested here. Pass the mock as `transport=`;
for the async client, the handler may be sync or `async def`, and both were executed.

Mock checklist:

- Return the documented body `{"model", "answers", "usage"}`.
- Include `x-typesafe-request-id` if the code under test reads `.request_id`. Without it, the property raises `TypeSafeError`.
- `MockTransport` does **not** enforce timeouts. Simulate them by raising `httpx2.ReadTimeout` in the handler.
  The SDK turns that into `TypeSafeAPITimeoutError`.
- Retries are real. A handler that returns 429 or 5xx is called `max_retries + 1` times with real sleeps,
  so in tests send `retry-after-ms: 1` or pass `RetryPolicy(max_retries=0)`.

This file was executed with pytest 9.1.1, and all 3 tests passed in 0.16 s:

```python
"""Offline tests for code that calls TypeSafe, using httpx2.MockTransport."""

import json

import httpx2
import pytest

from typesafe_sdk import (
    Choice,
    Noul,
    RetryPolicy,
    TypeSafeAPIConnectionError,
    TypeSafeClient,
    TypeSafeRateLimitError,
)


def answers_response(answers: dict, *, model: str = "jev-1.13.0") -> httpx2.Response:
    """A 200 response in the documented POST /v1/systemone shape."""
    return httpx2.Response(
        200,
        json={"model": model, "answers": answers, "usage": {"input_tokens": 120, "output_tokens": 12}},
        headers={"x-typesafe-request-id": "req_test"},  # without it, .request_id raises TypeSafeError
    )


@pytest.fixture
def recorded() -> list[httpx2.Request]:
    return []


@pytest.fixture
def client(recorded: list[httpx2.Request]):
    def handler(request: httpx2.Request) -> httpx2.Response:
        recorded.append(request)
        body = json.loads(request.content)
        assert set(body) == {"state", "model", "questions"}
        return answers_response(
            {
                "billing": {"type": "noul", "noul": 0.97},
                "tone": {
                    "type": "choice",
                    "choice": "angry",
                    "confidence": 0.81,
                    "probabilities": {"calm": 0.1, "angry": 0.9},
                },
            }
        )

    with TypeSafeClient(api_key="test-key", model="jev-1.13.0", transport=httpx2.MockTransport(handler)) as c:
        yield c


def classify(client: TypeSafeClient, ticket: str) -> str:
    """The code under test."""
    result = client.system_one(
        ticket,
        {
            "billing": Noul(instructions="Is this about billing?"),
            "tone": Choice(instructions="What is the tone?", criteria={"calm": None, "angry": None}),
        },
    )
    if result.nouls["billing"].noul >= 0.8 and result.choices["tone"].choice == "angry":
        return "billing-escalation"
    return "queue"


def test_routes_angry_billing(client: TypeSafeClient, recorded: list[httpx2.Request]) -> None:
    assert classify(client, "I was charged twice!") == "billing-escalation"
    sent = json.loads(recorded[0].content)
    assert recorded[0].url.path == "/v1/systemone"
    assert sent["model"] == "jev-1.13.0"
    assert sent["questions"]["tone"]["criteria"] == {"calm": None, "angry": None}


def test_rate_limit_surfaces_after_retries() -> None:
    attempts: list[str | None] = []

    def handler(request: httpx2.Request) -> httpx2.Response:
        attempts.append(request.headers.get("x-typesafe-retry-count"))
        return httpx2.Response(429, json={"error": "rate limited"}, headers={"retry-after-ms": "1"})

    with TypeSafeClient(api_key="k", transport=httpx2.MockTransport(handler)) as c:
        with pytest.raises(TypeSafeRateLimitError) as info:
            c.system_one("x", {"billing": Noul()})
    assert attempts == [None, "1", "2"]  # default max_retries=2
    assert info.value.retry_after_ms == 1.0


def test_connection_error_is_not_an_api_error() -> None:
    def handler(request: httpx2.Request) -> httpx2.Response:
        raise httpx2.ConnectError("connection refused")

    with TypeSafeClient(api_key="k", transport=httpx2.MockTransport(handler), retry=RetryPolicy(max_retries=0)) as c:
        with pytest.raises(TypeSafeAPIConnectionError):
            c.system_one("x", {"billing": Noul()})
```

The answer objects are frozen pydantic models, so you can also build fixtures directly without HTTP,
for example `NoulAnswer(noul=0.9)`. To compare Jev with an LLM through the same calling code,
TypeSafe's launch blog post ("Introducing System One Models & Jev",
https://typesafe.ai/blog/introducing-system-one-models-and-jev) links its "System One LLM wrapper", https://github.com/typesafe-ai/system-one-adapter-python. The repository's GitHub
description reads "Drop-in TypeSafeClient replacement backed by LLM APIs". The docs site
(docs.typesafe.ai) does not mention it as of 2026-10-03. That adapter was not evaluated here.

---

## Production Pattern: Bounded Concurrency Under the Rate Limit

Limits for `jev-1.13.0`, from https://docs.typesafe.ai/models (read 2026-10-03):

- **100K tokens per second / 80 requests per second.**
- The docs warn that limits "are adjusting dynamically ... can change without notice".
- A request over either limit gets `429`.

A bare `asyncio.Semaphore` limits **concurrency**, not **rate**. By Little's law, in-flight = rate x latency.
At 80 req/s and 0.4 s latency you need about 32 slots to reach the limit, while at 50 ms latency 32 slots
allow about 640 req/s, eight times over the limit. Use the semaphore to cap memory and connections,
and a rate limiter to respect the request and token budgets.

The token budget can bind before the request budget. Requests may be up to 64k tokens, and two of those
in one second already exceed 100K tokens/s.

The code below was executed. In the offline demo (240 states, 40 ms mock latency, 90% headroom), requests
started per 1 s window were `[72, 72, 72, 24]` with peak in-flight 32 and a total of 3.11 s.

```python
import asyncio
import time
from collections import deque
from collections.abc import Mapping, Sequence

from typesafe_sdk import AsyncTypeSafeClient, Question, RetryPolicy, SystemOneResponse, TypeSafeError

# Published limits for jev-1.13.0 (https://docs.typesafe.ai/models, read 2026-10-03; "adjusting dynamically").
REQUESTS_PER_SECOND = 80
TOKENS_PER_SECOND = 100_000


class SlidingWindowLimiter:
    """Admit a request only if the last `window` seconds stay under both limits (requests and tokens)."""

    def __init__(self, rps: int, tps: int, *, headroom: float = 0.9, window: float = 1.0) -> None:
        self._max_requests = int(rps * headroom)
        self._max_tokens = int(tps * headroom)
        self._window = window
        self._log: deque[tuple[float, int]] = deque()  # (start time, estimated tokens)
        self._lock = asyncio.Lock()  # FIFO: waiters are admitted in arrival order

    async def acquire(self, tokens: int) -> None:
        tokens = min(tokens, self._max_tokens)
        async with self._lock:
            while True:
                now = time.monotonic()
                while self._log and now - self._log[0][0] >= self._window:
                    self._log.popleft()
                used = sum(t for _, t in self._log)
                if len(self._log) < self._max_requests and used + tokens <= self._max_tokens:
                    self._log.append((now, tokens))
                    return
                await asyncio.sleep(self._window - (now - self._log[0][0]))


async def classify_all(
    client: AsyncTypeSafeClient,
    states: Sequence[str],
    questions: Mapping[str, Question],
    *,
    max_in_flight: int = 32,
    estimate_tokens=lambda state: len(state) // 3 + 200,  # heuristic, not the API tokenizer
) -> list[SystemOneResponse | TypeSafeError]:
    limiter = SlidingWindowLimiter(REQUESTS_PER_SECOND, TOKENS_PER_SECOND)
    in_flight = asyncio.Semaphore(max_in_flight)
    retry = RetryPolicy(max_retries=3, timeout=20.0)

    async def one(state: str) -> SystemOneResponse | TypeSafeError:
        async with in_flight:
            await limiter.acquire(estimate_tokens(state))
            try:
                return await client.system_one(state, questions, retry=retry)
            except TypeSafeError as error:  # includes connection and timeout errors
                return error

    return await asyncio.gather(*(one(s) for s in states))


# ---- offline demo: a mock transport with 40 ms latency that records request start times ----
if __name__ == "__main__":
    import httpx2

    from typesafe_sdk import Noul

    starts: list[float] = []
    peak = [0, 0]

    async def handler(request: httpx2.Request) -> httpx2.Response:
        starts.append(time.monotonic())
        peak[0] += 1
        peak[1] = max(peak)
        await asyncio.sleep(0.04)
        peak[0] -= 1
        return httpx2.Response(
            200,
            json={"model": "jev-1.13.0", "answers": {"spam": {"type": "noul", "noul": 0.1}}, "usage": {"input_tokens": 50, "output_tokens": 1}},
            headers={"x-typesafe-request-id": f"r{len(starts)}"},
        )

    async def main() -> None:
        async with AsyncTypeSafeClient(api_key="k", transport=httpx2.MockTransport(handler)) as client:
            t0 = time.monotonic()
            results = await classify_all(client, [f"message {i}" for i in range(240)], {"spam": Noul(instructions="Is this spam?")})
            elapsed = time.monotonic() - t0
        per_second = [sum(1 for s in starts if k <= s - t0 < k + 1) for k in range(int(elapsed) + 1)]
        print(f"{len(results)} results in {elapsed:.2f}s; starts per 1s window {per_second}; peak in flight {peak[1]}")
        assert all(isinstance(r, SystemOneResponse) for r in results)
        assert max(per_second) <= REQUESTS_PER_SECOND

    asyncio.run(main())
```

Design notes:

- **Why a sliding window, not a token bucket.** A first version used a token bucket with burst
  capacity equal to the rate. It started 142 requests in the first second (`[142, 72, 26]`), because
  the full bucket plus one second of refill admitted twice the limit. The API does not document
  whether its window is fixed, sliding or a bucket. A sliding log keeps **every** 1 s window under the
  limit, which is the conservative reading.
- **SDK retries bypass the limiter.** Retries happen inside `system_one`. A burst of 429s therefore
  produces extra requests that the limiter never counted. The SDK honours `Retry-After`, and the
  headroom absorbs some of this. For strict accounting, set `RetryPolicy(max_retries=0)` and requeue
  failed states through the limiter yourself.
- **Errors are returned, not raised.** One bad state should not cancel the batch through `gather`.
  Callers inspect `isinstance(r, TypeSafeError)`.
- **Several processes share one key.** Each process has its own limiter, so divide the limits by the
  number of workers, or move the limiter into a shared store.
- **The token estimate is a heuristic.** No public tokenizer is documented. Calibrate the divisor
  against `result.usage.input_tokens` on your own data.

---

## Production Pattern: Packing Many Questions per State

The docs recommend sending all of a state's questions in one request. "All questions are evaluated in
parallel, so adding more questions usually has little effect on response time"
(https://docs.typesafe.ai/patterns/fan-out). The state is ingested once.

Two budgets bound one request (https://docs.typesafe.ai/models, `jev-1.13.0`, read 2026-10-03):

- **64k tokens** for the state plus all questions combined.
- **32k tokens** for the state plus the single longest question.

When the questions do not fit, split them into several requests over the **same state**. The state is
billed once per request, so minimise the number of chunks. Executed:

```python
import json
from collections.abc import Callable, Mapping

from pydantic_core import to_json

from typesafe_sdk import Answer, JSONContent, Question, TypeSafeClient

# Context limits for jev-1.13.0 (https://docs.typesafe.ai/models, read 2026-10-03):
# 64k tokens for state + all questions; 32k for state + the single longest question.
TOTAL_BUDGET = 64_000
STATE_PLUS_LONGEST_BUDGET = 32_000
SAFETY = 0.8  # the API's tokenizer is not public: leave room for estimation error


def rough_tokens(value: object) -> int:
    """Heuristic: ~3 characters per token of compact JSON. Calibrate against usage.input_tokens."""
    if hasattr(value, "model_dump"):
        value = value.model_dump(exclude_none=True)
    return len(to_json(value)) // 3 + 1


def pack_questions(
    state: JSONContent,
    questions: Mapping[str, Question],
    estimate: Callable[[object], int] = rough_tokens,
) -> list[dict[str, Question]]:
    """Split one state's questions into requests that each fit both context budgets."""
    state_tokens = estimate(state)
    total_cap = int(TOTAL_BUDGET * SAFETY)
    pair_cap = int(STATE_PLUS_LONGEST_BUDGET * SAFETY)
    batches: list[dict[str, Question]] = []
    current: dict[str, Question] = {}
    used = state_tokens
    for name, question in questions.items():
        cost = estimate(question) + estimate(name)
        if state_tokens + cost > pair_cap:
            raise ValueError(f"state + question {name!r} exceeds the 32k budget; shorten the state or the question")
        if current and used + cost > total_cap:
            batches.append(current)
            current, used = {}, state_tokens
        current[name] = question
        used += cost
    if current:
        batches.append(current)
    return batches


def ask_many(client: TypeSafeClient, state: JSONContent, questions: Mapping[str, Question]) -> tuple[dict[str, Answer], int]:
    """Ask every question about one state; returns merged answers and billed input tokens."""
    answers: dict[str, Answer] = {}
    billed = 0
    for batch in pack_questions(state, questions):
        result = client.system_one(state, batch)
        answers.update(result.answers)
        billed += result.usage.input_tokens or 0  # the state is billed once per request
    return answers, billed


if __name__ == "__main__":
    import httpx2

    from typesafe_sdk import Noul

    requests: list[int] = []

    def handler(request: httpx2.Request) -> httpx2.Response:
        body = json.loads(request.content)
        requests.append(len(body["questions"]))
        answers = {name: {"type": "noul", "noul": 0.5} for name in body["questions"]}
        return httpx2.Response(200, json={"model": "jev-1.13.0", "answers": answers, "usage": {"input_tokens": len(request.content) // 3, "output_tokens": 1}})

    state = "contract text " * 3_000  # ~42k chars
    questions = {f"clause_{i}": Noul(instructions=f"Does the contract contain clause {i}? " + "context " * 150) for i in range(120)}
    with TypeSafeClient(api_key="k", transport=httpx2.MockTransport(handler)) as client:
        answers, billed = ask_many(client, state, questions)
    print(f"{len(answers)} answers from {len(requests)} requests, questions per request {requests}, mock-billed {billed}")
    assert set(answers) == set(questions)
    try:
        pack_questions("x" * 100_000, {"q": Noul()})
    except ValueError as error:
        print("oversized:", error)
# -> 120 answers from 2 requests, questions per request [86, 34], mock-billed 79357
# -> oversized: state + question 'q' exceeds the 32k budget; shorten the state or the question
```

Notes:

- The merged answers keep your question names, which must be unique across chunks. In an async
  pipeline, send the chunks concurrently through the limiter above.
- Before splitting, consider trimming the state. Accuracy "falls as the state grows with content
  unrelated to the decision" (https://docs.typesafe.ai/model-jaggedness/jev-1.13), so a smaller, more
  relevant state may beat a large one split into many requests.
- Pricing for `jev-1.13.0` is **$0.042 per million input tokens**; output tokens are free
  (https://docs.typesafe.ai/models, read 2026-10-03). Each extra chunk re-bills the full state, so with
  a 40k-token state each additional chunk costs about $0.0017. That figure is derived from the published price.

---

## Production Pattern: Fallback Wrapper

The handler must cover every failure: HTTP errors, connection failures and timeouts (which are not
`TypeSafeAPIError`), invalid 200 bodies, and client-side errors. It should also distinguish errors that
need a human (401/403) from transient ones. Executed:

```python
import logging
from collections.abc import Callable, Mapping
from dataclasses import dataclass
from typing import Generic, TypeVar

from typesafe_sdk import (
    JSONContent,
    Question,
    RetryPolicy,
    SystemOneResponse,
    TypeSafeAPIConnectionError,
    TypeSafeAPIError,
    TypeSafeAPIResponseValidationError,
    TypeSafeAuthenticationError,
    TypeSafeError,
    TypeSafePermissionDeniedError,
    TypeSafeClient,
)

log = logging.getLogger("app.typesafe")
T = TypeVar("T")

# Latency-critical path: 0.8 s per attempt, at most one retry, ~1.5 s retry budget.
# Wall clock can still reach budget + one attempt timeout (the budget does not cancel an attempt).
FAST = RetryPolicy(max_retries=1, backoff_initial=0.1, backoff_max=0.2, timeout=1.5)


@dataclass(frozen=True)
class Outcome(Generic[T]):
    value: T
    source: str  # "jev" or "fallback"
    reason: str | None = None
    request_id: str | None = None


def decide_or_fallback(
    client: TypeSafeClient,
    state: JSONContent,
    questions: Mapping[str, Question],
    decide: Callable[[SystemOneResponse], T],
    fallback: Callable[[], T],
) -> Outcome[T]:
    try:
        result = client.system_one(state, questions, retry=FAST, timeout=0.8)
    except (TypeSafeAuthenticationError, TypeSafePermissionDeniedError) as error:
        log.error("typesafe credentials rejected: %s", error)  # page someone; every call will fail
        return Outcome(fallback(), "fallback", f"auth:{error.status}", error.request_id)
    except TypeSafeAPIResponseValidationError as error:
        log.error("typesafe response did not match the model at %s", error.field_path)
        return Outcome(fallback(), "fallback", f"invalid-response:{error.field_path}", error.request_id)
    except TypeSafeAPIError as error:  # the API answered with an error status (after retries)
        log.warning("typesafe http %s: %s", error.status, error)
        return Outcome(fallback(), "fallback", f"http:{error.status}", error.request_id)
    except TypeSafeAPIConnectionError as error:  # includes TypeSafeAPITimeoutError; NOT a TypeSafeAPIError
        log.warning("typesafe unreachable: %s", error)
        return Outcome(fallback(), "fallback", type(error).__name__)
    except TypeSafeError as error:  # client-side: empty questions, unencodable state, ...
        log.error("typesafe request rejected before sending: %s", error)
        return Outcome(fallback(), "fallback", "client")
    return Outcome(decide(result), "jev", request_id=result.request_id)


if __name__ == "__main__":
    import httpx2

    from typesafe_sdk import Noul

    logging.basicConfig(format="%(levelname)s %(message)s")
    ok = {"model": "jev-1.13.0", "answers": {"spam": {"type": "noul", "noul": 0.93}}, "usage": {"input_tokens": 9, "output_tokens": 1}}

    def raises(exc: Exception):
        def handler(request: httpx2.Request) -> httpx2.Response:
            raise exc
        return handler

    scenarios = {
        "ok": lambda r: httpx2.Response(200, json=ok, headers={"x-typesafe-request-id": "r1"}),
        "overloaded": lambda r: httpx2.Response(529, json={"error": "overloaded"}),
        "bad key": lambda r: httpx2.Response(401, json={"error": "invalid api key"}),
        "timeout": raises(httpx2.ReadTimeout("slow")),
        "refused": raises(httpx2.ConnectError("refused")),
        "bad body": lambda r: httpx2.Response(200, json={"model": "jev-1.13.0", "answers": {"spam": {"type": "noul"}}, "usage": {}}),
    }
    for name, handler in scenarios.items():
        with TypeSafeClient(api_key="k", transport=httpx2.MockTransport(handler)) as client:
            outcome = decide_or_fallback(
                client, "WIN A FREE CRUISE", {"spam": Noul(instructions="Is this spam?")},
                decide=lambda r: r.nouls["spam"].noul >= 0.9,
                fallback=lambda: False,
            )
        print(f"{name:10} -> {outcome}")
    with TypeSafeClient(api_key="k", transport=httpx2.MockTransport(scenarios["ok"])) as client:
        print("empty     ->", decide_or_fallback(client, "x", {}, decide=lambda r: True, fallback=lambda: False))
```

Executed output (log lines omitted):

```text
ok         -> Outcome(value=True, source='jev', reason=None, request_id='r1')
overloaded -> Outcome(value=False, source='fallback', reason='http:529', request_id=None)
bad key    -> Outcome(value=False, source='fallback', reason='auth:401', request_id=None)
timeout    -> Outcome(value=False, source='fallback', reason='TypeSafeAPITimeoutError', request_id=None)
refused    -> Outcome(value=False, source='fallback', reason='TypeSafeAPIConnectionError', request_id=None)
bad body   -> Outcome(value=False, source='fallback', reason='invalid-response:answers.spam.noul', request_id=None)
empty      -> Outcome(value=False, source='fallback', reason='client', request_id=None)
```

The `except` clauses must stay in this order. `TypeSafeAPIResponseValidationError` and the 401/403
classes are subclasses of `TypeSafeAPIError`, so they must come before it. `TypeSafeError` comes last.

A fallback only handles failures. An answer that is *uncertain* (a `noul` near 0.5, or a low
`confidence`) is a successful call and needs a separate routing decision. See the
confidence-routing pattern at https://docs.typesafe.ai/patterns/confidence-routing.

---

## Changelog and Breaking Changes

From https://docs.typesafe.ai/sdk/python/changelog (read 2026-10-03):

| Version | Date | Breaking | Other |
|---|---|---|---|
| 0.7.2 | 2026-09-26 | none | `http2` extra; HTTP/2 docs |
| 0.7.1 | 2026-09-21 | none | API key validated early (at construction) and excluded from logged exceptions; gateway examples |
| 0.7.0 | 2026-09-18 | **ser/de moved from `msgspec` to `pydantic`** | `response_model=` added; `str` subclasses serialized as strings, no longer as lists of characters |
| 0.6.0 | 2026-09-15 | **`Score.criteria` is an ordered sequence, no longer a dict keyed by int** | `Mapping`/`Sequence` accepted on inputs; richer error messages; `RetryPolicy` validates values; exceptions and responses picklable |
| 0.5.7 | 2026-09-14 | n/a | initial public release |

Migration notes:

- **From < 0.6.0:** rewrite `Score(criteria={0: "low", 1: "high"})` as `Score(criteria=["low", "high"])`.
  Response maps are still keyed by int on the SDK side.
- **From < 0.7.0:** response and answer objects are pydantic models, no longer `msgspec.Struct`s.
  Use `model_dump()` instead of `msgspec.structs.asdict` / `msgspec.json.encode`.
  The objects are frozen and strict. Any code that relied on msgspec-specific behaviour must be reviewed.
  The SDK no longer depends on `msgspec`.
- **From < 0.7.1:** an invalid key now fails when `TypeSafeClient(...)` is called, not on the first
  request. If clients are created at import time, a missing key now breaks import, so build clients lazily.

---

## Unverified / Open Points

- **Live API behaviour.** Nothing here was run against `api.typesafe.ai`, so these are untested:
  real latency, the rate-limit window type, whether 429s carry `retry-after` or `retry-after-ms`,
  HTTP/2 negotiation, and the error-body shapes beyond what the docs and the OpenAPI spec show.
- **Token estimation.** No tokenizer is documented. The `len(json) // 3` heuristic is our own assumption
  and must be calibrated against `usage.input_tokens`.
- **`weight` / `beam_width`.** These appear in official SDK examples, but the docs themselves call them
  illustrative, and they are absent from the OpenAPI spec (version `0.2.0`). Do not send them in production.
- **Gateway behaviour.** Request-id and retry-header forwarding on OpenRouter, Vercel and the Pydantic AI
  Gateway was not checked.
- **Rate limits and prices** are as published on 2026-10-03 and are explicitly subject to change.
  Re-read https://docs.typesafe.ai/models before sizing anything.
- **Thread safety of the sync client** across threads was not tested. It holds no per-call mutable
  state of its own, and `httpx2.Client` thread-safety follows httpx2's guarantees, which were not
  re-verified here.
