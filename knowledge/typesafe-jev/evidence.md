# TypeSafe Jev - Evidence Ledger

> Official Documentation: https://docs.typesafe.ai/model-jaggedness/jev-1.13
> Official Documentation: https://docs.typesafe.ai/models
> Vendor evaluations: https://evals.typesafe.ai
> Launch post: https://typesafe.ai/blog/introducing-system-one-models-and-jev
> Last verified: 2026-10-03
> Model version this ledger is about: `jev-1.13.0` (the only current model; the aliases `jev-latest` and `jev-preview` both point to it, per https://docs.typesafe.ai/models fetched 2026-10-03). OpenRouter resolves the same model as `jev-1.13-20260917` (arXiv 2609.24574, Table 5).
> Code blocks in this document: two, both executed on CPython 3.12.10 with the standard library only. No TypeSafe SDK and no API call is involved.

## Overview

Jev, TypeSafe AI's first "System One" model, launched on 2026-09-15. It returns typed decisions (Choice, Score, Noul) with probabilities and does not generate text. In its first eighteen days the vendor's claims drew an unusually large body of independent testing: pre-registered audits, arXiv preprints, GitHub repositories with committed raw responses, security write-ups, and aggregator reviews.

This ledger re-reads each of those studies **at its primary source**, as of 2026-10-03. For each one it records the method, n, model version, metrics with their intervals, the caveats the authors state, the dates, and any corrections the authors published. Two published numbers were also recomputed offline (see [Re-checks run for this ledger](#re-checks-run-for-this-ledger)).

What this document is for:

- quoting a Jev number without overstating it;
- knowing which of the vendor's claims hold, which are contested, and which nobody has measured;
- knowing what the vendor's own limitations page said and when it changed.

The how-to material is in the dev-suite skills `typesafe-jev`, `typed-decision-models` and `decision-model-calibration`. This page does not repeat it. It is the evidence layer underneath them.

**Conventions.**

- Every number carries its source and the date it was read.
- All sources were read on 2026-10-03 unless another date is given.
- "Reported" means the author's statement, not recomputed here.
- "Secondary" means seen only in a review or aggregator, not at its primary source. Such items are marked **(secondary only)**.
- Percentages are as the source printed them.

---

## Table of Contents

1. [Evidence grades used here](#evidence-grades-used-here)
2. [Study table](#study-table)
3. [Vendor claim vs finding](#vendor-claim-vs-finding)
4. [Study-by-study notes](#study-by-study-notes)
5. [Prompt injection evidence](#prompt-injection-evidence)
6. [Re-checks run for this ledger](#re-checks-run-for-this-ledger)
7. [Well-established, contested, unknown](#well-established-contested-unknown)
8. [History of the vendor's jaggedness and models pages](#history-of-the-vendors-jaggedness-and-models-pages)
9. [Newer than 2026-09-28](#newer-than-2026-09-28)
10. [Where secondary sources diverge from primary ones](#where-secondary-sources-diverge-from-primary-ones)
11. [Sources](#sources)

---

## Evidence grades used here

| Grade | Meaning |
|---|---|
| **P-raw** | Primary source with committed per-item outputs or raw responses: the number can be recomputed by anyone. |
| **P** | Primary source that states its method and numbers; no per-item data (or not checked here). |
| **V** | Vendor-produced (TypeSafe). Primary for what the vendor claims, not independent evidence. |
| **S** | Secondary: a review, aggregator or news piece. Used only to locate primary sources, or flagged as secondary only. |

A study's grade says how checkable it is, not whether it is right.

---

## Study table

Every row was opened at its source on 2026-10-03. "Version" is the model version the study states.

| # | Study (primary URL) | Date(s) | Grade | Version | Task, n | Headline result (as published) |
|---|---|---|---|---|---|---|
| 1 | TypeSafe workflow evals, https://evals.typesafe.ai; code https://github.com/typesafe-ai/WorkflowEvals | Site undated; code repo created 2026-09-28 | V | `jev-1.13.0` (repo default) | 4 in-house workflows; cases 240 / 111 / 150 / 204 = 705 (repo README) | Jev 67.8% **agreement** with a GPT-6 Astra + Claude Fable 5.1 "high thinking" consensus. Sol 74.1%, Opus 5 73.1%, Sonnet 5 67.8% |
| 2 | Launch post, https://typesafe.ai/blog/introducing-system-one-models-and-jev | 2026-09-15; unchanged vs. a copy taken earlier on 2026-10-03 | V | "Jev" | — | "two orders of magnitude faster"; "can't hallucinate"; "0%" hallucination bar footnoted *"Our number is not empirical"* |
| 3 | sanand0 BANKING77 pilot, https://sanand0.github.io/llmevals/jev/ (repo `sanand0/llmevals/jev`) | 2026-09-18; updated 2026-09-28 (Luna added) | P-raw | `~typesafe/jev-latest` via OpenRouter | BANKING77, 77 cases (1 per intent), 10 models | Jev 58/77 = 75.3% [64.6, 83.6]: last of 10. ECE 0.138, AUROC 0.811 |
| 4 | simonmesmith/jev-banking77-experiment | 2026-09-18 | P-raw | `jev-1.13.0` | BANKING77 full test, n = 3,080, with 24 BM25-retrieved labelled examples per request | 92.40% [91.41, 93.29] vs. 93.66% published fine-tuned BERT |
| 5 | ickma2311/jev-baselines-eval (study 1) | 2026-09-18, three errata the same day | P-raw | via Vercel AI Gateway `typesafe-ai/jev` | Banking77 n = 208 paired; CLINC150 n = 200 | B0: Jev 0.832 vs. supervised encoder 0.933. B1: Jev 0.870, nano 0.795, Terra 0.915. Both pre-registered verdicts **AMBIGUOUS** |
| 6 | ickma2311/jev-baselines-eval `injection/` (study 2) | 2026-09-22 (pushed 2026-09-25) | P-raw | `jev-1.13.0` | 1,200 decisions per system, authority-framed injection | Confidence ranks hijacked decisions **above** resisted ones: AUROC 0.261 [0.190, 0.336] |
| 7 | anisselbd/jev-phishing-bench | 2026-09-16 (pass 1) / 2026-09-17 (controls) | P (raw emails not committed) | `jev-1.13.0` | PhishNChips v5.2, 2,000 emails, vs. Claude Haiku 4.5 | Jev 62.6% [60.5, 64.7] vs. Haiku 81.3% [79.5, 82.9]. Five Jev signal Nouls plus logistic regression: 95.0% [93.5, 96.2] on a held-out half |
| 8 | SamuelSacco/jev-exploration (ledger + experiments) | 2026-09-16 → 2026-10-01 | P-raw | `jev-1.13.0` | 800-item synthetic difficulty gradient, probes, re-analysis of others | ECE 2.1–2.5× its noise floor in every tier; "compression toward the middle"; 0.01 quantisation |
| 9 | scienthoon/jev-ood-calibration | Run 2026-09-19; **correction 2026-09-22** | P-raw | Gateway `typesafe-ai/jev` (no version exposed) | 3 public sets (3,721 items) + 900 synthetic tickets | Public sets: ECE 0.024–0.032. Synthetic: ECE 0.107 (4.4× floor); unknowable-rule Score at 44.7% accuracy, mean stated probability 0.74 |
| 10 | AHTOOOXA/jev-cyrillic-audit | Study 1 on 2026-09-20; Studies 2–3 on 2026-09-21; reinterpretation 2026-09-26 | P-raw | `jev-1.13.0` | RU vs. EN, n = 600 paired per dataset; then 15 languages | XNLI 88.3% → 77.3% (Δ −11.0 pp [−14.2, −7.8]), ECE 0.032 → 0.096. MASSIVE: no detectable difference |
| 11 | marcosmartinez/jev-acento | 2026-09-21 | P-raw | `jev-1.13.0` (pinned) | ES vs. EN, 3,200 paired items, 19,200 calls | −3.0 to −6.4 pp on all four datasets; Spanish instructions do not help |
| 12 | Archer Hume, https://archerhume.com/posts/jevs-architecture-unmasked | 2026-09-17 | P (evidence bundle linked) | `jev-1.13.0` | Black-box probes ("10,000 API calls") | Option position: correct-answer probability ≈0.50 with the card first vs. ≈0.88 last. Adding an irrelevant option lowers log-odds by −0.28 [−0.36, −0.19] |
| 13 | arXiv 2609.24574v2 (Ibrahim & Zaki), https://arxiv.org/abs/2609.24574 | v1 2026-09-21, v2 2026-09-23; runs 2026-09-20 | P-raw (records released) | `jev-1.13-20260917` (OpenRouter) | 18 social-science tasks, 7,977 items, vs. 19 LLMs; pre-registered (AsPredicted #312,511) | Trails the per-task best LLM on 14/15 tasks, median −11.6 macro-F1, at median 44× lower cost. Cascades match the LLM at ¼–½ of its cost |
| 14 | arXiv 2609.26758v2 (Sun, Xu, Shi, Yang), https://arxiv.org/abs/2609.26758 | v1 2026-09-22, v2 2026-09-23 | P | hosted "Jev" (version not named) | 1,200 binary workflow decisions; option-name reassignment | Hosted Jev: yes/no swap flips 32.5% of decisions (24× its retest floor); AUC 0.815 → 0.581 |
| 15 | xbill9 review, https://dev.to/gde/jev-after-eight-days-of-independent-tests-level-with-mid-price-llms-behind-the-frontier-1kln; full report in `xbill9/gemma4-dev/jev/reports` | Article 2026-09-24; report last changed 2026-09-24 | S (aggregator with its own re-scoring) | `jev-1.13.0` per the studies it read | 14 preprints, 104 repos, 33 posts read as of 2026-09-23 | "Level with mid-price LLMs, 6.5 to 11.5 points behind the frontier"; eight BANKING77 runs 0.753–0.840 (median 0.809) |
| 16 | Check Point Research, https://blog.checkpoint.com/ai-security/jev-is-not-a-language-model-but-it-breaks-like-one-prompt-injection-against-a-typed-decision-model/ | 2026-09-24 | P (no raw data) | not stated | 1 app, 3 attackers × 3 difficulty levels × 3 delivery modes × 3 runs (27 runs per attacker), 10 turns | Every configuration broken at least once; strongest attacker 25/27 at ≈$0.50 per break |
| 17 | VentureBeat (Louis Columbus), https://venturebeat.com/security/companies-are-putting-jev-in-charge-of-ai-agent-decisions-and-prompt-injection-can-influence-the-verdict | 2026-09-21 | S | not stated | One Octomind integration test | `rm -rf ~/.ssh`: block probability 0.76 → 0.48 after a fake "pre-approved" tool-output field |
| 18 | Hacker News thread 49717558, https://news.ycombinator.com/item?id=49717558 | 2026-09-15 | V (CEO comments) | — | 1,989 points at fetch time | CEO (`CompleteSkeptic`): architecture "close to the chest for now"; agrees Jev is a zero-shot classifier |
| 19 | AbdelStark/awesome-typesafe-jev | Last push 2026-10-02 | S | — | Curated index | Used only to locate sources; no number taken from it |

---

## Vendor claim vs finding

"Vendor source" is where TypeSafe makes the claim. "Finding" cites the independent primary source.

| Vendor claim | Vendor source | Finding | Status |
|---|---|---|---|
| "Jev achieves similar levels of intelligence on System One tasks compared to existing LLMs" | Launch post, 2026-09-15 | **Vendor's own eval:** 67.8% agreement, tied with Sonnet 5 (67.8%), below Sol (74.1%) and Opus 5 (73.1%). **Independent:** trails the per-task best LLM on 14/15 tasks, median −11.6 macro-F1 (#13). Last of 10 on a 77-case BANKING77 pilot (#3). −4.5 pp vs. GPT-5.6 Terra on CLINC150 (#5). Within its price band, Gemma 4 31B (0.611) and Qwen3 235B (0.604) score above Jev's median macro-F1 of 0.581 (#13) | **Overstated against the frontier.** Mid-tier accuracy is the consistent finding |
| 67.8% accuracy | evals.typesafe.ai | It is **agreement** with *"an average of the responses of GPT-6 Astra and Claude Fable 5.1, both at high thinking"*; there is no human ground truth. The launch post itself says this *"biases answers towards OpenAI and Anthropic's models"* | Accurate as agreement; not a correctness figure |
| "193.6x faster, 444.6x cheaper" | Homepage; launch post: *"we expect that these are on the higher end of real world gains"* | The comparator is not named. From the published overview points, 444.6× is consistent with Opus 5 (440× from rounded figures). The time multiplier nearest 193.6× is Sonnet 5 (195×). Averaged over all eight LLM workflow setups: 149.2× cheaper, 97.8× faster (recomputed below). Independent client-side ratios: 2.2× (#5), 2.9× (#7) | **Best case of the vendor's own grid**, not typical |
| "End-to-end response time is 70ms-500ms" | Launch post | Median client call 0.39 s (#3), 0.42–0.44 s (#5), p50 239 ms from France with a 163 ms network floor (#7). ≈320–340 ms p50 (#10). Server time "about 0.1 s" **(secondary only:** xbill9) | **Holds** as a range for single calls. Throughput is rate-limited (see below) |
| "40x-200x faster for the same levels of frontier intelligence" | Launch post | Against a nano-class LLM on the same 30 items: 0.423 s vs. 0.924 s = 2.18× (#5). The author frames this as two serving paths, not two models | **Not reproduced** outside the vendor's grid |
| Output tokens free; input $0.042 / MTok | Docs `/models`; launch post | Billing on the TypeSafe dashboard matched input-only pricing: 4,561,792 tokens (3.66M input, 0.90M output) billed $0.15 (#7). Measured $0.027 per 1,000 annotation items; $0.21 for the whole 7,977-item grid (#13). But GPT-6 Luna was cheaper than Jev on the BANKING77 pilot ($0.06 vs. $0.07 per 1,000, #3) | **Holds**. Price leadership is not universal |
| "can't hallucinate"; 0% hallucination bar | Launch post | Footnote, verbatim: *"Our number is not empirical. Schema matching is guaranteed, thus we can confidently add 0% into the plots."* Schema compliance is real: 0 invalid answers in 23,703 decision-model calls (#13) and 0 in 4,621 (#9). Wrong-but-valid answers are 1 − accuracy, e.g. 37.4% on phishing (#7) | **Schema claim holds. "Can't hallucinate" is definitional, not measured** |
| "Calibrated: higher confidence means higher accuracy" | Launch post | **Ranking:** yes, mostly: AUROC 0.811 (#3), 0.853 / 0.734 (#5). **Calibration:** good on familiar English sets (ECE 0.024–0.032 #9; 0.031 on MMLU #12; 0.032 EN XNLI #10). Clearly off elsewhere: 0.107 (#9), 0.138 (#3), 0.154 (#7), 0.157 median across social-science tasks (#13). The **direction** varies by task and primitive (#8, #9) | **Contested.** True in-domain; not a guarantee off-domain |
| "Always communicates confidence and uncertainty" | Launch post | Confidence does not drop where the answer is unknowable. Score at 44.7% accuracy carried a mean probability of 0.74 (#9). Empathy task: 78% of items at ≥0.9 confidence, accuracy 0.383 vs. a 0.371 base rate (#13). With no answer-relevant information, up to 0.80 on a salient option (arXiv 2610.01006, abstract only) | **Refuted for missing knowledge** |
| "More consistent: returns similar answers for similar inputs" | Launch post | Repeat-call flip rates 0.5–2.2% (#10), at most 1.33% (#14), 2.2% (#7). The vendor's own self-consistency cookbook: *"TypeSafe flips on 2 of the 8 questions."* Semantically irrelevant changes move answers: option names (#14), option position (#12), short natural context additions (61.4% targeted flips, arXiv 2609.30243, abstract only) | **Repeatable, not invariant** |
| Cardinality up to 255 | Launch post | 255-option cap observed, enforced at request validation (#12, #8) | **Holds** |
| Workflows *"were not deliberately chosen nor constructed to make our model look good, and are not in our training distribution"* | Launch post | Unverifiable from outside. The post itself adds *"they were made by individuals on our model capabilities team, so some bias could exist."* | **Unknown** |
| "We make all the data ourselves" (training data) | Launch post FAQ | Contamination cannot be checked. Accuracy on public benchmarks is "far above" what 3–8B models score, which the author reads as consistent with training exposure (#9, author's inference). The 37-dataset study rules out "shallow memorization but not memorized question-answer pairs" (arXiv 2609.37647, abstract only) | **Unknown** |
| Rate limits 100K tok/s, **80** req/s, adjusting dynamically | Docs `/models` (live 2026-10-03; the docs dump still says 40) | Gateway free tier: about 1 req/min after a burst (#5); "a handful of requests per minute" (#9). Not tested on paid direct accounts | Vendor figure; unverified independently |

---

## Study-by-study notes

Section numbers match the `#` column of the study table, so "(#9)" anywhere in this ledger means section 9 below. Studies 6 (ickma2311 injection), 16 (Check Point) and 17 (VentureBeat) are covered in [Prompt injection evidence](#prompt-injection-evidence).

### 1. TypeSafe workflow evals (evals.typesafe.ai) — grade V

**Method, verbatim from the site:**

- *"we assume that the code is correct, and measure against the current smartest large models. For this eval, the reference labels are generated via an average of the responses of GPT-6 Astra and Claude Fable 5.1, both at high thinking, answering every question in the harness. All other models are evaluated using the provider's default reasoning settings."*
- *"Each point averages one model configuration's accuracy, cost and time over the four workflows with equal weight, against the consensus labels."*

**Per-workflow Jev agreement** (site, fetched 2026-10-03):

| Workflow | Jev | Best on that page |
|---|---|---|
| Security Incidents | 61.7% | Opus 5, 66.2% |
| Agent Trace Observability | 71.6% | Sol, 76.6% |
| Invoice Processing | 61.8% | Sol, 79.1% |
| Customer Service | 76.0% | Sol, 78.3% |

**n.** The site states no case counts, run counts, variance or intervals. The published code repo (`typesafe-ai/WorkflowEvals`, created 2026-09-28, two weeks after launch) lists 240 / 111 / 150 / 204 cases. Its scoring metrics are `exact_actions` and `primary_action`.

**Data.** The datasets download from a Hugging Face collection and *"Runs load the latest `main` revision by default"*. A re-run may therefore not use the data behind the published plot unless `--dataset-revision` is pinned.

**What the vendor discloses itself (launch post):**

- reference bias toward OpenAI and Anthropic;
- workflows built by its own capabilities team;
- the LLM comparators ran through *"our System One LLM wrapper"*, which *"tends to be slower and more expensive"*;
- timings run *"from our laptops on the West Coast"*.

**Reading.** On the vendor's own grid, Jev's agreement equals Sonnet 5's at about 1/290th of the cost per case. It is not the most accurate configuration on any of the four workflows.

### 2. Launch post — grade V

- **Date.** 2026-09-15, by founder Diogo Almeida. The page text was identical between a copy taken earlier on 2026-10-03 and a fresh fetch later that day. Its earlier history was not checked.
- **"Can't hallucinate".** The post says Jev *"can't hallucinate"* and later footnotes the 0% bar as *"not empirical"*.
- **Benchmarks.** FAQ: *"We deliberately chose not to publish performance against public benchmarks."*
- **Doom demo.** *"10 queries a second (which ends up costing ~$7/hour)"*. This is a demo cost; no study re-measured it.

### 3. sanand0 — BANKING77 pilot — grade P-raw

- **Design.** 77 cases, one frozen case per intent (seed `20260918`). Ten models; exact-match accuracy, no LLM judge.
- **Access.** Jev was called through **OpenRouter's Decisions endpoint** (`~typesafe/jev-latest`). Chat models were JSON-prompted with verbalized 0–100 confidence.
- **Updated.** The page reads "updated 28 September 2026". The repo commit of 2026-09-28 is "Add GPT-6 Luna to Jev benchmark". `summary.json` was unchanged between the earlier copy and the repo on 2026-10-03.

**Results** (`summary.json`):

| Model | Accuracy | Notes |
|---|---|---|
| Jev | 58/77 = 75.3%, 95% CI [0.646, 0.836] | mean confidence 0.874; ECE (10 bins) 0.138; AUROC 0.811; Brier 0.171; median latency 0.387 s; $0.0717 per 1,000 |
| GPT-6 Astra | 68/77 = 88.3% | ECE 0.057 |
| Gemini 3.8 Flash | 87.0% | |
| GPT-6 Luna | 77.9% | $0.056 per 1,000, cheaper than Jev |

**Author caveats (verbatim):**

- *"That two-case quality gap is too small to generalize from"* (Luna vs. Jev);
- *"77 cases is too small to certify production thresholds"*;
- the risk/coverage statistic *"is exploratory because its threshold is chosen and evaluated on the same 77 cases"*;
- *"6 of 77 requests were missed by all 10 models."*

**Reading.** 75.3% is the *lowest* of the BANKING77 Jev runs: one item per intent, no label descriptions. Do not quote it as "Jev's BANKING77 accuracy".

### 4. simonmesmith — BANKING77 full test set — grade P-raw

- **Design.** Protocol written before the first development calls, with a $5 budget. Five input setups were screened on 154 held-out training messages. The two strongest were then compared on a separate 616: 24 examples scored 89.12% and 48 examples 88.80%. The 24-example setup won by two messages and was frozen: category definitions plus 24 BM25-retrieved labelled training examples. It was then run on all 3,080 test messages.

**Results:**

| Measure | Value |
|---|---|
| Accuracy | 2,846/3,080 = **92.40%**, Wilson [91.41, 93.29] |
| Macro-F1 | 0.9235 |
| Final-test cost | $0.4375 |
| Elapsed | 411.2 s with 4 workers |
| Mean / p95 per message | 0.533 s / 0.713 s |

**Screening on 154 items:**

| Setup | Accuracy |
|---|---|
| No labelled examples | 79.22% |
| 24 relevant examples | 90.26% |

**Caveats the author states:**

- *"This was not zero-shot classification."* Retrieval is part of the system, and a retrieval-only classifier was not measured, so Jev's contribution beyond retrieval is not quantified.
- *"Prior benchmark exposure is unknown."*
- 25 test messages also appear in training; excluding them gives 92.34%.
- The BERT figure is a published 2020 number, not re-run.
- The code and README were written by GPT-6 Astra Light, as the README states.

**Reading.** With retrieved examples, Jev comes within 1.26 pp of a fine-tuned BERT. With definitions only, it scored 79.22% on the screening set.

### 5. ickma2311 — pre-registered baselines (study 1) — grade P-raw

- **Date and access.** 2026-09-18. Jev via the Vercel AI Gateway (`POST /v4/ai/evaluation-model`).
- **Comparators.** LLMs were JSON-prompted and regex-parsed, with verbalized confidence (no logprobs available).
- **Erratum.** Three rounds the same day, after external review by GPT-6 Astra via Codex. The first round found that the headline cascade result **held only at the 1 pp margin and flipped sign at exact parity**. The second round corrected round 1's own explanation of that flip. The third reframed the latency comparison.

**Results:**

- **B0, Banking77 (n = 208 paired, reduced from 300 by rate limits):**

  | Method | Accuracy |
  |---|---|
  | Jev | 0.832 [0.779, 0.880] |
  | gpt-5.4-nano | 0.793 |
  | GPT-5.6 Terra | 0.875 |
  | Frozen `bge-small` + logistic regression (10,003 labels, 9 ms) | **0.933** |

  Paired, encoder − Jev = +10.1 pp [+5.3, +15.4].

- **B1, CLINC150 zero-shot (n = 200):**

  | Method | Accuracy | AUROC |
  |---|---|---|
  | Jev | 0.870 | 0.734 |
  | nano | 0.795 | 0.816 |
  | Terra | 0.915 | 0.670 |

  Jev − nano = +7.5 pp [+3.0, +12.5].

- **Cascade.**
  - At the pre-registered target (Terra − 1 pp), escalation is 0.220 for Jev and 0.485 for nano: Δ = +0.265 [−0.530, +0.595], **AMBIGUOUS**.
  - At exact parity the sign flips: *"Jev returns confidence exactly 1.0 on 102 of 200 items, 6 of which are wrong."*
- **Latency.** Median 0.423 s (Jev, Vercel) vs. 0.924 s (nano, Lightning), a 2.18× ratio, described as *"a comparison of two client-and-service configurations"*.
- **Throughput.** On the Vercel free tier, about 1 request/minute after a burst. The 200-item run took about 3.5 hours.
- **Deviation the author discloses.** The retained 208 B0 items are not representative: Terra scored 0.875 on them vs. 0.804 on the 92 omitted.

### 7. anisselbd — phishing benchmark — grade P

- **Design.** Pass 1 results were committed 2026-09-16 (22:37 UTC). The controls and the 12-hour re-pass followed on 2026-09-17. PhishNChips v5.2: 1,000 phishing and 1,000 legitimate emails, with **LLM-generated bodies**. Ground truth comes from URL-reputation feeds.
- **Request.** One request per email with nine questions: a verdict Choice, a mirror Noul, five signal Nouls, and two alternative verdict wordings.
- **Baseline.** Claude Haiku 4.5, no thinking, temperature 0.

| | Jev (`jev-1.13.0`) | Claude Haiku 4.5 |
|---|---|---|
| Accuracy | 62.6% [60.5, 64.7] | 81.3% [79.5, 82.9] |
| AUROC | 0.689 | 0.837 |
| ECE (10 bins) | 0.154 | 0.097 |
| p50 latency from France | 239 ms (network floor 163 ms) | 687 ms (floor 18 ms) |
| Cost per 1,000 emails, list price | $0.038 | $0.462 |
| Repeat-pass stability | 2.2% label flips; 1.0% on 200 emails 12 h later | 0.7% flips on 300 |

**Controls added on 2026-09-17, after the result was challenged.** Half A chose signals and weights; half B reports:

| Method (half B) | Accuracy | Comparison |
|---|---|---|
| Two-line regex/list rule | 91.8% | — |
| Jev best single signal | 89.4% | below the regex, p = 0.0032 |
| Haiku best single signal (same questions) | 94.2% | above Jev's single signal, p < 0.0001 |
| Jev five-signal logistic regression | **95.0%** [93.5, 96.2] | beats the regex |
| Haiku five-signal regression | 93.2% | tied with Jev: p = 0.063, and Haiku's AUROC is higher (0.991 vs. 0.982) |

**Other observations:**

- **Verdict wording.** Across the four verdict wordings, accuracy ranged 58.4–63.2%.
- **Signal design.** The author states the signal questions *"were written after reading [the dataset's] URL-evasion taxonomy"*.
- **Data.** Raw files contain full emails and are not committed. Report and metrics are.

### 8. SamuelSacco — evidence ledger and experiments — grade P-raw

- **Scope.** Maintained 2026-09-16 → 2026-10-01. The final-verdicts file is dated 2026-10-01, on `jev-1.13.0`.
- **Calibration.** Calibration figures in this repo are **Noul only**.

**Findings stated:**

- **Gradient.** ECE runs 2.1–2.5× the noise floor across four difficulty tiers (800 synthetic items).
- **Shape.** The distortion is *"compression toward the middle"*.
- **Thresholding.** p ≥ 0.9 gave a 1.000 hit rate in every tier at 21.5–32.5% coverage. The same rule gave 73.9% on the phishing data.
- **Label budget.** Transferring the Platt slope and refitting the intercept on 50 labels cut held-out ECE by 62% (61.7%). Below 30 labels, the correction *"can leave calibration four times worse than doing nothing."* The ledger also tested cross-domain slope transfer (issue #9): 0 of 3 new-domain slopes fell in the (2.11, 2.61) email band, so transfer is "PARTIAL, not established". Transfer across a retrain is untested.
- **Quantisation.** Every value in 281 committed responses is on a 0.01 grid (15,363 values). Exact endpoints differ by primitive:

  | Field | n | Exactly 0 | Exactly 1 |
  |---|---|---|---|
  | `choice.probabilities` | 240 | 70.4% | 20.4% |
  | `score.probabilities` | 180 | 47.2% | 16.7% |
  | `noul.noul` | 11,148 | 0% | 0% (range 0.01–0.99) |

- **Question isolation.** A code word in a sibling question's instructions scored 0.02, against 0.99 when placed in the shared state.
- **Verdicts.**
  - "Can't hallucinate": **Refuted**.
  - "193.6× / 444.6×": **Overstated**.
  - "Probabilities are calibrated": **Refuted at scale**.
  - "Choice and independent Nouls measure the same thing": **Contested, n = 1**.

**Stale row.** The verdict *"No published rate limits, SLA, or uptime commitment — Verified (absent)"* no longer matches the docs. On 2026-10-03, `/models` publishes 100K tok/s and 80 req/s (40 in the earlier snapshot). There is still no SLA.

### 9. scienthoon — OOD calibration, with correction — grade P-raw

- **Run.** 2026-09-19, through the Vercel AI Gateway with `zeroDataRetention: true`. The Gateway exposes no model version.
- **Calls.** 4,621, with 0 failed and 0 invalid.

**Public sets** (author: "likely in Jev's training mix"):

| Set | n | Accuracy | ECE |
|---|---|---|---|
| OpenBookQA | 500 | 94.2% | 0.024 |
| CommonsenseQA | 1,221 | 88.1% | 0.032 |
| HellaSwag | 2,000 | 86.1% | 0.029 |

**Synthetic tickets** (generated 2026-09-19; 5% of labels deliberately corrupted):

| Question | Accuracy | ECE |
|---|---|---|
| queue (Choice) | 89.0% | 0.082 |
| angry (Noul) | 91.7% | 0.079 |
| priority (Score; the rule is **not in the text**) | 44.7% | 0.325 |
| all 900 | 75.1% | 0.107 (noise floor 0.024) |

**Correction of 2026-09-22 (commit "Correct the refit-T column…").** The refit temperatures first published were dominated by the 1e-6 log floor applied to exact-zero probabilities. Refit with the floor at half a grid step (0.005):

| Primitive | Original T (1e-6 floor) | Corrected T (0.005 floor) | Correct answer at exactly 0 |
|---|---|---|---|
| Choice | 3.29 | **1.30** | 4.3% of items |
| Score | 3.40 | **1.92** | 2.0% of items |
| Noul | 0.66 | 0.66 | — (unchanged; Noul has no endpoints) |

- The author's summary: *"The direction survives … but the size does not … Read the sign, not the magnitude."*
- Accuracy, NLL, Brier and ECE are unaffected; Choice ECE moves 0.084 → 0.072.

**The `confidence` field.** Read as a probability of being correct, its ECE was 0.035 (OpenBookQA), 0.078 (HellaSwag) and 0.18 (synthetic). The author's advice: *"Don't threshold on the `confidence` field."*

### 10. AHTOOOXA — Russian and 14 other languages — grade P-raw

**Study 1** (pre-registered, `jev-1.13.0`, n = 600 paired per dataset):

| Dataset | Accuracy EN → RU | Paired Δ | ECE EN → RU |
|---|---|---|---|
| XNLI | 88.3% → 77.3% | −11.0 pp [−14.2, −7.8] | 0.032 → 0.096 (Δ +0.063 [+0.033, +0.088]) |
| MASSIVE | 86.7% vs. 85.2% | −1.5 pp [−3.5, +0.2] | 0.066 vs. 0.071, no detectable difference |

**Gating at confidence ≥ 0.9 on XNLI:**

| Language | Coverage | Accuracy on covered items |
|---|---|---|
| English | 70.5% | 97.2% |
| Russian | 51.8% | 88.7% |

**Other measurements:**

- **Determinism.** Flip rate 0.5–2.2% between identical requests. The API does **not** cache identical requests: probability vectors differ on 30–58% of repeats.
- **`confidence` field.** It matches `(k·p_max − 1)/(k − 1)` to within the API's 2-decimal rounding across 4,800 responses (max residual 0.02, r = 1.000). It is a function of `p_max`, not an independent estimate. Archer Hume reads the same formula from TypeSafe's official Python adapter source. The vendor also documents it: the docs' `confidence` page, section "How confidence is calculated" (live page and `llms-full.txt` alike), gives Choice confidence as `(p_max − 1/n)/(1 − 1/n)`, the same expression.
- **Tokens.** Russian text costs about 3.07× (XNLI) and 3.37× (MASSIVE) the state tokens of English.

**Studies 2–3** (2026-09-21, pre-registered; 21,600 + 33,720 calls, 0 errors):

- **All 14 languages are worse than English on XNLI**, from −4 to −16 pp. Swahili and Urdu are the worst.
- Tokenization cost does **not** predict the calibration loss: ρ = −0.10, permutation p = 0.64.
- The loss is concentrated in predictions drifting to `neutral`. Removing the `neutral` option recovered none of the entailment gap: mean recovery −9% [−34%, +10%] over 5 languages (Study 3, H2).

**Reinterpretation added 2026-09-26.**

- The drift matches the known XNLI translation artifact (Artetxe et al., 2020). XNLI premises and hypotheses were translated separately.
- In English, 98% of Jev's errors on gold entailment/contradiction items are already `neutral`. The 88% "lost to neutral" figure therefore *"describes Jev's general error mode on XNLI, not a language-specific mechanism"*.
- A jointly translated XNLI study (Study 4) was in progress at the last push (2026-09-26). No result was published by 2026-10-03.

### 11. marcosmartinez — Spanish — grade P-raw

- **Run.** `20260921-es-v1`: 19,200 calls, 3,200 paired items, `jev-1.13.0` pinned, $0.58, 0 errors.

**Effect of a Spanish state, with English instructions** (B − A, accuracy):

| Dataset | Δ accuracy |
|---|---|
| XNLI | −6.4 pp [−8.6, −4.3] |
| PAWS-X | −6.2 pp |
| MASSIVE | −3.7 pp |
| Belebele | −3.0 pp |

- **ECE.** Roughly doubled on XNLI (0.057 → 0.101) and PAWS-X (0.033 → 0.078).
- **Automation at p_max ≥ 0.9 on XNLI:**

  | Language | Coverage | Accuracy on covered items |
  |---|---|---|
  | English | 72.2% | 0.938 |
  | Spanish | 63.4% | 0.904 |

- **Spanish instructions** (C − B): no detectable difference on three of four datasets. PAWS-X +1.6 pp, "ambiguous".
- **Tokens.** Spanish text costs 17–38% more input tokens.

**Inconsistencies in the repo:**

- The README's "Reproduce" section still says *"The pre-registration is not frozen yet"*, while the commit history shows "Freeze the pre-registration" before "Phase 2: freeze, run and report". The findings section reports 0 deviations. Treat the "not frozen" text as stale.
- The author discloses the code *"was largely agent-written"* (commit, 2026-09-21).

### 12. Archer Hume — black-box architecture probes — grade P

- **Date.** 2026-09-17, `jev-1.13.0`, one early-access account.
- **Scale.** Methods list 1,029 probe records, 6,800 benchmark records, and eight follow-up sets (35 to 445 requests each). An evidence bundle is linked.

**Results relevant to evidence:**

- **Option position (reference-card task).** Mean correct-answer probability by where the card sat:

  | Card position | Mean probability (per order) | Correct |
  |---|---|---|
  | First | 0.50, 0.56 | 12/16 |
  | Middle | 0.49, 0.56 | 11/16 |
  | Last | 0.89, 0.87 | 16/16 |
  | Moved into the state | 1.00 | 48/48 |

- **Irrelevant fifth option.** The original study shifted the customer-vs-unknown log-odds from about +0.49 to +0.08. A ten-block replication found +0.38 → +0.11, mean change −0.28 [−0.36, −0.19], lower in all ten blocks. A changed description had an inconclusive effect. The author: it *"shows the options interact, not where in the model."*
- **Ticket order.** Reversing options moved a technical-support classification from roughly 0.84–0.89 to 0.93–0.96.
- **Calibration.** 1,200-item MMLU sample: 10-bin ECE 0.0313, with 990 items in the 0.9–1.0 bin. MMLU-Pro accuracy 84.6%. Fresh word problems: 32% accuracy at mean top probability 0.30.
- **Limits observed.**
  - About 32,768 tokens per branch (state plus one question) and about 65,536 per request.
  - Question isolation: a secret in a sibling question scored 0.00; in the state, 0.90–0.92.
  - The tokenizer matched none of 192 public tokenizers.
- **Labelled speculation.** The causal backbone and sparse-MoE parts are inference, as the author says (*"This is clearly all quite speculative"*).

### 13. arXiv 2609.24574 (Ibrahim & Zaki) — grade P-raw

Mirrors Ziems et al. (2024) on 18 computational-social-science tasks, 7,977 items. 15 evaluation tasks plus 3 pilot; pre-registered.

- **Accuracy.**
  - Jev trails the **per-task best LLM** on 14 of 15 tasks: median −11.6 macro-F1, significant after FDR correction on 12.
  - The paper itself calls the per-task best *"an oracle comparator that favors the LLM s[ide]"*.
  - Within the sub-$0.10-per-1,000 band, Jev's median macro-F1 of 0.581 beats four of six hosted open-weight baselines. It trails Gemma 4 31B (0.611) and Qwen3 235B (0.604).
- **Cost.**
  - $0.21 for all 7,977 items, $0.027 per 1,000 items.
  - The per-task best LLM costs a median **44×** more.
  - Table 3 lists every baseline's $/1,000 items: $0.04 (OSS 120B) to $7.90 (Fable 5.1). The median over the 19 LLMs is $0.501, which is 18.6× Jev's $0.027. That is the source of xbill9's "18.6x" (recomputed here, 2026-10-03).
  - Total study spend was $244.54.
- **Calibration.**
  - Better than the verbalized confidence of 16 of 19 LLMs.
  - The three frontier Claude models did better: median ECE 0.157 vs. 0.066.
  - At ≥0.9 confidence: median coverage 0.376, accuracy 0.815. The pre-registered routing hypothesis (≥0.85) was **unsupported**.
  - Empathy task: 78% of items at ≥0.9, accuracy 0.383 vs. a 0.371 base rate.
- **Cascades** (median task):

  | Partner | Threshold | Cascade | Partner alone | Cost |
  |---|---|---|---|---|
  | Gemini 3.8 Flash | 0.8 | 0.674 | 0.678 | 56% of the partner's |
  | Gemma 4 31B | 0.5 | 0.638 | 0.626 | — |
  | Haiku 4.5 | — | 0.639 | 0.622 | 27% of the partner's |

- **Schema.** Zero invalid answers across 23,703 decision-model calls.
- **Hypotheses.** All three pre-registered hypotheses about stated uncertainty were unsupported.

### 14. arXiv 2609.26758 (option names) — grade P

- **Data.** 1,200 binary questions from the "Typed Decisions" dataset (LocalLLaMA): invoice, agent-trace, security-alert and customer-service tasks, 300 each. Gold follows the definition, and 57.5% of gold answers are "yes".
- **Manipulation.** Only which name is bound to which definition changes.

**Results:**

- **Open models.** The abstract's headline (70.4 more flips per hundred, AUC 0.94 → 0.23) is for **Laya, an open-weight model**, not hosted Jev.
- **Hosted Jev (Section 4.5):**
  - yes/no flips **32.5%** of decisions, 30.4 pp more than 0/1 [27.6, 33.3];
  - balanced accuracy 71.3% → 51.6%;
  - AUC 81.5% → 58.1%;
  - retest floor at most 1.33%, so the flip rate is 24× the floor.

Flip rates by name pair (Table 6):

| Pair | Jev | Laya | Open-Jev |
|---|---|---|---|
| yes/no | 32.50% | 76.92% | 19.50% |
| false/true | 31.92% | 49.67% | 47.83% |
| absent/present | 15.17% | 47.75% | 24.06% |
| rejected/accepted | 13.83% | 45.75% | 8.17% |
| negative/positive | 4.17% | 48.25% | 23.44% |
| 0/1 | 2.08% | 6.50% | 3.61% |
| A/B | 1.67% | 6.00% | 26.00% |

**Authors' limitations:**

- Jev was queried *"at one point in time"* and is not deterministic.
- English only.
- *"We report the vulnerability and two mitigations but evaluate neither."* The two are option-ID debiasing (Zheng et al., 2024) and word-bias correction (Liusie et al., 2023).
- The Jev version is not named. The paper cites TypeSafe pages "Accessed 2026-09-22".

### 15. xbill9 — independent evidence review — grade S (aggregator)

- **Scope.** Read on 2026-09-23. Published on dev.to on 2026-09-24. The GitHub report was last changed 2026-09-24 ("Google Gemma calibration quote checked…").
- **Grading.** The author grades studies A/B/C. Each figure is marked "re-scored", "checked" or "reported".
- **Disclosures.** The author is a Google Developer Expert and AWS Community Builder, has *"no relationship with TypeSafe"*, and states that sources were gathered *"with AI assistance (Claude)"*.

Items in this ledger that are **(secondary only)** because they come from this review and were not re-opened here:

| Item | xbill9's wording | Notes |
|---|---|---|
| BANKING77 spread | "Eight runs … put Jev at 0.753 to 0.840, with a median of 0.809" | One of the eight is JevBench's own first-party run |
| Six-model study (Janardhan, `manjunathshiva/jev-frontier-bench`) | Jev 72.5%, ECE 0.161; Claude Fable 5.1 84.0%; GPT-6 Astra 79.0% | Source of "6.5 to 11.5 points behind the frontier" |
| Spam (bitnovus/jev-spam-eval) | 98.33% on 18,514 emails vs. TF-IDF LR 98.39% | Question wording tuned on labelled errors from the same sets |
| Server time | "about 105 ms in one run … about 76 ms once the network round trip is subtracted" | |
| Founder credit | TypeSafe's "co-invented RLHF" credit: Almeida is not an author of Christiano et al. (2017); he is an author of InstructGPT (arXiv 2203.02155) | |

**Its claim-travel table is useful:**

- Tom's Hardware's "194x faster and 445x cheaper against GPT-6 Astra" misnames the comparator: Astra is a reference labeller with no scored setup.
- Coverage that "Jev flips 70.4 of 100 answers" misattributes an open-model figure. Section 14 above confirms this against the paper.

### 18. Hacker News thread 49717558 — grade V (for the CEO's statements)

- **Thread.** Posted 2026-09-15 ("Introducing System One Models and Jev"); 1,989 points when fetched via the HN Algolia API on 2026-10-03.
- **CEO.** User `CompleteSkeptic` identifies as *"CEO here"* (comment 49718407).

**Statements usable as primary vendor statements:**

- *"architecture is close to the chest for now, but we have talked about writing a paper"* (49718824).
- To *"This is basically a zero-shot classifier … as accurately (they claim) as a frontier-level LLM"*: *"exactly right!"* (49718727).
- *"because these models are probabilistic, it's also possible to be confidently wrong"* (49718780).
- On "can't hallucinate": *"Would you say a linear classifier hallucinates?"* (49719080).
- *"strings (and all sequential data structures) are not allowed at all - this is how we make sure all outputs can be computed in parallel (thus no output token cost)"* (49719122).
- *"constrained decoding (OpenAI-style structured outputs) make models dumber unfortunately"* (49718849). This is an opinion; no evidence was given in the thread.

### 19. AbdelStark/awesome-typesafe-jev — grade S

- **Use here.** A curated index, last pushed 2026-10-02. It was used here only to find primary sources; no number in this ledger rests on it.
- **Its stance.** Its "Before you trust a decision" table cites studies not re-verified here (KoBBQ abstention, Janus cascade, jev-certify, jev-orderby-bench, an action-gate study). It explicitly warns: *"These are independent, study-specific observations, not a leaderboard or a guarantee for another task."*
- **Notable unverified entry.** *jev-does-not-play-dice*: Choice put 82.9% on face 1 of a fair die across 400 rolls. This is **(secondary only)**.

---

## Prompt injection evidence

The vendor documents the risk itself (jaggedness page, live 2026-10-03, item 6): *"State is data, and `jev-1.13` does not treat it as hostile by default. Content written to adversarially steer the model, whether that is an injected instruction, a deliberately misleading framing, or text that argues for its own classification, can move the answer."*

| Source | Date | Setup | Result | Grade |
|---|---|---|---|---|
| Check Point Research | 2026-09-24 | Due-diligence app over a "PonziCorp" report. One attacker-controlled section; 3 delivery modes × 3 difficulty levels × 3 attackers × 3 runs; 10 turns | All 9 attacker × level combinations broken at least once. Strongest attacker 25/27 at **$0.50 per break**, 4th turn on average. Marking the document untrusted: 16/27 broken, the same as inline. Anti-injection instruction: 18 → 17 of 27, "noise". Jev 59% attempts successful; "Model II with reasoning" 19% | P |
| ickma2311 `injection/` | 2026-09-22 | 1,200 decisions per system; authority-framed injection; Jev 1.13.0 plus two open replicas | Among injected tickets, Jev's confidence ranks hijacked above resisted: **AUROC 0.261 [0.190, 0.336]**. At τ = 0.85 the gate still rejects 69% of hijacked decisions | P-raw |
| arXiv 2609.28613 (Wu & Lim) | 2026-09-23 | 510 reconstructed InjecAgent cases | Malicious content *"shifts action probabilities but rarely causes Jev to select the attacker's target"*. Adaptive attacks raise success on fresh calls from 1.8% to 3.5% | P (abstract only) |
| arXiv 2609.30243 "JevOut" | 2026-09-24 | Optimised natural-looking context additions | Redirects 312 of 508 initially correct decisions (61.4%); 229 at ≥0.7 on the wrong option | P (abstract only) |
| VentureBeat | 2026-09-21 | One Octomind integration test, quoted | Block probability 0.76 → 0.48 and confidence 0.64 → 0.22 after a fake "pre-approved" field. VentureBeat: *"one command in one integration test, not a benchmark"* | S |

**Notes on the VentureBeat page.** It is behind a bot checkpoint and was read through a summarising fetch tool, which returned the quotes verbatim. It also reports two figures this ledger did not check at their own sources:

- *"Vercel said roughly 13% of its paid AI Gateway teams were running it within 24 hours"* **(secondary only)**;
- *"TypeSafe cleared 140,000 people from the waitlist within 36 hours"* **(secondary only)**.

**Reading.** The studies disagree on *how easily* a verdict moves:

- Check Point: easily, with an adaptive multi-turn attacker on one app.
- arXiv 2609.28613: rarely to a chosen target, on InjecAgent.

They agree that typed output is not a security boundary. The ickma2311 result adds a sharper warning: a confidence gate does not detect redirection; confidence was *higher* on hijacked decisions.

---

## Re-checks run for this ledger

### Vendor multipliers and the 67.8% mean (executed)

The data are the points published on https://evals.typesafe.ai, fetched 2026-10-03 and typed in by hand from the chart's `<title>` labels. The script checks that each overview point is the plain mean of the four workflow points, and computes speed and cost multipliers against Jev.

```python
# Points published on https://evals.typesafe.ai (fetched 2026-10-03), "workflow" setups only.
# Per workflow: (agreement %, USD per case, seconds per case), in the order
# security_incidents, agent_trace_observability, invoice_processing, customer_service.
per_workflow = {
    "Jev":         [(61.7, 0.0001, 0.3), (71.6, 0.0003, 0.5), (61.8, 0.0011, 0.5), (76.0, 0.0001, 0.4)],
    "sol":         [(62.5, 0.0295, 8.5), (76.6, 0.0575, 40.3), (79.1, 0.2152, 34.3), (78.3, 0.0323, 10.1)],
    "opus 5":      [(66.2, 0.0574, 15.1), (75.2, 0.1033, 27.4), (78.4, 0.4856, 92.1), (72.4, 0.0579, 16.6)],
    "sonnet 5":    [(60.8, 0.0271, 18.9), (68.0, 0.0545, 38.0), (72.9, 0.3616, 241.3), (69.3, 0.0264, 14.3)],
}
# Overview points (equal-weight mean over the four workflows), as published.
overview = {
    "Jev": (67.8, 0.0004, 0.4),
    "sol": (74.1, 0.0836, 23.3), "opus 5": (73.1, 0.1761, 37.8), "terra": (67.9, 0.0304, 10.1),
    "sonnet 5": (67.8, 0.1174, 78.1), "luna": (66.8, 0.0033, 12.9), "DS v4 pro": (65.5, 0.0413, 86.5),
    "DS v4 flash": (64.4, 0.0059, 51.9), "haiku 4.5": (53.6, 0.0195, 12.5),
}


def mean(xs):
    return sum(xs) / len(xs)


# 1. The overview point is the plain mean of the four workflow points.
for model, rows in per_workflow.items():
    acc = mean([r[0] for r in rows])
    print(f"{model:9s} mean of workflows {acc:.2f}%  published overview {overview[model][0]}%")

# 2. Speed and cost multipliers against Jev, from the rounded overview points.
_, jev_cost, jev_secs = overview["Jev"]
llms = [m for m in overview if m != "Jev"]
for m in llms:
    _, cost, secs = overview[m]
    print(f"{m:11s} cheaper x{cost / jev_cost:6.1f}  faster x{secs / jev_secs:6.1f}")
print(f"all 8 LLM setups: cheaper x{mean([overview[m][1] for m in llms]) / jev_cost:.1f}, "
      f"faster x{mean([overview[m][2] for m in llms]) / jev_secs:.1f}")
```

**Output (CPython 3.12.10):**

```text
Jev       mean of workflows 67.78%  published overview 67.8%
sol       mean of workflows 74.12%  published overview 74.1%
opus 5    mean of workflows 73.05%  published overview 73.1%
sonnet 5  mean of workflows 67.75%  published overview 67.8%
sol         cheaper x 209.0  faster x  58.2
opus 5      cheaper x 440.2  faster x  94.5
terra       cheaper x  76.0  faster x  25.2
sonnet 5    cheaper x 293.5  faster x 195.2
luna        cheaper x   8.2  faster x  32.2
DS v4 pro   cheaper x 103.2  faster x 216.2
DS v4 flash cheaper x  14.7  faster x 129.8
haiku 4.5   cheaper x  48.8  faster x  31.2
all 8 LLM setups: cheaper x149.2, faster x97.8
```

**What this establishes:**

- 67.8% is the equal-weight mean of 61.7 / 71.6 / 61.8 / 76.0.
- The homepage's **444.6×** is consistent only with Opus 5's cost (440× from rounded figures; Jev's per-case cost is published to a single significant digit).
- **193.6×** is closest to Sonnet 5's time (195×). Jev's 0.4 s is published to one decimal, so the true value lies in [0.35, 0.45) s. Within that range, DeepSeek V4 Pro workflow (86.5 s) and Opus 5 *prompt* (70.5 s) are also consistent with 193.6×. The vendor names neither comparator.
- Across all eight LLM workflow setups, Jev is about **149× cheaper and 98× faster per case** on the vendor's own grid. This reproduces xbill9's "97.8x faster and 149.2x cheaper".

### Wilson intervals of four published accuracies (executed)

```python
from math import sqrt


def wilson(k, n, z=1.959964):
    """95% Wilson score interval for k successes out of n."""
    p = k / n
    centre = (p + z * z / (2 * n)) / (1 + z * z / n)
    half = z * sqrt(p * (1 - p) / n + z * z / (4 * n * n)) / (1 + z * z / n)
    return centre - half, centre + half


# (study, correct, n, interval the study published)
published = [
    ("sanand0 BANKING77 pilot, Jev", 58, 77, (0.6465, 0.8360)),
    ("simonmesmith BANKING77 full test", 2846, 3080, (0.9141, 0.9329)),
    ("anisselbd phishing, Jev verdict", 1252, 2000, (0.605, 0.647)),
    ("anisselbd phishing, Claude Haiku 4.5", 1626, 2000, (0.795, 0.829)),
]
for name, k, n, (lo_pub, hi_pub) in published:
    lo, hi = wilson(k, n)
    print(f"{name:38s} {k}/{n} = {k / n:.4f}  Wilson [{lo:.4f}, {hi:.4f}]  published [{lo_pub}, {hi_pub}]")
```

**Output:**

```text
sanand0 BANKING77 pilot, Jev           58/77 = 0.7532  Wilson [0.6465, 0.8360]  published [0.6465, 0.836]
simonmesmith BANKING77 full test       2846/3080 = 0.9240  Wilson [0.9141, 0.9329]  published [0.9141, 0.9329]
anisselbd phishing, Jev verdict        1252/2000 = 0.6260  Wilson [0.6046, 0.6469]  published [0.605, 0.647]
anisselbd phishing, Claude Haiku 4.5   1626/2000 = 0.8130  Wilson [0.7953, 0.8295]  published [0.795, 0.829]
```

**Notes:**

- The phishing counts (1,252 and 1,626) are **inferred** from the published 62.6% and 81.3% of 2,000; the README gives percentages only. All four published intervals are reproduced.
- The 77-case interval spans 19 points. Per `summary.json`, Jev's interval [0.646, 0.836] overlaps **every** other model's marginal interval, even GPT-6 Astra's [0.793, 0.937]. The author's statement that Astra and Gemini *"were substantially stronger than Jev in this sample"* is about this sample, not a population claim.

---

## Well-established, contested, unknown

### Well-established

Replicated by independent studies with raw data, or stated by the vendor and confirmed:

1. **Schema validity.** Jev returns a valid option on every call: 0 invalid in 23,703 calls (arXiv 2609.24574), 0 in 4,621 (scienthoon). This is a structural property, not accuracy.
2. **Accuracy is mid-tier, not frontier.** It trails the best LLM on most tasks: 14/15 social-science tasks, BANKING77 pilot, CLINC150 vs. Terra. It is competitive with cheap LLMs and, with retrieved examples, near a fine-tuned BERT on BANKING77.
3. **Cost per decision is very low.** $0.027 per 1,000 annotation items (2609.24574); $0.038 per 1,000 emails (phishing). It is not always the cheapest: GPT-6 Luna on sanand0's pilot.
4. **Single-call latency is a few hundred ms client-side.** 0.24–0.44 s medians across #3, #5, #7 and #10. The vendor's 40–200× multipliers are not reproduced against small LLMs: 2.2× and 2.9×.
5. **Output is quantised to 0.01.** Choice and Score return exact 0 and 1. Noul never returned an exact 0 or 1 in 11,148 answers (observed range 0.01–0.99, SamuelSacco; also 0% in scienthoon). Whether it is clamped is not documented by the vendor. An exact 0 on the correct option cannot be repaired by temperature or Platt scaling.
6. **`confidence` is a function of the distribution, by the vendor's own definition.** For Choice, (k·p_max − 1)/(k − 1). The vendor publishes the formula on its `confidence` docs page ("How confidence is calculated"). AHTOOOXA's 4,800 responses match it to within rounding, and Hume read it from TypeSafe's adapter. marcosmartinez's `DEVIATIONS.md` adds a caveat: the largest residual is 0.0200, so the relationship is "close but does not hold exactly" on returned values. Either way, it is not a separate correctness estimate.
7. **Not deterministic; no caching.** Repeat flips 0.5–2.2%. The vendor's own cookbook shows flips on 2 of 8 questions.
8. **Questions in one request are isolated.** A sibling question's text is invisible to the others (Hume; SamuelSacco).
9. **Non-English costs accuracy on sentence-pair tasks.** XNLI −4 to −16 pp across 14 languages; −6.4 pp Spanish, −11.0 pp Russian. Writing instructions in Spanish does not help.
10. **Option names and option order move answers.** Names: 32.5% flips on the hosted model. Order: Hume, and since 2026-10-02 the vendor's own jaggedness page.
11. **A confidence gate is not a security boundary.** Prompt injection moves verdicts (Check Point; vendor docs). Confidence did not flag hijacked decisions (ickma2311 injection).
12. **Small-label recalibration helps a lot, in-domain.** SamuelSacco took a Platt slope fitted once, transferred it, and refit only the intercept on 50 labels. That cut held-out ECE by 62%. Fewer than about 30 labels can leave calibration up to four times worse. The caveat: the slope did not transfer cleanly across domains (0 of 3 new-domain slopes fell in the email band; verdict "PARTIAL"), and survival across a retrain is untested.

### Contested

Credible sources disagree, or the evidence is thin:

- **How calibrated the raw probabilities are.**
  - Near-calibrated on familiar English sets: ECE ≈0.02–0.03 (scienthoon public sets, Hume MMLU, AHTOOOXA EN XNLI).
  - Clearly miscalibrated on others: 0.107–0.157 across the primary studies here (scienthoon synthetic 0.107, sanand0 0.138, phishing 0.154, arXiv 2609.24574 median 0.157). xbill9 reports 0.161 for Janardhan's six-model study **(secondary only)**.
  - The **direction** differs:
    - over-confident on phishing;
    - "compression toward the middle" on the synthetic gradient;
    - Noul under-confident (T 0.66) while Choice and Score are over-confident (corrected T 1.30 and 1.92).
- **Whether Jev's confidence ranks errors better than an LLM's.** AUROC 0.853 vs. 0.795 on Banking77; 0.734 vs. 0.816 on CLINC150 (ickma2311). Neither direction is established.
- **Whether cascades save money at equal accuracy.**
  - 2609.24574: they match the LLM at ¼–½ of its cost.
  - ickma2311: the saving holds 1 pp below the partner but **the sign flips at exact parity**, because 102 of 200 items had confidence exactly 1.0.
- **Ease of prompt injection.** Check Point: cheap, multi-turn adaptive. arXiv 2609.28613: rarely reaches the attacker's chosen target.
- **Whether the language gap belongs to the language or to translated benchmarks.** The XNLI translation artifact is a live alternative (AHTOOOXA, 2026-09-26). Study 4 is not yet published.
- **Choice vs. Noul for the same decision.** The vendor removed its own Noul-vs-Choice example on 2026-10-02 (see the history section). SamuelSacco's evidence is n = 1. arXiv 2609.33209 reports Jev's negation pairs miss summing to 1 by 0.064 on average (abstract only).

### Unknown

Nobody has published it:

- **Architecture, parameter count, base model, and RLCD method.** No paper; "close to the chest" per the CEO on HN, 2026-09-15.
- **Training data and contamination** of any public benchmark.
- **Behaviour across model versions.** Only `jev-1.13.0` exists. Whether fitted calibration maps or thresholds survive a retrain is untested (SamuelSacco issue #9).
- **Service limits under load.** SLA, uptime and paid-tier throughput. Rate limits are documented as changing "without notice".
- **Calibration on real, non-synthetic, domain-shifted production data with human labels** at scale.
- **Italian, Portuguese and most other non-English languages.** AHTOOOXA covers fourteen XNLI languages. No dedicated Italian or Portuguese study was found. The only broader coverage is arXiv 2609.37647's single aggregate, "86.7% on Belebele across 122 languages" (abstract only), which gives no per-language figure here.
- **Whether generative LLMs show the same option-name sensitivity** under the 2609.26758 protocol. The paper did not test it.

---

## History of the vendor's jaggedness and models pages

Two copies of the documentation existed on 2026-10-03, and they disagree:

| Copy | How to get it | Jaggedness "Last reviewed" | Rate limit |
|---|---|---|---|
| Per-page markdown (current) | `https://docs.typesafe.ai/model-jaggedness/jev-1.13.md`, `https://docs.typesafe.ai/models.md` | **2026-10-02** | **80** requests/s |
| Full dump | `https://docs.typesafe.ai/llms-full.txt`, still served unchanged on 2026-10-03 (byte-identical to the copy taken earlier that day) | **2026-09-17** | **40** requests/s |

**Practical consequence.** Any agent or tool that reads `llms-full.txt` receives the older guidance. It sees the removed "structural invariants" section, does not see the option-order failure mode, and gets the old rate limit. Prefer the per-page `.md` URLs.

### Jaggedness page: what changed between the 2026-09-17 review and the 2026-10-02 review

A line-by-line diff of the two copies shows exactly three content changes:

1. **"Last reviewed"** moved from 2026-09-17 to 2026-10-02.
2. **Failure mode #8 was replaced.** It was "Common-sense structural invariants" (*"Ask each decision one way; enforce identities in code"*). It is now "Choice option order" (*"Reorder the options and check the answer is consistent"*).
3. **The section body was replaced.**

   **Removed** (verbatim from the 2026-09-17 copy):

   > `jev-1.13` is extremely consistent, meaning you should expect quantitatively similar outputs for semantically similar inputs. However there are many structural invariants one might imagine to hold that simply aren't guaranteed by the model.

   It went on to give two examples:

   - "Is the customer asking for a refund?" asked as a Noul (0.22) and as a yes/no Choice (yes 0.01, no 0.99, confidence 0.97) on the same ticket.
   - A question and its negation as two Nouls: `refund` 0.72 + `not_refund` 0.47 = 1.19.

   Its advice: *"Don't carry a threshold tuned on a Noul over to a Choice, and don't hold the model to arithmetic identities between separate questions."*

   **Added** (verbatim from the 2026-10-02 copy):

   > In some cases, we observed that the order of a [Choice](/primitives/choice)'s options can affect the answer, and `jev-1.13` leans toward the option that comes first.
   >
   > **Instead:** reorder the options to double check that the answer stays consistent.

Everything else on the page is unchanged between the two copies, including the adversarial-content wording quoted above, the counting and date sections, context rot, and the closing tip inviting reports on Discord (present in both copies). The only other differences are formatting (the intro line became a blockquote) and the Mintlify footer line on the per-page `.md`.

**Context, not causation.**

- The option-order effect was measured independently before the vendor documented it: Hume on 2026-09-17, then arXiv 2609.26758 on 2026-09-22 (for option *names*).
- The removed refund 0.72 / 0.47 example had been quoted as a vendor-acknowledged weakness by the xbill9 review (2026-09-23).
- Independent work on the removed topic continued after removal: arXiv 2609.33209 and 2609.33971 on probability coherence (2026-09-27, abstracts only).
- The vendor gives no reason for the removal. **The structural-invariant problem has not been shown fixed.** The text was simply dropped. Keep treating a Noul threshold and a Choice threshold as different instruments.

Intermediate versions between 2026-09-17 and 2026-10-02 could not be checked: the Wayback Machine CDX API returned HTTP 429 on 2026-10-03.

### Models page: what changed

| Field | Dump (older) | Live (2026-10-03) |
|---|---|---|
| Rate limits | 100K tokens/s / **40** requests/s | 100K tokens/s / **80** requests/s |
| `GET /v1/models` response fields | `models`, `name`, `description`, `release_date` without `required` | the same fields marked `required` |

The price ($42 / Btok, $0.042 / Mtok input, output free), context limits (64k per request; 32k for state plus the longest question) and aliases are the same in both. Both copies carry the same warning: *"Rate limits are adjusting dynamically … the limits above can change without notice."*

### Other pages compared

43 per-page `.md` files were diffed against their `llms-full.txt` sections: the introduction, concept, primitive, confidence, pattern, demo, models, API, legal, agent-skill and SDK pages. Cookbooks were not diffed.

A prose-level diff found real text changes only on the jaggedness and models pages. Everything else was formatting:

- MDX components and heading anchors;
- image `src` attributes;
- `required` attributes added on API parameters (`api.md`);
- interactive components on the `primitives/score` and `confidence` pages. The `primitives/score` explainer (*"How the score and confidence are calculated"*) exists only inside a component, which the dump strips, so it cannot be dated from this comparison. The `confidence` page's own "How confidence is calculated" section, with the Noul `|2p − 1|` and Choice `(p_max − 1/n)/(1 − 1/n)` formulas, is plain markdown and is present in both copies.

---

## Newer than 2026-09-28

**Repositories:**

| Item | Date | What it adds | Checked here |
|---|---|---|---|
| typesafe-ai/WorkflowEvals | created 2026-09-28 | Vendor eval code; first public case counts (705 total) | Yes (README) |
| sanand0 pilot update | 2026-09-28 | GPT-6 Luna added; Jev no longer the cheapest in the set | Yes |
| SamuelSacco ledger, FINAL-VERDICTS | 2026-10-01 | Laya comparison; verdict table | Yes |
| AbdelStark/awesome-typesafe-jev | 2026-10-02 | Index | Yes (index only) |
| Jaggedness page review | 2026-10-02 | Option-order section added; invariants section removed | Yes |

**arXiv papers.** Abstract only: each was read on arXiv (export API, 2026-10-03), not in full, and is not graded beyond that.

| Paper | Date | Abstract claim |
|---|---|---|
| 2609.37647 (Deußer, Sparrenberg, Sifa) | 2026-09-29 | `jev-1.13.0` zero-shot on 37 datasets, 346,009 requests, under $10. Beats Qwen3.8-27B on 27 of 37 (none of Qwen's 9 leads outside the bootstrap intervals) and Gemma-4-E4B on all 37. "Choice probabilities are well calibrated"; binary probabilities "poorly placed relative to a fixed 0.5 threshold". Rotating options leaves accuracy unchanged |
| 2609.38827 (Gao et al.) | 2026-09-30 | Ordinal-scale bias. On ANLI, 38.8% of predictions and 51.3% of errors go to Neutral. On 36 ordinal datasets, final decisions use 67–76% of the effective gold support |
| 2609.39496 (Zhong, Li, Lai) | 2026-09-30 | With the right numeric answer absent, correct rejection falls to 7% despite a none-of-the-above option (99% when present). A Boolean check gets 99%. A tuned threshold raises rejection to 79% |
| 2610.01006 (Shankaranarayana, Runje, Jannink) | 2026-10-01 | Confidence calibrated on familiar closed-choice tasks but *"fails as an indicator of missing knowledge"*. Up to 0.80 on a salient option with no relevant information; exceeds accuracy by 0.21–0.33 on post-boundary news |
| 2609.32160 (Tang, Zheng); predates 2026-09-28 but is listed here as the first review paper | 2026-09-26 | Review of 28 papers from 19–24 September. *"the typed readout itself has not shown an independent accuracy advantage over comparable label-probability readouts"*; 14-item evaluation checklist |

The arXiv listing for "Jev" returned more than 40 further 2026-09-24 → 2026-10-01 preprints, including domain applications. They are not covered here.

---

## Where secondary sources diverge from primary ones

Check these before repeating a number you saw secondhand:

| Circulating statement | Primary source says |
|---|---|
| "Jev flips 70.4 of 100 answers when options are renamed" | 70.4 is for the open Laya model. Hosted Jev: **32.5%**, AUC 0.815 → 0.581 (2609.26758 §4.5) |
| "194x faster and 445x cheaper than GPT-6 Astra" | Astra is a reference labeller, not a scored setup. 444.6× matches Opus 5's cost; the comparator is unnamed by the vendor |
| "Jev scores 67.8% accuracy" | 67.8% **agreement with two LLMs**, no human labels |
| "Jev's BANKING77 accuracy is 75.3%" | That is n = 77 with no label descriptions, and the lowest published run. With retrieved examples on all 3,080 test items: 92.40% |
| "Jev is 18.6× cheaper than LLMs on social-science tasks" (xbill9's table, row "19 LLMs, social-science tasks (median)") | Both numbers are in arXiv 2609.24574v2, but they are **different aggregations**. The paper's own headline: the per-task best LLM costs a **median 44×** more. 18.6× is the median over all 19 LLM baselines of Table 3's measured $/1,000 items ($0.501, GLM 5.3) divided by Jev's $0.027, recomputed here on 2026-10-03. Do not quote either as "Jev vs. LLMs" without saying which comparator |
| "Choice temperature 3.29, Score 3.40" (scienthoon, as first published) | **Corrected 2026-09-22** to 1.30 and 1.92. The originals were an artefact of the log floor |
| "Jev is overconfident" (unqualified) | It depends on primitive and data: Noul T 0.66 (under-confident) on the synthetic set; near-calibrated on public English sets |
| "No published rate limits" (SamuelSacco ledger, still on 2026-10-01) | `/models` publishes 100K tok/s and 80 req/s (2026-10-03), with a "can change without notice" warning |
| A summarising fetch of evals.typesafe.ai reported the Security Incidents top workflow as "58.8% (haiku 4.5)" | The page's own points: Opus 5 66.2% is top. Haiku 4.5 58.8% and Jev 61.7% are both below it. Read the page, not a summary |

---

## Sources

All read on 2026-10-03 unless stated.

**Vendor (TypeSafe):**

- https://typesafe.ai/blog/introducing-system-one-models-and-jev (2026-09-15)
- https://evals.typesafe.ai and the four workflow pages
- https://github.com/typesafe-ai/WorkflowEvals
- https://docs.typesafe.ai/model-jaggedness/jev-1.13.md
- https://docs.typesafe.ai/models.md
- https://docs.typesafe.ai/llms-full.txt
- https://docs.typesafe.ai/cookbooks/consistency_choice_cookbook (via `llms-full.txt`)
- Hacker News thread https://news.ycombinator.com/item?id=49717558 (via https://hn.algolia.com/api/v1/items/49717558)

**Independent studies (primary):**

- https://github.com/sanand0/llmevals/tree/main/jev, https://sanand0.github.io/llmevals/jev/
- https://github.com/simonmesmith/jev-banking77-experiment
- https://github.com/ickma2311/jev-baselines-eval (including `injection/`)
- https://github.com/anisselbd/jev-phishing-bench
- https://github.com/SamuelSacco/jev-exploration (`docs/claims-audit.md`, `docs/FINAL-VERDICTS.md`)
- https://github.com/scienthoon/jev-ood-calibration (correction commit 2026-09-22)
- https://github.com/AHTOOOXA/jev-cyrillic-audit
- https://github.com/marcosmartinez/jev-acento
- https://archerhume.com/posts/jevs-architecture-unmasked
- https://arxiv.org/abs/2609.24574 (v2, PDF and HTML)
- https://arxiv.org/abs/2609.26758 (v2, PDF and HTML)
- https://arxiv.org/abs/2609.28613 (abstract)
- https://arxiv.org/abs/2609.30243 (abstract)
- https://blog.checkpoint.com/ai-security/jev-is-not-a-language-model-but-it-breaks-like-one-prompt-injection-against-a-typed-decision-model/ (2026-09-24)

**Secondary (used to locate sources, or flagged):**

- https://dev.to/gde/jev-after-eight-days-of-independent-tests-level-with-mid-price-llms-behind-the-frontier-1kln and https://github.com/xbill9/gemma4-dev/blob/main/jev/reports/Jev%20independent%20evidence%20review.md
- https://venturebeat.com/security/companies-are-putting-jev-in-charge-of-ai-agent-decisions-and-prompt-injection-can-influence-the-verdict (2026-09-21)
- https://github.com/AbdelStark/awesome-typesafe-jev
