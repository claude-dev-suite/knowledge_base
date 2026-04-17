# RAG Evaluation -- Complete Guide

## TL;DR

RAG evaluation measures the quality of your retrieval-augmented generation pipeline across two dimensions: retrieval quality (are the right documents being retrieved?) and generation quality (is the answer correct, faithful to sources, and relevant to the query?). The standard framework is RAGAS, which provides four core metrics: faithfulness, answer relevancy, context precision, and context recall. Effective evaluation requires a golden dataset (50-200 annotated query-answer pairs), consistent metrics, and a pipeline that runs evaluation automatically on every retrieval or generation change. This guide covers the full evaluation landscape.

---

## The Two Dimensions of RAG Quality

### Dimension 1: Retrieval Quality

Did the retriever find the right documents?

| Metric | What It Measures | Formula |
|--------|-----------------|---------|
| Context Precision | Are the relevant docs ranked high? | Relevant docs at top / total relevant |
| Context Recall | Were all relevant docs retrieved? | Retrieved relevant / total relevant |
| Hit Rate (MRR) | Is at least one relevant doc in top-K? | 1/rank_of_first_relevant |
| NDCG@K | Ranked quality of retrieval | DCG@K / ideal DCG@K |

### Dimension 2: Generation Quality

Is the answer good?

| Metric | What It Measures | Formula |
|--------|-----------------|---------|
| Faithfulness | Is the answer grounded in retrieved context? | Supported claims / total claims |
| Answer Relevancy | Does the answer address the query? | Similarity(answer, query) |
| Answer Correctness | Is the answer factually correct? | F1 against ground truth |
| Hallucination Rate | Does the answer contain unsupported claims? | 1 - faithfulness |

### The Evaluation Matrix

```
                     Good Retrieval    Bad Retrieval
                    +-----------------+-----------------+
Good Generation     | Best case:      | Lucky: LLM used |
                    | System works    | parametric      |
                    | as intended     | knowledge       |
                    +-----------------+-----------------+
Bad Generation      | Wasted context: | Worst case:     |
                    | Retriever OK    | Nothing works   |
                    | but LLM failed  |                 |
                    +-----------------+-----------------+
```

**Key insight**: you must evaluate both dimensions independently. A system can have good retrieval but bad generation (LLM ignoring context) or bad retrieval but good generation (LLM using parametric knowledge).

---

## RAGAS Framework

RAGAS (Retrieval Augmented Generation Assessment) is the de facto standard for RAG evaluation.

### Installation

```bash
pip install ragas
```

### Core Metrics

#### 1. Faithfulness

Measures whether the generated answer is grounded in the retrieved context. A faithfulness score of 1.0 means every claim in the answer can be traced back to the context.

**How it works**:
1. Decompose the answer into individual claims/statements
2. For each claim, check if it is supported by the context (NLI)
3. Faithfulness = supported claims / total claims

```python
from ragas.metrics import faithfulness
from ragas import evaluate
from datasets import Dataset

# Prepare evaluation data
eval_data = {
    "question": [
        "How does PostgreSQL RLS work?",
        "What is the default Kafka retention period?",
    ],
    "answer": [
        "PostgreSQL RLS uses CREATE POLICY to define row-level access rules that filter rows based on the current user.",
        "Kafka retains messages for 7 days by default, configured via retention.ms.",
    ],
    "contexts": [
        ["PostgreSQL Row Level Security uses CREATE POLICY statements to define per-row access control. Policies filter rows based on the current user's role."],
        ["Apache Kafka default retention period is 7 days (168 hours). This is controlled by the retention.ms broker configuration."],
    ],
}

dataset = Dataset.from_dict(eval_data)
result = evaluate(dataset, metrics=[faithfulness])
print(f"Faithfulness: {result['faithfulness']:.4f}")
```

#### 2. Answer Relevancy

Measures whether the answer actually addresses the query. Uses reverse question generation: generates questions from the answer and compares them to the original query.

```python
from ragas.metrics import answer_relevancy

result = evaluate(dataset, metrics=[answer_relevancy])
print(f"Answer Relevancy: {result['answer_relevancy']:.4f}")
```

#### 3. Context Precision

Measures whether the relevant documents are ranked higher than irrelevant ones. A precision of 1.0 means all relevant documents appear before any irrelevant ones.

```python
from ragas.metrics import context_precision

# Requires ground_truth for context precision
eval_data_with_gt = {
    "question": ["How does PostgreSQL RLS work?"],
    "answer": ["PostgreSQL RLS uses CREATE POLICY..."],
    "contexts": [
        [
            "PostgreSQL RLS uses CREATE POLICY statements...",  # Relevant (rank 1)
            "PostgreSQL supports multiple index types...",       # Irrelevant (rank 2)
            "Row Level Security filters rows by user...",        # Relevant (rank 3)
        ]
    ],
    "ground_truth": ["PostgreSQL Row Level Security uses CREATE POLICY to define row-level access control."],
}

dataset = Dataset.from_dict(eval_data_with_gt)
result = evaluate(dataset, metrics=[context_precision])
print(f"Context Precision: {result['context_precision']:.4f}")
```

#### 4. Context Recall

Measures whether all the information needed to answer the question was retrieved. Compares the ground truth answer against the retrieved contexts.

```python
from ragas.metrics import context_recall

result = evaluate(dataset, metrics=[context_recall])
print(f"Context Recall: {result['context_recall']:.4f}")
```

### Full RAGAS Evaluation

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)
from datasets import Dataset


def run_full_ragas_evaluation(
    questions: list[str],
    answers: list[str],
    contexts: list[list[str]],
    ground_truths: list[str],
) -> dict:
    """Run complete RAGAS evaluation."""
    eval_data = {
        "question": questions,
        "answer": answers,
        "contexts": contexts,
        "ground_truth": ground_truths,
    }

    dataset = Dataset.from_dict(eval_data)
    result = evaluate(
        dataset,
        metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
    )

    print("=== RAGAS Evaluation Results ===")
    for metric_name, score in result.items():
        if isinstance(score, float):
            print(f"  {metric_name}: {score:.4f}")

    return result


# Generate evaluation data from your RAG pipeline
def evaluate_rag_pipeline(pipeline, eval_queries: list[dict]) -> dict:
    """Run a RAG pipeline on eval queries and evaluate with RAGAS."""
    questions = []
    answers = []
    contexts = []
    ground_truths = []

    for eq in eval_queries:
        result = pipeline.invoke(eq["query"])

        questions.append(eq["query"])
        answers.append(result["answer"])
        contexts.append(result["source_documents"])
        ground_truths.append(eq["expected_answer"])

    return run_full_ragas_evaluation(questions, answers, contexts, ground_truths)
```

---

## Beyond RAGAS: Additional Evaluation Methods

### LLM-as-Judge

Use a strong LLM (Claude Opus, GPT-4) to evaluate answer quality:

```python
import anthropic

client = anthropic.Anthropic()


def llm_judge(
    query: str,
    answer: str,
    ground_truth: str,
    context: str,
) -> dict:
    """Use Claude as a judge for RAG answer quality."""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=500,
        messages=[{
            "role": "user",
            "content": (
                "Evaluate this RAG system's answer on four dimensions.\n"
                "Score each 1-5 (1=terrible, 5=perfect).\n\n"
                f"Query: {query}\n\n"
                f"Retrieved Context:\n{context[:2000]}\n\n"
                f"System Answer: {answer}\n\n"
                f"Ground Truth Answer: {ground_truth}\n\n"
                "Rate:\n"
                "1. Correctness (does the answer match ground truth?): X/5\n"
                "2. Faithfulness (is the answer grounded in context?): X/5\n"
                "3. Relevance (does it answer the question?): X/5\n"
                "4. Completeness (does it cover all aspects?): X/5\n\n"
                "Respond in format: correctness=X,faithfulness=X,relevance=X,completeness=X"
            ),
        }],
    )

    text = response.content[0].text
    scores = {}
    for part in text.split(","):
        if "=" in part:
            key, val = part.strip().split("=")
            scores[key.strip()] = int(val.strip())
    return scores
```

### Pairwise Comparison

Compare two pipeline configurations head-to-head:

```python
def pairwise_compare(
    query: str,
    answer_a: str,
    answer_b: str,
    ground_truth: str,
) -> str:
    """Compare two answers and pick the better one."""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=100,
        messages=[{
            "role": "user",
            "content": (
                f"Query: {query}\n\n"
                f"Answer A: {answer_a}\n\n"
                f"Answer B: {answer_b}\n\n"
                f"Ground Truth: {ground_truth}\n\n"
                "Which answer is better? Respond with ONLY 'A', 'B', or 'TIE'."
            ),
        }],
    )
    return response.content[0].text.strip()
```

### Human Evaluation

For final validation, human evaluation is the gold standard:

```python
# Human evaluation template
evaluation_template = {
    "query": "How to configure RLS in PostgreSQL?",
    "retrieved_context": "[context snippets]",
    "system_answer": "[generated answer]",
    "ratings": {
        "correctness": None,      # 1-5
        "completeness": None,     # 1-5
        "faithfulness": None,     # 1-5
        "helpfulness": None,      # 1-5
        "has_hallucination": None, # yes/no
    },
    "annotator_notes": "",
}
```

---

## Evaluation Pipeline Architecture

### Continuous Evaluation

```python
import json
import datetime
from pathlib import Path


class RAGEvaluationPipeline:
    """Automated RAG evaluation pipeline."""

    def __init__(
        self,
        eval_dataset_path: str,
        results_dir: str = "./eval_results",
    ):
        self.eval_dataset = self._load_dataset(eval_dataset_path)
        self.results_dir = Path(results_dir)
        self.results_dir.mkdir(exist_ok=True)

    def _load_dataset(self, path: str) -> list[dict]:
        with open(path) as f:
            return json.load(f)

    def run_evaluation(
        self,
        pipeline,
        pipeline_name: str,
        pipeline_config: dict,
    ) -> dict:
        """Run full evaluation and save results."""
        questions, answers, contexts, ground_truths = [], [], [], []

        for item in self.eval_dataset:
            result = pipeline.invoke(item["query"])
            questions.append(item["query"])
            answers.append(result["answer"])
            contexts.append(result.get("contexts", []))
            ground_truths.append(item["expected_answer"])

        # RAGAS evaluation
        ragas_results = run_full_ragas_evaluation(
            questions, answers, contexts, ground_truths
        )

        # Save results
        output = {
            "pipeline_name": pipeline_name,
            "pipeline_config": pipeline_config,
            "timestamp": datetime.datetime.now().isoformat(),
            "num_queries": len(self.eval_dataset),
            "metrics": {k: v for k, v in ragas_results.items() if isinstance(v, float)},
            "per_query_results": [
                {
                    "query": q,
                    "answer": a,
                    "ground_truth": gt,
                }
                for q, a, gt in zip(questions, answers, ground_truths)
            ],
        }

        filename = f"{pipeline_name}_{datetime.datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
        with open(self.results_dir / filename, "w") as f:
            json.dump(output, f, indent=2)

        return output

    def compare_runs(self, run_a_path: str, run_b_path: str) -> dict:
        """Compare two evaluation runs."""
        with open(run_a_path) as f:
            run_a = json.load(f)
        with open(run_b_path) as f:
            run_b = json.load(f)

        comparison = {}
        for metric in run_a["metrics"]:
            a_val = run_a["metrics"][metric]
            b_val = run_b["metrics"].get(metric, 0)
            diff = b_val - a_val
            comparison[metric] = {
                "run_a": a_val,
                "run_b": b_val,
                "diff": diff,
                "improved": diff > 0,
            }
            print(
                f"{metric}: {a_val:.4f} -> {b_val:.4f} "
                f"({'improved' if diff > 0 else 'regressed'} by {abs(diff):.4f})"
            )

        return comparison
```

---

## Interpreting Evaluation Results

### Score Ranges and What They Mean

| Metric | Poor | Acceptable | Good | Excellent |
|--------|------|-----------|------|-----------|
| Faithfulness | <0.60 | 0.60-0.75 | 0.75-0.90 | >0.90 |
| Answer Relevancy | <0.50 | 0.50-0.70 | 0.70-0.85 | >0.85 |
| Context Precision | <0.40 | 0.40-0.60 | 0.60-0.80 | >0.80 |
| Context Recall | <0.50 | 0.50-0.70 | 0.70-0.85 | >0.85 |

### Diagnostic Matrix

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| High recall, low precision | Retriever over-fetches | Reduce K, add reranking |
| Low recall, high precision | Retriever too selective | Increase K, use hybrid search |
| High faithfulness, low relevancy | Answer is grounded but off-topic | Improve prompt instructions |
| Low faithfulness, high relevancy | Answer is relevant but hallucinates | Add stronger grounding instructions, use Self-RAG |
| All metrics low | Fundamental pipeline issues | Check embeddings, chunking, and prompt |
| Metrics vary wildly across queries | Inconsistent query handling | Add adaptive routing |

---

## Testing Specific Pipeline Changes

### A/B Testing a Change

```python
def ab_test_change(
    eval_pipeline: RAGEvaluationPipeline,
    pipeline_a,    # Baseline
    pipeline_b,    # With change
    config_a: dict,
    config_b: dict,
) -> dict:
    """A/B test two pipeline configurations."""
    results_a = eval_pipeline.run_evaluation(pipeline_a, "baseline", config_a)
    results_b = eval_pipeline.run_evaluation(pipeline_b, "variant", config_b)

    print("\n=== A/B Test Results ===")
    for metric in results_a["metrics"]:
        a = results_a["metrics"][metric]
        b = results_b["metrics"][metric]
        diff = b - a
        pct = (diff / a * 100) if a > 0 else 0
        status = "BETTER" if diff > 0 else "WORSE" if diff < 0 else "SAME"
        print(f"  {metric}: {a:.4f} -> {b:.4f} ({diff:+.4f}, {pct:+.1f}%) [{status}]")

    return {"baseline": results_a, "variant": results_b}
```

### What to A/B Test

| Change | Metric to Watch | Expected Impact |
|--------|----------------|-----------------|
| Chunk size | Context precision + recall | Smaller chunks = higher precision, lower recall |
| Embedding model | All metrics | Better model = across-the-board improvement |
| Adding reranker | Context precision | Significant precision improvement |
| Hybrid search | Context recall | Better recall for exact-match queries |
| Prompt engineering | Faithfulness + relevancy | Direct impact on generation quality |
| Adding CRAG | Faithfulness | Fewer hallucinations on out-of-domain queries |

---

## Common Pitfalls

1. **Evaluating on too few queries.** 10-20 queries is not enough to draw conclusions. Minimum 50, ideally 100-200, covering the full distribution of query types.
2. **Not segmenting by query type.** Overall metrics hide important patterns. A system that scores 0.80 faithfulness overall may score 0.95 on factual queries but 0.50 on reasoning queries. Segment your eval set.
3. **Optimizing for one metric at the expense of others.** Maximizing faithfulness (e.g., by copying context verbatim) tanks answer relevancy and helpfulness. Monitor all metrics together.
4. **Using the eval set for tuning.** If you tune hyperparameters (chunk size, K, alpha) on the same eval set you report metrics on, you will overfit. Split into tuning and holdout sets.
5. **Not accounting for LLM variability.** LLM outputs are stochastic. Run evaluation 3x and average, or set temperature=0 for reproducibility.
6. **Trusting RAGAS blindly.** RAGAS uses an LLM internally for some metrics (faithfulness, relevancy). LLM-based evaluation can be wrong. Spot-check a random 10% of scores against human judgment.
7. **Evaluating once and declaring victory.** RAG quality degrades over time as the corpus changes, query distribution shifts, and model versions update. Run evaluation on a schedule (weekly or per-release).

---

## References

- RAGAS: https://docs.ragas.io/
- Es et al. "RAGAS: Automated Evaluation of Retrieval Augmented Generation" (2023). https://arxiv.org/abs/2309.15217
- LlamaIndex evaluation: https://docs.llamaindex.ai/en/stable/module_guides/evaluating/
- Zheng et al. "Judging LLM-as-a-Judge" (NeurIPS 2023)
- BEIR Benchmark: https://github.com/beir-cellar/beir
