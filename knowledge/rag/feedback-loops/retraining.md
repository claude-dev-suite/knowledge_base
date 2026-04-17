# Feedback Loops -- Retraining: Embedding Fine-Tuning and Reranker Adaptation from Feedback

## TL;DR

Collected feedback becomes valuable when it flows back into the RAG pipeline as training data. This article covers two primary retraining strategies: (1) embedding model fine-tuning using feedback-derived query-document pairs to improve retrieval relevance, and (2) reranker adaptation using positive/negative document labels to improve ranking quality. Both approaches use contrastive learning where positive feedback creates positive pairs (query, relevant doc) and negative feedback creates hard negatives (query, irrelevant doc that was retrieved).

---

## Feedback to Training Data

```python
from dataclasses import dataclass
import json
from pathlib import Path


@dataclass
class TrainingPair:
    query: str
    positive_doc: str
    negative_docs: list[str]
    score: float


class FeedbackToTrainingConverter:
    """Convert collected feedback into training pairs.

    Strategy:
    - Thumbs up + retrieved docs -> (query, doc) is a positive pair
    - Thumbs down + retrieved docs -> (query, doc) is a hard negative
    - Corrections -> original response is negative, correction is positive
    """

    def __init__(self, doc_store):
        self.doc_store = doc_store

    def convert(self, feedback_records: list[dict]) -> list[TrainingPair]:
        """Convert feedback records into training pairs."""
        # Group by query
        query_feedback: dict[str, list[dict]] = {}
        for record in feedback_records:
            query = record.get("query", "")
            if query:
                query_feedback.setdefault(query, []).append(record)

        pairs = []
        for query, records in query_feedback.items():
            positive_docs = []
            negative_docs = []

            for record in records:
                doc_ids = record.get("doc_ids", [])
                signal = record.get("signal_value", 0)

                if signal > 0:
                    for doc_id in doc_ids:
                        doc_text = self.doc_store.get(doc_id)
                        if doc_text:
                            positive_docs.append(doc_text)
                elif signal < 0:
                    for doc_id in doc_ids:
                        doc_text = self.doc_store.get(doc_id)
                        if doc_text:
                            negative_docs.append(doc_text)

            if positive_docs:
                for pos in positive_docs:
                    pairs.append(TrainingPair(
                        query=query,
                        positive_doc=pos,
                        negative_docs=negative_docs[:5],
                        score=max(r.get("signal_value", 0) for r in records),
                    ))

        return pairs

    def export_for_sentence_transformers(
        self, pairs: list[TrainingPair], output_path: str
    ) -> int:
        """Export as JSONL for sentence-transformers fine-tuning."""
        path = Path(output_path)
        count = 0
        with open(path, "w") as f:
            for pair in pairs:
                # Format: {"query": ..., "pos": [...], "neg": [...]}
                record = {
                    "query": pair.query,
                    "pos": [pair.positive_doc[:2000]],
                    "neg": [d[:2000] for d in pair.negative_docs[:3]],
                }
                f.write(json.dumps(record) + "\n")
                count += 1
        return count
```

---

## Embedding Fine-Tuning

```python
def fine_tune_embedding_model(
    training_data_path: str,
    base_model: str = "BAAI/bge-small-en-v1.5",
    output_dir: str = "./fine_tuned_embeddings",
    epochs: int = 3,
    batch_size: int = 32,
    learning_rate: float = 2e-5,
) -> dict:
    """Fine-tune an embedding model using feedback-derived training pairs.

    Uses sentence-transformers with MultipleNegativesRankingLoss:
    - Positive pairs: (query, relevant_doc) from thumbs-up feedback
    - Hard negatives: docs from thumbs-down feedback
    - In-batch negatives: other queries' positives serve as negatives
    """
    from sentence_transformers import (
        SentenceTransformer,
        InputExample,
        losses,
    )
    from torch.utils.data import DataLoader

    # Load base model
    model = SentenceTransformer(base_model)

    # Load training data
    examples = []
    with open(training_data_path) as f:
        for line in f:
            record = json.loads(line)
            query = record["query"]
            for pos in record.get("pos", []):
                examples.append(InputExample(texts=[query, pos]))
            # Add hard negatives if available
            for neg in record.get("neg", []):
                examples.append(InputExample(texts=[query, neg], label=0.0))

    if not examples:
        return {"status": "error", "message": "No training examples found"}

    # Create dataloader
    dataloader = DataLoader(examples, shuffle=True, batch_size=batch_size)

    # Loss: MultipleNegativesRankingLoss for contrastive learning
    loss = losses.MultipleNegativesRankingLoss(model)

    # Fine-tune
    model.fit(
        train_objectives=[(dataloader, loss)],
        epochs=epochs,
        warmup_steps=int(len(dataloader) * 0.1),
        output_path=output_dir,
        show_progress_bar=True,
    )

    return {
        "status": "success",
        "training_examples": len(examples),
        "epochs": epochs,
        "output_dir": output_dir,
        "base_model": base_model,
    }
```

---

## Reranker Adaptation

```python
def fine_tune_reranker(
    training_data_path: str,
    base_model: str = "cross-encoder/ms-marco-MiniLM-L-6-v2",
    output_dir: str = "./fine_tuned_reranker",
    epochs: int = 2,
    batch_size: int = 16,
) -> dict:
    """Fine-tune a cross-encoder reranker from feedback data.

    Cross-encoders take (query, document) pairs and output a
    relevance score. Fine-tuning with feedback data teaches the
    model domain-specific relevance.

    Training format:
    - Positive pairs (score=1): query + doc from thumbs-up
    - Negative pairs (score=0): query + doc from thumbs-down
    """
    from sentence_transformers import CrossEncoder, InputExample
    from torch.utils.data import DataLoader

    model = CrossEncoder(base_model, num_labels=1)

    examples = []
    with open(training_data_path) as f:
        for line in f:
            record = json.loads(line)
            query = record["query"]
            for pos in record.get("pos", []):
                examples.append(InputExample(texts=[query, pos[:512]], label=1.0))
            for neg in record.get("neg", []):
                examples.append(InputExample(texts=[query, neg[:512]], label=0.0))

    if len(examples) < 50:
        return {"status": "insufficient_data", "count": len(examples), "minimum": 50}

    dataloader = DataLoader(examples, shuffle=True, batch_size=batch_size)

    model.fit(
        train_dataloader=dataloader,
        epochs=epochs,
        warmup_steps=int(len(dataloader) * 0.1),
        output_path=output_dir,
        show_progress_bar=True,
    )

    return {
        "status": "success",
        "training_examples": len(examples),
        "output_dir": output_dir,
    }
```

---

## Evaluation of Retrained Models

```python
def evaluate_retrained_model(
    original_model,
    retrained_model,
    eval_queries: list[dict],
    vector_store,
) -> dict:
    """A/B compare original vs retrained model on held-out feedback data."""
    original_scores = {"relevancy": [], "recall": []}
    retrained_scores = {"relevancy": [], "recall": []}

    for item in eval_queries:
        query = item["query"]
        relevant_ids = set(item.get("relevant_doc_ids", []))

        # Original model retrieval
        orig_emb = original_model.encode(query)
        orig_results = vector_store.query(orig_emb, top_k=10)
        orig_ids = set(r.get("id", "") for r in orig_results)
        orig_recall = len(relevant_ids & orig_ids) / max(len(relevant_ids), 1)
        original_scores["recall"].append(orig_recall)

        # Retrained model retrieval
        new_emb = retrained_model.encode(query)
        new_results = vector_store.query(new_emb, top_k=10)
        new_ids = set(r.get("id", "") for r in new_results)
        new_recall = len(relevant_ids & new_ids) / max(len(relevant_ids), 1)
        retrained_scores["recall"].append(new_recall)

    return {
        "original_avg_recall": sum(original_scores["recall"]) / max(len(original_scores["recall"]), 1),
        "retrained_avg_recall": sum(retrained_scores["recall"]) / max(len(retrained_scores["recall"]), 1),
        "improvement": (
            sum(retrained_scores["recall"]) - sum(original_scores["recall"])
        ) / max(len(original_scores["recall"]), 1),
        "eval_queries": len(eval_queries),
    }
```

---

## Common Pitfalls

1. **Fine-tuning with too little data.** Embedding fine-tuning needs 500+ positive pairs minimum. Reranker fine-tuning needs 200+. Below these thresholds, the model may overfit or not improve.

2. **Not holding out evaluation data.** Use 80/20 split: 80% for training, 20% for evaluation. Never evaluate on training data.

3. **Reindexing the entire corpus after fine-tuning.** After fine-tuning embeddings, all existing vectors are stale (computed with the old model). You must re-embed and reindex the entire corpus.

4. **Fine-tuning too frequently.** Monthly or quarterly is sufficient. More frequent fine-tuning risks overfitting to recent feedback patterns.

5. **Not A/B testing retrained models.** Always compare retrained vs original on held-out data before deploying. Retrained models can regress on queries outside the feedback distribution.

---

## References

- Sentence Transformers fine-tuning: https://www.sbert.net/docs/training/overview.html
- MultipleNegativesRankingLoss: https://www.sbert.net/docs/package_reference/losses.html
- Cross-encoder training: https://www.sbert.net/examples/training/cross-encoder/
