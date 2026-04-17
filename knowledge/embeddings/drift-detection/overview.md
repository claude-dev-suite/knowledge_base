# Embedding Drift Detection -- Comprehensive Guide

## Overview / TL;DR

Embedding drift is the gradual or sudden shift in the statistical distribution of embedding vectors over time. It manifests as queries or documents producing vectors that are systematically different from the baseline distribution, causing retrieval quality degradation without any obvious error. Drift has three root causes: model updates (the embedding model changes), data shift (the content being embedded changes), and code changes (preprocessing or chunking logic changes). This guide covers drift detection strategies, monitoring architecture, and response playbooks for production RAG systems.

---

## What Is Embedding Drift?

Embedding drift occurs when the distribution of embedding vectors in your system changes over time. There are two distinct types:

### Query Drift

The distribution of query embeddings shifts. This happens when:
- Users start asking different types of questions (new product launch, seasonal topics).
- Query preprocessing changes (new tokenization, new stop word list).
- The embedding model is updated (API provider rolls out a new version).

### Corpus Drift

The distribution of corpus (document) embeddings shifts. This happens when:
- New documents are added that cover different topics than the original corpus.
- Documents are updated with new terminology.
- Chunking strategy changes (different chunk sizes, different overlap).
- The embedding model is updated (requiring re-embedding).

### Cross-Distribution Drift

The relationship between query and corpus distributions changes. Even if neither distribution changes individually, the mapping between them can drift. This is the hardest to detect and the most impactful on retrieval quality.

---

## Causes of Embedding Drift

### Cause 1: Model Updates

API-based embedding providers (OpenAI, Cohere, Voyage) may update their models. Even "stable" model versions can have subtle changes in infrastructure that affect numerical outputs.

**Detection**: Compare embeddings of a fixed set of reference texts between time periods.

```python
import numpy as np
from openai import OpenAI

client = OpenAI()

# Reference texts that should produce stable embeddings
REFERENCE_TEXTS = [
    "Machine learning is a subset of artificial intelligence.",
    "Python is a popular programming language for data science.",
    "Kubernetes orchestrates containerized applications at scale.",
    "PostgreSQL is a powerful open-source relational database.",
    "React is a JavaScript library for building user interfaces.",
]


def compute_reference_embeddings(model: str = "text-embedding-3-small") -> np.ndarray:
    """Compute embeddings for reference texts. Store as baseline."""
    response = client.embeddings.create(model=model, input=REFERENCE_TEXTS)
    return np.array([item.embedding for item in response.data])


def check_model_drift(
    current_embeddings: np.ndarray,
    baseline_embeddings: np.ndarray,
    threshold: float = 0.001,
) -> dict:
    """Check if the embedding model has drifted from baseline.

    Compares reference text embeddings between current and baseline.
    If the model is unchanged, these should be identical.
    """
    # Per-reference-text cosine similarity
    similarities = []
    for i in range(len(REFERENCE_TEXTS)):
        sim = np.dot(current_embeddings[i], baseline_embeddings[i])
        similarities.append(sim)

    avg_sim = np.mean(similarities)
    min_sim = np.min(similarities)

    drifted = (1.0 - avg_sim) > threshold

    return {
        "drifted": drifted,
        "avg_similarity": float(avg_sim),
        "min_similarity": float(min_sim),
        "per_reference": similarities,
        "drift_magnitude": float(1.0 - avg_sim),
    }
```

### Cause 2: Data Shift

New documents cover topics not represented in the original corpus. The centroid and spread of the corpus embedding distribution changes.

```python
import numpy as np


def detect_data_shift(
    baseline_embeddings: np.ndarray,
    new_embeddings: np.ndarray,
) -> dict:
    """Detect shift in corpus embeddings by comparing distributions.

    Uses centroid distance and variance change as indicators.
    """
    # Centroid comparison
    baseline_centroid = baseline_embeddings.mean(axis=0)
    new_centroid = new_embeddings.mean(axis=0)
    centroid_distance = 1.0 - np.dot(
        baseline_centroid / np.linalg.norm(baseline_centroid),
        new_centroid / np.linalg.norm(new_centroid),
    )

    # Variance comparison
    baseline_var = np.var(baseline_embeddings, axis=0).mean()
    new_var = np.var(new_embeddings, axis=0).mean()
    variance_ratio = new_var / baseline_var

    # Inter-cluster similarity
    n_sample = min(1000, len(baseline_embeddings), len(new_embeddings))
    sample_base = baseline_embeddings[np.random.choice(len(baseline_embeddings), n_sample, replace=False)]
    sample_new = new_embeddings[np.random.choice(len(new_embeddings), n_sample, replace=False)]

    cross_sims = np.dot(sample_base, sample_new.T)
    avg_cross_sim = cross_sims.mean()

    within_base_sims = np.dot(sample_base, sample_base.T)
    np.fill_diagonal(within_base_sims, 0)
    avg_within_base = within_base_sims.sum() / (n_sample * (n_sample - 1))

    return {
        "centroid_distance": float(centroid_distance),
        "variance_ratio": float(variance_ratio),
        "avg_cross_similarity": float(avg_cross_sim),
        "avg_within_baseline_similarity": float(avg_within_base),
        "similarity_drop": float(avg_within_base - avg_cross_sim),
    }
```

### Cause 3: Code Changes

Preprocessing changes (tokenization, text cleaning, chunking) produce different text inputs to the same model, resulting in different embeddings.

**Detection**: Hash the preprocessing pipeline configuration and compare over time. Any change in the hash should trigger a re-embedding alert.

---

## Consequences of Undetected Drift

| Consequence | Severity | Example |
|------------|----------|---------|
| Retrieval quality degradation | High | Relevant docs no longer appear in top-10 |
| Inconsistent user experience | Medium | Same query returns different results on different days |
| Re-ranker confusion | Medium | Re-ranker trained on original distribution underperforms |
| Cache invalidation | Low | Cached queries return stale results |
| Evaluation metric noise | Medium | A/B tests become unreliable due to drift |

Undetected drift can cause a 10-25% drop in retrieval quality over weeks without any obvious error in logs or monitoring. The system still returns results -- just worse ones.

---

## Monitoring Strategy

### Three-Layer Monitoring

1. **Model stability monitoring**: Compare reference text embeddings periodically. If they change, the model has been updated.

2. **Distribution monitoring**: Track the centroid, spread, and density of both query and corpus embedding distributions. Significant shifts indicate data drift.

3. **Quality monitoring**: Track retrieval metrics (Hit@10, MRR) on a golden evaluation set. This is the ground truth -- if metrics drop, something has drifted.

```python
import json
import time
from datetime import datetime
from dataclasses import dataclass, asdict


@dataclass
class DriftReport:
    """Periodic drift monitoring report."""
    timestamp: str
    model_drift: dict
    query_distribution: dict
    corpus_distribution: dict
    quality_metrics: dict
    alerts: list[str]


class EmbeddingDriftMonitor:
    """Comprehensive embedding drift monitoring system."""

    def __init__(
        self,
        baseline_reference_embs: np.ndarray,
        baseline_query_sample: np.ndarray,
        baseline_corpus_sample: np.ndarray,
        baseline_quality: dict,
    ):
        self.baseline_ref = baseline_reference_embs
        self.baseline_query = baseline_query_sample
        self.baseline_corpus = baseline_corpus_sample
        self.baseline_quality = baseline_quality
        self.reports: list[DriftReport] = []

    def run_check(
        self,
        current_reference_embs: np.ndarray,
        current_query_sample: np.ndarray,
        current_corpus_sample: np.ndarray,
        current_quality: dict,
    ) -> DriftReport:
        """Run a comprehensive drift check."""
        alerts = []

        # Layer 1: Model stability
        model_drift = check_model_drift(current_reference_embs, self.baseline_ref)
        if model_drift["drifted"]:
            alerts.append(f"MODEL_DRIFT: similarity={model_drift['avg_similarity']:.6f}")

        # Layer 2: Distribution monitoring
        query_dist = detect_data_shift(self.baseline_query, current_query_sample)
        if query_dist["centroid_distance"] > 0.05:
            alerts.append(f"QUERY_DRIFT: centroid_distance={query_dist['centroid_distance']:.4f}")

        corpus_dist = detect_data_shift(self.baseline_corpus, current_corpus_sample)
        if corpus_dist["centroid_distance"] > 0.05:
            alerts.append(f"CORPUS_DRIFT: centroid_distance={corpus_dist['centroid_distance']:.4f}")

        # Layer 3: Quality monitoring
        quality_drop = {}
        for metric in self.baseline_quality:
            if metric in current_quality:
                drop = self.baseline_quality[metric] - current_quality[metric]
                quality_drop[metric] = drop
                if drop > 0.05:  # 5% quality drop threshold
                    alerts.append(f"QUALITY_DROP: {metric} dropped by {drop:.4f}")

        report = DriftReport(
            timestamp=datetime.utcnow().isoformat(),
            model_drift=model_drift,
            query_distribution=query_dist,
            corpus_distribution=corpus_dist,
            quality_metrics={"current": current_quality, "drop": quality_drop},
            alerts=alerts,
        )

        self.reports.append(report)
        return report
```

---

## Sampling Strategy

You do not need to monitor every embedding. Sample strategically:

```python
import numpy as np
from collections import deque


class EmbeddingSampler:
    """Collect embedding samples for drift detection.

    Uses reservoir sampling to maintain a fixed-size representative sample
    of recent embeddings without storing everything.
    """

    def __init__(self, sample_size: int = 1000, window_hours: int = 24):
        self.sample_size = sample_size
        self.samples: deque = deque(maxlen=sample_size * 2)
        self.count = 0

    def add(self, embedding: np.ndarray):
        """Add an embedding to the sample using reservoir sampling."""
        self.count += 1

        if len(self.samples) < self.sample_size:
            self.samples.append(embedding.copy())
        else:
            # Reservoir sampling: replace with probability sample_size/count
            j = np.random.randint(0, self.count)
            if j < self.sample_size:
                self.samples[j] = embedding.copy()

    def get_sample(self) -> np.ndarray:
        """Return the current sample as a numpy array."""
        if not self.samples:
            return np.array([])
        return np.array(list(self.samples))

    def reset(self):
        """Reset the sampler for a new monitoring period."""
        self.samples.clear()
        self.count = 0
```

---

## Response Playbook

When drift is detected, follow this playbook:

### Alert: MODEL_DRIFT

```
Root cause: The embedding model was updated by the provider.

Response:
1. Confirm by computing reference embeddings again. If they differ from
   yesterday's reference, the model changed.
2. Re-embed the ENTIRE corpus with the new model.
3. Update the baseline reference embeddings.
4. Re-run the evaluation set and update quality baselines.
5. If quality dropped, consider rolling back to a pinned model version.
```

### Alert: QUERY_DRIFT

```
Root cause: Users are asking different types of questions.

Response:
1. Analyze the query distribution shift (new topics? new query patterns?).
2. If quality metrics are stable, no action needed (natural usage evolution).
3. If quality dropped, the corpus may need expansion to cover new topics.
4. Update the query sample baseline to reflect new normal.
```

### Alert: CORPUS_DRIFT

```
Root cause: New documents cover different topics than the original corpus.

Response:
1. Check if new documents were added (expected behavior).
2. Verify that new documents are being embedded correctly.
3. If the corpus expanded significantly, re-evaluate chunk sizes and overlap.
4. Update the corpus sample baseline.
```

### Alert: QUALITY_DROP

```
Root cause: Retrieval quality has degraded. May be caused by any of the above.

Response:
1. Run the full evaluation set to confirm the drop.
2. Check for model drift (reference embeddings changed?).
3. Check for preprocessing changes (chunking, text cleaning).
4. Check for corpus changes (new documents with different characteristics).
5. If no root cause found, investigate the evaluation set itself
   (is it still representative?).
```

---

## Common Pitfalls

1. **Only monitoring quality metrics, not distributions.** Quality metrics on a golden eval set detect drift only after it has impacted retrieval. Distribution monitoring catches drift earlier.
2. **Not establishing a baseline.** Without a baseline distribution, you cannot detect drift. Record baseline embeddings for reference texts, query samples, and corpus samples at deployment time.
3. **Ignoring silent model updates.** API providers may update models without notification. Always monitor reference text embeddings.
4. **Setting thresholds too tight.** Natural variation exists in embedding distributions. Setting drift thresholds too tight causes alert fatigue. Calibrate thresholds using the first 2-4 weeks of production data.
5. **Not having a response playbook.** Detecting drift is useless if the team does not know how to respond. Document response procedures before the first alert fires.

---

## References

- Arize Embedding Drift Monitoring -- https://docs.arize.com/arize/machine-learning/embeddings
- Evidently AI Embedding Drift -- https://docs.evidentlyai.com/user-guide/customization/embeddings-drift
- Population Stability Index -- https://en.wikipedia.org/wiki/Population_Stability_Index
- Maximum Mean Discrepancy -- https://jmlr.csail.mit.edu/papers/v13/gretton12a.html
