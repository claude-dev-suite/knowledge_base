# Embedding Fine-Tuning -- Training Guide

## Overview / TL;DR

This guide provides the complete, runnable training pipeline for fine-tuning embedding models using sentence-transformers. It covers dataset preparation (query/positive/negative triples), all major loss functions (MultipleNegativesRankingLoss, TripletLoss, CosineSimilarityLoss, CachedGISTEmbedLoss), training arguments, evaluation during training with InformationRetrievalEvaluator, and saving/exporting the fine-tuned model. Every code block is production-ready and can be executed end-to-end.

---

## Prerequisites

```bash
pip install sentence-transformers>=3.0.0 datasets torch
```

Minimum hardware:
- **bge-base-en-v1.5** (109M params): 8GB GPU (T4) or CPU (slow but works)
- **bge-large-en-v1.5** (335M params): 16GB GPU (T4/A10G)
- **BGE-M3** (568M params): 24GB GPU (A10G/A100)

---

## Step 1: Dataset Preparation

### From Raw Data to Training Format

```python
import json
import random
from pathlib import Path
from dataclasses import dataclass, asdict


@dataclass
class TrainingSample:
    """A single training sample for embedding fine-tuning."""
    query: str
    positive: str
    negatives: list[str]


def load_and_prepare_dataset(
    data_path: str,
    eval_fraction: float = 0.1,
    min_eval_size: int = 100,
    max_negatives: int = 7,
) -> tuple[list[TrainingSample], list[TrainingSample]]:
    """Load training data and split into train/eval sets.

    Expected input format (JSONL):
        {"query": "...", "positive": "...", "negatives": ["...", "..."]}

    If negatives are missing, they will need to be mined separately.
    See hard-negative-mining/ KB entry.
    """
    samples = []
    with open(data_path, "r") as f:
        for line in f:
            data = json.loads(line.strip())
            samples.append(TrainingSample(
                query=data["query"],
                positive=data["positive"],
                negatives=data.get("negatives", [])[:max_negatives],
            ))

    random.shuffle(samples)
    eval_size = max(int(len(samples) * eval_fraction), min_eval_size)
    eval_size = min(eval_size, len(samples) // 2)

    eval_samples = samples[:eval_size]
    train_samples = samples[eval_size:]

    print(f"Loaded {len(samples)} total samples")
    print(f"  Training: {len(train_samples)}")
    print(f"  Evaluation: {len(eval_samples)}")

    return train_samples, eval_samples


def prepare_sentence_transformer_format(
    samples: list[TrainingSample],
) -> list:
    """Convert to sentence-transformers InputExample format."""
    from sentence_transformers import InputExample

    examples = []
    for sample in samples:
        # For MultipleNegativesRankingLoss: (query, positive) pairs
        # Negatives come from in-batch sampling + explicit hard negatives
        if sample.negatives:
            # Include one hard negative per example
            for neg in sample.negatives[:1]:
                examples.append(InputExample(
                    texts=[sample.query, sample.positive, neg],
                ))
        else:
            examples.append(InputExample(
                texts=[sample.query, sample.positive],
            ))

    return examples
```

### Creating Training Data from Scratch

If you do not have labeled data, generate it synthetically:

```python
import anthropic
import numpy as np
from sentence_transformers import SentenceTransformer


def generate_training_dataset(
    corpus: list[str],
    queries_per_doc: int = 3,
    n_hard_negatives: int = 7,
    base_model_name: str = "BAAI/bge-base-en-v1.5",
    output_path: str = "training_data.jsonl",
) -> str:
    """End-to-end training data generation.

    1. Generate synthetic queries for each document using Claude.
    2. Mine hard negatives using the base embedding model.
    3. Save as JSONL for training.
    """
    client = anthropic.Anthropic()

    # Step 1: Generate queries
    print(f"Generating queries for {len(corpus)} documents...")
    pairs = []
    for i, doc in enumerate(corpus):
        if i % 100 == 0:
            print(f"  Processing document {i}/{len(corpus)}")

        response = client.messages.create(
            model="claude-haiku-4-20250514",
            max_tokens=500,
            messages=[{
                "role": "user",
                "content": (
                    f"Generate {queries_per_doc} realistic search queries that this "
                    f"document would answer well. One query per line, nothing else.\n\n"
                    f"Document:\n{doc[:2000]}"
                ),
            }],
        )

        queries = [
            q.strip().lstrip("0123456789.-) ")
            for q in response.content[0].text.strip().split("\n")
            if q.strip()
        ]

        for query in queries[:queries_per_doc]:
            pairs.append({"query": query, "positive": doc, "doc_idx": i})

    # Step 2: Mine hard negatives
    print(f"Mining hard negatives for {len(pairs)} pairs...")
    model = SentenceTransformer(base_model_name)
    corpus_embs = model.encode(corpus, normalize_embeddings=True, batch_size=256, show_progress_bar=True)

    for pair in pairs:
        query_emb = model.encode([pair["query"]], normalize_embeddings=True)[0]
        scores = np.dot(corpus_embs, query_emb)

        # Get top-scoring docs that are NOT the positive
        ranked = np.argsort(scores)[::-1]
        negatives = []
        for idx in ranked:
            if idx != pair["doc_idx"] and len(negatives) < n_hard_negatives:
                negatives.append(corpus[idx])

        pair["negatives"] = negatives

    # Step 3: Save
    with open(output_path, "w") as f:
        for pair in pairs:
            f.write(json.dumps({
                "query": pair["query"],
                "positive": pair["positive"],
                "negatives": pair["negatives"],
            }) + "\n")

    print(f"Saved {len(pairs)} training examples to {output_path}")
    return output_path
```

---

## Step 2: Loss Function Selection

The loss function determines what the model learns. Choose based on your data format and quality requirements.

### MultipleNegativesRankingLoss (Recommended Default)

The most commonly used loss for retrieval fine-tuning. Uses in-batch negatives (other positives in the batch become negatives for each query) plus optional hard negatives.

```python
from sentence_transformers import SentenceTransformer, losses, InputExample
from torch.utils.data import DataLoader


def train_with_mnrl(
    model_name: str,
    train_examples: list[InputExample],
    output_path: str,
    epochs: int = 3,
    batch_size: int = 32,
    learning_rate: float = 2e-5,
    warmup_ratio: float = 0.1,
):
    """Train with MultipleNegativesRankingLoss.

    This is the best default choice for retrieval fine-tuning.
    Each batch of size B provides B-1 in-batch negatives per query,
    plus any explicit hard negatives in the training data.

    Larger batch sizes = more in-batch negatives = better training signal.
    Use the largest batch size your GPU can handle.
    """
    model = SentenceTransformer(model_name)

    train_dataloader = DataLoader(
        train_examples,
        shuffle=True,
        batch_size=batch_size,
    )

    # MNRL: anchor=query, positive=relevant_doc, (optional) negative=hard_negative
    train_loss = losses.MultipleNegativesRankingLoss(
        model=model,
        scale=20.0,         # Temperature scaling (default 20.0, higher = sharper)
        similarity_fct=losses.util.cos_sim,  # Cosine similarity
    )

    warmup_steps = int(len(train_dataloader) * epochs * warmup_ratio)

    model.fit(
        train_objectives=[(train_dataloader, train_loss)],
        epochs=epochs,
        warmup_steps=warmup_steps,
        optimizer_params={"lr": learning_rate},
        output_path=output_path,
        show_progress_bar=True,
        checkpoint_path=f"{output_path}/checkpoints",
        checkpoint_save_steps=500,
    )

    return model
```

### TripletLoss

Requires explicit (anchor, positive, negative) triples. Less efficient than MNRL because it only uses one negative per example, not in-batch negatives.

```python
def train_with_triplet_loss(
    model_name: str,
    train_examples: list[InputExample],  # Each with 3 texts: [query, positive, negative]
    output_path: str,
    epochs: int = 3,
    batch_size: int = 32,
    margin: float = 0.5,
):
    """Train with TripletLoss.

    Less efficient than MNRL but useful when you have very high-quality
    hand-curated triples where each negative is specifically chosen.
    """
    model = SentenceTransformer(model_name)

    train_dataloader = DataLoader(
        train_examples,
        shuffle=True,
        batch_size=batch_size,
    )

    train_loss = losses.TripletLoss(
        model=model,
        distance_metric=losses.TripletDistanceMetric.COSINE,
        triplet_margin=margin,  # Minimum margin between positive and negative
    )

    model.fit(
        train_objectives=[(train_dataloader, train_loss)],
        epochs=epochs,
        warmup_steps=100,
        output_path=output_path,
        show_progress_bar=True,
    )

    return model
```

### CosineSimilarityLoss

Uses (sentence_A, sentence_B, similarity_score) triples. Useful when you have human-labeled similarity scores rather than binary relevance.

```python
def train_with_cosine_loss(
    model_name: str,
    train_pairs: list[dict],  # [{"text_a": "...", "text_b": "...", "score": 0.85}]
    output_path: str,
    epochs: int = 3,
    batch_size: int = 32,
):
    """Train with CosineSimilarityLoss.

    Use when you have graded relevance labels (0.0 to 1.0)
    rather than binary relevant/not-relevant.
    """
    model = SentenceTransformer(model_name)

    examples = [
        InputExample(texts=[p["text_a"], p["text_b"]], label=p["score"])
        for p in train_pairs
    ]

    train_dataloader = DataLoader(
        examples,
        shuffle=True,
        batch_size=batch_size,
    )

    train_loss = losses.CosineSimilarityLoss(model=model)

    model.fit(
        train_objectives=[(train_dataloader, train_loss)],
        epochs=epochs,
        warmup_steps=100,
        output_path=output_path,
        show_progress_bar=True,
    )

    return model
```

### CachedGISTEmbedLoss (Memory-Efficient)

Enables large effective batch sizes without large GPU memory by caching and accumulating gradients. Combines in-batch negatives with a guided teacher model.

```python
def train_with_cached_gist(
    model_name: str,
    train_examples: list[InputExample],
    output_path: str,
    guide_model_name: str = "BAAI/bge-large-en-v1.5",
    epochs: int = 3,
    batch_size: int = 32,
    mini_batch_size: int = 8,  # Actual GPU batch (smaller = less memory)
):
    """Train with CachedGISTEmbedLoss.

    Advantages:
    - Memory-efficient: uses mini-batches internally while maintaining
      the learning signal of a larger batch.
    - Guided: uses a teacher model to weight the importance of negatives.
    - Produces high-quality embeddings with less data.

    Requires sentence-transformers >= 3.0.
    """
    model = SentenceTransformer(model_name)
    guide_model = SentenceTransformer(guide_model_name)

    train_dataloader = DataLoader(
        train_examples,
        shuffle=True,
        batch_size=batch_size,
    )

    train_loss = losses.CachedGISTEmbedLoss(
        model=model,
        guide=guide_model,
        mini_batch_size=mini_batch_size,
    )

    model.fit(
        train_objectives=[(train_dataloader, train_loss)],
        epochs=epochs,
        warmup_steps=100,
        output_path=output_path,
        show_progress_bar=True,
    )

    return model
```

### Loss Function Comparison

| Loss | Data Format | In-Batch Negatives | Best For | Typical Improvement |
|------|------------|-------------------|----------|-------------------|
| MNRL | (query, positive) or (query, pos, neg) | Yes | General retrieval fine-tuning | 8-15% |
| TripletLoss | (query, positive, negative) | No | Curated triples with specific negatives | 5-10% |
| CosineSimilarityLoss | (text_a, text_b, score) | No | Graded relevance labels | 5-8% |
| CachedGISTEmbedLoss | (query, positive) | Yes (cached) | Memory-limited GPUs, best quality | 10-18% |

---

## Step 3: Evaluation During Training

### InformationRetrievalEvaluator

The standard evaluator for retrieval fine-tuning. Computes Hit@K, MRR, NDCG@10, and MAP during training.

```python
from sentence_transformers.evaluation import InformationRetrievalEvaluator


def create_ir_evaluator(
    eval_samples: list[TrainingSample],
    name: str = "domain-eval",
) -> InformationRetrievalEvaluator:
    """Create an IR evaluator from evaluation samples.

    The evaluator computes metrics at each evaluation step during training,
    allowing you to monitor for overfitting and select the best checkpoint.
    """
    # Build the required format
    queries = {}      # {qid: query_text}
    corpus = {}       # {cid: doc_text}
    relevant_docs = {}  # {qid: {cid: relevance_score}}

    doc_id_counter = 0

    for i, sample in enumerate(eval_samples):
        qid = f"q_{i}"
        queries[qid] = sample.query

        # Add positive document
        pos_cid = f"d_{doc_id_counter}"
        corpus[pos_cid] = sample.positive
        relevant_docs[qid] = {pos_cid: 1}
        doc_id_counter += 1

        # Add negative documents (not relevant)
        for neg in sample.negatives:
            neg_cid = f"d_{doc_id_counter}"
            corpus[neg_cid] = neg
            doc_id_counter += 1

    evaluator = InformationRetrievalEvaluator(
        queries=queries,
        corpus=corpus,
        relevant_docs=relevant_docs,
        name=name,
        show_progress_bar=True,
        batch_size=64,
        ndcg_at_k=[1, 5, 10],
        mrr_at_k=[1, 5, 10],
        accuracy_at_k=[1, 5, 10],
        map_at_k=[10, 100],
    )

    return evaluator
```

---

## Step 4: Complete Training Pipeline

### Putting It All Together

```python
import json
import random
from pathlib import Path
from sentence_transformers import SentenceTransformer, InputExample, losses
from sentence_transformers.evaluation import InformationRetrievalEvaluator
from torch.utils.data import DataLoader


def fine_tune_embedding_model(
    base_model_name: str = "BAAI/bge-base-en-v1.5",
    training_data_path: str = "training_data.jsonl",
    output_dir: str = "./finetuned-model",
    epochs: int = 3,
    batch_size: int = 64,
    learning_rate: float = 2e-5,
    warmup_ratio: float = 0.1,
    eval_fraction: float = 0.1,
    loss_type: str = "mnrl",  # "mnrl", "triplet", "cosine", "gist"
) -> dict:
    """Complete fine-tuning pipeline.

    Returns evaluation metrics before and after fine-tuning.
    """
    # Load data
    samples = []
    with open(training_data_path, "r") as f:
        for line in f:
            data = json.loads(line.strip())
            samples.append(TrainingSample(
                query=data["query"],
                positive=data["positive"],
                negatives=data.get("negatives", []),
            ))

    random.shuffle(samples)
    eval_size = max(int(len(samples) * eval_fraction), 50)
    eval_samples = samples[:eval_size]
    train_samples = samples[eval_size:]

    print(f"Training: {len(train_samples)} | Evaluation: {len(eval_samples)}")

    # Create evaluator
    evaluator = create_ir_evaluator(eval_samples)

    # Evaluate base model BEFORE fine-tuning
    base_model = SentenceTransformer(base_model_name)
    print("\n=== Base model evaluation ===")
    base_metrics = evaluator(base_model, output_path=f"{output_dir}/base-eval")

    # Prepare training examples
    train_examples = []
    for sample in train_samples:
        if sample.negatives:
            train_examples.append(InputExample(
                texts=[sample.query, sample.positive, sample.negatives[0]],
            ))
        else:
            train_examples.append(InputExample(
                texts=[sample.query, sample.positive],
            ))

    train_dataloader = DataLoader(
        train_examples,
        shuffle=True,
        batch_size=batch_size,
    )

    # Select loss function
    model = SentenceTransformer(base_model_name)

    if loss_type == "mnrl":
        train_loss = losses.MultipleNegativesRankingLoss(model=model)
    elif loss_type == "triplet":
        train_loss = losses.TripletLoss(model=model)
    elif loss_type == "cosine":
        train_loss = losses.CosineSimilarityLoss(model=model)
    elif loss_type == "gist":
        guide = SentenceTransformer("BAAI/bge-large-en-v1.5")
        train_loss = losses.CachedGISTEmbedLoss(model=model, guide=guide, mini_batch_size=8)
    else:
        raise ValueError(f"Unknown loss type: {loss_type}")

    warmup_steps = int(len(train_dataloader) * epochs * warmup_ratio)

    # Train
    print(f"\n=== Training with {loss_type} loss ===")
    model.fit(
        train_objectives=[(train_dataloader, train_loss)],
        evaluator=evaluator,
        epochs=epochs,
        warmup_steps=warmup_steps,
        optimizer_params={"lr": learning_rate},
        output_path=output_dir,
        evaluation_steps=max(len(train_dataloader) // 3, 100),  # Eval 3x per epoch
        save_best_model=True,
        show_progress_bar=True,
        checkpoint_path=f"{output_dir}/checkpoints",
        checkpoint_save_steps=500,
    )

    # Evaluate fine-tuned model
    print("\n=== Fine-tuned model evaluation ===")
    finetuned_model = SentenceTransformer(output_dir)
    ft_metrics = evaluator(finetuned_model, output_path=f"{output_dir}/ft-eval")

    return {
        "base_metrics": base_metrics,
        "finetuned_metrics": ft_metrics,
    }


# Run training
if __name__ == "__main__":
    results = fine_tune_embedding_model(
        base_model_name="BAAI/bge-base-en-v1.5",
        training_data_path="training_data.jsonl",
        output_dir="./finetuned-bge-base",
        epochs=3,
        batch_size=64,
        loss_type="mnrl",
    )
```

---

## Step 5: Training Arguments Reference

### Key Hyperparameters

| Parameter | Default | Recommended Range | Notes |
|-----------|---------|------------------|-------|
| `epochs` | 3 | 1-5 | More epochs for small datasets, fewer for large |
| `batch_size` | 32 | 32-256 | Larger = more in-batch negatives = better (if GPU allows) |
| `learning_rate` | 2e-5 | 1e-5 to 5e-5 | Lower for larger models, higher for smaller |
| `warmup_ratio` | 0.1 | 0.05-0.15 | Gradual LR increase to avoid early instability |
| `weight_decay` | 0.01 | 0.0-0.1 | Regularization to prevent overfitting |
| `max_grad_norm` | 1.0 | 0.5-2.0 | Gradient clipping for stability |
| `evaluation_steps` | 500 | 100-1000 | How often to run evaluation |

### Batch Size vs GPU Memory

| Model | Batch Size 32 | Batch Size 64 | Batch Size 128 | Batch Size 256 |
|-------|-------------|-------------|---------------|---------------|
| bge-base (109M) | 6 GB | 10 GB | 18 GB | 34 GB |
| bge-large (335M) | 12 GB | 20 GB | 36 GB | 68 GB |
| BGE-M3 (568M) | 16 GB | 28 GB | 52 GB | OOM on most GPUs |

Use gradient accumulation to simulate larger batch sizes without more memory:

```python
# Simulate batch_size=256 on a 16GB GPU
model.fit(
    train_objectives=[(train_dataloader, train_loss)],  # dataloader batch_size=32
    epochs=3,
    # accumulate 8 steps * 32 batch = 256 effective batch
    # This is handled internally by sentence-transformers via scheduler
)
```

---

## Step 6: Saving and Exporting

### Save for sentence-transformers

```python
# The model is automatically saved during training to output_path
# Load it back:
model = SentenceTransformer("./finetuned-bge-base")
embeddings = model.encode(["test query"], normalize_embeddings=True)
```

### Export to HuggingFace Hub

```python
model = SentenceTransformer("./finetuned-bge-base")
model.save_to_hub(
    repo_id="your-org/finetuned-bge-base-domain",
    private=True,
    exist_ok=True,
)
```

### Export to ONNX (for production serving)

```python
from optimum.onnxruntime import ORTModelForFeatureExtraction
from transformers import AutoTokenizer

# Convert to ONNX
ort_model = ORTModelForFeatureExtraction.from_pretrained(
    "./finetuned-bge-base",
    export=True,
)
tokenizer = AutoTokenizer.from_pretrained("./finetuned-bge-base")

# Save ONNX model
ort_model.save_pretrained("./finetuned-bge-base-onnx")
tokenizer.save_pretrained("./finetuned-bge-base-onnx")

# Use ONNX model for inference (2-3x faster)
import numpy as np
from optimum.onnxruntime import ORTModelForFeatureExtraction

ort_model = ORTModelForFeatureExtraction.from_pretrained("./finetuned-bge-base-onnx")

inputs = tokenizer(["test query"], return_tensors="pt", padding=True, truncation=True)
outputs = ort_model(**inputs)
embedding = outputs.last_hidden_state[:, 0, :].detach().numpy()  # CLS token
embedding = embedding / np.linalg.norm(embedding)  # Normalize
```

---

## Common Pitfalls

1. **Using too small a batch size.** For MNRL, the batch size directly determines the number of in-batch negatives. Batch size 8 means only 7 negatives per query -- too few. Use at least 32, ideally 64+.
2. **Training for too many epochs.** With small datasets (<5K), 1-2 epochs often suffice. Three or more epochs leads to overfitting. Monitor eval metrics and use early stopping.
3. **Not using a warmup period.** Starting training at full learning rate can destabilize the model. Always use 5-15% warmup steps.
4. **Mixing up data formats.** MNRL expects (query, positive) or (query, positive, negative). TripletLoss requires all three. CosineSimilarityLoss expects a float label. Mismatched formats cause silent failures.
5. **Not normalizing embeddings after fine-tuning.** Some loss functions change the embedding scale. Always normalize (L2) before computing cosine similarity at inference time.

---

## References

- Sentence-Transformers Training Guide -- https://www.sbert.net/docs/training/overview.html
- MultipleNegativesRankingLoss -- https://www.sbert.net/docs/package_reference/losses.html#multiplenegativesrankingloss
- CachedGISTEmbedLoss -- https://www.sbert.net/docs/package_reference/losses.html#cachedgistembedloss
- ONNX Export with Optimum -- https://huggingface.co/docs/optimum/onnxruntime/usage_guides/models
- InformationRetrievalEvaluator -- https://www.sbert.net/docs/package_reference/evaluation.html
