# TypeSafe Jev with the Vercel AI SDK (`@ai-sdk/typesafe-ai` + `experimental_evaluate`)

> Official Documentation: https://ai-sdk.dev/providers/ai-sdk-providers/typesafe-ai
> Evaluation guide: https://ai-sdk.dev/docs/ai-sdk-core/evaluation
> `experimental_evaluate` reference: https://ai-sdk.dev/docs/reference/ai-sdk-core/evaluate
> AI Gateway evaluation: https://vercel.com/docs/ai-gateway/modalities/evaluation
> AI Gateway TypeSafe-compatible API: https://vercel.com/docs/ai-gateway/sdks-and-apis/typesafe
> Source: https://github.com/vercel/ai/tree/main/packages/typesafe-ai
> Last verified: 2026-10-03
> Verified against: `ai` **7.0.127**, `@ai-sdk/typesafe-ai` **3.0.12**, `@ai-sdk/provider` 4.0.21, `@ai-sdk/provider-utils` 5.0.53, `@ai-sdk/gateway` 4.0.103, TypeScript 7.0.2 (`strict`), Node.js 22.22.0. Docs pages fetched as markdown on 2026-10-03.

## Overview

AI SDK 7 has an **experimental evaluation API**, `experimental_evaluate`, that answers named questions about one shared `state` with an *evaluation model*. Three question types exist: `choice`, `score` and `boolean`. TypeSafe's Jev is the native implementation: `@ai-sdk/typesafe-ai` maps the three types onto TypeSafe's Choice, Score and Noul primitives and sends all questions in one `POST /v1/systemone` request. OpenAI, Anthropic and Google evaluation models also exist, but they adapt structured LLM output; the AI SDK docs state their Boolean probabilities are "prompted estimates of P(true) ... not guaranteed to be calibrated" and that their Choice and Score answers "do not include probability distributions".

The value of going through the AI SDK rather than `@typesafe-ai/sdk` directly:

- one interface for Jev, LLM-based evaluators and AI Gateway string IDs, with registries and aliases;
- **response validation** in core (distributions must be complete and sum to 1 within the declared rounding, scores must match the weighted mean, choices must exist) - the TypeSafe SDK does none of this;
- AI SDK retries, lifecycle callbacks, OpenTelemetry spans and the `Experimental_EvaluationMockModelV4` test double.

The cost: a renamed answer shape (`boolean`/`probability` instead of `noul`), confidence moved to `providerMetadata`, the score `legend` dropped, a different env var and base URL, and an API marked experimental that "may change in patch releases".

This document covers the provider and the core function at the level of the shipped code; every TypeScript snippet was type-checked and the behavioural claims about retries, validation and errors were executed offline with a fake `fetch`. The TypeSafe SDK itself is covered in `typesafe-jev/javascript-sdk.md`.

---

## Table of Contents

1. [Installation and Versions](#installation-and-versions)
2. [Provider Setup: `createTypeSafeAi`](#provider-setup-createtypesafeai)
3. [`experimental_evaluate`: Parameters and Result](#experimental_evaluate-parameters-and-result)
4. [Question Types and the `boolean` -> `noul` Mapping](#question-types-and-the-boolean---noul-mapping)
5. [Criteria Forms and Input Validation](#criteria-forms-and-input-validation)
6. [Answers and Type Inference](#answers-and-type-inference)
7. [Rounding and Response Validation](#rounding-and-response-validation)
8. [Confidence via `providerMetadata`](#confidence-via-providermetadata)
9. [End-to-End with a Fake `fetch` (executed)](#end-to-end-with-a-fake-fetch-executed)
10. [Retries](#retries)
11. [Errors](#errors)
12. [Cancellation and Timeouts](#cancellation-and-timeouts)
13. [Warnings and `providerOptions`](#warnings-and-provideroptions)
14. [Testing with `Experimental_EvaluationMockModelV4`](#testing-with-experimental_evaluationmockmodelv4)
15. [Registries, Aliases and Default-Provider Strings](#registries-aliases-and-default-provider-strings)
16. [Lifecycle Callbacks and Telemetry](#lifecycle-callbacks-and-telemetry)
17. [AI Gateway](#ai-gateway)
18. [Workflow Serialization](#workflow-serialization)
19. [AI SDK Provider vs `@typesafe-ai/sdk`](#ai-sdk-provider-vs-typesafe-aisdk)
20. [What Was Not Verified](#what-was-not-verified)

---

## Installation and Versions

```sh
pnpm add @ai-sdk/typesafe-ai
```

(Verbatim from https://ai-sdk.dev/providers/ai-sdk-providers/typesafe-ai.) You also need `ai` itself. The Vercel changelog "TypeSafe AI's Jev now available on AI Gateway" (published 2026-09-16) states: "AI SDK 7.0.105 onwards supports the `evaluate` API".

| Package | Version verified | Notes |
|---|---|---|
| `ai` | 7.0.127 | exports `experimental_evaluate`, error classes, `customProvider`, `createProviderRegistry`; `ai/test` exports `Experimental_EvaluationMockModelV4` |
| `@ai-sdk/typesafe-ai` | 3.0.12 | `engines.node >= 22`; peer dependency `zod ^3.25.76 \|\| ^4.1.8` (it uses `zod/v4` to parse responses) |
| `@ai-sdk/provider` | 4.0.21 | defines `Experimental_EvaluationModelV4` |
| `@ai-sdk/gateway` | 4.0.103 | dependency of `ai`; `GatewayEvaluationModelId = 'liquid/d1' \| 'typesafe-ai/jev' \| (string & {})` |

Provider changelog highlights (from the package's `CHANGELOG.md`): 3.0.0 "Add the TypeSafe provider for `experimental_evaluate`, supporting native Choice, Score, and Boolean questions in one request, structured JSON rubrics, probability distributions, confidence metadata, and workflow serialization"; 3.0.2 "surface the `error_type` code instead of a generic failure message"; 3.0.10 "use standards-compliant User-Agent header"; later patches are dependency bumps.

---

## Provider Setup: `createTypeSafeAi`

```ts
import { createTypeSafeAi, typeSafeAi } from '@ai-sdk/typesafe-ai';

const customTypeSafe = createTypeSafeAi({
  apiKey: 'your-api-key',
});
```

(Verbatim from the provider page.)

`TypeSafeAiProviderSettings` (from `dist/index.d.ts` of 3.0.12):

| Setting | Type | Default | Behaviour (from `dist/index.js`) |
|---|---|---|---|
| `apiKey` | `string` | env `TYPESAFE_AI_API_KEY` | Resolved **lazily, on each call** (inside a headers function). A missing key does not throw at `createTypeSafeAi()`; the evaluate call rejects with `LoadAPIKeyError`. |
| `baseURL` | `string` | `https://api.typesafe.ai/v1` | **Includes `/v1`**. The model posts to `${baseURL}/systemone`. Trailing slash removed. |
| `headers` | `Record<string, string>` | - | Added to every request, after `Authorization`. |
| `fetch` | `FetchFunction` | global `fetch` | For proxies and tests. Not serialized for workflows. |

The provider object:

| Member | Result |
|---|---|
| `evaluationModel(modelId)` | `Experimental_EvaluationModelV4` with `provider: 'typesafe.evaluation'`, `specificationVersion: 'v4'`, `supportedQuestionTypes: ['choice', 'score', 'boolean']` |
| `languageModel`, `embeddingModel`, `imageModel` | throw `NoSuchModelError` |
| `specificationVersion` | `'v4'` (a `ProviderV4`, so it can be a registry member or `fallbackProvider`) |

`modelId` is typed `'jev-latest' | (string & {})` (exported as `Experimental_TypeSafeAiEvaluationModelId`): autocomplete offers `jev-latest`, any string is accepted. Per https://docs.typesafe.ai/models (2026-10-03) the useful IDs are `jev-latest` and `jev-preview` (both -> `jev-1.13.0`) and the pinned `jev-1.13.0`.

`typeSafeAi` is `createTypeSafeAi()` with defaults, created at import time; since the key is read per call, setting `TYPESAFE_AI_API_KEY` after import still works.

> **Env var trap:** the provider reads `TYPESAFE_AI_API_KEY`; TypeSafe's own SDKs read `TYPESAFE_API_KEY`. The base URLs also differ (`.../v1` here, no `/v1` in `@typesafe-ai/sdk`).

---

## `experimental_evaluate`: Parameters and Result

Parameters (reference page, confirmed against the `evaluate` declaration in `ai/dist/index.d.ts`):

| Parameter | Type | Description |
|---|---|---|
| `model` | `Experimental_EvaluationModel` (= `string \| Experimental_EvaluationModelV4`) | Required. A model instance, or a string ID resolved by `globalThis.AI_SDK_DEFAULT_PROVIDER` or, when none is set, AI Gateway. |
| `state` | `string \| object \| array` | Required, JSON-compatible. "An array is one state, not a batch of unrelated inputs." |
| `questions` | `Record<string, Experimental_EvaluationQuestion>` | Required, non-empty. Declared with a `const` type parameter, so literals are preserved without `as const`. |
| `maxRetries` | `number` | Non-negative integer, default 2. |
| `abortSignal` | `AbortSignal` | Cancels the call and pending retry delays. |
| `headers` | `Record<string, string>` | Extra HTTP headers. |
| `providerOptions` | `ProviderOptions` | `typesafe` has no options; `gateway` options apply to Gateway models. |
| `telemetry` | `TelemetryOptions` | Integrations, recording controls, function ID. |
| `runtimeContext` | `Record<string, unknown>` | Passed to callbacks; selectively included in telemetry. |
| `onStart` / `onEnd` | callbacks | See [Lifecycle callbacks](#lifecycle-callbacks-and-telemetry). |

There is **no `timeout` parameter**; see [Cancellation and timeouts](#cancellation-and-timeouts).

Result: `Promise<Experimental_EvaluationResult<QUESTIONS>>` with

| Field | Content for TypeSafe |
|---|---|
| `answers` | one typed answer per question ID |
| `usage` | `{ inputTokens, outputTokens, totalTokens }`, each possibly `undefined`; `totalTokens` only when both are known |
| `warnings` | e.g. unsupported `providerOptions.typesafe.*` keys |
| `rounding` | `{ probabilityDecimals: 2, scoreDecimals: 2 }` (always, from the provider) |
| `providerMetadata` | `{ typesafe: { confidence: { [questionId]: number } } }` |
| `response` | `{ modelId, timestamp, headers, body }` - `modelId` is TypeSafe's resolved version (e.g. `jev-1.13.0`) when the API returns `model`; `body` is the raw JSON |

Scope limits from the evaluation guide: one complete result for one shared state; no streaming, no multilabel classification, no batching of unrelated states ("Run separate calls for separate states"); no partial success - "Successful calls return an answer for every question".

---

## Question Types and the `boolean` -> `noul` Mapping

| AI SDK `type` | TypeSafe primitive | Criteria | Answer | Limits (provider page) |
|---|---|---|---|---|
| `choice` | Choice | non-empty map: option -> description (`null` allowed) | `choice` (literal union of keys), `probabilities` | 1-255 options |
| `score` | Score | ordered array, lowest first; core accepts `null` entries (executed), but the TypeSafe HTTP API reference and OpenAPI spec list only string/object/array for Score levels - live acceptance not verified | fractional `score` in `[0, levels - 1]`, `probabilities` keyed `"0"`, `"1"`, ... | 2-10 levels |
| `boolean` | Noul | optional `{ true?, false? }` | `probability` = P(true) | - |

"The SDK uses the neutral name `boolean` and maps it to TypeSafe's `noul` field" (provider page). Concretely, `doEvaluate` sends each question unchanged except `type: 'boolean'` becomes `type: 'noul'`, and converts each answer back:

| TypeSafe answer | AI SDK answer |
|---|---|
| `{ type: 'noul', noul }` | `{ type: 'boolean', probability: noul }` |
| `{ type: 'choice', choice, probabilities, confidence }` | `{ type: 'choice', choice, probabilities }` - `confidence` moves to `providerMetadata.typesafe.confidence[id]` |
| `{ type: 'score', score, legend, probabilities, confidence }` | `{ type: 'score', score, probabilities }` - `legend` is **dropped**, `confidence` moves to metadata |

Boolean probability "always means P(true), not confidence in either outcome" (provider page): 0.98 is a strong yes, 0.02 a strong no.

---

## Criteria Forms and Input Validation

"State, instructions, and descriptions accept JSON objects and arrays in addition to strings. Descriptions may be `null`. Boolean true/false criteria are optional." (provider page). Core "treats structured descriptions as content; it does not interpret their keys" (evaluation guide) - so `{ includes: [...] }` in the example below is just text-like data for the model, not a schema.

```ts
const result = await experimental_evaluate({
  model: typeSafeAi.evaluationModel('jev-latest'),
  state: {
    message: 'I was charged twice. Please refund the duplicate.',
  },
  questions: {
    department: {
      type: 'choice',
      instructions: 'Which team should handle this?',
      criteria: {
        billing: { includes: ['Charges', 'Invoices', 'Refunds'] },
        technical: ['Bugs', 'Outages'],
        other: null,
      },
    },
    severity: {
      type: 'score',
      instructions: 'How severe is the issue?',
      criteria: ['Cosmetic', 'Workaround exists', 'Blocking; no workaround'],
    },
    requestsRefund: {
      type: 'boolean',
      instructions: 'Is the customer requesting money back?',
    },
  },
});
```

(Verbatim excerpt of the provider page's example, imports omitted; the full example, with `import { typeSafeAi } from '@ai-sdk/typesafe-ai'` and `import { experimental_evaluate } from 'ai'`, type-checks against the verified versions.)

### What core validates before calling the provider (`validateEvaluationInput`, `ai` 7.0.127)

All failures throw `InvalidArgumentError` before any I/O:

| Rule | Message (`message` field) |
|---|---|
| `state` is a JSON-compatible string, plain object or array | `must be a JSON-compatible string, object, or array` |
| `questions` is a non-empty plain object | `must be a nonempty question map` |
| every question has JSON-compatible `instructions` (string/object/array; **`null` and missing are rejected**) | `instructions must be a JSON-compatible string, object, or array` |
| `choice` criteria: non-empty plain object | `choice criteria must be a nonempty option map` |
| `score` criteria: array with at least two entries | `score criteria must contain at least two ordered levels` |
| `boolean` criteria: absent, or an object with only `true`/`false` keys | `boolean criteria may only describe true and false` |
| any other `type` | `question type must be choice, score, or boolean` |
| each description is `null` or JSON-compatible string/object/array | `criteria descriptions must be JSON-compatible strings, objects, arrays, or null` |

"JSON-compatible" excludes functions, class instances (anything whose prototype is not `Object.prototype` or `null` - so `Date`, `Map`), cycles, symbol keys, `undefined` values and non-finite numbers. Serialize dates to strings yourself.

Note the difference from `@typesafe-ai/sdk`, where `noul()` defaults `instructions` to `null` and the HTTP docs mark it required: here `instructions` is mandatory for every type.

### What the TypeSafe provider adds (`doEvaluate`)

| Rule | Error |
|---|---|
| a Choice with more than 255 options | `InvalidArgumentError`, message `TypeSafe Choice questions support at most 255 options.` |
| a Score with more than 10 levels | `InvalidArgumentError`, message `TypeSafe Score questions support at most 10 levels.` |

Both throw inside the retry wrapper but are not retryable, so they surface unchanged on the first attempt with zero HTTP calls (executed below).

Finally, `supportedQuestionTypes` is checked by core before the provider is called; any unsupported type "fails the entire call with `Experimental_EvaluationUnsupportedQuestionTypeError`". The TypeSafe model supports all three, so this matters only when you swap in a model that does not.

---

## Answers and Type Inference

The answer type is derived from each question (from `ai/dist/index.d.ts`):

```ts
type EvaluationAnswer<QUESTION extends EvaluationQuestion> = QUESTION extends {
  type: 'choice';
  criteria: infer CRITERIA;
} ? {
  type: 'choice';
  choice: Extract<keyof CRITERIA, string>;
  probabilities?: Record<Extract<keyof CRITERIA, string>, number>;
} : QUESTION extends {
  type: 'score';
} ? {
  type: 'score';
  score: number;
  probabilities?: Record<string, number>;
} : {
  type: 'boolean';
  probability: number;
};
```

(Verbatim from `ai` 7.0.127 `dist/index.d.ts`.)

Consequences:

- `answers.x.choice` is the literal union of your option keys - a `switch` over it is exhaustiveness-checked.
- `probabilities` is **optional in the type** for Choice and Score, because LLM-based evaluation models do not return it. With TypeSafe it is always present (the provider's response schema requires it), but you still need `?.` or a non-null assertion. Score `probabilities` are `Record<string, number>` - unlike `@typesafe-ai/sdk`, the level keys are not narrowed to `"0" | "1" | ...`.
- There is no `confidence` and no `legend` on answers.

Reading the selected option's probability, verbatim from the evaluation guide:

```ts
const answer = result.answers.department;
const selectedProbability = answer.probabilities?.[answer.choice];
if (selectedProbability != null && selectedProbability >= 0.9) {
  // Route automatically; otherwise use the application's review path.
}
```

---

## Rounding and Response Validation

"TypeSafe returns scores and probabilities rounded to two decimal places. `result.rounding` reports this precision so the SDK can account for rounding when checking distribution sums and weighted means. The returned values are preserved unchanged; rounding means probabilities may not add up to exactly one." (provider page)

The TypeSafe provider always declares `rounding: { probabilityDecimals: 2, scoreDecimals: 2 }`. Core then validates every answer (`validateEvaluationAnswers`, `ai` 7.0.127); a failure throws `InvalidResponseDataError` **after** the call and is **not retried**:

| Check | Tolerance with TypeSafe's rounding | Message |
|---|---|---|
| exactly one answer per question ID, matching `type` | - | `Evaluation must return exactly one answer for every question.` / `Question "<id>" returned an answer with the wrong type.` |
| Choice: `choice` is one of the criteria keys | - | `Question "<id>" selected an unknown option.` |
| Choice/Score distribution has exactly the expected keys, each a finite number in `[0, 1]` | - | `Question "<id>" must have a complete distribution of finite probabilities in [0, 1].` |
| distribution sums to 1 | `1e-6 + n * 0.005` for `n` options/levels (3 options: +/-0.015) | `Question "<id>" probabilities must sum to 1 within the declared rounding precision.` |
| Choice: the selected option has maximal probability | `1e-6` | `Question "<id>" did not select a highest-probability option.` |
| Score in `[0, levels - 1]` | - | `Question "<id>" score must be in [0, <max>].` |
| Score equals the probability-weighted mean | `1e-6 + sum(i * 0.005) + 0.005` | `Question "<id>" score must equal the probability-weighted mean within the declared rounding precision.` |
| Boolean probability finite in `[0, 1]` | - | `Question "<id>" must return P(true) as a finite probability in [0, 1].` |

The documented rule in prose: "Validation then also allows half a unit in the last decimal place per rounded probability or score, accumulated over the sum or weighted mean. For example, probabilities rounded to two decimal places can sum to `0.99` even when their unrounded values sum to one." "Invalid output is rejected; native values are preserved, never silently normalized." (evaluation guide)

Practical effects:

- Never assume `sum(probabilities) === 1` or that a `0` probability is exactly zero.
- A mock or fixture you write must satisfy these checks (the executed tests below include one that sums to 0.99 and passes, and one that sums to 0.95 and fails).
- Ties are possible with two-decimal values; the "highest-probability" check uses `>` with tolerance, so a tie with the selected option passes.

---

## Confidence via `providerMetadata`

"Confidence is a separate TypeSafe statistic, available under `result.providerMetadata.typesafe.confidence[questionId]` for Choice and Score answers. Boolean probability always means P(true), not confidence in either outcome. Choose decision thresholds in application code." (provider page). The evaluation guide adds: "It is not the selected option's probability or a portable confidence measure."

Implementation details (provider 3.0.12): the map contains only Choice/Score questions whose TypeSafe answer had a non-null `confidence`; Boolean questions never appear. `providerMetadata` is typed as generic JSON, so cast when reading:

```ts
const confidence = r.providerMetadata?.typesafe?.confidence as Record<string, number> | undefined;
```

(Line from the executed `v03-mock-registry.mts` below.)

How the number relates to the distribution: TypeSafe's confidence page (https://docs.typesafe.ai/confidence, fetched 2026-10-03) publishes the formulas and calls them "exact": Choice confidence is `(p_max - 1/n) / (1 - 1/n)` for `n` options, and Score confidence is `max(0, 1 - sum_i p_i * |i - m| / MAD_unif)` - the probability-weighted distance from the most likely level `m`, normalised by the same average distance for an even spread (`MAD_unif = (1/n) * sum_i |i - (n-1)/2|`). Recomputing from the two-decimal `probabilities` can differ from the returned value in the last digit (the API example pairs `p_max = 0.88` over three options with 0.81, where the formula gives 0.82), so threshold on the returned field.

Which to threshold on:

| Signal | Use |
|---|---|
| `answers.x.probability` (boolean) | P(true); route on `>= t_yes` / `<= t_no` with a review band between |
| `answers.x.probabilities?.[answers.x.choice]` | selected-option probability; portable across providers that return distributions |
| `providerMetadata.typesafe.confidence.x` | TypeSafe's spread-aware statistic for Choice/Score; only TypeSafe (and Gateway with native fallbacks) provides it |

Thresholds should be fitted on labelled examples from your workflow; the AI SDK docs say so explicitly ("using labeled data from the task rather than assuming that the same threshold behaves identically across providers"). See the decision-calibration KB documents for the fitting procedure.

---

## End-to-End with a Fake `fetch` (executed)

The following ran green offline. The fake `fetch` returns the TypeSafe response shape documented at https://docs.typesafe.ai/api (two-decimal values, `legend` and `confidence` included), so the test exercises the real provider code path: request mapping, Zod parsing, answer conversion, core validation, usage and metadata.

```ts
// Executed offline: ai 7.0.127 + @ai-sdk/typesafe-ai 3.0.12, Node 22.22.0 (v01-e2e.mts)
import assert from "node:assert/strict";
import { createTypeSafeAi } from "@ai-sdk/typesafe-ai";
import { experimental_evaluate } from "ai";

let sentUrl = "";
let sentHeaders: Record<string, string> = {};
let sentBody: any;

// Fake fetch returning the documented TypeSafe response shape (values rounded to 2 dp).
const fakeFetch: typeof globalThis.fetch = async (input, init) => {
  sentUrl = String(input);
  sentHeaders = init?.headers as Record<string, string>;
  sentBody = JSON.parse(String(init?.body));
  return new Response(
    JSON.stringify({
      model: "jev-1.13.0",
      answers: {
        department: {
          type: "choice",
          choice: "billing",
          probabilities: { billing: 0.88, technical: 0.1, other: 0.01 }, // sums to 0.99
          confidence: 0.81,
        },
        severity: {
          type: "score",
          score: 1.05,
          legend: { "0": "Cosmetic", "1": "Workaround exists", "2": "Blocking; no workaround" },
          probabilities: { "0": 0.0, "1": 0.95, "2": 0.05 },
          confidence: 0.92,
        },
        requestsRefund: { type: "noul", noul: 0.97 },
      },
      usage: { input_tokens: 318, output_tokens: 34 },
    }),
    { status: 200, headers: { "content-type": "application/json", "x-typesafe-request-id": "req_1" } },
  );
};

const typesafe = createTypeSafeAi({ apiKey: "test-key", fetch: fakeFetch });

const result = await experimental_evaluate({
  model: typesafe.evaluationModel("jev-latest"),
  state: { message: "I was charged twice. Please refund the duplicate." },
  questions: {
    department: {
      type: "choice",
      instructions: "Which team should handle this?",
      criteria: {
        billing: { includes: ["Charges", "Invoices", "Refunds"] },
        technical: ["Bugs", "Outages"],
        other: null,
      },
    },
    severity: {
      type: "score",
      instructions: "How severe is the issue?",
      criteria: ["Cosmetic", "Workaround exists", "Blocking; no workaround"],
    },
    requestsRefund: {
      type: "boolean",
      instructions: "Is the customer requesting money back?",
    },
  },
});

// Typed answers: `choice` is the literal union of the criteria keys.
const dept: "billing" | "technical" | "other" = result.answers.department.choice;
assert.equal(dept, "billing");
assert.equal(result.answers.severity.score, 1.05);
assert.equal(result.answers.requestsRefund.probability, 0.97);
assert.deepEqual(result.answers.department.probabilities, { billing: 0.88, technical: 0.1, other: 0.01 });

// Confidence is not on the answer; it is provider metadata keyed by question ID.
assert.deepEqual(result.providerMetadata?.typesafe?.confidence, { department: 0.81, severity: 0.92 });
assert.deepEqual(result.rounding, { probabilityDecimals: 2, scoreDecimals: 2 });
assert.deepEqual(result.usage, { inputTokens: 318, outputTokens: 34, totalTokens: 352 });
assert.equal(result.response.modelId, "jev-1.13.0"); // the resolved version, not the alias
assert.equal(result.response.headers?.["x-typesafe-request-id"], "req_1");
assert.deepEqual(result.warnings, []);
assert.ok(!("legend" in result.answers.severity)); // the TypeSafe legend is dropped

// What went over the wire.
assert.equal(sentUrl, "https://api.typesafe.ai/v1/systemone");
assert.equal(sentBody.model, "jev-latest");
assert.equal(sentBody.questions.requestsRefund.type, "noul"); // 'boolean' renamed to 'noul'
assert.deepEqual(sentBody.questions.department.criteria.billing, { includes: ["Charges", "Invoices", "Refunds"] });
assert.equal(sentHeaders.authorization, "Bearer test-key");
console.log("user-agent:", sentHeaders["user-agent"]);
console.log(JSON.stringify(result.answers));
console.log("ok");
```

Output:

```text
user-agent: ai/7.0.127 ai-sdk-provider-utils/5.0.53 node.js/22
{"department":{"type":"choice","choice":"billing","probabilities":{"billing":0.88,"technical":0.1,"other":0.01}},"severity":{"type":"score","score":1.05,"probabilities":{"0":0,"1":0.95,"2":0.05}},"requestsRefund":{"type":"boolean","probability":0.97}}
ok
```

Observations from this run:

- The request body is `{ model, state, questions }` with the `boolean` question renamed; header names are lower-cased by `provider-utils`.
- The provider computes a `user-agent` suffix `ai-sdk-typesafe-ai/3.0.12`, but in this combination the header that reached `fetch` was core's (`ai/7.0.127 ...`) - core passes its own `user-agent` in the per-call headers, which override provider headers in `combineHeaders`. Harmless, but do not rely on the provider suffix for log filtering.
- `response.headers` exposes the TypeSafe request ID (`x-typesafe-request-id`) for support tickets.

---

## Retries

There is "no provider-side retry loop"; core retries according to `maxRetries` (default 2). From `ai` 7.0.127 / `provider-utils` 5.0.53:

| Aspect | Behaviour |
|---|---|
| Attempts | `1 + maxRetries` (default 3). `maxRetries` must be a non-negative integer (`InvalidArgumentError` otherwise). |
| What is retried | `APICallError` with `isRetryable === true`: status **408, 409, 429, or >= 500** (includes TypeSafe's 529); also connection failures: a `TypeError('fetch failed')` **with a `cause`** (what Node's undici throws) becomes a retryable `APICallError` `Cannot connect to API: <cause>`; the same `TypeError` without a `cause` is passed through unwrapped and not retried (both executed below). For Gateway models, a `GatewayError` with `isRetryable === true` is retried as well. |
| Not retried | other 4xx, `InvalidArgumentError`, `InvalidResponseDataError` (validation runs after the retry wrapper), abort errors. |
| Backoff | exponential: 2000 ms, then 4000 ms (initial 2000 ms, factor 2), no jitter. |
| Server hints | `retry-after-ms` (ms), else `retry-after` (seconds or HTTP date) - used when `0 <= ms < 60000`, or when it is shorter than the backoff. |
| After exhaustion | throws **`RetryError`** (`reason: 'maxRetriesExceeded'`, `errors[]`, `lastError`), *not* the `APICallError`. |
| Non-retryable error after a retry | `RetryError` with `reason: 'errorNotRetryable'` - unless it happens on the **final** attempt: the attempt-count check runs before the retryability check, so e.g. 429, 429, then 400 with `maxRetries: 2` yields `reason: 'maxRetriesExceeded'` (both executed). |
| Non-retryable error on the first attempt, or `maxRetries: 0` | the original error. |

So `APICallError.isInstance(err)` alone misses persistent 429/5xx failures; also check `RetryError.isInstance(err)` and inspect `err.lastError`. Compare: `@typesafe-ai/sdk` rethrows its last typed error after retries, backs off from 500 ms with jitter, and does **not** retry 409.

---

## Errors

| Error (all exported from `ai`) | When | Retried |
|---|---|---|
| `InvalidArgumentError` | core input validation; provider 255/10 limits; bad `maxRetries` | no |
| `Experimental_EvaluationUnsupportedQuestionTypeError` | model lacks a question type; has `questionId`, `questionType`, `provider`, `modelId` | no (before I/O) |
| `NoSuchModelError` (`modelType: 'evaluationModel'`) | string ID cannot be resolved; registry has no such model; provider has no `evaluationModel` | no |
| `NoSuchProviderError` | registry prefix unknown | no |
| `UnsupportedModelVersionError` | a model that is not `specificationVersion: 'v4'` | no |
| `LoadAPIKeyError` | no `apiKey` and no `TYPESAFE_AI_API_KEY` at call time | no |
| `APICallError` | non-2xx from TypeSafe; `statusCode`, `responseHeaders`, `responseBody`, `isRetryable`, `url`, `requestBodyValues` | if `isRetryable` |
| `RetryError` | retries exhausted (see above) | - |
| `InvalidResponseDataError` | answer validation failed (see [Rounding](#rounding-and-response-validation)) | no |
| `APICallError` "Invalid JSON response" (`cause`: `TypeValidationError`) | a 2xx body that does not match the provider's Zod response schema (executed below) | no (`isRetryable: false`) |

Use the marker-based `isInstance` checks rather than `instanceof`: the AI SDK error reference (https://ai-sdk.dev/docs/reference/ai-sdk-errors) says "The static guard also works when multiple AI SDK package versions are loaded", and the `Experimental_EvaluationUnsupportedQuestionTypeError` reference page calls it the check "which works across package copies".

`APICallError.message` comes from the TypeSafe error body via the provider's `errorToMessage`, in this order: `message`; `error` (string) or `error.message`; `detail` if a string, otherwise `JSON.stringify(detail)`; `error_type`; else `"TypeSafe request failed"`. A FastAPI-style `detail` array therefore appears as a JSON string in the message. Edge case (executed): a body with `"detail": null` yields the message `"null"`, not the `error_type`, because `JSON.stringify(null)` is the string `"null"` and short-circuits the `??` chain.

### Executed error tests

```ts
// Executed offline: ai 7.0.127 + @ai-sdk/typesafe-ai 3.0.12, Node 22.22.0 (v02-errors.mts)
import assert from "node:assert/strict";
import { createTypeSafeAi } from "@ai-sdk/typesafe-ai";
import {
  APICallError,
  Experimental_EvaluationUnsupportedQuestionTypeError,
  InvalidArgumentError,
  InvalidResponseDataError,
  LoadAPIKeyError,
  NoSuchModelError,
  RetryError,
  experimental_evaluate,
} from "ai";

const json = (body: unknown, status = 200, headers: Record<string, string> = {}) =>
  new Response(JSON.stringify(body), { status, headers: { "content-type": "application/json", ...headers } });

const refund = {
  refund: { type: "boolean", instructions: "Is the customer asking for a refund?" },
} as const;

function provider(respond: (n: number) => Response) {
  let n = 0;
  const p = createTypeSafeAi({ apiKey: "k", fetch: async () => respond(++n) });
  return { model: p.evaluationModel("jev-latest"), calls: () => n };
}

// 1. 422: not retryable -> APICallError on the first attempt, message from `detail`.
{
  const { model, calls } = provider(() =>
    json({ detail: "questions.refund.instructions: field required", error_type: "invalid_request" }, 422),
  );
  const err = await experimental_evaluate({ model, state: "x", questions: refund }).catch((e: unknown) => e);
  assert.ok(APICallError.isInstance(err));
  assert.equal(err.statusCode, 422);
  assert.equal(err.isRetryable, false);
  assert.equal(err.message, "questions.refund.instructions: field required");
  assert.equal(calls(), 1);
}

// 2. 429 on every attempt: core retries (maxRetries 2), then throws RetryError, not APICallError.
{
  const { model, calls } = provider(() => json({ message: "rate limited" }, 429, { "retry-after-ms": "0" }));
  const err = await experimental_evaluate({ model, state: "x", questions: refund }).catch((e: unknown) => e);
  assert.ok(RetryError.isInstance(err));
  assert.equal(err.reason, "maxRetriesExceeded");
  assert.equal(err.errors.length, 3);
  assert.ok(APICallError.isInstance(err.lastError) && err.lastError.statusCode === 429);
  assert.equal(calls(), 3);
}

// 3. maxRetries: 0 -> the APICallError itself.
{
  const { model, calls } = provider(() => json({ message: "overloaded" }, 529));
  const err = await experimental_evaluate({ model, state: "x", questions: refund, maxRetries: 0 }).catch((e: unknown) => e);
  assert.ok(APICallError.isInstance(err) && err.isRetryable && err.statusCode === 529);
  assert.equal(calls(), 1);
}

// 4. Rounding: a 2-dp distribution summing to 0.99 passes; one summing to 0.95 is rejected
//    after the call (InvalidResponseDataError is not retried).
const choiceQ = {
  route: {
    type: "choice",
    instructions: "Route this ticket.",
    criteria: { billing: null, shipping: null, technical: null },
  },
} as const;
const choiceAnswer = (p: Record<string, number>) =>
  json({ model: "jev-1.13.0", answers: { route: { type: "choice", choice: "billing", probabilities: p, confidence: 0.5 } } });
{
  const { model } = provider(() => choiceAnswer({ billing: 0.5, shipping: 0.25, technical: 0.24 }));
  const ok = await experimental_evaluate({ model, state: "x", questions: choiceQ });
  assert.equal(ok.answers.route.choice, "billing");
}
{
  const { model, calls } = provider(() => choiceAnswer({ billing: 0.5, shipping: 0.25, technical: 0.2 }));
  const err = await experimental_evaluate({ model, state: "x", questions: choiceQ }).catch((e: unknown) => e);
  assert.ok(InvalidResponseDataError.isInstance(err));
  assert.equal(err.message, 'Question "route" probabilities must sum to 1 within the declared rounding precision.');
  assert.equal(calls(), 1);
}

// 5. Input validation (core) and TypeSafe limits (provider) before any HTTP call.
{
  const { model, calls } = provider(() => json({}));
  const tooFew = await experimental_evaluate({
    model,
    state: "x",
    questions: { s: { type: "score", instructions: "?", criteria: ["only one"] } },
  }).catch((e: unknown) => e);
  assert.ok(InvalidArgumentError.isInstance(tooFew));
  const eleven = Array.from({ length: 11 }, (_, i) => `level ${i}`);
  const tooMany = await experimental_evaluate({
    model,
    state: "x",
    questions: { s: { type: "score", instructions: "?", criteria: eleven } },
  }).catch((e: unknown) => e);
  assert.ok(InvalidArgumentError.isInstance(tooMany));
  assert.equal(tooMany.message, "TypeSafe Score questions support at most 10 levels.");
  assert.equal(calls(), 0);
}

// 6. Missing key: createTypeSafeAi() does not throw; the call does.
{
  delete process.env.TYPESAFE_AI_API_KEY;
  const p = createTypeSafeAi({ fetch: async () => json({}) });
  const err = await experimental_evaluate({ model: p.evaluationModel("jev-latest"), state: "x", questions: refund }).catch(
    (e: unknown) => e,
  );
  assert.ok(LoadAPIKeyError.isInstance(err));
}

// 7. Unsupported question type, checked before I/O; non-evaluation factories throw.
{
  const { model } = provider(() => json({}));
  const err = await experimental_evaluate({
    model: { ...model, supportedQuestionTypes: ["choice"], doEvaluate: model.doEvaluate.bind(model) },
    state: "x",
    questions: refund,
  }).catch((e: unknown) => e);
  assert.ok(Experimental_EvaluationUnsupportedQuestionTypeError.isInstance(err));
  assert.equal(err.questionId, "refund");
  assert.throws(() => createTypeSafeAi({ apiKey: "k" }).languageModel("jev-latest"), (e) => NoSuchModelError.isInstance(e));
}

// 8. Unknown providerOptions.typesafe keys become warnings, not errors.
{
  const { model } = provider(() =>
    json({ model: "jev-1.13.0", answers: { refund: { type: "noul", noul: 0.9 } }, usage: null }),
  );
  const r = await experimental_evaluate({
    model,
    state: "x",
    questions: refund,
    providerOptions: { typesafe: { temperature: 0 } },
  });
  assert.deepEqual(r.warnings, [{ type: "unsupported", feature: "providerOptions.typesafe.temperature" }]);
  assert.deepEqual(r.usage, { inputTokens: undefined, outputTokens: undefined, totalTokens: undefined });
}

// 9. abortSignal: AbortSignal.timeout() rejects with the signal's reason, no retry.
//    (AbortSignal.timeout uses an unref'd timer; keep the loop alive in a bare script.)
{
  const keepAlive = setTimeout(() => {}, 1000);
  let n = 0;
  const p = createTypeSafeAi({
    apiKey: "k",
    fetch: (_url, init) =>
      new Promise((_res, rej) => {
        n++;
        init?.signal?.addEventListener("abort", () => rej(init.signal?.reason), { once: true });
      }),
  });
  const err = await experimental_evaluate({
    model: p.evaluationModel("jev-latest"),
    state: "x",
    questions: refund,
    abortSignal: AbortSignal.timeout(50),
  }).catch((e: unknown) => e);
  assert.ok(err instanceof Error);
  console.log("abort error:", err.name, "-", err.message);
  assert.equal(n, 1);
  clearTimeout(keepAlive);
}

console.log("ok");
```

Output (stderr warnings included):

```text
(node:6452) Warning: AI SDK Warning System: To turn off warning logging, set the AI_SDK_LOG_WARNINGS global to false.
(node:6452) Warning: AI SDK Warning (typesafe.evaluation / jev-latest): The feature "providerOptions.typesafe.temperature" is not supported.
abort error: TimeoutError - The operation was aborted due to timeout
ok
```

The 422 and 429 bodies in this test are invented shapes chosen to exercise each branch; the live API's error bodies were not observed. Case 8 also shows that `usage: null` in a response (allowed by the provider's schema) yields `undefined` token counts rather than an error.

Connection failures and schema mismatches, executed the same way:

```ts
// Executed offline: ai 7.0.127 + @ai-sdk/typesafe-ai 3.0.12, Node 22.22.0 (v08-network.mts)
import assert from "node:assert/strict";
import { createTypeSafeAi } from "@ai-sdk/typesafe-ai";
import { APICallError, RetryError, experimental_evaluate } from "ai";

const questions = { r: { type: "boolean", instructions: "Is this a refund request?" } } as const;

// A network failure as undici reports it: TypeError("fetch failed") with a `cause`.
let n = 0;
const down = createTypeSafeAi({
  apiKey: "k",
  fetch: async () => {
    n++;
    throw new TypeError("fetch failed", { cause: new Error("connect ECONNREFUSED 127.0.0.1:443") });
  },
});
const e1 = await experimental_evaluate({ model: down.evaluationModel("jev-latest"), state: "x", questions, maxRetries: 1 })
  .catch((e: unknown) => e);
assert.ok(RetryError.isInstance(e1));
assert.ok(APICallError.isInstance(e1.lastError) && e1.lastError.isRetryable);
assert.equal(e1.lastError.message, "Cannot connect to API: connect ECONNREFUSED 127.0.0.1:443");
assert.equal(n, 2);

// The same TypeError without a `cause` is passed through untouched and not retried.
n = 0;
const bare = createTypeSafeAi({ apiKey: "k", fetch: async () => { n++; throw new TypeError("fetch failed"); } });
const e2 = await experimental_evaluate({ model: bare.evaluationModel("jev-latest"), state: "x", questions, maxRetries: 1 })
  .catch((e: unknown) => e);
assert.ok(e2 instanceof TypeError && !APICallError.isInstance(e2));
assert.equal(n, 1);

// A 200 body that does not match the provider's response schema -> APICallError (not retryable).
const odd = createTypeSafeAi({
  apiKey: "k",
  fetch: async () => new Response('{"answers": 5}', { headers: { "content-type": "application/json" } }),
});
const e3 = await experimental_evaluate({ model: odd.evaluationModel("jev-latest"), state: "x", questions })
  .catch((e: unknown) => e);
assert.ok(APICallError.isInstance(e3));
console.log(e3.message.split("\n")[0], "| retryable:", e3.isRetryable, "| cause:", (e3.cause as Error)?.name);
console.log("ok");
```

Output: `Invalid JSON response | retryable: false | cause: AI_TypeValidationError` then `ok`.

---

## Cancellation and Timeouts

- `experimental_evaluate` has **no timeout option** and the provider sets none, so a call is bounded only by `fetch` itself (Node's `fetch` has no overall deadline by default) and your `abortSignal`. Always pass one on request paths: `abortSignal: AbortSignal.timeout(ms)` or `AbortSignal.any([req.signal, AbortSignal.timeout(ms)])`.
- The signal covers the whole operation including retry delays. Abort errors are never retried (`isAbortError` short-circuits the retry loop).
- As executed above, an expired `AbortSignal.timeout()` rejects with the signal's reason - a `DOMException` named `TimeoutError` - **unwrapped**, not an `APICallError`. Check `err.name === 'TimeoutError' || err.name === 'AbortError'`.
- `AbortSignal.timeout` timers are unref'd in Node: in a bare script with nothing else keeping the event loop alive, the process can exit before the abort fires (seen while writing the test above). In servers this does not arise.
- Worst case without a signal and with retryable failures: three attempts of unbounded duration plus 2 s + 4 s of backoff (or server-specified delays under 60 s).

---

## Warnings and `providerOptions`

"No provider-specific options are currently defined. Unknown entries under `providerOptions.typesafe` produce unsupported-option warnings." (provider page). Executed (case 8 above): `providerOptions: { typesafe: { temperature: 0 } }` produced `warnings: [{ type: 'unsupported', feature: 'providerOptions.typesafe.temperature' }]` and the call succeeded. Core also logs warnings through Node's warning system unless `globalThis.AI_SDK_LOG_WARNINGS = false`.

`providerOptions.gateway` is ignored by the direct TypeSafe provider (it only inspects the `typesafe` key) and is meaningful only for Gateway models.

---

## Testing with `Experimental_EvaluationMockModelV4`

"For tests, use `Experimental_EvaluationMockModelV4` from `ai/test`." (evaluation guide). Its constructor takes optional `provider`, `modelId`, `supportedQuestionTypes` and `doEvaluate`. Core validation still runs against what your mock returns, so fixtures must be valid distributions. Executed:

```ts
// Executed offline: ai 7.0.127 + @ai-sdk/typesafe-ai 3.0.12, Node 22.22.0 (v03-mock-registry.mts)
import assert from "node:assert/strict";
import { createTypeSafeAi } from "@ai-sdk/typesafe-ai";
import {
  createProviderRegistry,
  customProvider,
  experimental_evaluate,
  type Experimental_EvaluationModel,
} from "ai";
import { Experimental_EvaluationMockModelV4 } from "ai/test";

// Application code under test: route on P(true) and on the TypeSafe confidence statistic.
const questions = {
  department: {
    type: "choice",
    instructions: "Which team should handle this?",
    criteria: { billing: "Charges and refunds", support: "Other requests" },
  },
  requestsRefund: { type: "boolean", instructions: "Is the customer requesting money back?" },
} as const;

export async function route(model: Experimental_EvaluationModel, message: string) {
  const r = await experimental_evaluate({ model, state: { message }, questions, maxRetries: 0 });
  const confidence = r.providerMetadata?.typesafe?.confidence as Record<string, number> | undefined;
  const deptConfidence = confidence?.department;
  if (deptConfidence === undefined || deptConfidence < 0.6) return "human-review" as const;
  if (r.answers.requestsRefund.probability >= 0.8) return "refunds-queue" as const;
  return r.answers.department.choice;
}

// Unit test with the official mock: no HTTP, full core validation still runs.
function mockModel(pRefund: number, confidence: number) {
  return new Experimental_EvaluationMockModelV4({
    provider: "typesafe.evaluation",
    modelId: "jev-latest",
    supportedQuestionTypes: ["choice", "score", "boolean"],
    doEvaluate: async () => ({
      answers: {
        department: { type: "choice", choice: "billing", probabilities: { billing: 0.9, support: 0.1 } },
        requestsRefund: { type: "boolean", probability: pRefund },
      },
      rounding: { probabilityDecimals: 2, scoreDecimals: 2 },
      warnings: [],
      providerMetadata: { typesafe: { confidence: { department: confidence } } },
    }),
  });
}
assert.equal(await route(mockModel(0.97, 0.9), "refund please"), "refunds-queue");
assert.equal(await route(mockModel(0.1, 0.9), "invoice question"), "billing");
assert.equal(await route(mockModel(0.97, 0.4), "???"), "human-review");

// Aliases through customProvider + registry, resolved without network.
const typesafe = createTypeSafeAi({ apiKey: "k" });
const registry = createProviderRegistry({
  triage: customProvider({
    evaluationModels: { native: typesafe.evaluationModel("jev-1.13.0") },
    fallbackProvider: typesafe,
  }),
});
const native = registry.evaluationModel("triage:native");
assert.equal(native.modelId, "jev-1.13.0");
assert.equal(native.provider, "typesafe.evaluation");
assert.deepEqual(native.supportedQuestionTypes, ["choice", "score", "boolean"]);
assert.equal(registry.evaluationModel("triage:jev-latest").modelId, "jev-latest"); // via fallbackProvider

// Lifecycle callback: observe usage and the resolved model per call.
const ends: string[] = [];
await experimental_evaluate({
  model: mockModel(0.5, 0.7),
  state: "x",
  questions,
  onEnd: (event) => {
    ends.push(`${event.response.modelId} ${event.usage.totalTokens}`);
  },
});
assert.deepEqual(ends, ["jev-latest undefined"]);
console.log("ok");
```

Two layers of test are worth keeping separate: the mock model tests **your decision logic** (thresholds, fallbacks) with core validation in the loop; the fake-`fetch` test in the end-to-end section tests **the wire contract** with the real provider. Neither needs an API key.

---

## Registries, Aliases and Default-Provider Strings

Verbatim from the evaluation guide (it imports `@ai-sdk/openai`, which was not installed here, so this block was not type-checked; the TypeSafe-only part is exercised in the executed mock test above):

```ts
import { typeSafeAi } from '@ai-sdk/typesafe-ai';
import { openai } from '@ai-sdk/openai';
import {
  customProvider,
  createProviderRegistry,
  experimental_evaluate,
} from 'ai';

const registry = createProviderRegistry({
  triage: customProvider({
    evaluationModels: {
      native: typeSafeAi.evaluationModel('jev-latest'),
      compact: openai.evaluationModel('gpt-6-luna'),
    },
    fallbackProvider: typeSafeAi,
  }),
  openai,
});

const result = await experimental_evaluate({
  model: registry.evaluationModel('triage:native'),
  state: 'I was charged twice.',
  questions: {
    department: {
      type: 'choice',
      instructions: 'Which team should handle this?',
      criteria: { billing: 'Charges and refunds', support: 'Other requests' },
    },
  },
});

result.answers.department.choice; // 'billing' | 'support'
```

Rules from the guide and `resolveEvaluationModel` (`ai` 7.0.127):

- Registry IDs are `providerId:modelId`; only the first separator is used, so model IDs may contain it. `separator` changes it.
- Custom aliases win over the `fallbackProvider`; a fallback "only resolves unknown model IDs; it does not retry failed evaluations or substitute a model when a question type is unsupported".
- A **string** `model` resolves through `globalThis.AI_SDK_DEFAULT_PROVIDER` if set, otherwise AI Gateway (`resolveEvaluationModel` in `ai` 7.0.127: `globalThis.AI_SDK_DEFAULT_PROVIDER ?? gateway`; the evaluation guide agrees). Note that the `experimental_evaluate` reference page (fetched 2026-10-03) contradicts this with "Evaluation never implicitly falls back to Gateway" - the shipped code does fall back. If the default provider has no `evaluationModel` method, the call throws `NoSuchModelError` ("The default provider does not support evaluation models..."). With `globalThis.AI_SDK_DEFAULT_PROVIDER = typeSafeAi`, `model: 'jev-latest'` works directly. "Global configuration affects other AI SDK functions too; avoid changing it per request in a shared process."
- Registry language/image middleware does not wrap evaluation models. Keep the inferred registry type (or `Experimental_EvaluationProviderRegistry`) to retain the experimental `evaluationModel` method.

---

## Lifecycle Callbacks and Telemetry

- `onStart(event)` fires before the model call with `callId`, `operationId: 'ai.evaluate'`, `provider`, `modelId`, `state`, `questions`, `maxRetries`, `headers`, `providerOptions`, `runtimeContext`.
- `onEnd(event)` fires after successful validation with all of the above plus `answers`, `usage`, `warnings`, `rounding`, `providerMetadata`, `response`. It does not fire on failure.
- Callback exceptions are swallowed (`notify` catches them), so a logging bug cannot fail an evaluation.
- Telemetry integrations get `experimental_onEvaluateStart/End` and `experimental_onEvaluationModelCallStart/End` (`operationId: 'ai.evaluate.doEvaluate'`; "The logical model call includes any provider retries", per the TSDoc in `ai` 7.0.127 `dist/index.d.ts`).
- OpenTelemetry (from https://ai-sdk.dev/docs/ai-sdk-core/telemetry): two `CLIENT` spans named `evaluate {modelId}` - a root for the operation and a child for model work including retries - with `gen_ai.operation.name: "evaluate"`; token usage on the child span. State, questions and answers are recorded under `ai.evaluation.*` only when the `experimental_evaluation` supplemental option is enabled. No `gen_ai.evaluation.result` event is emitted.

Since `state` usually contains customer text, keep evaluation recording off (the default) unless your telemetry backend is cleared for that data.

---

## AI Gateway

AI Gateway serves Jev under the ID **`typesafe-ai/jev`** and is the default resolver for string model IDs.

### String IDs (Gateway default)

"String IDs use Vercel AI Gateway by default. Configure Gateway authentication with `AI_GATEWAY_API_KEY` or Vercel OIDC" (evaluation guide). Verbatim from https://vercel.com/docs/ai-gateway/modalities/evaluation (page `last_updated: 2026-09-22`), type-checked here:

```typescript
import { experimental_evaluate as evaluate } from 'ai';
import { gateway } from '@ai-sdk/gateway';

export async function GET() {
  const result = await evaluate({
    model: gateway.evaluationModel('typesafe-ai/jev'),
    state: 'The support agent issued a full refund to the customer.',
    questions: {
      refunded: {
        type: 'boolean',
        instructions: 'Was a refund issued?',
      },
    },
  });

  return Response.json(result.answers);
}
```

`model: 'typesafe-ai/jev'` (a plain string) is equivalent when no default provider is configured.

### Conditional evaluation fallbacks

Gateway can rerun a *successful but uncertain* evaluation on another model. "Conditional model entries are available only for evaluation requests. Keep the configuration under `providerOptions.gateway.models`. `experimental_evaluate` does not have a top-level fallback option." Verbatim from the AI Gateway provider docs (https://ai-sdk.dev/providers/ai-sdk-providers/ai-gateway, "Conditional Evaluation Fallbacks Example"), type-checked here:

```ts
import type { GatewayProviderOptions } from '@ai-sdk/gateway';
import { experimental_evaluate } from 'ai';

const questions = {
  department: {
    type: 'choice',
    instructions: 'Which team should handle this?',
    criteria: { billing: 'Charges and refunds', support: 'Other requests' },
  },
  requestsRefund: {
    type: 'boolean',
    instructions: 'Is the customer requesting money back?',
  },
} as const;

const result = await experimental_evaluate({
  model: 'typesafe-ai/jev',
  state: 'I was charged twice. Please refund the duplicate.',
  questions,
  providerOptions: {
    gateway: {
      models: [
        {
          model: 'openai/gpt-5.6-sol',
          when: {
            any: [
              { question: 'department', confidenceBelow: 0.6 },
              {
                question: 'requestsRefund',
                probabilityBetween: [0.4, 0.6],
              },
            ],
          },
        },
      ],
    } satisfies GatewayProviderOptions,
  },
});

// The model that produced the answers, the fallback when the condition matched.
console.log(result.response.modelId);
console.log(JSON.stringify(result.providerMetadata?.gateway?.routing, null, 2));
```

Condition grammar (`EvaluationFallbackCondition` in `@ai-sdk/gateway` 4.0.103): exactly one of `{ question?, confidenceBelow }`, `{ question?, probabilityBetween: [lo, hi] }`, `{ any: [...] }`, `{ all: [...] }`, `{ atLeast: { count, conditions: [...] } }`. "Without `question`, `confidenceBelow` checks every Choice and Score question and `probabilityBetween` every Boolean question. Groups nest at most five levels deep." Only the **first** entry of `models` may be conditional; the rest are plain fallback IDs. `GatewayProviderOptions<'department' | 'requestsRefund'>` narrows `question` to your IDs at compile time:

```ts
// Type-checked: tsc 7.0.2 --strict, @ai-sdk/gateway 4.0.103 (excerpt of v05-gateway-instance.mts)
import type { GatewayProviderOptions } from '@ai-sdk/gateway';
const opts = {
  models: [{ model: 'openai/gpt-5.6-sol', when: { question: 'department', confidenceBelow: 0.6 } }],
} satisfies GatewayProviderOptions<'department' | 'requestsRefund'>;
const bad = {
  // @ts-expect-error -- 'deparment' is not a declared question ID
  models: [{ model: 'openai/gpt-5.6-sol', when: { question: 'deparment', confidenceBelow: 0.6 } }],
} satisfies GatewayProviderOptions<'department' | 'requestsRefund'>;
```

When a fallback ran: `result.response.modelId` names the fallback model, `providerMetadata.gateway.routing.modelAttempts` lists the attempts (each successful stage with its own `generationId`, `usage` and cost; the first fallback attempt lists `triggeredBy`), and "Triggered requests bill both stages and report their combined usage" (Vercel TypeSafe-compatible API page). On the TypeSafe-compatible API, if an LLM answered a Choice/Score, "`confidence: 0` and `probabilities: {}` mean those values are unavailable". How the same situation is represented through `experimental_evaluate` was **not verified**: core rejects an empty `probabilities` object as an incomplete distribution, and the AI SDK answer type allows `probabilities` to be absent, so code reading Gateway results should handle a missing distribution and a missing `typesafe` confidence entry.

### Other Gateway facts

| Topic | Fact (source, date) |
|---|---|
| Minimum AI SDK | 7.0.105 for `evaluate` (Vercel changelog, 2026-09-16) |
| HTTP API without the AI SDK | `POST https://ai-gateway.vercel.sh/v1/evaluate` with `model`, `state`, `questions` (AI SDK field names, `boolean`/`probability`) (Vercel evaluation page, 2026-09-22) |
| TypeSafe-compatible API | `https://ai-gateway.vercel.sh/typesafe` + `@typesafe-ai/sdk` (`noul` field names); see `typesafe-jev/javascript-sdk.md` |
| Not available via | the OpenAI-, Anthropic- and Cohere-compatible endpoints (Vercel evaluation page) |
| Per-request privacy | `providerOptions.gateway.zeroDataRetention: true`, `disallowPromptTraining` (Vercel changelog example uses `zeroDataRetention`) |
| Billing | through Gateway, or direct to TypeSafe with a TypeSafe key under BYOK (Vercel TypeSafe-compatible API page) |
| Cost metadata | example response for a 275-input-token call shows `"cost": "0.00001155"` (USD) - consistent with TypeSafe's list price of $0.042 per million input tokens (https://docs.typesafe.ai/models, 2026-10-03): 275 x 0.042 / 10^6 = 0.00001155 |
| Other evaluation models on Gateway | `GatewayEvaluationModelId` also names `liquid/d1` (`@ai-sdk/gateway` 4.0.103 types) |

The Vercel changelog also quotes "TypeSafe reports Jev was up to 193.6x faster and 444.6x cheaper than LLMs on its workflow evaluations" - a vendor claim, reproduced here only with that attribution.

---

## Workflow Serialization

"Evaluation models support workflow serialization. A custom fetch function is not serialized; restored models use the default fetch implementation." (provider page)

Observed by executing the model's `WORKFLOW_SERIALIZE` static (provider 3.0.12): the serialized form is `{ modelId, config: { provider, baseURL, headers } }`, where `headers` is the **resolved** header map - including `authorization: Bearer <apiKey>` in plain text:

```ts
// Executed offline: @ai-sdk/typesafe-ai 3.0.12, @ai-sdk/provider-utils 5.0.53 (v07-serialize.mts)
import { createTypeSafeAi } from "@ai-sdk/typesafe-ai";
import { WORKFLOW_SERIALIZE } from "@ai-sdk/provider-utils";
const model = createTypeSafeAi({ apiKey: "sk-secret-123", fetch: globalThis.fetch }).evaluationModel("jev-latest");
const ctor = model.constructor as unknown as Record<symbol, (m: unknown) => unknown>;
console.log(JSON.stringify(ctor[WORKFLOW_SERIALIZE](model)));
// {"modelId":"jev-latest","config":{"provider":"typesafe.evaluation","baseURL":"https://api.typesafe.ai/v1","headers":{"authorization":"Bearer sk-secret-123","user-agent":"ai-sdk-typesafe-ai/3.0.12"}}}
```

If you persist workflow state, treat it as containing the TypeSafe credential. When `headers` is absent from a restored config, the model falls back to reading `TYPESAFE_AI_API_KEY` at call time (code path in `doEvaluate`).

---

## AI SDK Provider vs `@typesafe-ai/sdk`

| Aspect | `@ai-sdk/typesafe-ai` 3.0.12 via `experimental_evaluate` | `@typesafe-ai/sdk` 0.6.0 |
|---|---|---|
| Env var | `TYPESAFE_AI_API_KEY` | `TYPESAFE_API_KEY` |
| Base URL default | `https://api.typesafe.ai/v1` | `https://api.typesafe.ai` |
| Missing key | rejects at call time (`LoadAPIKeyError`) | throws in the constructor |
| Yes/no type | `boolean` -> `probability` | `noul` -> `noul` |
| Confidence | `providerMetadata.typesafe.confidence[id]` | `answer.confidence` |
| Score legend | dropped | `answer.legend` |
| Score key typing | `Record<string, number>` | `"0" \| "1" \| ...` |
| `instructions` | required (string/object/array) | optional, nullable |
| 255 options / 10 levels | checked locally | sent to the server |
| Response validation | full (keys, sums, means, ranges) | none |
| Retries | core: 2, backoff 2 s/4 s, no jitter, 408/409/429/5xx, `RetryError` wrapper | 2, backoff 500 ms/1 s with 25 % jitter, 408/429/5xx, last error rethrown |
| Timeout | none built in - pass `abortSignal` | 10 s per attempt |
| Model ID returned | `result.response.modelId` | `result.model` |
| Request ID | `result.response.headers['x-typesafe-request-id']` | `withResponse().requestId` / `APIError.requestId` |
| Extras | registries, Gateway fallbacks, OTel spans, mock model | browser guard, log redaction, `models.list()` |
| Stability | experimental, "may change in patch releases" | 0.x semver |

Choose the AI SDK path when your app already uses AI SDK 7, you want Gateway fallbacks or a provider-neutral interface, or you value response validation. Choose `@typesafe-ai/sdk` for the full TypeSafe answer (legend, confidence on the answer), a built-in per-attempt timeout, and a non-experimental surface.

---

## What Was Not Verified

- No call reached TypeSafe or AI Gateway (no keys); every runtime claim comes from executing the shipped code against fake `fetch` implementations or from reading it.
- Connection failures were simulated with a thrown `TypeError`; real undici/DNS/TLS failures were not produced.
- How Gateway shapes LLM-fallback answers through `experimental_evaluate` (missing vs empty `probabilities`) was inferred from types and Vercel's prose, not observed.
- The registry example importing `@ai-sdk/openai` was not type-checked (package not installed).
- Behaviour on edge runtimes was not exercised; the provider declares `engines.node >= 22`.
