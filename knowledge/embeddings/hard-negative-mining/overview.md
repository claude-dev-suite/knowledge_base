# Hard Negative Mining -- Comprehensive Guide

## Overview / TL;DR

Hard negatives are documents that are superficially similar to a query (high keyword overlap or semantic proximity) but are not actually relevant. They are the most challenging and most informative examples for training embedding models. Using only random or easy negatives during fine-tuning teaches the model to distinguish obviously different texts -- a trivial task. Hard negatives force the model to learn subtle distinctions that matter for real retrieval. Adding hard negatives to training typically improves retrieval quality by 5-15% over in-batch-only negatives, making hard negative mining the single most impactful data preparation technique for embedding fine-tuning.

---

## What Are Hard Negatives?

### The Negative Difficulty Spectrum

```
Easy Negative (random)        Hard Negative (similar but wrong)       False Negative (actually relevant)
|                              |                                       |
"The weather is nice today"    "Machine learning uses statistics"       "ML algorithms learn from data"
                               (same domain, wrong answer)             (genuinely relevant, wrong label)

Query: "How does deep learning differ from machine learning?"
```

| Type | Example | Training Signal | Risk |
|------|---------|----------------|------|
| Random negative | Unrelated topic | Weak (model already knows this is different) | Wasted training step |
| Easy negative | Same broad domain | Moderate | Low |
| **Hard negative** | Similar topic, wrong answer | **Strong (forces fine-grained learning)** | **Optimal** |
| Very hard negative | Almost relevant | Very strong but risky | May actually be relevant (false negative) |
| False negative | Actually relevant | **Harmful (teaches wrong signal)** | Must avoid |

### Why In-Batch Negatives Are Not Enough

MultipleNegativesRankingLoss (MNRL) uses other examples in the same batch as negatives. With batch size 64, each query gets 63 negatives. However, these negatives are randomly sampled from the training set and are typically easy -- they come from different topics entirely.

| Negative Source | Typical Difficulty | Training Improvement |
|----------------|-------------------|---------------------|
| In-batch (random) only | Easy | Baseline |
| BM25-mined | Medium | +3-5% |
| Dense-mined | Hard | +5-10% |
| Cross-encoder scored | Very hard | +8-15% |

---

## Impact on Embedding Quality

### Experimental Results

| Training Configuration | NDCG@10 (Medical) | NDCG@10 (Legal) | NDCG@10 (General) |
|-----------------------|-------------------|------------------|--------------------|
| MNRL, in-batch only | 60.2 | 58.5 | 63.1 |
| MNRL + 1 random negative | 61.5 | 59.8 | 64.2 |
| MNRL + 1 BM25 negative | 63.8 | 62.1 | 66.0 |
| MNRL + 1 dense negative | 65.2 | 63.5 | 67.5 |
| MNRL + 3 dense negatives | 66.8 | 64.8 | 68.2 |
| MNRL + 7 dense negatives | 67.5 | 65.2 | 68.5 |
| MNRL + cross-encoder scored | 68.2 | 66.5 | 69.1 |

**Key finding**: Going from random to dense-mined negatives accounts for ~5 NDCG points improvement. Adding more negatives (1 -> 7) provides diminishing returns. Cross-encoder scoring provides the best quality but at highest computational cost.

---

## Why Hard Negatives Matter

### Learning Theory Perspective

Embedding models learn by contrast: pulling relevant pairs closer and pushing irrelevant pairs apart in the vector space. The gradient signal from a training example is proportional to how "surprising" the example is:

- **Easy negative** (similarity = 0.1): The model already knows this is irrelevant. Small gradient. Little learning.
- **Hard negative** (similarity = 0.7): The model thinks this is relevant, but it is not. Large gradient. Significant learning.

The optimal negative is one that the current model scores just below the positive -- hard enough to provide a strong signal, but not so hard that it is actually relevant (false negative).

### Practical Analogy

Teaching a medical student to distinguish diseases:
- **Easy test**: "Is this patient's cough caused by pneumonia or a broken arm?" (trivial)
- **Hard test**: "Is this patient's chest X-ray showing pneumonia or lung cancer?" (requires expertise)

Hard negatives are the hard tests that build real expertise.

---

## The False Negative Problem

The biggest risk in hard negative mining is accidentally including truly relevant documents as negatives. This teaches the model to push relevant documents away from queries -- the opposite of what you want.

### Mitigation Strategies

1. **Relevance filtering**: After mining, use a cross-encoder or LLM to verify that each negative is truly irrelevant.
2. **Score gap**: Only use negatives that score between a lower and upper similarity threshold (e.g., 0.3-0.7). Very high-scoring candidates are likely relevant.
3. **Denoised training**: Use loss functions that are robust to label noise (e.g., MarginMSE with soft labels from a teacher model).

```python
import numpy as np


def filter_false_negatives(
    query: str,
    candidate_negatives: list[str],
    cross_encoder,
    relevance_threshold: float = 0.5,
) -> list[str]:
    """Filter out candidates that might be false negatives.

    Uses a cross-encoder to score query-document relevance.
    Documents scoring above the threshold are removed (likely relevant).
    """
    pairs = [(query, neg) for neg in candidate_negatives]
    scores = cross_encoder.predict(pairs)

    filtered = [
        neg for neg, score in zip(candidate_negatives, scores)
        if score < relevance_threshold
    ]

    n_removed = len(candidate_negatives) - len(filtered)
    if n_removed > 0:
        print(f"Removed {n_removed} potential false negatives "
              f"({n_removed/len(candidate_negatives)*100:.1f}%)")

    return filtered
```

---

## Mining Strategy Overview

| Strategy | Speed | Quality | Cost | Best For |
|----------|-------|---------|------|----------|
| BM25 (sparse retrieval) | Fast | Medium | Free | First pass, keyword-overlap negatives |
| Dense retrieval | Medium | High | Free (local model) | Primary hard negative source |
| Cross-encoder scoring | Slow | Highest | Moderate (compute) | Final filtering and scoring |
| TAS-B (distillation) | Medium | High | Free | Combining teacher + student signals |
| Curriculum (easy -> hard) | Medium | High | Free | Gradual difficulty increase |

See `mining-strategies.md` in this directory for detailed implementation of each strategy.

---

## Quick Start: Basic Hard Negative Mining

```python
import numpy as np
from sentence_transformers import SentenceTransformer


def mine_hard_negatives_simple(
    queries: list[str],
    positives: list[str],
    corpus: list[str],
    model_name: str = "BAAI/bge-base-en-v1.5",
    n_negatives: int = 7,
    min_similarity: float = 0.1,
    max_similarity: float = 0.8,
) -> list[dict]:
    """Simple hard negative mining using dense retrieval.

    For each query, finds the most similar corpus documents
    that are NOT the positive, within a similarity range.
    """
    model = SentenceTransformer(model_name)

    # Build positive set for exclusion
    positive_set = set(positives)

    # Embed corpus
    print(f"Embedding {len(corpus)} documents...")
    corpus_embs = model.encode(corpus, normalize_embeddings=True, batch_size=256, show_progress_bar=True)

    training_data = []

    for i, (query, positive) in enumerate(zip(queries, positives)):
        if i % 500 == 0:
            print(f"Mining negatives for query {i}/{len(queries)}")

        query_emb = model.encode([query], normalize_embeddings=True)[0]
        scores = np.dot(corpus_embs, query_emb)

        # Filter by similarity range and exclude positive
        candidates = []
        for idx in np.argsort(scores)[::-1]:
            sim = scores[idx]
            if sim > max_similarity:
                continue  # Too similar, may be relevant
            if sim < min_similarity:
                break  # Too dissimilar, not useful
            if corpus[idx] in positive_set:
                continue  # Exclude known positives

            candidates.append(corpus[idx])
            if len(candidates) >= n_negatives:
                break

        training_data.append({
            "query": query,
            "positive": positive,
            "negatives": candidates,
        })

    return training_data
```

---

## Common Pitfalls

1. **Using only random negatives.** This is the #1 mistake. Random negatives provide weak training signal and leave 5-15% NDCG improvement on the table.
2. **Mining negatives from the same model being fine-tuned.** After fine-tuning, the hardest negatives may no longer be hard. Iterative mining (mine -> train -> mine again) addresses this.
3. **Not filtering false negatives.** Hard negatives that are actually relevant teach the model the wrong thing. Always verify with a cross-encoder or LLM.
4. **Using too many negatives.** Beyond 7-10 negatives per query, diminishing returns set in. The computational cost increases linearly but quality improvement is marginal.
5. **Mining negatives from a different corpus than production.** If you mine from Wikipedia but deploy on medical literature, the negatives will not be representative.

---

## References

- Xiong et al., "Approximate Nearest Neighbor Negative Contrastive Learning for Dense Text Retrieval" (ANCE) -- https://arxiv.org/abs/2007.00808
- Hofstatter et al., "Efficiently Teaching an Effective Dense Retriever with Balanced Topic Aware Sampling" (TAS-B) -- https://arxiv.org/abs/2104.06967
- Qu et al., "RocketQA: An Optimized Training Approach to Dense Passage Retrieval" -- https://arxiv.org/abs/2010.08191
- sentence-transformers Mining -- https://www.sbert.net/examples/training/ms_marco/README.html
