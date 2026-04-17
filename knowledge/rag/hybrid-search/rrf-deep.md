# Reciprocal Rank Fusion -- Deep Dive

## TL;DR

Reciprocal Rank Fusion (RRF) combines multiple ranked result lists into a single ranking by summing `1/(k + rank)` across all lists for each document. It is the standard fusion method for hybrid search because it requires no score normalization, is parameter-light (only `k`), and consistently outperforms individual rankers. This article covers the mathematical foundation, implementation from scratch, tuning the `k` parameter, comparison with alternatives, and multi-list extension beyond two retrievers.

---

## The RRF Formula

### Definition

Given `n` ranked lists `L_1, L_2, ..., L_n`, the RRF score for document `d` is:

```
RRF(d) = SUM_{i=1}^{n} 1 / (k + rank_i(d))
```

Where:
- `rank_i(d)` is the 1-based rank of document `d` in list `L_i` (1 = highest ranked)
- If `d` does not appear in list `L_i`, that term is 0 (document gets no contribution from that list)
- `k` is a constant that controls rank damping (default: 60)

### Mathematical Intuition

The function `1/(k + rank)` is a hyperbolic decay. The key properties:

1. **Rank 1 contribution**: `1/(k+1)`. With k=60, this is `1/61 = 0.01639`.
2. **Rank 10 contribution**: `1/(k+10) = 1/70 = 0.01429`. Only 12.8% less than rank 1.
3. **Rank 100 contribution**: `1/(k+100) = 1/160 = 0.00625`. 62% less than rank 1.
4. **Rank 1000 contribution**: `1/(k+1000) = 1/1060 = 0.000943`. Negligible.

The high `k` value (60) creates a "soft" decay: the difference between rank 1 and rank 10 is small, but between rank 10 and rank 100 it is substantial. This means RRF rewards documents that appear consistently across lists rather than documents that rank extremely high in just one list.

**Contrast with `k=1`**: contribution at rank 1 is `1/2 = 0.5`, at rank 10 is `1/11 = 0.091` -- a 5.5x difference. Low `k` makes RRF more "winner-take-all", heavily favoring top-ranked documents in any single list.

### Score Range

For a document appearing at rank 1 in all `n` lists:

```
max_score = n / (k + 1)
```

For two lists with k=60: `max_score = 2/61 = 0.03279`

This score has no absolute meaning -- it is only useful for relative ordering.

---

## Implementation from Scratch

### Basic Python Implementation

```python
from collections import defaultdict


def reciprocal_rank_fusion(
    ranked_lists: list[list[str]],
    k: int = 60,
) -> list[tuple[str, float]]:
    """
    Fuse multiple ranked result lists using Reciprocal Rank Fusion.

    Args:
        ranked_lists: List of ranked lists, each containing document IDs
                      in descending relevance order.
        k: Damping constant. Higher k = more equal weighting across ranks.

    Returns:
        List of (doc_id, rrf_score) tuples sorted by score descending.
    """
    rrf_scores: dict[str, float] = defaultdict(float)

    for result_list in ranked_lists:
        for rank_position, doc_id in enumerate(result_list, start=1):
            rrf_scores[doc_id] += 1.0 / (k + rank_position)

    return sorted(rrf_scores.items(), key=lambda x: x[1], reverse=True)


# Example usage
vector_results = ["doc_A", "doc_B", "doc_C", "doc_D", "doc_E"]
bm25_results = ["doc_C", "doc_A", "doc_F", "doc_B", "doc_G"]

fused = reciprocal_rank_fusion([vector_results, bm25_results], k=60)

# doc_A: 1/(60+1) + 1/(60+2) = 0.01639 + 0.01613 = 0.03252
# doc_C: 1/(60+3) + 1/(60+1) = 0.01587 + 0.01639 = 0.03226
# doc_B: 1/(60+2) + 1/(60+4) = 0.01613 + 0.01563 = 0.03175
# doc_D: 1/(60+4) + 0         = 0.01563
# doc_E: 1/(60+5) + 0         = 0.01538
# doc_F: 0         + 1/(60+3) = 0.01587
# doc_G: 0         + 1/(60+5) = 0.01538
```

### Weighted RRF

When you trust one retriever more than another, multiply each contribution by a weight:

```python
def weighted_rrf(
    ranked_lists: list[list[str]],
    weights: list[float],
    k: int = 60,
) -> list[tuple[str, float]]:
    """
    Weighted Reciprocal Rank Fusion.

    Args:
        ranked_lists: List of ranked result lists.
        weights: Weight for each list (should sum to 1.0 for interpretability).
        k: Damping constant.
    """
    assert len(ranked_lists) == len(weights), "Must provide one weight per list"
    rrf_scores: dict[str, float] = defaultdict(float)

    for result_list, weight in zip(ranked_lists, weights):
        for rank_position, doc_id in enumerate(result_list, start=1):
            rrf_scores[doc_id] += weight / (k + rank_position)

    return sorted(rrf_scores.items(), key=lambda x: x[1], reverse=True)


# Trust vector search more for conceptual queries
fused = weighted_rrf(
    [vector_results, bm25_results],
    weights=[0.7, 0.3],
    k=60,
)
```

### RRF with Score Passthrough

Sometimes you want the RRF ranking but also need access to the original scores for downstream processing (e.g., confidence thresholds):

```python
from dataclasses import dataclass


@dataclass
class FusedResult:
    doc_id: str
    rrf_score: float
    source_scores: dict[str, float]  # retriever_name -> original score
    source_ranks: dict[str, int]     # retriever_name -> rank


def rrf_with_metadata(
    ranked_lists: dict[str, list[tuple[str, float]]],
    k: int = 60,
) -> list[FusedResult]:
    """
    RRF that preserves original scores and ranks.

    Args:
        ranked_lists: {retriever_name: [(doc_id, score), ...]} in rank order.
    """
    rrf_scores: dict[str, float] = defaultdict(float)
    source_scores: dict[str, dict[str, float]] = defaultdict(dict)
    source_ranks: dict[str, dict[str, int]] = defaultdict(dict)

    for retriever_name, results in ranked_lists.items():
        for rank, (doc_id, score) in enumerate(results, start=1):
            rrf_scores[doc_id] += 1.0 / (k + rank)
            source_scores[doc_id][retriever_name] = score
            source_ranks[doc_id][retriever_name] = rank

    fused = []
    for doc_id, rrf_score in sorted(
        rrf_scores.items(), key=lambda x: x[1], reverse=True
    ):
        fused.append(FusedResult(
            doc_id=doc_id,
            rrf_score=rrf_score,
            source_scores=source_scores[doc_id],
            source_ranks=source_ranks[doc_id],
        ))
    return fused


# Usage
results = rrf_with_metadata({
    "vector": [("doc_A", 0.92), ("doc_B", 0.87), ("doc_C", 0.81)],
    "bm25":   [("doc_C", 14.2), ("doc_A", 11.8), ("doc_D", 9.3)],
})

for r in results[:5]:
    print(f"{r.doc_id}: RRF={r.rrf_score:.5f} | ranks={r.source_ranks}")
```

---

## Multi-List RRF (More Than 2 Retrievers)

RRF naturally extends to any number of ranked lists. This is useful when combining:

- Dense vector search (e.g., OpenAI embeddings)
- BM25 keyword search
- Metadata-boosted search (e.g., recency-weighted)
- Multi-embedding search (e.g., title embedding + content embedding)
- ColBERT retrieval

```python
# Three-retriever fusion
dense_results = ["A", "B", "C", "D", "E"]
bm25_results = ["C", "A", "F", "B", "G"]
colbert_results = ["A", "C", "B", "G", "H"]

fused = reciprocal_rank_fusion(
    [dense_results, bm25_results, colbert_results],
    k=60,
)
# doc_A appears in all 3 lists at ranks 1, 2, 1:
# 1/61 + 1/62 + 1/61 = 0.04892 -- highest score
```

### Weighted Multi-List

```python
fused = weighted_rrf(
    [dense_results, bm25_results, colbert_results],
    weights=[0.4, 0.3, 0.3],
    k=60,
)
```

### When to Use >2 Retrievers

| Configuration | When |
|--------------|------|
| Dense + BM25 | Default hybrid (covers 90% of cases) |
| Dense + BM25 + ColBERT | When ColBERT is available and you need maximum recall |
| Dense + BM25 + recency-boosted | Time-sensitive domains (news, changelogs, CVEs) |
| Title-embedding + Content-embedding + BM25 | When titles carry distinct information from body text |
| Multi-model dense (e5 + voyage) | Hedging across embedding models for robustness |

---

## Tuning the k Parameter

### What k Controls

`k` controls how quickly RRF score decays with rank:

| k value | Rank 1 score | Rank 10 score | Rank 1 / Rank 10 ratio |
|---------|-------------|--------------|----------------------|
| 1 | 0.500 | 0.091 | 5.50x |
| 10 | 0.091 | 0.050 | 1.82x |
| 60 | 0.016 | 0.014 | 1.14x |
| 100 | 0.010 | 0.009 | 1.11x |
| 1000 | 0.001 | 0.001 | 1.01x |

**Low k (1-10)**: Strongly favors top-ranked documents. Good when you trust each individual retriever's top results and want to boost documents that are #1 in any list.

**Medium k (20-60)**: Balanced. Rewards consistency across lists. The default `k=60` from the original paper (Cormack et al. 2009) works well in most settings.

**High k (100-1000)**: Nearly equal weight for all ranked positions. Essentially counts "how many lists contain this document" rather than "where it ranks". Good when ranks within individual lists are noisy.

### How to Tune k Empirically

```python
from ragas.metrics import context_precision, context_recall
from ragas import evaluate

def evaluate_rrf_k(
    vector_results: list[list[str]],
    bm25_results: list[list[str]],
    golden_relevance: list[list[str]],
    k_values: list[int],
) -> dict[int, dict[str, float]]:
    """Evaluate RRF across multiple k values."""
    results = {}
    for k in k_values:
        fused_per_query = []
        for v_res, b_res in zip(vector_results, bm25_results):
            fused = reciprocal_rank_fusion([v_res, b_res], k=k)
            fused_per_query.append([doc_id for doc_id, _ in fused])

        # Compute retrieval metrics
        precision = compute_precision_at_k(fused_per_query, golden_relevance, k=10)
        recall = compute_recall_at_k(fused_per_query, golden_relevance, k=10)
        ndcg = compute_ndcg(fused_per_query, golden_relevance, k=10)

        results[k] = {"P@10": precision, "R@10": recall, "NDCG@10": ndcg}
        print(f"k={k:4d}: P@10={precision:.4f}  R@10={recall:.4f}  NDCG@10={ndcg:.4f}")

    return results

# Typical search
k_values = [1, 5, 10, 20, 40, 60, 80, 100, 200]
results = evaluate_rrf_k(vector_lists, bm25_lists, golden, k_values)
```

### Guidelines for k Selection

- **Start with k=60** (the well-tested default).
- If your individual retrievers are high-quality and you want to trust their top results, try lower k (20-40).
- If your retrievers are noisy or return many partially-relevant results, try higher k (80-100).
- k has diminishing impact above ~200 -- all ranks become nearly equal.

---

## RRF vs. Alternatives

### RRF vs. Weighted Linear Combination

| Aspect | RRF | Weighted Linear |
|--------|-----|----------------|
| Requires score normalization | No | Yes |
| Handles different score scales | Automatically | Must normalize |
| Parameter complexity | Just k | Per-retriever weights + normalization method |
| Sensitivity to outliers | Low (rank-based) | High (one extreme score dominates) |
| Fine-grained control | Limited | More granular |
| Standard approach | SIGIR 2009, widely adopted | Common but brittle |

**Recommendation**: Use RRF as the default. Switch to weighted linear only if you have well-calibrated scores and need fine-grained per-retriever control, and you have an eval set to validate.

### RRF vs. CombMNZ

CombMNZ multiplies the normalized sum of scores by the number of lists that return the document:

```
CombMNZ(d) = |{lists containing d}| * SUM(normalized_score_i(d))
```

CombMNZ rewards documents appearing in more lists (the multiplier) but requires score normalization. RRF achieves a similar "consensus bonus" implicitly through rank summation without needing normalization.

### RRF vs. Borda Count

Borda count assigns `(N - rank)` points where N is the number of candidates. It is equivalent to RRF with `k=0` and inverted (counting from the bottom). Borda gives too much credit to exact rank position and is sensitive to the number of candidates. RRF's damping constant `k` makes it more robust.

### RRF vs. Learned Fusion (LambdaMART, Neural)

Learned fusion models (LambdaMART, neural rankers) can outperform RRF when trained on sufficient in-domain data. However:

- They require labeled training data
- They can overfit to the training distribution
- They add model serving complexity
- RRF is a strong unsupervised baseline that learned models must beat to justify their cost

**Practical guideline**: Start with RRF. Only invest in learned fusion if RRF leaves measurable gaps on your eval set AND you have 1000+ labeled query-relevance pairs.

---

## RRF in Existing Systems

### Elasticsearch / OpenSearch

Both support RRF natively:

```json
{
  "retriever": {
    "rrf": {
      "retrievers": [
        {
          "standard": {
            "query": {
              "match": { "content": "PostgreSQL RLS" }
            }
          }
        },
        {
          "knn": {
            "field": "embedding",
            "query_vector": [0.1, 0.2, ...],
            "k": 20,
            "num_candidates": 100
          }
        }
      ],
      "rank_constant": 60,
      "rank_window_size": 100
    }
  }
}
```

### Weaviate

Weaviate's `RANKED` fusion type uses RRF internally:

```python
results = collection.query.hybrid(
    query="search text",
    alpha=0.5,
    fusion_type=HybridFusion.RANKED,  # This is RRF
    limit=10,
)
```

### Qdrant

```python
results = client.query_points(
    collection_name="docs",
    prefetch=[dense_prefetch, sparse_prefetch],
    query=models.FusionQuery(fusion=models.Fusion.RRF),
    limit=10,
)
```

---

## Edge Cases and Gotchas

1. **Unequal list lengths.** If vector search returns 50 results and BM25 returns 5, documents only in the vector list get contributions from ranks 1-50 while the BM25 list only contributes at ranks 1-5. This biases toward the longer list. **Fix**: truncate all lists to the same length before fusion.

2. **Tie-breaking.** When two documents have identical RRF scores, break ties by the best individual rank across all lists:

```python
def rrf_with_tiebreaker(ranked_lists, k=60):
    scores = defaultdict(float)
    best_rank = defaultdict(lambda: float('inf'))
    for results in ranked_lists:
        for rank, doc_id in enumerate(results, 1):
            scores[doc_id] += 1.0 / (k + rank)
            best_rank[doc_id] = min(best_rank[doc_id], rank)
    return sorted(
        scores.items(),
        key=lambda x: (x[1], -best_rank[x[0]]),
        reverse=True,
    )
```

3. **0-based vs 1-based ranking.** The original paper uses 1-based ranks. Using 0-based ranks changes the score magnitudes. Always use 1-based to match the standard formula: `rank_i(d) starts at 1`.

4. **Empty lists.** If one retriever returns no results (e.g., BM25 finds no keyword match), RRF gracefully degenerates to ranking by the other retriever only. No special handling needed.

5. **Duplicate documents.** If the same document appears multiple times in a single list (from different chunks of the same source), deduplicate within each list before fusion. Use the best (lowest) rank for each unique document.

---

## Common Pitfalls

1. **Using k=0 or k=1.** Extreme low k makes RRF degenerate into a winner-take-all system where rank-1 documents dominate. The smoothing effect that makes RRF robust comes from `k >= 10`.
2. **Fusing lists of different lengths without truncation.** The longer list gets more cumulative score contribution. Truncate all lists to the same length.
3. **Applying RRF to score lists instead of rank lists.** RRF takes ranked document IDs, not scores. If you pass scores, you must sort first.
4. **Not evaluating k.** The default k=60 is good but not universally optimal. A quick grid search over [20, 40, 60, 80, 100] takes minutes and can improve NDCG by 1-3%.
5. **Confusing RRF with MRR.** Mean Reciprocal Rank (MRR) is an evaluation metric. Reciprocal Rank Fusion (RRF) is a fusion strategy. They share the `1/rank` formula shape but serve entirely different purposes.

---

## References

- Cormack, Clarke, Buettcher. "Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods" (SIGIR 2009)
- Elasticsearch RRF docs: https://www.elastic.co/guide/en/elasticsearch/reference/current/rrf.html
- Benham & Culpepper. "Risk-Reward Trade-offs in Rank Fusion" (ADCS 2017)
- Lin, Ma, et al. "Pyserini: A Python Toolkit for Reproducible Information Retrieval Research with Sparse and Dense Representations" (SIGIR 2021)
