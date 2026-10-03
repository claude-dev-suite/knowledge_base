# Calibrating Decision-Model Probabilities - Deep Reference

> Official Documentation: https://scikit-learn.org/stable/modules/calibration.html
> Official Documentation: https://docs.typesafe.ai/confidence
> Reference implementation (debiased estimator): https://github.com/p-lambda/verified_calibration
> Last verified: 2026-10-03
> Verified against: Python 3.12.10, numpy 2.5.3, scipy 1.18.1, scikit-learn 1.9.1, typesafe-sdk 0.7.2 (PyPI), TypeSafe docs for `jev-1.13`

## Overview

This page is the long-form companion of the dev-suite skill `ai-integration/decision-model-calibration`. The skill gives the procedure; this page gives the estimators behind it, what each one gets wrong at small sample sizes, which recalibration map fits which failure, and the numbers from simulations that show it. It applies to any classifier that returns class probabilities: TypeSafe Jev (`probabilities` on Choice/Score, `noul` on Noul), an LLM read through logprobs (see `provider-logprobs.md`), or your own scikit-learn model.

Every function on this page is copied verbatim from the companion module `kb_calib.py` (and the longer examples from its `doc_examples.py`); all of them were executed offline on simulated data with a test suite (29 tests, all passing on 2026-10-03). The module itself is **not** published in this knowledge base: the imports and the private `_equal_mass_bins` helper the functions rely on are shown at the start of [ECE estimators and their bias](#ece-estimators-and-their-bias), and the sibling pages assume the same imports. Nothing on this page called a real API. Numbers labelled "simulation" come from `exp_calibration.py` / `exp_extra.py` in the same folder, with fixed seeds; they illustrate behaviour, they are not measurements of Jev or any other model.

Sibling pages: `order-bias.md` (position and label bias), `conformal.md` (prediction sets), `provider-logprobs.md` (which providers expose probabilities at all).

---

## Table of Contents

1. [What exactly gets calibrated](#what-exactly-gets-calibrated)
2. [ECE estimators and their bias](#ece-estimators-and-their-bias)
3. [The noise floor, by simulation](#the-noise-floor-by-simulation)
4. [Proper scores: Brier, log loss and the Brier decomposition](#proper-scores-brier-log-loss-and-the-brier-decomposition)
5. [Reliability diagrams](#reliability-diagrams)
6. [Recalibration maps and when each fits](#recalibration-maps-and-when-each-fits)
7. [The rounding floor: 0.005 vs 1e-6](#the-rounding-floor-0005-vs-1e-6)
8. [scikit-learn 1.9: method="temperature" and FrozenEstimator](#scikit-learn-19-methodtemperature-and-frozenestimator)
9. [Sample-size guidance](#sample-size-guidance)
10. [Per-question fitting](#per-question-fitting)
11. [Bootstrap confidence intervals](#bootstrap-confidence-intervals)
12. [Contextual calibration (Zhao et al. 2021)](#contextual-calibration-zhao-et-al-2021)
13. [Checklist](#checklist)
14. [Sources](#sources)

---

## What exactly gets calibrated

A model is **calibrated** when, among all items it gives probability p, a fraction p is correct. There are three increasingly strict versions, and the metric you pick decides which one you are checking:

| Notion | Condition | Metric that checks it |
|---|---|---|
| **Top-label (confidence) calibration** | P(prediction correct \| top probability = p) = p | ECE on `(max prob, correct)` - all ECE functions below take this pair |
| **Classwise calibration** | for every class j: P(y = j \| p_j = q) = q | `classwise_ece` (Kull et al., NeurIPS 2019); Nixon et al. 2019 call the equal-width/adaptive versions SCE/ACE |
| **Canonical (full-vector) calibration** | P(y \| p-vector) = p-vector | not estimable with binning beyond a few classes; proper scores (Brier, log loss) are the practical proxy |

What to feed in, per primitive:

- **Jev Choice / Score**: the `probabilities` map, in a fixed option order (`to_matrix` below). Confidence for top-label metrics is the max probability.
- **Jev Noul**: the `noul` value is P(yes). For top-label metrics use `max(p, 1 - p)` and correctness at your threshold; for fitting, use `p` against the 0/1 label.
- **Never Jev's `confidence` field.** TypeSafe's Confidence page defines it as a function of the distribution, not a probability of being right: for a Choice the page's own code computes `(count * peak - 1) / (count - 1)` with `peak` the top probability; for a Score it is a spread measure around the peak level (https://docs.typesafe.ai/confidence, interactive explorer source, fetched 2026-10-03). A bar on `confidence` is an ordering until it is checked against labels.

**Rounding.** The hosted API's documented examples carry two decimals (`{"billing": 0.88, "technical": 0.12, "sales": 0.0}`, https://docs.typesafe.ai/api, fetched 2026-10-03), and the `typed-decision-models` skill's evidence section reports that probabilities are rounded to 0.01 and Choice/Score can return exact `0` or `1`. That matters for every log-based fit - see [the rounding floor](#the-rounding-floor-0005-vs-1e-6). The self-hosted, TypeSafe-compatible endpoint in llama.cpp (`/v1/systemone`, see `provider-logprobs.md`) shows four decimals in its README example and says the values there are "shortened"; whether it rounds at all was not verified.

```python
def to_matrix(answers, options):
    """[{'billing': 0.88, ...}, ...] -> (n, k) array in a fixed option order."""
    return np.array([[a[o] for o in options] for a in answers], float)

def top_label(probs):
    """(confidence, predicted index) for each row of an (n, k) matrix."""
    probs = np.asarray(probs, float)
    return probs.max(axis=1), probs.argmax(axis=1)

def round_like_jev(probs, quantum=0.01):
    """Round to the API's quantum (0.01), as the hosted model does. Rows may no
    longer sum to exactly 1, and small classes become exact zeros."""
    return np.round(np.asarray(probs, float) / quantum) * quantum
```

---

## ECE estimators and their bias

All binned ECE estimators share one form: sort items into bins, and average |accuracy - mean confidence| per bin, weighted by bin mass. They differ in how bins are made and whether the sampling noise inside a bin is subtracted.

| Estimator | Bins | Bias | Source |
|---|---|---|---|
| `ece_equal_width` | 15 bins of width 1/15 (Guo et al. 2017 use M = 15) | positive, large at small n; empty and near-empty bins at the extremes | Guo et al., ICML 2017 (arXiv 1706.04599) |
| `ece` (equal-mass) | 10 bins with the same number of items | still positive; in the simulation below slightly lower than equal-width on the calibrated model and slightly higher on the over-confident one | Nixon et al. 2019 (arXiv 1904.01685) recommend adaptive bins; Roelofs et al. find equal-mass less biased |
| `ece_sweep` | equal-mass, the **largest** bin count that keeps per-bin accuracy monotone | lowest of the L1 estimators here | Roelofs, Cain, Shlens & Mozer, AISTATS 2022 (arXiv 2012.08668) |
| `debiased_l2_ce` | equal-mass; subtracts the per-bin binomial variance from the squared gap | close to unbiased for the L2 error | Kumar, Liang & Ma, NeurIPS 2019 (arXiv 1909.10155); same formula as `unbiased_l2_ce` in `p-lambda/verified_calibration` |
| `l2_ce` | equal-mass, plug-in L2 | the biased L2 counterpart, for comparison | - |
| `classwise_ece` | per class, equal-mass or equal-width | as its base estimator, averaged over classes | Kull et al., NeurIPS 2019 (arXiv 1910.12656) |

Roelofs et al. summarise the comparison in their abstract: "binning-based estimators with bins of equal mass (number of instances) have lower bias than estimators with bins of equal width", and they recommend "the debiased estimator (Brocker, 2012; Ferro and Fricker, 2012) and a method we propose, ECE_sweep" (arXiv 2012.08668v3 abstract, fetched 2026-10-03).

The Kumar et al. debiasing, per bin b with n_b items, mean accuracy a_b and mean confidence c_b:

```
CE^2 ≈ Σ_b (n_b / n) · [ (a_b - c_b)^2  -  a_b (1 - a_b) / (n_b - 1) ]      then clip at 0 and take the square root
```

The subtracted term is the expected squared error that a perfectly calibrated bin shows from binomial noise alone. The reference library's `unbiased_square_ce` gives bins with fewer than 2 items a 0 contribution; ours does the same.

The module's imports (used by the functions on all four pages) and its equal-mass binning helper, verbatim from `kb_calib.py`:

```python
import math
from collections import Counter

import numpy as np
from scipy import stats
from scipy.optimize import minimize, minimize_scalar
from scipy.special import expit, log_softmax, logit, softmax
```

```python
def _equal_mass_bins(confidence, n_bins):
    order = np.argsort(np.asarray(confidence, float), kind="stable")
    return [b for b in np.array_split(order, n_bins) if len(b)]
```

```python
def ece(confidence, correct, n_bins=10):
    """Expected calibration error with equal-mass bins (less biased than equal-width)."""
    confidence = np.asarray(confidence, float)
    correct = np.asarray(correct, float)
    order = np.argsort(confidence, kind="stable")
    total = len(confidence)
    return sum(
        len(b) / total * abs(correct[b].mean() - confidence[b].mean())
        for b in np.array_split(order, n_bins)
        if len(b)
    )

def ece_equal_width(confidence, correct, n_bins=15):
    """Guo et al. (2017) ECE: n_bins bins of equal width on [0, 1]. Empty bins are skipped."""
    confidence = np.asarray(confidence, float)
    correct = np.asarray(correct, float)
    idx = np.minimum((confidence * n_bins).astype(int), n_bins - 1)
    total = len(confidence)
    out = 0.0
    for b in range(n_bins):
        m = idx == b
        if m.any():
            out += m.sum() / total * abs(correct[m].mean() - confidence[m].mean())
    return out

def ece_sweep(confidence, correct, p=1):
    """ECE_sweep (Roelofs et al., AISTATS 2022): equal-mass bins, using the LARGEST
    number of bins for which per-bin accuracy is still monotone in confidence."""
    confidence = np.asarray(confidence, float)
    correct = np.asarray(correct, float)
    n = len(confidence)
    best = 1
    for b in range(1, n + 1):
        bins = _equal_mass_bins(confidence, b)
        accs = [correct[i].mean() for i in bins]
        if np.all(np.diff(accs) >= 0):
            best = b
        else:
            break
    bins = _equal_mass_bins(confidence, best)
    err = sum(len(i) / n * abs(correct[i].mean() - confidence[i].mean()) ** p for i in bins)
    return err ** (1.0 / p), best

def debiased_l2_ce(confidence, correct, n_bins=10):
    """Debiased L2 calibration error (Kumar, Liang & Ma, NeurIPS 2019), equal-mass bins.

    Per bin: (mean_acc - mean_conf)^2 - mean_acc(1 - mean_acc)/(n_b - 1), weighted
    by bin mass, clipped at 0, then square-rooted. Same estimator as
    `unbiased_l2_ce` in github.com/p-lambda/verified_calibration (bins with < 2
    items contribute 0 there too).
    """
    confidence = np.asarray(confidence, float)
    correct = np.asarray(correct, float)
    n = len(confidence)
    total = 0.0
    for i in _equal_mass_bins(confidence, n_bins):
        if len(i) < 2:
            continue
        a = correct[i].mean()
        total += len(i) / n * ((a - confidence[i].mean()) ** 2 - a * (1 - a) / (len(i) - 1))
    return max(total, 0.0) ** 0.5

def l2_ce(confidence, correct, n_bins=10):
    """Plug-in (biased) L2 calibration error on the same equal-mass bins."""
    confidence = np.asarray(confidence, float)
    correct = np.asarray(correct, float)
    n = len(confidence)
    return sum(
        len(i) / n * (correct[i].mean() - confidence[i].mean()) ** 2
        for i in _equal_mass_bins(confidence, n_bins)
    ) ** 0.5

def classwise_ece(probs, labels, n_bins=10, equal_mass=True):
    """Classwise ECE (Kull et al., NeurIPS 2019; Nixon et al. 2019 'SCE'/'ACE'):
    the mean over classes of the ECE of 'P(class j)' vs 'is class j'. With
    equal_mass=True this is the adaptive variant (ACE without thresholding)."""
    probs = np.asarray(probs, float)
    labels = np.asarray(labels)
    k = probs.shape[1]
    fn = ece if equal_mass else ece_equal_width
    return float(np.mean([fn(probs[:, j], (labels == j).astype(float), n_bins) for j in range(k)]))
```

### How biased are they? (simulation)

Mean over 300 draws per row. Confidences ~ U(0.3, 1.0) for the calibrated case, U(0.4, 1.0) with accuracy = confidence - 0.10 for the over-confident case. True calibration error is 0 and 0.100 respectively (L1 and L2 coincide for a constant gap). Source: `exp_calibration.py`, sections 1 and 1b, 2026-10-03.

**Perfectly calibrated model (true CE = 0):**

| n | equal-width (15) | equal-mass (10) | sweep | L2 plug-in | L2 debiased |
|---|---|---|---|---|---|
| 50 | 0.155 | 0.153 | 0.083 | 0.188 | 0.054 |
| 100 | 0.113 | 0.109 | 0.065 | 0.134 | 0.037 |
| 300 | 0.065 | 0.061 | 0.042 | 0.077 | 0.021 |
| 1000 | 0.034 | 0.033 | 0.028 | 0.041 | 0.010 |
| 3000 | 0.020 | 0.019 | 0.019 | 0.024 | 0.006 |

**Over-confident model (true CE = 0.100):**

| n | equal-width (15) | equal-mass (10) | sweep | L2 plug-in | L2 debiased |
|---|---|---|---|---|---|
| 50 | 0.173 | 0.180 | 0.114 | 0.221 | 0.089 |
| 100 | 0.136 | 0.141 | 0.109 | 0.172 | 0.085 |
| 300 | 0.106 | 0.108 | 0.100 | 0.128 | 0.094 |
| 1000 | 0.101 | 0.101 | 0.101 | 0.110 | 0.100 |
| 3000 | 0.100 | 0.100 | 0.100 | 0.103 | 0.100 |

Reading it:

- At n = 50 a perfect model shows a plug-in ECE of ~0.15 - larger than the true error of the bad model. **Comparing two raw ECEs at n < 300 says little**; compare each against its noise floor, or use the sweep/debiased estimators.
- The plug-in bias is **additive noise**, so it shrinks the gap between a good and a bad model at small n (0.153 vs 0.180 at n = 50) and vanishes by n ≈ 1,000.
- The debiased L2 estimator is slightly **negative**-biased on the bad model at small n (0.089 at n = 50): the bracketed sum is a (nearly) unbiased estimate of CE², but its square root is biased downward (Jensen's inequality), and the noise of the subtracted variance term adds to that. The clip at 0 pushes the other way, which is why the same estimator still reads 0.054 at n = 50 on the perfectly calibrated model. That is the expected price of debiasing, not a bug.
- The equal-width vs equal-mass difference is small here because confidences are spread uniformly; it grows when confidences pile up near 1.0, which is the usual case for a confident model: equal-width then leaves most bins nearly empty.

Recommendation: report **equal-mass ECE with its noise floor** (comparable to most published numbers) and **debiased L2** (closest to the truth); quote `ece_sweep`'s chosen bin count when you use it.

---

## The noise floor, by simulation

The noise floor answers: what ECE would a perfectly calibrated model show **on this many items, with these confidences**? Simulate outcomes as Bernoulli(confidence), recompute ECE, repeat.

```python
def ece_noise_floor(confidence, n_bins=10, n_sims=2000, seed=0, estimator=None):
    """ECE a *perfectly calibrated* model would show on this many items with these
    confidences. A measured ECE inside this band is indistinguishable from noise.
    Returns the (median, 95th percentile)."""
    estimator = estimator or ece
    rng = np.random.default_rng(seed)
    confidence = np.asarray(confidence, float)
    sims = [estimator(confidence, rng.random(confidence.size) < confidence, n_bins) for _ in range(n_sims)]
    return np.percentile(sims, [50, 95])
```

The floor depends on the confidence profile, not just on n. Median / 95th percentile of equal-mass ECE (10 bins), 1,000 simulations each (`exp_extra.py`, 2026-10-03):

| n | confidences ~ U(0.4, 1.0) | confidences ~ Beta(20, 1) (mean ≈ 0.95) |
|---|---|---|
| 30 | 0.191 / 0.265 | 0.071 / 0.126 |
| 60 | 0.130 / 0.185 | 0.054 / 0.089 |
| 100 | 0.099 / 0.141 | 0.047 / 0.069 |
| 200 | 0.072 / 0.104 | 0.033 / 0.050 |
| 500 | 0.045 / 0.065 | 0.021 / 0.032 |
| 1000 | 0.032 / 0.048 | 0.015 / 0.022 |

A confident model has a lower floor because Bernoulli noise p(1 - p) is small near 1. The skill's "at n = 60 a perfectly calibrated model already scores ≈ 0.045" is in line with the confident profile (0.054 here); for a model whose confidences spread from 0.4 to 1.0 the floor at n = 60 is ~0.13. Always compute the floor **from your own confidence vector**, which is exactly what `ece_noise_floor(confidence)` does.

Interpretation rule: a measured ECE below the floor's 95th percentile is consistent with perfect calibration at that n. It does not prove calibration; it means the data cannot show miscalibration.

---

## Proper scores: Brier, log loss and the Brier decomposition

ECE only looks at the top label and is not a proper scoring rule: a model can lower ECE by predicting the base rate for everything. Report a proper score next to it.

- **Brier** (multiclass): mean over items of Σ_j (p_j - 1[y = j])². Range 0 to 2. Rewards both calibration and discrimination; insensitive to tiny probabilities.
- **Log loss / NLL**: mean of -log p(true). Punishes a confident wrong answer without bound - which is exactly why it needs a floor when the API returns exact zeros (`nll` uses the same `FLOOR` as the fits).

```python
def brier(probs, labels):
    """Multiclass Brier score; probs is (n, k), labels are column indices. Range [0, 2]."""
    probs = np.asarray(probs, float)
    onehot = np.eye(probs.shape[1])[labels]
    return np.mean(np.sum((probs - onehot) ** 2, axis=1))

def nll(probs, labels, floor=FLOOR):
    """Mean negative log-likelihood of the true class (log loss), with a floor for zeros."""
    probs = np.asarray(probs, float)
    p = np.clip(probs[np.arange(len(labels)), labels], floor, 1.0)
    return float(-np.mean(np.log(p)))
```

### The decomposition

For a binary forecast grouped into K bins (n_k items, mean forecast f̄_k, observed frequency ō_k, overall frequency ō), Murphy's (1973) decomposition is `BS = REL - RES + UNC`, with REL = Σ n_k (f̄_k - ō_k)² / n (miscalibration), RES = Σ n_k (ō_k - ō)² / n (how far bins separate from the base rate), UNC = ō (1 - ō). It is exact only when every forecast in a bin is identical. With continuous forecasts two extra terms appear (Stephenson, Coelho & Jolliffe, *Weather and Forecasting* 23(4), 2008): within-bin variance WBV = Σ (f_i - f̄_k)² / n and within-bin covariance WBC = Σ (f_i - f̄_k)(o_i - ō_k) / n, giving the exact identity

```
BS = REL - RES + UNC + WBV - 2·WBC
```

(derived by expanding (f_i - o_i)² around the bin means; the test suite checks it to 1e-9).

```python
def brier_decomposition(p_yes, y, n_bins=10):
    """Binary Brier score = REL - RES + UNC + WBV - 2*WBC (exact identity).

    REL/RES/UNC: Murphy (1973) on equal-mass bins; WBV/WBC: within-bin variance and
    covariance (Stephenson et al., Weather and Forecasting 2008), which are zero
    only when every forecast in a bin is identical."""
    p = np.asarray(p_yes, float)
    y = np.asarray(y, float)
    n = len(p)
    obar = y.mean()
    rel = res = wbv = wbc = 0.0
    for i in _equal_mass_bins(p, n_bins):
        fk, ok = p[i].mean(), y[i].mean()
        rel += len(i) * (fk - ok) ** 2
        res += len(i) * (ok - obar) ** 2
        wbv += np.sum((p[i] - fk) ** 2)
        wbc += np.sum((p[i] - fk) * (y[i] - ok))
    out = {
        "brier": float(np.mean((p - y) ** 2)),
        "reliability": rel / n,
        "resolution": res / n,
        "uncertainty": obar * (1 - obar),
        "within_bin_variance": wbv / n,
        "within_bin_covariance": wbc / n,
    }
    return out
```

Simulation (`exp_calibration.py` 5b, n = 5,000, a yes/no model compressed toward 0.5 with slope 0.5):

| | Brier | REL | RES | UNC | WBV | WBC |
|---|---|---|---|---|---|---|
| raw | 0.1446 | 0.0114 | 0.1160 | 0.2499 | 0.0008 | 0.0007 |
| after Platt (fit on 1,000) | 0.1335 | 0.0005 | 0.1160 | 0.2499 | 0.0010 | 0.0009 |

A monotone recalibration moves **REL** and leaves **RES** essentially untouched: it cannot make the model discriminate better, only say what it knows honestly. If Brier is bad because RES is low, no calibration map helps - you need a better question, better state, or a better model.

---

## Reliability diagrams

A reliability diagram plots per-bin accuracy against per-bin confidence. With small n, always draw the per-bin interval; without it, noise looks like structure. `ascii_reliability` prints one in a terminal or CI log: `|` marks the bin's mean confidence, `#` the accuracy bar, `-` the Wilson 95% interval.

```python
def reliability_table(confidence, correct, n_bins=10, equal_mass=True):
    """Rows of (lo, hi, count, mean_confidence, accuracy, wilson_lo, wilson_hi)."""
    confidence = np.asarray(confidence, float)
    correct = np.asarray(correct, float)
    if equal_mass:
        groups = _equal_mass_bins(confidence, n_bins)
    else:
        idx = np.minimum((confidence * n_bins).astype(int), n_bins - 1)
        groups = [np.flatnonzero(idx == b) for b in range(n_bins)]
        groups = [g for g in groups if len(g)]
    rows = []
    for g in groups:
        k, m = int(correct[g].sum()), len(g)
        lo, hi = wilson_interval(k, m)
        rows.append((confidence[g].min(), confidence[g].max(), m, confidence[g].mean(), k / m, lo, hi))
    return rows

def ascii_reliability(confidence, correct, n_bins=10, width=40):
    """Text reliability diagram: '|' marks mean confidence, '#' the accuracy bar,
    '-' the Wilson 95% interval around it."""
    lines = ["conf range     n   conf  acc   0" + " " * (width - 2) + "1"]
    for lo, hi, m, conf, acc, wlo, whi in reliability_table(confidence, correct, n_bins):
        row = [" "] * (width + 1)
        for j in range(int(round(wlo * width)), int(round(whi * width)) + 1):
            row[j] = "-"
        for j in range(int(round(acc * width)) + 1):
            row[j] = "#"
        row[int(round(conf * width))] = "|"
        lines.append(f"{lo:4.2f}-{hi:4.2f} {m:5d}  {conf:4.2f}  {acc:4.2f}  " + "".join(row))
    return "\n".join(lines)
```

Output on a simulated over-confident 4-way model (true T = 2.0, n = 1,000, 8 equal-mass bins), before and after a temperature fitted on 300 separate items (`exp_calibration.py` 5c):

```
conf range     n   conf  acc   0                                      1
0.27-0.52   125  0.46  0.40  #################-|--
0.52-0.62   125  0.56  0.48  ####################---|
0.62-0.72   125  0.66  0.52  ######################---  |
0.72-0.80   125  0.76  0.54  #######################---    |
0.80-0.88   125  0.84  0.66  ############################---   |
0.88-0.93   125  0.91  0.65  ###########################---      |
0.93-0.98   125  0.96  0.79  #################################--   |
0.98-1.00   125  0.99  0.84  ###################################--   |
after:
conf range     n   conf  acc   0                                      1
0.26-0.42   125  0.37  0.44  ###############|###---
0.42-0.47   125  0.45  0.44  ##################|---
0.47-0.53   125  0.50  0.49  ####################|---
0.53-0.58   125  0.55  0.58  ######################|#----
0.58-0.64   125  0.62  0.63  #########################|---
0.64-0.71   125  0.68  0.67  ###########################|---
0.71-0.79   125  0.75  0.76  ##############################|---
0.79-0.84   125  0.82  0.87  #################################|##--
```

How to read the shape:

- `|` to the **right** of the interval in every bin: over-confident - a temperature T > 1 fixes it.
- `|` to the right at high confidence and to the left at low confidence: the same, seen from both ends.
- `|` to the **left** at high confidence and right at low confidence: compressed toward the middle (under-confident, T < 1) - or, for a Noul, a slope < 1 that Platt fixes.
- A gap of the same sign everywhere for a Noul: a shifted intercept (the model says yes too often) - needs Platt, not temperature.
- Note the "after" diagram: temperature also **moves items between bins** (confidences dropped from 0.27-1.00 to 0.26-0.84). Comparing per-bin numbers before and after is meaningless; compare the summary metrics on the same test items.

---

## Recalibration maps and when each fits

All maps are fitted on a calibration split and evaluated on a different test split. All log-based maps take the same `floor` (see the next section).

| Map | Parameters | Fixes | Preserves argmax? | Minimum labels (rule of thumb, see tables) | Source |
|---|---|---|---|---|---|
| Temperature | 1 | uniform over/under-confidence | yes | ~20-50 per question | Guo et al., ICML 2017 |
| Platt (binary) | 2 | slope **and** intercept: compression toward 0.5, "says yes too often" | no (moves the 0.5 crossing) | ~50 | Platt 1999; scikit-learn `method="sigmoid"` |
| Isotonic (binary) | non-parametric, monotone | any monotone distortion | no | ~1,000 | scikit-learn docs: "not recommended when the number of calibration samples is too low (≪1000)" |
| Vector scaling | 2k | per-class slope + per-class offset: label prior, position prior | no | ~100-300 for k = 4 | Guo et al. 2017 |
| Matrix scaling / Dirichlet | k² + k | class confusions, interactions | no | ~1,000 for k = 4, with regularisation | Kull et al., NeurIPS 2019 (Dirichlet calibration = log-transform + linear layer + softmax) |
| Contextual calibration | k (estimated **without labels**) | label prior revealed by a content-free input | no | 0 labels (one extra call per question) | Zhao et al., ICML 2021 |

The isotonic quote is from the scikit-learn 1.9.1 `CalibratedClassifierCV` docstring (installed package, read 2026-10-03). Kull et al.'s abstract describes Dirichlet calibration as "equivalent to log-transforming the uncalibrated probabilities, followed by one linear layer and softmax" (arXiv 1910.12656 abstract) - which is what `fit_matrix_scaling` does on log-probabilities, with an L2 pull toward the identity instead of the paper's off-diagonal/intercept regulariser (ODIR).

```python
# Jev rounds probabilities to 0.01 and returns exact zeros. log(0) is -inf, and a
# tiny floor (1e-6) turns every zero into a huge negative logit that inflates the
# fitted temperature. Half the rounding quantum is the honest floor.
FLOOR = 0.005
```

```python
def fit_temperature(probs, labels, floor=FLOOR):
    logits = np.log(np.clip(probs, floor, 1.0))
    labels = np.asarray(labels)

    def nll_t(t):
        p = softmax(logits / t, axis=1)
        return -np.mean(np.log(p[np.arange(len(labels)), labels]))

    return minimize_scalar(nll_t, bounds=(0.05, 20.0), method="bounded").x

def apply_temperature(probs, t, floor=FLOOR):
    return softmax(np.log(np.clip(probs, floor, 1.0)) / t, axis=1)

def fit_platt(p_yes, y, floor=FLOOR):
    from sklearn.linear_model import LogisticRegression

    x = logit(np.clip(np.asarray(p_yes, float), floor, 1 - floor)).reshape(-1, 1)
    return LogisticRegression(C=1e6).fit(x, y)  # effectively unregularised: two parameters

def apply_platt(model, p_yes, floor=FLOOR):
    x = logit(np.clip(np.asarray(p_yes, float), floor, 1 - floor)).reshape(-1, 1)
    return model.predict_proba(x)[:, 1]

def fit_isotonic(p_yes, y):
    """Monotone, non-parametric map. Needs ~1,000 calibration items (scikit-learn docs)."""
    from sklearn.isotonic import IsotonicRegression

    return IsotonicRegression(y_min=0.0, y_max=1.0, out_of_bounds="clip").fit(
        np.asarray(p_yes, float), np.asarray(y, float)
    )
```

```python
def _fit_affine(logits, labels, n_params, unpack, l2):
    labels = np.asarray(labels)
    n, k = logits.shape

    def loss(theta):
        W, b = unpack(theta, k)
        z = logits @ W.T + b
        lp = log_softmax(z, axis=1)
        reg = l2 * (np.sum((W - np.eye(k)) ** 2) + np.sum(b ** 2))
        return -np.mean(lp[np.arange(n), labels]) + reg

    theta0 = np.zeros(n_params)
    # start from the identity map
    if n_params == 2 * k:
        theta0[:k] = 1.0
    else:
        theta0[: k * k] = np.eye(k).ravel()
    res = minimize(loss, theta0, method="L-BFGS-B")
    return unpack(res.x, k)

def _unpack_vector(theta, k):
    return np.diag(theta[:k]), theta[k:]

def _unpack_matrix(theta, k):
    return theta[: k * k].reshape(k, k), theta[k * k :]

def fit_vector_scaling(probs, labels, floor=FLOOR, l2=1e-3):
    """Vector scaling (Guo et al. 2017): per-class slope + intercept on log-probs.
    2k parameters; the intercepts absorb a per-class prior/position bias."""
    logits = np.log(np.clip(np.asarray(probs, float), floor, 1.0))
    return _fit_affine(logits, labels, 2 * probs.shape[1], _unpack_vector, l2)

def fit_matrix_scaling(probs, labels, floor=FLOOR, l2=1e-2):
    """Matrix scaling on log-probs = Dirichlet calibration (Kull et al. 2019) with an
    L2 pull toward the identity. k^2 + k parameters: needs many labels per class."""
    logits = np.log(np.clip(np.asarray(probs, float), floor, 1.0))
    return _fit_affine(logits, labels, probs.shape[1] ** 2 + probs.shape[1], _unpack_matrix, l2)

def apply_affine(probs, W, b, floor=FLOOR):
    logits = np.log(np.clip(np.asarray(probs, float), floor, 1.0))
    return softmax(logits @ W.T + b, axis=1)
```

### Which one, by calibration-set size (simulation)

**4-way Choice-like model**, true T = 2.0 plus a per-position log-bias (+0.6, 0, 0, -0.3), outputs rounded to 0.01; held-out NLL / ECE / accuracy on 5,000 items, mean of 40 calibration draws per row (`exp_calibration.py` section 4). Uncalibrated: NLL 1.072, ECE 0.168, accuracy 0.606.

| n_cal | temperature NLL / ECE / acc | vector NLL / ECE / acc | matrix NLL / ECE / acc |
|---|---|---|---|
| 20 | 0.964 / 0.063 / 0.606 | 1.242 / 0.147 / 0.577 | 1.499 / 0.231 / 0.529 |
| 30 | 0.963 / 0.062 / 0.606 | 1.108 / 0.093 / 0.587 | 1.265 / 0.151 / 0.557 |
| 50 | 0.954 / 0.045 / 0.606 | 1.002 / 0.058 / 0.599 | 1.075 / 0.086 / 0.583 |
| 100 | 0.948 / 0.037 / 0.606 | 0.974 / 0.045 / 0.606 | 1.015 / 0.063 / 0.598 |
| 300 | 0.947 / 0.034 / 0.606 | 0.948 / 0.036 / 0.609 | 0.960 / 0.042 / 0.607 |
| 1000 | 0.945 / 0.028 / 0.606 | 0.940 / 0.027 / 0.611 | 0.943 / 0.030 / 0.610 |

- Temperature never changes accuracy (0.606 everywhere): it rescales logits, so the argmax is fixed.
- Vector and matrix scaling **lose accuracy** below ~100 labels (matrix at n = 20 drops it from 0.606 to 0.529) and only beat temperature at n ≈ 1,000, where the per-position offsets start to pay. For position bias specifically, removing it at the source (`order-bias.md`) is cheaper in labels.

**Binary Noul-like model**, compressed toward 0.5 (true slope 0.5, no intercept shift); held-out Brier / top-label ECE on 5,000 items, mean of 40 draws (`exp_calibration.py` 4b). Uncalibrated: Brier 0.1446, ECE 0.099.

| n_cal | Platt Brier / ECE | isotonic Brier / ECE | temperature Brier / ECE |
|---|---|---|---|
| 20 | 0.1572 / 0.082 | 0.1668 / 0.121 | 0.1401 / 0.054 |
| 30 | 0.1461 / 0.060 | 0.1562 / 0.097 | 0.1382 / 0.051 |
| 50 | 0.1419 / 0.041 | 0.1509 / 0.080 | 0.1353 / 0.036 |
| 100 | 0.1364 / 0.027 | 0.1435 / 0.064 | 0.1342 / 0.027 |
| 300 | 0.1343 / 0.019 | 0.1373 / 0.037 | 0.1336 / 0.019 |
| 1000 | 0.1334 / 0.013 | 0.1349 / 0.023 | 0.1332 / 0.014 |
| 3000 | 0.1332 / 0.012 | 0.1340 / 0.017 | 0.1332 / 0.012 |

- At 20 labels **Platt and isotonic make the Brier score worse than doing nothing** (0.1572 and 0.1668 vs 0.1446); at 30, Platt is still slightly worse (0.1461). This is the simulation counterpart of the skill's warning that fewer than ~30 labels can make calibration worse.
- Isotonic trails Platt at every size up to 3,000 on a distortion that is exactly sigmoid-shaped; it only wins when the distortion is not.
- Temperature (a Platt with the intercept pinned at 0) is the best small-sample choice **when the only problem is the slope**.

**When the intercept is wrong**, temperature cannot fix it. A model that says yes too often (true logit shifted by +1.0, slope 1), Platt fitted on 300 items (`exp_calibration.py` 4c):

| | value |
|---|---|
| fitted Platt slope / intercept | 0.91 / -0.73 (truth 1.00 / -1.00) |
| held-out Brier, raw | 0.1563 |
| held-out Brier, temperature only (T = 1.34) | 0.1529 |
| held-out Brier, Platt | 0.1341 |

Decision rule: **Choice/Score → temperature; Noul → Platt** (≥ 50 labels), falling back to temperature on the two-column matrix `[1 - p, p]` when labels are scarce and the reliability diagram shows no constant offset. Vector/matrix scaling and isotonic only with ≥ 1,000 labels per question.

---

## The rounding floor: 0.005 vs 1e-6

Every log-based fit computes log p. A hosted API that rounds to 0.01 returns exact zeros; log 0 = -inf, so code clips at a floor. The floor is a modelling choice with a large effect, because each item whose **true** class was reported as 0.00 contributes -log(floor) to the loss, and the optimiser can only shrink that by flattening everything (raising T).

**Reproduction** (`exp_calibration.py` section 3): a 4-way model with true over-confidence T = 2.0, n = 4,000, outputs rounded to 0.01 → 24.1% of all reported probabilities are exact zeros.

| floor | fitted T |
|---|---|
| 0.005 | 1.69 |
| 0.001 | 1.88 |
| 1e-4 | 2.14 |
| 1e-6 | 2.65 |
| 1e-12 | 4.69 |
| (unrounded outputs, floor 1e-12) | 1.91 |

Across true temperatures and seeds (20 seeds, n_cal = 500, held-out ECE on 3,000):

| true T | floor | mean fitted T | held-out ECE after |
|---|---|---|---|
| 1.0 | 0.005 | 0.98 | 0.025 |
| 1.0 | 0.002 | 0.98 | 0.025 |
| 1.0 | 0.001 | 0.99 | 0.025 |
| 1.0 | 1e-6 | 0.99 | 0.025 |
| 1.5 | 0.005 | 1.42 | 0.026 |
| 1.5 | 0.002 | 1.46 | 0.025 |
| 1.5 | 0.001 | 1.49 | 0.025 |
| 1.5 | 1e-6 | 1.64 | 0.029 |
| 2.0 | 0.005 | 1.75 | 0.034 |
| 2.0 | 0.002 | 1.87 | 0.028 |
| 2.0 | 0.001 | 1.95 | 0.024 |
| 2.0 | 1e-6 | 2.82 | 0.055 |
| 3.0 | 0.005 | 2.15 | 0.052 |
| 3.0 | 0.002 | 2.39 | 0.040 |
| 3.0 | 0.001 | 2.57 | 0.033 |
| 3.0 | 1e-6 | 4.65 | 0.045 |

What this shows, honestly:

- **A tiny floor (1e-6, 1e-12) inflates T**, sometimes by more than 2x, and the over-correction costs held-out ECE (0.055 vs 0.024 at true T = 2).
- **0.005 (half the rounding quantum) under-corrects** when the model is strongly over-confident (2.15 for a true 3.0): a zero really means "below 0.005", so pricing it at exactly 0.005 makes the model look a bit less wrong than it was. It never over-corrects in these runs, and it is the floor the skill uses.
- **0.001-0.002 did best** on held-out ECE when T ≥ 2 in this simulation. The best value depends on how far the true probabilities of zeroed classes fall below 0.005, which you cannot see.
- The skill's own simulation (different seeds and data) reported 1.84 with 0.005 and 3.02 with 1e-6 for a true 2.0 - the same direction and a similar size. The skill also cites an independent Jev audit whose published temperatures (3.29, 3.40) were corrected by the author on 2026-09-22 to 1.30 and 1.92 for exactly this artefact; that audit was not re-read for this page.

Practical rule: **fit with 0.005, refit with 0.001, and report both T values**. If they differ by more than ~10%, choose by cross-validated held-out Brier, which needs no floor and therefore cannot favour one:

```python
def select_floor_cv(probs, labels, floors=(0.005, 0.002, 0.001), n_splits=5, seed=0):
    """Pick the log floor for a temperature fit by cross-validated held-out Brier
    (Brier needs no floor, so the yardstick does not depend on the choice).
    Returns (best_floor, {floor: (mean_T, mean_heldout_brier)})."""
    probs = np.asarray(probs, float)
    labels = np.asarray(labels)
    rng = np.random.default_rng(seed)
    folds = np.array_split(rng.permutation(len(labels)), n_splits)
    table = {}
    for fl in floors:
        ts, bs = [], []
        for i in range(n_splits):
            test = folds[i]
            train = np.concatenate([folds[j] for j in range(n_splits) if j != i])
            t = fit_temperature(probs[train], labels[train], floor=fl)
            ts.append(t)
            bs.append(brier(apply_temperature(probs[test], t, floor=fl), labels[test]))
        table[fl] = (float(np.mean(ts)), float(np.mean(bs)))
    best = min(table, key=lambda f: table[f][1])
    return best, table
```

In 10 simulated corpora of 500 items with true T = 2.0, `select_floor_cv` (floors 0.005 / 0.002 / 0.001 / 1e-6) picked 0.001 six times and 0.002 four times; with true T = 1.0 it picked 0.005 in six of ten (`probe_floor.py`). The Brier differences between floors are small (e.g. 0.5202 / 0.5182 / 0.5173 / 0.5211 for one corpus, where 1e-6 was worst), so the choice matters most for the reported T, less for the calibrated probabilities. The selection is itself noisy: in the same probe, with 1e-6 among the candidates, it picked 1e-6 in 3 of 10 corpora at true T = 1.0 and at true T = 3.0 (n = 500), and in 2-4 of 10 at n = 100. Keep 1e-6 out of the candidate list (the default `floors` already does) and treat a pick between neighbouring floors as a tie.

A principled alternative is an interval-censored likelihood (a reported 0.00 means "true value in [0, 0.005)"), which removes the floor altogether. It is not implemented or tested here.

---

## scikit-learn 1.9: method="temperature" and FrozenEstimator

Facts checked against the installed scikit-learn 1.9.1 (2026-10-03):

- `CalibratedClassifierCV(estimator=None, *, method='sigmoid', cv=None, n_jobs=None, ensemble='auto')` - `method` is one of `'sigmoid'`, `'isotonic'`, `'temperature'`; the docstring marks `'temperature'` as `.. versionchanged:: 1.8  Added option 'temperature'`.
- The docstring: temperature scaling "naturally supports multi-class calibration by applying `softmax(classifier_logits/T)` with a value of `T` (temperature) that optimizes the log loss", while sigmoid and isotonic "extend to multi-class classification using a One-vs-Rest (OvR) strategy with post-hoc renormalization".
- `ensemble="auto"` "will use `False` if the `estimator` is a `FrozenEstimator`, and `True` otherwise".
- `cv="prefit"` is **gone**, not just deprecated. In 1.9.1 it raises `sklearn.utils._param_validation.InvalidParameterError: The 'cv' parameter of CalibratedClassifierCV must be an int in the range [2, inf), an object implementing 'split' and 'get_n_splits', an iterable or None. Got 'prefit' instead.` Wrap an already-fitted model in `sklearn.frozen.FrozenEstimator` instead.
- The fitted temperature is stored as an inverse temperature: `calibrated_classifiers_[0].calibrators[0].beta_` (private class `_TemperatureScaling`, so treat the attribute path as an implementation detail).

Executed example (synthetic 3-class data; a random forest, which is typically under-confident):

```python
from sklearn.calibration import CalibratedClassifierCV
from sklearn.datasets import make_classification
from sklearn.ensemble import RandomForestClassifier
from sklearn.frozen import FrozenEstimator
from sklearn.model_selection import train_test_split

X, y = make_classification(n_samples=6000, n_classes=3, n_informative=6, random_state=0)
X_train, X_rest, y_train, y_rest = train_test_split(X, y, test_size=0.5, random_state=0)
X_cal, X_test, y_cal, y_test = train_test_split(X_rest, y_rest, test_size=0.5, random_state=0)

clf = RandomForestClassifier(random_state=0).fit(X_train, y_train)
calibrated = CalibratedClassifierCV(FrozenEstimator(clf), method="temperature").fit(X_cal, y_cal)
temperature = 1 / calibrated.calibrated_classifiers_[0].calibrators[0].beta_
```

Output: `T = 0.53`, held-out log loss 0.445 → 0.341. T < 1 means the forest's probabilities were too flat and got sharpened.

### Pitfall: feeding rounded API probabilities to scikit-learn

`_TemperatureScaling` converts its input with `_convert_to_logits(decision_values, eps=1e-12)`: if every entry is in [0, 1] **and every row sums to 1** (checked with `isclose`), it takes `log(p + 1e-12)`; otherwise it treats the input as logits as-is (scikit-learn 1.9.1 source, `sklearn/calibration.py`). Two consequences for API probabilities rounded to 0.01 (`exp_calibration.py` section 3 and `probe_sk.py`):

| what you pass | how scikit-learn reads it | fitted T (true 2.0) |
|---|---|---|
| raw rounded rows (27.9% of rows sum to 0.99 or 1.01) | as **logits** - silently | 0.39 |
| rows renormalised, zeros kept | as probabilities, floor 1e-12 | 4.69 |
| floored at 0.005, then renormalised (`apply_temperature(probs, 1.0)`) | as probabilities | 1.69 (same as `fit_temperature`) |

So for API probabilities, either use `fit_temperature` directly, or floor-and-renormalise before wrapping them in an estimator.

---

## Sample-size guidance

Rules of thumb, each tied to the evidence on this page or the skill:

| Goal | Labels needed | Evidence |
|---|---|---|
| Accuracy to ±10 points | ~60-80 | Wilson 95% half-width at accuracy 0.80: 0.109 at n = 50, 0.088 at n = 77, 0.078 at n = 100 |
| Accuracy to ±5 points | ~250 | half-width 0.055 at n = 200, 0.035 at n = 500 |
| Detect an ECE gap of ~0.05 | 500-1,000 | noise floor p95 0.065 at n = 500 and 0.048 at n = 1,000 for spread confidences |
| Fit one temperature | 20-50 | temperature helps even at n = 20 in the tables above |
| Fit Platt (2 parameters) | ≥ 50 | worse than raw at 20, break-even near 30 |
| Vector scaling (k = 4) | ≥ 300 | loses accuracy below ~100 |
| Matrix scaling / isotonic | ≥ 1,000 | scikit-learn docstring (≪1000); tables above |
| Per-class thresholds | enough items **of that class** | rare classes dominate the requirement |
| Split conformal at miscoverage α | ≥ ⌈1/α⌉ - 1 (9 at α = 0.1) for a finite quantile; ~500 for a stable one | `conformal.md` |

Wilson intervals (`exp_extra.py`, accuracy 0.80):

| n | 95% interval | half-width |
|---|---|---|
| 30 | [0.627, 0.905] | 0.139 |
| 50 | [0.670, 0.888] | 0.109 |
| 77 | [0.703, 0.878] | 0.088 |
| 100 | [0.711, 0.867] | 0.078 |
| 200 | [0.739, 0.850] | 0.055 |
| 500 | [0.763, 0.833] | 0.035 |
| 1000 | [0.774, 0.824] | 0.025 |
| 2000 | [0.782, 0.817] | 0.018 |

```python
def wilson_interval(k, n, z=1.959964):
    """Wilson score interval for a binomial proportion k/n."""
    if n == 0:
        return (0.0, 1.0)
    phat = k / n
    denom = 1 + z * z / n
    centre = (phat + z * z / (2 * n)) / denom
    half = z * math.sqrt(phat * (1 - phat) / n + z * z / (4 * n * n)) / denom
    return (max(0.0, centre - half), min(1.0, centre + half))
```

All of these are **per question**: labels for `department` do nothing for `is_urgent`.

---

## Per-question fitting

One correction per question, never shared across questions, primitives or fields: the distortion depends on the instructions, the options, their order and names (see `order-bias.md`), and the state shape. A shared T averages incompatible errors. Store the model version next to each fitted map and refit when it changes.

```python
MIN_LABELS = 50  # below ~30 a fit can make things worse; 50 is the working minimum


def fit_per_question(corpus):
    """corpus: {question_id: {"kind": "choice"|"noul", "x": probs or p_yes, "y": labels}}.
    One correction per question, never shared; None means 'not enough labels, ship raw'."""
    fitted = {}
    for qid, d in corpus.items():
        if len(d["y"]) < MIN_LABELS:
            fitted[qid] = None
        elif d["kind"] == "choice":
            fitted[qid] = ("temperature", fit_temperature(d["x"], d["y"]))
        else:
            fitted[qid] = ("platt", fit_platt(d["x"], d["y"]))
    return fitted


p_yes, y_yes = simulate_noul(300, slope=0.5, rng=np.random.default_rng(1))
corpus = {
    "department": {"kind": "choice", "x": probs[:200], "y": labels[:200]},
    "is_urgent": {"kind": "noul", "x": p_yes, "y": y_yes},
    "is_legal_threat": {"kind": "noul", "x": p_yes[:25], "y": y_yes[:25]},
}
fits = fit_per_question(corpus)
```

Output (simulated): `department` → temperature 1.64; `is_urgent` → Platt with slope 2.24 (the simulated Noul was compressed with slope 0.5, so the correction stretches by ~2); `is_legal_threat` (25 labels) → `None`, ship raw and keep collecting labels.

For thresholds after the fit, the skill's `risk_coverage` / `lowest_bar_for` are in the module unchanged:

```python
def risk_coverage(confidence, correct):
    """For each candidate bar: the share of items kept and the error rate among them."""
    confidence = np.asarray(confidence, float)
    correct = np.asarray(correct, bool)
    rows = []
    for bar in np.unique(confidence):
        kept = confidence >= bar
        rows.append((bar, kept.mean(), 1 - correct[kept].mean()))
    return rows

def lowest_bar_for(target_error, confidence, correct, min_kept=30):
    """Lowest bar whose kept items err at most target_error, with at least min_kept items."""
    for bar, coverage, error in risk_coverage(confidence, correct):
        if error <= target_error and coverage * len(confidence) >= min_kept:
            return bar, coverage, error
    return None
```

---

## Bootstrap confidence intervals

Every metric on a few hundred items needs an interval. Resample **items** (rows) with replacement and recompute; for a before/after comparison on the same items, resample the **pairs** jointly so the shared item difficulty cancels.

```python
def bootstrap_ci(statistic, *arrays, n_boot=2000, alpha=0.05, seed=0):
    """Percentile bootstrap CI of statistic(*arrays) resampling rows (items) jointly."""
    rng = np.random.default_rng(seed)
    arrays = [np.asarray(a) for a in arrays]
    n = len(arrays[0])
    vals = []
    for _ in range(n_boot):
        idx = rng.integers(0, n, n)
        vals.append(statistic(*[a[idx] for a in arrays]))
    lo, hi = np.percentile(vals, [100 * alpha / 2, 100 * (1 - alpha / 2)])
    return float(statistic(*arrays)), float(lo), float(hi)

def paired_bootstrap_diff(statistic, a_arrays, b_arrays, n_boot=2000, alpha=0.05, seed=0):
    """CI of statistic(b) - statistic(a) on the SAME items (e.g. ECE before/after a fit)."""
    rng = np.random.default_rng(seed)
    a_arrays = [np.asarray(x) for x in a_arrays]
    b_arrays = [np.asarray(x) for x in b_arrays]
    n = len(a_arrays[0])
    diffs = []
    for _ in range(n_boot):
        idx = rng.integers(0, n, n)
        diffs.append(statistic(*[x[idx] for x in b_arrays]) - statistic(*[x[idx] for x in a_arrays]))
    lo, hi = np.percentile(diffs, [100 * alpha / 2, 100 * (1 - alpha / 2)])
    point = statistic(*b_arrays) - statistic(*a_arrays)
    return float(point), float(lo), float(hi)
```

What it shows (`exp_calibration.py` section 5): true T = 2.0, temperature fitted on 100 items (T = 1.89), evaluated on only 150 held-out items.

| metric | value [95% CI] |
|---|---|
| ECE before | 0.173 [0.127, 0.256] |
| ECE after | 0.133 [0.090, 0.208] |
| noise floor at n = 150 (after), median / p95 | 0.094 / 0.131 |
| paired difference after - before | -0.040 [-0.111, 0.031] |
| accuracy (Wilson) | 0.620 [0.540, 0.694] |
| Brier before / after | 0.5534 / 0.5088 |
| classwise ECE before / after | 0.090 / 0.073 |

The fit is genuinely good (the true T is 2.0), yet on 150 test items the paired CI of the ECE change **includes zero** and the "after" ECE sits at the noise floor's 95th percentile. The proper score moved more clearly (Brier 0.553 → 0.509). With a test set this small, report "no evidence of miscalibration after the fit", not "ECE improved by 23%".

One report per question, on held-out data:

```python
def calibration_report(probs, labels, n_bins=10):
    """Everything to put next to ONE question's numbers, on held-out data."""
    conf, pred = top_label(probs)
    correct = pred == labels
    k, n = int(correct.sum()), len(correct)
    floor_med, floor_p95 = ece_noise_floor(conf, n_bins)
    return {
        "n": n,
        "accuracy": (k / n, *wilson_interval(k, n)),
        "ece": bootstrap_ci(lambda c, ok: ece(c, ok, n_bins), conf, correct),
        "ece_noise_floor": (float(floor_med), float(floor_p95)),
        "debiased_l2_ce": debiased_l2_ce(conf, correct, n_bins),
        "brier": brier(probs, labels),
        "over_confident_bins": float(np.mean([
            ok.mean() < c.mean()
            for c, ok in zip(np.array_split(np.sort(conf), n_bins),
                             np.array_split(correct[np.argsort(conf)], n_bins))
        ])),
    }


report = calibration_report(probs[200:], labels[200:])
```

Output for 200 held-out items of an uncalibrated simulated model: accuracy 0.605 [0.536, 0.670]; ECE 0.157 [0.113, 0.229] against a noise floor of 0.061 / 0.091; debiased L2 0.165; Brier 0.519; every bin over-confident (`over_confident_bins` = 1.0). That one is clearly miscalibrated: the ECE's lower CI bound is above the floor's 95th percentile.

---

## Contextual calibration (Zhao et al. 2021)

*Calibrate Before Use* (Zhao, Wallace, Feng, Klein & Singh, ICML 2021, arXiv 2102.09690) targets a different failure: a label prior baked into the prompt ("bias of language models towards predicting certain answers, e.g., those that are placed near the end of the prompt or are common in the pre-training data", abstract). It needs **no labels**: ask the same question about a content-free input, read the distribution p_cf, and divide it out.

The official code (`tonyzhaozh/few-shot-learning`, `run_classification.py`, fetched 2026-10-03) uses three content-free inputs, averages their label probabilities, and builds a diagonal correction:

```python
# verbatim, https://github.com/tonyzhaozh/few-shot-learning/blob/main/run_classification.py
        content_free_inputs = ["N/A", "", "[MASK]"]
        p_cf = get_p_content_free(params, train_sentences, train_labels, content_free_inputs=content_free_inputs)
```

```python
# verbatim, same file, eval_accuracy()
        if mode == "diagonal_W":
            W = np.linalg.inv(np.identity(num_classes) * p_cf)
            b = np.zeros([num_classes, 1])
```

The official code then takes `argmax(W @ p)` for accuracy only. `contextual_calibration` additionally renormalises the rows so the output is a distribution (our addition, flagged in the docstring):

```python
def contextual_calibration(probs, p_cf):
    """Zhao et al. (ICML 2021) 'Calibrate Before Use', diagonal W = diag(p_cf)^-1, b = 0.

    p_cf: the model's (normalized) class probabilities for content-free inputs such
    as "N/A", "" and "[MASK]", averaged. The official code takes argmax of W @ p;
    renormalizing the rows (done here) is our addition so the output is a distribution.
    """
    p_cf = np.asarray(p_cf, float)
    p_cf = p_cf / p_cf.sum()
    q = np.asarray(probs, float) / p_cf
    return q / q.sum(axis=1, keepdims=True)
```

Simulation (`exp_calibration.py` section 6): a 3-way model whose logits carry a +1.0 bias on class 0; 3,000 items.

| | accuracy | share predicted class 0 (true share 0.318) | NLL |
|---|---|---|---|
| raw | 0.642 | 0.528 | 0.804 |
| contextual (exact p_cf) | 0.679 | 0.333 | 0.718 |
| contextual (p_cf 20% off) | 0.675 | - | - |

With Jev, the content-free item is a request whose `state` carries no content. p_cf depends only on the question (instructions, options, their order), not on the item, so it costs **one extra request per question wording**, cached:

```python
from typesafe_sdk import Choice

QUESTION = Choice(instructions="Which team should handle this ticket?",
                  criteria={"billing": None, "technical": None, "sales": None})


def content_free_prior(client, question, inputs=("N/A",)):
    """p_cf for ONE question: its answer on inputs that carry no content. It does not
    depend on the item, so compute it once per question wording and cache it."""
    rows = []
    for state in inputs:
        answer = client.system_one(state, {"q": question}).choices["q"]
        rows.append([answer.probabilities[o] for o in question.criteria])
    p_cf = np.clip(np.mean(rows, axis=0), 0.005, None)  # rounded API: no exact zeros
    return dict(zip(question.criteria, p_cf / p_cf.sum()))


p_cf = content_free_prior(client, QUESTION)
```

Against the offline mock (which gives `billing` a +0.4 label prior and the first position a +0.8 lean), p_cf came out as billing 0.65 / technical 0.19 / sales 0.16: a content-free probe measures the **position lean of that particular order too**, so recompute it whenever the option order changes. The original paper also used `""` and `"[MASK]"`; whether the hosted Jev API accepts an empty `state` was not verified, so the snippet defaults to `"N/A"` only.

Contextual calibration and temperature compose: divide out p_cf first, then fit a temperature on labelled data (the correction changes the logits, so refit T after adopting it).

---

## Checklist

- [ ] Probabilities from the right field (`probabilities` / `noul`), never `confidence`; option order fixed in `to_matrix`
- [ ] Held-out split; metric reported per question with: accuracy + Wilson CI, equal-mass ECE + bootstrap CI + noise floor from your own confidences, debiased L2, Brier
- [ ] Reliability diagram with per-bin intervals; shape read (over-confident, compressed, offset)
- [ ] Map chosen by failure and label count: temperature (Choice/Score), Platt (Noul, ≥ 50), nothing fancier below ~1,000
- [ ] Fit at floor 0.005 and 0.001; if T differs > 10%, choose with `select_floor_cv`
- [ ] scikit-learn: `FrozenEstimator`, not `cv="prefit"`; never pass rounded rows that do not sum to 1
- [ ] Before/after claims backed by a paired bootstrap CI that excludes 0
- [ ] Model version, option wording and option order stored next to every fitted map

---

## Sources

- Guo, Pleiss, Sun & Weinberger, *On Calibration of Modern Neural Networks*, ICML 2017 - arXiv 1706.04599
- Nixon, Dusenberry, Jerfel, Nguyen, Liu, Zhang & Tran, *Measuring Calibration in Deep Learning*, 2019 - arXiv 1904.01685
- Kumar, Liang & Ma, *Verified Uncertainty Calibration*, NeurIPS 2019 - arXiv 1909.10155; code https://github.com/p-lambda/verified_calibration (`calibration/utils.py`, commit ee81c34, read 2026-10-03)
- Roelofs, Cain, Shlens & Mozer, *Mitigating Bias in Calibration Error Estimation*, AISTATS 2022 - arXiv 2012.08668
- Kull, Perello-Nieto, Kängsepp, Silva Filho, Song & Flach, *Beyond temperature scaling: Dirichlet calibration*, NeurIPS 2019 - arXiv 1910.12656
- Zhao, Wallace, Feng, Klein & Singh, *Calibrate Before Use*, ICML 2021 - arXiv 2102.09690; code https://github.com/tonyzhaozh/few-shot-learning
- Stephenson, Coelho & Jolliffe, *Two Extra Components in the Brier Score Decomposition*, Weather and Forecasting 23(4):752-757 (2008) - cited for the WBV/WBC terms; the identity itself is verified numerically by the test suite, the paper was not re-read for this page
- scikit-learn 1.9.1 `sklearn/calibration.py` (installed package): `CalibratedClassifierCV`, `_TemperatureScaling`, `_convert_to_logits`
- TypeSafe docs: https://docs.typesafe.ai/confidence, https://docs.typesafe.ai/api (fetched 2026-10-03)
