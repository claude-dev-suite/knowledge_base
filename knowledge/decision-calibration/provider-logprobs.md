# Class Probabilities from LLM Providers (logprobs) - Deep Reference

> Official Documentation: https://platform.claude.com/docs/en/api/openai-sdk
> Official Documentation: https://developers.openai.com/api/reference/resources/chat
> Official Documentation: https://ai.google.dev/api/generate-content
> Official Documentation: https://github.com/vllm-project/vllm/blob/v0.30.0/docs/features/structured_outputs.md
> Official Documentation: https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md
> Last verified: 2026-10-03
> Verified against: Anthropic docs (platform.claude.com, fetched 2026-10-03); OpenAI developer docs (developers.openai.com, fetched 2026-10-03) and `openai` Python SDK 2.54.0; Gemini API reference (ai.google.dev, fetched 2026-10-03); vLLM v0.30.0 (released 2026-09-22, source at that tag); llama.cpp `master` at commit 436f6f8 (2026-10-03); `tiktoken` 0.14.0

## Overview

Calibration, order debiasing and conformal sets (the sibling pages `calibration.md`, `order-bias.md`, `conformal.md`) all start from a probability per class. A typed decision model such as TypeSafe Jev returns one natively. A general LLM returns one only if its API exposes **token log-probabilities**, and only for a carefully shaped prompt. This page re-verifies, against each provider's current documentation and source, what you can get, shows the exact request shapes, gives the single-token-enum recipe that turns token logprobs into a class posterior, and covers the fallbacks for providers that expose nothing - Claude above all.

Every request below is either quoted verbatim from the provider's docs (URL above it) or built and parsed by code that was executed offline: the OpenAI SDK against an `httpx.MockTransport` returning the documented response shape, the parsers against mock JSON in each provider's documented shape. **No real API was called**; whether a given model accepts a given request today was not tested.

---

## Table of Contents

1. [The matrix](#the-matrix)
2. [Anthropic (Claude)](#anthropic-claude)
3. [OpenAI](#openai)
4. [Google Gemini](#google-gemini)
5. [vLLM](#vllm)
6. [llama.cpp server](#llamacpp-server)
7. [The single-token-enum recipe](#the-single-token-enum-recipe)
8. [Without logprobs: sampling frequency, verbalized confidence, panels](#without-logprobs-sampling-frequency-verbalized-confidence-panels)
9. [Checklist](#checklist)
10. [Sources](#sources)

---

## The matrix

As of 2026-10-03:

| Provider | Class probabilities? | How | Caveats |
|---|---|---|---|
| **TypeSafe Jev** (hosted) | yes, native | `probabilities` (Choice/Score), `noul` (Noul) | API examples show 2 decimals; exact 0/1 occur (`calibration.md`) |
| **Anthropic Claude** | **no** | - | Messages API has no logprobs parameter; OpenAI-compat `logprobs`/`top_logprobs` "Ignored", response `logprobs` "Always empty" |
| **OpenAI** | yes, restricted | Chat Completions `logprobs` + `top_logprobs` (0-20); Responses `include: ["message.output_text.logprobs"]` + `top_logprobs` | GPT-6 family: remove them unless reasoning effort is `none`; GPT-6 Astra and GPT-6.1 Sol do not support `none` |
| **Google Gemini** | documented, but **not on 3.x models** | `generationConfig.responseLogprobs` + `logprobs` (0-20); `text/x.enum` for an enum answer | Google staff on the developer forum (2026-08-05): "logprobs are no longer returned for 3.X models" |
| **vLLM** (self-hosted) | yes | `logprobs`/`top_logprobs`, capped by `max_logprobs` (default 20); `logprob_token_ids` for exact label tokens; `structured_outputs.choice` | default `logprobs_mode` is `raw_logprobs`: values **before** logit processors |
| **llama.cpp server** (self-hosted) | yes; plus a TypeSafe-compatible `/v1/systemone` for decision models | `n_probs`; `post_sampling_probs`; `grammar` / `json_schema` | default probabilities are pre-sampling (no grammar applied); post-sampling ones are after the whole sampler chain |

---

## Anthropic (Claude)

**Native Messages API.** The `POST /v1/messages` reference has no `logprobs` (or similarly named) request parameter and no such response field: the API reference page (https://platform.claude.com/docs/en/api/messages/create.md, fetched 2026-10-03) contains zero occurrences of "logprob".

**OpenAI SDK compatibility layer.** The request-parameter table (https://platform.claude.com/docs/en/api/openai-sdk, fetched 2026-10-03) lists, verbatim:

| Field | Support status |
|---|---|
| `logprobs` | Ignored |
| `top_logprobs` | Ignored |
| `response_format` | Ignored. For JSON output, use [Structured Outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs) with the native Claude API |
| `n` | Must be exactly 1 |
| `temperature` | Between 0 and 1 (inclusive). Values greater than 1 are capped at 1. |

and in the response-field table: `logprobs` - "Always empty".

**Typed, not probabilistic.** Structured outputs constrain Claude to a JSON schema that may contain an `enum`, so the answer is always one of your labels - but you get one label, not a distribution. The docs' request shape, verbatim (https://platform.claude.com/docs/en/build-with-claude/structured-outputs, fetched 2026-10-03):

```bash
# verbatim, https://platform.claude.com/docs/en/build-with-claude/structured-outputs
    curl https://api.anthropic.com/v1/messages \
      -H "content-type: application/json" \
      -H "x-api-key: $ANTHROPIC_API_KEY" \
      -H "anthropic-version: 2023-06-01" \
      -d '{
        "model": "claude-opus-5-5",
        "max_tokens": 1024,
        "messages": [
          {
            "role": "user",
            "content": "Extract the key information from this email: John Smith (john@example.com) is interested in our Enterprise plan and wants to schedule a demo for next Tuesday at 2pm."
          }
        ],
        "output_config": {
          "format": {
            "type": "json_schema",
            "schema": {
              "type": "object",
              "properties": {
                "name": {"type": "string"},
                "email": {"type": "string"},
                "plan_interest": {"type": "string"},
                "demo_requested": {"type": "boolean"}
              },
              "required": ["name", "email", "plan_interest", "demo_requested"],
              "additionalProperties": false
            }
          }
        }
      }'
```

The same shape with a single `enum` field, as a builder (executed by the tests; not sent to the API). It and the other request builders on this page share one prompt helper from the companion module, `kb_calib.py` (not published in this knowledge base; its imports are listed in `calibration.md`):

```python
def _classify_prompt(question, labels, text):
    return f"{question}\nAnswer with exactly one of: {', '.join(labels)}.\n\n{text}"
```

```python
def anthropic_enum_request(model, question, labels, text):
    """Messages API body with JSON outputs: a typed enum, NOT a distribution."""
    return {
        "model": model,
        "max_tokens": 64,
        "messages": [{"role": "user", "content": _classify_prompt(question, labels, text)}],
        "output_config": {
            "format": {
                "type": "json_schema",
                "schema": {
                    "type": "object",
                    "properties": {"label": {"type": "string", "enum": list(labels)}},
                    "required": ["label"],
                    "additionalProperties": False,
                },
            }
        },
    }
```

Two documented caveats that matter for classification:

- **Enum casing**: "Structured outputs don't guarantee the capitalization of string `enum` and `const` values: Claude may return a value that differs from your schema only in capitalization, typically in the first letter of a word following a space." The docs' advice: "Compare enum values case-insensitively, and avoid enum values that differ only in capitalization." (same page, *Invalid outputs*).
- **Sampling parameters are fixed on recent models**: `temperature` - "Deprecated. Models released after Claude Opus 4.6 do not support setting temperature. A value of 1.0 will be accepted for backwards compatibility, all other values will be rejected with a 400 error." `top_p`: only values ≥ 0.99 accepted; `top_k`: "any value will be rejected with a 400 error" (Messages API reference, fetched 2026-10-03). For the sampling-frequency fallback below that is convenient: repeated calls already sample at temperature 1.

So a Claude "probability" always costs more than one call: see [the fallbacks](#without-logprobs-sampling-frequency-verbalized-confidence-panels).

---

## OpenAI

**Chat Completions parameters**, verbatim from the API reference (https://developers.openai.com/api/reference/resources/chat, fetched 2026-10-03):

> `logprobs: optional boolean or null` - Whether to return log probabilities of the output tokens or not. If true, returns the log probabilities of each output token returned in the `content` of `message`.
>
> `top_logprobs: optional number or null` - An integer between 0 and 20 specifying the maximum number of most likely tokens to return at each token position, each with an associated log probability. In some cases, the number of returned tokens may be fewer than requested. `logprobs` must be set to `true` if this parameter is used.

Response: `choices[].logprobs` is `object { content, refusal }`; each `content` entry is a `ChatCompletionTokenLogprob` with `token`, `bytes`, `logprob` and `top_logprobs: array of object { token, bytes, logprob }`.

**Responses API**: `top_logprobs` has the same 0-20 range, and the per-token logprobs are requested with `include: ["message.output_text.logprobs"]` ("Include logprobs with assistant messages") (https://developers.openai.com/api/reference/resources/responses/methods/create.md, fetched 2026-10-03).

**GPT-6 restriction**, verbatim from the latest-model guide (https://developers.openai.com/api/docs/guides/latest-model.md, fetched 2026-10-03):

> **Unsupported parameters:** When reasoning effort is not `none`, remove `temperature`, `top_p`, and `top_logprobs`. For Chat Completions, also remove `logprobs`. For Responses, remove `message.output_text.logprobs` from `include`.

> **Reasoning effort:** ... GPT-6 Astra and GPT-6.1 Sol do not support `none`; use `low` instead. GPT-6 Sol and GPT-6 Luna support `none`.

Combined: on **GPT-6 Sol and GPT-6 Luna** logprobs are available only with `reasoning_effort: "none"`; on **GPT-6 Astra and GPT-6.1 Sol** there is no setting that allows them. (A third-party SDK, the Vercel AI SDK's OpenAI provider page, states more broadly that "GPT-6 and later models do not support `temperature`, `topP`, `logprobs`" and strips them with a warning - so through that SDK you may get no logprobs even where OpenAI allows them.) The API reference's own example, verbatim:

```bash
# verbatim, https://developers.openai.com/api/reference/resources/chat (example "Logprobs")
curl https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "gpt-6-sol",
    "messages": [
      {
        "role": "user",
        "content": "Hello!"
      }
    ],
    "reasoning_effort": "none",
    "logprobs": true,
    "top_logprobs": 2
  }'
```

A classification request with the official Python SDK (`openai` 2.54.0), executed against a mock transport that returns the documented response shape:

```python
from openai import OpenAI

LABELS = ["billing", "technical", "sales"]
oai = OpenAI(api_key="test", http_client=httpx.Client(transport=httpx.MockTransport(openai_mock)))

completion = oai.chat.completions.create(
    model="gpt-6-sol",
    messages=[{"role": "user", "content": "Which team should handle this ticket? "
               "Answer with exactly one of: billing, technical, sales.\n\nI was charged twice."}],
    reasoning_effort="none",  # required for logprobs on GPT-6 Sol / Luna
    logprobs=True,
    top_logprobs=20,
    max_completion_tokens=1,
)
probs, covered = class_probs_from_logprobs(
    openai_chat_first_token(completion.model_dump()), LABELS)
```

Output: billing 0.825, technical 0.165, sales 0.010, `covered` = 0.973 - the total probability (sum of exp(logprob)) that the model put on the three exact label tokens at that position. The mock deliberately included the variants `bill` and `Billing`: they are **different tokens** and are not counted as `billing` by the default normaliser (`str.strip`). Either constrain the output so variants cannot occur (vLLM / llama.cpp below), or pass a normaliser such as `lambda t: t.strip().lower()` and decide consciously which variants count.

The helpers:

```python
def class_probs_from_logprobs(token_logprobs, labels, normalize_token=str.strip):
    """Class posterior from ONE generated position's top-logprob list.

    token_logprobs: iterable of (token_text, logprob). Tokens are matched to labels
    after normalize_token (default strips the leading space many tokenizers add);
    several surface forms of one label are summed. Returns (probs dict renormalized
    over the labels, raw mass the labels covered). Labels absent from the list get 0:
    the list is truncated at top-k, so a low `covered` means the posterior is unreliable."""
    mass = {lab: 0.0 for lab in labels}
    for tok, lp in token_logprobs:
        key = normalize_token(tok)
        if key in mass:
            mass[key] += math.exp(lp)
    covered = sum(mass.values())
    if covered == 0:
        return {lab: 1.0 / len(labels) for lab in labels}, 0.0
    return {lab: m / covered for lab, m in mass.items()}, covered

def openai_chat_first_token(resp):
    """OpenAI Chat Completions (and vLLM's OpenAI-compatible server) response dict ->
    [(token, logprob)] for the first generated token."""
    first = resp["choices"][0]["logprobs"]["content"][0]
    return [(t["token"], t["logprob"]) for t in first["top_logprobs"]]
```

**JSON output moves the label.** With a JSON schema, the first generated token is `{"` or similar, not the label; read the position where a label first appears:

```python
def openai_label_position(resp, labels, normalize_token=str.strip):
    """With JSON/structured output the label is NOT the first token (that is '{').
    Return [(token, logprob)] at the first position whose top list contains a label."""
    for pos in resp["choices"][0]["logprobs"]["content"]:
        tops = [(t["token"], t["logprob"]) for t in pos["top_logprobs"]]
        if any(normalize_token(tok) in labels for tok, _ in tops):
            return tops
    return []
```

**Single-token check.** The recipe below needs each label to be one token. For the `o200k_base` encoding in `tiktoken` 0.14.0 (executed):

```python
import tiktoken

enc = tiktoken.get_encoding("o200k_base")
for label in ["billing", "technical", "sales", "account_management", "Billing", " billing"]:
    print(f"{label!r:22} -> {len(enc.encode(label))} token(s)")
```

Output: `billing`, `technical`, `sales`, `Billing` and `" billing"` are 1 token each; `account_management` is 2. Which encoding GPT-6 models use was **not verified** (the OpenAI docs fetched for this page do not state it); check against the model's tokenizer, or simply inspect the returned `top_logprobs` - a multi-token label shows up as a fragment.

The request builder used by the tests:

```python
def openai_request(model, question, labels, text, top_logprobs=20):
    """Chat Completions body. On GPT-6 Sol/Luna logprobs need reasoning_effort 'none';
    GPT-6 Astra and GPT-6.1 Sol do not support 'none', so they return no logprobs."""
    return {
        "model": model,
        "messages": [{"role": "user", "content": _classify_prompt(question, labels, text)}],
        "reasoning_effort": "none",
        "logprobs": True,
        "top_logprobs": top_logprobs,  # 0-20
        "max_completion_tokens": 1,
    }
```

---

## Google Gemini

The API reference documents the fields (https://ai.google.dev/api/generate-content, `GenerationConfig`, fetched 2026-10-03), verbatim:

> `responseLogprobs` `boolean` Optional. If true, export the logprobs results in response.
>
> `logprobs` `integer` Optional. Only valid if [`responseLogprobs`]. This sets the number of top logprobs, including the chosen candidate, to return at each decoding step in the [`logprobsResult`]. The number must be in the range of [0, 20].

The response carries `candidates[].avgLogprobs` and `candidates[].logprobsResult`, whose `topCandidates[]` has one entry per decoding step ("Length = total number of decoding steps"), each a list of `candidates` "Sorted by log probability in descending order" with `token`, `tokenId` and `logProbability`.

For an enum answer: `responseMimeType` - "`text/x.enum`: ENUM as a string response in the response candidates" (same page). The reference marks `responseSchema` **deprecated** ("Deprecated. Use `responseFormat` instead"), and `responseFormat.text.mimeType` only lists `APPLICATION_JSON` and `TEXT_PLAIN`, so `responseFormat` has no enum MIME type. Note also that the current `responseSchema` description names only `application/json` as a compatible MIME type: the pairing `text/x.enum` + `responseSchema` with an `enum`, used by `gemini_request` below, is the long-standing enum recipe but is **not shown** in the reference or the structured-output guide fetched on 2026-10-03, and was not tested against the API here. Expect this to change.

**Availability.** The fields are documented, but on 2026-08-05 a Google staff member answered a developer-forum report that logprobs no longer work on Gemini 3.1 Pro and 3.6 Flash: "This is currently WAI, logprobs are no longer returned for 3.X models" (https://discuss.ai.google.dev/t/missing-logprobs-support-in-the-newest-gemini-models-3-1-pro-3-6-flash-on-vertex-ai-and-ai-studio/176557). Users report the error "Logprobs is not supported for this model" on 3.x models in Vertex as well. This is a forum statement, not documentation; check the model you use. In thread 176557 the original poster reports that Gemini 2.5 Flash still returned logprobs, but in thread 132426 (*Were Logprobs disabled for Gemini 3/3.1 in Vertex API?*, https://discuss.ai.google.dev/t/132426, fetched 2026-10-03) Vertex AI users report "Update: It just stopped working on 2.5 too!" (2026-03-16) and that logprobs "for 2.5 models seems to have been disabled today" (2026-04-30). Do not assume a 2.5 model returns them on Vertex AI either.

Request builder (executed by the tests; not sent):

```python
def gemini_request(question, labels, text, top=20):
    """generateContent body. text/x.enum + responseSchema enum constrains the answer;
    responseLogprobs/logprobs ask for the distribution (rejected by 3.x models)."""
    return {
        "contents": [{"role": "user", "parts": [{"text": _classify_prompt(question, labels, text)}]}],
        "generationConfig": {
            "responseMimeType": "text/x.enum",
            "responseSchema": {"type": "STRING", "enum": list(labels)},
            "responseLogprobs": True,
            "logprobs": top,  # 0-20
        },
    }

def gemini_first_token(resp):
    """Gemini generateContent response dict -> [(token, logProbability)] at step 0."""
    step = resp["candidates"][0]["logprobsResult"]["topCandidates"][0]
    return [(c["token"], c["logProbability"]) for c in step["candidates"]]
```

---

## vLLM

Verified against the v0.30.0 tag (released 2026-09-22).

**`logprobs`** (`vllm/sampling_params.py`, verbatim):

> Number of log probabilities to return per output token. When set to `None`, no probability is returned. If set to a non-`None` value, the result includes the log probabilities of the specified number of most likely tokens, as well as the chosen tokens. Note that the implementation follows the OpenAI API: The API will always return the log probability of the sampled token, so there may be up to `logprobs+1` elements in the response. When set to -1, return all `vocab_size` log probabilities.

**`max_logprobs`** (`vllm/config/model.py`): `max_logprobs: int = Field(default=20, ge=-1)` - "Maximum number of log probabilities to return when `logprobs` is specified in `SamplingParams`. The default value comes the default for the OpenAI Chat Completions API. -1 means no cap, i.e. all (output_length * vocab_size) logprobs are allowed to be returned and it may cause OOM."

**`logprobs_mode`** (same file), verbatim:

> `logprobs_mode: LogprobsMode = "raw_logprobs"` - Indicates the content returned in the logprobs and prompt_logprobs. Supported mode: 1) raw_logprobs, 2) processed_logprobs, 3) raw_logits, 4) processed_logits. Raw means the values before applying any logit processors, like bad words. Processed means the values after applying all processors, including temperature and top_k/top_p.

So by default you read the model's **unconstrained** next-token distribution, even when `structured_outputs` forces the output to be one of your labels. For single-token labels that is what you want: renormalise over the label tokens (the recipe below). Whether `processed_logprobs` includes the structured-output token mask is not stated in the docstring and was not tested.

**`logprob_token_ids`** (Chat Completions request, `vllm/entrypoints/openai/chat_completion/protocol.py`, v0.30.0), verbatim description:

> Specific vocab token IDs to return logprobs for at each generated position, in addition to the sampled token. More efficient than `top_logprobs=-1` when only a small fixed label set is needed (e.g. multilabel scoring where each label corresponds to a known vocab id). When set, this explicit token selection takes precedence over the natural top-k selected by `top_logprobs`. Requires `logprobs=True`.

That removes the top-20 truncation problem entirely: ask for exactly your label token ids.

**Constrained choice**, verbatim from the structured-outputs docs (`docs/features/structured_outputs.md`, v0.30.0):

```python
# verbatim, https://github.com/vllm-project/vllm/blob/v0.30.0/docs/features/structured_outputs.md
        extra_body={"structured_outputs": {"choice": ["positive", "negative"]}},
```

(the docs note that the old `guided_choice` field was removed in v0.12.0 and maps to `{"structured_outputs": {"choice": ...}}`). A full call through the OpenAI SDK, executed against a mock of the server:

```python
vllm = OpenAI(api_key="EMPTY", base_url="http://localhost:8000/v1",
              http_client=httpx.Client(transport=httpx.MockTransport(vllm_mock)))
completion = vllm.chat.completions.create(
    model="my-model",
    messages=[{"role": "user", "content": "Which team? Answer billing, technical or sales.\n\nRefund please."}],
    logprobs=True,
    top_logprobs=20,
    max_tokens=1,
    extra_body={"structured_outputs": {"choice": LABELS}},
)
probs, covered = class_probs_from_logprobs(openai_chat_first_token(completion.model_dump()), LABELS)
```

Output: billing 0.889, technical 0.091, sales 0.020, covered 0.998.

```python
def vllm_request(model, question, labels, text, top_logprobs=20, label_token_ids=None):
    """vLLM OpenAI-compatible body: `structured_outputs.choice` constrains the text;
    `logprob_token_ids` (v0.30) returns logprobs for exactly the label tokens."""
    body = {
        "model": model,
        "messages": [{"role": "user", "content": _classify_prompt(question, labels, text)}],
        "logprobs": True,
        "top_logprobs": top_logprobs,
        "max_tokens": 1,
        "structured_outputs": {"choice": list(labels)},
    }
    if label_token_ids:
        body["logprob_token_ids"] = list(label_token_ids)
    return body
```

---

## llama.cpp server

Verified against the `tools/server/README.md` and server source at commit 436f6f8 (2026-10-03).

Parameters, verbatim from the README:

> `n_probs`: If greater than 0, the response also contains the probabilities of top N tokens for each generated token given the sampling settings. Note that for temperature < 0 the tokens are sampled greedily but token probabilities are still being calculated via a simple softmax of the logits without considering any other sampler settings. Default: `0`
>
> `post_sampling_probs`: Returns the probabilities of top `n_probs` tokens after applying sampling chain.
>
> `grammar`: Set grammar for grammar-based sampling. Default: no grammar
>
> `json_schema`: Set a JSON schema for grammar-based sampling ...

Response: `completion_probabilities` is an array with one entry per generated token, each with `id`, `logprob`, `token`, `bytes` and `top_logprobs`; "if `post_sampling_probs` is set to `true`: `logprob` will be replaced with `prob`, with the value between 0.0 and 1.0; `top_logprobs` will be replaced with `top_probs` ... Number of elements in `top_probs` may be less than `n_probs`".

What the source does (`tools/server/server-context.cpp`, `populate_token_probs`):

- default (`post_sampling_probs` false): `get_token_probabilities(ctx_tgt, idx, n_probs_request)` - a softmax over the raw logits, **before** the sampler chain, so the grammar is **not** reflected;
- `post_sampling_probs` true: `common_sampler_get_candidates(...)` after the whole chain - the grammar **and** temperature, top-k, top-p, min-p and the rest - and candidates with probability 0.0 are dropped ("Filter 0.0 probailities").

For class probabilities use the **default** path and renormalise over the label tokens yourself; post-sampling values are distorted by temperature and truncation (the default sampler order is `["dry", "top_k", "typ_p", "top_p", "min_p", "xtc", "temperature"]`).

```python
def llamacpp_request(question, labels, text, n_probs=20):
    """llama.cpp /completion body: a GBNF grammar over the labels, raw (pre-sampling)
    probabilities for the first token. Renormalize over the labels yourself."""
    grammar = "root ::= " + " | ".join('"' + lab + '"' for lab in labels)
    return {
        "prompt": _classify_prompt(question, labels, text) + "\nAnswer: ",
        "n_predict": 1,
        "n_probs": n_probs,
        "grammar": grammar,
    }

def llamacpp_first_token(resp):
    """llama.cpp /completion response with n_probs > 0 -> [(token, logprob)].
    Handles both shapes: top_logprobs (default) and top_probs (post_sampling_probs)."""
    first = resp["completion_probabilities"][0]
    if "top_probs" in first:
        return [(t["token"], math.log(t["prob"]) if t["prob"] > 0 else -math.inf) for t in first["top_probs"]]
    return [(t["token"], t["logprob"]) for t in first["top_logprobs"]]
```

**A TypeSafe-compatible endpoint.** The same server now serves decision models directly, verbatim from the README:

> ### POST `/v1/systemone`: TypeSafe-compatible System One API
>
> Answers typed questions about a `state` with a decision model.
>
> Follows the [TypeSafe API](https://docs.typesafe.ai/api), streaming is not supported.

It answers `choice`, `score` and `noul` questions with `probabilities` / `noul`, like the hosted API; "The number of options of a `choice` question is limited by the model, for example: 52 for openjev, 255 for laya and clef"; "The questions of a request are answered independently, an answer does not depend on the other questions. The exception is clef: it reads all the questions in one prompt and decides them jointly." - which matters for `order-bias.md`'s single-request rotation: with clef, the rotated copies are **not** independent. And, verbatim: "The probabilities are scaled with the temperatures stored in the model file. They are not guaranteed to be calibrated for your data." Calibrate them like any other.

---

## The single-token-enum recipe

Why it works: if every label is exactly one token, the model's next-token distribution restricted to the label tokens and renormalised,

```
P(label_j | prompt) = p(t_j) / Σ_k p(t_k)
```

is exactly the class posterior **under the constraint** "the answer is one of the labels" - the same distribution a grammar that allows only those tokens would produce by renormalising. For multi-token labels this breaks: the first token is shared by several labels or splits one label, the constrained decoder renormalises at every step, and the product of per-step probabilities is not the probability the unconstrained model would assign to the full string.

Steps:

1. **Choose labels that are single tokens** in the target model's tokenizer, distinct after normalisation (no `Billing` vs `billing`), without a shared prefix. Check them (`tiktoken` above, or the tokenizer of your self-hosted model); in vLLM pass their ids as `logprob_token_ids`.
2. **Make the label the first generated token**: instruct "Answer with exactly one of: ...", end the prompt where the label begins (llama.cpp: `"\nAnswer: "`), set max tokens to 1. With JSON output, locate the label position instead (`openai_label_position`).
3. **Request the top logprobs** at that position (20 is the cap on OpenAI, Gemini and vLLM's default).
4. **Renormalise over the labels** (`class_probs_from_logprobs`) and **record `covered`**: the raw mass the labels received. A low value (say < 0.5) means the model wanted to say something else - a refusal, an explanation, a label variant - and the renormalised distribution is unreliable for that item. Route it like an out-of-distribution item.
5. **Mind the leading space**: many BPE tokenizers have separate tokens for `billing` and `" billing"`; the default normaliser strips whitespace so both count. That is right only if both cannot be different labels.
6. **Calibrate on your labels** (`calibration.md`): logprob-derived probabilities of instruction-tuned models are typically over-confident. The skill cites the GPT-4 Technical Report: MMLU ECE 0.007 for the pre-trained model vs 0.074 after post-training (Fig. 8) - not re-read for this page.
7. **Remove order bias** if labels are presented as a list (`order-bias.md`): letter-ID selection bias is exactly what Zheng et al. measured on logprob-based MCQ scoring.

---

## Without logprobs: sampling frequency, verbalized confidence, panels

For Claude (and Gemini 3.x, and GPT-6 Astra / GPT-6.1 Sol), the options all cost more than one call per item:

| Approach | Calls per item | Resolution | Known calibration behaviour |
|---|---|---|---|
| **Sampling frequency**: ask k times, count labels | k | ~1/k (10 samples cannot tell 0.93 from 0.97) | consistency-based scores are among the black-box methods Xiong et al. evaluate; calibrate on labels |
| **Verbalized confidence**: ask for the label and a number | 1 | whatever the model writes | Tian et al. (EMNLP 2023): for RLHF-LMs "such as ChatGPT, GPT-4, and Claude", verbalized confidences "are typically better-calibrated than the model's conditional probabilities", "often reducing the expected calibration error by a relative 50%". Xiong et al. (ICLR 2024): "LLMs, when verbalizing their confidence, tend to be overconfident" |
| **Panel of prompts / models voting** | panel size | ~1/panel size | reduces prompt-specific variance; same calibration need |

Sampling with a Dirichlet (Jeffreys) pseudocount so unseen labels are not exact zeros - which would break the log-based fits in `calibration.md` for the same reason the rounding floor does:

```python
def sampling_frequency(samples, labels, pseudocount=0.5):
    """Label frequencies over k independent samples, Dirichlet-smoothed (Jeffreys
    0.5 by default) so an unseen label is not an exact zero. With k samples the
    resolution is ~1/k: 10 samples cannot tell 0.93 from 0.97."""
    counts = Counter(samples)
    total = len(samples) + pseudocount * len(labels)
    return {lab: (counts.get(lab, 0) + pseudocount) / total for lab in labels}
```

```python
def frequency_distribution(ask, prompt, k=10):
    """k independent calls to a typed classifier with no logprobs (e.g. Claude with a
    JSON-schema enum). Cost: k calls. Resolution: about 1/k."""
    return sampling_frequency([ask(prompt) for _ in range(k)], LABELS)


dist = frequency_distribution(ask_claude, "Which team should handle: 'I was charged twice'?", k=10)
```

Executed with a fake classifier that answers billing / technical / sales with probabilities 0.7 / 0.25 / 0.05: 10 samples gave 0.565 / 0.391 / 0.043 - a reminder of how coarse k = 10 is. On recent Claude models `temperature` is fixed at 1.0 (see above), so repeated calls do sample; the ten calls can run concurrently.

Parsing a verbalized number:

```python
def parse_verbalized_confidence(text):
    """Pull a 0-1 or 0-100% number out of a 'Confidence: 85%' style answer; None if absent."""
    import re

    m = re.search(r"confidence\s*[:=]\s*([0-9]*\.?[0-9]+)\s*(%?)", text, re.I)
    if not m:
        return None
    v = float(m.group(1))
    if m.group(2) == "%" or v > 1.0:
        v /= 100.0
    return min(max(v, 0.0), 1.0)
```

Whatever the source - frequency, verbalized number, panel vote share - it is a score, not yet a probability. Fit a Platt or temperature map on labelled items and check it with the noise floor and a held-out split, exactly as for logprobs.

---

## Checklist

- [ ] Provider and exact model checked against the matrix; for GPT-6 the reasoning effort that the model allows; for Gemini whether the model still returns logprobs
- [ ] Labels are single tokens, distinct after normalisation; ids passed via `logprob_token_ids` on vLLM
- [ ] Label is the first generated token (or its position located); max tokens 1
- [ ] Probabilities renormalised over labels; `covered` logged and low-coverage items routed out
- [ ] For self-hosted: raw (pre-sampling) probabilities read, not post-sampling ones
- [ ] No logprobs: sampling frequency with k ≥ 10, or verbalized confidence, then calibrated
- [ ] Order bias handled; calibration map fitted per question and model version

---

## Sources

- Anthropic: OpenAI SDK compatibility https://platform.claude.com/docs/en/api/openai-sdk; Messages API https://platform.claude.com/docs/en/api/messages/create; Structured outputs https://platform.claude.com/docs/en/build-with-claude/structured-outputs (all fetched 2026-10-03)
- OpenAI: Chat reference https://developers.openai.com/api/reference/resources/chat; Responses create https://developers.openai.com/api/reference/resources/responses/methods/create.md; latest-model guide https://developers.openai.com/api/docs/guides/latest-model.md (fetched 2026-10-03)
- Vercel AI SDK OpenAI provider page https://ai-sdk.dev/providers/ai-sdk-providers/openai.md (third-party; fetched 2026-10-03)
- Google: https://ai.google.dev/api/generate-content (fetched 2026-10-03); developer forum thread 176557 (staff reply 2026-08-05) and thread 132426 (user reports from 2026-03-14)
- vLLM v0.30.0: `vllm/sampling_params.py`, `vllm/config/model.py`, `vllm/entrypoints/openai/chat_completion/protocol.py`, `docs/features/structured_outputs.md` (source at tag v0.30.0)
- llama.cpp at 436f6f89e1e581249900b37a5b8a12a36a6d0912: `tools/server/README.md`, `tools/server/server-context.cpp`, `tools/server/server-task.cpp`
- Tian et al., *Just Ask for Calibration*, EMNLP 2023 - arXiv 2305.14975
- Xiong et al., *Can LLMs Express Their Uncertainty?*, ICLR 2024 - arXiv 2306.13063
- Zheng et al., *Large Language Models Are Not Robust Multiple Choice Selectors*, ICLR 2024 - arXiv 2309.03882
