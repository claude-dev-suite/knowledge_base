# Cross-Encoder Training -- Complete Training Guide

## TL;DR

This guide walks through fine-tuning a cross-encoder reranker from scratch using sentence-transformers. It covers the full workflow: preparing training data in MS MARCO format, mining hard negatives from BM25 and dense retrieval, choosing between BCE loss and margin ranking loss, configuring the sentence-transformers CrossEncoder trainer, training with proper hyperparameters, and distilling from a strong LLM reranker into a fast cross-encoder. Every step includes working Python code with the exact class names and parameters from the sentence-transformers library, so you can copy-paste and adapt to your domain.

---

## Training Data Preparation

### MS MARCO Format

The standard training format for cross-encoders is query-document-label triples:

```
query_id | query_text | doc_id | doc_text | label
```

Where label is:
- **Binary**: 0 (not relevant), 1 (relevant)
- **Score**: continuous relevance score (0.0 to 1.0)

```python
"""
Prepare training data for cross-encoder fine-tuning.
"""
import csv
import json
import random
from dataclasses import dataclass
from pathlib import Path

import numpy as np


@dataclass
class TrainingExample:
    """A single training example for cross-encoder training."""
    query: str
    document: str
    label: float  # 0.0 (not relevant) to 1.0 (highly relevant)


class TrainingDataPreparator:
    """
    Prepare training data from various sources.

    Supports:
    - Manual relevance judgments (TSV/CSV)
    - LLM-generated labels
    - Click-through data
    - Distillation from a teacher model
    """

    def from_relevance_judgments(
        self,
        judgments_path: str,
        queries_path: str,
        documents_path: str,
    ) -> list[TrainingExample]:
        """
        Load training data from manual relevance judgments.

        Expected judgments format (TSV):
        query_id \t doc_id \t relevance
        """
        # Load queries
        queries = {}
        with open(queries_path) as f:
            for line in f:
                parts = line.strip().split("\t")
                queries[parts[0]] = parts[1]

        # Load documents
        documents = {}
        with open(documents_path) as f:
            for line in f:
                parts = line.strip().split("\t")
                documents[parts[0]] = parts[1]

        # Load judgments
        examples = []
        with open(judgments_path) as f:
            reader = csv.reader(f, delimiter="\t")
            for row in reader:
                q_id, doc_id, relevance = row[0], row[1], float(row[2])
                if q_id in queries and doc_id in documents:
                    examples.append(TrainingExample(
                        query=queries[q_id],
                        document=documents[doc_id],
                        label=relevance,
                    ))

        print(f"Loaded {len(examples)} training examples")
        return examples

    def from_llm_labels(
        self,
        queries: list[str],
        documents: list[str],
        model: str = "claude-sonnet-4-20250514",
        batch_size: int = 10,
    ) -> list[TrainingExample]:
        """
        Generate training labels using an LLM judge.

        Cost-effective way to create training data when manual
        annotation is not feasible.
        """
        import anthropic
        client = anthropic.Anthropic()

        examples = []

        for i in range(0, len(queries), batch_size):
            batch_queries = queries[i : i + batch_size]
            batch_docs = documents[i : i + batch_size]

            for query, doc in zip(batch_queries, batch_docs):
                response = client.messages.create(
                    model=model,
                    max_tokens=10,
                    messages=[{
                        "role": "user",
                        "content": (
                            "Rate the relevance of this document to the query.\n"
                            "Score: 0 (not relevant), 1 (somewhat relevant), "
                            "2 (relevant), 3 (highly relevant).\n\n"
                            f"Query: {query}\n\n"
                            f"Document: {doc[:1000]}\n\n"
                            "Score (0-3):"
                        ),
                    }],
                )

                try:
                    score = int(response.content[0].text.strip()[0])
                    label = score / 3.0  # Normalize to [0, 1]
                except (ValueError, IndexError):
                    label = 0.0

                examples.append(TrainingExample(
                    query=query, document=doc, label=label,
                ))

        return examples

    def save_training_data(
        self,
        examples: list[TrainingExample],
        output_path: str,
    ) -> None:
        """Save training data in JSON format."""
        data = [
            {"query": ex.query, "document": ex.document, "label": ex.label}
            for ex in examples
        ]
        with open(output_path, "w") as f:
            json.dump(data, f, indent=2)
        print(f"Saved {len(data)} examples to {output_path}")
```

---

## Hard Negative Mining

### Why Hard Negatives Are Critical

Cross-encoder quality depends heavily on the quality of negative examples. Random negatives are too easy -- the model learns nothing useful from distinguishing "Python asyncio" from "Roman Empire history."

### Mining from BM25

```python
"""
Mine hard negatives from BM25 retrieval.
"""
import json
from typing import Callable

import numpy as np


class BM25HardNegativeMiner:
    """
    Mine hard negatives using BM25 retrieval.

    BM25 negatives are documents that match query keywords but
    are not actually relevant. These teach the model to look
    beyond keyword overlap.
    """

    def __init__(self, bm25_search_fn: Callable):
        """
        Args:
            bm25_search_fn: function(query, k) -> list[dict]
                Returns ranked documents with 'text' and 'doc_id' fields.
        """
        self.search = bm25_search_fn

    def mine_negatives(
        self,
        queries: list[str],
        positive_doc_ids: list[set[str]],
        negatives_per_query: int = 5,
        top_k: int = 100,
        skip_top: int = 0,
    ) -> list[list[str]]:
        """
        Mine hard negatives for a set of queries.

        Args:
            queries: list of query texts
            positive_doc_ids: for each query, set of known-relevant doc IDs
            negatives_per_query: how many hard negatives to mine per query
            top_k: how deep to search for candidates
            skip_top: skip top-N results (too likely to be relevant)

        Returns:
            list of lists of hard negative document texts
        """
        all_negatives = []

        for i, (query, pos_ids) in enumerate(zip(queries, positive_doc_ids)):
            results = self.search(query, top_k)

            # Select non-relevant documents from the ranked list
            negatives = []
            for j, doc in enumerate(results):
                if j < skip_top:
                    continue  # Skip very top results (likely relevant)
                if doc["doc_id"] not in pos_ids:
                    negatives.append(doc["text"])
                if len(negatives) >= negatives_per_query:
                    break

            all_negatives.append(negatives)

            if (i + 1) % 100 == 0:
                print(f"Mined negatives for {i + 1}/{len(queries)} queries")

        # Stats
        avg_negs = np.mean([len(n) for n in all_negatives])
        print(f"Average negatives per query: {avg_negs:.1f}")

        return all_negatives


class DenseHardNegativeMiner:
    """
    Mine hard negatives from dense (embedding) retrieval.

    Dense negatives are documents that are semantically similar
    to the query but not relevant. These teach the model to
    distinguish semantic similarity from relevance.
    """

    def __init__(self, embedding_fn, vector_store):
        """
        Args:
            embedding_fn: function(text) -> np.ndarray
            vector_store: vector store with search(vector, k) method
        """
        self.embed = embedding_fn
        self.store = vector_store

    def mine_negatives(
        self,
        queries: list[str],
        positive_doc_ids: list[set[str]],
        negatives_per_query: int = 5,
        top_k: int = 100,
    ) -> list[list[str]]:
        """Mine hard negatives from dense retrieval."""
        all_negatives = []

        for query, pos_ids in zip(queries, positive_doc_ids):
            query_vector = self.embed(query)
            results = self.store.search(query_vector, top_k)

            negatives = []
            for doc in results:
                if doc["doc_id"] not in pos_ids:
                    negatives.append(doc["text"])
                if len(negatives) >= negatives_per_query:
                    break

            all_negatives.append(negatives)

        return all_negatives


def combine_hard_negatives(
    bm25_negatives: list[list[str]],
    dense_negatives: list[list[str]],
    total_per_query: int = 7,
    bm25_ratio: float = 0.5,
) -> list[list[str]]:
    """
    Combine BM25 and dense hard negatives.

    Mixing sources creates harder training data because BM25 and
    dense retrieval find different types of distractors.
    """
    combined = []
    bm25_count = int(total_per_query * bm25_ratio)
    dense_count = total_per_query - bm25_count

    for bm25_negs, dense_negs in zip(bm25_negatives, dense_negatives):
        selected = bm25_negs[:bm25_count] + dense_negs[:dense_count]
        random.shuffle(selected)
        combined.append(selected[:total_per_query])

    return combined
```

---

## Training with sentence-transformers

### BCE Loss Training

```python
"""
Train a cross-encoder with Binary Cross-Entropy loss.
"""
import json
import math

import torch
from sentence_transformers import InputExample
from sentence_transformers.cross_encoder import CrossEncoder
from sentence_transformers.cross_encoder.evaluation import (
    CERerankingEvaluator,
)
from sklearn.model_selection import train_test_split
from torch.utils.data import DataLoader


def train_cross_encoder_bce(
    training_data_path: str,
    output_dir: str = "./cross-encoder-finetuned",
    base_model: str = "cross-encoder/ms-marco-MiniLM-L-12-v2",
    num_epochs: int = 3,
    batch_size: int = 32,
    learning_rate: float = 2e-5,
    max_length: int = 512,
    warmup_fraction: float = 0.1,
) -> CrossEncoder:
    """
    Train a cross-encoder using BCE loss.

    Best for: binary relevance labels (relevant / not relevant).
    """
    # Load training data
    with open(training_data_path) as f:
        data = json.load(f)

    # Split into train and dev
    train_data, dev_data = train_test_split(
        data, test_size=0.1, random_state=42,
    )

    print(f"Training examples: {len(train_data)}")
    print(f"Dev examples: {len(dev_data)}")

    # Convert to InputExamples
    train_examples = [
        InputExample(texts=[d["query"], d["document"]], label=float(d["label"]))
        for d in train_data
    ]

    # Prepare dev evaluator
    dev_samples = {}
    for d in dev_data:
        query = d["query"]
        if query not in dev_samples:
            dev_samples[query] = {"positive": set(), "negative": set()}
        if d["label"] >= 0.5:
            dev_samples[query]["positive"].add(d["document"])
        else:
            dev_samples[query]["negative"].add(d["document"])

    evaluator = CERerankingEvaluator(dev_samples, name="dev")

    # Initialize cross-encoder
    model = CrossEncoder(
        base_model,
        num_labels=1,  # Regression (single score output)
        max_length=max_length,
    )

    # Create DataLoader
    train_dataloader = DataLoader(
        train_examples,
        shuffle=True,
        batch_size=batch_size,
    )

    # Calculate warmup steps
    total_steps = len(train_dataloader) * num_epochs
    warmup_steps = math.ceil(total_steps * warmup_fraction)

    # Train
    model.fit(
        train_dataloader=train_dataloader,
        evaluator=evaluator,
        epochs=num_epochs,
        warmup_steps=warmup_steps,
        output_path=output_dir,
        optimizer_params={"lr": learning_rate},
        weight_decay=0.01,
        evaluation_steps=500,
        save_best_model=True,
        show_progress_bar=True,
    )

    print(f"\nTraining complete. Best model saved to {output_dir}")
    return model
```

### Margin Ranking Loss Training

```python
"""
Train a cross-encoder with Margin Ranking Loss.
"""
import json
import math
import random

from sentence_transformers import InputExample
from sentence_transformers.cross_encoder import CrossEncoder
from sentence_transformers.cross_encoder.evaluation import (
    CERerankingEvaluator,
)
from torch.utils.data import DataLoader


def prepare_pairwise_data(
    data: list[dict],
    hard_negatives: list[list[str]] | None = None,
) -> list[InputExample]:
    """
    Convert pointwise labels to pairwise training examples.

    Margin ranking loss requires (query, positive_doc, negative_doc) triples.
    """
    # Group by query
    by_query = {}
    for d in data:
        query = d["query"]
        if query not in by_query:
            by_query[query] = {"positive": [], "negative": []}
        if d["label"] >= 0.5:
            by_query[query]["positive"].append(d["document"])
        else:
            by_query[query]["negative"].append(d["document"])

    # Create pairwise examples
    examples = []
    for query, docs in by_query.items():
        for pos_doc in docs["positive"]:
            negatives = docs["negative"]

            if not negatives:
                continue

            for neg_doc in negatives[:5]:  # Max 5 negatives per positive
                # Label: 1.0 means first doc is more relevant
                examples.append(InputExample(
                    texts=[query, pos_doc, neg_doc],
                    label=1.0,
                ))

    random.shuffle(examples)
    print(f"Created {len(examples)} pairwise training examples")
    return examples


def train_cross_encoder_margin(
    training_data_path: str,
    output_dir: str = "./cross-encoder-margin",
    base_model: str = "cross-encoder/ms-marco-MiniLM-L-12-v2",
    num_epochs: int = 3,
    batch_size: int = 16,
    learning_rate: float = 2e-5,
    margin: float = 1.0,
) -> CrossEncoder:
    """
    Train a cross-encoder using Margin Ranking Loss.

    Best for: when you have explicit positive/negative pairs and want
    clear score separation between relevant and irrelevant documents.
    """
    with open(training_data_path) as f:
        data = json.load(f)

    train_data, dev_data = train_test_split(
        data, test_size=0.1, random_state=42,
    )

    # Prepare pairwise training data
    train_examples = prepare_pairwise_data(train_data)

    # Prepare dev evaluator
    dev_samples = {}
    for d in dev_data:
        query = d["query"]
        if query not in dev_samples:
            dev_samples[query] = {"positive": set(), "negative": set()}
        if d["label"] >= 0.5:
            dev_samples[query]["positive"].add(d["document"])
        else:
            dev_samples[query]["negative"].add(d["document"])

    evaluator = CERerankingEvaluator(dev_samples, name="dev")

    # Initialize
    model = CrossEncoder(base_model, num_labels=1, max_length=512)

    train_dataloader = DataLoader(
        train_examples, shuffle=True, batch_size=batch_size,
    )

    total_steps = len(train_dataloader) * num_epochs
    warmup_steps = math.ceil(total_steps * 0.1)

    model.fit(
        train_dataloader=train_dataloader,
        evaluator=evaluator,
        epochs=num_epochs,
        warmup_steps=warmup_steps,
        output_path=output_dir,
        optimizer_params={"lr": learning_rate},
        evaluation_steps=500,
        save_best_model=True,
    )

    return model


# Helper for split
from sklearn.model_selection import train_test_split
```

---

## Distillation from a Strong Reranker

### Teacher-Student Training

```python
"""
Distill knowledge from a strong LLM reranker into a fast cross-encoder.
"""
import json
import math

import anthropic
import numpy as np
from sentence_transformers import InputExample
from sentence_transformers.cross_encoder import CrossEncoder
from sentence_transformers.cross_encoder.evaluation import (
    CERerankingEvaluator,
)
from torch.utils.data import DataLoader


class LLMRerankerTeacher:
    """
    Use an LLM as a teacher model to generate soft labels
    for cross-encoder distillation.
    """

    def __init__(self, model: str = "claude-sonnet-4-20250514"):
        self.client = anthropic.Anthropic()
        self.model = model

    def score_pair(self, query: str, document: str) -> float:
        """Get a relevance score from the LLM teacher."""
        response = self.client.messages.create(
            model=self.model,
            max_tokens=10,
            messages=[{
                "role": "user",
                "content": (
                    "Rate the relevance of this document to the query "
                    "on a scale from 0.0 to 1.0, where 0.0 means completely "
                    "irrelevant and 1.0 means perfectly relevant.\n\n"
                    f"Query: {query}\n\n"
                    f"Document: {document[:1500]}\n\n"
                    "Relevance score (0.0-1.0):"
                ),
            }],
        )

        try:
            score = float(response.content[0].text.strip())
            return max(0.0, min(1.0, score))
        except (ValueError, IndexError):
            return 0.0

    def generate_soft_labels(
        self,
        queries: list[str],
        documents: list[list[str]],
    ) -> list[list[float]]:
        """
        Generate soft labels for all query-document pairs.

        Args:
            queries: list of queries
            documents: for each query, list of candidate documents

        Returns:
            For each query, list of relevance scores (soft labels)
        """
        all_scores = []

        for i, (query, docs) in enumerate(zip(queries, documents)):
            scores = []
            for doc in docs:
                score = self.score_pair(query, doc)
                scores.append(score)
            all_scores.append(scores)

            if (i + 1) % 20 == 0:
                print(f"Scored {i + 1}/{len(queries)} queries")

        return all_scores


def train_distilled_cross_encoder(
    queries: list[str],
    documents: list[list[str]],
    teacher_scores: list[list[float]],
    output_dir: str = "./cross-encoder-distilled",
    base_model: str = "cross-encoder/ms-marco-MiniLM-L-12-v2",
    num_epochs: int = 5,
    batch_size: int = 32,
    learning_rate: float = 3e-5,
) -> CrossEncoder:
    """
    Train a cross-encoder using soft labels from an LLM teacher.

    The student learns to match the teacher's relevance scores,
    effectively compressing LLM-level reranking quality into a
    model that runs 100-1000x faster.
    """
    # Create training examples with soft labels
    train_examples = []
    for query, docs, scores in zip(queries, documents, teacher_scores):
        for doc, score in zip(docs, scores):
            train_examples.append(InputExample(
                texts=[query, doc],
                label=score,  # Soft label from teacher
            ))

    print(f"Training examples: {len(train_examples)}")
    print(f"Score distribution: mean={np.mean([e.label for e in train_examples]):.3f}, "
          f"std={np.std([e.label for e in train_examples]):.3f}")

    # Split for evaluation
    from sklearn.model_selection import train_test_split
    train_ex, dev_ex = train_test_split(
        train_examples, test_size=0.1, random_state=42,
    )

    # Build evaluator
    dev_samples = {}
    for ex in dev_ex:
        query = ex.texts[0]
        doc = ex.texts[1]
        if query not in dev_samples:
            dev_samples[query] = {"positive": set(), "negative": set()}
        if ex.label >= 0.5:
            dev_samples[query]["positive"].add(doc)
        else:
            dev_samples[query]["negative"].add(doc)

    evaluator = CERerankingEvaluator(dev_samples, name="distill-dev")

    # Initialize student model
    model = CrossEncoder(base_model, num_labels=1, max_length=512)

    train_dataloader = DataLoader(
        train_ex, shuffle=True, batch_size=batch_size,
    )

    total_steps = len(train_dataloader) * num_epochs
    warmup_steps = math.ceil(total_steps * 0.1)

    # MSE loss is used by default for num_labels=1 (regression)
    # This makes the student match the teacher's scores
    model.fit(
        train_dataloader=train_dataloader,
        evaluator=evaluator,
        epochs=num_epochs,
        warmup_steps=warmup_steps,
        output_path=output_dir,
        optimizer_params={"lr": learning_rate},
        evaluation_steps=500,
        save_best_model=True,
    )

    print(f"\nDistillation complete. Model saved to {output_dir}")
    return model
```

### Distillation Cost Analysis

| Step | Cost (1000 queries x 20 docs) | Time |
|---|---|---|
| LLM teacher scoring | $10-30 (Claude Sonnet) | 2-4 hours |
| Student training | $2-5 (GPU) | 30-60 min |
| **Total** | **$12-35** | **3-5 hours** |
| **Per-inference savings** | 100-1000x faster than LLM | Forever |

---

## Advanced Training Techniques

### Curriculum Learning

```python
"""
Curriculum learning: train on easy examples first, then hard ones.
"""
import numpy as np
from sentence_transformers import InputExample


def sort_by_difficulty(
    examples: list[InputExample],
    cross_encoder: CrossEncoder,
    batch_size: int = 64,
) -> list[InputExample]:
    """
    Sort training examples by difficulty (easy to hard).

    Difficulty is measured by how wrong the current model's
    prediction is: examples where the model is most confused
    are the hardest.
    """
    # Score all examples with current model
    pairs = [(ex.texts[0], ex.texts[1]) for ex in examples]
    predictions = cross_encoder.predict(pairs, batch_size=batch_size)

    # Difficulty = |prediction - label| (higher = harder)
    difficulties = []
    for ex, pred in zip(examples, predictions):
        difficulty = abs(pred - ex.label)
        difficulties.append(difficulty)

    # Sort easy to hard
    sorted_indices = np.argsort(difficulties)
    sorted_examples = [examples[i] for i in sorted_indices]

    easy_avg = np.mean(difficulties[sorted_indices[:len(sorted_indices)//3]])
    hard_avg = np.mean(difficulties[sorted_indices[-len(sorted_indices)//3:]])
    print(f"Difficulty range: easy={easy_avg:.3f}, hard={hard_avg:.3f}")

    return sorted_examples
```

### Label Smoothing

```python
"""
Label smoothing to prevent overconfident predictions.
"""


def apply_label_smoothing(
    examples: list[InputExample],
    smoothing: float = 0.1,
) -> list[InputExample]:
    """
    Apply label smoothing to training examples.

    Converts hard labels (0 or 1) to soft labels (0.05 or 0.95)
    to prevent the model from being overconfident.
    """
    smoothed = []
    for ex in examples:
        label = ex.label
        if label == 1.0:
            label = 1.0 - smoothing  # 1.0 -> 0.9
        elif label == 0.0:
            label = smoothing  # 0.0 -> 0.1

        smoothed.append(InputExample(
            texts=ex.texts,
            label=label,
        ))

    return smoothed
```

---

## Training Hyperparameter Grid Search

```python
"""
Hyperparameter search for cross-encoder training.
"""
import itertools
import json
from pathlib import Path


def hyperparameter_search(
    training_data_path: str,
    dev_data: list[dict],
    base_model: str = "cross-encoder/ms-marco-MiniLM-L-12-v2",
    output_base: str = "./hp_search",
) -> dict:
    """
    Grid search over key hyperparameters.

    The most impactful hyperparameters for cross-encoder training:
    1. Learning rate (most important)
    2. Number of epochs
    3. Batch size
    """
    param_grid = {
        "learning_rate": [1e-5, 2e-5, 3e-5, 5e-5],
        "epochs": [2, 3, 5],
        "batch_size": [16, 32],
    }

    # Build dev evaluator
    dev_samples = {}
    for d in dev_data:
        query = d["query"]
        if query not in dev_samples:
            dev_samples[query] = {"positive": set(), "negative": set()}
        if d["label"] >= 0.5:
            dev_samples[query]["positive"].add(d["document"])
        else:
            dev_samples[query]["negative"].add(d["document"])

    results = []

    configs = list(itertools.product(
        param_grid["learning_rate"],
        param_grid["epochs"],
        param_grid["batch_size"],
    ))

    for i, (lr, epochs, bs) in enumerate(configs):
        print(f"\n{'='*50}")
        print(f"Config {i+1}/{len(configs)}: lr={lr}, epochs={epochs}, bs={bs}")
        print(f"{'='*50}")

        run_dir = f"{output_base}/lr{lr}_ep{epochs}_bs{bs}"

        try:
            model = train_cross_encoder_bce(
                training_data_path=training_data_path,
                output_dir=run_dir,
                base_model=base_model,
                num_epochs=epochs,
                batch_size=bs,
                learning_rate=lr,
            )

            # Evaluate on dev
            evaluator = CERerankingEvaluator(dev_samples, name="dev")
            dev_score = evaluator(model, output_path=run_dir)

            results.append({
                "learning_rate": lr,
                "epochs": epochs,
                "batch_size": bs,
                "dev_score": dev_score,
                "model_path": run_dir,
            })

            print(f"Dev score: {dev_score:.4f}")

        except Exception as e:
            print(f"Failed: {e}")
            results.append({
                "learning_rate": lr,
                "epochs": epochs,
                "batch_size": bs,
                "dev_score": -1,
                "error": str(e),
            })

    # Find best
    best = max(results, key=lambda x: x.get("dev_score", -1))
    print(f"\n{'='*50}")
    print(f"Best config: {best}")
    print(f"{'='*50}")

    # Save results
    with open(f"{output_base}/search_results.json", "w") as f:
        json.dump(results, f, indent=2)

    return best


# Import the needed evaluator
from sentence_transformers.cross_encoder.evaluation import CERerankingEvaluator
```

---

## Common Pitfalls

1. **Training without hard negatives**: using only random negatives produces a model that can distinguish obviously irrelevant documents but fails on subtle cases. Always mine hard negatives from BM25 and/or dense retrieval.

2. **Learning rate too high**: cross-encoders are sensitive to learning rate. Rates above 5e-5 often cause catastrophic forgetting. Start with 2e-5 and grid search down to 1e-5 if needed.

3. **Too many epochs on small data**: cross-encoders overfit quickly. With fewer than 5K examples, rarely go beyond 3 epochs. Use early stopping on a validation set.

4. **Not using a pretrained reranker as base**: starting from a raw BERT model wastes training time learning what relevance means. Always start from a model already trained on MS MARCO (e.g., `cross-encoder/ms-marco-MiniLM-L-12-v2`).

5. **Truncating long documents silently**: if your documents are longer than max_length (typically 512 tokens), the cross-encoder silently drops the excess. Verify that your passage lengths fit within the token budget.

6. **Not mixing negative sources**: BM25 negatives and dense negatives teach different things. Use a mix (e.g., 50% BM25, 50% dense) for the most robust training.

---

## References

- sentence-transformers CrossEncoder: https://www.sbert.net/docs/cross_encoder/usage/usage.html
- Nogueira, R. and Cho, K. "Passage Re-ranking with BERT." arXiv 2019.
- Hofstatter, S. et al. "Efficiently Teaching an Effective Dense Retriever with Balanced Topic Aware Sampling." SIGIR 2021.
- Wang, L. et al. "Improving Text Embeddings with Large Language Models." arXiv 2024.
- MS MARCO dataset: https://microsoft.github.io/msmarco/
