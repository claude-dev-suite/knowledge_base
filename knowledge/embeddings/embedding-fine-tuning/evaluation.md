# Evaluating Fine-Tuned Embeddings

## Overview / TL;DR

Evaluating fine-tuned embedding models requires retrieval-specific metrics (Hit@K, MRR, NDCG@10), a well-constructed golden evaluation set, comparison against the base model on your corpus, and automated regression testing for ongoing quality assurance. This guide covers each metric with formulas and code, explains how to construct a golden eval set for embeddings, provides A/B testing methodology, and shows how to integrate embedding quality checks into CI/CD pipelines. Without rigorous evaluation, you cannot know whether fine-tuning actually improved retrieval or introduced regressions.

---

## Retrieval Metrics

### Hit@K (Accuracy@K)

The fraction of queries for which at least one relevant document appears in the top-K results.

```
Hit@K = (number of queries with at least one relevant doc in top-K) / (total queries)
```

**Interpretation**: Hit@5 = 0.85 means 85% of queries have a relevant result in the top 5. This is the most intuitive metric -- "how often does retrieval work?"

```python
import numpy as np


def hit_at_k(
    query_embeddings: np.ndarray,
    doc_embeddings: np.ndarray,
    relevant_doc_indices: list[set[int]],
    k: int = 10,
) -> float:
    """Compute Hit@K for a set of queries.

    Args:
        query_embeddings: (n_queries, dims) array.
        doc_embeddings: (n_docs, dims) array.
        relevant_doc_indices: For each query, a set of relevant document indices.
        k: Number of top results to consider.

    Returns:
        Hit@K score between 0 and 1.
    """
    hits = 0
    scores = np.dot(query_embeddings, doc_embeddings.T)

    for i in range(len(query_embeddings)):
        top_k_indices = np.argsort(scores[i])[::-1][:k]
        if any(idx in relevant_doc_indices[i] for idx in top_k_indices):
            hits += 1

    return hits / len(query_embeddings)
```

### Mean Reciprocal Rank (MRR)

The average of 1/rank for the first relevant document across all queries.

```
MRR = (1/N) * sum(1/rank_i for i in queries)
```

**Interpretation**: MRR = 0.75 means the first relevant document is, on average, around rank 1.33. Higher MRR means relevant docs appear earlier in results.

```python
def mrr_at_k(
    query_embeddings: np.ndarray,
    doc_embeddings: np.ndarray,
    relevant_doc_indices: list[set[int]],
    k: int = 10,
) -> float:
    """Compute MRR@K.

    Only considers the first relevant document within the top K results.
    """
    rr_sum = 0.0
    scores = np.dot(query_embeddings, doc_embeddings.T)

    for i in range(len(query_embeddings)):
        top_k_indices = np.argsort(scores[i])[::-1][:k]
        for rank, idx in enumerate(top_k_indices, start=1):
            if idx in relevant_doc_indices[i]:
                rr_sum += 1.0 / rank
                break

    return rr_sum / len(query_embeddings)
```

### NDCG@K (Normalized Discounted Cumulative Gain)

Measures the quality of ranking, accounting for both relevance and position. Higher positions are weighted more heavily (logarithmic discount).

```
DCG@K = sum(rel_i / log2(i + 1) for i in range(1, K+1))
NDCG@K = DCG@K / IDCG@K  (normalized by ideal ranking)
```

**Interpretation**: NDCG@10 = 0.80 means the ranking achieves 80% of the quality of a perfect ranking. This is the most comprehensive single metric for retrieval quality.

```python
def ndcg_at_k(
    query_embeddings: np.ndarray,
    doc_embeddings: np.ndarray,
    relevance_scores: list[dict[int, float]],  # {doc_idx: relevance_score}
    k: int = 10,
) -> float:
    """Compute NDCG@K with graded relevance.

    Args:
        relevance_scores: For each query, a dict mapping doc_idx to relevance score.
            Binary relevance: {doc_idx: 1.0} for relevant, absent for irrelevant.
            Graded relevance: {doc_idx: 0.0-3.0} for different levels.
    """
    ndcg_sum = 0.0
    scores = np.dot(query_embeddings, doc_embeddings.T)

    for i in range(len(query_embeddings)):
        top_k_indices = np.argsort(scores[i])[::-1][:k]

        # DCG: actual ranking
        dcg = 0.0
        for rank, idx in enumerate(top_k_indices):
            rel = relevance_scores[i].get(idx, 0.0)
            dcg += rel / np.log2(rank + 2)  # rank starts at 0, log2(1) = 0

        # IDCG: ideal ranking (sort by relevance)
        ideal_rels = sorted(relevance_scores[i].values(), reverse=True)[:k]
        idcg = sum(rel / np.log2(rank + 2) for rank, rel in enumerate(ideal_rels))

        if idcg > 0:
            ndcg_sum += dcg / idcg

    return ndcg_sum / len(query_embeddings)
```

### Combined Evaluation Function

```python
from dataclasses import dataclass


@dataclass
class RetrievalMetrics:
    hit_at_1: float
    hit_at_5: float
    hit_at_10: float
    mrr_at_10: float
    ndcg_at_10: float
    map_at_10: float


def evaluate_retrieval(
    query_embeddings: np.ndarray,
    doc_embeddings: np.ndarray,
    relevant_doc_indices: list[set[int]],
) -> RetrievalMetrics:
    """Compute all retrieval metrics in a single pass."""
    scores = np.dot(query_embeddings, doc_embeddings.T)
    n_queries = len(query_embeddings)

    hit_1 = hit_5 = hit_10 = 0
    mrr_sum = 0.0
    ndcg_sum = 0.0
    map_sum = 0.0

    for i in range(n_queries):
        ranked_indices = np.argsort(scores[i])[::-1]

        # Hit@K
        top_1 = set(ranked_indices[:1])
        top_5 = set(ranked_indices[:5])
        top_10 = set(ranked_indices[:10])
        relevant = relevant_doc_indices[i]

        if top_1 & relevant:
            hit_1 += 1
        if top_5 & relevant:
            hit_5 += 1
        if top_10 & relevant:
            hit_10 += 1

        # MRR@10
        for rank, idx in enumerate(ranked_indices[:10], start=1):
            if idx in relevant:
                mrr_sum += 1.0 / rank
                break

        # NDCG@10
        dcg = 0.0
        for rank, idx in enumerate(ranked_indices[:10]):
            if idx in relevant:
                dcg += 1.0 / np.log2(rank + 2)
        ideal_rels = min(len(relevant), 10)
        idcg = sum(1.0 / np.log2(r + 2) for r in range(ideal_rels))
        if idcg > 0:
            ndcg_sum += dcg / idcg

        # MAP@10
        relevant_found = 0
        precision_sum = 0.0
        for rank, idx in enumerate(ranked_indices[:10], start=1):
            if idx in relevant:
                relevant_found += 1
                precision_sum += relevant_found / rank
        if relevant:
            map_sum += precision_sum / min(len(relevant), 10)

    return RetrievalMetrics(
        hit_at_1=hit_1 / n_queries,
        hit_at_5=hit_5 / n_queries,
        hit_at_10=hit_10 / n_queries,
        mrr_at_10=mrr_sum / n_queries,
        ndcg_at_10=ndcg_sum / n_queries,
        map_at_10=map_sum / n_queries,
    )
```

---

## Golden Evaluation Set Construction

A golden eval set is a curated set of (query, relevant_documents) pairs used to measure retrieval quality. This is the most important artifact in your embedding evaluation pipeline.

### Requirements

1. **Representative queries**: Queries should reflect actual user behavior, not synthetic or toy examples.
2. **Complete relevance labels**: For each query, ALL relevant documents in the corpus should be labeled (not just the top-1).
3. **Sufficient size**: At least 100 queries for statistically significant results. 200-500 is better.
4. **Diverse query types**: Include short queries, long queries, specific queries, and broad queries.

### Construction Methods

#### Method 1: Manual Annotation

```python
import json


def create_golden_eval_set_manual(
    queries: list[str],
    corpus: list[str],
    annotations_path: str,
) -> dict:
    """Create a golden eval set with manual annotations.

    Process:
    1. For each query, retrieve top-50 documents with the base model.
    2. Have a human annotator label each as relevant (1) or not (0).
    3. Save annotations.

    This is the gold standard but expensive (~2 min per query).
    """
    from sentence_transformers import SentenceTransformer
    import numpy as np

    model = SentenceTransformer("BAAI/bge-base-en-v1.5")
    corpus_embs = model.encode(corpus, normalize_embeddings=True, batch_size=256)

    eval_set = {"queries": {}, "corpus": {}, "relevant_docs": {}}

    for qid, query in enumerate(queries):
        q_emb = model.encode([query], normalize_embeddings=True)[0]
        scores = np.dot(corpus_embs, q_emb)
        top_50 = np.argsort(scores)[::-1][:50]

        eval_set["queries"][f"q_{qid}"] = query

        # Present top-50 for human annotation
        candidates = []
        for rank, doc_idx in enumerate(top_50):
            doc_id = f"d_{doc_idx}"
            eval_set["corpus"][doc_id] = corpus[doc_idx]
            candidates.append({
                "doc_id": doc_id,
                "rank": rank + 1,
                "score": float(scores[doc_idx]),
                "text_preview": corpus[doc_idx][:200],
                "relevant": None,  # Human fills this in
            })

        eval_set["relevant_docs"][f"q_{qid}"] = candidates

    with open(annotations_path, "w") as f:
        json.dump(eval_set, f, indent=2)

    print(f"Created annotation template for {len(queries)} queries")
    print(f"Annotate {annotations_path} and set 'relevant' to true/false")
    return eval_set
```

#### Method 2: LLM-Assisted Annotation

```python
import anthropic
import json


def create_golden_eval_set_llm(
    queries: list[str],
    corpus: list[str],
    base_model_name: str = "BAAI/bge-base-en-v1.5",
    top_n_candidates: int = 30,
) -> dict:
    """Create a golden eval set using LLM for relevance annotation.

    Faster than manual annotation but less accurate.
    Recommended: use LLM for initial labeling, then human review of edge cases.
    """
    from sentence_transformers import SentenceTransformer
    import numpy as np

    model = SentenceTransformer(base_model_name)
    corpus_embs = model.encode(corpus, normalize_embeddings=True, batch_size=256)
    client = anthropic.Anthropic()

    eval_set = {"queries": {}, "corpus": {}, "relevant_docs": {}}

    for qid, query in enumerate(queries):
        q_emb = model.encode([query], normalize_embeddings=True)[0]
        scores = np.dot(corpus_embs, q_emb)
        top_candidates = np.argsort(scores)[::-1][:top_n_candidates]

        eval_set["queries"][f"q_{qid}"] = query

        # Use LLM to judge relevance
        candidates_text = ""
        for rank, doc_idx in enumerate(top_candidates):
            doc_id = f"d_{doc_idx}"
            eval_set["corpus"][doc_id] = corpus[doc_idx]
            candidates_text += f"\n[DOC-{rank}]: {corpus[doc_idx][:300]}\n"

        response = client.messages.create(
            model="claude-haiku-4-20250514",
            max_tokens=500,
            messages=[{
                "role": "user",
                "content": (
                    f"Query: {query}\n\n"
                    f"For each document below, output its ID if it is relevant "
                    f"to the query. Output one ID per line. Only include truly "
                    f"relevant documents.\n{candidates_text}"
                ),
            }],
        )

        # Parse relevant doc IDs
        relevant = set()
        for line in response.content[0].text.strip().split("\n"):
            for rank, doc_idx in enumerate(top_candidates):
                if f"DOC-{rank}" in line:
                    relevant.add(f"d_{doc_idx}")

        eval_set["relevant_docs"][f"q_{qid}"] = {doc_id: 1 for doc_id in relevant}

        if qid % 10 == 0:
            print(f"  Processed {qid}/{len(queries)} queries")

    return eval_set
```

---

## A/B Testing: Base vs Fine-Tuned Model

### Side-by-Side Evaluation

```python
from sentence_transformers import SentenceTransformer
import numpy as np
from dataclasses import dataclass


@dataclass
class ABTestResult:
    base_metrics: RetrievalMetrics
    finetuned_metrics: RetrievalMetrics
    improvement: dict
    statistical_significance: dict


def ab_test_models(
    base_model_name: str,
    finetuned_model_path: str,
    eval_queries: list[str],
    eval_corpus: list[str],
    relevant_doc_indices: list[set[int]],
) -> ABTestResult:
    """Compare base and fine-tuned models on the same evaluation set."""
    base_model = SentenceTransformer(base_model_name, trust_remote_code=True)
    ft_model = SentenceTransformer(finetuned_model_path)

    # Embed corpus with both models
    base_corpus_embs = base_model.encode(eval_corpus, normalize_embeddings=True, batch_size=256)
    ft_corpus_embs = ft_model.encode(eval_corpus, normalize_embeddings=True, batch_size=256)

    # Embed queries with both models
    base_query_embs = base_model.encode(eval_queries, normalize_embeddings=True, batch_size=64)
    ft_query_embs = ft_model.encode(eval_queries, normalize_embeddings=True, batch_size=64)

    # Evaluate both
    base_metrics = evaluate_retrieval(base_query_embs, base_corpus_embs, relevant_doc_indices)
    ft_metrics = evaluate_retrieval(ft_query_embs, ft_corpus_embs, relevant_doc_indices)

    # Compute improvements
    improvement = {
        "hit_at_1": ft_metrics.hit_at_1 - base_metrics.hit_at_1,
        "hit_at_5": ft_metrics.hit_at_5 - base_metrics.hit_at_5,
        "hit_at_10": ft_metrics.hit_at_10 - base_metrics.hit_at_10,
        "mrr_at_10": ft_metrics.mrr_at_10 - base_metrics.mrr_at_10,
        "ndcg_at_10": ft_metrics.ndcg_at_10 - base_metrics.ndcg_at_10,
    }

    # Statistical significance (paired bootstrap test)
    significance = bootstrap_significance_test(
        base_query_embs, ft_query_embs,
        base_corpus_embs, ft_corpus_embs,
        relevant_doc_indices,
    )

    # Print comparison
    print("\n" + "=" * 60)
    print(f"{'Metric':<15} {'Base':>10} {'Fine-tuned':>12} {'Delta':>10} {'p-value':>10}")
    print("-" * 60)
    for metric in ["hit_at_1", "hit_at_5", "hit_at_10", "mrr_at_10", "ndcg_at_10"]:
        base_val = getattr(base_metrics, metric)
        ft_val = getattr(ft_metrics, metric)
        delta = improvement[metric]
        p_val = significance.get(metric, 1.0)
        sig = " *" if p_val < 0.05 else ""
        print(f"{metric:<15} {base_val:>10.4f} {ft_val:>12.4f} {delta:>+10.4f} {p_val:>10.4f}{sig}")
    print("=" * 60)

    return ABTestResult(
        base_metrics=base_metrics,
        finetuned_metrics=ft_metrics,
        improvement=improvement,
        statistical_significance=significance,
    )


def bootstrap_significance_test(
    base_q_embs: np.ndarray,
    ft_q_embs: np.ndarray,
    base_d_embs: np.ndarray,
    ft_d_embs: np.ndarray,
    relevant: list[set[int]],
    n_bootstrap: int = 1000,
) -> dict:
    """Paired bootstrap test for statistical significance.

    Tests whether the fine-tuned model's improvement is statistically
    significant (p < 0.05) or could be due to random variation.
    """
    n_queries = len(relevant)
    np.random.seed(42)

    # Compute per-query NDCG@10 for both models
    base_scores_matrix = np.dot(base_q_embs, base_d_embs.T)
    ft_scores_matrix = np.dot(ft_q_embs, ft_d_embs.T)

    base_ndcgs = []
    ft_ndcgs = []

    for i in range(n_queries):
        for scores_matrix, ndcg_list in [(base_scores_matrix, base_ndcgs), (ft_scores_matrix, ft_ndcgs)]:
            ranked = np.argsort(scores_matrix[i])[::-1][:10]
            dcg = sum(
                1.0 / np.log2(r + 2) for r, idx in enumerate(ranked)
                if idx in relevant[i]
            )
            ideal = min(len(relevant[i]), 10)
            idcg = sum(1.0 / np.log2(r + 2) for r in range(ideal))
            ndcg_list.append(dcg / idcg if idcg > 0 else 0.0)

    base_ndcgs = np.array(base_ndcgs)
    ft_ndcgs = np.array(ft_ndcgs)
    observed_diff = ft_ndcgs.mean() - base_ndcgs.mean()

    # Bootstrap: resample queries and compute difference
    count_more_extreme = 0
    for _ in range(n_bootstrap):
        indices = np.random.choice(n_queries, size=n_queries, replace=True)
        boot_diff = ft_ndcgs[indices].mean() - base_ndcgs[indices].mean()
        if boot_diff <= 0:  # One-sided test: is fine-tuned truly better?
            count_more_extreme += 1

    p_value = count_more_extreme / n_bootstrap

    return {"ndcg_at_10": p_value}
```

---

## Automated Regression Testing

### CI/CD Integration

```python
import json
import sys
from pathlib import Path


def run_embedding_regression_test(
    model_path: str,
    eval_data_path: str,
    baseline_metrics_path: str,
    regression_threshold: float = 0.02,  # Max allowed degradation
) -> bool:
    """Run regression test: verify model meets quality baseline.

    Returns True if model passes, False if quality has regressed.
    Intended for CI/CD pipelines.
    """
    from sentence_transformers import SentenceTransformer
    import numpy as np

    # Load eval data
    with open(eval_data_path, "r") as f:
        eval_data = json.load(f)

    queries = list(eval_data["queries"].values())
    corpus = list(eval_data["corpus"].values())
    corpus_ids = list(eval_data["corpus"].keys())

    relevant = []
    for qid in eval_data["queries"]:
        rel_docs = eval_data["relevant_docs"].get(qid, {})
        rel_indices = set()
        for doc_id in rel_docs:
            if doc_id in corpus_ids:
                rel_indices.add(corpus_ids.index(doc_id))
        relevant.append(rel_indices)

    # Embed and evaluate
    model = SentenceTransformer(model_path, trust_remote_code=True)
    q_embs = model.encode(queries, normalize_embeddings=True, batch_size=64)
    d_embs = model.encode(corpus, normalize_embeddings=True, batch_size=256)

    metrics = evaluate_retrieval(q_embs, d_embs, relevant)

    # Compare against baseline
    with open(baseline_metrics_path, "r") as f:
        baseline = json.load(f)

    passed = True
    results = {}

    for metric_name in ["hit_at_10", "mrr_at_10", "ndcg_at_10"]:
        current = getattr(metrics, metric_name)
        base = baseline[metric_name]
        diff = current - base

        status = "PASS" if diff >= -regression_threshold else "FAIL"
        if status == "FAIL":
            passed = False

        results[metric_name] = {
            "current": current,
            "baseline": base,
            "diff": diff,
            "status": status,
        }
        print(f"  {metric_name}: {current:.4f} (baseline: {base:.4f}, diff: {diff:+.4f}) [{status}]")

    # Save results
    results_path = Path(model_path) / "regression_test_results.json"
    with open(results_path, "w") as f:
        json.dump(results, f, indent=2)

    return passed


if __name__ == "__main__":
    passed = run_embedding_regression_test(
        model_path=sys.argv[1],
        eval_data_path="eval_data.json",
        baseline_metrics_path="baseline_metrics.json",
    )
    sys.exit(0 if passed else 1)
```

### Saving Baseline Metrics

```python
def save_baseline_metrics(
    model_path: str,
    eval_data_path: str,
    output_path: str = "baseline_metrics.json",
):
    """Save current model metrics as the baseline for regression tests.

    Run this after validating a new fine-tuned model is good.
    """
    from sentence_transformers import SentenceTransformer
    import numpy as np

    with open(eval_data_path, "r") as f:
        eval_data = json.load(f)

    queries = list(eval_data["queries"].values())
    corpus = list(eval_data["corpus"].values())
    corpus_ids = list(eval_data["corpus"].keys())

    relevant = []
    for qid in eval_data["queries"]:
        rel_docs = eval_data["relevant_docs"].get(qid, {})
        rel_indices = {corpus_ids.index(doc_id) for doc_id in rel_docs if doc_id in corpus_ids}
        relevant.append(rel_indices)

    model = SentenceTransformer(model_path, trust_remote_code=True)
    q_embs = model.encode(queries, normalize_embeddings=True, batch_size=64)
    d_embs = model.encode(corpus, normalize_embeddings=True, batch_size=256)

    metrics = evaluate_retrieval(q_embs, d_embs, relevant)

    baseline = {
        "hit_at_1": metrics.hit_at_1,
        "hit_at_5": metrics.hit_at_5,
        "hit_at_10": metrics.hit_at_10,
        "mrr_at_10": metrics.mrr_at_10,
        "ndcg_at_10": metrics.ndcg_at_10,
        "map_at_10": metrics.map_at_10,
        "model_path": model_path,
        "eval_data": eval_data_path,
    }

    with open(output_path, "w") as f:
        json.dump(baseline, f, indent=2)

    print(f"Saved baseline metrics to {output_path}")
```

---

## Common Pitfalls

1. **Using the training set for evaluation.** Always use a held-out set. Training set metrics will be misleadingly high.
2. **Too few eval queries.** With 20 queries, a single lucky/unlucky result swings metrics by 5%. Use at least 100, ideally 200+.
3. **Incomplete relevance labels.** If you only label the top-1 result as relevant but there are 5 relevant documents, NDCG will undercount the model's quality.
4. **Not testing statistical significance.** A 2% improvement on 50 queries may be noise. Use bootstrap testing to verify significance.
5. **Evaluating on a different corpus than production.** If your eval corpus is 1,000 docs but production has 1M, the difficulty is completely different. Use a corpus of similar scale for evaluation.

---

## References

- NDCG Metric Explained -- https://en.wikipedia.org/wiki/Discounted_cumulative_gain
- BEIR: A Heterogeneous Benchmark for Zero-shot Evaluation of IR Models -- https://arxiv.org/abs/2104.08663
- sentence-transformers Evaluation -- https://www.sbert.net/docs/package_reference/evaluation.html
- Bootstrap Significance Testing for IR -- https://dl.acm.org/doi/10.1145/1390334.1390374
