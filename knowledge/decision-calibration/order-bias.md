# Option-Order and Label Bias in Decision Models - Deep Reference

> Official Documentation: https://docs.typesafe.ai/model-jaggedness/jev-1.13
> Official Documentation: https://docs.typesafe.ai/api
> Paper: https://arxiv.org/abs/2309.03882 (Zheng et al., ICLR 2024 - PriDe)
> Last verified: 2026-10-03
> Verified against: typesafe-sdk 0.7.2 (Python, offline via `httpx2.MockTransport`), TypeSafe docs for `jev-1.13`, numpy 2.5.3, scipy 1.18.1

## Overview

A classifier that picks from a list of options can prefer an option because of **where** it sits or **what it is called**, not what it means. For LLMs answering multiple-choice questions this is well documented; for typed decision models like TypeSafe Jev, the vendor documents it for `jev-1.13` itself. This page covers the literature, how to measure the effect on your own questions, the two families of fixes (permutation averaging and PriDe-style prior estimation), the single-request rotation trick that makes averaging cheap on Jev, and the separate, larger problem of option **names**.

It extends section 4 of the skill `ai-integration/decision-model-calibration`. Every function is copied verbatim from the companion module `kb_calib.py`; every longer example from `doc_examples.py`, executed offline against a mock of the `/v1/systemone` endpoint that returns the documented response shape. The mock's biases are chosen by us; its numbers illustrate the procedure and say nothing about Jev's real behaviour. The module is not published in this knowledge base: the functions assume its imports (listed in `calibration.md` under *ECE estimators and their bias*) and `wilson_interval` from `calibration.md`; the examples also use `from typesafe_sdk import Choice, TypeSafeClient` and a mock client, `client`, plus `tickets` / `CRITERIA` fixtures that are not shown.

---

## Table of Contents

1. [What the literature found](#what-the-literature-found)
2. [What TypeSafe documents for jev-1.13](#what-typesafe-documents-for-jev-113)
3. [Measuring: flip rate, RStd and the test-retest floor](#measuring-flip-rate-rstd-and-the-test-retest-floor)
4. [Cyclic rotation in a single Jev request](#cyclic-rotation-in-a-single-jev-request)
5. [Permutation averaging vs PriDe prior estimation](#permutation-averaging-vs-pride-prior-estimation)
6. [Label-renaming sensitivity](#label-renaming-sensitivity)
7. [Order of operations with calibration](#order-of-operations-with-calibration)
8. [Checklist](#checklist)
9. [Sources](#sources)

---

## What the literature found

**Zheng, Zhou, Meng, Zhou & Huang, *Large Language Models Are Not Robust Multiple Choice Selectors*, ICLR 2024 (spotlight; arXiv 2309.03882).**
- Studies 20 LLMs on three MCQ benchmarks. Models "prefer to select specific option IDs as answers (like 'Option A')" - **selection bias** - and the bias "primarily stems from LLMs' token bias, where the model a priori assigns more probabilistic mass to specific option ID tokens (e.g., A/B/C/D)" (abstract).
- Measures it with **RStd**, "the standard deviation of recalls of different option IDs" (paper).
- Proposes **PriDe**: estimate the prior over option IDs "by permutating option contents on a small number of test samples, and then applies the estimated prior to debias the remaining samples" (abstract). Default estimation share α = 5% of test samples; reported cost ×1.15 for PriDe at 5% versus ×4 for cyclic permutation on 4-option questions (paper, via the HTML version, fetched 2026-10-03).
- Baselines: **Cyclic Permutation** averages the observed probabilities over the n cyclic shifts; **Full Permutation** does the same over all n! orders.

**Pezeshkpour & Hruschka, *Large Language Models Sensitivity to The Order of Options in Multiple-Choice Questions*, 2023 (arXiv 2308.11483).**
- "a considerable performance gap of approximately 13% to 75% in LLMs on different benchmarks, when answer options are reordered, even when using demonstrations in a few-shot setting" (abstract).
- Conjecture: the sensitivity "arises when LLMs are uncertain about the prediction between the top-2/3 choices". To **amplify** the bias, place the top two choices first and last; to **mitigate** it, place them adjacent (abstract).
- Practical reading: order bias concentrates on the items that are already close calls - the same items a threshold sends to review. Averaging over orders mostly changes those.

**Zhao, Wallace, Feng, Klein & Singh, *Calibrate Before Use*, ICML 2021 (arXiv 2102.09690).**
- Few-shot prompts are biased "towards predicting certain answers, e.g., those that are placed near the end of the prompt or are common in the pre-training data" (abstract). The fix, **contextual calibration**, is in `calibration.md`; it removes a label prior, measured with no labels.

**Sun, Xu, Shi & Yang, *Type-Safe Is Not Error-Free: A Constrained Decision Head Follows the Option Name, Not the Rubric Bound to It*, 2026 (arXiv 2609.26758v2; v1 submitted 2026-09-22).** The study most directly about Jev - see [Label-renaming sensitivity](#label-renaming-sensitivity).

---

## What TypeSafe documents for jev-1.13

The model-jaggedness page lists option order among the model's known limits (https://docs.typesafe.ai/model-jaggedness/jev-1.13, live page fetched 2026-10-03; the older `llms.txt` dump lags this page):

> In some cases, we observed that the order of a [Choice](/primitives/choice)'s options can affect the answer, and `jev-1.13` leans toward the option that comes first.
>
> **Instead:** reorder the options to double check that the answer stays consistent.

Two structural facts from the API docs make rotation practical on Jev:

- A Choice can have "a maximum of 255 options" (https://docs.typesafe.ai/api).
- "Jev ingests the `state` once and evaluates every question against it in parallel. The 64k budget covers the `state` plus all questions combined; the 32k budget applies to the `state` plus the single longest question" (https://docs.typesafe.ai/models). So k copies of the same question in **one** request cost k× the question tokens, but the state only once and one round trip.

Other measurements, reported in the skill `typed-decision-models` and **not re-verified for this page**: a reference card moved from first to last position changed the mean probability on the right answer from 0.50 to 0.89 (Archer Hume, 10,000 calls); adding an irrelevant fifth option shrank the log-odds between two others from +0.49 to +0.08, i.e. options **interact**. Interaction matters below: PriDe assumes they do not.

Note what differs from the LLM literature: Jev options have **names**, not letter IDs, and the model reads a rubric per option. The "token bias on A/B/C/D" mechanism does not apply literally; a position lean and a name effect both do.

---

## Measuring: flip rate, RStd and the test-retest floor

Three numbers, all computed offline over your labelled (or even unlabelled) corpus:

| Metric | Definition | Needs labels? |
|---|---|---|
| **Flip rate** | share of items whose top choice is not the same under every order | no |
| **RStd** (Zheng et al.) | std over positions of recall-when-the-answer-sits-at-that-position | yes |
| **Test-retest floor** | share of items whose top choice differs between two identical calls | no |

A flip rate means nothing without the floor: the vendor's self-consistency cookbook notes that "picked labels can flip inside a single condition, including TypeSafe" and that "TypeSafe flips on 2 of the 8 questions" (TypeSafe cookbook *Self-consistency: choices*, in `llms.txt`, fetched 2026-10-03); Pydantic AI's decision-model docs say Jev's numbers "move by a few hundredths from one run to the next" (Pydantic AI docs, `docs/models/decision.md`, main branch, read 2026-10-03). Order effects have to be measured **against** that noise.

```python
def flip_rate(winners_by_order):
    """winners_by_order: (n_items, n_orders) array of the option each order picked.
    Share of items whose pick is not the same under every order, with a Wilson CI."""
    w = np.asarray(winners_by_order)
    flipped = ~np.all(w == w[:, :1], axis=1)
    k, n = int(flipped.sum()), len(flipped)
    return k / n, wilson_interval(k, n)

def recall_std(pred_positions, true_positions, k):
    """RStd (Zheng et al.): std of per-position recall. 0 = no selection bias."""
    pred = np.asarray(pred_positions)
    true = np.asarray(true_positions)
    recalls = [float((pred[true == j] == j).mean()) for j in range(k) if (true == j).any()]
    return float(np.std(recalls)), recalls

def paired_flip_rate(a, b):
    """Share of items whose answer differs between two runs/conditions, with Wilson CI."""
    a, b = np.asarray(a), np.asarray(b)
    k, n = int((a != b).sum()), len(a)
    return k / n, wilson_interval(k, n)
```

An offline audit over a corpus, one request per item carrying every rotation (executed against the mock):

```python
def order_audit(client, states, instructions, criteria, limit=None):
    """Offline audit: every rotation of the options, one request per item.
    Returns the flip rate (with CI) and the per-item averaged distributions."""
    picks, averaged = [], []
    for state in states:
        orders = rotations(criteria, limit)
        questions = {f"order_{i}": Choice(instructions=instructions, criteria=c)
                     for i, c in enumerate(orders)}
        result = client.system_one(state, questions)
        answers = [result.choices[k] for k in questions]
        picks.append([a.choice for a in answers])
        averaged.append({o: np.mean([a.probabilities[o] for a in answers]) for o in criteria})
    rate, ci = flip_rate(np.array(picks))
    return rate, ci, averaged


rate, ci, averaged = order_audit(client, tickets, "Which team should handle this ticket?", CRITERIA)
```

On the mock (first-position lean +0.8 logits, 30 tickets, 3 options): flip rate 0.500, 95% CI [0.332, 0.668]. Caveat on that interval: the 30 tickets are 6 distinct texts repeated 5 times and the mock is deterministic, so the effective sample is 6 items (3 of them flip) and the Wilson CI, which assumes 30 independent items, is too narrow; it is shown only to exercise the code. On real traffic, audit a few hundred **distinct** items before deciding.

What a simulation shows about the shape of the damage (`exp_order.py`; 2,000 items, 4 options, content logits ~ N(0, 1.2), observed = softmax(1.5·content + position log-bias (+0.8, +0.2, 0, -0.2))):

| | value |
|---|---|
| items whose pick depends on order | 0.402 [0.380, 0.423] |
| per-position recall, single fixed order | 0.697 / 0.554 / 0.499 / 0.419 |
| RStd, single order | 0.101 |

A modest log-bias - smaller than the content signal on most items - flips 40% of picks. In this model a flip can only happen where the top content logits are within the bias range of each other, i.e. among close calls (Pezeshkpour & Hruschka's top-2/3 conjecture, reproduced by construction); the share of flips by margin was not tabulated.

---

## Cyclic rotation in a single Jev request

`rotations` produces the n cyclic shifts of the options (each option visits each position exactly once) or, with `limit`, an evenly spread subset that keeps the original order first. Cyclic shifts are enough for averaging out a **position** effect: n orders instead of n!.

```python
def rotations(criteria, limit=None):
    """Cyclic rotations of a Choice's options (Zheng et al., ICLR 2024): n orders, not n!."""
    items = list(criteria.items())
    n = len(items)
    count = n if limit is None else min(limit, n)
    shifts = sorted({round(j * n / count) % n for j in range(count)})  # evenly spread, first order kept
    return [dict(items[i:] + items[:i]) for i in shifts]
```

`rotations` with 8 options (`exp_order.py`):

| limit | first option of each order |
|---|---|
| None | opt0, opt1, opt2, opt3, opt4, opt5, opt6, opt7 |
| 4 | opt0, opt2, opt4, opt6 |
| 3 | opt0, opt3, opt5 |
| 2 | opt0, opt4 |

With a limit, positions are not visited equally (with 3 of 8 rotations, some options never sit first), so the average is only partially debiased. Use the full cycle when k is small (k ≤ ~6) and the question is short.

The Jev call, unchanged from the skill:

```python
def order_checked_choice(client, state, instructions, criteria, limit=None):
    """Ask the same Choice under several option orders in ONE request and average.

    Questions in a request are independent and share one read of the state, so
    the extra orders cost only their question tokens, not another round trip.
    """
    from typesafe_sdk import Choice

    orders = rotations(criteria, limit)
    questions = {f"order_{i}": Choice(instructions=instructions, criteria=c) for i, c in enumerate(orders)}
    result = client.system_one(state, questions)
    answers = [result.choices[key] for key in questions]
    averaged = {o: sum(a.probabilities[o] for a in answers) / len(answers) for o in criteria}
    winner = max(averaged, key=averaged.get)
    stable = all(a.choice == winner for a in answers)
    return winner, averaged, stable
```

Against a mock that adds +0.8 to whichever option is listed first, with content logits billing 0.9 / technical 0.6 / sales -1.0 and `technical` listed first in the caller's order (`exp_order.py`):

| | value |
|---|---|
| requests sent | 1 (three questions `order_0..2`) |
| averaged | billing 0.517, technical 0.397, sales 0.087 |
| winner | billing |
| stable | False - the first-listed option won in at least one order |

Without rotation, the caller's order (technical first) would have returned `technical`. The SDK sends the model name you construct the client with; the default when none is given is `jev-latest` (observed in the request body sent by `TypeSafeClient` 0.7.2 through the mock transport; `typesafe_sdk/constants.py` sets `DEFAULT_MODEL = "jev-latest"`, overridable with the `TYPESAFE_DEFAULT_MODEL` environment variable). Pin a version (e.g. `model="jev-1.13"`) so a measured flip rate stays attached to a model.

Cost on Jev: n × (question tokens), one state read, one round trip. The questions in a request are evaluated "in parallel" (models page), so latency should stay roughly flat in n; this was not measured here.

---

## Permutation averaging vs PriDe prior estimation

Write P_obs(d_i | x^I) for the probability of the option sitting at **position** d_i under permutation I. PriDe's model (Zheng et al., as given in the paper) is

```
P_obs(d_i | q, x^I) = Z⁻¹ · P_prior(d_i | q) · P_debiased(o_{f_I(i)} | q, x)
```

A position prior times a content term. Under cyclic permutations every option visits every position once, so averaging **log** P_obs over the n shifts cancels the content term per position:

```
P_prior(d_i | q) = softmax( (1/|I|) Σ_I log P_obs(d_i | q, x^I) )
```

(formula as given in the paper, fetched 2026-10-03). PriDe estimates this on a small share α of items, averages it into a global prior, and then debiases every other item from a **single** order: P_debiased ∝ P_obs / P_prior.

```python
def cyclic_average(observed, geometric=False):
    """observed: (n_orders, k) probabilities IN POSITION SPACE for k cyclic shifts,
    where row s shows option j at position (j - s) mod k. Returns the option-space
    average: arithmetic (Zheng et al.'s 'Cyclic Permutation' baseline) or geometric."""
    observed = np.asarray(observed, float)
    m, k = observed.shape
    opt = np.array([[observed[s, (j - s) % k] for j in range(k)] for s in range(m)])
    if geometric:
        g = np.exp(np.mean(np.log(np.clip(opt, 1e-12, 1.0)), axis=0))
        return g / g.sum()
    return opt.mean(axis=0)

def pride_prior(observed_sets):
    """PriDe prior over positions (Zheng et al., ICLR 2024):
    per item, softmax(mean over the k cyclic shifts of log P_obs(position));
    then averaged over the estimation items ('global prior').

    observed_sets: list of (k, k) arrays, position-space probabilities for the k
    cyclic shifts of one item."""
    priors = []
    for obs in observed_sets:
        lp = np.log(np.clip(np.asarray(obs, float), 1e-12, 1.0))
        priors.append(softmax(lp.mean(axis=0)))
    prior = np.mean(priors, axis=0)
    return prior / prior.sum()

def pride_debias(observed_positions, prior):
    """P_debiased(position i) ∝ P_obs(position i) / prior(i), for a single order."""
    q = np.asarray(observed_positions, float) / np.asarray(prior, float)
    return q / q.sum(axis=-1, keepdims=True)
```

| | cyclic (arithmetic) | cyclic (geometric) | PriDe |
|---|---|---|---|
| Calls per item | n orders | n orders | 1 (+ n on the α estimation items) |
| Cost, 4 options, α = 5% | ×4 | ×4 | ×1.15 (paper) |
| Assumes | nothing beyond "position effect averages out" | multiplicative prior | multiplicative prior, **shared across items** of a question |
| Also returns | a per-item stability flag (`stable`) | same | nothing per item |
| Breaks when | n is large (tokens) | same | options interact; the prior differs by item type; order or option set changes |

Simulation (`exp_order.py`, same 2,000 items as above, metrics on the 95% not used for estimation):

| method | accuracy | RStd | ECE | questions per item |
|---|---|---|---|---|
| single fixed order | 0.543 | 0.101 | 0.125 | 1.00 |
| cyclic mean (arithmetic) | 0.558 | 0.013 | 0.086 | 4.00 |
| cyclic geometric | 0.557 | 0.012 | 0.102 | 4.00 |
| PriDe, prior from 5% | 0.557 | 0.012 | 0.102 | 1.15 |

The estimated prior was [0.423, 0.232, 0.190, 0.155], equal to the true softmax of the position bias - **because the simulation satisfies PriDe's assumption exactly** (an independent multiplicative position term, no noise). The same holds in the unit test, where `cyclic_average(..., geometric=True)` recovers the content distribution exactly. Real models do not satisfy it exactly: the Hume measurement above shows options interacting, which a single position prior cannot represent. So:

1. **Audit with full cyclic averaging** on a few hundred items (it needs no assumption).
2. If you want PriDe's cost, estimate the prior on α of items, then **check it**: on a held-out sample, compare PriDe's picks with full cyclic picks. Adopt PriDe only if they agree about as often as two identical calls do (the test-retest floor).
3. Re-estimate the prior whenever the instructions, option set, option count, order, names or model version change - it is a property of that exact question.

The arithmetic mean was better calibrated than the geometric one here (ECE 0.086 vs 0.102): averaging probabilities is a mixture, which is less confident than a product of experts. Either way, **fit the calibration map on the debiased output**, not on single-order probabilities.

PriDe on Jev, executed against the mock (first-position lean +0.8):

```python
def estimate_position_prior(client, states, instructions, criteria):
    """PriDe step 1 on a small sample (the paper uses 5% of items): all k cyclic
    orders per item, read back in POSITION space, softmax of the mean log-prob."""
    observed_sets = []
    for state in states:
        orders = rotations(criteria)
        questions = {f"order_{i}": Choice(instructions=instructions, criteria=c)
                     for i, c in enumerate(orders)}
        result = client.system_one(state, questions)
        observed_sets.append(np.array([
            [result.choices[f"order_{i}"].probabilities[o] for o in order]  # position order
            for i, order in enumerate(orders)
        ]))
    return pride_prior(observed_sets)


def debiased_choice(client, state, instructions, criteria, prior):
    """PriDe step 2: ONE order per item, divide out the position prior."""
    answer = client.system_one(state, {"q": Choice(instructions=instructions, criteria=criteria)}).choices["q"]
    observed = np.array([answer.probabilities[o] for o in criteria])
    q = pride_debias(np.clip(observed, 0.005, None), prior)
    return dict(zip(criteria, q))


prior = estimate_position_prior(client, tickets[:6], "Which team should handle this ticket?", CRITERIA)
fixed = debiased_choice(client, "Error on the invoice page", "Which team should handle this ticket?", CRITERIA, prior)
```

Output: prior [0.515, 0.243, 0.243] over positions (truth: softmax([0.8, 0, 0]) = [0.526, 0.237, 0.237] before the API's 0.01 rounding), and for "Error on the invoice page" - which mentions both an error and an invoice - the debiased distribution technical 0.478 / billing 0.461 / sales 0.061: a near-tie that the single biased order would have reported as a clear win for whichever option was listed first. The 0.005 clip before dividing is the same rounding-floor logic as in `calibration.md`.

---

## Label-renaming sensitivity

Sun, Xu, Shi & Yang (arXiv 2609.26758v2; v1 submitted 2026-09-22) change **only which option name is attached to which rubric** - "the question, state, rubric wording, and set of option names remain exactly the same" - on Jev and two Jev-like open-weight models. From the abstract:

- On 1,200 workflow decisions, renaming two options from 0/1 to no/yes "changes 70.4 more answers per hundred (95% CI: [67.6, 73.1]) and shifts AUC from .94 to .23, revealing a systematic reversal in the decision ranking rather than simple uncertainty."
- "The same operation has little effect with neutral option names"; across all 4 predicates the effect "is at least 7.4x larger than under the neutral control, and becomes stronger as the number of options increases."
- Read-out geometry matters: "a second model family that mean-pools over the full option span flips 4.1x less often."
- **Hosted model:** "the swap changes AUC from .8146 to .5806 and produces 24x as many answer flips as its test-retest floor."
- "replacing the option names with random character strings returns all model families to the neutral-control regime without reducing accuracy. The failure therefore depends on the semantic polarity of the option names rather than on the renaming operation itself."

So the effect is not "any rename hurts": it is that a name with its own meaning (yes/no, approve/reject) pulls the decision toward that meaning, overriding the rubric bound to it. (The skill summarises this as "AUC 0.81 → 0.58 on the hosted model with 24× the test-retest flip rate", which matches the abstract.)

Measure it on your own question exactly as the paper does - a rename of names only, against the test-retest floor, in one request per item:

```python
def rename_audit(client, states, instructions, criteria, rename):
    """Flips caused by renaming options (rubrics untouched) vs the test-retest floor.
    rename: {old_name: new_name}. A ratio well above 1 means the NAME is steering."""
    renamed = {rename.get(o, o): r for o, r in criteria.items()}
    back = {v: k for k, v in rename.items()}
    run_a, run_b, run_r = [], [], []
    for state in states:
        questions = {"a": Choice(instructions=instructions, criteria=criteria),
                     "b": Choice(instructions=instructions, criteria=criteria),
                     "r": Choice(instructions=instructions, criteria=renamed)}
        res = client.system_one(state, questions)
        run_a.append(res.choices["a"].choice)
        run_b.append(res.choices["b"].choice)
        run_r.append(back.get(res.choices["r"].choice, res.choices["r"].choice))
    floor, floor_ci = paired_flip_rate(run_a, run_b)
    effect, effect_ci = paired_flip_rate(run_a, run_r)
    return {"retest_flips": (floor, floor_ci), "rename_flips": (effect, effect_ci),
            "ratio": effect / floor if floor else math.inf}


noisy = TypeSafeClient(api_key="test", model="jev-1.13", transport=jev_mock(content, noise=0.6, seed=3))
audit = rename_audit(noisy, tickets, "Which team should handle this ticket?", CRITERIA,
                     {"billing": "money", "technical": "it", "sales": "commercial"})
```

On a mock where the name `money` itself adds +1.2 to the logit (and 0.6 logit noise per call), 30 items (again 6 distinct texts repeated 5 times; only the noise differs between repeats): test-retest flips 0.133, rename flips 0.367, ratio 2.75. On real traffic, use a few hundred items; the paper's hosted-model ratio was 24×.

Practice that follows from the evidence:

- **Treat option names as part of the model's input, frozen like an API.** Once wording is tuned and calibrated, a rename is a model change: re-measure, refit.
- **Avoid polar names whose everyday meaning competes with the rubric** (yes/no, true/false, approve/deny) when the rubric means something more specific. The paper found random character strings restored neutral behaviour without hurting accuracy; whether opaque names suit your code and logs is a design trade-off, not tested here.
- **Put the meaning in the rubric**, and keep names short and neutral; the rubric is what you want the model to follow.
- For a Noul, there are no names to rename - but its `criteria` descriptions for true/false (where supported) carry the same risk; this was not studied in the paper.

---

## Order of operations with calibration

1. Fix the wording, option names and option **set**.
2. Audit flips (cyclic) and the test-retest floor; decide: none / cyclic in production / PriDe.
3. Optionally divide out a content-free prior (`calibration.md`, contextual calibration) - computed in the **same order** you will serve.
4. Fit the temperature (or Platt) on the **debiased** probabilities, per question.
5. Choose thresholds or conformal sets (`conformal.md`) on the calibrated output.

Changing anything in step 1 invalidates steps 2-5.

---

## Checklist

- [ ] Flip rate measured over cyclic rotations, with Wilson CI, next to a test-retest floor from duplicate questions in the same request
- [ ] RStd / per-position recall computed if labels exist
- [ ] Decision recorded: no debiasing, cyclic in production (`stable` flag used for routing), or PriDe (prior validated against full cyclic on held-out items)
- [ ] All rotations fit the 64k request / 32k state-plus-longest-question budgets
- [ ] Option names neutral and frozen; rename audit run before any rename ships
- [ ] Calibration map fitted on debiased probabilities; refit after any wording/order/name/model change

---

## Sources

- Zheng, Zhou, Meng, Zhou & Huang, *Large Language Models Are Not Robust Multiple Choice Selectors*, ICLR 2024 - arXiv 2309.03882 (v4); ICLR 2024 poster https://iclr.cc/virtual/2024/poster/17638
- Pezeshkpour & Hruschka, *Large Language Models Sensitivity to The Order of Options in Multiple-Choice Questions*, 2023 - arXiv 2308.11483
- Zhao, Wallace, Feng, Klein & Singh, *Calibrate Before Use*, ICML 2021 - arXiv 2102.09690
- Sun, Xu, Shi & Yang, *Type-Safe Is Not Error-Free*, 2026 - arXiv 2609.26758v2 (abstract read 2026-10-03; the full paper was not)
- TypeSafe: https://docs.typesafe.ai/model-jaggedness/jev-1.13, https://docs.typesafe.ai/api, https://docs.typesafe.ai/models, cookbook *Self-consistency: choices* (all fetched 2026-10-03)
- Pydantic AI, *Decision models* page (`docs/models/decision.md`, main branch, read 2026-10-03)
