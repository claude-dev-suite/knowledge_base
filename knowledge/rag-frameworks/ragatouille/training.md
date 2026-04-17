# RAGatouille -- Training: Custom Model Fine-Tuning with Domain Data

## Overview

While pretrained ColBERT models (like `colbertv2.0`) work well for general-purpose retrieval, fine-tuning on domain-specific data can improve retrieval quality by 10-25% on in-domain queries. RAGatouille provides a training API that wraps ColBERT's training pipeline, making it accessible without deep knowledge of the underlying infrastructure.

This guide covers preparing training data, configuring training runs, evaluating fine-tuned models, and deploying them in production RAG pipelines.

---

## When to Fine-Tune

Fine-tuning is worth the effort when:

1. **Domain-specific vocabulary**: Medical, legal, or scientific text where pretrained models underperform on specialized terms
2. **Non-English or multilingual content**: The base `colbertv2.0` is English-only
3. **Specific retrieval patterns**: Your queries follow patterns not well-represented in the pretraining data (e.g., code search, structured data queries)
4. **High-stakes applications**: Where a 10-15% improvement in recall justifies the training investment

Fine-tuning is NOT needed when:

- Your domain is well-covered by web text (general knowledge, news, common technical topics)
- You have fewer than 100 query-document pairs
- You need the model running in under a week (fine-tuning takes experimentation)

---

## Training Data Preparation

### Data Format

ColBERT training requires query-positive-negative triplets:

```python
# Each training example is a tuple: (query, positive_passage, negative_passage)
# Or: (query, positive_passage) with negatives mined automatically

training_data = [
    {
        "query": "What are the side effects of metformin?",
        "positive": "Common side effects of metformin include nausea, diarrhea, "
                    "and stomach pain. Less common effects include vitamin B12 "
                    "deficiency with long-term use.",
        "negative": "Metformin is a first-line medication for type 2 diabetes. "
                    "It works by decreasing glucose production in the liver.",
    },
    {
        "query": "How does metformin reduce blood sugar?",
        "positive": "Metformin reduces blood glucose by decreasing hepatic glucose "
                    "production, reducing intestinal absorption of glucose, and "
                    "improving insulin sensitivity.",
        "negative": "Common side effects of metformin include nausea, diarrhea, "
                    "and stomach pain.",
    },
]
```

### Creating Training Data from Existing QA Pairs

```python
import json
from pathlib import Path


def create_training_data_from_qa(
    qa_pairs: list[dict],       # [{"question": "...", "answer": "...", "context": "..."}]
    corpus: list[str],          # all documents in the collection
    negatives_per_query: int = 3,
) -> list[tuple[str, str, str]]:
    """
    Convert QA pairs into ColBERT training triplets.
    Uses the context/answer as positive, random corpus docs as negatives.
    """
    import random

    triplets = []
    for qa in qa_pairs:
        query = qa["question"]
        positive = qa.get("context", qa["answer"])

        # Sample hard negatives (random documents that are NOT the answer)
        negatives = random.sample(
            [doc for doc in corpus if doc != positive],
            min(negatives_per_query, len(corpus) - 1),
        )

        for negative in negatives:
            triplets.append((query, positive, negative))

    return triplets


# Example
qa_pairs = [
    {
        "question": "What are the contraindications for metformin?",
        "context": "Metformin is contraindicated in patients with severe renal "
                   "impairment (eGFR below 30 mL/min), metabolic acidosis, and "
                   "diabetic ketoacidosis.",
    },
    {
        "question": "What is the recommended starting dose of metformin?",
        "context": "The recommended starting dose is 500 mg twice daily or 850 mg "
                   "once daily with meals. Dose may be increased by 500 mg weekly.",
    },
]

corpus = [
    "Metformin is a first-line medication for type 2 diabetes.",
    "Common side effects of metformin include gastrointestinal symptoms.",
    "Metformin is contraindicated in severe renal impairment.",
    "The recommended starting dose is 500 mg twice daily.",
    "Insulin resistance is a hallmark of type 2 diabetes.",
    "Blood glucose monitoring is essential for diabetes management.",
]

triplets = create_training_data_from_qa(qa_pairs, corpus, negatives_per_query=2)
print(f"Created {len(triplets)} training triplets")
```

### Mining Hard Negatives with BM25

Better negatives produce better models. BM25-mined negatives are passages that match keywords but are not relevant:

```python
import rank_bm25
import random


def mine_hard_negatives_bm25(
    queries: list[str],
    positives: list[str],
    corpus: list[str],
    negatives_per_query: int = 5,
    top_k_candidates: int = 20,
) -> list[tuple[str, str, str]]:
    """
    Mine hard negatives using BM25.
    For each query, retrieve top-K BM25 results and use non-positive ones as negatives.
    """
    # Build BM25 index
    tokenized_corpus = [doc.lower().split() for doc in corpus]
    bm25 = rank_bm25.BM25Okapi(tokenized_corpus)

    triplets = []
    for query, positive in zip(queries, positives):
        # BM25 retrieval
        tokenized_query = query.lower().split()
        scores = bm25.get_scores(tokenized_query)

        # Sort by score (descending)
        ranked_indices = sorted(
            range(len(corpus)), key=lambda i: scores[i], reverse=True
        )

        # Select hard negatives (high BM25 score but not the positive)
        negatives = []
        for idx in ranked_indices[:top_k_candidates]:
            if corpus[idx] != positive and len(negatives) < negatives_per_query:
                negatives.append(corpus[idx])

        # If not enough hard negatives, add random ones
        while len(negatives) < negatives_per_query:
            rand_doc = random.choice(corpus)
            if rand_doc != positive and rand_doc not in negatives:
                negatives.append(rand_doc)

        for neg in negatives:
            triplets.append((query, positive, neg))

    return triplets


queries = [qa["question"] for qa in qa_pairs]
positives = [qa["context"] for qa in qa_pairs]

hard_triplets = mine_hard_negatives_bm25(
    queries, positives, corpus, negatives_per_query=3
)
print(f"Mined {len(hard_triplets)} hard negative triplets")
```

### Saving Training Data

```python
import json


def save_training_data(
    triplets: list[tuple[str, str, str]],
    output_path: str,
):
    """Save training triplets in JSONL format."""
    with open(output_path, "w") as f:
        for query, positive, negative in triplets:
            json.dump({
                "query": query,
                "positive": positive,
                "negative": negative,
            }, f)
            f.write("\n")

    print(f"Saved {len(triplets)} triplets to {output_path}")


def load_training_data(input_path: str) -> list[tuple[str, str, str]]:
    """Load training triplets from JSONL format."""
    triplets = []
    with open(input_path) as f:
        for line in f:
            data = json.loads(line)
            triplets.append((data["query"], data["positive"], data["negative"]))
    return triplets


save_training_data(hard_triplets, "training_data.jsonl")
```

---

## Training a Custom Model

### Basic Fine-Tuning

```python
from ragatouille import RAGTrainer

# Initialize the trainer
trainer = RAGTrainer(
    model_name="my-domain-colbert",
    pretrained_model_name="colbert-ir/colbertv2.0",  # base model to fine-tune
    language_code="en",
)

# Prepare training pairs
# RAGTrainer expects (query, positive, negative) or (query, positive)
training_pairs = [
    ("What are metformin side effects?",
     "Common side effects include nausea, diarrhea, and stomach pain.",
     "Metformin is a biguanide medication used for type 2 diabetes."),
    ("How does metformin work?",
     "Metformin reduces hepatic glucose production and improves insulin sensitivity.",
     "The drug was first approved by the FDA in 1995."),
    # ... add hundreds to thousands of examples
]

# Prepare the data (creates ColBERT-format training files)
trainer.prepare_training_data(
    raw_data=training_pairs,
    all_documents=corpus,           # full corpus for additional negative mining
    data_out_path="./training_data/",
    num_new_negatives=10,           # mine additional negatives per query
    mine_hard_negatives=True,       # use the base model for hard negative mining
)
```

### Running the Training

```python
# Train the model
model_path = trainer.train(
    batch_size=32,              # batch size (reduce if GPU OOM)
    nbits=2,                    # quantization bits for PLAID (2 or 4)
    maxsteps=500_000,           # max training steps
    use_ib_negatives=True,      # use in-batch negatives (recommended)
    dim=128,                    # embedding dimension
    learning_rate=5e-6,         # learning rate
    doc_maxlen=256,             # max document length in tokens
    query_maxlen=32,            # max query length in tokens
    warmup_steps=5000,          # LR warmup steps
    accumsteps=1,               # gradient accumulation steps
)

print(f"Model saved to: {model_path}")
```

### Training Parameters Guide

| Parameter | Default | Range | Effect |
|-----------|---------|-------|--------|
| `batch_size` | 32 | 8-128 | Larger = faster, needs more GPU RAM |
| `nbits` | 2 | 1, 2, 4 | Lower = smaller index, slightly lower quality |
| `maxsteps` | 500000 | 10000-1000000 | More steps = better convergence on large data |
| `learning_rate` | 5e-6 | 1e-6 to 1e-5 | Higher = faster learning, risk of instability |
| `dim` | 128 | 32-256 | Higher = better quality, larger index |
| `doc_maxlen` | 256 | 128-512 | Match your typical document length |
| `query_maxlen` | 32 | 16-64 | Match your typical query length |
| `warmup_steps` | 5000 | 1000-10000 | Stabilizes early training |
| `use_ib_negatives` | True | True/False | Almost always True for better training |

### Training on Multiple GPUs

```python
# For multi-GPU training, set environment variables before running
import os
os.environ["CUDA_VISIBLE_DEVICES"] = "0,1,2,3"

# RAGTrainer automatically uses DataParallel for multi-GPU
trainer = RAGTrainer(
    model_name="my-domain-colbert-multigpu",
    pretrained_model_name="colbert-ir/colbertv2.0",
)

# Increase batch size proportionally to GPU count
model_path = trainer.train(
    batch_size=128,  # 32 per GPU x 4 GPUs
    maxsteps=200_000,
    use_ib_negatives=True,
)
```

---

## Evaluating the Fine-Tuned Model

### Building an Evaluation Dataset

```python
def create_eval_dataset(
    qa_pairs: list[dict],
    k: int = 10,
) -> tuple[list[str], list[list[str]]]:
    """
    Create an evaluation dataset from QA pairs.
    Returns (queries, relevant_doc_ids) for retrieval evaluation.
    """
    queries = [qa["question"] for qa in qa_pairs]
    relevant_docs = [[qa["context"]] for qa in qa_pairs]
    return queries, relevant_docs


eval_queries, eval_relevant = create_eval_dataset(eval_qa_pairs)
```

### Comparing Base vs. Fine-Tuned Models

```python
from ragatouille import RAGPretrainedModel
import numpy as np


def evaluate_retrieval(
    rag: RAGPretrainedModel,
    queries: list[str],
    relevant_docs: list[list[str]],
    k_values: list[int] = [1, 5, 10],
) -> dict:
    """Compute retrieval metrics: MRR, Recall@K, NDCG@K."""
    results = {f"recall@{k}": [] for k in k_values}
    results["mrr"] = []

    max_k = max(k_values)

    for query, relevant in zip(queries, relevant_docs):
        search_results = rag.search(query=query, k=max_k)
        retrieved_texts = [r["content"] for r in search_results]

        # MRR
        rr = 0.0
        for rank, text in enumerate(retrieved_texts, 1):
            if any(rel in text or text in rel for rel in relevant):
                rr = 1.0 / rank
                break
        results["mrr"].append(rr)

        # Recall@K
        for k in k_values:
            top_k_texts = retrieved_texts[:k]
            found = sum(
                1 for rel in relevant
                if any(rel in t or t in rel for t in top_k_texts)
            )
            results[f"recall@{k}"].append(found / len(relevant))

    # Average metrics
    return {metric: np.mean(scores) for metric, scores in results.items()}


# Evaluate base model
base_rag = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")
base_rag.index(index_name="eval_base", collection=corpus)

base_metrics = evaluate_retrieval(base_rag, eval_queries, eval_relevant)
print("Base model:")
for metric, score in base_metrics.items():
    print(f"  {metric}: {score:.4f}")

# Evaluate fine-tuned model
finetuned_rag = RAGPretrainedModel.from_pretrained("./models/my-domain-colbert/")
finetuned_rag.index(index_name="eval_finetuned", collection=corpus)

finetuned_metrics = evaluate_retrieval(finetuned_rag, eval_queries, eval_relevant)
print("\nFine-tuned model:")
for metric, score in finetuned_metrics.items():
    print(f"  {metric}: {score:.4f}")

# Comparison
print("\nImprovement:")
for metric in base_metrics:
    delta = finetuned_metrics[metric] - base_metrics[metric]
    print(f"  {metric}: {delta:+.4f} ({delta/base_metrics[metric]*100:+.1f}%)")
```

### Per-Query Analysis

```python
def analyze_query_results(
    base_rag: RAGPretrainedModel,
    finetuned_rag: RAGPretrainedModel,
    query: str,
    k: int = 5,
):
    """Compare retrieval results between base and fine-tuned models."""
    base_results = base_rag.search(query=query, k=k)
    finetuned_results = finetuned_rag.search(query=query, k=k)

    print(f"Query: {query}\n")

    print("Base model results:")
    for i, r in enumerate(base_results):
        print(f"  [{i+1}] ({r['score']:.2f}) {r['content'][:100]}...")

    print("\nFine-tuned model results:")
    for i, r in enumerate(finetuned_results):
        print(f"  [{i+1}] ({r['score']:.2f}) {r['content'][:100]}...")

    # Find differences
    base_texts = set(r["content"][:100] for r in base_results)
    ft_texts = set(r["content"][:100] for r in finetuned_results)

    new_results = ft_texts - base_texts
    lost_results = base_texts - ft_texts

    if new_results:
        print(f"\nNew in fine-tuned: {len(new_results)} results")
    if lost_results:
        print(f"Lost from base: {len(lost_results)} results")
```

---

## Advanced Training Techniques

### Knowledge Distillation from Cross-Encoders

Train ColBERT using a cross-encoder teacher model for better training signal:

```python
from sentence_transformers import CrossEncoder
import json


def create_distillation_data(
    queries: list[str],
    corpus: list[str],
    cross_encoder_model: str = "cross-encoder/ms-marco-MiniLM-L-12-v2",
    top_k: int = 50,
    output_path: str = "distillation_data.jsonl",
):
    """
    Use a cross-encoder to score query-document pairs,
    then create training data with soft labels.
    """
    ce = CrossEncoder(cross_encoder_model)

    with open(output_path, "w") as f:
        for query in queries:
            # Score all documents with cross-encoder
            pairs = [(query, doc) for doc in corpus]
            scores = ce.predict(pairs, batch_size=64)

            # Sort by score
            ranked = sorted(
                zip(corpus, scores), key=lambda x: x[1], reverse=True
            )

            # Top document as positive, lower-ranked as negatives
            positive = ranked[0][0]
            for neg_doc, neg_score in ranked[5:15]:  # medium-hard negatives
                json.dump({
                    "query": query,
                    "positive": positive,
                    "negative": neg_doc,
                    "pos_score": float(ranked[0][1]),
                    "neg_score": float(neg_score),
                }, f)
                f.write("\n")

    print(f"Created distillation data at {output_path}")
```

### Progressive Training

Start with easy negatives, then move to harder ones:

```python
from ragatouille import RAGTrainer


def progressive_training(
    trainer: RAGTrainer,
    easy_data: list[tuple],
    hard_data: list[tuple],
    corpus: list[str],
):
    """
    Stage 1: Train on easy (random) negatives to learn basic relevance.
    Stage 2: Continue training on hard (BM25-mined) negatives for fine-grained discrimination.
    """
    # Stage 1: Easy negatives
    print("Stage 1: Training with easy negatives...")
    trainer.prepare_training_data(
        raw_data=easy_data,
        all_documents=corpus,
        data_out_path="./training_data_stage1/",
        num_new_negatives=5,
        mine_hard_negatives=False,  # random negatives only
    )

    model_path_stage1 = trainer.train(
        batch_size=32,
        maxsteps=100_000,
        learning_rate=1e-5,     # higher LR for initial learning
        warmup_steps=5000,
    )

    # Stage 2: Hard negatives (from stage 1 model)
    print("Stage 2: Training with hard negatives...")
    trainer_stage2 = RAGTrainer(
        model_name="my-model-stage2",
        pretrained_model_name=model_path_stage1,  # start from stage 1
    )

    trainer_stage2.prepare_training_data(
        raw_data=hard_data,
        all_documents=corpus,
        data_out_path="./training_data_stage2/",
        num_new_negatives=10,
        mine_hard_negatives=True,
    )

    model_path_final = trainer_stage2.train(
        batch_size=32,
        maxsteps=50_000,
        learning_rate=2e-6,     # lower LR for fine-tuning
        warmup_steps=2000,
    )

    return model_path_final
```

---

## Deploying Fine-Tuned Models

### Index with Custom Model

```python
from ragatouille import RAGPretrainedModel

# Load your fine-tuned model
rag = RAGPretrainedModel.from_pretrained("./models/my-domain-colbert/")

# Build production index
index_path = rag.index(
    index_name="production_v1",
    collection=production_documents,
    split_documents=True,
    max_document_length=256,
)

# Package for deployment
import shutil
deployment_path = "/opt/app/model"
shutil.copytree("./models/my-domain-colbert/", f"{deployment_path}/model/")
shutil.copytree(index_path, f"{deployment_path}/index/")
```

### Docker Deployment

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN pip install ragatouille fastapi uvicorn

# Copy model and index
COPY model/ /app/model/
COPY index/ /app/index/
COPY server.py /app/

ENV MODEL_PATH=/app/model
ENV INDEX_PATH=/app/index

EXPOSE 8000
CMD ["uvicorn", "server:app", "--host", "0.0.0.0", "--port", "8000"]
```

```python
# server.py for Docker
import os
from fastapi import FastAPI
from pydantic import BaseModel
from ragatouille import RAGPretrainedModel

app = FastAPI()

INDEX_PATH = os.environ.get("INDEX_PATH", "./index")
rag = RAGPretrainedModel.from_index(INDEX_PATH)


class SearchRequest(BaseModel):
    query: str
    k: int = 5


@app.post("/search")
async def search(request: SearchRequest):
    results = rag.search(query=request.query, k=request.k)
    return {"results": results}


@app.get("/health")
async def health():
    return {"status": "ok"}
```

---

## Common Pitfalls

1. **Too few training examples**: ColBERT fine-tuning needs at least 500-1000 query-positive pairs to see meaningful improvement. With fewer than 100, the model may overfit or not improve at all.

2. **Easy negatives only**: Random negatives teach the model obvious distinctions but not fine-grained relevance. Always include BM25-mined hard negatives for the best results.

3. **Not evaluating on held-out data**: Always keep 10-20% of your QA pairs for evaluation. Training metrics alone do not reflect real-world retrieval quality.

4. **Forgetting to rebuild the index**: After fine-tuning, you must rebuild the PLAID index with the new model. The old index is incompatible with the new embeddings.

5. **Training too long**: ColBERT fine-tuning converges relatively quickly. Monitor the training loss and stop early if it plateaus. Overtraining degrades generalization.

6. **Mismatched document/query lengths**: Set `doc_maxlen` and `query_maxlen` to match your actual data. Too short truncates content; too long wastes computation on padding.

---

## References

- RAGatouille training: https://github.com/AnswerDotAI/RAGatouille#training
- ColBERT training: https://github.com/stanford-futuredata/ColBERT#training
- "ColBERTv2: Effective and Efficient Retrieval via Lightweight Late Interaction"
- Hard negative mining: "Approximate Nearest Neighbor Negative Contrastive Learning for Dense Text Retrieval"
