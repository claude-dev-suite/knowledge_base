# TypeSafe JavaScript / TypeScript SDK (`@typesafe-ai/sdk`) - Complete Reference

> Official Documentation: https://docs.typesafe.ai/sdk/javascript
> API Reference (generated from the SDK's TSDoc): https://docs.typesafe.ai/sdk/javascript/api
> HTTP API Reference: https://docs.typesafe.ai/api
> SDK Repository: https://github.com/typesafe-ai/typesafe-sdk-js (source pinned by the docs at tag `v0.6.0`)
> Last verified: 2026-10-03
> Verified against: `@typesafe-ai/sdk` **0.6.0** (published 2026-09-15 per the SDK changelog), its shipped `dist/index.d.mts` and `dist/index.mjs`, Node.js 22.22.0, TypeScript 7.0.2 (`strict`). Model facts from https://docs.typesafe.ai/models (current model `jev-1.13.0`).

## Overview

`@typesafe-ai/sdk` is the official client for TypeSafe's System One endpoint (`POST /v1/systemone`), which evaluates one `state` against a map of typed questions (`noul` yes/no, `choice`, `score`) with the Jev model and returns calibrated probabilities. The SDK is small (one client class, one resource, three question helpers, twelve error classes) and its value is in three places:

1. **Type inference** - answers are typed from the question map you pass: a `choice` answer's `.choice` is the literal union of your criteria keys, a `score` answer's `.probabilities` is keyed by the rubric indices as strings.
2. **Transport policy** - per-attempt timeout, capped exponential backoff with jitter, `Retry-After` handling, typed error classes, cancellation via `AbortSignal`.
3. **Safety rails** - refuses to run in a browser unless you opt in, redacts credentials in logs, keeps the API key in a private field.

This document goes below the quick reference: every exported symbol, the exact runtime behaviour read from the shipped JavaScript (not just the types), and offline-executed examples showing what goes over the wire. Everything stated as "behaviour" below was either read from `dist/index.mjs` of 0.6.0 or executed against it; where the docs and the code differ, both are given.

For the request model, question design and calibration, see the sibling documents in this knowledge base. For the Vercel AI SDK provider (`@ai-sdk/typesafe-ai`) see `typesafe-jev/vercel-ai-sdk.md` - it is a **different** client with a different env var, answer shape and retry policy.

---

## Table of Contents

1. [Installation and Package Shape](#installation-and-package-shape)
2. [Exports at a Glance](#exports-at-a-glance)
3. [Creating the Client (`TypeSafeClientConfig`)](#creating-the-client-typesafeclientconfig)
4. [Environment Variables and `ENV`](#environment-variables-and-env)
5. [`systemOne`: Request and Options](#systemone-request-and-options)
6. [Question Helpers and Type Inference](#question-helpers-and-type-inference)
7. [Response and Answer Interfaces](#response-and-answer-interfaces)
8. [`APIPromise`: `withResponse`, `asResponse`, `map`](#apipromise-withresponse-asresponse-map)
9. [Timeouts and Cancellation with `AbortSignal`](#timeouts-and-cancellation-with-abortsignal)
10. [Retries (`RetryPolicy`)](#retries-retrypolicy)
11. [Error Classes](#error-classes)
12. [Logging](#logging)
13. [Models Resource](#models-resource)
14. [HTTP Headers the SDK Sends](#http-headers-the-sdk-sends)
15. [Testing with a Custom `fetch` (executed)](#testing-with-a-custom-fetch-executed)
16. [Server-Side Only: `dangerouslyAllowBrowser`](#server-side-only-dangerouslyallowbrowser)
17. [Gateways and Extension Properties](#gateways-and-extension-properties)
18. [`VERSION`, `LOG_LEVELS` and Other Constants](#version-log_levels-and-other-constants)
19. [Gotchas Checklist](#gotchas-checklist)
20. [What Was Not Verified](#what-was-not-verified)

---

## Installation and Package Shape

```sh
npm install @typesafe-ai/sdk
```

(Verbatim from https://docs.typesafe.ai/sdk/javascript, which states "Node.js 20 or newer".)

Facts from the installed `package.json` of 0.6.0:

| Field | Value |
|---|---|
| `version` | `0.6.0` |
| `license` | MIT |
| `type` | `module` |
| `engines.node` | `>=20` |
| `exports["."]` | `import` -> `dist/index.mjs` (+ `index.d.mts`), `require` -> `dist/index.cjs` (+ `index.d.cts`) |
| `sideEffects` | `false` |
| runtime `dependencies` | none |

So the package works from both ESM (`import`) and CommonJS (`require`), ships its own declarations, and has no transitive runtime dependencies - it uses the global `fetch`, `Response`, `Headers`, `AbortController` and `setTimeout`.

The package `scripts` mention a JSR publish (`push:jsr`), so a JSR build likely exists as well; this document was verified only against the npm tarball.

### Release history (from https://docs.typesafe.ai/sdk/javascript/changelog, fetched 2026-10-03)

| Version | Date | Notes |
|---|---|---|
| 0.5.7 | 2026-09-11 | Initial public release |
| 0.6.0 | 2026-09-15 | **Breaking:** `score` criteria are an ordered sequence (array) instead of a dictionary keyed by integers |

If you find code that passes `{ 0: "...", 1: "..." }` as score criteria, it was written for 0.5.x. In 0.6.0 the `score()` helper throws `TypeSafeError("Score criteria must be a list of descriptions indexed by score from zero, not a map.")`, and a hand-built `{ type: "score", criteria: {...} }` is rejected by `systemOne`'s client-side validation (see [systemOne](#systemone-request-and-options)).

---

## Exports at a Glance

The complete export list of `dist/index.d.mts` (0.6.0):

| Kind | Names |
|---|---|
| Client | `TypeSafeClient` |
| Promise wrapper | `APIPromise` |
| Question helpers | `noul`, `choice`, `score` |
| Errors | `TypeSafeError`, `APIError`, `BadRequestError`, `AuthenticationError`, `PermissionDeniedError`, `NotFoundError`, `UnprocessableEntityError`, `RateLimitError`, `InternalServerError`, `APIConnectionError`, `APITimeoutError`, `APIUserAbortError` |
| Constants | `ENV`, `LOG_LEVELS`, `VERSION` |
| Types (type-only) | `TypeSafeClientConfig`, `RequestOptions`, `RetryPolicy`, `Fetch`, `Logger`, `LogLevel`, `EnvVar`, `SystemOneRequest`, `SystemOneRequestPayload`, `SystemOneResult`, `Questions`, `Question`, `NoulQuestion`, `ChoiceQuestion`, `ScoreQuestion`, `ChoiceCriteria`, `ScoreCriteria`, `Description`, `EntryType`, `JsonValue`, `NoulResponse`, `ChoiceResponse`, `ScoreResponse`, `ScoreLegend`, `ScoreOf`, `ResultFor`, `Usage`, `ModelCard`, `Models`, `WithResponse` |

`Models` is exported as a type only - you reach the instance through `client.models`. There is no exported constructor for it.

---

## Creating the Client (`TypeSafeClientConfig`)

```ts
new TypeSafeClient(config?: TypeSafeClientConfig)
```

Precedence: **explicit value -> environment variable -> SDK default** for the four options that have an environment variable (`apiKey`, `baseURL`, `defaultModel`, `logLevel`); every other option is explicit value -> SDK default. Empty or whitespace-only environment values are ignored (the SDK reads `process.env[name]?.trim() || undefined`). The explicit value is taken with `??`, so an explicit empty string (e.g. `apiKey: ""`) is used as-is and does **not** fall back to the environment (executed against 0.6.0: the request went out with `Authorization: Bearer ` and no key, even with `TYPESAFE_API_KEY` set).

| Option | Type | Default | Notes |
|---|---|---|---|
| `apiKey` | `string` | `TYPESAFE_API_KEY` | **Required** - the constructor throws `TypeSafeError` if neither is set. Stored in a `#private` field: not a public property, not in `JSON.stringify(client)`. |
| `baseURL` | `string` | `TYPESAFE_BASE_URL`, then `https://api.typesafe.ai` | Trailing slashes are stripped. Paths `/v1/systemone` and `/v1/models` are appended, so the base must **not** include `/v1`. |
| `defaultModel` | `string` | `TYPESAFE_DEFAULT_MODEL`, then `jev-latest` | Used when a request omits `model`. |
| `logLevel` | `"debug" \| "info" \| "warn" \| "error" \| "off"` | `TYPESAFE_LOG_LEVEL`, then `warn` | An unknown value (from code or env) throws `TypeSafeError` at construction. |
| `logger` | `Logger` | `console` with a `[typesafe-sdk]` prefix | Filtered to `logLevel` and above. |
| `retry` | `Partial<RetryPolicy>` | see [Retries](#retries-retrypolicy) | Merged over SDK defaults and validated. |
| `timeout` | `number` (ms) | `10000` | **Per attempt**, no total budget. Must be finite and `> 0`. |
| `defaultHeaders` | `Record<string, string>` | `{}` | Per-call `headers` override them (case-insensitive merge). |
| `dangerouslyAllowBrowser` | `boolean` | `false` | See [Server-side only](#server-side-only-dangerouslyallowbrowser). |
| `fetch` | `Fetch` = `(input: string, init?: RequestInit) => Promise<Response>` | global `fetch` | For proxies, custom agents, and tests. If omitted and no global `fetch` exists, the constructor throws. |

The constructed client exposes the resolved values as **read-only** properties: `baseURL`, `defaultModel`, `logLevel`, `logger` (already level-filtered), `retry` (a fully resolved `RetryPolicy`), `timeout`, `defaultHeaders`, `fetch`, `models`. The API key is deliberately not among them.

### Constructor validation (executed)

The constructor is eager: a missing key, an invalid log level, a non-positive timeout or an out-of-range retry setting all throw `TypeSafeError` immediately, not on the first request. Validation rules read from `dist/index.mjs`:

| Setting | Rule | Error message prefix |
|---|---|---|
| `timeout` | finite, `> 0` | `` `timeout` must be a positive number of milliseconds `` |
| `retry.maxRetries` | integer, `>= 0` | `` `retry.maxRetries` must be a non-negative integer `` |
| `retry.backoffInitialMs`, `backoffMaxMs`, `maxRetryAfterMs` | finite, `>= 0` | `... must be a non-negative number of milliseconds` |
| `retry.backoffJitter` | finite, `0..1` | `` `retry.backoffJitter` must be between 0 and 1 `` |
| `retry.httpStatuses` | every entry an integer `100..999` | `` `retry.httpStatuses` must contain HTTP status codes `` |
| `logLevel` | one of `LOG_LEVELS` | `Invalid log level "<v>" from <source>. Expected one of: debug, info, warn, error, off.` |

The following was executed offline against 0.6.0 (Node 22.22.0); every assertion passed. It also shows the environment fallbacks.

```ts
// Executed offline: @typesafe-ai/sdk 0.6.0, Node 22.22.0 (excerpt of t04-config-logging.mts)
import assert from "node:assert/strict";
import { TypeSafeClient, TypeSafeError } from "@typesafe-ai/sdk";

const json = (body: unknown, status = 200) =>
  new Response(JSON.stringify(body), { status, headers: { "content-type": "application/json" } });

// 1. Missing key: the constructor throws, it does not wait for the first call.
delete process.env.TYPESAFE_API_KEY;
assert.throws(() => new TypeSafeClient({ fetch: async () => json({}) }), TypeSafeError);

// 2. Environment fallbacks (blank values are ignored) and explicit values winning.
process.env.TYPESAFE_API_KEY = "env-key";
process.env.TYPESAFE_BASE_URL = "https://proxy.example.com/typesafe///";
process.env.TYPESAFE_DEFAULT_MODEL = "   ";
const fromEnv = new TypeSafeClient({ fetch: async () => json({}) });
assert.equal(fromEnv.baseURL, "https://proxy.example.com/typesafe"); // trailing slashes stripped
assert.equal(fromEnv.defaultModel, "jev-latest"); // whitespace-only env value ignored
assert.equal(fromEnv.timeout, 10000);
assert.equal(fromEnv.retry.maxRetries, 2);
assert.deepEqual([...fromEnv.retry.httpStatuses].slice(0, 3), [408, 429, 500]);
assert.equal(fromEnv.retry.httpStatuses.size, 102); // 408, 429, 500..599
assert.equal(JSON.stringify(fromEnv).includes("env-key"), false); // key is a private field
delete process.env.TYPESAFE_BASE_URL;
delete process.env.TYPESAFE_DEFAULT_MODEL;

// 3. Invalid configuration is rejected eagerly.
assert.throws(() => new TypeSafeClient({ apiKey: "k", timeout: 0 }), /`timeout` must be a positive number/);
assert.throws(() => new TypeSafeClient({ apiKey: "k", retry: { backoffJitter: 2 } }), /between 0 and 1/);
assert.throws(
  () => new TypeSafeClient({ apiKey: "k", logLevel: "verbose" as never }),
  /Invalid log level "verbose"/,
);
```

### Lifetime

The client holds no sockets or timers of its own (all I/O goes through `fetch`), has no `close()` method, and is safe to create once per process and share. One internal counter numbers requests for log lines (`#1 POST /v1/systemone`). Create a second client when you need a different key, base URL or default model - for example one for `api.typesafe.ai` and one for a gateway.

---

## Environment Variables and `ENV`

The `ENV` constant maps option names to environment-variable names; it is exported so tooling can reference the names without string literals.

| `ENV` key | Variable | Used when |
|---|---|---|
| `ENV.apiKey` | `TYPESAFE_API_KEY` | `apiKey` omitted |
| `ENV.baseURL` | `TYPESAFE_BASE_URL` | `baseURL` omitted |
| `ENV.defaultModel` | `TYPESAFE_DEFAULT_MODEL` | `defaultModel` omitted |
| `ENV.logLevel` | `TYPESAFE_LOG_LEVEL` | `logLevel` omitted |

`EnvVar` is the union type of those four strings. The SDK reads `process.env` only when `process` exists (`typeof process === "undefined"` is handled), so on runtimes without `process` (e.g. some edge runtimes) you must pass `apiKey` explicitly.

> Not the same as the Vercel AI SDK provider: `@ai-sdk/typesafe-ai` reads **`TYPESAFE_AI_API_KEY`** and defaults its base URL to `https://api.typesafe.ai/v1` (with `/v1`). A project using both clients needs both variables, or explicit keys.

---

## `systemOne`: Request and Options

```ts
systemOne<const Q extends Questions>(
  request: SystemOneRequest<Q>,
  options?: RequestOptions,
): APIPromise<SystemOneResult<Q>>
```

### `SystemOneRequest<Q>`

| Field | Type | Required | Notes |
|---|---|---|---|
| `state` | `EntryType` = `string \| { [key: string]: JsonValue } \| JsonValue[] \| null` | yes | The content to evaluate. The HTTP API docs list `string \| object \| array`; the SDK type also admits `null`. |
| `questions` | `Q extends Questions` (= `{ [name: string]: Question }`) | yes | Must be non-empty (checked client-side). Keys are your names; answers come back under them. Per https://docs.typesafe.ai/api the key "is not sent to the underlying model and is not used in inference". |
| `model` | `string` | no | Omitted -> `client.defaultModel` (`jev-latest` unless configured). |

`SystemOneRequestPayload` is the same shape with `model: string` required - the body the SDK actually posts after filling in the default.

**Additional properties are forwarded.** The TSDoc says "Additional properties on a request variable are forwarded, including `null` values", and the implementation posts `{ ...request, model }`. TypeScript's excess-property check rejects unknown keys in a fresh object literal, but not in a variable; see [Gateways and extension properties](#gateways-and-extension-properties) for the practical use (AI Gateway's `providerOptions`).

### Client-side validation (synchronous!)

Before any I/O, `systemOne` runs `validateQuestions`:

| Check | Error (`TypeSafeError`) |
|---|---|
| `questions` has no keys | `At least one question is required.` |
| a `score` question whose `criteria` is not an array | `Score question "<name>" has criteria that are not a list; ...` |
| a `score` question with fewer than 2 criteria | `Score question "<name>" has <n> criteria; at least two scores are required.` |

These checks, and the per-call `timeout` and `retry` validation (e.g. `{ retry: { maxRetries: -1 } }`, executed against 0.6.0), **throw synchronously** from `systemOne` - `systemOne` is not an `async` function. `await client.systemOne(...)` inside `try/catch` catches them; `client.systemOne(...).catch(...)` does **not**, because no promise exists yet.

Not checked client-side in 0.6.0: the API's documented limits of **255 options per Choice** and **up to 10 Score levels** (https://docs.typesafe.ai/api, fetched 2026-10-03), and question `type` values other than `noul`/`choice`/`score`. Those go to the server. (The Vercel AI SDK provider does check 255/10 locally.) Validate them yourself if you build questions from user input.

### `RequestOptions` (second argument)

| Option | Type | Behaviour |
|---|---|---|
| `signal` | `AbortSignal` | Cancels the in-flight attempt **and** any pending retry wait; rejects with `APIUserAbortError`. |
| `timeout` | `number` (ms) | Overrides `client.timeout` for this call; per attempt. Must be `> 0`. |
| `retry` | `Partial<RetryPolicy>` | Merged over the **client's** resolved policy (not over SDK defaults). |
| `headers` | `Record<string, string>` | Merged over `defaultHeaders`; last value wins regardless of header-name casing. |

The SDK always sets `Authorization`, `Accept`, `User-Agent`, `X-TypeSafe-SDK`, `X-TypeSafe-Runtime`, `Content-Type` and (on retries) `X-TypeSafe-Retry-Count` **after** your headers, so you cannot override those through `defaultHeaders` or `headers` (merge order in `fetchWithRetries`: your headers first, SDK headers second).

### The request is dispatched eagerly

`systemOne` starts the HTTP request as soon as it is called, not when the returned promise is awaited. Executed against 0.6.0: calling `systemOne` without ever awaiting it still performed one `fetch` call, and when the server answered 401 the process received an `unhandledRejection` with an `AuthenticationError` (which terminates Node by default). Always await or attach a handler, even for fire-and-forget calls.

```ts
// Executed offline: @typesafe-ai/sdk 0.6.0, Node 22.22.0 (t08-eager.mts)
// Output: "unhandledRejection: AuthenticationError" then "fetch calls: 1"
import { TypeSafeClient, noul } from "@typesafe-ai/sdk";
let calls = 0;
const client = new TypeSafeClient({
  apiKey: "k",
  logLevel: "off",
  fetch: async () => (calls++, new Response('{"message":"bad key"}', { status: 401 })),
});
process.on("unhandledRejection", (e) => console.log("unhandledRejection:", (e as Error).name));
client.systemOne({ state: "x", questions: { u: noul("?") } }); // never awaited
await new Promise((r) => setTimeout(r, 100));
console.log("fetch calls:", calls);
```

---

## Question Helpers and Type Inference

The three helpers build the question objects and, more importantly, **preserve literal types** through `const` type parameters so the answers can be typed precisely.

| Helper | Signature (0.6.0) | Runtime check |
|---|---|---|
| `noul` | `(instructions?: EntryType, criteria?: { true?: EntryType; false?: EntryType } \| null) => NoulQuestion` | none; `instructions` defaults to `null` |
| `choice` | `<const T extends ChoiceCriteria>(instructions: EntryType, criteria: T) => ChoiceQuestion<T>` | throws `TypeSafeError` if `criteria` is an array |
| `score` | `<const T extends ScoreCriteria>(instructions: EntryType, criteria: T) => ScoreQuestion<T>` | throws `TypeSafeError` if `criteria` is not an array |

Type aliases behind them:

- `EntryType = string | { [key: string]: JsonValue } | JsonValue[] | null` - instructions and criteria descriptions may be text, a JSON object or a JSON array (structured instructions, see https://docs.typesafe.ai/concepts/how-to-build-with-system-one).
- `ChoiceCriteria = { [label: string]: Description }` where `Description = EntryType` (`null` = undescribed label).
- `ScoreCriteria = readonly [EntryType, EntryType, ...EntryType[]]` - **at least two** entries at the type level, indexed from zero, lowest level first. The SDK's TSDoc says Score entries "may be `null`", but the HTTP API reference (`array<string | object | array>`) and the OpenAPI spec do not list `null` for Score levels (they do for Choice descriptions); whether the live API accepts a `null` Score level was not verified.

You do not have to use the helpers: a plain object such as `{ type: "noul", instructions: "..." }` placed directly in `questions` is inferred just as precisely because `systemOne` itself declares `<const Q extends Questions>`. The helpers add the runtime array/map checks and readability.

`noul` accepts no arguments at all (`noul()` -> `{ type: "noul", instructions: null, criteria: undefined }`). The HTTP docs mark `instructions` as required for every question type, while the SDK types and the OpenAPI spec make it nullable; a question with no instructions is legal for the SDK but its usefulness depends on the server - not verified against the live API here.

### How answer types are derived

```ts
type ResultFor<T extends Question> =
  T extends NoulQuestion ? NoulResponse
  : T extends ScoreQuestion<infer S> ? ScoreResponse<S>
  : T extends ChoiceQuestion<infer E> ? ChoiceResponse<E>
  : never;

type ScoreOf<T extends ScoreCriteria> =
  number extends T["length"] ? number : Extract<keyof T, `${number}`>;
```

(From `dist/index.d.mts` of 0.6.0, line breaks added; tokens unchanged.)

So:

- `ChoiceResponse<T>.choice` is `keyof T & string` - the union of your labels; `.probabilities` is `{ readonly [label in keyof T]: number }`.
- `ScoreResponse<T>.probabilities` and `.legend` are keyed by `ScoreOf<T>`: for a fixed-length tuple that is `"0" | "1" | ...`; for a variable-length array it degrades to `number`.
- `ScoreResponse<T>.legend[k]` has type `T[k]`, i.e. the literal description string when the rubric is a literal.

### Inference in practice (type-checked)

```ts
// Type-checked: tsc 7.0.2 --strict against @typesafe-ai/sdk 0.6.0 (t01-inference.mts)
import { choice, noul, score, TypeSafeClient } from "@typesafe-ai/sdk";

const client = new TypeSafeClient({ apiKey: "test-key" });

export async function triage(ticket: string) {
  const res = await client.systemOne({
    state: { ticket },
    questions: {
      team: choice("Which team should pick up `ticket`?", {
        infra: "Deploys, availability, and on-call incidents.",
        billing: "Payments, invoices, and subscriptions.",
        other: null,
      }),
      urgent: noul("Does `ticket` need attention right now?", {
        true: "A customer-facing outage or data loss.",
        false: "Anything that can wait for the next business day.",
      }),
      severity: score("How severe is the impact?", [
        "Cosmetic.",
        "Degraded for some users.",
        "Full outage.",
      ]),
    },
  });

  // Literal label union inferred from the criteria keys.
  const team: "infra" | "billing" | "other" = res.answers.team.choice;
  const pInfra: number = res.answers.team.probabilities.infra;
  // Noul answers carry only `noul` (P(yes)).
  const pUrgent: number = res.answers.urgent.noul;
  // Score keys are the tuple indices as strings.
  const level: "0" | "1" | "2" = "2";
  const pFull: number = res.answers.severity.probabilities[level];
  const legendFull: "Full outage." = res.answers.severity.legend["2"];
  // @ts-expect-error -- "3" is not a level of a three-entry rubric
  res.answers.severity.probabilities["3"];
  // @ts-expect-error -- not a label of this choice question
  res.answers.team.probabilities.sales;
  return { team, pInfra, pUrgent, pFull, legendFull, model: res.model, usage: res.usage };
}

// A rubric typed as a plain array widens the key type to `number`, and is
// rejected at compile time because ScoreCriteria needs at least two entries.
const levels: string[] = ["low", "high"];
// @ts-expect-error -- string[] is not assignable to readonly [EntryType, EntryType, ...EntryType[]]
score("How risky?", levels);
// @ts-expect-error -- a single-level rubric is a compile-time error
score("How risky?", ["only one"]);
```

Practical consequences:

- Build rubrics **inline or `as const`**. A rubric loaded from config as `string[]` does not type-check against `score()`; assert it to a tuple (`as unknown as readonly [string, string, ...string[]]`) after validating its length at runtime, and accept that answer keys become `number`-typed.
- Choice label unions survive into your routing code, so a `switch (answer.choice)` is exhaustively checked.
- The state is **not** typed against the questions: backtick references such as `` `ticket` `` inside instructions are plain text to TypeScript.

---

## Response and Answer Interfaces

```ts
// Type-checked restatement (t09-overrides.mts): assignable both ways to the exported SystemOneResult<Q>
interface SystemOneResult<Q extends Questions> {
  readonly model: string;                                   // resolved model ID, e.g. "jev-1.13.0"
  readonly answers: { readonly [K in keyof Q]: ResultFor<Q[K]> };
  readonly usage: Usage;                                    // { input_tokens, output_tokens }
}
```

(TSDoc comments of `dist/index.d.mts` replaced by this document's field comments.)

| Interface | Fields |
|---|---|
| `NoulResponse` | `type: "noul"`, `noul: number` - probability of yes, 0..1 |
| `ChoiceResponse<T>` | `type: "choice"`, `choice` (highest-probability label), `confidence: number`, `probabilities` keyed by label |
| `ScoreResponse<T>` | `type: "score"`, `score: number` (probability-weighted, may fall between levels), `confidence: number`, `legend` (level -> your description), `probabilities` keyed by level string |
| `Usage` | `input_tokens: number`, `output_tokens: number` (snake_case, exactly as on the wire) |

Example wire response (verbatim from https://docs.typesafe.ai/api):

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

Points that matter in code:

- **No runtime validation of responses.** The SDK returns `JSON.parse` of the body cast to `SystemOneResult<Q>`. If a gateway or mock returns a body without one of your keys, `answers.<key>` is `undefined` at runtime even though the type says it exists (demonstrated in the executed test below). Treat responses from anything other than `api.typesafe.ai` defensively.
- **Field names are snake_case and unconverted** (`input_tokens`, `release_date`). The SDK does not camel-case anything.
- **`noul` has no `confidence`.** The docs define `confidence` only for Choice and Score; for a yes/no question, the distance of `noul` from 0.5 is the signal.
- **`confidence` is not the selected option's probability.** It is "derived from the answer's probability distribution" (https://docs.typesafe.ai/api). https://docs.typesafe.ai/confidence publishes the formulas and calls them "exact": Choice confidence is `(p_max - 1/n) / (1 - 1/n)` (equivalently `(n * p_max - 1) / (n - 1)`) for `n` options, and Score confidence is `max(0, 1 - sum_i p_i * |i - m| / MAD_unif)` with `m` the most likely level and `MAD_unif = (1/n) * sum_i |i - (n-1)/2|`. Example: `p_max = 0.88` over 3 options gives 0.82, while the docs' API example shows 0.81 for that (rounded) distribution - presumably the API computes confidence before rounding (an inference, not documented), so recomputing from the two-decimal `probabilities` can differ in the last digit. Prefer the returned field for thresholds.
- **Values are rounded to two decimals** by the API (stated on the AI SDK provider page, https://ai-sdk.dev/providers/ai-sdk-providers/typesafe-ai, fetched 2026-10-03), so probabilities may not sum to exactly 1 and `0.0` can be a rounded small value. Never test `p === 0`.
- **`model`** is the versioned ID that answered (`jev-1.13.0`) even when you sent the alias `jev-latest` - log it next to every decision so threshold changes can be traced to a model change (https://docs.typesafe.ai/models).

### Price and limits that affect client code

From https://docs.typesafe.ai/models (fetched 2026-10-03), for `jev-1.13.0`:

| Item | Value |
|---|---|
| Price | $42 per billion input tokens ($0.042 per million); output tokens are free |
| Rate limits | 100K tokens/s and 80 requests/s - "adjusting dynamically ... can change without notice" |
| Context | 64k tokens per request; 32k for `state` plus the longest question |
| Input | text only (string, JSON object, or array of text values) |

`usage.input_tokens * 0.042 / 1e6` gives the USD cost of a call at that price; for the documented 304-token example that is about $0.0000128.

---

## `APIPromise`: `withResponse`, `asResponse`, `map`

`systemOne` and `models.list` return `APIPromise<T>`, a `Promise<T>` subclass:

| Member | Returns | Behaviour |
|---|---|---|
| `await p` / `then` / `catch` / `finally` | parsed `T` | Parses the body once (memoised). |
| `withResponse()` | `Promise<{ data: T; response: Response; requestId: string \| undefined }>` | Parsed data plus the `Response` (body already consumed) and `x-typesafe-request-id`. |
| `asResponse()` | `Promise<Response>` | The raw `Response`, body unread. Non-2xx still rejects with `APIError`. "Don't also `await` the parsed result on the same promise" (TSDoc) - you own the body. |
| `map(fn)` | `APIPromise<U>` | Transforms the parsed value; the result still has `withResponse()`/`asResponse()` and shares the single HTTP response and parse. |

Implementation details from `dist/index.mjs`:

- The underlying HTTP exchange (including retries) runs once per `systemOne` call; `then`, `withResponse` and `map` all hang off the same response promise.
- Before handing over a `Response`, the SDK drains a **clone** of it under the per-attempt timeout (`bufferResponse`), so a slow body is covered by `timeout`, and the original stays readable for `asResponse()`.
- Non-2xx responses never reach `asResponse()`: they are turned into `APIError` subclasses inside the retry loop.

`requestId` is what TypeSafe support needs; log it on failures (`APIError.requestId` carries the same value).

---

## Timeouts and Cancellation with `AbortSignal`

Two independent mechanisms, which map to **different error classes**:

| Mechanism | Scope | Error | Retried? |
|---|---|---|---|
| `timeout` (client or per call) | each attempt separately, covering connect + full body | `APITimeoutError` (subclass of `APIConnectionError`), `timeoutMs` = configured value | yes, if `retry.apiTimeoutError` (default `true`) |
| `signal` (per call) | the whole call: current attempt **and** retry waits | `APIUserAbortError` | never |

Internally each attempt creates its own `AbortController`; both the SDK's timer and your `signal` abort it, and the SDK checks which fired to choose the class. Consequently **`AbortSignal.timeout(ms)` passed as `signal` produces `APIUserAbortError`, not `APITimeoutError`** - from the SDK's point of view the caller cancelled. Branch on both classes if you use both mechanisms.

### There is no total budget

`timeout` is per attempt and "there is no total retry budget" (TSDoc). With defaults (`timeout` 10 s, `maxRetries` 2, backoff 500 ms then 1000 ms minus up to 25 % jitter) a call that keeps timing out takes up to about 3 x 10 s + 1.5 s = 31.5 s before failing. If the server sends `Retry-After`, each wait can be as long as `maxRetryAfterMs` (60 s by default), so the theoretical worst case with defaults is 3 x 10 s + 2 x 60 s = 150 s. For a request path with a latency SLO, pass a `signal` that encodes the total budget.

```ts
// Type-checked: tsc 7.0.2 --strict, @typesafe-ai/sdk 0.6.0 (t10-deadline.mts)
import { APIUserAbortError, TypeSafeClient, noul } from "@typesafe-ai/sdk";

const client = new TypeSafeClient(); // TYPESAFE_API_KEY from the environment

// A total deadline across all attempts: the SDK's `timeout` is per attempt,
// so combine your own budget with the caller's signal.
export async function decideWithin(totalMs: number, callerSignal?: AbortSignal) {
  const signal = callerSignal
    ? AbortSignal.any([callerSignal, AbortSignal.timeout(totalMs)])
    : AbortSignal.timeout(totalMs);
  try {
    const r = await client.systemOne(
      { state: "x", questions: { urgent: noul("Is this urgent?") } },
      { signal, timeout: Math.min(totalMs, 5_000) },
    );
    return r.answers.urgent.noul;
  } catch (err) {
    if (err instanceof APIUserAbortError) return undefined; // budget spent or caller gone: fall back
    throw err;
  }
}
```

`AbortSignal.any` needs Node 20.3+ (within the SDK's `>=20` range only from 20.3). In an HTTP server, pass the request's own abort signal as `callerSignal` so a disconnected client stops the TypeSafe call and its retries.

The executed timeout/abort behaviour is part of the error test in [Error Classes](#error-classes) (cases 5 and 6).

---

## Retries (`RetryPolicy`)

All fields, defaults from both the TSDoc and `DEFAULT_RETRY_POLICY` in `dist/index.mjs` (they agree):

| Field | Default | Meaning |
|---|---|---|
| `maxRetries` | `2` | Retries after the first attempt (`0` disables). Total attempts = `maxRetries + 1`. |
| `backoffInitialMs` | `500` | First backoff; doubled per retry. |
| `backoffMaxMs` | `5000` | Cap for the exponential backoff. |
| `backoffJitter` | `0.25` | Fraction of each backoff randomly **subtracted** (0..1). |
| `httpStatuses` | `408`, `429`, `500`-`599` | Statuses that are retried (a `Set` of 102 codes). |
| `respectRetryAfter` | `true` | Honour `retry-after-ms` / `Retry-After`. |
| `maxRetryAfterMs` | `60000` | Server delays longer than this are ignored in favour of backoff. |
| `apiConnectionError` | `true` | Retry connection failures, including an interrupted response body. |
| `apiTimeoutError` | `true` | Retry `APITimeoutError`. |

### The exact algorithm (0.6.0)

For zero-based retry number `attempt`:

1. If `respectRetryAfter` and the response has headers: parse `retry-after-ms` (non-negative number of ms) first; otherwise `Retry-After` as seconds, or as an HTTP date (`max(0, date - now)`). If the result exists and is `<= maxRetryAfterMs`, wait exactly that long.
2. Otherwise wait `round(min(backoffInitialMs * 2^attempt, backoffMaxMs) * (1 - random() * backoffJitter))`. With defaults: retry 1 waits 375-500 ms, retry 2 waits 750-1000 ms.

What is retried:

| Failure | Retried when |
|---|---|
| HTTP status in `httpStatuses` | retries remain |
| `APITimeoutError` | `apiTimeoutError` is `true` and retries remain |
| `APIConnectionError` (thrown `fetch`, broken body) | `apiConnectionError` is `true` and retries remain |
| `APIUserAbortError` | never |
| any other status (400, 401, 403, 404, 422, ...) | never |

Each retry carries `X-TypeSafe-Retry-Count: <n>`; the first attempt omits the header. When retries are exhausted, the **last** error is thrown as-is (no wrapper class), so `instanceof RateLimitError` works after retries.

`529 Overloaded`, which the HTTP docs list alongside 429, falls in `500..599`, is retried by default, and surfaces as `InternalServerError` if it persists.

### Overrides

```ts
// Type-checked: tsc 7.0.2 --strict, @typesafe-ai/sdk 0.6.0 (excerpt of t09-overrides.mts)
import { TypeSafeClient, noul } from "@typesafe-ai/sdk";

const client = new TypeSafeClient({ apiKey: "k" });
const req = { state: "x", questions: { u: noul("?") } };

new TypeSafeClient({ retry: { maxRetries: 5, backoffMaxMs: 2_000 } });                 // client-wide
await client.systemOne(req, { retry: { maxRetries: 0 } });                               // this call only
await client.systemOne(req, { retry: { httpStatuses: new Set([429, 529]) } });           // narrower set
```

A per-call `retry` merges over the **client's** policy, so unspecified fields keep client overrides. `httpStatuses` is copied into a new `Set` at resolution, so mutating your set afterwards has no effect.

---

## Error Classes

```
Error
 └─ TypeSafeError                       base; also thrown for config/validation errors
     ├─ APIError                        non-2xx response: status, headers, body, requestId
     │   ├─ BadRequestError             400
     │   ├─ AuthenticationError         401
     │   ├─ PermissionDeniedError       403
     │   ├─ NotFoundError               404
     │   ├─ UnprocessableEntityError    422
     │   ├─ RateLimitError              429  (+ retryAfterMs)
     │   └─ InternalServerError         >= 500 (incl. 529)
     ├─ APIConnectionError              DNS/TLS/connection/body failure
     │   └─ APITimeoutError             per-attempt timeout (+ timeoutMs)
     └─ APIUserAbortError               caller's AbortSignal fired
```

Any other non-2xx status (e.g. 409, 413) becomes a plain `APIError`. `error.name` equals the class name (`new.target.name`), so it survives logging.

### `APIError` fields

| Field | Content |
|---|---|
| `status` | HTTP status |
| `headers` | the response `Headers` |
| `body` | parsed JSON, else the response text, else `undefined` for an empty body |
| `requestId` | `x-typesafe-request-id` or `undefined` |
| `message` | `"<status> <detail>"` - see below |
| `RateLimitError.retryAfterMs` | from `retry-after-ms` / `Retry-After`, or `undefined` |

The message detail is extracted in this order: a string body; `body.error` (string); `body.error.message`; `body.message`; `body.detail` (string); `body.detail.message`; a FastAPI-style `body.detail[]` array flattened to `"loc.path: msg; ..."` with the leading `body` segment dropped. Without any of those, the raw body is appended (truncated to 200 characters), or `"<status> status code (no body)"`.

### Executed error/retry/timeout test

Every assertion below passed against 0.6.0 with a fake `fetch` (no network):

```ts
// Executed offline: @typesafe-ai/sdk 0.6.0, Node 22.22.0 (t03-errors-retries.mts)
import assert from "node:assert/strict";
import {
  APIConnectionError,
  APIError,
  APITimeoutError,
  APIUserAbortError,
  AuthenticationError,
  InternalServerError,
  RateLimitError,
  TypeSafeClient,
  TypeSafeError,
  UnprocessableEntityError,
  noul,
  type Fetch,
} from "@typesafe-ai/sdk";

const json = (body: unknown, status = 200, headers: Record<string, string> = {}) =>
  new Response(JSON.stringify(body), {
    status,
    headers: { "content-type": "application/json", ...headers },
  });

const OK = {
  model: "jev-1.13.0",
  answers: { urgent: { type: "noul", noul: 0.42 } },
  usage: { input_tokens: 10, output_tokens: 20 },
};
const ask = { state: "x", questions: { urgent: noul("Is this urgent?") } };

// 1. 429 is retried; retry-after-ms is honoured; retries carry X-TypeSafe-Retry-Count.
{
  const seen: Array<string | undefined> = [];
  const fetch: Fetch = async (_url, init) => {
    seen.push((init?.headers as Record<string, string>)["X-TypeSafe-Retry-Count"]);
    return seen.length < 3
      ? json({ message: "slow down" }, 429, { "retry-after-ms": "0" })
      : json(OK);
  };
  const client = new TypeSafeClient({ apiKey: "k", fetch, logLevel: "off" });
  const res = await client.systemOne(ask);
  assert.equal(res.answers.urgent.noul, 0.42);
  assert.deepEqual(seen, [undefined, "1", "2"]);
}

// 2. Retries exhausted -> the last APIError subclass is thrown (no wrapper error).
{
  let n = 0;
  const fetch: Fetch = async () => {
    n++;
    return json({ message: "slow down" }, 429, { "retry-after": "0" });
  };
  const client = new TypeSafeClient({ apiKey: "k", fetch, logLevel: "off" });
  const err = await client.systemOne(ask).catch((e: unknown) => e);
  assert.ok(err instanceof RateLimitError);
  assert.ok(err instanceof APIError && err instanceof TypeSafeError);
  assert.equal(err.status, 429);
  assert.equal(err.retryAfterMs, 0);
  assert.equal(err.message, "429 slow down");
  assert.equal(n, 3); // 1 attempt + maxRetries (2)
}

// 3. 422 is not retried; FastAPI-style `detail[]` is flattened into the message.
{
  let n = 0;
  const fetch: Fetch = async () => {
    n++;
    return json(
      {
        detail: [
          { loc: ["body", "questions", "q", "criteria"], msg: "List should have at least 1 item", type: "too_short" },
        ],
      },
      422,
      { "x-typesafe-request-id": "req_422" },
    );
  };
  const client = new TypeSafeClient({ apiKey: "k", fetch, logLevel: "off" });
  const err = await client.systemOne(ask).catch((e: unknown) => e);
  assert.ok(err instanceof UnprocessableEntityError);
  assert.equal(err.message, "422 questions.q.criteria: List should have at least 1 item");
  assert.equal(err.requestId, "req_422");
  assert.equal(n, 1);
}

// 4. 401 -> AuthenticationError; 529 -> InternalServerError (any 5xx), retried by default.
{
  const client401 = new TypeSafeClient({
    apiKey: "k",
    logLevel: "off",
    fetch: async () => json({ error: { message: "invalid key" } }, 401),
  });
  const e401 = await client401.systemOne(ask).catch((e: unknown) => e);
  assert.ok(e401 instanceof AuthenticationError);
  assert.equal(e401.message, "401 invalid key");

  let n = 0;
  const client529 = new TypeSafeClient({
    apiKey: "k",
    logLevel: "off",
    retry: { backoffInitialMs: 0 },
    fetch: async () => (++n, new Response("", { status: 529 })),
  });
  const e529 = await client529.systemOne(ask).catch((e: unknown) => e);
  assert.ok(e529 instanceof InternalServerError);
  assert.equal(e529.message, "529 status code (no body)");
  assert.equal(n, 3);
}

// A fetch that never answers but honours its AbortSignal, like the global fetch.
const hang: Fetch = (_url, init) =>
  new Promise((_resolve, reject) => {
    init?.signal?.addEventListener("abort", () => reject(init.signal?.reason), { once: true });
  });

// 5. The per-attempt `timeout` -> APITimeoutError (a subclass of APIConnectionError).
{
  const client = new TypeSafeClient({ apiKey: "k", fetch: hang, logLevel: "off" });
  const started = Date.now();
  const err = await client
    .systemOne(ask, { timeout: 50, retry: { maxRetries: 0 } })
    .catch((e: unknown) => e);
  assert.ok(err instanceof APITimeoutError);
  assert.ok(err instanceof APIConnectionError);
  assert.equal(err.timeoutMs, 50);
  assert.equal(err.message, "Request timed out after 50ms.");
  assert.ok(Date.now() - started < 1000);
}

// 6. A caller signal, including AbortSignal.timeout(), -> APIUserAbortError, never retried.
{
  let n = 0;
  const counting: Fetch = (url, init) => (n++, hang(url, init));
  const client = new TypeSafeClient({ apiKey: "k", fetch: counting, logLevel: "off" });
  const err = await client
    .systemOne(ask, { signal: AbortSignal.timeout(50) })
    .catch((e: unknown) => e);
  assert.ok(err instanceof APIUserAbortError);
  assert.ok(!(err instanceof APIConnectionError));
  assert.equal(n, 1);
}

// 7. A thrown network error -> APIConnectionError, retried per `apiConnectionError`.
{
  let n = 0;
  const client = new TypeSafeClient({
    apiKey: "k",
    logLevel: "off",
    retry: { backoffInitialMs: 0 },
    fetch: async () => {
      n++;
      throw new TypeError("fetch failed");
    },
  });
  const err = await client.systemOne(ask).catch((e: unknown) => e);
  assert.ok(err instanceof APIConnectionError);
  assert.equal(err.message, "Connection error: fetch failed");
  assert.equal(n, 3);
}

// 8. Client-side validation throws synchronously, before any request.
{
  const client = new TypeSafeClient({ apiKey: "k", fetch: hang, logLevel: "off" });
  assert.throws(() => client.systemOne({ state: "x", questions: {} }), {
    name: "TypeSafeError",
    message: "At least one question is required.",
  });
  assert.throws(
    () => client.systemOne({ state: "x", questions: { s: { type: "score", criteria: ["a"] as never } } }),
    { message: 'Score question "s" has 1 criteria; at least two scores are required.' },
  );
}

console.log("ok");
```

The 401/422/429 bodies in this test are invented shapes chosen to exercise each message-extraction branch; the real API's error bodies were not observed (no API key). The AI Gateway documents its TypeSafe-compatible error body as `{ "message": "...", "error_type": "invalid_request" }` (https://vercel.com/docs/ai-gateway/sdks-and-apis/typesafe), which the `body.message` branch handles.

### A production handler

| Error | Typical action |
|---|---|
| `AuthenticationError`, `PermissionDeniedError` | configuration bug - alert, do not retry |
| `BadRequestError`, `UnprocessableEntityError` | your question/state is malformed - log `message` + `requestId`, fix the code |
| `RateLimitError`, `InternalServerError` (after SDK retries) | degrade: queue the work, or fall back to a default decision / human review |
| `APITimeoutError`, `APIConnectionError` | same as above; consider a lower `timeout` with more retries if latency matters |
| `APIUserAbortError` | the caller left or your budget expired - stop silently |
| plain `TypeSafeError` | thrown synchronously by config or question validation - a code bug |

Because Jev returns probabilities, the safe fallback for an unanswered decision is usually the same path you use for low-confidence answers (human review, conservative default), not a guess.

---

## Logging

- `logLevel` filters a `Logger` (`debug`/`info`/`warn`/`error` methods taking `(message, ...args)`, compatible with `console`). Order: `debug` < `info` < `warn` < `error` < `off`. Default `warn` - which in 0.6.0 means **silent**, because the SDK itself only emits `info` and `debug` lines.
- `info` logs one summary per attempt (`#1 POST /v1/systemone <- 200 in 12ms (request req_x)`), retries (`retrying in 412ms (retry 1/2) after 429`), timeouts, aborts and connection errors.
- `debug` adds the outgoing headers and **request body**, and the parsed response/error bodies.
- Redaction applies to header values only: `authorization`, `proxy-authorization`, `x-api-key` keep their scheme and the last four characters of secrets longer than eight characters (`Bearer ***ijkl`); `cookie`/`set-cookie` become `***`. **Bodies are never redacted**, so `debug` writes your `state` (customer text, PII) to the log sink.
- The default sink is `console` with a `[typesafe-sdk]` prefix.

```ts
// Executed offline: @typesafe-ai/sdk 0.6.0, Node 22.22.0 (excerpt of t04-config-logging.mts)
import assert from "node:assert/strict";
import { TypeSafeClient, noul, type Fetch, type Logger } from "@typesafe-ai/sdk";

const json = (body: unknown, status = 200) =>
  new Response(JSON.stringify(body), { status, headers: { "content-type": "application/json" } });

// A custom logger at debug level: credentials are redacted, bodies are not.
const lines: string[] = [];
const logger: Logger = {
  debug: (m, ...a) => lines.push(`debug ${m} ${JSON.stringify(a)}`),
  info: (m) => lines.push(`info ${m}`),
  warn: (m) => lines.push(`warn ${m}`),
  error: (m) => lines.push(`error ${m}`),
};
const fetch: Fetch = async () =>
  json({ model: "jev-1.13.0", answers: { u: { type: "noul", noul: 0.5 } }, usage: { input_tokens: 1, output_tokens: 1 } });
const client = new TypeSafeClient({
  apiKey: "sk-live-abcdefghijkl",
  fetch,
  logger,
  logLevel: "debug",
  defaultHeaders: { "X-Team": "support" },
});
await client.systemOne({ state: "secret text", questions: { u: noul("?") } }, { headers: { "x-team": "billing" } });
const out = lines.join("\n");
assert.ok(out.includes("Bearer ***ijkl")); // last four characters kept
assert.ok(!out.includes("abcdefghijkl"));
assert.ok(out.includes("secret text")); // request bodies are logged verbatim at debug
assert.ok(out.includes('"x-team":"billing"')); // per-call header wins, case-insensitively
assert.ok(lines.some((l) => /^info #1 POST \/v1\/systemone <- 200 in \d+ms$/.test(l)));
```

To ship logs to a structured logger (pino, winston), pass an adapter object with the four methods; the `...args` are structured values (header maps, bodies, the error) rather than pre-formatted strings.

---

## Models Resource

```ts
client.models.list(options?: RequestOptions): APIPromise<ModelCard[]>
// ModelCard = { readonly name: string; readonly description: string; readonly release_date: string }
```

`GET /v1/models` returns `{ "models": [...] }`; the SDK unwraps it to the array and throws `TypeSafeError("Unexpected response shape from GET /v1/models; expected { models: [...] }.")` if `models` is not an array. Per https://docs.typesafe.ai/models the list "currently lists the aliases" (`jev-latest`, `jev-preview`); versioned IDs such as `jev-1.13.0` are accepted by the `model` field whether or not they are listed.

Verbatim from https://docs.typesafe.ai/models:

```typescript
import { TypeSafeClient } from "@typesafe-ai/sdk";

const client = new TypeSafeClient();
const models = await client.models.list();
for (const model of models) {
  console.log(model.name, model.release_date, model.description);
}
```

Aliases at 2026-10-03 (same page): `jev-latest` -> `jev-1.13.0`; `jev-preview` -> `jev-1.13.0` ("There is no preview build available right now"). If you tuned thresholds against a version, pin `jev-1.13.0` via `defaultModel` or per-request `model` instead of the alias.

---

## HTTP Headers the SDK Sends

Observed in the executed fake-fetch test (first attempt) and read from `fetchWithRetries`:

| Header | Value |
|---|---|
| `Authorization` | `Bearer <apiKey>` |
| `Accept` | `application/json` |
| `User-Agent` | `typesafe-sdk/0.6.0` |
| `X-TypeSafe-SDK` | `typesafe-sdk/0.6.0` |
| `X-TypeSafe-Runtime` | e.g. `node/22.22.0 (win32; x64)`, `bun/<v>`, `deno/<v>`, `vercel-edge`, `cloudflare-workers`, `browser`, `unknown` |
| `Content-Type` | `application/json` (only when there is a body, i.e. `systemOne`) |
| `X-TypeSafe-Retry-Count` | `1`, `2`, ... on retries only |

Plus your `defaultHeaders` and per-call `headers` (which cannot override the rows above). The response header `x-typesafe-request-id` is surfaced as `requestId`.

---

## Testing with a Custom `fetch` (executed)

The `fetch` option is the intended seam for tests: it receives the final URL and `RequestInit` (method, headers as a plain object, JSON string body, the SDK's `AbortSignal`), and returns a standard `Response`. No HTTP mocking library is needed. The following file ran green against 0.6.0 and pins down what the SDK sends and returns:

```ts
// Executed offline: @typesafe-ai/sdk 0.6.0, Node 22.22.0 (t02-fake-fetch.mts)
import assert from "node:assert/strict";
import { choice, noul, score, TypeSafeClient, type Fetch } from "@typesafe-ai/sdk";

// A recorded call, so the test can assert on what the SDK actually sent.
type Call = { url: string; init: RequestInit | undefined };

function fakeFetch(respond: (call: Call, n: number) => Response): { fetch: Fetch; calls: Call[] } {
  const calls: Call[] = [];
  const fetch: Fetch = async (url, init) => {
    const call = { url, init };
    calls.push(call);
    return respond(call, calls.length);
  };
  return { fetch, calls };
}

const json = (body: unknown, status = 200, headers: Record<string, string> = {}) =>
  new Response(JSON.stringify(body), {
    status,
    headers: { "content-type": "application/json", ...headers },
  });

const { fetch, calls } = fakeFetch(() =>
  json(
    {
      model: "jev-1.13.0",
      answers: {
        team: {
          type: "choice",
          choice: "billing",
          probabilities: { infra: 0.1, billing: 0.88, other: 0.02 },
          confidence: 0.81,
        },
        urgent: { type: "noul", noul: 0.95 },
        severity: {
          type: "score",
          score: 1.05,
          legend: { "0": "Cosmetic.", "1": "Degraded.", "2": "Outage." },
          probabilities: { "0": 0.0, "1": 0.95, "2": 0.05 },
          confidence: 0.92,
        },
      },
      usage: { input_tokens: 318, output_tokens: 34 },
    },
    200,
    { "x-typesafe-request-id": "req_123" },
  ),
);

const client = new TypeSafeClient({ apiKey: "sk-test-0123456789", fetch, logLevel: "off" });

const { data, response, requestId } = await client
  .systemOne({
    state: "I was charged twice. Please fix this ASAP.",
    questions: {
      team: choice("Which team?", { infra: null, billing: null, other: null }),
      urgent: noul("Is this urgent?"),
      severity: score("How severe?", ["Cosmetic.", "Degraded.", "Outage."]),
    },
  })
  .withResponse();

assert.equal(data.answers.team.choice, "billing");
assert.equal(data.answers.urgent.noul, 0.95);
assert.equal(data.answers.severity.score, 1.05);
assert.equal(response.status, 200);
assert.equal(requestId, "req_123");

// What went over the wire.
const [call] = calls;
assert.equal(call.url, "https://api.typesafe.ai/v1/systemone");
assert.equal(call.init?.method, "POST");
const sent = JSON.parse(String(call.init?.body));
assert.equal(sent.model, "jev-latest"); // defaultModel filled in by the SDK
assert.deepEqual(sent.questions.urgent, { type: "noul", instructions: "Is this urgent?" }); // criteria: undefined dropped by JSON
const headers = call.init?.headers as Record<string, string>;
assert.equal(headers.Authorization, "Bearer sk-test-0123456789");
assert.equal(headers["User-Agent"], "typesafe-sdk/0.6.0");
assert.equal(headers["Content-Type"], "application/json");
assert.ok(headers["X-TypeSafe-Runtime"].startsWith("node/"));
assert.ok(!("X-TypeSafe-Retry-Count" in headers)); // only sent on retries

// .map() transforms the parsed body; the same APIPromise still exposes withResponse().
const mapped = client
  .systemOne({ state: "x", questions: { urgent: noul("Is this urgent?") } })
  .map((r) => r.answers.urgent.noul >= 0.9);
const { data: isUrgent, requestId: rid2 } = await mapped.withResponse();
assert.equal(isUrgent, true);
assert.equal(rid2, "req_123");

// The SDK does not validate the response against the question map: the types promise
// an answer under every key, but a body without it yields undefined at runtime.
const missing = await client.systemOne({ state: "x", questions: { other: noul("?") } });
assert.equal(missing.answers.other, undefined);
console.log("sent headers:", Object.keys(headers).join(", "));
console.log("ok");
```

Output: `sent headers: Authorization, Accept, User-Agent, X-TypeSafe-SDK, X-TypeSafe-Runtime, Content-Type` then `ok`.

Testing guidance that follows from the implementation:

- **Fakes must honour `init.signal`** if you test timeouts: the SDK aborts the per-attempt controller and expects `fetch` to reject. A fake that ignores the signal will hang until its own completion - the `hang` helper in the error test shows the minimal correct shape.
- **Set `retry: { maxRetries: 0 }` or `backoffInitialMs: 0`** in tests that exercise 429/5xx, or send `retry-after-ms: 0`, to avoid real backoff delays.
- **Pin `logLevel: "off"`** (or inject a capturing logger) to keep test output clean regardless of `TYPESAFE_LOG_LEVEL` in CI.
- **Answer fixtures should be realistic**: two-decimal probabilities that may sum to 0.99/1.01, `confidence` on choice/score only, and `legend` on score. Since the SDK does not validate, a sloppy fixture will not fail where production would differ.
- **Decision logic should be tested independently of the SDK**: wrap `systemOne` behind your own function that returns your domain decision, and unit-test the thresholds with plain answer objects.

---

## Server-Side Only: `dangerouslyAllowBrowser`

The constructor refuses to run when browser page globals exist (`window`, `window.document` and `navigator` all defined) unless `dangerouslyAllowBrowser: true`:

> TypeSafeClient is running in a browser, which would expose your API key to anyone using the page. Call the API from a server instead, or pass `dangerouslyAllowBrowser: true` if you understand the risk.

(Verbatim error message from `dist/index.mjs`.) The executed check:

```ts
// Executed offline: @typesafe-ai/sdk 0.6.0, Node 22.22.0 (excerpt of t04-config-logging.mts)
import assert from "node:assert/strict";
import { TypeSafeClient } from "@typesafe-ai/sdk";

// Browser guard: page globals present and no opt-in -> TypeSafeError.
const g = globalThis as Record<string, unknown>;
g.window = { document: {} };
assert.throws(() => new TypeSafeClient({ apiKey: "k" }), /running in a browser/);
assert.doesNotThrow(() => new TypeSafeClient({ apiKey: "k", dangerouslyAllowBrowser: true }));
delete g.window;
```

Guidance:

- TypeSafe API keys are account credentials billed per input token; a key in a bundle is a key published. Call TypeSafe from a server route (Next.js route handler / server action, Express, a worker) and return only the decision the UI needs.
- Workers and edge runtimes are not browsers (no `window.document`), so the guard does not fire there; the SDK identifies them in `X-TypeSafe-Runtime` (`cloudflare-workers`, `vercel-edge`). On runtimes without `process.env`, pass `apiKey` explicitly.
- Legitimate uses of the flag are narrow: a local demo with a throwaway key, or an Electron/desktop context where the "page user" is the key owner. Even then prefer a short-lived proxy.
- Through Vercel AI Gateway, the gateway (not TypeSafe) authenticates you and the browser-exposure argument applies to the gateway key in the same way.

---

## Gateways and Extension Properties

Because `baseURL` is configurable and the SDK posts to `${baseURL}/v1/systemone`, any TypeSafe-compatible endpoint works. Vercel AI Gateway exposes one at `https://ai-gateway.vercel.sh/typesafe` (model `typesafe-ai/jev`; supported endpoints `POST /typesafe/v1/systemone` and `GET /typesafe/v1/models`). Verbatim from https://vercel.com/docs/ai-gateway/sdks-and-apis/typesafe (page `last_updated: 2026-09-21`), type-checked here against 0.6.0:

```typescript
import { TypeSafeClient } from '@typesafe-ai/sdk';

const client = new TypeSafeClient({
  apiKey: process.env.AI_GATEWAY_API_KEY,
  baseURL: 'https://ai-gateway.vercel.sh/typesafe',
});

const result = await client.systemOne({
  model: 'typesafe-ai/jev',
  state: 'I was charged twice for my subscription.',
  questions: {
    refund: {
      type: 'noul',
      instructions: 'Is the customer asking for money back?',
    },
  },
});
```

The TypeSafe docs also show OpenRouter (`base_url="https://openrouter.ai/api"`, model `~typesafe/jev-latest`) and Pydantic AI Gateway (`https://gateway-us.pydantic.dev/proxy/typesafe`) with the **Python** SDK (https://docs.typesafe.ai/sdk/python/usage, "Configuring the base URL"); the JavaScript equivalents use `baseURL`/`defaultModel` but were not exercised here.

### Passing gateway-only fields (`providerOptions`)

The Gateway's evaluation fallbacks are configured with a `providerOptions.gateway.models` field that is not part of `SystemOneRequest`. The Vercel page notes "The official TypeSafe SDK forwards additional request properties at runtime, but its request type does not currently include `providerOptions`. Supply the extension through an intermediate object." Executed against 0.6.0, including the compile-time behaviour:

```ts
// Executed offline: @typesafe-ai/sdk 0.6.0, Node 22.22.0 (excerpt of t05-extras.mts)
import assert from "node:assert/strict";
import { TypeSafeClient, choice, noul, type Fetch } from "@typesafe-ai/sdk";

let lastBody: Record<string, unknown> = {};
const fetch: Fetch = async (_url, init) => {
  lastBody = JSON.parse(String(init?.body));
  return new Response(
    JSON.stringify({
      model: "typesafe-ai/jev",
      answers: { intent: { type: "choice", choice: "billing", probabilities: { billing: 0.7, account: 0.3 }, confidence: 0.55 } },
      usage: { input_tokens: 1, output_tokens: 1 },
    }),
    { headers: { "content-type": "application/json", "x-ai-gateway-evaluation-fallback-triggered": "true" } },
  );
};
const client = new TypeSafeClient({ apiKey: "k", baseURL: "https://ai-gateway.vercel.sh/typesafe", fetch, logLevel: "off" });

// 1. Extra request properties are forwarded at runtime. A fresh object literal with an
//    unknown property is a compile error (excess-property check); a variable is not.
const questions = {
  intent: choice("Which team should handle this?", {
    billing: "Charges and refunds",
    account: "Sign-in and account access",
  }),
};
// @ts-expect-error -- 'providerOptions' does not exist in type SystemOneRequest
client.systemOne({ state: "x", questions, providerOptions: {} });
const request = {
  model: "typesafe-ai/jev",
  state: "I was charged twice and cannot sign in.",
  questions,
  providerOptions: {
    gateway: { models: [{ model: "openai/gpt-6-astra", when: { question: "intent", confidenceBelow: 0.6 } }] },
  },
};
const { data, response } = await client.systemOne(request).withResponse();
assert.equal(data.answers.intent.choice, "billing");
assert.ok("providerOptions" in lastBody);
assert.equal(response.headers.get("x-ai-gateway-evaluation-fallback-triggered"), "true");

// 2. asResponse(): the raw Response, body still unread (the SDK buffered a clone).
const raw = await client.systemOne({ state: "x", questions: { u: noul("?") } }).asResponse();
const text = await raw.text();
assert.ok(text.includes('"typesafe-ai/jev"'));
```

Gateway facts to handle in code (all from the Vercel page above):

- When a fallback answered, the response carries `x-ai-gateway-evaluation-fallback-triggered: true`, `-final-model`, `-primary-model` and `-triggering-questions` (percent-encoded JSON array) headers - read them via `withResponse()`, since the typed result has no field for them.
- "If a language model returns the final Choice or Score answer, `confidence: 0` and `probabilities: {}` mean those values are unavailable. They do not mean the model measured zero confidence." A confidence-threshold router written for native Jev answers would treat those as maximally uncertain; check `probabilities` for emptiness first.
- The gateway adds a `provider_metadata.gateway` object (routing, `cost` in USD as a string) to the body. The SDK passes it through untyped; access it with a cast.
- Gateway errors use `{ "message": ..., "error_type": ... }`; the SDK's message extraction picks up `message`.

---

## `VERSION`, `LOG_LEVELS` and Other Constants

Executed output of `console.log(VERSION, ENV, LOG_LEVELS)` against 0.6.0:

```text
0.6.0 {
  apiKey: 'TYPESAFE_API_KEY',
  baseURL: 'TYPESAFE_BASE_URL',
  defaultModel: 'TYPESAFE_DEFAULT_MODEL',
  logLevel: 'TYPESAFE_LOG_LEVEL'
} [ 'debug', 'info', 'warn', 'error', 'off' ]
```

- `VERSION` - the SDK version string (`"0.6.0"`), also sent in `User-Agent` and `X-TypeSafe-SDK`. Useful in your own diagnostics or to assert the installed version in a health check.
- `LOG_LEVELS` - `readonly LogLevel[]`, most to least verbose; use it to validate a log level read from your own config before passing it in.
- `ENV` - see [Environment Variables](#environment-variables-and-env).

---

## Gotchas Checklist

1. **Validation errors are synchronous throws.** Use `await` inside `try`, not `.catch()` on the call expression.
2. **Requests start immediately** and an unawaited failure becomes an `unhandledRejection`.
3. **`timeout` is per attempt**; with defaults a failing call can take ~31.5 s (150 s if the server asks for long `Retry-After`). Use a `signal` for a total budget.
4. **`AbortSignal.timeout()` yields `APIUserAbortError`**, not `APITimeoutError`.
5. **`baseURL` must not end in `/v1`** (unlike the AI SDK provider's).
6. **Different env var than the AI SDK provider**: `TYPESAFE_API_KEY` here, `TYPESAFE_AI_API_KEY` there.
7. **No response validation**: a missing answer is `undefined` at runtime. Defend when using gateways or mocks.
8. **Limits 255 options / 10 levels are not checked locally.**
9. **`debug` logs bodies unredacted** - never enable it in production with real customer `state`.
10. **Pin the model ID** (`jev-1.13.0`) if your thresholds were calibrated on it; `jev-latest` moves.
11. **Round-trip probabilities are two-decimal**; don't compare with `===` against 0 or 1, and don't expect exact sums.
12. **Score rubric must be a tuple type** for precise key inference; `string[]` does not type-check against `score()`.

---

## What Was Not Verified

- No call was made to the real API (no API key was available). Request/response shapes were taken from https://docs.typesafe.ai/api and exercised against fake `fetch` implementations; the exact error bodies the live API returns for 401/422/429 were not observed.
- Behaviour on Bun, Deno, Cloudflare Workers and Vercel Edge was read from the runtime-detection code, not executed.
- The CommonJS build (`dist/index.cjs`) was not executed; only the ESM build was.
- Whether the live API accepts a question with `instructions: null` (as `noul()` produces by default) was not tested.
- The JSR distribution, if published, was not inspected.
