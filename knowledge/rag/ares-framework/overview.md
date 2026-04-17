# ARES Framework -- Automated RAG Evaluation with Synthetic Data and PPI

## TL;DR

ARES (Automated RAG Evaluation System) is a Stanford research framework that evaluates RAG pipelines without requiring large human-annotated datasets. It works in three stages: (1) generate synthetic training data from your corpus using an LLM, (2) fine-tune lightweight DeBERTa classifiers on that synthetic data to predict context relevance, answer faithfulness, and answer relevance, and (3) use Prediction-Powered Inference (PPI) to combine classifier predictions with a small set of human-annotated examples (typically 50-150) to produce statistically valid confidence intervals for your pipeline's quality metrics. This eliminates the need for thousands of human judgments while maintaining statistical rigor that pure LLM-as-judge approaches lack.

---

## The Problem ARES Solves

### Why Existing Approaches Fall Short

RAG evaluation faces a fundamental tension between cost and reliability:

| Approach | Cost | Statistical Rigor | Domain Adaptability |
|---|---|---|---|
| Full human evaluation | Very high ($5K-50K) | Gold standard | Perfect |
| LLM-as-judge (GPT-4, Claude) | Moderate ($50-500) | No confidence intervals | Good but biased |
| RAGAS (reference-free) | Low ($10-50) | No confidence intervals | Variable |
| ARES | Low ($5-30) | Confidence intervals via PPI | Strong (domain-adapted classifiers) |

LLM-as-judge methods produce a single point estimate (e.g., "faithfulness = 0.82") with no indication of how uncertain that estimate is. ARES produces confidence intervals (e.g., "faithfulness = 0.82 +/- 0.04, 95% CI") that account for both classifier error and sampling variance.

### When ARES Wins

ARES is the strongest choice when:

- You need to compare two pipeline configurations with statistical significance
- Your domain is specialized (legal, medical, financial) and general-purpose judges are unreliable
- You want to run evaluations repeatedly (daily CI, A/B tests) at low marginal cost
- You need defensible quality metrics with confidence intervals for stakeholders

### When ARES Is Overkill

Skip ARES when:

- You are prototyping and need a quick quality check (use RAGAS instead)
- You have fewer than 50 labeled examples and cannot annotate more
- Your RAG pipeline changes infrequently and you can afford occasional LLM-as-judge runs
- You need per-query diagnostics rather than aggregate pipeline metrics

---

## Architecture Deep Dive

### The Three-Stage Pipeline

```
Stage 1: Synthetic Data Generation
  Corpus documents --> LLM generates (query, answer, context) triples
  --> Positive examples (faithful answers) + negative examples (unfaithful/irrelevant)
  --> 5K-20K synthetic training examples

Stage 2: Classifier Fine-Tuning
  Synthetic data --> Fine-tune 3 DeBERTa-v3-base classifiers:
    - Context Relevance classifier: P(context is relevant | query, context)
    - Answer Faithfulness classifier: P(answer is faithful | context, answer)
    - Answer Relevance classifier: P(answer is relevant | query, answer)
  --> 3 trained classifiers (~20 min training each on single GPU)

Stage 3: Prediction-Powered Inference (PPI)
  Trained classifiers score your RAG pipeline's outputs (unlabeled, N=500-2000)
  + Small human-annotated set (labeled, n=50-150)
  --> PPI combines both to produce confidence intervals
  --> Output: "Faithfulness = 0.84 [0.79, 0.89] (95% CI)"
```

### Why Three Classifiers Instead of One LLM?

ARES separates evaluation into three independent classifiers because:

1. **Domain adaptation**: each classifier fine-tunes on your corpus, learning domain-specific patterns that a general LLM misses
2. **Speed**: DeBERTa-v3-base processes a query-context pair in ~5ms vs. ~2s for an LLM judge. Evaluating 2000 examples takes 10 seconds instead of 60+ minutes
3. **Cost**: after the one-time training cost, inference is essentially free (local model). LLM judge costs scale linearly with evaluation size
4. **Consistency**: classifiers produce deterministic scores at temperature 0. LLM judges have inherent variance between runs

---

## Prediction-Powered Inference (PPI)

PPI is the statistical core of ARES and what distinguishes it from simpler classifier-based approaches.

### The Key Insight

Suppose you have:
- A classifier that can score 2000 unlabeled examples (cheap but noisy)
- Human annotations for 100 of those same examples (expensive but accurate)

PPI combines both to produce an estimate that is:
- More accurate than using only the 100 human labels (because it leverages the 2000 classifier predictions)
- More trustworthy than using only the classifier (because it corrects for classifier bias using the human labels)

### Mathematical Foundation

```
PPI estimator:

  theta_PPI = theta_classifier + correction

Where:
  theta_classifier = (1/N) * SUM_{i=1}^{N} f(x_i)
    --> classifier's mean prediction on all N unlabeled examples

  correction = (1/n) * SUM_{j=1}^{n} [y_j - f(x_j)]
    --> mean difference between human labels y_j and classifier predictions f(x_j)
    --> on the n human-annotated examples

The confidence interval width depends on:
  - Variance of the classifier predictions
  - Variance of the correction term (human vs classifier disagreement)
  - Size of the labeled set n (larger n = tighter CI)
  - Size of the unlabeled set N (larger N = tighter CI, but diminishing returns)
```

### Practical Implications

```python
"""
Demonstration of PPI confidence interval behavior.
"""
import numpy as np
from scipy import stats


def ppi_confidence_interval(
    classifier_predictions: np.ndarray,
    human_labels: np.ndarray,
    classifier_on_labeled: np.ndarray,
    alpha: float = 0.05,
) -> dict:
    """
    Compute PPI confidence interval for a binary classification metric.

    Args:
        classifier_predictions: classifier scores on ALL unlabeled examples (N,)
        human_labels: ground truth labels for the labeled subset (n,)
        classifier_on_labeled: classifier scores on the labeled subset (n,)
        alpha: significance level (0.05 for 95% CI)

    Returns:
        dict with point estimate and confidence interval bounds
    """
    N = len(classifier_predictions)
    n = len(human_labels)

    # Classifier's estimate on the full unlabeled set
    theta_f = np.mean(classifier_predictions)

    # Correction: how much the classifier is off on the labeled set
    correction = np.mean(human_labels - classifier_on_labeled)

    # PPI point estimate
    theta_ppi = theta_f + correction

    # Variance of the correction term
    residuals = human_labels - classifier_on_labeled
    var_correction = np.var(residuals, ddof=1) / n

    # Variance of the classifier estimate
    var_classifier = np.var(classifier_predictions, ddof=1) / N

    # Total variance (sum because the two terms are independent)
    var_total = var_correction + var_classifier

    # Confidence interval
    z = stats.norm.ppf(1 - alpha / 2)
    ci_lower = theta_ppi - z * np.sqrt(var_total)
    ci_upper = theta_ppi + z * np.sqrt(var_total)

    return {
        "point_estimate": float(theta_ppi),
        "ci_lower": float(ci_lower),
        "ci_upper": float(ci_upper),
        "ci_width": float(ci_upper - ci_lower),
        "classifier_estimate": float(theta_f),
        "correction": float(correction),
        "n_labeled": n,
        "n_unlabeled": N,
    }


# Example: evaluating faithfulness
np.random.seed(42)

# Simulate: classifier is slightly overconfident (predicts 0.85, truth is 0.80)
N_unlabeled = 1000
n_labeled = 100

true_faithfulness = 0.80
classifier_bias = 0.05

# Classifier predictions on all examples
all_predictions = np.random.binomial(
    1, true_faithfulness + classifier_bias, N_unlabeled
).astype(float)

# Human labels on the labeled subset
labeled_indices = np.random.choice(N_unlabeled, n_labeled, replace=False)
human_labels = np.random.binomial(1, true_faithfulness, n_labeled).astype(float)
classifier_on_labeled = all_predictions[labeled_indices]

result = ppi_confidence_interval(
    all_predictions, human_labels, classifier_on_labeled
)

print(f"Classifier estimate (biased): {result['classifier_estimate']:.3f}")
print(f"PPI estimate (corrected):     {result['point_estimate']:.3f}")
print(
    f"95% CI: [{result['ci_lower']:.3f}, {result['ci_upper']:.3f}] "
    f"(width: {result['ci_width']:.3f})"
)
print(f"True value: {true_faithfulness}")
```

### How Many Human Annotations Do You Need?

| Labeled Examples (n) | Unlabeled Examples (N) | Typical CI Width | Cost (at $0.30/label) |
|---|---|---|---|
| 25 | 500 | +/- 0.08-0.12 | $7.50 |
| 50 | 1000 | +/- 0.05-0.08 | $15 |
| 100 | 1000 | +/- 0.04-0.06 | $30 |
| 150 | 2000 | +/- 0.03-0.05 | $45 |
| 300 | 2000 | +/- 0.02-0.04 | $90 |

The sweet spot is 50-150 human annotations with 1000-2000 unlabeled examples. Beyond 150 labeled examples, the marginal improvement in CI width is small.

---

## Synthetic Data Generation

### Why Synthetic Data Works

ARES fine-tunes classifiers on synthetic data rather than requiring thousands of human annotations. This works because:

1. The classifiers only need to learn the pattern "what does relevant/faithful/relevant look like in this domain" -- not produce perfect judgments
2. PPI corrects for any systematic bias the classifier has via the small human-annotated set
3. Synthetic data from strong LLMs captures most of the domain-specific patterns

### Generation Strategy

```python
"""
Synthetic training data generation for ARES classifiers.
"""
import json
import random
from dataclasses import dataclass, field

import anthropic


@dataclass
class SyntheticExample:
    query: str
    context: str
    answer: str
    context_relevance: int  # 0 or 1
    answer_faithfulness: int  # 0 or 1
    answer_relevance: int  # 0 or 1


class ARESSyntheticGenerator:
    """
    Generate synthetic training data for ARES classifiers.

    Produces positive and negative examples for three dimensions:
    context relevance, answer faithfulness, and answer relevance.
    """

    def __init__(self, model: str = "claude-sonnet-4-20250514"):
        self.client = anthropic.Anthropic()
        self.model = model

    def generate_query_from_passage(self, passage: str) -> str:
        """Generate a natural question that the passage answers."""
        response = self.client.messages.create(
            model=self.model,
            max_tokens=200,
            messages=[{
                "role": "user",
                "content": (
                    "Given this passage, generate a single natural question "
                    "that someone might ask which this passage answers. "
                    "Output ONLY the question, nothing else.\n\n"
                    f"Passage: {passage[:1500]}"
                ),
            }],
        )
        return response.content[0].text.strip()

    def generate_faithful_answer(self, query: str, context: str) -> str:
        """Generate an answer that is faithful to the context."""
        response = self.client.messages.create(
            model=self.model,
            max_tokens=300,
            messages=[{
                "role": "user",
                "content": (
                    "Answer this question using ONLY information from the "
                    "provided context. Do not add any information not present "
                    "in the context.\n\n"
                    f"Question: {query}\n\n"
                    f"Context: {context[:2000]}\n\n"
                    "Answer:"
                ),
            }],
        )
        return response.content[0].text.strip()

    def generate_unfaithful_answer(self, query: str, context: str) -> str:
        """Generate an answer that contains plausible but unsupported claims."""
        response = self.client.messages.create(
            model=self.model,
            max_tokens=300,
            messages=[{
                "role": "user",
                "content": (
                    "Answer this question. Include some information from the "
                    "context but also add 1-2 plausible-sounding claims that "
                    "are NOT supported by the context. Make the unsupported "
                    "claims blend in naturally.\n\n"
                    f"Question: {query}\n\n"
                    f"Context: {context[:2000]}\n\n"
                    "Answer (with some unsupported claims mixed in):"
                ),
            }],
        )
        return response.content[0].text.strip()

    def generate_irrelevant_answer(self, query: str) -> str:
        """Generate an answer that does not address the question."""
        response = self.client.messages.create(
            model=self.model,
            max_tokens=300,
            messages=[{
                "role": "user",
                "content": (
                    "Generate a response that sounds informative but does NOT "
                    "actually answer this question. The response should be "
                    "on a tangentially related topic.\n\n"
                    f"Question: {query}\n\n"
                    "Off-topic response:"
                ),
            }],
        )
        return response.content[0].text.strip()

    def generate_training_batch(
        self,
        corpus_passages: list[str],
        num_examples: int = 1000,
    ) -> list[SyntheticExample]:
        """
        Generate a balanced training batch.

        For each passage, generates:
        - 1 fully positive example (relevant context, faithful answer, relevant answer)
        - 1 unfaithful example (relevant context, unfaithful answer, relevant query)
        - 1 irrelevant-context example (random context, faithful-ish answer)
        - 1 irrelevant-answer example (relevant context, off-topic answer)
        """
        examples = []
        passages_needed = num_examples // 4

        selected = random.sample(
            corpus_passages,
            min(passages_needed, len(corpus_passages)),
        )

        for passage in selected:
            query = self.generate_query_from_passage(passage)

            # Positive example: everything is good
            faithful_answer = self.generate_faithful_answer(query, passage)
            examples.append(SyntheticExample(
                query=query,
                context=passage,
                answer=faithful_answer,
                context_relevance=1,
                answer_faithfulness=1,
                answer_relevance=1,
            ))

            # Unfaithful answer: context is relevant but answer hallucinates
            unfaithful_answer = self.generate_unfaithful_answer(query, passage)
            examples.append(SyntheticExample(
                query=query,
                context=passage,
                answer=unfaithful_answer,
                context_relevance=1,
                answer_faithfulness=0,
                answer_relevance=1,
            ))

            # Irrelevant context: random passage paired with query
            random_passage = random.choice(corpus_passages)
            while random_passage == passage:
                random_passage = random.choice(corpus_passages)
            examples.append(SyntheticExample(
                query=query,
                context=random_passage,
                answer=faithful_answer,
                context_relevance=0,
                answer_faithfulness=1,
                answer_relevance=1,
            ))

            # Irrelevant answer: context is fine but answer is off-topic
            irrelevant_answer = self.generate_irrelevant_answer(query)
            examples.append(SyntheticExample(
                query=query,
                context=passage,
                answer=irrelevant_answer,
                context_relevance=1,
                answer_faithfulness=0,
                answer_relevance=0,
            ))

        random.shuffle(examples)
        return examples[:num_examples]

    def save_training_data(
        self,
        examples: list[SyntheticExample],
        output_path: str,
    ) -> None:
        """Save synthetic training data to JSON."""
        data = []
        for ex in examples:
            data.append({
                "query": ex.query,
                "context": ex.context,
                "answer": ex.answer,
                "context_relevance": ex.context_relevance,
                "answer_faithfulness": ex.answer_faithfulness,
                "answer_relevance": ex.answer_relevance,
            })

        with open(output_path, "w") as f:
            json.dump(data, f, indent=2)

        print(f"Saved {len(data)} synthetic examples to {output_path}")

        # Distribution summary
        cr_pos = sum(1 for d in data if d["context_relevance"] == 1)
        af_pos = sum(1 for d in data if d["answer_faithfulness"] == 1)
        ar_pos = sum(1 for d in data if d["answer_relevance"] == 1)
        print(f"Context relevance: {cr_pos}/{len(data)} positive")
        print(f"Answer faithfulness: {af_pos}/{len(data)} positive")
        print(f"Answer relevance: {ar_pos}/{len(data)} positive")
```

---

## Classifier Fine-Tuning

### Training the Three Classifiers

```python
"""
Fine-tune DeBERTa classifiers for ARES evaluation dimensions.
"""
import json

import numpy as np
import torch
from sklearn.metrics import accuracy_score, f1_score, roc_auc_score
from sklearn.model_selection import train_test_split
from torch.utils.data import DataLoader, Dataset
from transformers import (
    AutoModelForSequenceClassification,
    AutoTokenizer,
    TrainingArguments,
    Trainer,
)


class ARESClassifierDataset(Dataset):
    """Dataset for ARES classifier training."""

    def __init__(
        self,
        examples: list[dict],
        tokenizer,
        dimension: str,
        max_length: int = 512,
    ):
        self.tokenizer = tokenizer
        self.dimension = dimension
        self.max_length = max_length

        # Build input pairs based on dimension
        self.inputs = []
        self.labels = []

        for ex in examples:
            if dimension == "context_relevance":
                text_a = ex["query"]
                text_b = ex["context"]
            elif dimension == "answer_faithfulness":
                text_a = ex["context"]
                text_b = ex["answer"]
            elif dimension == "answer_relevance":
                text_a = ex["query"]
                text_b = ex["answer"]
            else:
                raise ValueError(f"Unknown dimension: {dimension}")

            self.inputs.append((text_a, text_b))
            self.labels.append(ex[dimension])

    def __len__(self):
        return len(self.inputs)

    def __getitem__(self, idx):
        text_a, text_b = self.inputs[idx]
        encoding = self.tokenizer(
            text_a,
            text_b,
            max_length=self.max_length,
            padding="max_length",
            truncation=True,
            return_tensors="pt",
        )
        return {
            "input_ids": encoding["input_ids"].squeeze(),
            "attention_mask": encoding["attention_mask"].squeeze(),
            "labels": torch.tensor(self.labels[idx], dtype=torch.long),
        }


def compute_metrics(eval_pred):
    """Compute evaluation metrics for classifier training."""
    logits, labels = eval_pred
    predictions = np.argmax(logits, axis=-1)
    probs = torch.softmax(torch.tensor(logits), dim=-1)[:, 1].numpy()

    return {
        "accuracy": accuracy_score(labels, predictions),
        "f1": f1_score(labels, predictions),
        "auc_roc": roc_auc_score(labels, probs),
    }


def train_ares_classifier(
    training_data_path: str,
    dimension: str,
    output_dir: str,
    base_model: str = "microsoft/deberta-v3-base",
    epochs: int = 3,
    batch_size: int = 16,
    learning_rate: float = 2e-5,
) -> dict:
    """
    Train a single ARES classifier for one evaluation dimension.

    Args:
        training_data_path: path to synthetic training data JSON
        dimension: one of 'context_relevance', 'answer_faithfulness', 'answer_relevance'
        output_dir: where to save the trained model
        base_model: HuggingFace model ID
        epochs: number of training epochs
        batch_size: training batch size
        learning_rate: learning rate

    Returns:
        dict with training results and validation metrics
    """
    with open(training_data_path) as f:
        all_data = json.load(f)

    # Split into train and validation
    train_data, val_data = train_test_split(
        all_data, test_size=0.1, random_state=42,
        stratify=[d[dimension] for d in all_data],
    )

    tokenizer = AutoTokenizer.from_pretrained(base_model)
    model = AutoModelForSequenceClassification.from_pretrained(
        base_model, num_labels=2,
    )

    train_dataset = ARESClassifierDataset(train_data, tokenizer, dimension)
    val_dataset = ARESClassifierDataset(val_data, tokenizer, dimension)

    training_args = TrainingArguments(
        output_dir=output_dir,
        num_train_epochs=epochs,
        per_device_train_batch_size=batch_size,
        per_device_eval_batch_size=batch_size * 2,
        learning_rate=learning_rate,
        weight_decay=0.01,
        warmup_ratio=0.1,
        eval_strategy="epoch",
        save_strategy="epoch",
        load_best_model_at_end=True,
        metric_for_best_model="auc_roc",
        logging_steps=50,
        fp16=torch.cuda.is_available(),
        dataloader_num_workers=4,
    )

    trainer = Trainer(
        model=model,
        args=training_args,
        train_dataset=train_dataset,
        eval_dataset=val_dataset,
        compute_metrics=compute_metrics,
    )

    train_result = trainer.train()
    eval_result = trainer.evaluate()

    # Save the best model
    trainer.save_model(f"{output_dir}/best")
    tokenizer.save_pretrained(f"{output_dir}/best")

    print(f"\n=== {dimension} Classifier Results ===")
    print(f"Training loss: {train_result.training_loss:.4f}")
    print(f"Validation accuracy: {eval_result['eval_accuracy']:.4f}")
    print(f"Validation F1: {eval_result['eval_f1']:.4f}")
    print(f"Validation AUC-ROC: {eval_result['eval_auc_roc']:.4f}")

    return {
        "dimension": dimension,
        "train_loss": train_result.training_loss,
        "eval_metrics": eval_result,
    }


def train_all_ares_classifiers(
    training_data_path: str,
    output_base_dir: str = "./ares_classifiers",
) -> dict:
    """Train all three ARES classifiers."""
    results = {}
    for dimension in [
        "context_relevance",
        "answer_faithfulness",
        "answer_relevance",
    ]:
        print(f"\n{'='*60}")
        print(f"Training {dimension} classifier...")
        print(f"{'='*60}")

        result = train_ares_classifier(
            training_data_path=training_data_path,
            dimension=dimension,
            output_dir=f"{output_base_dir}/{dimension}",
        )
        results[dimension] = result

    return results
```

---

## Running ARES Evaluation on Your Pipeline

### End-to-End Evaluation Flow

```python
"""
Complete ARES evaluation of a RAG pipeline.
"""
import json

import numpy as np
import torch
from scipy import stats
from transformers import AutoModelForSequenceClassification, AutoTokenizer


class ARESEvaluator:
    """
    Evaluate a RAG pipeline using trained ARES classifiers + PPI.
    """

    def __init__(self, classifiers_dir: str):
        self.classifiers = {}
        self.tokenizers = {}

        for dimension in [
            "context_relevance",
            "answer_faithfulness",
            "answer_relevance",
        ]:
            model_path = f"{classifiers_dir}/{dimension}/best"
            self.classifiers[dimension] = (
                AutoModelForSequenceClassification.from_pretrained(model_path)
            )
            self.tokenizers[dimension] = (
                AutoTokenizer.from_pretrained(model_path)
            )
            self.classifiers[dimension].eval()

        self.device = torch.device(
            "cuda" if torch.cuda.is_available() else "cpu"
        )
        for dim in self.classifiers:
            self.classifiers[dim].to(self.device)

    def _predict_dimension(
        self,
        dimension: str,
        text_pairs: list[tuple[str, str]],
        batch_size: int = 32,
    ) -> np.ndarray:
        """Run classifier on text pairs and return P(positive)."""
        tokenizer = self.tokenizers[dimension]
        model = self.classifiers[dimension]
        all_probs = []

        for i in range(0, len(text_pairs), batch_size):
            batch = text_pairs[i : i + batch_size]
            texts_a = [p[0] for p in batch]
            texts_b = [p[1] for p in batch]

            encoding = tokenizer(
                texts_a, texts_b,
                max_length=512,
                padding=True,
                truncation=True,
                return_tensors="pt",
            ).to(self.device)

            with torch.no_grad():
                outputs = model(**encoding)
                probs = torch.softmax(outputs.logits, dim=-1)[:, 1]
                all_probs.extend(probs.cpu().numpy())

        return np.array(all_probs)

    def score_pipeline_outputs(
        self,
        queries: list[str],
        contexts: list[str],
        answers: list[str],
    ) -> dict:
        """
        Score all pipeline outputs with the three classifiers.

        Returns per-example scores for each dimension.
        """
        cr_pairs = list(zip(queries, contexts))
        af_pairs = list(zip(contexts, answers))
        ar_pairs = list(zip(queries, answers))

        return {
            "context_relevance": self._predict_dimension(
                "context_relevance", cr_pairs
            ),
            "answer_faithfulness": self._predict_dimension(
                "answer_faithfulness", af_pairs
            ),
            "answer_relevance": self._predict_dimension(
                "answer_relevance", ar_pairs
            ),
        }

    def evaluate_with_ppi(
        self,
        pipeline_scores: dict,
        human_labels: dict,
        labeled_indices: list[int],
        alpha: float = 0.05,
    ) -> dict:
        """
        Apply PPI to combine classifier scores with human labels.

        Args:
            pipeline_scores: dict of dimension -> np.ndarray of classifier scores
            human_labels: dict of dimension -> np.ndarray of human labels
                          for the labeled subset
            labeled_indices: indices into pipeline_scores that have human labels
            alpha: significance level for confidence intervals

        Returns:
            dict of dimension -> {point_estimate, ci_lower, ci_upper}
        """
        results = {}

        for dimension in pipeline_scores:
            all_scores = pipeline_scores[dimension]
            labeled_scores = all_scores[labeled_indices]
            true_labels = human_labels[dimension]

            N = len(all_scores)
            n = len(true_labels)

            # Binarize classifier scores at 0.5 threshold
            all_binary = (all_scores >= 0.5).astype(float)
            labeled_binary = (labeled_scores >= 0.5).astype(float)

            # PPI estimate
            theta_f = np.mean(all_binary)
            correction = np.mean(true_labels - labeled_binary)
            theta_ppi = theta_f + correction

            # Confidence interval
            residuals = true_labels - labeled_binary
            var_correction = np.var(residuals, ddof=1) / n
            var_classifier = np.var(all_binary, ddof=1) / N
            var_total = var_correction + var_classifier

            z = stats.norm.ppf(1 - alpha / 2)
            ci_lower = theta_ppi - z * np.sqrt(var_total)
            ci_upper = theta_ppi + z * np.sqrt(var_total)

            results[dimension] = {
                "point_estimate": float(np.clip(theta_ppi, 0, 1)),
                "ci_lower": float(np.clip(ci_lower, 0, 1)),
                "ci_upper": float(np.clip(ci_upper, 0, 1)),
                "ci_width": float(ci_upper - ci_lower),
                "classifier_raw": float(theta_f),
                "ppi_correction": float(correction),
                "n_labeled": n,
                "n_unlabeled": N,
            }

        return results

    def compare_pipelines(
        self,
        pipeline_a_scores: dict,
        pipeline_b_scores: dict,
        human_labels_a: dict,
        human_labels_b: dict,
        labeled_indices_a: list[int],
        labeled_indices_b: list[int],
        alpha: float = 0.05,
    ) -> dict:
        """
        Compare two pipeline configurations with statistical significance.

        Returns: for each dimension, whether the difference is significant.
        """
        results_a = self.evaluate_with_ppi(
            pipeline_a_scores, human_labels_a, labeled_indices_a, alpha
        )
        results_b = self.evaluate_with_ppi(
            pipeline_b_scores, human_labels_b, labeled_indices_b, alpha
        )

        comparison = {}
        for dimension in results_a:
            a = results_a[dimension]
            b = results_b[dimension]
            diff = b["point_estimate"] - a["point_estimate"]

            # CIs overlap test (conservative)
            significant = (
                b["ci_lower"] > a["ci_upper"]
                or a["ci_lower"] > b["ci_upper"]
            )

            comparison[dimension] = {
                "pipeline_a": a["point_estimate"],
                "pipeline_b": b["point_estimate"],
                "difference": diff,
                "significant": significant,
                "pipeline_a_ci": (a["ci_lower"], a["ci_upper"]),
                "pipeline_b_ci": (b["ci_lower"], b["ci_upper"]),
            }

            status = "SIGNIFICANT" if significant else "NOT SIGNIFICANT"
            direction = "B > A" if diff > 0 else "A > B"
            print(
                f"{dimension}: A={a['point_estimate']:.3f} "
                f"[{a['ci_lower']:.3f},{a['ci_upper']:.3f}], "
                f"B={b['point_estimate']:.3f} "
                f"[{b['ci_lower']:.3f},{b['ci_upper']:.3f}] "
                f"-- {direction}, {status}"
            )

        return comparison
```

---

## Cost and Time Analysis

### One-Time Setup Costs

| Step | LLM Calls | GPU Time | Approx. Cost |
|---|---|---|---|
| Generate 5K synthetic examples | 5K calls (Claude Haiku) | -- | $3-5 |
| Fine-tune 3 classifiers | -- | 3 x 20 min (single GPU) | $2-5 (cloud GPU) |
| Annotate 100 human labels | -- | 2-4 hours human time | $30-50 |
| **Total one-time setup** | | | **$35-60** |

### Per-Evaluation Costs

| Step | Cost | Time |
|---|---|---|
| Run RAG pipeline on eval set (1000 queries) | Depends on pipeline | 10-30 min |
| Score with ARES classifiers | ~$0 (local inference) | 30-60 sec |
| PPI computation | ~$0 (numpy) | < 1 sec |
| **Total per evaluation** | **~$0** | **< 2 min** |

This makes ARES ideal for CI/CD integration where you run evaluation on every pipeline change.

---

## Common Pitfalls

1. **Too few human annotations for PPI**: with fewer than 50 labeled examples, PPI confidence intervals become too wide to be useful. The point estimate may still be reasonable, but you cannot make confident comparisons between pipelines.

2. **Synthetic data quality**: if your synthetic generation LLM produces poor negative examples (too obviously wrong), the classifier will not learn to catch subtle issues. Use a strong model (Claude Sonnet or better) for generation and manually inspect 20-30 examples before committing to a full training run.

3. **Domain mismatch in base model**: DeBERTa-v3-base is trained on general English text. For highly specialized domains (e.g., clinical notes, legal statutes), consider intermediate pretraining on domain text before ARES fine-tuning.

4. **Ignoring classifier calibration**: ARES classifiers output probabilities, but these probabilities may not be well-calibrated (a score of 0.8 may not mean 80% chance of relevance). PPI corrects for this at the aggregate level, but per-example scores should be interpreted with caution.

5. **Reusing the same human labels across pipeline iterations**: PPI assumes the human labels are from the same distribution as the unlabeled data. If you change your pipeline significantly (different retriever, different prompt), re-annotate a fresh labeled set.

6. **Overfitting classifiers to synthetic patterns**: if your synthetic data has artifacts (e.g., unfaithful answers always start with "According to..."), the classifier learns shortcuts rather than genuine faithfulness assessment. Diversify your generation prompts and validate on a held-out set.

---

## References

- Saad-Falcon, J. et al. "ARES: An Automated Evaluation Framework for Retrieval-Augmented Generation Systems." arXiv 2023. https://arxiv.org/abs/2311.09476
- Angelopoulos, A. et al. "Prediction-Powered Inference." Science, 2023.
- Stanford ARES GitHub: https://github.com/stanford-futuredata/ARES
- ares-ai PyPI package: https://pypi.org/project/ares-ai/
- Saad-Falcon, J. et al. "Benchmarking ARES Against Other RAG Evaluation Approaches." 2024.
