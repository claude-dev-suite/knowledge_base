# Cross-Encoder Training -- Evaluation, A/B Testing, and Deployment

## TL;DR

After training a cross-encoder, you need to rigorously evaluate it before deploying. This guide covers offline evaluation metrics (NDCG@10, MRR@10, MAP), benchmark comparison against off-the-shelf models, A/B testing methodology for comparing your fine-tuned model against production baselines, and ONNX export for fast inference deployment. It provides complete Python code for running offline evaluation, designing and analyzing online A/B tests, and exporting your PyTorch cross-encoder to ONNX format for 2-4x inference speedup. The evaluation step is where most teams cut corners, leading to production models that are worse than the baselines they replaced.

---

## Offline Evaluation Metrics

### Ranking Metrics Explained

| Metric | What It Measures | Range | Use When |
|---|---|---|---|
| NDCG@K | Ranking quality with graded relevance | [0, 1] | You have graded labels (0-3) |
| MRR@K | Rank of the first relevant result | [0, 1] | You only care about the top-1 result |
| MAP | Mean average precision | [0, 1] | You have binary labels |
| Recall@K | Fraction of relevant docs in top-K | [0, 1] | You need coverage guarantees |
| Precision@K | Fraction of top-K that are relevant | [0, 1] | You want high-precision results |

### Computing Ranking Metrics

```python
"""
Compute ranking metrics for cross-encoder evaluation.
"""
import json
import math

import numpy as np


def ndcg_at_k(
    relevance_scores: list[float],
    k: int = 10,
) -> float:
    """
    Compute Normalized Discounted Cumulative Gain at K.

    NDCG@K is the primary metric for evaluating rerankers
    because it accounts for both relevance grade and position.

    Args:
        relevance_scores: list of relevance scores in ranked order
                         (as returned by the reranker)
        k: cutoff position

    Returns:
        NDCG@K score in [0, 1]
    """
    scores = relevance_scores[:k]

    # DCG
    dcg = sum(
        (2 ** rel - 1) / math.log2(pos + 2)
        for pos, rel in enumerate(scores)
    )

    # Ideal DCG (sort by relevance, best first)
    ideal_scores = sorted(relevance_scores, reverse=True)[:k]
    idcg = sum(
        (2 ** rel - 1) / math.log2(pos + 2)
        for pos, rel in enumerate(ideal_scores)
    )

    if idcg == 0:
        return 0.0

    return dcg / idcg


def mrr_at_k(
    relevance_scores: list[float],
    k: int = 10,
    threshold: float = 1.0,
) -> float:
    """
    Compute Mean Reciprocal Rank at K.

    MRR measures how quickly the first relevant result appears.

    Args:
        relevance_scores: list of relevance scores in ranked order
        k: cutoff position
        threshold: minimum score to be considered relevant

    Returns:
        Reciprocal rank of first relevant result, or 0 if none in top-K
    """
    for pos, score in enumerate(relevance_scores[:k]):
        if score >= threshold:
            return 1.0 / (pos + 1)
    return 0.0


def mean_average_precision(
    relevance_scores: list[float],
    threshold: float = 1.0,
) -> float:
    """
    Compute Average Precision.

    MAP is the mean of AP scores across all queries.
    """
    relevant = 0
    sum_precision = 0.0

    for pos, score in enumerate(relevance_scores):
        if score >= threshold:
            relevant += 1
            precision_at_pos = relevant / (pos + 1)
            sum_precision += precision_at_pos

    if relevant == 0:
        return 0.0

    return sum_precision / relevant


def recall_at_k(
    relevance_scores: list[float],
    total_relevant: int,
    k: int = 10,
    threshold: float = 1.0,
) -> float:
    """
    Compute Recall@K.

    What fraction of all relevant documents appear in top-K?
    """
    if total_relevant == 0:
        return 0.0

    retrieved_relevant = sum(
        1 for score in relevance_scores[:k] if score >= threshold
    )

    return retrieved_relevant / total_relevant
```

### Full Evaluation Pipeline

```python
"""
Complete cross-encoder evaluation pipeline.
"""
from sentence_transformers.cross_encoder import CrossEncoder


class CrossEncoderEvaluator:
    """
    Evaluate a cross-encoder reranker on a held-out test set.
    """

    def __init__(
        self,
        model: CrossEncoder,
        test_data: list[dict],
    ):
        """
        Args:
            model: trained cross-encoder
            test_data: list of dicts with keys:
                - query: str
                - documents: list[str] (candidate documents)
                - relevance: list[float] (ground truth relevance per doc)
        """
        self.model = model
        self.test_data = test_data

    def evaluate(
        self,
        ks: list[int] = [1, 3, 5, 10],
        batch_size: int = 64,
    ) -> dict:
        """
        Run evaluation across all metrics and K values.
        """
        all_ndcg = {k: [] for k in ks}
        all_mrr = {k: [] for k in ks}
        all_recall = {k: [] for k in ks}
        all_map = []

        for item in self.test_data:
            query = item["query"]
            documents = item["documents"]
            true_relevance = item["relevance"]

            # Score with cross-encoder
            pairs = [(query, doc) for doc in documents]
            scores = self.model.predict(pairs, batch_size=batch_size)

            # Sort documents by predicted score (highest first)
            sorted_indices = np.argsort(scores)[::-1]
            reranked_relevance = [true_relevance[i] for i in sorted_indices]

            # Compute metrics
            total_relevant = sum(1 for r in true_relevance if r >= 1.0)

            for k in ks:
                all_ndcg[k].append(ndcg_at_k(reranked_relevance, k))
                all_mrr[k].append(mrr_at_k(reranked_relevance, k))
                all_recall[k].append(
                    recall_at_k(reranked_relevance, total_relevant, k)
                )

            all_map.append(mean_average_precision(reranked_relevance))

        # Aggregate
        results = {"MAP": float(np.mean(all_map))}

        for k in ks:
            results[f"NDCG@{k}"] = float(np.mean(all_ndcg[k]))
            results[f"MRR@{k}"] = float(np.mean(all_mrr[k]))
            results[f"Recall@{k}"] = float(np.mean(all_recall[k]))

        # Print report
        print("=" * 60)
        print("Cross-Encoder Evaluation Results")
        print("=" * 60)
        print(f"\n  MAP: {results['MAP']:.4f}\n")

        for k in ks:
            print(f"  @{k}: NDCG={results[f'NDCG@{k}']:.4f}  "
                  f"MRR={results[f'MRR@{k}']:.4f}  "
                  f"Recall={results[f'Recall@{k}']:.4f}")

        return results

    def compare_with_baseline(
        self,
        baseline_model: CrossEncoder,
        ks: list[int] = [1, 5, 10],
    ) -> dict:
        """
        Compare fine-tuned model against a baseline (off-the-shelf).
        """
        print("\n=== Baseline Model ===")
        baseline_evaluator = CrossEncoderEvaluator(
            baseline_model, self.test_data,
        )
        baseline_results = baseline_evaluator.evaluate(ks)

        print("\n=== Fine-Tuned Model ===")
        finetuned_results = self.evaluate(ks)

        # Comparison
        print("\n=== Comparison ===")
        for metric in baseline_results:
            base = baseline_results[metric]
            fine = finetuned_results[metric]
            diff = fine - base
            pct = (diff / max(base, 0.001)) * 100
            arrow = "+" if diff > 0 else ""
            print(f"  {metric}: {base:.4f} -> {fine:.4f} ({arrow}{diff:.4f}, {arrow}{pct:.1f}%)")

        return {
            "baseline": baseline_results,
            "finetuned": finetuned_results,
            "improvement": {
                metric: finetuned_results[metric] - baseline_results[metric]
                for metric in baseline_results
            },
        }

    def per_query_analysis(
        self,
        batch_size: int = 64,
    ) -> list[dict]:
        """
        Analyze per-query performance to identify failure modes.
        """
        per_query = []

        for item in self.test_data:
            query = item["query"]
            documents = item["documents"]
            true_relevance = item["relevance"]

            pairs = [(query, doc) for doc in documents]
            scores = self.model.predict(pairs, batch_size=batch_size)

            sorted_indices = np.argsort(scores)[::-1]
            reranked_relevance = [true_relevance[i] for i in sorted_indices]

            ndcg = ndcg_at_k(reranked_relevance, 10)
            mrr = mrr_at_k(reranked_relevance, 10)

            per_query.append({
                "query": query,
                "ndcg_10": ndcg,
                "mrr_10": mrr,
                "num_relevant": sum(1 for r in true_relevance if r >= 1),
                "num_candidates": len(documents),
                "top_score": float(max(scores)),
                "score_spread": float(max(scores) - min(scores)),
            })

        # Sort by NDCG to find worst queries
        per_query.sort(key=lambda x: x["ndcg_10"])

        print("\n=== Worst 5 Queries ===")
        for pq in per_query[:5]:
            print(f"  NDCG={pq['ndcg_10']:.4f} MRR={pq['mrr_10']:.4f} "
                  f"Q: {pq['query'][:60]}...")

        print("\n=== Best 5 Queries ===")
        for pq in per_query[-5:]:
            print(f"  NDCG={pq['ndcg_10']:.4f} MRR={pq['mrr_10']:.4f} "
                  f"Q: {pq['query'][:60]}...")

        return per_query
```

---

## A/B Testing Against Production

### Designing the A/B Test

```python
"""
A/B testing framework for cross-encoder deployment.
"""
import hashlib
import json
import random
import time
from dataclasses import dataclass, field
from datetime import datetime


@dataclass
class ABTestConfig:
    """Configuration for a cross-encoder A/B test."""
    test_name: str
    control_model: str  # Path to current production model
    treatment_model: str  # Path to fine-tuned model
    traffic_split: float = 0.5  # Fraction going to treatment
    min_queries: int = 1000  # Minimum queries before drawing conclusions
    max_duration_hours: int = 168  # Maximum test duration (1 week)
    significance_level: float = 0.05


class CrossEncoderABTest:
    """
    Run an A/B test comparing two cross-encoder models in production.

    Uses query-level hashing for consistent assignment:
    the same query always goes to the same variant.
    """

    def __init__(self, config: ABTestConfig):
        self.config = config
        self.control = CrossEncoder(config.control_model)
        self.treatment = CrossEncoder(config.treatment_model)
        self.results = {"control": [], "treatment": []}
        self.start_time = datetime.now()

    def assign_variant(self, query: str, user_id: str = "") -> str:
        """
        Assign a query to control or treatment.

        Uses deterministic hashing so the same query+user always
        gets the same variant (important for consistency).
        """
        hash_input = f"{self.config.test_name}:{query}:{user_id}"
        hash_value = int(hashlib.md5(hash_input.encode()).hexdigest(), 16)
        if (hash_value % 100) < (self.config.traffic_split * 100):
            return "treatment"
        return "control"

    def rerank(
        self,
        query: str,
        documents: list[str],
        user_id: str = "",
    ) -> dict:
        """
        Rerank documents using the assigned variant's model.

        Returns reranked documents + variant assignment for logging.
        """
        variant = self.assign_variant(query, user_id)
        model = self.treatment if variant == "treatment" else self.control

        start = time.time()
        pairs = [(query, doc) for doc in documents]
        scores = model.predict(pairs)
        latency = time.time() - start

        sorted_indices = np.argsort(scores)[::-1]
        reranked = [documents[i] for i in sorted_indices]
        reranked_scores = [float(scores[i]) for i in sorted_indices]

        return {
            "variant": variant,
            "reranked_documents": reranked,
            "scores": reranked_scores,
            "latency_ms": latency * 1000,
        }

    def record_outcome(
        self,
        variant: str,
        query: str,
        clicked_position: int | None,
        dwell_time_seconds: float | None = None,
        user_rating: int | None = None,
    ) -> None:
        """Record user interaction for analysis."""
        self.results[variant].append({
            "query": query,
            "clicked_position": clicked_position,
            "dwell_time": dwell_time_seconds,
            "rating": user_rating,
            "timestamp": datetime.now().isoformat(),
        })

    def analyze(self) -> dict:
        """
        Analyze A/B test results.

        Returns statistical comparison between variants.
        """
        from scipy import stats

        analysis = {}

        for metric_name, extract_fn in [
            ("click_through_rate", lambda r: 1 if r["clicked_position"] is not None else 0),
            ("mrr", lambda r: 1.0 / (r["clicked_position"] + 1) if r["clicked_position"] is not None else 0),
            ("avg_click_position", lambda r: r["clicked_position"] if r["clicked_position"] is not None else float("nan")),
        ]:
            control_values = [extract_fn(r) for r in self.results["control"]]
            treatment_values = [extract_fn(r) for r in self.results["treatment"]]

            # Filter NaN for position metric
            control_clean = [v for v in control_values if not (isinstance(v, float) and np.isnan(v))]
            treatment_clean = [v for v in treatment_values if not (isinstance(v, float) and np.isnan(v))]

            if not control_clean or not treatment_clean:
                continue

            control_mean = np.mean(control_clean)
            treatment_mean = np.mean(treatment_clean)
            diff = treatment_mean - control_mean

            # Statistical test
            t_stat, p_value = stats.ttest_ind(treatment_clean, control_clean)
            significant = p_value < self.config.significance_level

            analysis[metric_name] = {
                "control_mean": float(control_mean),
                "treatment_mean": float(treatment_mean),
                "difference": float(diff),
                "relative_change": float(diff / max(control_mean, 0.001)),
                "p_value": float(p_value),
                "significant": significant,
                "control_n": len(control_clean),
                "treatment_n": len(treatment_clean),
            }

            status = "SIGNIFICANT" if significant else "not significant"
            direction = "BETTER" if diff > 0 else "WORSE"
            print(
                f"{metric_name}: control={control_mean:.4f}, "
                f"treatment={treatment_mean:.4f}, "
                f"diff={diff:+.4f} ({direction}), "
                f"p={p_value:.4f} ({status})"
            )

        # Decision recommendation
        total_queries = len(self.results["control"]) + len(self.results["treatment"])
        enough_data = total_queries >= self.config.min_queries

        if enough_data:
            mrr_analysis = analysis.get("mrr", {})
            if mrr_analysis.get("significant") and mrr_analysis.get("difference", 0) > 0:
                analysis["recommendation"] = "DEPLOY treatment (significant improvement)"
            elif mrr_analysis.get("significant") and mrr_analysis.get("difference", 0) < 0:
                analysis["recommendation"] = "KEEP control (treatment is worse)"
            else:
                analysis["recommendation"] = "INCONCLUSIVE (no significant difference)"
        else:
            analysis["recommendation"] = f"CONTINUE testing (need {self.config.min_queries - total_queries} more queries)"

        print(f"\nRecommendation: {analysis['recommendation']}")
        return analysis
```

---

## ONNX Export for Fast Deployment

### Exporting to ONNX

```python
"""
Export cross-encoder to ONNX for fast inference.
"""
import time

import numpy as np
import torch
from transformers import AutoModelForSequenceClassification, AutoTokenizer


def export_to_onnx(
    model_path: str,
    output_path: str = "./cross-encoder.onnx",
    max_length: int = 512,
    opset_version: int = 14,
) -> str:
    """
    Export a cross-encoder to ONNX format.

    ONNX provides 2-4x inference speedup over PyTorch,
    especially on CPU.
    """
    tokenizer = AutoTokenizer.from_pretrained(model_path)
    model = AutoModelForSequenceClassification.from_pretrained(model_path)
    model.eval()

    # Create dummy input
    dummy_input = tokenizer(
        "sample query",
        "sample document text for export",
        max_length=max_length,
        padding="max_length",
        truncation=True,
        return_tensors="pt",
    )

    # Export
    torch.onnx.export(
        model,
        (
            dummy_input["input_ids"],
            dummy_input["attention_mask"],
        ),
        output_path,
        input_names=["input_ids", "attention_mask"],
        output_names=["logits"],
        dynamic_axes={
            "input_ids": {0: "batch_size", 1: "sequence_length"},
            "attention_mask": {0: "batch_size", 1: "sequence_length"},
            "logits": {0: "batch_size"},
        },
        opset_version=opset_version,
        do_constant_folding=True,
    )

    # Verify
    import onnx
    onnx_model = onnx.load(output_path)
    onnx.checker.check_model(onnx_model)

    print(f"ONNX model exported to {output_path}")
    print(f"Model size: {Path(output_path).stat().st_size / 1e6:.1f} MB")

    return output_path


class ONNXCrossEncoder:
    """
    Fast ONNX cross-encoder for production inference.
    """

    def __init__(
        self,
        onnx_path: str,
        tokenizer_path: str,
        max_length: int = 512,
    ):
        import onnxruntime as ort

        self.tokenizer = AutoTokenizer.from_pretrained(tokenizer_path)
        self.max_length = max_length

        # Use all available CPU cores
        sess_options = ort.SessionOptions()
        sess_options.inter_op_num_threads = 4
        sess_options.intra_op_num_threads = 4

        self.session = ort.InferenceSession(
            onnx_path,
            sess_options=sess_options,
            providers=["CPUExecutionProvider"],
        )

    def predict(
        self,
        pairs: list[tuple[str, str]],
        batch_size: int = 32,
    ) -> np.ndarray:
        """
        Score query-document pairs using ONNX runtime.
        """
        all_scores = []

        for i in range(0, len(pairs), batch_size):
            batch = pairs[i : i + batch_size]
            queries = [p[0] for p in batch]
            docs = [p[1] for p in batch]

            encoding = self.tokenizer(
                queries, docs,
                max_length=self.max_length,
                padding=True,
                truncation=True,
                return_tensors="np",
            )

            outputs = self.session.run(
                ["logits"],
                {
                    "input_ids": encoding["input_ids"].astype(np.int64),
                    "attention_mask": encoding["attention_mask"].astype(np.int64),
                },
            )

            logits = outputs[0]
            if logits.shape[-1] == 1:
                scores = logits.squeeze(-1)
            else:
                # If classification output, take softmax probability of positive class
                from scipy.special import softmax
                scores = softmax(logits, axis=-1)[:, 1]

            all_scores.extend(scores.tolist())

        return np.array(all_scores)


def benchmark_inference(
    pytorch_model_path: str,
    onnx_model_path: str,
    num_pairs: int = 100,
    max_length: int = 512,
) -> dict:
    """
    Benchmark ONNX vs PyTorch inference speed.
    """
    # Generate test pairs
    pairs = [
        (f"test query number {i}", f"test document content for benchmarking pair {i}")
        for i in range(num_pairs)
    ]

    # PyTorch benchmark
    pt_model = CrossEncoder(pytorch_model_path, max_length=max_length)
    start = time.time()
    pt_scores = pt_model.predict(pairs)
    pt_time = time.time() - start

    # ONNX benchmark
    onnx_model = ONNXCrossEncoder(onnx_model_path, pytorch_model_path, max_length)
    start = time.time()
    onnx_scores = onnx_model.predict(pairs)
    onnx_time = time.time() - start

    # Score agreement
    correlation = np.corrcoef(pt_scores, onnx_scores)[0, 1]
    max_diff = float(np.max(np.abs(pt_scores - onnx_scores)))

    results = {
        "pytorch_time_ms": pt_time * 1000,
        "onnx_time_ms": onnx_time * 1000,
        "speedup": pt_time / max(onnx_time, 0.001),
        "per_pair_pytorch_ms": (pt_time / num_pairs) * 1000,
        "per_pair_onnx_ms": (onnx_time / num_pairs) * 1000,
        "score_correlation": float(correlation),
        "max_score_difference": max_diff,
        "num_pairs": num_pairs,
    }

    print("Inference Benchmark")
    print(f"  PyTorch: {results['pytorch_time_ms']:.1f}ms total, "
          f"{results['per_pair_pytorch_ms']:.2f}ms per pair")
    print(f"  ONNX:    {results['onnx_time_ms']:.1f}ms total, "
          f"{results['per_pair_onnx_ms']:.2f}ms per pair")
    print(f"  Speedup: {results['speedup']:.1f}x")
    print(f"  Score correlation: {results['score_correlation']:.6f}")
    print(f"  Max score diff:   {results['max_score_difference']:.6f}")

    return results


from pathlib import Path
from sentence_transformers.cross_encoder import CrossEncoder
```

---

## Deployment Checklist

| Step | Check | Why |
|---|---|---|
| Offline evaluation | NDCG@10 > baseline | Verify quality improvement |
| Per-query analysis | No category has NDCG < 0.3 | Catch category-specific regressions |
| Latency benchmark | < 100ms for 20 docs (CPU) | Ensure acceptable user experience |
| ONNX export | Score correlation > 0.999 | Verify export fidelity |
| A/B test setup | Deterministic assignment | Prevent cross-contamination |
| A/B test duration | 1000+ queries per variant | Sufficient statistical power |
| A/B test metrics | CTR, MRR, user ratings | Measure what matters |
| Rollback plan | Can revert in < 5 minutes | Risk mitigation |

---

## Common Pitfalls

1. **Evaluating on the training set**: always evaluate on a held-out test set that was never seen during training. Train/validation/test split should be strict.

2. **Only reporting aggregate metrics**: NDCG@10 of 0.85 sounds great, but if 20% of queries score below 0.50, you have a tail quality problem. Always analyze per-query results.

3. **Comparing against wrong baseline**: compare your fine-tuned model against the exact model currently in production, not against a random baseline. If production uses `cross-encoder/ms-marco-MiniLM-L-12-v2`, benchmark against that specific checkpoint.

4. **Deploying without A/B testing**: offline metrics do not always predict online performance. Query distributions in production differ from test sets. Always A/B test before full deployment.

5. **ONNX export with wrong opset**: some operations are not supported in older ONNX opsets. Use opset 14+ for DeBERTa models. Verify the exported model produces the same scores as PyTorch.

6. **Not measuring latency in production conditions**: benchmark on the same hardware (CPU/GPU) and with the same batch sizes you will use in production. A model that is fast on an A100 may be too slow on a CPU-only deployment.

7. **Ignoring score distribution shifts**: a fine-tuned model may produce different score ranges than the baseline. If downstream logic uses hardcoded thresholds (e.g., "discard documents with score < 0.5"), verify that the new model's score distribution is compatible.

---

## References

- Nogueira, R. and Cho, K. "Passage Re-ranking with BERT." arXiv 2019.
- NDCG: https://en.wikipedia.org/wiki/Discounted_cumulative_gain
- ONNX Runtime: https://onnxruntime.ai/docs/
- sentence-transformers evaluation: https://www.sbert.net/docs/cross_encoder/training/evaluation.html
- MS MARCO leaderboard: https://microsoft.github.io/msmarco/
- Hofstatter, S. et al. "Efficiently Teaching an Effective Dense Retriever." SIGIR 2021.
