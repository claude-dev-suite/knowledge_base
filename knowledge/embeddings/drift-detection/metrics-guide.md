# Embedding Drift Metrics -- Implementation Guide

## Overview / TL;DR

This guide covers the statistical metrics used to detect embedding drift: KL divergence, Population Stability Index (PSI), Maximum Mean Discrepancy (MMD), centroid distance tracking, and cosine similarity distribution shift. Each metric is implemented with scipy and numpy, visualized with matplotlib, and calibrated with recommended thresholds. The goal is to detect meaningful drift while avoiding false alarms from natural distribution variation.

---

## Metric 1: Centroid Distance

The simplest and most interpretable metric. Track the centroid (mean vector) of the embedding distribution and measure how far it moves.

```python
import numpy as np


def centroid_distance(
    baseline_embeddings: np.ndarray,
    current_embeddings: np.ndarray,
) -> dict:
    """Compute the distance between centroids of two embedding sets.

    Returns:
        Dictionary with cosine distance, euclidean distance, and normalized measures.
    """
    baseline_centroid = baseline_embeddings.mean(axis=0)
    current_centroid = current_embeddings.mean(axis=0)

    # Normalize centroids for cosine comparison
    bc_norm = baseline_centroid / np.linalg.norm(baseline_centroid)
    cc_norm = current_centroid / np.linalg.norm(current_centroid)

    cosine_distance = 1.0 - np.dot(bc_norm, cc_norm)
    euclidean_distance = np.linalg.norm(baseline_centroid - current_centroid)

    return {
        "cosine_distance": float(cosine_distance),
        "euclidean_distance": float(euclidean_distance),
    }


# Thresholds
# cosine_distance < 0.01: no drift
# cosine_distance 0.01-0.05: minor drift (monitor)
# cosine_distance 0.05-0.10: significant drift (investigate)
# cosine_distance > 0.10: severe drift (action required)
```

---

## Metric 2: KL Divergence (Per-Dimension)

KL divergence measures how one probability distribution differs from another. For high-dimensional embeddings, compute KL divergence per dimension and aggregate.

```python
import numpy as np
from scipy import stats


def kl_divergence_per_dimension(
    baseline_embeddings: np.ndarray,
    current_embeddings: np.ndarray,
    n_bins: int = 50,
    epsilon: float = 1e-10,
) -> dict:
    """Compute KL divergence between embedding distributions, per dimension.

    Discretizes each dimension into histogram bins and computes KL divergence
    for each dimension, then aggregates.
    """
    n_dims = baseline_embeddings.shape[1]
    kl_values = []

    for dim in range(n_dims):
        # Get values for this dimension
        base_vals = baseline_embeddings[:, dim]
        curr_vals = current_embeddings[:, dim]

        # Compute histograms with shared bins
        min_val = min(base_vals.min(), curr_vals.min())
        max_val = max(base_vals.max(), curr_vals.max())
        bins = np.linspace(min_val, max_val, n_bins + 1)

        base_hist, _ = np.histogram(base_vals, bins=bins, density=True)
        curr_hist, _ = np.histogram(curr_vals, bins=bins, density=True)

        # Add epsilon to avoid log(0)
        base_hist = base_hist + epsilon
        curr_hist = curr_hist + epsilon

        # Normalize to valid probability distributions
        base_hist = base_hist / base_hist.sum()
        curr_hist = curr_hist / curr_hist.sum()

        # KL divergence: sum(P * log(P/Q))
        kl = stats.entropy(curr_hist, base_hist)  # scipy convention: entropy(p, q) = KL(p||q)
        kl_values.append(kl)

    kl_values = np.array(kl_values)

    return {
        "mean_kl": float(kl_values.mean()),
        "max_kl": float(kl_values.max()),
        "median_kl": float(np.median(kl_values)),
        "std_kl": float(kl_values.std()),
        "top_drifted_dims": np.argsort(kl_values)[::-1][:10].tolist(),
        "top_kl_values": kl_values[np.argsort(kl_values)[::-1][:10]].tolist(),
    }


# Thresholds
# mean_kl < 0.01: no drift
# mean_kl 0.01-0.05: minor drift
# mean_kl 0.05-0.20: significant drift
# mean_kl > 0.20: severe drift
```

---

## Metric 3: Population Stability Index (PSI)

PSI is widely used in credit scoring to detect distribution shifts. It is symmetric (unlike KL divergence) and has well-established interpretation guidelines.

```python
import numpy as np


def population_stability_index(
    baseline_embeddings: np.ndarray,
    current_embeddings: np.ndarray,
    n_bins: int = 10,
    method: str = "pca_projected",
    n_components: int = 10,
) -> dict:
    """Compute Population Stability Index for embedding distributions.

    PSI = sum((actual% - expected%) * ln(actual% / expected%))

    Standard interpretation:
        PSI < 0.10: no significant shift
        PSI 0.10-0.25: moderate shift (monitor)
        PSI > 0.25: significant shift (action needed)

    For high-dimensional embeddings, we project to lower dimensions first
    using PCA, then compute PSI on each component.
    """
    from sklearn.decomposition import PCA

    if method == "pca_projected":
        # Fit PCA on baseline
        pca = PCA(n_components=n_components)
        base_projected = pca.fit_transform(baseline_embeddings)
        curr_projected = pca.transform(current_embeddings)
    else:
        base_projected = baseline_embeddings
        curr_projected = current_embeddings

    n_dims = base_projected.shape[1]
    psi_values = []
    epsilon = 1e-8

    for dim in range(n_dims):
        # Compute decile boundaries from baseline
        boundaries = np.percentile(base_projected[:, dim], np.linspace(0, 100, n_bins + 1))
        boundaries[0] = -np.inf
        boundaries[-1] = np.inf

        # Bin both distributions
        base_counts = np.histogram(base_projected[:, dim], bins=boundaries)[0]
        curr_counts = np.histogram(curr_projected[:, dim], bins=boundaries)[0]

        # Convert to percentages
        base_pct = base_counts / base_counts.sum() + epsilon
        curr_pct = curr_counts / curr_counts.sum() + epsilon

        # PSI for this dimension
        psi = np.sum((curr_pct - base_pct) * np.log(curr_pct / base_pct))
        psi_values.append(psi)

    psi_values = np.array(psi_values)

    return {
        "mean_psi": float(psi_values.mean()),
        "max_psi": float(psi_values.max()),
        "total_psi": float(psi_values.sum()),
        "per_component": psi_values.tolist(),
        "interpretation": _interpret_psi(psi_values.mean()),
    }


def _interpret_psi(psi: float) -> str:
    if psi < 0.10:
        return "NO_SHIFT"
    elif psi < 0.25:
        return "MODERATE_SHIFT"
    else:
        return "SIGNIFICANT_SHIFT"
```

---

## Metric 4: Maximum Mean Discrepancy (MMD)

MMD is a kernel-based two-sample test that measures the distance between distributions in a reproducing kernel Hilbert space. It does not require discretization and works well for high-dimensional data.

```python
import numpy as np


def maximum_mean_discrepancy(
    baseline_embeddings: np.ndarray,
    current_embeddings: np.ndarray,
    kernel: str = "rbf",
    gamma: float | None = None,
    n_sample: int = 1000,
) -> dict:
    """Compute Maximum Mean Discrepancy between two embedding sets.

    MMD^2 = E[k(x,x')] - 2*E[k(x,y)] + E[k(y,y')]
    where x ~ P (baseline), y ~ Q (current), k is a kernel function.

    A permutation test provides p-values for significance.
    """
    # Subsample for computational efficiency
    if len(baseline_embeddings) > n_sample:
        idx = np.random.choice(len(baseline_embeddings), n_sample, replace=False)
        X = baseline_embeddings[idx]
    else:
        X = baseline_embeddings

    if len(current_embeddings) > n_sample:
        idx = np.random.choice(len(current_embeddings), n_sample, replace=False)
        Y = current_embeddings[idx]
    else:
        Y = current_embeddings

    # Set kernel bandwidth via median heuristic
    if gamma is None:
        combined = np.vstack([X[:100], Y[:100]])
        pairwise_dists = np.linalg.norm(
            combined[:, np.newaxis] - combined[np.newaxis, :], axis=2
        )
        median_dist = np.median(pairwise_dists[pairwise_dists > 0])
        gamma = 1.0 / (2.0 * median_dist ** 2)

    # Compute kernel matrices
    def rbf_kernel(A, B):
        sq_dists = np.sum(A ** 2, axis=1, keepdims=True) + np.sum(B ** 2, axis=1) - 2 * np.dot(A, B.T)
        return np.exp(-gamma * sq_dists)

    Kxx = rbf_kernel(X, X)
    Kyy = rbf_kernel(Y, Y)
    Kxy = rbf_kernel(X, Y)

    n = len(X)
    m = len(Y)

    # Unbiased MMD^2 estimator
    np.fill_diagonal(Kxx, 0)
    np.fill_diagonal(Kyy, 0)
    mmd2 = Kxx.sum() / (n * (n - 1)) + Kyy.sum() / (m * (m - 1)) - 2 * Kxy.mean()

    # Permutation test for p-value
    n_permutations = 200
    combined = np.vstack([X, Y])
    total = len(combined)
    perm_mmd2s = []

    for _ in range(n_permutations):
        perm = np.random.permutation(total)
        perm_X = combined[perm[:n]]
        perm_Y = combined[perm[n:n + m]]

        perm_Kxx = rbf_kernel(perm_X, perm_X)
        perm_Kyy = rbf_kernel(perm_Y, perm_Y)
        perm_Kxy = rbf_kernel(perm_X, perm_Y)

        np.fill_diagonal(perm_Kxx, 0)
        np.fill_diagonal(perm_Kyy, 0)
        perm_mmd2 = perm_Kxx.sum() / (n * (n - 1)) + perm_Kyy.sum() / (m * (m - 1)) - 2 * perm_Kxy.mean()
        perm_mmd2s.append(perm_mmd2)

    p_value = np.mean([p >= mmd2 for p in perm_mmd2s])

    return {
        "mmd2": float(mmd2),
        "mmd": float(np.sqrt(max(mmd2, 0))),
        "p_value": float(p_value),
        "significant": p_value < 0.05,
        "gamma": float(gamma),
    }
```

---

## Metric 5: Cosine Similarity Distribution Shift

Track the distribution of pairwise cosine similarities within the embedding set. A shift in this distribution indicates structural changes in the embedding space.

```python
import numpy as np
from scipy import stats


def similarity_distribution_shift(
    baseline_embeddings: np.ndarray,
    current_embeddings: np.ndarray,
    n_pairs: int = 10000,
) -> dict:
    """Compare the distribution of pairwise cosine similarities.

    Randomly samples pairs from each set and compares the distributions
    using KS test and descriptive statistics.
    """
    def sample_pairwise_sims(embeddings, n):
        n_embs = len(embeddings)
        idx_a = np.random.randint(0, n_embs, size=n)
        idx_b = np.random.randint(0, n_embs, size=n)
        # Avoid self-pairs
        mask = idx_a != idx_b
        idx_a = idx_a[mask]
        idx_b = idx_b[mask]
        sims = np.sum(embeddings[idx_a] * embeddings[idx_b], axis=1)
        return sims

    base_sims = sample_pairwise_sims(baseline_embeddings, n_pairs)
    curr_sims = sample_pairwise_sims(current_embeddings, n_pairs)

    # Kolmogorov-Smirnov test
    ks_stat, ks_pvalue = stats.ks_2samp(base_sims, curr_sims)

    return {
        "baseline_mean_sim": float(base_sims.mean()),
        "current_mean_sim": float(curr_sims.mean()),
        "mean_shift": float(curr_sims.mean() - base_sims.mean()),
        "baseline_std": float(base_sims.std()),
        "current_std": float(curr_sims.std()),
        "ks_statistic": float(ks_stat),
        "ks_pvalue": float(ks_pvalue),
        "significant_shift": ks_pvalue < 0.05,
    }
```

---

## Visualization

### Dashboard Plots

```python
import matplotlib
matplotlib.use("Agg")  # Non-interactive backend for servers
import matplotlib.pyplot as plt
import numpy as np


def plot_drift_dashboard(
    reports: list[dict],
    output_path: str = "drift_dashboard.png",
):
    """Generate a drift monitoring dashboard with four panels.

    Args:
        reports: List of drift check results, each with timestamp and metrics.
        output_path: Path to save the dashboard image.
    """
    fig, axes = plt.subplots(2, 2, figsize=(14, 10))
    fig.suptitle("Embedding Drift Monitoring Dashboard", fontsize=14, fontweight="bold")

    timestamps = [r["timestamp"] for r in reports]
    x = range(len(timestamps))

    # Panel 1: Centroid distance over time
    centroid_dists = [r.get("centroid_distance", 0) for r in reports]
    axes[0, 0].plot(x, centroid_dists, "b-o", markersize=3)
    axes[0, 0].axhline(y=0.05, color="orange", linestyle="--", label="Warning (0.05)")
    axes[0, 0].axhline(y=0.10, color="red", linestyle="--", label="Critical (0.10)")
    axes[0, 0].set_title("Centroid Distance")
    axes[0, 0].set_ylabel("Cosine Distance")
    axes[0, 0].legend(fontsize=8)

    # Panel 2: PSI over time
    psi_values = [r.get("mean_psi", 0) for r in reports]
    axes[0, 1].plot(x, psi_values, "g-o", markersize=3)
    axes[0, 1].axhline(y=0.10, color="orange", linestyle="--", label="Moderate (0.10)")
    axes[0, 1].axhline(y=0.25, color="red", linestyle="--", label="Significant (0.25)")
    axes[0, 1].set_title("Population Stability Index (PSI)")
    axes[0, 1].set_ylabel("PSI")
    axes[0, 1].legend(fontsize=8)

    # Panel 3: Mean KL divergence over time
    kl_values = [r.get("mean_kl", 0) for r in reports]
    axes[1, 0].plot(x, kl_values, "r-o", markersize=3)
    axes[1, 0].axhline(y=0.05, color="orange", linestyle="--", label="Warning (0.05)")
    axes[1, 0].axhline(y=0.20, color="red", linestyle="--", label="Critical (0.20)")
    axes[1, 0].set_title("Mean KL Divergence")
    axes[1, 0].set_ylabel("KL Divergence")
    axes[1, 0].legend(fontsize=8)

    # Panel 4: Retrieval quality (NDCG@10) over time
    ndcg_values = [r.get("ndcg_at_10", 0) for r in reports]
    axes[1, 1].plot(x, ndcg_values, "m-o", markersize=3)
    if ndcg_values:
        baseline = ndcg_values[0]
        axes[1, 1].axhline(y=baseline * 0.95, color="orange", linestyle="--", label="5% drop")
        axes[1, 1].axhline(y=baseline * 0.90, color="red", linestyle="--", label="10% drop")
    axes[1, 1].set_title("Retrieval Quality (NDCG@10)")
    axes[1, 1].set_ylabel("NDCG@10")
    axes[1, 1].legend(fontsize=8)

    plt.tight_layout()
    plt.savefig(output_path, dpi=150, bbox_inches="tight")
    plt.close()
    print(f"Dashboard saved to {output_path}")
```

### Similarity Distribution Comparison

```python
def plot_similarity_distributions(
    baseline_sims: np.ndarray,
    current_sims: np.ndarray,
    output_path: str = "similarity_distribution.png",
):
    """Plot baseline vs current pairwise similarity distributions."""
    fig, ax = plt.subplots(figsize=(10, 6))

    ax.hist(baseline_sims, bins=100, alpha=0.5, density=True, label="Baseline", color="blue")
    ax.hist(current_sims, bins=100, alpha=0.5, density=True, label="Current", color="red")
    ax.axvline(baseline_sims.mean(), color="blue", linestyle="--", linewidth=1)
    ax.axvline(current_sims.mean(), color="red", linestyle="--", linewidth=1)

    ax.set_xlabel("Cosine Similarity")
    ax.set_ylabel("Density")
    ax.set_title("Pairwise Similarity Distribution: Baseline vs Current")
    ax.legend()

    plt.tight_layout()
    plt.savefig(output_path, dpi=150)
    plt.close()
```

---

## Threshold Calibration

### Establishing Baseline Thresholds

Run drift metrics daily for the first 2-4 weeks (with no actual drift) to establish natural variation:

```python
def calibrate_thresholds(
    daily_metrics: list[dict],
    percentile: float = 99.0,
) -> dict:
    """Calibrate drift thresholds from natural variation data.

    Use 2-4 weeks of daily metrics where no drift occurred.
    Set thresholds at the 99th percentile of observed variation.
    """
    centroid_dists = [m["centroid_distance"] for m in daily_metrics]
    psi_values = [m["mean_psi"] for m in daily_metrics]
    kl_values = [m["mean_kl"] for m in daily_metrics]

    thresholds = {
        "centroid_distance": float(np.percentile(centroid_dists, percentile)),
        "mean_psi": float(np.percentile(psi_values, percentile)),
        "mean_kl": float(np.percentile(kl_values, percentile)),
    }

    print("Calibrated thresholds:")
    for metric, threshold in thresholds.items():
        print(f"  {metric}: {threshold:.6f}")

    return thresholds
```

---

## Common Pitfalls

1. **Using raw high-dimensional KL divergence.** KL divergence on 1024 dimensions requires enormous sample sizes for stability. Use PCA projection or per-dimension aggregation instead.
2. **Setting static thresholds.** Natural variation depends on your data. Calibrate thresholds from your own baseline period rather than using universal values.
3. **Monitoring only one metric.** Each metric captures different aspects of drift. Use centroid distance (location), PSI (shape), and similarity distribution (structure) together.
4. **Not accounting for sample size.** Small samples produce noisy metrics. Ensure at least 500 embeddings per sample for stable results. Use 1000+ for high-dimensional spaces.
5. **Forgetting to update baselines.** After confirmed, intentional changes (new model deployment, corpus expansion), update the baseline or all future readings will show false positives.

---

## References

- KL Divergence -- https://en.wikipedia.org/wiki/Kullback%E2%80%93Leibler_divergence
- Population Stability Index -- https://scholarworks.wmich.edu/dissertations/3208/
- Maximum Mean Discrepancy -- https://jmlr.csail.mit.edu/papers/v13/gretton12a.html
- Kolmogorov-Smirnov Test -- https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.ks_2samp.html
- Arize Embedding Monitoring -- https://docs.arize.com/arize/machine-learning/embeddings
