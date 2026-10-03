# Conformal Prediction Sets for Decision Models - Deep Reference

> Official Documentation: https://arxiv.org/abs/2107.07511 (Angelopoulos & Bates, *A Gentle Introduction to Conformal Prediction*)
> Reference implementation (RAPS): https://github.com/aangelopoulos/conformal_classification
> Last verified: 2026-10-03
> Verified against: Python 3.12.10, numpy 2.5.3, scipy 1.18.1 (`scipy.stats.betabinom`), typesafe-sdk 0.7.2 response shape

## Overview

A calibrated probability says "0.9 means right nine times in ten **on average**". A conformal prediction set says "the true option is in this set at least 90% of the time", with a finite-sample guarantee that holds for any model - provided future traffic is exchangeable with the calibration data. For a decision model (TypeSafe Jev Choice/Score, an LLM read through logprobs, any classifier) this turns "pick one or hand off" into a three-way policy: one option → act, several → ask, none → out of distribution.

This page goes beyond section 6 of the skill `ai-integration/decision-model-calibration`: the quantile and its edge cases, LAC vs APS vs RAPS with working code, class-conditional coverage, how to check coverage in production, what drift does to the guarantee, and what a set size actually tells you. Every function is copied verbatim from the companion module `kb_calib.py` (executed by its test suite, 29 tests passing on 2026-10-03); experiment numbers come from `exp_conformal.py` (fixed seeds, simulated data - they illustrate the methods, they do not measure any real model). The module is not published in this knowledge base; the functions assume its imports (`numpy as np`, `scipy.stats as stats`, `collections.Counter`, ...), listed in `calibration.md` under *ECE estimators and their bias*, and the end-to-end example reuses `fit_temperature` / `apply_temperature` and the simulator `simulate_choice` from the same module.

---

## Table of Contents

1. [The guarantee, precisely](#the-guarantee-precisely)
2. [The conformal quantile and its edge cases](#the-conformal-quantile-and-its-edge-cases)
3. [LAC: least ambiguous sets](#lac-least-ambiguous-sets)
4. [APS: adaptive prediction sets](#aps-adaptive-prediction-sets)
5. [RAPS: regularized APS](#raps-regularized-aps)
6. [Comparison on simulated data](#comparison-on-simulated-data)
7. [Class-conditional coverage](#class-conditional-coverage)
8. [Checking coverage](#checking-coverage)
9. [Exchangeability and drift](#exchangeability-and-drift)
10. [Acting on set sizes](#acting-on-set-sizes)
11. [A Jev policy end to end](#a-jev-policy-end-to-end)
12. [Checklist](#checklist)
13. [Sources](#sources)

---

## The guarantee, precisely

Split conformal: hold out n labelled calibration items, compute a **non-conformity score** s(x, y) for each (higher = the label fits worse), take the quantile q̂ below, and for a new x return C(x) = {y : s(x, y) ≤ q̂}. Then (Angelopoulos & Bates, Gentle Introduction, arXiv 2107.07511):

```
1 - α  ≤  P( Y_test ∈ C(X_test) )  ≤  1 - α + 1/(n + 1)
```

What this does and does not say:

- **Marginal**, averaged over calibration sets and test items. It is not per class, per customer, per language or per item; see [class-conditional coverage](#class-conditional-coverage).
- **Any model, any score**: validity does not need calibrated probabilities. Calibration (temperature first, see `calibration.md`) makes sets **smaller and more adaptive**, not more valid.
- The **upper bound needs continuous scores (no ties)**. Probabilities rounded to 0.01, as the hosted Jev API returns them, produce many ties; then coverage can exceed 1 - α + 1/(n + 1), i.e. sets are larger than necessary. Randomized scores (APS/RAPS with `rng`) break ties.
- **Conditional on one calibration set**, coverage is random: "P(Y_test ∈ C(X_test) | {(X_i, Y_i)}) ~ Beta(n + 1 - l, l)" with l = ⌊(n + 1)α⌋ (same paper). The authors note that "choosing n=1000 calibration points leads to coverage typically between .88 and .92" for α = 0.1.
- **Exchangeability** of calibration and test items is the only assumption, and drift breaks it (see below).

---

## The conformal quantile and its edge cases

q̂ is the ⌈(n + 1)(1 - α)⌉-th smallest calibration score. When that rank exceeds n there are too few items to certify the level: q̂ = ∞ and every set must contain every option.

```python
def conformal_quantile(scores, alpha):
    """The ceil((n+1)(1-alpha))-th smallest calibration score; +inf when that rank
    exceeds n (too few calibration items: the set must be everything)."""
    s = np.sort(np.asarray(scores, float))
    n = len(s)
    k = int(np.ceil((n + 1) * (1 - alpha)))
    return np.inf if k > n else float(s[k - 1])
```

The skill's LAC-specific version (score = 1 - p(true), so its "everything" value is 1.0 instead of ∞) is kept unchanged in the module and agrees with the generic one (tested):

```python
def conformal_qhat(p_true_label, alpha):
    """p_true_label: calibrated probability each calibration item gave its *true* label."""
    scores = np.sort(1.0 - np.asarray(p_true_label, float))
    n = len(scores)
    k = int(np.ceil((n + 1) * (1 - alpha)))
    return 1.0 if k > n else scores[k - 1]  # too few items: the set must be everything

def prediction_set(probabilities, qhat):
    return [option for option, p in probabilities.items() if 1.0 - p <= qhat]
```

Minimum calibration sizes. With α = 0.1, ⌈(n + 1)·0.9⌉ ≤ n first holds at n = 9 (`exp_conformal.py`: 5 and 8 items → ∞, 9 items → finite). In general n ≥ ⌈1/α⌉ - 1: 19 for α = 0.05, 99 for α = 0.01. That is the bare minimum for a finite set, not a recommendation: with n = 9, q̂ is the largest calibration score (l = 1), and coverage given that calibration set is Beta(9, 1)-distributed - mean 0.9, but P(coverage < 0.75) = 0.75⁹ ≈ 0.075 (scipy `beta(9, 1).cdf(0.75)`). In practice use hundreds (see [checking coverage](#checking-coverage) for the band at a given n).

The Gentle Introduction computes the same quantile with numpy as `np.quantile(cal_scores, np.ceil((n+1)*(1-alpha))/n, method='higher')`; `conformal_quantile` sorts instead, which gives the same value and handles the k > n case explicitly.

---

## LAC: least ambiguous sets

Score = 1 - p̂(true label) (Sadinle, Lei & Wasserman, *Least Ambiguous Set-Valued Classifiers with Bounded Error Levels*, JASA 2019, arXiv 1609.00451). The set keeps every option whose probability is at least 1 - q̂.

```python
def lac_scores(probs, labels):
    probs = np.asarray(probs, float)
    return 1.0 - probs[np.arange(len(labels)), labels]

def lac_sets(probs, qhat):
    """Boolean (n, k) membership matrix."""
    return (1.0 - np.asarray(probs, float)) <= qhat
```

Properties:

- Smallest **average** set size among valid set predictors, if the probabilities were the true conditional ones (Sadinle et al.'s oracle result: the optimal sets are level sets of the conditional class probabilities).
- **Can be empty**: when no option reaches 1 - q̂. Sadinle et al. note that "the optimal classifier can sometimes output the empty set" (abstract). An empty set is a useful signal (nothing fits) - see [acting on set sizes](#acting-on-set-sizes).
- **Uneven coverage**: easy classes are over-covered and hard/rare ones under-covered (table below).
- Used for LLM multiple-choice QA by Kumar et al., *Conformal Prediction with Large Language Models for Multi-Choice Question Answering* (arXiv 2305.18404), which reports that conformal uncertainty "is tightly correlated with prediction accuracy".

---

## APS: adaptive prediction sets

Score = total probability of the options ranked **at or above** the true one (Romano, Sesia & Candès, *Classification with Valid and Adaptive Coverage*, NeurIPS 2020, arXiv 2006.02544). The set adds options in descending probability until the mass reaches q̂. The Gentle Introduction writes the score as s(x, y) = Σ_{j=1..k} f̂(x)_{π_j(x)} where y = π_k(x), and implements it with:

```python
# verbatim, Angelopoulos & Bates, arXiv 2107.07511v6, Figure 3 ("Python code for adaptive prediction sets")
cal_pi = cal_smx.argsort(1)[:,::-1]; cal_srt = np.take_along_axis(cal_smx,cal_pi,axis=1).cumsum(axis=1)
cal_scores = np.take_along_axis(cal_srt,cal_pi.argsort(axis=1),axis=1)[range(n),cal_labels]
# Get the score quantile
qhat = np.quantile(cal_scores, np.ceil((n+1)*(1-alpha))/n, interpolation='higher')
```

The paper's code passes `interpolation='higher'`; NumPy renamed that argument to `method`, and the installed numpy 2.5.3 rejects the old name (`TypeError: quantile() got an unexpected keyword argument 'interpolation'`, checked 2026-10-03). Use `method='higher'`, as the paper's own Figure 2 already does. The same applies to the RAPS reference code's `pick_kreg` quoted below.

The module version, vectorised, with an optional randomisation:

```python
def _sorted_cumsum(probs):
    order = np.argsort(-probs, axis=1, kind="stable")
    srt = np.take_along_axis(probs, order, axis=1)
    return order, srt, np.cumsum(srt, axis=1)

def aps_scores(probs, labels, rng=None):
    """Mass of all classes ranked at or above the true one. With rng, the true
    class's own mass is multiplied by U~Uniform(0,1) (randomized APS: exact coverage)."""
    probs = np.asarray(probs, float)
    n = len(labels)
    order, srt, cum = _sorted_cumsum(probs)
    rank = np.argsort(order, axis=1)[np.arange(n), labels]  # 0-based rank of true class
    s = cum[np.arange(n), rank]
    if rng is not None:
        s = s - (1 - rng.random(n)) * srt[np.arange(n), rank]
    return s

def aps_sets(probs, qhat, rng=None):
    """Include classes in descending order while the cumulative mass (optionally
    randomized) stays <= qhat. Non-randomized sets always keep the top class, so
    they never come back empty; randomized sets can be empty."""
    probs = np.asarray(probs, float)
    n, k = probs.shape
    order, srt, cum = _sorted_cumsum(probs)
    if rng is None:
        # Gentle-intro rule: keep while cumulative mass <= qhat, so the true class is
        # covered exactly when its score <= qhat. Forcing the top class in only adds
        # coverage, never removes it.
        keep_sorted = cum <= qhat
        keep_sorted[:, 0] = True
    else:
        u = rng.random((n, 1))
        keep_sorted = (cum - (1 - u) * srt) <= qhat
    out = np.zeros_like(keep_sorted)
    np.put_along_axis(out, order, keep_sorted, axis=1)
    return out
```

- **Non-randomized** (`rng=None`): keep options while the cumulative mass ≤ q̂ (the Gentle Introduction's rule, so the true label is covered exactly when its score ≤ q̂), and always keep the top option (only adds coverage). These sets are conservative.
- **Randomized** (`rng` given): the true option's own mass is multiplied by U ~ Uniform(0, 1) in the score and in the set rule. This gives exact coverage in expectation and breaks ties from rounded probabilities; sets can then be empty.
- The RAPS reference code uses a different non-randomized convention: it includes the option that **crosses** q̂ (`sizes_base = ((cumsum + penalties_cumsum) <= tau).sum(axis=1) + 1` in `gcq`, `aangelopoulos/conformal_classification`, commit 881eba9). Both conventions are valid; theirs is larger.
- Adaptive: an easy item (one option with most of the mass) gets a small set, a hard item a large one - which LAC does less well.

---

## RAPS: regularized APS

APS sets get large when the model spreads mass thinly over many unlikely options. RAPS (Angelopoulos, Bates, Malik & Jordan, *Uncertainty Sets for Image Classifiers using Conformal Prediction*, ICLR 2021 spotlight, arXiv 2009.14193) adds a penalty λ · max(0, rank - k_reg) to the APS score, which stops sets from growing past k_reg options unless the mass really demands it; the paper reports sets "often factors of 5 to 10 smaller than a stand-alone Platt scaling baseline" on ImageNet (abstract).

```python
def raps_scores(probs, labels, lam=0.01, k_reg=1, rng=None):
    """APS score + lam * max(0, rank - k_reg), rank 1-based."""
    probs = np.asarray(probs, float)
    n = len(labels)
    order, srt, cum = _sorted_cumsum(probs)
    rank0 = np.argsort(order, axis=1)[np.arange(n), labels]
    s = cum[np.arange(n), rank0] + lam * np.maximum(0, rank0 + 1 - k_reg)
    if rng is not None:
        s = s - (1 - rng.random(n)) * srt[np.arange(n), rank0]
    return s

def raps_sets(probs, qhat, lam=0.01, k_reg=1, rng=None):
    probs = np.asarray(probs, float)
    n, k = probs.shape
    order, srt, cum = _sorted_cumsum(probs)
    pen = lam * np.maximum(0, np.arange(1, k + 1) - k_reg)[None, :]
    total = cum + pen
    if rng is None:
        keep_sorted = total <= qhat
        keep_sorted[:, 0] = True
    else:
        u = rng.random((n, 1))
        keep_sorted = (total - (1 - u) * srt) <= qhat
    out = np.zeros_like(keep_sorted)
    np.put_along_axis(out, order, keep_sorted, axis=1)
    return out
```

The cumulative penalty matches the reference code, which adds λ to every rank column from `kreg` on and takes `np.cumsum` (`self.penalties[:, kreg:] += lamda`, `penalties_cumsum = np.cumsum(penalties, axis=1)`).

Choosing k_reg and λ, as the reference code does (on a **tuning split** separate from the calibration split, 30% of the calibration data by default: `pct_paramtune = 0.3`):

- `pick_kreg`: `kstar = np.quantile(gt_locs_kstar, 1-alpha, interpolation='higher') + 1` - the (1 - α) quantile of the true label's 0-based rank, plus one.
- `pick_lamda_size`: try `[0.001, 0.01, 0.1, 0.2, 0.5]` ("predefined grid, change if more precision desired") and keep the λ with the smallest average set size.

`raps_pick_lambda` below differs from the reference in one respect: `pick_lamda_size` calibrates each candidate and measures its set size on the **same** tuning loader, whereas `raps_pick_lambda` calibrates on one half of the tuning data and measures size on the other half. Its docstring's "Reference-code rule" refers to the grid and the smallest-size criterion, not to that split.

```python
def raps_pick_kreg(probs, labels, alpha):
    """Reference-code rule (aangelopoulos/conformal_classification, pick_kreg): the
    (1 - alpha) quantile of the true label's 0-based rank, plus 1. Use a TUNING split,
    not the calibration split."""
    order = np.argsort(-np.asarray(probs, float), axis=1, kind="stable")
    rank0 = np.argsort(order, axis=1)[np.arange(len(labels)), labels]
    return int(np.quantile(rank0, 1 - alpha, method="higher")) + 1

def raps_pick_lambda(probs, labels, alpha, k_reg, grid=(0.001, 0.01, 0.1, 0.2, 0.5), seed=0):
    """Reference-code rule (pick_lamda_size): the grid value with the smallest mean set
    size, judged by an internal half/half split of the TUNING data."""
    probs = np.asarray(probs, float)
    labels = np.asarray(labels)
    idx = np.random.default_rng(seed).permutation(len(labels))
    a, b = idx[: len(idx) // 2], idx[len(idx) // 2 :]
    best, best_size = grid[0], np.inf
    for lam in grid:
        qhat = conformal_quantile(raps_scores(probs[a], labels[a], lam, k_reg), alpha)
        size = raps_sets(probs[b], qhat, lam, k_reg).sum(axis=1).mean()
        if size < best_size:
            best, best_size = lam, size
    return best
```

On a 5-option simulated model (`test_kb_calib.py`, tuning on 1,000 items) these picked k_reg = 3 and λ = 0.1. For decision models with only 2-6 options, RAPS behaves almost like APS (the penalty rarely binds); it matters with tens of options.

---

## Comparison on simulated data

6-option model whose logits carry a class-prior tilt log(0.40 / 0.25 / 0.15 / 0.10 / 0.06 / 0.04) (the realised label mix over all 12,000 simulated items is 0.282 / 0.221 / 0.174 / 0.134 / 0.108 / 0.082, flatter than the weights because the per-item logits are wide), reported probabilities over-confident (T = 2) and rounded to 0.01, then temperature-scaled on 1,000 separate items (fitted T = 1.45). 200 random splits, n_cal = 500, n_test = 2,000, α = 0.10 (`exp_conformal.py`):

| method | mean coverage | min / max over splits | mean set size | share of singletons | worst-class coverage (mean) |
|---|---|---|---|---|---|
| LAC | 0.897 | 0.849 / 0.931 | 1.86 | 0.400 | 0.818 |
| APS (non-randomized) | 0.897 | 0.851 / 0.944 | 2.25 | 0.013 | 0.818 |
| APS (randomized) | 0.896 | 0.834 / 0.940 | 1.99 | 0.257 | 0.819 |
| RAPS (λ = 0.05, k_reg = 1, non-randomized) | 0.898 | 0.864 / 0.944 | 2.25 | 0.001 | 0.816 |
| LAC, class-conditional | 0.902 | 0.863 / 0.953 | 2.21 | 0.198 | 0.857 |

Expected 95% band for the empirical coverage of a correct pipeline at n_cal = 500, n_test = 2,000: [0.869, 0.928] (`conformal_coverage_band`). Of the 200 splits, 1% (APS) to 6.5% (randomized APS) fell outside it - LAC 3%, RAPS 1.5%, class-conditional LAC 3% (re-run of `exp_conformal.py`, 2026-10-03); the conservative non-randomized methods leave the band less often. The means sit at 0.90.

- **LAC gives the smallest sets** and by far the most singletons. With a well-calibrated model it is the default.
- **Non-randomized APS/RAPS almost never return a single option** here: the rule keeps the second option whenever the top two options together carry at most q̂, and with q̂ ≈ 0.92 (one split, APS) that holds for about 97% of test items, so a singleton needs the top two to exceed q̂ - which is rare. Randomization fixes most of that (25.7% singletons). If you want "act alone" decisions, use LAC or randomized APS.
- The mean coverages under 0.90 (0.896-0.898) are within Monte-Carlo noise of 0.90 over 200 splits; the test suite asserts ≥ 0.89 for each method.

---

## Class-conditional coverage

Marginal coverage can hide a class that is covered far less often. Averaged over the 200 splits above, LAC's worst class was covered only 81.8% of the time; on one split in detail, per-class coverage ran 0.943 / 0.943 / 0.914 / 0.875 / 0.845 / 0.857 from class 0 to class 5 (prior weight 0.40 down to 0.04; the three rarer classes are the ones under-covered). If the rare class is the expensive one ("legal threat", "fraud"), that is the wrong way round.

**Mondrian (class-conditional) conformal**: one quantile per **true** class, computed from that class's calibration items only; option c enters the set when its own score passes its own q̂_c. Each class then gets ≥ 1 - α coverage, at the price of larger sets and a per-class minimum of ⌈1/α⌉ - 1 calibration items.

```python
def class_conditional_qhats(scores, labels, alpha, n_classes):
    """One quantile per true class, from that class's calibration items only."""
    scores = np.asarray(scores, float)
    labels = np.asarray(labels)
    return np.array([conformal_quantile(scores[labels == c], alpha) for c in range(n_classes)])

def class_conditional_lac_sets(probs, qhats):
    """Class c is in the set when 1 - p_c <= qhat_c."""
    return (1.0 - np.asarray(probs, float)) <= np.asarray(qhats)[None, :]
```

Same split: per-class quantiles 0.849 / 0.923 / 0.910 / 0.963 / 0.977 / 0.968 from 148 / 114 / 84 / 59 / 41 / 54 calibration items; per-class coverage 0.901 / 0.943 / 0.911 / 0.915 / 0.941 / 0.929; mean set size 2.24 (vs 1.92 for marginal LAC). With 41 items the rare-class quantile is noisy - the per-class Beta band is wide - so give rare, costly classes more labels, or merge classes for the conformal step.

The same idea extends to any **group** you can see at prediction time (language, channel, customer tier): compute q̂ per group. You cannot condition on something you only learn later.

---

## Checking coverage

`coverage_report` gives everything to look at for one evaluation: marginal coverage, mean size, the size histogram, per-class coverage and **coverage by set size** (the Gentle Introduction's size-stratified coverage: group items by set cardinality and look at the worst group).

```python
def coverage_report(sets, labels):
    sets = np.asarray(sets, bool)
    labels = np.asarray(labels)
    n, k = sets.shape
    covered = sets[np.arange(n), labels]
    sizes = sets.sum(axis=1)
    per_class = {c: float(covered[labels == c].mean()) for c in range(k) if (labels == c).any()}
    return {
        "coverage": float(covered.mean()),
        "mean_size": float(sizes.mean()),
        "size_histogram": dict(sorted(Counter(sizes.tolist()).items())),
        "per_class_coverage": per_class,
        "coverage_by_size": {int(s): float(covered[sizes == s].mean()) for s in np.unique(sizes)},
    }
```

Whether an observed coverage is "fine" depends on n_cal and n_test. Coverage given the calibration set is Beta(n + 1 - l, l); the observed frequency on n_test items is then beta-binomial:

```python
def conformal_coverage_band(n_cal, alpha, n_test, level=0.95):
    """Range the EMPIRICAL test coverage should fall in for a correct split-conformal
    pipeline: coverage given the calibration set ~ Beta(n+1-l, l), l = floor((n+1)alpha)
    (Angelopoulos & Bates, Gentle Introduction, Sec. 3); the test frequency is then
    beta-binomial."""
    l = int(np.floor((n_cal + 1) * alpha))
    if l < 1:
        return (1.0, 1.0)
    dist = stats.betabinom(n_test, n_cal + 1 - l, l)
    lo, hi = dist.ppf([(1 - level) / 2, 1 - (1 - level) / 2])
    return (lo / n_test, hi / n_test)
```

Examples: n_cal = 500, α = 0.1, n_test = 2,000 → [0.869, 0.928] at 95%. With too few calibration items (l < 1) the function returns (1.0, 1.0): q̂ is infinite and every set is everything.

A production monitor - label a sample of production items after the fact (human review, downstream outcome), record whether the true label was in the set, and alarm when the rate leaves the band:

```python
def coverage_alarm(covered_flags, n_cal, alpha=ALPHA, level=0.99):
    """covered_flags: 1/0 per recent LABELLED production item (true label in the set?).
    Alarm when the observed coverage leaves the band a correct pipeline produces."""
    n = len(covered_flags)
    lo, hi = conformal_coverage_band(n_cal, alpha, n, level)
    observed = float(np.mean(covered_flags))
    return {"observed": observed, "band": (lo, hi), "alarm": not lo <= observed <= hi}


status = coverage_alarm(np.random.default_rng(5).random(400) < 0.84, n_cal=500)
```

Executed with 400 simulated items covered at 84% against a calibration of 500: band [0.845, 0.948] at 99%, observed 0.840 → alarm. Note the band is computed for the **test** sample size you actually have; 400 labels give a much wider band than 2,000.

---

## Exchangeability and drift

The guarantee needs calibration and production items to be exchangeable. Calibrate on traffic A, serve traffic B (`exp_conformal.py`, LAC, q̂ from 500 calibration items, 4,000 test items):

| traffic B | coverage | mean set size |
|---|---|---|
| same mix | 0.899 | 1.89 |
| class mix reversed (rare classes become common) | 0.910 | 1.91 |
| harder items (content signal × 0.6) | 0.874 | 2.36 |
| model more over-confident on B (T 2.0 → 3.5) | 0.838 | 1.49 |
| harder + more over-confident | 0.786 | 1.77 |

- A pure **label-mix** shift barely moved marginal coverage here (it did move per-class coverage); a **difficulty** shift cost ~2.5 points; a change in the **model's own miscalibration** - a new model version, a new language, a reworded question - cost 6-11 points while the sets got **smaller**. Smaller sets plus falling coverage is the signature to alarm on.
- Kumar et al. (arXiv 2305.18404) "investigate the exchangeability assumption required by conformal prediction to out-of-subject questions" for LLM MCQA - i.e. calibrating on one subject and testing on another - as a realistic stress case (abstract).
- For TypeSafe Jev in particular the vendor lists the model's known failure areas on the jaggedness page (https://docs.typesafe.ai/model-jaggedness/jev-1.13): accuracy shifts as the state grows ("context rot"), dates, counting, option order. A traffic change toward any of these is a drift event; so is any change of `model`, instructions, options, option order or names.

What to do:

1. **Recalibrate** (q̂ and the temperature beneath it) on recent labelled traffic when the monitor alarms, and on every model/wording change.
2. **Stratify** by any observable variable that shifts (language, channel): one q̂ per group.
3. For known covariate shift, weighted conformal prediction re-weights calibration items by the likelihood ratio between the two input distributions (Tibshirani, Barber, Candès & Ramdas, *Conformal Prediction Under Covariate Shift*, NeurIPS 2019, arXiv 1904.06019) - not implemented or tested here.

---

## Acting on set sizes

The skill's rule (singleton → act, several → ask, empty → out of distribution) is right, with one correction the data makes clear: **a singleton is not "certain"**. Conformal guarantees coverage averaged over all items, not within singletons. One split, LAC, α = 0.1 (`exp_conformal.py`):

| set size | share of items | top-1 accuracy | true label in set |
|---|---|---|---|
| 1 | 0.383 | 0.899 | 0.899 |
| 2 | 0.378 | 0.634 | 0.910 |
| 3 | 0.184 | 0.499 | 0.919 |
| 4 | 0.050 | 0.394 | 0.960 |
| 5 | 0.005 | 0.091 | 0.909 |
| 6 | 0.001 | 1.000 | 1.000 |

Singletons were right 89.9% of the time - about 1 - α, not 99%. So:

- **Pick α from the error you accept on automatic actions**, not from the coverage you want on review queues. If a singleton action may be wrong at most 2% of the time, either use a much smaller α (larger sets, fewer singletons) or gate singletons additionally with a calibrated threshold (`lowest_bar_for` in `calibration.md`), or use a risk-controlling method that targets the error of the action directly (e.g. *Conformal Risk Control*, Angelopoulos, Bates, Fisch, Lei & Schuster, arXiv 2208.02814 - not implemented here).
- **Two or three options**: show exactly these options to the person or the fallback model, in probability order. The set is the useful output - it narrows the choice while keeping the truth inside ~91-92% of the time here.
- **Empty set** (LAC or randomized APS): no option fits as well as calibration items usually did. Treat as out of distribution: escalate, and log it - a rising empty-set rate is an early drift signal.
- **Everything** (set = all options): the model has no usable signal on this item; the decision model added nothing, route it as if it were not there.
- Track the **hand-off rate** (share of non-singletons) next to coverage: a policy that sends 60% of traffic to review may cost more than having no decision model at all.

---

## A Jev policy end to end

Fit a temperature on half of the labelled calibration items, the LAC quantile on the other half (never both on the same items), then act. Executed offline on simulated Jev-shaped answers (`probabilities` dicts):

```python
ALPHA = 0.10
OPTIONS = ["billing", "technical", "sales", "other"]


def build_conformal_policy(cal_answers, cal_labels, alpha=ALPHA):
    """cal_answers: list of Jev `probabilities` dicts; cal_labels: list of option names.
    Fit the temperature on one half, the conformal quantile on the other."""
    P = np.array([[a[o] for o in OPTIONS] for a in cal_answers])
    y = np.array([OPTIONS.index(l) for l in cal_labels])
    half = len(y) // 2
    t = fit_temperature(P[:half], y[:half])
    Q = apply_temperature(P[half:], t)
    qhat = conformal_qhat(Q[np.arange(len(Q)), y[half:]], alpha)
    return t, qhat


def act(answer_probs, t, qhat):
    q = apply_temperature(np.array([[answer_probs[o] for o in OPTIONS]]), t)[0]
    options = prediction_set(dict(zip(OPTIONS, q)), qhat)
    if len(options) == 1:
        return "auto", options
    if not options:
        return "out_of_distribution", options
    return "ask", options  # show these options to a person, in this order of probability


P_sim, y_sim, _ = simulate_choice(1000, k=4, true_t=2.0, rng=np.random.default_rng(4))
t, qhat = build_conformal_policy([dict(zip(OPTIONS, r)) for r in P_sim],
                                 [OPTIONS[i] for i in y_sim])
decision = act({"billing": 0.86, "technical": 0.11, "sales": 0.03, "other": 0.0}, t, qhat)
```

Output: T = 1.71, q̂ = 0.895, and for `{"billing": 0.86, "technical": 0.11, "sales": 0.03, "other": 0.0}` the decision `("ask", ["billing", "technical"])` - after temperature scaling `technical` keeps enough probability (≥ 1 - q̂ ≈ 0.105) to stay in the set, so the item goes to a person with two options instead of one.

Add order debiasing (`order-bias.md`) **before** this step if the question shows option-order flips: the conformal quantile should be computed on the same (debiased) probabilities you serve.

---

## Checklist

- [ ] Temperature (or Platt) fitted on one split, conformal quantile on another, coverage checked on a third
- [ ] n_cal ≥ several hundred; per-class n ≥ ⌈1/α⌉ - 1 if using class-conditional quantiles
- [ ] Method chosen: LAC by default; randomized APS when adaptivity matters; RAPS only with many options
- [ ] Coverage report on held-out data: marginal inside the beta-binomial band, per-class, by set size
- [ ] α chosen from the acceptable error of **automatic** actions; singleton accuracy checked, not assumed
- [ ] Production monitor on labelled samples; alarm on coverage outside the band and on falling set sizes
- [ ] Recalibrate on any model, wording, option, order or traffic-mix change

---

## Sources

- Angelopoulos & Bates, *A Gentle Introduction to Conformal Prediction and Distribution-Free Uncertainty Quantification* - arXiv 2107.07511 (v6, fetched 2026-10-03)
- Sadinle, Lei & Wasserman, *Least Ambiguous Set-Valued Classifiers with Bounded Error Levels*, JASA 2019 - arXiv 1609.00451
- Romano, Sesia & Candès, *Classification with Valid and Adaptive Coverage*, NeurIPS 2020 - arXiv 2006.02544
- Angelopoulos, Bates, Malik & Jordan, *Uncertainty Sets for Image Classifiers using Conformal Prediction*, ICLR 2021 - arXiv 2009.14193; code https://github.com/aangelopoulos/conformal_classification (`conformal.py`, commit 881eba9, read 2026-10-03)
- Kumar, Lu, Gupta, Palepu, Bellamy, Raskar & Beam, *Conformal Prediction with Large Language Models for Multi-Choice Question Answering*, 2023 - arXiv 2305.18404
- Tibshirani, Barber, Candès & Ramdas, *Conformal Prediction Under Covariate Shift*, NeurIPS 2019 - arXiv 1904.06019 (pointer only)
- Angelopoulos, Bates, Fisch, Lei & Schuster, *Conformal Risk Control* - arXiv 2208.02814 (pointer only)
- TypeSafe: https://docs.typesafe.ai/api (response shape), https://docs.typesafe.ai/model-jaggedness/jev-1.13 (fetched 2026-10-03)
