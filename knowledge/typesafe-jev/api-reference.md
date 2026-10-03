# TypeSafe Jev - HTTP API Reference

> Official Documentation: https://docs.typesafe.ai/api
> Models, limits and pricing: https://docs.typesafe.ai/models
> Confidence: https://docs.typesafe.ai/confidence
> OpenAPI spec: https://api.typesafe.ai/openapi.json
> Last verified: 2026-10-03
> Verified against: OpenAPI spec `info.version` **0.2.0** (re-fetched 2026-10-03, byte-identical to the earlier copy), model **`jev-1.13.0`**, `typesafe-sdk` **0.7.2** (Python), `@typesafe-ai/sdk` **0.6.0** (JS), `@ai-sdk/typesafe-ai` **3.0.12**, `pydantic-ai-slim[typesafe]` **2.54.0**, `langchain-typesafe` **0.0.1a3**, `genai-prices` **0.1.9**

## Overview

Jev is TypeSafe's "System One" decision model. Its HTTP API has exactly two operations: `POST /v1/systemone`, which evaluates one `state` against a map of typed questions (Noul = yes/no probability, Choice = pick one of N options, Score = ordered rubric) and returns one probability-bearing answer per question, and `GET /v1/models`, which lists model names. There is no streaming, no sampling parameter, no batch endpoint and no other resource.

This document is the wire-level reference: every request and response field, every limit and who enforces it, the error bodies, the headers the SDKs send and read, the confidence formulas with recomputed examples, and a table of every place where the docs, the OpenAPI spec and the SDKs disagree. It goes below the `typesafe-jev` skill, which covers the SDKs and question design.

**Provenance of code blocks.** Every code block is preceded by a line saying what it is:

- *Verbatim* - copied unchanged from the cited official page.
- *Executed* - run offline on 2026-10-03. There was no API key and the real API was never called. HTTP examples ran against `httpx.MockTransport`, a scripted fake `fetch`, or a local Python stand-in server that validates request bodies with the Python SDK's generated wire models (`typesafe_sdk._schemas.models`, which mirror `openapi.json`). Mock responses reuse the response bodies from the docs. **Mock behaviour is not evidence of server behaviour**; where a block shows what the server returns, the source is named.
- *Type-checked* - compiled with `tsc` 7.0.2 (`strict`, `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`) and also executed with Node 22.22.0.

---

## Table of Contents

1. [Endpoints, Base URL and Authentication](#endpoints-base-url-and-authentication)
2. [Headers](#headers)
3. [Request Body: POST /v1/systemone](#request-body-post-v1systemone)
4. [Question Types](#question-types)
5. [Structured State, Instructions and Criteria](#structured-state-instructions-and-criteria)
6. [Validation Rules: Who Enforces What](#validation-rules-who-enforces-what)
7. [Response Body](#response-body)
8. [Answer Types](#answer-types)
9. [GET /v1/models](#get-v1models)
10. [Errors](#errors)
11. [Retries and Rate Limits](#retries-and-rate-limits)
12. [Context Limits](#context-limits)
13. [Models, Aliases and Pinning](#models-aliases-and-pinning)
14. [Pricing](#pricing)
15. [Confidence Formulas](#confidence-formulas)
16. [Example: curl](#example-curl)
17. [Example: Raw Python with httpx](#example-raw-python-with-httpx)
18. [Example: Raw fetch (TypeScript)](#example-raw-fetch-typescript)
19. [Discrepancies Between Docs, OpenAPI and SDKs](#discrepancies-between-docs-openapi-and-sdks)
20. [Not Documented Anywhere](#not-documented-anywhere)
21. [Sources](#sources)

---

## Endpoints, Base URL and Authentication

| Item | Value | Source |
|---|---|---|
| Base URL | `https://api.typesafe.ai` | docs `/api`; `DEFAULT_BASE_URL` in `typesafe_sdk/constants.py` |
| Evaluate | `POST /v1/systemone` | docs `/api`, OpenAPI `operationId: systemone_v1_systemone_post` |
| List models | `GET /v1/models` | docs `/models`, OpenAPI `operationId: models_v1_v1_models_get` |
| Auth scheme | HTTP Bearer: `Authorization: Bearer <API_KEY>` | docs `/api`; OpenAPI `securitySchemes.HTTPBearer` (`type: http`, `scheme: bearer`), applied to both operations |
| Key issuance | https://console.typesafe.ai/keys | docs `/introduction/quickstart` |
| Playground | https://console.typesafe.ai/playground | docs `/introduction/quickstart` |
| Conventional env var | `TYPESAFE_API_KEY` | both official SDKs; **`@ai-sdk/typesafe-ai` reads `TYPESAFE_AI_API_KEY` instead** (see [Discrepancies](#discrepancies-between-docs-openapi-and-sdks)) |
| Base-URL override env var | `TYPESAFE_BASE_URL` | Python SDK `BASE_URL_ENV`; JS SDK `ENV.baseURL` |
| Default-model env var (SDKs only) | `TYPESAFE_DEFAULT_MODEL` | Python SDK `DEFAULT_MODEL_ENV`; JS SDK `ENV.defaultModel`. Used when a call omits `model`; falls back to `jev-latest` |

There is no API version header and no query parameter on either operation. The version lives in the path (`/v1`).

The API key is a single opaque string. The Python SDK strips surrounding whitespace from the key, then rejects it *before* sending if it is empty, contains a space, or is not printable ASCII (`resolve_and_validate_api_key` in `typesafe_sdk/_core/config.py`). That is a client-side check, not a documented key format.

*Verbatim* - https://docs.typesafe.ai/introduction/quickstart (the canonical minimal call):

```bash
curl -X POST https://api.typesafe.ai/v1/systemone \
  -H "Authorization: Bearer $TYPESAFE_API_KEY" \
  -H "Content-Type: application/json" \
  -d @- <<'EOF'
  {
    "state": "Hi, I've been trying to connect my Stripe account for 3 days and the integration keeps failing. I'm losing sales. Please help ASAP.",
    "model": "jev-latest",
    "questions": {
      "urgency": {
        "type": "noul",
        "instructions": "Does this message express urgency?"
      }
    }
  }
EOF
```

---

## Headers

### Request headers

| Header | Required | Notes | Source |
|---|---|---|---|
| `Authorization: Bearer <key>` | yes | Missing or invalid gives `401` | docs `/api` |
| `Content-Type: application/json` | yes on POST | Body is JSON | docs `/api` |
| `Accept: application/json` | no | Both SDKs send it | SDK source |
| `User-Agent: typesafe-sdk/<version>` | no | Sent by both official SDKs | Python `transport.prepare`, JS `fetchWithRetries` |
| `X-TypeSafe-SDK: typesafe-sdk/<version>` | no | SDK telemetry header | same |
| `X-TypeSafe-Runtime` | no | Python: `python/<ver> (<platform>; <machine>)`; JS: runtime/platform string | same |
| `X-TypeSafe-Retry-Count: <n>` | no | Added by both SDKs on retry attempts only (1, 2, ...); stripped from user-supplied headers on the first attempt | same |

The `X-TypeSafe-*` request headers are SDK conventions. The API docs do not document them, and nothing says the server requires them. A raw client can omit them. What the server does with them is not documented.

### Response headers

| Header | Meaning | Source |
|---|---|---|
| `x-typesafe-request-id` | Per-request identifier. Log it and quote it in support requests. Exposed as `SystemOneResponse.request_id` / `TypeSafeAPIError.request_id` (Python) and `requestId` (JS) | Python SDK docs `/sdk/python/api/types/responses`, `/sdk/python/api/exceptions`; `REQUEST_ID_HEADER` constant |
| `retry-after` | Seconds (or an HTTP date) to wait. The docs say the SDKs "honor the `retry-after` header when the response carries one" | docs `/models` |
| `retry-after-ms` | Milliseconds to wait. **The API docs do not mention it**, but both SDKs read it *before* `retry-after` | Python `parse_retry_after`, JS `parseRetryAfter` |
| `content-type` | `application/json` on success and on 422 | OpenAPI response `content` |

The Python SDK raises `TypeSafeError("The response did not include a request ID.")` if you read `.request_id` on a response that lacked the header, so the header is expected but not guaranteed by any contract.

---

## Request Body: POST /v1/systemone

OpenAPI schema `SystemOneRequest`. Required: `model`, `questions`, `state`.

| Field | Type | Required | Constraints | Notes |
|---|---|---|---|---|
| `state` | `string \| object \| array` | yes | OpenAPI: `anyOf [string, object(additionalProperties: true), array(items: {})]`. **`null` is not allowed** | The content every question refers to. Text only: "No image, audio, or video input" (docs `/models`) |
| `model` | `string` | yes (HTTP) | none in spec | Model id or alias. SDKs fill it with the client's default model when omitted (`TYPESAFE_DEFAULT_MODEL`, else `jev-latest`); the raw API requires it |
| `questions` | `map<string, Question>` | yes | OpenAPI `minProperties: 1` | Keys are your ids. Answers come back under the same keys. **Keys are not sent to the model** (docs `/api`, `/primitives`) |

No other top-level field is documented. The SDKs can forward extra fields (`extra_body` in Python; the JS SDK spreads the whole request object into the body), but the server's treatment of unknown fields is not documented. Do not rely on it.

`Question` is a discriminated union on `type` (OpenAPI `discriminator.propertyName: type`, mapping `noul`/`choice`/`score`). This matters for error locations: a validation error inside a question carries the tag in its `loc` path (`["body","questions","urgency","score","criteria"]`, the OpenAPI example). See [Errors](#errors).

*Verbatim* - https://docs.typesafe.ai/api, "Example request":

```json
{
  "state": "Help! My payouts have been failing for 3 days.",
  "model": "jev-latest",
  "questions": {
    "is_urgent": {
      "type": "noul",
      "instructions": "Does this convey urgency?"
    }
  }
}
```

Questions of different types can be mixed in one request. All questions see the same state and are evaluated independently and in parallel (docs `/concepts/state`, `/models`). No maximum number of questions per request is documented. The practical bound is the 64k-token request budget (see [Context Limits](#context-limits)).

---

## Question Types

All three share `type` and `instructions`. Each type defines its own `criteria`.

### Noul (`type: "noul"`)

OpenAPI `NoulQuestion`, `required: ["type"]`.

| Field | Type | Required (OpenAPI) | Required (docs) | Notes |
|---|---|---|---|---|
| `type` | `"noul"` | yes | yes | |
| `instructions` | `string \| object \| array \| null` | **no** | **yes** | "The yes/no question or statement to evaluate." The OpenAPI examples include a statement ("This message contains unsolicited advertising.") and an object (`{"task": "Identify unsolicited advertising."}`) |
| `criteria` | `NoulCriteria \| null` | no | no | Object with optional `true` and `false`, each `string \| object \| array \| null` |
| `criteria.true` | `string \| object \| array \| null` | no | no | "What a yes (value near 1) means." |
| `criteria.false` | `string \| object \| array \| null` | no | no | "What a no (value near 0) means." |

Keep the polarity of `criteria` aligned with `instructions`. The jaggedness page says a Noul "where `true` maps to no and `false` maps to yes will perform worse" (`/model-jaggedness/jev-1.13`).

### Choice (`type: "choice"`)

OpenAPI `ChoiceQuestion`, `required: ["criteria", "type"]`.

| Field | Type | Required | Constraints | Notes |
|---|---|---|---|---|
| `type` | `"choice"` | yes | | |
| `instructions` | `string \| object \| array \| null` | OpenAPI no / docs yes | | "What the model should decide." |
| `criteria` | `map<string, string \| object \| array \| null>` | yes | **max 255 options** (docs `/api`, `/primitives/choice`); not in the OpenAPI spec | Option name to description. `null` means "interpreted by its name alone" (OpenAPI description) |

- The option **keys** are returned verbatim as `choice` and as the keys of `probabilities`. Unlike question ids, option keys *are* part of what the model sees. The OpenAPI text says "A choice without a description is interpreted by its name alone".
- Option order matters. The jaggedness page reports that `jev-1.13` "leans toward the option that comes first". JSON objects are ordered on the wire in practice, but the order is whatever your serializer emits.
- No minimum option count is documented anywhere. The server's behaviour with 0 or 1 option is unknown.
- "adding options costs a few tokens each" (docs `/primitives/choice`).

### Score (`type: "score"`)

OpenAPI `ScoreQuestion`, `required: ["criteria", "type"]`.

| Field | Type | Required | Constraints | Notes |
|---|---|---|---|---|
| `type` | `"score"` | yes | | |
| `instructions` | `string \| object \| array \| null` | OpenAPI no / docs yes | | "What the model should rate." |
| `criteria` | `array<string \| object \| array>` | yes | docs: **at least 2, at most 10** levels; OpenAPI: `minItems: 1`, no max | Ordered low to high. "Each description's position determines its score, starting at zero." |

Level *i* is the array index. The answer's `legend` and `probabilities` are keyed by `"0"`, `"1"`, ... as strings. The docs' advanced page lists `null` as an accepted level description, but the OpenAPI item schema does not include `null` (see [Discrepancies](#discrepancies-between-docs-openapi-and-sdks)). Avoid `null` levels.

---

## Structured State, Instructions and Criteria

Every free-text slot accepts JSON structure. The JS SDK calls this type `EntryType`.

*Verbatim* - table from https://docs.typesafe.ai/primitives/advanced:

| Field | Applies to | Accepted shape |
| - | - | - |
| `instructions` | Choice, Score, Noul | `string`, `object`, `array`, or `null` |
| `criteria` values (option descriptions) | Choice | `string`, `object`, `array`, or `null` |
| `criteria` entries (level descriptions) | Score | `string`, `object`, `array`, or `null` |
| `criteria.true` and `criteria.false` | Noul | `string`, `object`, `array`, or `null` |

`state` accepts `string`, `object` or `array`. It is a required field, and `null` is not allowed in OpenAPI.

### Structured instructions

Put the question in one field and the data it needs in sibling fields, then refer to those fields by name in backticks.

*Verbatim* - https://docs.typesafe.ai/api:

```json
"instructions": {
  "potential_duplicate": {
    "name": "John Smith",
    "location": "Oakland, California",
    "last_employer": "Google"
  },
  "question": "Is the resume for the same person as `potential_duplicate`?"
}
```

Field names inside a structured `instructions` object are free-form. The docs' examples use `question`, `focus`, `note`, `inspect`, `compare`, `field`, `extracted_value` and `task`, and none of them is reserved. An array is also accepted.

*Verbatim* - https://docs.typesafe.ai/primitives/advanced:

```json
"instructions": {
  "question": "Does the claimed sender identity conflict with the sending domain?",
  "compare": ["ticket.sender.display_name", "ticket.sender.email"],
  "focus": "Compare the named organization with the email domain."
}
```

### Backtick paths into `state`

The how-to-build page (https://docs.typesafe.ai/concepts/how-to-build-with-system-one) says to "Use a backticked dot-and-index path to point a question at a specific nested value, such as `support.tickets[0].message`" and to "include the backtick characters around each path inside the question". Documented forms:

| Form | Example from the docs |
|---|---|
| dotted | `` `ticket.message` ``, `` `request.date` `` |
| indexed | `` `support.tickets[0].message` ``, `` `items[{i}]` `` (built in an f-string) |
| mixed | `` `trace.tool_calls[1].arguments.unit` `` |
| sibling field of a structured instruction | `` `potential_duplicate` ``, `` `field` `` |

**Inference, not documented:** nothing in the docs or the OpenAPI spec describes server-side resolution or validation of these paths. They read as prose that the model interprets. A misspelled path therefore produces no error, only a worse answer. Check paths in your own code before sending.

### Structured criteria

A Choice option description, a Score level or a Noul `true`/`false` can each be an object or an array. The docs' contrastive pattern uses the same keys across options (`what`, `not_for`, `examples`). A Choice option value can even be a subtree (`/primitives/advanced`, "Walking a taxonomy"). For a Score with structured levels, the answer's `legend` echoes the object or array back (OpenAPI `ScoreAnswer.legend` values are `string | object | array`).

---

## Validation Rules: Who Enforces What

What each layer checks *before* the request reaches the model. "-" means no check. Python and JS columns come from reading the SDK source; PAI and AI SDK come from reading their source.

| Rule | Docs | OpenAPI 0.2.0 | Python `typesafe-sdk` 0.7.2 | JS `@typesafe-ai/sdk` 0.6.0 | `@ai-sdk/typesafe-ai` 3.0.12 | Pydantic AI 2.54.0 |
|---|---|---|---|---|---|---|
| `questions` non-empty | implied | `minProperties: 1` | `TypeSafeError` | `TypeSafeError` | - | n/a |
| `instructions` present | "required" | optional, nullable | optional (omitted from wire when `None`) | optional | - | n/a |
| Score ≥ 2 levels | "should have at least two" | `minItems: 1` | **≥ 1** only | **≥ 2** (type: tuple `[E, E, ...E[]]`; runtime check) | - | n/a |
| Score ≤ 10 levels | "the API accepts up to 10" | - | - | - | `InvalidArgumentError` | `_JEV_MAX_SCORE_LEVELS = 10` |
| Choice ≤ 255 options | "maximum of 255" | - | - | - | `InvalidArgumentError` | `_JEV_MAX_CHOICE_OPTIONS = 255` |
| Choice criteria is a map | yes | `type: object` | Pydantic | `choice()` helper throws on an array | - | n/a |
| Unknown question field | - | not forbidden (`additionalProperties` unset) | `extra="forbid"` on `Noul`/`Choice`/`Score` objects | not checked | - | n/a |

Consequences:

- A 1-level Score passes the OpenAPI schema and the Python SDK. The docs say a Score "should" have two levels, which reads as advice rather than a hard limit. Whether the server rejects 1 level is not documented.
- An 11-level Score or a 256-option Choice passes OpenAPI-level validation and both official SDKs, so it is rejected (if at all) only by the server. The Pydantic AI source and docs say "an 11th is a 400 from the API" and "a 256th option is a 400 from the API". That is a third-party statement; the TypeSafe docs only list 422 for validation failures. **Validate these limits in your own code** (the raw clients below do).

---

## Response Body

OpenAPI `SystemOneResponse`. Required: `model`, `answers`, `usage`.

| Field | Type | Notes |
|---|---|---|
| `model` | `string` | "Name of the model that answered the questions. May differ from the alias supplied in the request." The docs say it reports the **versioned id** (`jev-1.13.0`) when you sent an alias. Log it on every call |
| `answers` | `map<string, Answer>` | One per question, under the same key. OpenAPI `minProperties: 1`. Each answer's `type` matches its question's |
| `usage.input_tokens` | `integer` | "Number of billable input tokens used to evaluate the request." |
| `usage.output_tokens` | `integer` | "Output tokens are currently free of charge." |

*Verbatim* - https://docs.typesafe.ai/api, "Example response":

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "is_urgent": {
      "type": "noul",
      "noul": 0.95
    }
  },
  "usage": { "input_tokens": 296, "output_tokens": 20 }
}
```

Forward compatibility: the Python SDK drops (with a warning) any answer whose `type` it does not recognise rather than failing the whole response. Raw clients should do the same: switch on `type` and ignore unknown values.

---

## Answer Types

`Answer` is a discriminated union on `type` (`noul` / `choice` / `score`).

### Noul answer

| Field | Type | Notes |
|---|---|---|
| `type` | `"noul"` | |
| `noul` | `number` in [0, 1] | "Probability of a yes answer or a true statement... values near 0.5 indicate uncertainty." |

There is **no `confidence` field** on a Noul answer. The probability itself carries the uncertainty (docs `/confidence`, "Noul").

### Choice answer

| Field | Type | Notes |
|---|---|---|
| `type` | `"choice"` | |
| `choice` | `string` | The option key with the highest probability |
| `probabilities` | `map<string, number>` | **Every** option, including zero-probability ones (docs examples show `"sales": 0.0`). "values sum to approximately 1" |
| `confidence` | `number` in [0, 1] | Derived from `probabilities`; formula below |

*Verbatim* - https://docs.typesafe.ai/api, "Choice answer":

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "department": {
      "type": "choice",
      "choice": "billing",
      "probabilities": { "billing": 0.88, "technical": 0.12, "sales": 0.0 },
      "confidence": 0.81
    }
  },
  "usage": { "input_tokens": 318, "output_tokens": 34 }
}
```

### Score answer

| Field | Type | Notes |
|---|---|---|
| `type` | `"score"` | |
| `score` | `number` | Expected level: Σ i·pᵢ. "May fall between integer levels." Every docs example satisfies `score == Σ i·pᵢ` exactly (checked below) |
| `legend` | `map<string, string \| object \| array>` | `"0"`.. `"n-1"` mapped to your level descriptions. The API page types it as `map<string, string>`; OpenAPI allows object/array |
| `probabilities` | `map<string, number>` | Same keys as `legend`; sums to approximately 1 |
| `confidence` | `number` in [0, 1] | Ordinal-aware formula below |

*Verbatim* - https://docs.typesafe.ai/api, "Score answer":

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "frustration": {
      "type": "score",
      "score": 1.05,
      "legend": { "0": "Calm", "1": "Frustrated", "2": "Very angry" },
      "probabilities": { "0": 0.0, "1": 0.95, "2": 0.05 },
      "confidence": 0.92
    }
  },
  "usage": { "input_tokens": 304, "output_tokens": 18 }
}
```

Level keys are **strings** on the wire. The Python SDK coerces them to `int` (`ScoreAnswer.legend: dict[int, ...]`, `probabilities: dict[int, float]`). The JS SDK types them by tuple index. With raw JSON, sort with `key=int`. Lexical sorting happens to give the same order for 1 to 10 levels (keys `"0"`..`"9"`, verified below), but it breaks the moment anyone allows an 11th level.

### Numeric properties of answers

- **Rounding.** Every docs example shows probabilities, `score` and `confidence` to at most 2 decimals. `@ai-sdk/typesafe-ai` 3.0.12 hard-codes `rounding: { probabilityDecimals: 2, scoreDecimals: 2 }` in its result. The API docs do not state a rounding rule.
- **Confidence is computed before rounding.** Recomputing `confidence` from the rounded probabilities matches the returned value within ±0.01 for all 16 Choice/Score answers in the docs' JSON examples (script below). The `api.md` Choice example gives 0.82 recomputed against 0.81 returned, which is consistent with an unrounded p_max in [0.875, 0.8767) (p_max rounding half-up to 0.88 needs p_max ≥ 0.875; confidence (3p_max - 1)/2 rounding to 0.81 needs p_max < 0.8767). Do not assert exact equality between your recomputation and `confidence`.
- **Sums.** "values sum to approximately 1" (OpenAPI). After 2-decimal rounding, the sum can differ from 1 by a few hundredths. Renormalise before using the probabilities as features.
- **Ties.** How the server picks `choice`, or the Score peak level, when two probabilities are equal is not documented. With rounded values, an apparent tie need not be a real one.

---

## GET /v1/models

Returns the names your account can send in `model`.

| Field | Type | Notes |
|---|---|---|
| `models` | `array<ModelMetadata>` | "It currently lists the aliases" (docs). OpenAPI: "Available models and aliases" |
| `models[].name` | `string` | Accepted by `model` |
| `models[].description` | `string` | |
| `models[].release_date` | `string` | `YYYY-MM-DD` |

"Versioned IDs such as `jev-1.13.0` are accepted by the `model` field whether or not they appear in the list" (docs `/models`). So do not validate `model` against this list.

*Verbatim* - https://docs.typesafe.ai/models:

```bash
curl https://api.typesafe.ai/v1/models \
  -H "Authorization: Bearer $TYPESAFE_API_KEY"
```

OpenAPI example payload: `{"models": [{"description": "General-purpose system one model.", "name": "jev-latest", "release_date": "2026-09-15"}]}`. The OpenAPI spec also lists a 422 response for this GET. It takes no parameters, so that response is most likely FastAPI's auto-generated default and practically unreachable (inference, not documented).

---

## Errors

### Status codes

| Status | Documented by | Meaning | Retry? |
|---|---|---|---|
| `401 Unauthorized` | docs `/api` | Missing or invalid API key | No. Fix the key |
| `422 Unprocessable Entity` | docs `/api` **and** OpenAPI (the only error in the spec) | Body failed validation; FastAPI-style `detail[]` | No. Fix the body |
| `429 Too Many Requests` | docs `/api`, `/models` | Over the tokens/s **or** requests/s limit | Yes, with backoff; honour `retry-after` |
| `529 Overloaded` | docs `/api` | TypeSafe temporarily overloaded | Yes, with backoff |
| `400 Bad Request` | **Pydantic AI docs only** | Said to be returned for a 256th Choice option and an 11th Score level. (For a context-limit overflow the Pydantic AI docs name only a `ModelHTTPError` with `max_tokens_exceeded`, without a status code) | No |
| `403`, `404`, `408`, other `5xx` | not documented by TypeSafe | The SDKs map them anyway (below) | `408` and `5xx`: SDKs retry |

The TypeSafe docs do not document the body shape of 401, 429 and 529 responses. The SDKs' message extractors accept `{"error": "..."}`, `{"error": {"message": "..."}}`, `{"message": "..."}`, `{"detail": "..."}`, `{"detail": {"message": "..."}}` and `{"detail": [...]}`, and fall back to raw text. That is a list of what the clients tolerate, not a server contract.

### 422 body (FastAPI / Pydantic v2 shape)

OpenAPI `HTTPValidationError`: `{"detail": ValidationError[]}`. Each `ValidationError` has:

| Field | Type | Required | Meaning (OpenAPI description) |
|---|---|---|---|
| `loc` | `array<string \| integer>` | yes | "Path to the invalid value: the request location followed by field names and array indices." Starts with `"body"` |
| `msg` | `string` | yes | "Human-readable explanation of the validation failure." |
| `type` | `string` | yes | "Machine-readable validation error code." (e.g. `missing`) |
| `input` | any | no | "The input value that failed validation." |
| `ctx` | `object` | no | "Additional context used to explain the validation failure." (e.g. `{"min_length": 1}`) |

The OpenAPI examples:

- `[{"loc": ["body","state"], "msg": "Field required", "type": "missing"}]`
- `loc: ["body","questions","urgency","score","criteria"]` (note the discriminator tag `score` between the question id and the field), `input: {"type": "score"}`, `ctx: {"min_length": 1}`

*Executed* - `sim422.py`. Validates bad bodies against the Python SDK's generated wire model `SystemOneRequest` (pydantic 2.13.5) and prints the errors with FastAPI's `"body"` prefix. **This reproduces the shape of the OpenAPI examples. It is not output from the live server**, whose `msg` wording may differ. The `null_state` line is abridged.

```text
missing_state -> [{"loc": ["body", "state"], "msg": "Field required", "type": "missing"}]
empty_score -> [{"loc": ["body", "questions", "urgency", "score", "criteria"], "msg": "List should have at least 1 item after validation, not 0", "type": "too_short"}]
unknown_type -> [{"loc": ["body", "questions", "q"], "msg": "Input tag 'bool' found using 'type' does not match any of the expected tags: 'noul', 'choice', 'score'", "type": "union_tag_invalid"}]
no_questions -> [{"loc": ["body", "questions"], "msg": "Dictionary should have at least 1 item after validation, not 0", "type": "too_short"}]
null_state -> [{"loc": ["body", "state", "str"], ...}, {"loc": ["body", "state", "dict[str,any]"], ...}, {"loc": ["body", "state", "list[any]"], ...}]
```

The `empty_score` location matches the OpenAPI example exactly, which strongly suggests (but does not prove) that the server is FastAPI on Pydantic v2 with a tagged union. Practical consequences:

- When you map `loc` back to your request, drop the leading `"body"` and the tag segment that follows a question id (`noul`/`choice`/`score`). The Python SDK does exactly this for *response* validation paths (`SystemOneResponse._decode` deletes `location[2]`).
- In the simulation, a `null` or wrong-typed value under an `anyOf` produces **one error per alternative** (`str`, `dict[str,any]`, `list[any]`), so a single mistake yields three `detail` entries. If the server is Pydantic v2 (likely, not confirmed), expect the same; do not assume `detail` has one entry per mistake.

### SDK exception mapping (for reference when reading SDK logs)

| Status | Python 0.7.2 | JS 0.6.0 |
|---|---|---|
| 400 | `TypeSafeBadRequestError` | `BadRequestError` |
| 401 | `TypeSafeAuthenticationError` | `AuthenticationError` |
| 403 | `TypeSafePermissionDeniedError` | `PermissionDeniedError` |
| 404 | `TypeSafeNotFoundError` | `NotFoundError` |
| 422 | `TypeSafeUnprocessableEntityError` | `UnprocessableEntityError` |
| 429 | `TypeSafeRateLimitError` (`.retry_after_ms`) | `RateLimitError` (`retryAfterMs`) |
| ≥ 500 (incl. 529) | `TypeSafeInternalServerError` | `InternalServerError` |
| other non-2xx | `TypeSafeAPIError` | `APIError` |
| network / timeout | `TypeSafeAPIConnectionError` / `TypeSafeAPITimeoutError` | `APIConnectionError` / `APITimeoutError` (+ `APIUserAbortError`) |
| 2xx with a malformed body | `TypeSafeAPIResponseValidationError(.field_path)` | - |

---

## Retries and Rate Limits

### Limits (docs `/models`, read 2026-10-03)

| Limit | Value | Notes |
|---|---|---|
| Tokens per second | **100K** | Over it: `429` |
| Requests per second | **80** | Over it: `429`. The `llms-full.txt` dump fetched during this research still said **40**, so the live page had already moved |
| Plan | "Higher limits are available on custom and enterprise plans" (sales@typesafe.ai) | |

The docs warn that "Rate limits are adjusting dynamically... the limits above can change without notice". Treat both numbers as the values on the date read, not as constants.

Which limit binds depends on the average request size. *Executed* - `cost.py` (same constants as above):

```text
tokens/request where the two limits cross: 1250.0
avg 318 tokens -> max 80.0 req/s
avg 1250 tokens -> max 80.0 req/s
avg 4000 tokens -> max 25.0 req/s
avg 32000 tokens -> max 3.1 req/s
```

Below ~1,250 input tokens per request, the 80 req/s limit binds. Above it, the 100K tokens/s limit binds. Packing many questions into one request (fan-out) uses a request-limit slot only once.

### Retry behaviour you should match in a raw client

"When you receive a `429 Too Many Requests` or `529 Overloaded` response, retry the request with exponential backoff instead of retrying immediately" (docs `/api`). The SDK defaults, read from source:

| Setting | Python 0.7.2 `RetryPolicy` | JS 0.6.0 `retry` |
|---|---|---|
| Retries after the first attempt | `max_retries=2` | `maxRetries: 2` |
| Retried statuses | `{408, 429, *range(500, 600)}` | 408, 429, ≥ 500 |
| Connection errors / timeouts | retried | retried (not caller aborts) |
| Backoff | 0.5 s doubling, cap 5 s, minus up to 25 % jitter | 500 ms doubling, cap 5000 ms, jitter 0.25 |
| `retry-after-ms` / `retry-after` | honoured, `-ms` first; `retry-after` may be an HTTP date | same order; **ignored if > `maxRetryAfterMs` (60 000)**, falling back to backoff |
| Per-attempt timeout | 10 s (`DEFAULT_TIMEOUT`) | 10 000 ms |
| Total budget | `timeout=30.0` s: stops before a retry whose delay would cross it | none, so pass an `AbortSignal` |

Billing on retries is not documented. A request that timed out on the client may still have been processed and billed, so budget for that.

---

## Context Limits

Docs `/models`, 2026-10-03, for `jev-1.13`:

| Budget | Covers |
|---|---|
| **64k tokens per request** | `state` + all questions combined |
| **32k tokens** | `state` + the single **longest** question |

"Jev ingests the `state` once and evaluates every question against it in parallel." The state is counted once per request, so the 32k budget is the one that a growing `state` hits first. The 64k budget only binds when the questions themselves are large (many big structured questions). `genai-prices` 0.1.9 records Jev's `context_window` as **32000**. Pydantic AI 2.54.0 sets no `context_window` of its own and takes it from genai-prices (comment in `pydantic_ai/profiles/typesafe.py`).

Not documented:

- **Whether "64k"/"32k" means 65,536/32,768 or 64,000/32,000.** `genai-prices` uses 32,000. Stay below 32,000 to be safe.
- **The tokenizer.** There is no token-counting endpoint, so you cannot pre-count exactly. `usage.input_tokens` after the fact is the only measurement.
- **The error returned when you exceed a budget.** Pydantic AI's docs say the request "fails with a `ModelHTTPError` (`max_tokens_exceeded`)". `ModelHTTPError` wraps any non-2xx status, and neither those docs nor the installed `pydantic_ai/models/typesafe.py` name the status code. The TypeSafe docs do not say.

Accuracy also degrades as the state grows with irrelevant content ("Jev suffers from context rot", `/model-jaggedness/jev-1.13`). Filter the state before you approach the limit.

---

## Models, Aliases and Pinning

Docs `/models`, 2026-10-03:

| Name | Resolves to | Meaning |
|---|---|---|
| `jev-1.13.0` | itself | Current versioned model ("Jev 1.13") |
| `jev-latest` | `jev-1.13.0` | "The most recent stable, official release." Default in both SDKs (Python `DEFAULT_MODEL` constant; a hard-coded fallback in the JS client constructor) |
| `jev-preview` | `jev-1.13.0` | "The most recent release, whether or not it is an official one." "There is no preview build available right now." |

- Aliases move when a release ships, and "the answers behind it can change without a change on your side". The vendor's guidance: "If you have tuned confidence thresholds against a specific version, pin that version's ID instead of the alias."
- `response.model` reports the versioned id that answered. Store it next to every decision so you can see an alias move in your data.
- `GET /v1/models` "currently lists the aliases". Versioned ids are accepted even when unlisted.
- History visible in the docs: the cookbooks quote `jev-1.12` with the same $0.042/Mtok price "as of 2026-09". Older versioned ids may exist; whether they are still served is **not documented**.
- The jaggedness page's own example constructs `TypeSafeClient(model="jev-1.13")`, a two-part id that the models page does not list. Whether the API accepts `jev-1.13` (a minor-version alias) is **unverified**. Use `jev-1.13.0`.
- One set of weights serves every account. There is no fine-tuning, LoRA or per-account model (docs `/models`, "Customizing Jev").

---

## Pricing

Docs `/models`, read 2026-10-03:

| Item | Price |
|---|---|
| Input tokens | **$0.042 per million** ($42 per billion, "Btok") |
| Output tokens | **free** ("Charged per input token. Output tokens are free.") |

`genai-prices` 0.1.9 carries the same figure (`input_mtok=Decimal('0.042')`) for `jev-1.13.0`, `jev-latest` and `jev-preview`, and notes "a versioned id is billed the same". The docs' cookbooks quote the same $0.042 rate "as of 2026-08" and "as of 2026-09", so it has not changed in that window.

*Executed* - `cost.py`:

```python
from decimal import Decimal

PRICE_PER_MTOK_INPUT = Decimal("0.042")  # docs.typesafe.ai/models, 2026-10-03; output tokens free
TOKENS_PER_SECOND, REQUESTS_PER_SECOND = 100_000, 80  # same page, same date; "adjusting dynamically"


def cost_usd(input_tokens: int) -> Decimal:
    return input_tokens * PRICE_PER_MTOK_INPUT / 1_000_000


print("one 318-token request:", cost_usd(318))
print("1M such requests:", cost_usd(318 * 1_000_000))
print("one request at the 64k ceiling (65,536 tokens):", cost_usd(65_536))
print("tokens/request where the two limits cross:", TOKENS_PER_SECOND / REQUESTS_PER_SECOND)
for avg in (318, 1_250, 4_000, 32_000):
    print(f"avg {avg} tokens -> max {min(REQUESTS_PER_SECOND, TOKENS_PER_SECOND / avg):.1f} req/s")
```

```text
one 318-token request: 0.000013356
1M such requests: 13.356
one request at the 64k ceiling (65,536 tokens): 0.002752512
```

(318 input tokens is the `usage.input_tokens` of the docs' Choice example.) The price is per input token, and the state is counted once, so adding a question costs only that question's tokens. That is why the docs recommend asking many questions per request.

---

## Confidence Formulas

Source: https://docs.typesafe.ai/confidence. `confidence` is returned on **Choice and Score** answers only. It is a deterministic statistic of the answer's own `probabilities`. It is not a second model output, and it is not calibrated separately.

### Noul (not returned; the docs' suggested equivalent)

$$\text{confidence} = |2p - 1|$$

This is 0 at p = 0.5 and 1 at p = 0 or 1. It equals the Choice formula applied to a two-option Choice.

### Choice

For *n* options with top probability p_max:

$$\text{confidence} = \frac{p_{\max} - \frac{1}{n}}{1 - \frac{1}{n}} = \frac{n\,p_{\max} - 1}{n - 1}$$

Only the top probability counts: (0.6, 0.3, 0.1) and (0.6, 0.2, 0.2) both give 0.4. The same p_max means more as *n* grows. With 255 options, p_max = 0.05 already gives confidence ≈ 0.046 > 0. The docs also suggest two alternatives computed from `probabilities`: **top probability** p_max (set its threshold per question, because its meaning depends on *n*) and the **top-to-second ratio** p_max / p_second.

### Score

For *n* levels 0..n-1, peak level *m* (the most likely level):

$$\text{confidence} = \max\left(0,\ 1 - \frac{\sum_i p_i\,|i - m|}{\text{MAD}_{\text{unif}}}\right),\qquad \text{MAD}_{\text{unif}} = \frac{1}{n}\sum_i \left|i - \frac{n-1}{2}\right|$$

Probability mass next to the peak costs less than mass far from it. That makes Score confidence ordinal-aware where Choice confidence is not.

**Derived property (computed, not stated in the docs):** for **3 levels with the peak in the middle**, Score confidence equals the Choice formula exactly. With m = 1 the spread is 1 - p₁, and 1 - (1-p₁)/(2/3) = (3p₁-1)/2. The two formulas diverge only when the peak is at an end (distance 2 counts double) or when there are more levels.

### Reference implementation and worked examples

*Executed* - `confidence.py`. These functions are equivalent to the docs' `choice_confidence` / `score_confidence` snippets, plus a numeric sort for wire keys:

```python
from collections.abc import Mapping, Sequence


def noul_confidence(p: float) -> float:
    """Distance from 0.5 on a 0..1 scale. The API returns no confidence for Noul."""
    return abs(2 * p - 1)


def choice_confidence(probabilities: Sequence[float]) -> float:
    n = len(probabilities)
    return (max(probabilities) - 1 / n) / (1 - 1 / n)


def mad_uniform(n: int) -> float:
    return sum(abs(i - (n - 1) / 2) for i in range(n)) / n


def score_confidence(probabilities: Sequence[float]) -> float:
    """probabilities[i] is the probability of level i; the first maximum is the peak."""
    m = probabilities.index(max(probabilities))
    spread = sum(p * abs(i - m) for i, p in enumerate(probabilities))
    return max(0.0, 1 - spread / mad_uniform(len(probabilities)))


def score_levels(wire_probabilities: Mapping[str, float]) -> list[float]:
    """Wire keys are strings ("0", "1", ...): order them numerically, not lexically."""
    return [wire_probabilities[k] for k in sorted(wire_probabilities, key=int)]


def expected_score(probabilities: Sequence[float]) -> float:
    return sum(i * p for i, p in enumerate(probabilities))


if __name__ == "__main__":
    print("MAD_unif by level count:",
          {n: round(mad_uniform(n), 4) for n in range(2, 11)})

    print("-- Noul")
    for p in (0.5, 0.95, 0.24, 0.02):
        print(f"p={p}: |2p-1| = {noul_confidence(p):.2f}")

    print("-- Choice")
    for label, probs, documented in [
        ("api.md department", [0.88, 0.12, 0.0], 0.81),
        ("confidence.md (0.6,0.3,0.1)", [0.6, 0.3, 0.1], 0.4),
        ("confidence.md (0.6,0.2,0.2)", [0.6, 0.2, 0.2], 0.4),
        ("explorer 'clear winner'", [0.90, 0.06, 0.04], None),
        ("explorer 'spread out'", [0.40, 0.33, 0.27], None),
        ("explorer 'even split'", [1 / 3] * 3, None),
        ("primitives/choice four-way", [0.34, 0.4, 0.02, 0.24], 0.2),
    ]:
        c = choice_confidence(probs)
        top, second = sorted(probs, reverse=True)[:2]
        ratio = top / second if second else float("inf")
        print(f"{label}: confidence={c:.4f} documented={documented} "
              f"top={top} top/second={ratio:.2f}")

    print("-- Score")
    for label, probs, documented in [
        ("api.md frustration", [0.0, 0.95, 0.05], 0.92),
        ("bug_severity", [0.0, 0.57, 0.43], 0.35),
        ("neighbours (0,.5,.5)", [0.0, 0.5, 0.5], 0.25),
        ("ends (.5,0,.5)", [0.5, 0.0, 0.5], 0.0),
        ("5-level 'clear peak'", [0, 0.05, 0.90, 0.05, 0], None),
        ("5-level 'split neighbours'", [0, 0.55, 0.45, 0, 0], None),
        ("5-level 'split ends'", [0.55, 0, 0, 0, 0.45], None),
        ("5-level 'even'", [0.2] * 5, None),
    ]:
        print(f"{label}: score={expected_score(probs):.2f} "
              f"confidence={score_confidence(probs):.4f} "
              f"choice-formula={choice_confidence(probs):.4f} documented={documented}")

    print("-- wire keys")
    wire = {"0": 0.0, "1": 0.95, "2": 0.05}
    print(score_levels(wire), round(score_confidence(score_levels(wire)), 4))
    ten = {str(i): (0.91 if i == 9 else 0.01) for i in range(10)}
    print("10 levels, lexical order == numeric order:",
          sorted(ten) == sorted(ten, key=int))

    print("-- tie sensitivity (5 levels, two equal peaks)")
    p = [0.35, 0.0, 0.35, 0.30, 0.0]
    for m in (0, 2):
        spread = sum(q * abs(i - m) for i, q in enumerate(p))
        print(f"peak={m}: confidence={max(0.0, 1 - spread / mad_uniform(5)):.4f}")
```

Output (2026-10-03):

```text
MAD_unif by level count: {2: 0.5, 3: 0.6667, 4: 1.0, 5: 1.2, 6: 1.5, 7: 1.7143, 8: 2.0, 9: 2.2222, 10: 2.5}
-- Noul
p=0.5: |2p-1| = 0.00
p=0.95: |2p-1| = 0.90
p=0.24: |2p-1| = 0.52
p=0.02: |2p-1| = 0.96
-- Choice
api.md department: confidence=0.8200 documented=0.81 top=0.88 top/second=7.33
confidence.md (0.6,0.3,0.1): confidence=0.4000 documented=0.4 top=0.6 top/second=2.00
confidence.md (0.6,0.2,0.2): confidence=0.4000 documented=0.4 top=0.6 top/second=3.00
explorer 'clear winner': confidence=0.8500 documented=None top=0.9 top/second=15.00
explorer 'spread out': confidence=0.1000 documented=None top=0.4 top/second=1.21
explorer 'even split': confidence=0.0000 documented=None top=0.3333333333333333 top/second=1.00
primitives/choice four-way: confidence=0.2000 documented=0.2 top=0.4 top/second=1.18
-- Score
api.md frustration: score=1.05 confidence=0.9250 choice-formula=0.9250 documented=0.92
bug_severity: score=1.43 confidence=0.3550 choice-formula=0.3550 documented=0.35
neighbours (0,.5,.5): score=1.50 confidence=0.2500 choice-formula=0.2500 documented=0.25
ends (.5,0,.5): score=1.00 confidence=0.0000 choice-formula=0.2500 documented=0.0
5-level 'clear peak': score=2.00 confidence=0.9167 choice-formula=0.8750 documented=None
5-level 'split neighbours': score=1.45 confidence=0.6250 choice-formula=0.4375 documented=None
5-level 'split ends': score=1.80 confidence=0.0000 choice-formula=0.4375 documented=None
5-level 'even': score=2.00 confidence=0.0000 choice-formula=0.0000 documented=None
-- wire keys
[0.0, 0.95, 0.05] 0.925
10 levels, lexical order == numeric order: True
-- tie sensitivity (5 levels, two equal peaks)
peak=0: confidence=0.0000
peak=2: confidence=0.1667
```

Reading the output:

- `MAD_unif` for 3 levels is 2/3 and for 5 levels is 1.2, matching the docs ("It divides that by 1.2"). Every value from 2 to 10 levels is listed, so you can recompute by hand.
- Every documented value is reproduced within the 2-decimal rounding of the returned probabilities. That includes bug_severity (0, 0.57, 0.43) → 0.355 ("≈ 0.35" in the docs), neighbours (0, .5, .5) → 0.25, and ends (.5, 0, .5) → 0 against 0.25 under the Choice formula.
- **Tie sensitivity.** With two equal peaks the result depends on which one is taken as *m*. The docs' code takes the first maximum, but the server's rule is undocumented. For (0.35, 0, 0.35, 0.30, 0) the two choices give 0 against 0.17. If you gate on Score confidence, treat near-ties as low confidence regardless of the returned value.

*Executed* - consistency check of all 16 Choice/Score answers that appear in JSON code blocks on the live docs pages (`scan_examples.py`, which applies the same formulas). Every returned `confidence` is within ±0.011 of the value recomputed from the rounded probabilities, and every Score answer's `score` equals Σ i·pᵢ:

```text
OK  api.md choice [0.88, 0.12, 0.0] doc= 0.81 computed= 0.82
OK  api.md score [0.0, 0.95, 0.05] doc= 0.92 computed= 0.925  score=1.05 E[level]=1.050
OK  introduction_quickstart.md choice [0.85, 0.0, 0.15] doc= 0.78 computed= 0.775
OK  primitives_choice.md choice [0.04, 0.35, 0.61] doc= 0.42 computed= 0.415
OK  primitives_choice.md choice [0.0, 0.26, 0.0, 0.0, 0.74] doc= 0.67 computed= 0.675
OK  primitives_choice.md choice [0.34, 0.4, 0.02, 0.24] doc= 0.2 computed= 0.2
OK  primitives_choice.md choice [0.84, 0.16, 0.0] doc= 0.76 computed= 0.76
OK  primitives_score.md score [0.0, 0.57, 0.43] doc= 0.35 computed= 0.355  score=1.43 E[level]=1.430
OK  primitives_score.md score [0.0, 0.76, 0.24] doc= 0.64 computed= 0.64  score=1.24 E[level]=1.240
OK  primitives_score.md score [0.0, 0.72, 0.28] doc= 0.58 computed= 0.58  score=1.28 E[level]=1.280
OK  primitives_score.md score [0.0, 0.91, 0.09] doc= 0.87 computed= 0.865  score=1.09 E[level]=1.090
... (5 further answers at confidence 1.0, all OK)
16 answers checked
```

The script parses only fenced `json` code blocks. Four further Score answers on `/primitives/score` sit in its interactive component and prose rather than in a JSON block: (0, .14, .86, 0, 0) → 0.89, (0, 0, .48, .52) → 0.52, (0, .74, .26) → 0.61 and (.45, .55, 0) → 0.33. Re-checked by hand on 2026-10-03, they recompute to 0.883, 0.52, 0.61 and 0.325, and their `score` values equal Σ i·pᵢ. This confirms that the documented formulas are the ones the API's examples were produced with. Calibration (whether p = 0.8 is right 80 % of the time) is a separate question, covered in the `decision-calibration` KB docs.

---

## Example: curl

*Executed* - against the local stand-in server (`mock_server.py`) with `TYPESAFE_BASE_URL=http://127.0.0.1:8765` and `TYPESAFE_API_KEY=test-key`. Without the override, the same script targets `https://api.typesafe.ai`. The request body exercises a structured `instructions` object, structured Choice criteria with a `null` option, Noul criteria and a Score. It validated against the SDK's OpenAPI-derived wire model.

`request.json`:

```json
{
  "state": "Help! My payouts have been failing for 3 days.",
  "model": "jev-1.13.0",
  "questions": {
    "is_urgent": {
      "type": "noul",
      "instructions": "Does this convey urgency?",
      "criteria": {"true": "Explicitly time-sensitive", "false": "No urgency expressed"}
    },
    "department": {
      "type": "choice",
      "instructions": {
        "question": "Which team should handle this message?",
        "focus": "Classify the customer's primary request, not every topic mentioned."
      },
      "criteria": {
        "billing": {"what": "Payments, invoicing, refunds", "not_for": "Bugs or outages"},
        "technical": {"what": "Bugs, outages, integrations", "not_for": "Charges or refunds"},
        "sales": null
      }
    },
    "frustration": {
      "type": "score",
      "instructions": "How frustrated is the customer?",
      "criteria": ["Calm", "Frustrated", "Very angry"]
    }
  }
}
```

`curl_examples.sh`:

```bash
BASE_URL="${TYPESAFE_BASE_URL:-https://api.typesafe.ai}"

# 1. Evaluate: print the body, keep the response headers for the request id.
curl -sS --fail-with-body -X POST "$BASE_URL/v1/systemone" \
  -H "Authorization: Bearer $TYPESAFE_API_KEY" \
  -H "Content-Type: application/json" \
  -D headers.txt \
  --data @request.json
echo
grep -i '^x-typesafe-request-id' headers.txt

# 2. List models.
curl -sS "$BASE_URL/v1/models" -H "Authorization: Bearer $TYPESAFE_API_KEY"
echo

# 3. A Score with no levels: the 422 body names the field.
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE_URL/v1/systemone" \
  -H "Authorization: Bearer $TYPESAFE_API_KEY" -H "Content-Type: application/json" \
  -d '{"state":"x","model":"jev-latest","questions":{"urgency":{"type":"score","criteria":[]}}}'
```

Output against the mock (answers are the docs' example answers, not a model evaluation):

```text
{"model": "jev-1.13.0", "answers": {"is_urgent": {"type": "noul", "noul": 0.95}, "department": {"type": "choice", "choice": "billing", "probabilities": {"billing": 0.88, "technical": 0.12, "sales": 0.0}, "confidence": 0.81}, "frustration": {"type": "score", "score": 1.05, "legend": {"0": "Calm", "1": "Frustrated", "2": "Very angry"}, "probabilities": {"0": 0.0, "1": 0.95, "2": 0.05}, "confidence": 0.92}}, "usage": {"input_tokens": 318, "output_tokens": 34}}
x-typesafe-request-id: req_mock_1
{"models": [{"name": "jev-latest", "description": "General-purpose system one model.", "release_date": "2026-09-15"}]}
422
```

`--fail-with-body` makes curl exit non-zero on a 4xx/5xx while still printing the error body. That matters with a 422, where the body is the diagnosis.

---

## Example: Raw Python with httpx

No SDK. It mirrors the official SDKs' defaults (retry set, backoff, `retry-after-ms` before `retry-after`, a 10 s per-attempt timeout) and adds local checks for the limits that OpenAPI does not encode.

*Executed* - `jev_http.py`, httpx 0.28.1, Python 3.12:

```python
"""Minimal raw-HTTP client for POST /v1/systemone and GET /v1/models (no TypeSafe SDK)."""

import email.utils
import os
import random
import time
from collections.abc import Callable, Mapping
from typing import Any

import httpx

BASE_URL = os.environ.get("TYPESAFE_BASE_URL", "https://api.typesafe.ai")
REQUEST_ID_HEADER = "x-typesafe-request-id"
# Same retryable set as both official SDKs (408, 429, every 5xx), which covers 529.
RETRYABLE = {408, 429, *range(500, 600)}
MAX_CHOICE_OPTIONS = 255  # docs.typesafe.ai/api; not encoded in openapi.json
MIN_SCORE_LEVELS, MAX_SCORE_LEVELS = 2, 10  # docs: "at least two", "up to 10"


class JevAPIError(Exception):
    def __init__(self, status: int, body: Any, request_id: str | None) -> None:
        self.status, self.body, self.request_id = status, body, request_id
        super().__init__(f"{status} {describe_error(body)} (request_id={request_id})")


def describe_error(body: Any) -> str:
    """Flatten a FastAPI-style 422 body: {"detail": [{"loc": [...], "msg": ..., "type": ...}]}."""
    if isinstance(body, dict) and isinstance(body.get("detail"), list):
        parts = []
        for err in body["detail"]:
            path = ".".join(str(p) for p in err.get("loc", []) if p != "body")
            parts.append(f"{path}: {err.get('msg')} [{err.get('type')}]")
        return "; ".join(parts)
    if isinstance(body, dict) and isinstance(body.get("detail"), str):
        return body["detail"]
    return str(body)[:200]


def check_questions(questions: Mapping[str, Mapping[str, Any]]) -> None:
    """Fail locally on the limits the docs state but the OpenAPI spec does not enforce."""
    if not questions:
        raise ValueError("questions must contain at least one entry")
    for name, q in questions.items():
        kind, criteria = q.get("type"), q.get("criteria")
        if kind == "choice":
            if not isinstance(criteria, Mapping) or not criteria:
                raise ValueError(f"{name}: choice criteria must be a non-empty object")
            if len(criteria) > MAX_CHOICE_OPTIONS:
                raise ValueError(f"{name}: {len(criteria)} options > {MAX_CHOICE_OPTIONS}")
        elif kind == "score":
            if not isinstance(criteria, list):
                raise ValueError(f"{name}: score criteria must be an array")
            if not MIN_SCORE_LEVELS <= len(criteria) <= MAX_SCORE_LEVELS:
                raise ValueError(f"{name}: {len(criteria)} levels, need 2..10")
        elif kind == "noul":
            if criteria is not None and not set(criteria) <= {"true", "false"}:
                raise ValueError(f"{name}: noul criteria keys are 'true'/'false' only")
        else:
            raise ValueError(f"{name}: unknown question type {kind!r}")


def retry_after_seconds(headers: httpx.Headers) -> float | None:
    """`retry-after-ms` first, then `retry-after` as seconds or an HTTP date (SDK order)."""
    raw_ms = headers.get("retry-after-ms")
    if raw_ms is not None:
        try:
            if float(raw_ms) >= 0:
                return float(raw_ms) / 1000
        except ValueError:
            pass
    raw = headers.get("retry-after")
    if raw is None:
        return None
    try:
        seconds = float(raw)
        return seconds if seconds >= 0 else None
    except ValueError:
        try:
            return max(0.0, email.utils.parsedate_to_datetime(raw).timestamp() - time.time())
        except (TypeError, ValueError):
            return None


def backoff_seconds(attempt: int, initial: float = 0.5, cap: float = 5.0, jitter: float = 0.25) -> float:
    """SDK defaults: 0.5 s doubling to 5 s, minus up to 25 % jitter."""
    delay = min(cap, initial * 2**attempt)
    return delay * (1 - random.random() * jitter)


def _send(
    client: httpx.Client,
    method: str,
    path: str,
    body: Any | None,
    max_retries: int,
    sleep: Callable[[float], None],
) -> httpx.Response:
    for attempt in range(max_retries + 1):
        last = attempt == max_retries
        try:
            response = client.request(method, path, json=body)
        except httpx.TransportError:  # includes timeouts
            if last:
                raise
            sleep(backoff_seconds(attempt))
            continue
        if response.is_success:
            return response
        if response.status_code in RETRYABLE and not last:
            hinted = retry_after_seconds(response.headers)
            sleep(hinted if hinted is not None and hinted <= 60 else backoff_seconds(attempt))
            continue
        try:
            error_body = response.json()
        except ValueError:
            error_body = response.text or None
        raise JevAPIError(response.status_code, error_body, response.headers.get(REQUEST_ID_HEADER))
    raise AssertionError("unreachable")


def make_client(api_key: str | None = None, transport: httpx.BaseTransport | None = None) -> httpx.Client:
    key = api_key or os.environ["TYPESAFE_API_KEY"]
    return httpx.Client(
        base_url=BASE_URL,
        headers={"Authorization": f"Bearer {key}", "Accept": "application/json"},
        timeout=httpx.Timeout(10.0),  # per attempt, like the SDK default
        transport=transport,
    )


def system_one(
    client: httpx.Client,
    state: str | Mapping[str, Any] | list[Any],
    questions: Mapping[str, Mapping[str, Any]],
    model: str = "jev-latest",
    max_retries: int = 2,
    sleep: Callable[[float], None] = time.sleep,
) -> tuple[dict[str, Any], str | None]:
    """Return (decoded body, x-typesafe-request-id)."""
    check_questions(questions)
    body = {"state": state, "model": model, "questions": dict(questions)}
    response = _send(client, "POST", "/v1/systemone", body, max_retries, sleep)
    return response.json(), response.headers.get(REQUEST_ID_HEADER)


def list_models(client: httpx.Client, sleep: Callable[[float], None] = time.sleep) -> list[dict[str, str]]:
    return _send(client, "GET", "/v1/models", None, 2, sleep).json()["models"]
```

*Executed* - the offline test driver (`test_jev_http.py`). It plays the API with `httpx.MockTransport`: success with a request id, 429 (`retry-after: 2`) then 529 (`retry-after-ms: 250`) then 200, 422 not retried, 401 not retried, a read timeout retried, the local limit checks, and `GET /v1/models`. Abridged to the assertions that matter:

```python
def test_429_then_success_honours_retry_after():
    handler, seen = scripted(
        httpx.Response(429, json={"detail": "rate limited"}, headers={"retry-after": "2"}),
        httpx.Response(529, text="Overloaded", headers={"retry-after-ms": "250"}),
        httpx.Response(200, json=OK_BODY),
    )
    waits = []
    client = jev_http.make_client("k", httpx.MockTransport(handler))
    jev_http.system_one(client, "x", QUESTIONS, sleep=waits.append)
    assert waits == [2.0, 0.25], waits
    assert len(seen) == 3
```

Output:

```text
choice recomputed 0.82 returned 0.81
score recomputed 0.925 returned 0.92
PASS test_success_and_request_id
PASS test_429_then_success_honours_retry_after
422 -> 422 questions.urgency.score.criteria: List should have at least 1 item after validation, not 0 [too_short] (request_id=req_9)
PASS test_422_detail_is_flattened_and_not_retried
PASS test_401_not_retried
local reject: s: 1 levels, need 2..10
local reject: s: 11 levels, need 2..10
local reject: c: 256 options > 255
local reject: n: noul criteria keys are 'true'/'false' only
local reject: questions must contain at least one entry
PASS test_local_limits
PASS test_models
PASS test_timeout_is_retried
```

Design notes:

- `retry_after_seconds` ignores a hint above 60 s and falls back to backoff, like the JS SDK's `maxRetryAfterMs`. The Python SDK has no such cap but bounds the whole call to 30 s.
- `check_questions` enforces 2..10 levels. That is stricter than the Python SDK (≥ 1) and matches the docs and the JS SDK.
- The `422` message drops `"body"` from `loc` but keeps the `score` tag. Drop it as well if you map errors back onto your own question objects.

---

## Example: Raw fetch (TypeScript)

*Type-checked and executed* - `jev-fetch.ts`, `tsc` 7.0.2 with `strict`, `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`, `erasableSyntaxOnly`; run with Node 22.22.0 (native type stripping):

```ts
// Raw fetch client for POST /v1/systemone (no TypeSafe SDK). Wire types follow openapi.json 0.2.0.

type Json = string | number | boolean | null | Json[] | { [key: string]: Json };
type Entry = string | { [key: string]: Json } | Json[];

export type Question =
  | { type: "noul"; instructions?: Entry | null; criteria?: { true?: Entry | null; false?: Entry | null } | null }
  | { type: "choice"; instructions?: Entry | null; criteria: { [option: string]: Entry | null } }
  | { type: "score"; instructions?: Entry | null; criteria: Entry[] };

export type Answer =
  | { type: "noul"; noul: number }
  | { type: "choice"; choice: string; probabilities: Record<string, number>; confidence: number }
  | {
      type: "score";
      score: number;
      legend: Record<string, Entry>; // keys are "0", "1", ... (strings on the wire)
      probabilities: Record<string, number>;
      confidence: number;
    };

export interface SystemOneResponse {
  model: string; // the versioned id that answered, e.g. "jev-1.13.0"
  answers: Record<string, Answer>;
  usage: { input_tokens: number; output_tokens: number };
}

export interface ValidationIssue {
  loc: (string | number)[];
  msg: string;
  type: string;
  input?: unknown;
  ctx?: Record<string, unknown>;
}

export class JevAPIError extends Error {
  readonly status: number;
  readonly body: unknown;
  readonly requestId: string | null;
  constructor(status: number, body: unknown, requestId: string | null) {
    super(`${status} ${describe(body)} (request_id=${requestId})`);
    this.status = status;
    this.body = body;
    this.requestId = requestId;
  }
}

function describe(body: unknown): string {
  const detail = (body as { detail?: unknown } | null)?.detail;
  if (Array.isArray(detail)) {
    return (detail as ValidationIssue[])
      .map((d) => `${d.loc.filter((p) => p !== "body").join(".")}: ${d.msg} [${d.type}]`)
      .join("; ");
  }
  return typeof detail === "string" ? detail : JSON.stringify(body)?.slice(0, 200) ?? "";
}

const RETRYABLE = (status: number) => status === 408 || status === 429 || status >= 500;

function retryAfterMs(headers: Headers): number | undefined {
  const ms = headers.get("retry-after-ms");
  if (ms !== null && Number.isFinite(Number(ms)) && Number(ms) >= 0) return Number(ms);
  const raw = headers.get("retry-after");
  if (raw === null) return undefined;
  const seconds = Number(raw);
  if (Number.isFinite(seconds)) return seconds >= 0 ? seconds * 1000 : undefined;
  const date = Date.parse(raw);
  return Number.isNaN(date) ? undefined : Math.max(0, date - Date.now());
}

const backoffMs = (attempt: number) => Math.min(500 * 2 ** attempt, 5000) * (1 - Math.random() * 0.25);

export interface CallOptions {
  apiKey: string;
  baseURL?: string;
  fetchImpl?: typeof fetch;
  maxRetries?: number;
  attemptTimeoutMs?: number;
  sleep?: (ms: number) => Promise<void>;
}

export async function systemOne(
  body: { state: Entry; model: string; questions: Record<string, Question> },
  opts: CallOptions,
): Promise<{ data: SystemOneResponse; requestId: string | null }> {
  const {
    apiKey,
    baseURL = "https://api.typesafe.ai",
    fetchImpl = fetch,
    maxRetries = 2,
    attemptTimeoutMs = 10_000,
    sleep = (ms) => new Promise((r) => setTimeout(r, ms)),
  } = opts;
  for (let attempt = 0; ; attempt++) {
    const last = attempt >= maxRetries;
    let res: Response;
    try {
      res = await fetchImpl(`${baseURL}/v1/systemone`, {
        method: "POST",
        headers: { Authorization: `Bearer ${apiKey}`, "Content-Type": "application/json", Accept: "application/json" },
        body: JSON.stringify(body),
        signal: AbortSignal.timeout(attemptTimeoutMs),
      });
    } catch (err) {
      if (last) throw err;
      await sleep(backoffMs(attempt));
      continue;
    }
    const requestId = res.headers.get("x-typesafe-request-id");
    if (res.ok) return { data: (await res.json()) as SystemOneResponse, requestId };
    if (RETRYABLE(res.status) && !last) {
      const hinted = retryAfterMs(res.headers);
      await sleep(hinted !== undefined && hinted <= 60_000 ? hinted : backoffMs(attempt));
      continue;
    }
    const text = await res.text();
    let parsed: unknown = text;
    try {
      parsed = JSON.parse(text);
    } catch {
      // keep plain text (e.g. a proxy's HTML error page)
    }
    throw new JevAPIError(res.status, parsed, requestId);
  }
}
```

*Type-checked and executed* - the driver (`test.ts`) injects a scripted `fetch`. Excerpt:

```ts
const out = await systemOne(body, {
  apiKey: "test-key",
  fetchImpl: fakeFetch(
    [
      json(429, { detail: "rate limited" }, { "retry-after": "1" }),
      new Response("Overloaded", { status: 529, headers: { "retry-after-ms": "250" } }),
      json(200, ok, { "x-typesafe-request-id": "req_ts_1" }),
    ],
    seen,
  ),
  sleep: async (ms) => {
    waits.push(ms);
  },
});
assert.deepEqual(waits, [1000, 250]);
```

Output:

```text
PASS retry path; choice = billing confidence = 0.81
PASS 422 -> 422 questions.urgency.score.criteria: List should have at least 1 item after validation, not 0 [too_short] (request_id=req_ts_2)
```

Keep this server-side. The key goes in a header, and the JS SDK refuses to run in a browser unless `dangerouslyAllowBrowser: true` is set. A raw `fetch` has no such guard.

---

## Discrepancies Between Docs, OpenAPI and SDKs

Checked 2026-10-03. "Docs" = live pages on docs.typesafe.ai; "OpenAPI" = `openapi.json` 0.2.0.

| # | Topic | Docs | OpenAPI 0.2.0 | SDKs / integrations | Practical rule |
|---|---|---|---|---|---|
| 1 | `instructions` required | "required" on all three types (`/api`) | optional and nullable on all three | Python, JS: optional; Python omits it from the wire when `None` | Always send it. The question ids are not seen by the model, so without `instructions` the model has nothing to answer |
| 2 | Score minimum levels | "should have at least two" | `minItems: 1` | Python ≥ 1; JS ≥ 2 (type + runtime); AI SDK provider: none | Send ≥ 2 |
| 3 | Score maximum levels | "the API accepts up to 10" | not encoded | Python, JS: not checked; AI SDK provider and Pydantic AI: 10 | Check ≤ 10 locally |
| 4 | Choice maximum options | 255 | not encoded | Python, JS: not checked; AI SDK provider and Pydantic AI: 255 | Check ≤ 255 locally |
| 5 | Status for limit violations | Error table lists only 401/422/429/529; "malformed question" → 422 | only 422 declared | Pydantic AI docs/source: "a 400 from the API" for option/level overflow; `max_tokens_exceeded` over context | Handle 400 *and* 422 as non-retryable client errors |
| 6 | `null` Score level | allowed (`/primitives/advanced` table) | items are `string \| object \| array` only | Python `ScoreModel`: no `None`; JS `EntryType` includes `null` | Never send `null` levels |
| 7 | `state: null` | not mentioned | not allowed (required, no null branch) | JS `SystemOneRequest.state: EntryType` (allows `null`); Python: no `None` | Never send `null` state |
| 8 | `legend` value type | `map<string, string>` (`/api`) | `string \| object \| array` | Python `dict[int, str \| dict \| list]`; JS typed as your level entries | Expect structured values back when you sent structured levels |
| 9 | Score keys | strings `"0"`.. | strings | Python coerces to `int`; JS types by tuple index | Sort with `key=int` on raw JSON |
| 10 | `model` in response | "reports the versioned ID that answered" | example value `"jev-latest"`; "May differ from the alias" | - | Trust the docs; log it |
| 11 | `usage` presence | required | required | Python: the `usage` object is required but its two token fields default to `None`; AI SDK: whole object `nullish` | Required in practice; code defensively |
| 12 | `retry-after-ms` | not mentioned (only `retry-after`) | no headers declared | Both SDKs honour it first | Honour both |
| 13 | Long `retry-after` | - | - | JS ignores > 60 s; Python honours any value within its 30 s total budget | Cap your wait |
| 14 | API key env var | `TYPESAFE_API_KEY` | - | `@ai-sdk/typesafe-ai` 3.0.12 reads **`TYPESAFE_AI_API_KEY`**; its `baseURL` default includes `/v1` (`https://api.typesafe.ai/v1`) | Set both env vars if you mix the AI SDK with the official SDKs |
| 15 | Requests/second | 80 (live `/models`) | - | `llms-full.txt` dump: 40 | The live page wins; the dump lags |
| 16 | Model id `jev-1.13` | used in the jaggedness page's code | - | not listed in `/models` | Use `jev-1.13.0` |
| 17 | `/v1/models` contents | "currently lists the aliases" | "models and aliases" | - | Don't validate `model` against the list |
| 18 | GET `/v1/models` 422 | - | declared | - | Unreachable FastAPI default; ignore |
| 19 | Choice confidence in `/api` example | 0.81 | - | recomputed from shown probabilities: 0.82 | Rounding, not a bug (see [Confidence](#confidence-formulas)) |
| 20 | Unknown fields in a question | - | not forbidden | Python question objects `extra="forbid"`; JS forwards anything | Don't send extra fields |
| 21 | Noul `noul()` instructions | required | optional | JS `noul(instructions?, criteria?)` | Always pass instructions |

---

## Not Documented Anywhere

Questions that no primary source answered on 2026-10-03. Test them in staging before depending on them:

- The response body shapes of 401, 429, 529 and (if it exists) 400.
- The exact status and body for a context-budget overflow. The TypeSafe docs are silent; the Pydantic AI docs say `max_tokens_exceeded`.
- Whether 1-level Scores, 0- or 1-option Choices, 11-level Scores and 256-option Choices are rejected, and with which status.
- Whether 64k/32k are binary (65,536/32,768) or decimal (64,000/32,000), and which tokenizer counts them.
- The server's tie-breaking rule for `choice` and for the Score peak *m*.
- How the server rounds values (inferred from examples: 2 decimals) and whether rounding happens before or after `score` is computed.
- Whether timed-out or retried requests are billed.
- Server handling of unknown top-level or question fields.
- Whether `jev-1.13` (without patch) or older ids such as `jev-1.12.x` are still accepted.
- Whether the server sends `retry-after-ms`, or only `retry-after`.
- Idempotency keys: none are documented. `POST /v1/systemone` has no side effects beyond billing, so retrying is safe for correctness.

---

## Sources

- API reference: https://docs.typesafe.ai/api (`.md` re-fetched 2026-10-03, identical to the earlier copy)
- Models, limits, pricing, aliases: https://docs.typesafe.ai/models (`.md` re-fetched 2026-10-03, identical)
- Confidence: https://docs.typesafe.ai/confidence (`.md` re-fetched 2026-10-03, identical)
- Structure: https://docs.typesafe.ai/primitives/advanced; backtick paths: https://docs.typesafe.ai/concepts/how-to-build-with-system-one
- State: https://docs.typesafe.ai/concepts/state; Score levels: https://docs.typesafe.ai/primitives/score; Choice: https://docs.typesafe.ai/primitives/choice
- Jaggedness (reviewed by the vendor 2026-10-02): https://docs.typesafe.ai/model-jaggedness/jev-1.13
- OpenAPI: https://api.typesafe.ai/openapi.json (version 0.2.0)
- Python SDK reference: https://docs.typesafe.ai/sdk/python/api/exceptions, https://docs.typesafe.ai/sdk/python/api/retries, https://docs.typesafe.ai/sdk/python/api/constants; source read from the installed `typesafe-sdk` 0.7.2 wheel
- JS SDK: https://docs.typesafe.ai/sdk/javascript; source read from the installed `@typesafe-ai/sdk` 0.6.0 `dist/`
- Pydantic AI TypeSafe docs (main branch) and installed `pydantic_ai/profiles/typesafe.py` (2.54.0)
- `@ai-sdk/typesafe-ai` 3.0.12 installed `dist/index.js`; `genai-prices` 0.1.9 installed `data.py`
