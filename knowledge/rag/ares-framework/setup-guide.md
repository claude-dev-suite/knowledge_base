# ARES Framework -- Setup Guide

## TL;DR

This guide walks through setting up ARES end-to-end: installing the ares-ai package, preparing your corpus, generating synthetic training data with an LLM, fine-tuning the three DeBERTa classifiers, collecting the minimum viable human annotation set, running PPI estimation, and interpreting the results. Every step includes working Python code and expected outputs so you can follow along with your own RAG pipeline.

---

## Installation and Environment Setup

### Prerequisites

| Requirement | Minimum | Recommended |
|---|---|---|
| Python | 3.9 | 3.11 |
| GPU | Not required (but slow) | 1x NVIDIA A10/L4 (24GB) |
| RAM | 16GB | 32GB |
| Disk | 5GB for models | 10GB |
| LLM API key | Any (for synthetic data) | Anthropic or OpenAI |

### Install the ares-ai Package

```bash
# Create a virtual environment
python -m venv ares-env
source ares-env/bin/activate  # Linux/macOS
# ares-env\Scripts\activate   # Windows

# Install core package
pip install ares-ai

# Install additional dependencies for custom workflows
pip install \
    transformers>=4.36.0 \
    datasets>=2.16.0 \
    torch>=2.1.0 \
    sentence-transformers>=2.3.0 \
    anthropic>=0.40.0 \
    scikit-learn>=1.3.0 \
    pandas>=2.1.0
```

### Verify Installation

```python
"""
Verify ARES installation and check available components.
"""
import torch
import transformers

print(f"PyTorch version: {torch.__version__}")
print(f"CUDA available: {torch.cuda.is_available()}")
if torch.cuda.is_available():
    print(f"GPU: {torch.cuda.get_device_name(0)}")
    print(f"GPU memory: {torch.cuda.get_device_properties(0).total_mem / 1e9:.1f} GB")
print(f"Transformers version: {transformers.__version__}")

# Check that DeBERTa tokenizer loads correctly
from transformers import AutoTokenizer
tokenizer = AutoTokenizer.from_pretrained("microsoft/deberta-v3-base")
test_enc = tokenizer("test query", "test document", return_tensors="pt")
print(f"DeBERTa tokenizer OK: {test_enc['input_ids'].shape}")
```

---

## Step 1: Prepare Your Corpus

ARES generates synthetic data from your actual corpus documents. The quality of your evaluation depends directly on having representative passages.

### Extracting Passages from Your Knowledge Base

```python
"""
Extract and prepare corpus passages for ARES synthetic data generation.
"""
import hashlib
import json
import re
from dataclasses import dataclass
from pathlib import Path


@dataclass
class CorpusPassage:
    """A passage extracted from the corpus for synthetic data generation."""
    text: str
    source: str
    doc_id: str
    char_count: int


class CorpusPreparator:
    """
    Extract passages from a document corpus for ARES training.

    Passages should be representative of the actual chunks your RAG
    pipeline retrieves -- same length, same content type.
    """

    def __init__(
        self,
        min_chars: int = 200,
        max_chars: int = 2000,
        target_chars: int = 800,
    ):
        self.min_chars = min_chars
        self.max_chars = max_chars
        self.target_chars = target_chars

    def extract_from_markdown_files(
        self,
        docs_dir: str,
    ) -> list[CorpusPassage]:
        """Extract passages from a directory of markdown files."""
        passages = []

        for md_file in Path(docs_dir).rglob("*.md"):
            content = md_file.read_text(encoding="utf-8")
            file_passages = self._split_into_passages(
                content, source=str(md_file)
            )
            passages.extend(file_passages)

        print(f"Extracted {len(passages)} passages from {docs_dir}")
        print(f"Avg length: {sum(p.char_count for p in passages) / max(len(passages), 1):.0f} chars")
        return passages

    def extract_from_chunks(
        self,
        chunks: list[dict],
        text_field: str = "text",
        source_field: str = "source",
    ) -> list[CorpusPassage]:
        """
        Extract passages from pre-chunked data (e.g., from your vector store).

        This is the preferred approach because it matches the exact chunks
        your RAG pipeline retrieves.
        """
        passages = []
        for chunk in chunks:
            text = chunk[text_field].strip()
            if self.min_chars <= len(text) <= self.max_chars:
                doc_id = hashlib.md5(text.encode()).hexdigest()[:12]
                passages.append(CorpusPassage(
                    text=text,
                    source=chunk.get(source_field, "unknown"),
                    doc_id=doc_id,
                    char_count=len(text),
                ))

        print(f"Accepted {len(passages)}/{len(chunks)} chunks")
        return passages

    def _split_into_passages(
        self,
        content: str,
        source: str,
    ) -> list[CorpusPassage]:
        """Split a document into passages using section boundaries."""
        # Split on markdown headers
        sections = re.split(r'\n#{1,3}\s+', content)
        passages = []

        for section in sections:
            section = section.strip()
            if len(section) < self.min_chars:
                continue
            if len(section) > self.max_chars:
                # Split long sections into paragraphs
                paragraphs = section.split("\n\n")
                current = ""
                for para in paragraphs:
                    if len(current) + len(para) > self.target_chars and current:
                        passages.append(self._make_passage(current, source))
                        current = para
                    else:
                        current = current + "\n\n" + para if current else para
                if current and len(current) >= self.min_chars:
                    passages.append(self._make_passage(current, source))
            else:
                passages.append(self._make_passage(section, source))

        return passages

    def _make_passage(self, text: str, source: str) -> CorpusPassage:
        text = text.strip()
        doc_id = hashlib.md5(text.encode()).hexdigest()[:12]
        return CorpusPassage(
            text=text, source=source, doc_id=doc_id, char_count=len(text),
        )

    def save_passages(
        self,
        passages: list[CorpusPassage],
        output_path: str,
    ) -> None:
        """Save passages to JSON for synthetic data generation."""
        data = [
            {"text": p.text, "source": p.source, "doc_id": p.doc_id}
            for p in passages
        ]
        with open(output_path, "w") as f:
            json.dump(data, f, indent=2)
        print(f"Saved {len(data)} passages to {output_path}")


# Usage
preparator = CorpusPreparator(min_chars=200, max_chars=2000)

# Option A: from markdown files
# passages = preparator.extract_from_markdown_files("./docs")

# Option B: from your vector store's chunks (preferred)
# chunks = vector_store.get_all_chunks()
# passages = preparator.extract_from_chunks(chunks)
```

### Corpus Quality Checklist

Before generating synthetic data, verify your corpus:

| Check | Why It Matters | How to Verify |
|---|---|---|
| Passage diversity | Classifiers need varied examples | Check topic distribution across passages |
| Passage length matches pipeline | Classifier learns your actual chunk patterns | Compare passage length to retriever chunk_size |
| No duplicate passages | Duplicates waste synthetic generation budget | Deduplicate by content hash |
| Sufficient volume | Need 500+ unique passages minimum | Count unique passages |
| Clean formatting | LLM generates better from clean text | Spot-check 20 random passages |

---

## Step 2: Generate Synthetic Training Data

### Configuration

```python
"""
Generate synthetic training data for ARES with quality controls.
"""
import json
import random
import time
from dataclasses import dataclass

import anthropic


@dataclass
class SyntheticConfig:
    """Configuration for synthetic data generation."""
    num_examples: int = 5000
    model: str = "claude-sonnet-4-20250514"
    max_retries: int = 3
    retry_delay: float = 2.0
    positive_ratio: float = 0.5
    seed: int = 42
    # Quality filters
    min_query_length: int = 10
    max_query_length: int = 200
    min_answer_length: int = 20
    max_answer_length: int = 500


class RobustSyntheticGenerator:
    """
    Generate synthetic training data with retry logic,
    quality filtering, and progress tracking.
    """

    def __init__(self, config: SyntheticConfig):
        self.config = config
        self.client = anthropic.Anthropic()
        random.seed(config.seed)

    def _call_llm(self, prompt: str, max_tokens: int = 300) -> str:
        """Call LLM with retry logic."""
        for attempt in range(self.config.max_retries):
            try:
                response = self.client.messages.create(
                    model=self.config.model,
                    max_tokens=max_tokens,
                    messages=[{"role": "user", "content": prompt}],
                )
                return response.content[0].text.strip()
            except Exception as e:
                if attempt < self.config.max_retries - 1:
                    wait = self.config.retry_delay * (2 ** attempt)
                    print(f"  Retry {attempt + 1}: {e}. Waiting {wait}s...")
                    time.sleep(wait)
                else:
                    raise

    def generate_query(self, passage: str) -> str:
        """Generate a natural question from a passage."""
        prompt = (
            "You are generating evaluation data for a search system. "
            "Given this passage from a knowledge base, write a single "
            "natural question that a user might ask which this passage "
            "answers. The question should be specific enough that the "
            "passage is clearly relevant.\n\n"
            "Rules:\n"
            "- Output ONLY the question, no prefix or explanation\n"
            "- Make it sound like a real user query, not a test question\n"
            "- Avoid yes/no questions; prefer 'how', 'what', 'why' questions\n\n"
            f"Passage:\n{passage[:1500]}"
        )
        return self._call_llm(prompt, max_tokens=150)

    def generate_faithful_answer(
        self, query: str, passage: str,
    ) -> str:
        """Generate an answer strictly grounded in the passage."""
        prompt = (
            "Answer this question using ONLY information from the provided "
            "passage. Do not add any external knowledge.\n\n"
            f"Question: {query}\n\n"
            f"Passage:\n{passage[:2000]}\n\n"
            "Answer (grounded only in the passage above):"
        )
        return self._call_llm(prompt, max_tokens=400)

    def generate_unfaithful_answer(
        self, query: str, passage: str,
    ) -> str:
        """Generate an answer with plausible but unsupported claims."""
        prompt = (
            "Answer this question. Start with information from the passage, "
            "but then add 1-2 specific claims that sound plausible but are "
            "NOT supported by the passage. These unsupported claims should "
            "blend in naturally and not be obviously wrong.\n\n"
            f"Question: {query}\n\n"
            f"Passage:\n{passage[:2000]}\n\n"
            "Answer (include some unsupported claims):"
        )
        return self._call_llm(prompt, max_tokens=400)

    def generate_off_topic_answer(self, query: str) -> str:
        """Generate a response that does not answer the question."""
        prompt = (
            "Generate a response that sounds knowledgeable but does NOT "
            "answer this question. The response should be about a related "
            "but different topic.\n\n"
            f"Question: {query}\n\n"
            "Off-topic response:"
        )
        return self._call_llm(prompt, max_tokens=400)

    def _passes_quality_filter(
        self, query: str, answer: str,
    ) -> bool:
        """Basic quality filter for generated examples."""
        if not query or not answer:
            return False
        if len(query) < self.config.min_query_length:
            return False
        if len(query) > self.config.max_query_length:
            return False
        if len(answer) < self.config.min_answer_length:
            return False
        if len(answer) > self.config.max_answer_length:
            return False
        # Reject if the LLM included meta-commentary
        reject_phrases = [
            "as an ai", "i cannot", "i'm sorry",
            "here is", "here's a", "sure,",
        ]
        lower_answer = answer.lower()
        if any(phrase in lower_answer[:50] for phrase in reject_phrases):
            return False
        return True

    def generate_full_dataset(
        self,
        passages: list[dict],
        output_path: str,
    ) -> list[dict]:
        """
        Generate a complete synthetic training dataset.

        For each selected passage, generates 4 examples:
        1. Positive (relevant context + faithful answer + relevant answer)
        2. Unfaithful (relevant context + unfaithful answer)
        3. Irrelevant context (wrong passage)
        4. Irrelevant answer (off-topic response)
        """
        num_passages_needed = self.config.num_examples // 4
        selected = random.sample(
            passages, min(num_passages_needed, len(passages)),
        )

        all_examples = []
        failed = 0

        for i, passage_data in enumerate(selected):
            passage = passage_data["text"]
            if (i + 1) % 50 == 0:
                print(
                    f"Progress: {i + 1}/{len(selected)} passages "
                    f"({len(all_examples)} examples, {failed} filtered)"
                )

            try:
                query = self.generate_query(passage)

                # 1. Positive example
                faithful = self.generate_faithful_answer(query, passage)
                if self._passes_quality_filter(query, faithful):
                    all_examples.append({
                        "query": query,
                        "context": passage,
                        "answer": faithful,
                        "context_relevance": 1,
                        "answer_faithfulness": 1,
                        "answer_relevance": 1,
                    })
                else:
                    failed += 1

                # 2. Unfaithful example
                unfaithful = self.generate_unfaithful_answer(query, passage)
                if self._passes_quality_filter(query, unfaithful):
                    all_examples.append({
                        "query": query,
                        "context": passage,
                        "answer": unfaithful,
                        "context_relevance": 1,
                        "answer_faithfulness": 0,
                        "answer_relevance": 1,
                    })
                else:
                    failed += 1

                # 3. Irrelevant context
                random_passage = random.choice(passages)["text"]
                while random_passage == passage:
                    random_passage = random.choice(passages)["text"]
                all_examples.append({
                    "query": query,
                    "context": random_passage,
                    "answer": faithful,
                    "context_relevance": 0,
                    "answer_faithfulness": 1,
                    "answer_relevance": 1,
                })

                # 4. Off-topic answer
                off_topic = self.generate_off_topic_answer(query)
                if self._passes_quality_filter(query, off_topic):
                    all_examples.append({
                        "query": query,
                        "context": passage,
                        "answer": off_topic,
                        "context_relevance": 1,
                        "answer_faithfulness": 0,
                        "answer_relevance": 0,
                    })
                else:
                    failed += 1

            except Exception as e:
                print(f"  Error on passage {i}: {e}")
                failed += 1
                continue

        random.shuffle(all_examples)

        # Save
        with open(output_path, "w") as f:
            json.dump(all_examples, f, indent=2)

        # Report
        print(f"\nGeneration complete:")
        print(f"  Total examples: {len(all_examples)}")
        print(f"  Filtered out: {failed}")
        print(f"  Saved to: {output_path}")

        # Label distribution
        for dim in ["context_relevance", "answer_faithfulness", "answer_relevance"]:
            pos = sum(1 for ex in all_examples if ex[dim] == 1)
            print(f"  {dim}: {pos}/{len(all_examples)} positive ({pos/len(all_examples)*100:.1f}%)")

        return all_examples
```

---

## Step 3: Fine-Tune Classifiers

### Training Script

```python
"""
Complete classifier training script with logging and checkpointing.
"""
import json
import os
from pathlib import Path

import numpy as np
import torch
from sklearn.metrics import (
    accuracy_score,
    classification_report,
    f1_score,
    roc_auc_score,
)
from sklearn.model_selection import train_test_split
from torch.utils.data import Dataset
from transformers import (
    AutoModelForSequenceClassification,
    AutoTokenizer,
    EarlyStoppingCallback,
    Trainer,
    TrainingArguments,
)


DIMENSIONS = ["context_relevance", "answer_faithfulness", "answer_relevance"]

INPUT_PAIRS = {
    "context_relevance": ("query", "context"),
    "answer_faithfulness": ("context", "answer"),
    "answer_relevance": ("query", "answer"),
}


class PairClassificationDataset(Dataset):
    """Dataset for training ARES classifiers."""

    def __init__(self, examples, tokenizer, dimension, max_length=512):
        self.tokenizer = tokenizer
        self.max_length = max_length
        field_a, field_b = INPUT_PAIRS[dimension]

        self.pairs = [(ex[field_a], ex[field_b]) for ex in examples]
        self.labels = [ex[dimension] for ex in examples]

    def __len__(self):
        return len(self.pairs)

    def __getitem__(self, idx):
        text_a, text_b = self.pairs[idx]
        enc = self.tokenizer(
            text_a, text_b,
            max_length=self.max_length,
            padding="max_length",
            truncation=True,
            return_tensors="pt",
        )
        return {
            "input_ids": enc["input_ids"].squeeze(),
            "attention_mask": enc["attention_mask"].squeeze(),
            "labels": torch.tensor(self.labels[idx], dtype=torch.long),
        }


def compute_metrics(eval_pred):
    logits, labels = eval_pred
    preds = np.argmax(logits, axis=-1)
    probs = torch.softmax(torch.tensor(logits), dim=-1)[:, 1].numpy()

    return {
        "accuracy": accuracy_score(labels, preds),
        "f1": f1_score(labels, preds),
        "auc_roc": roc_auc_score(labels, probs),
    }


def train_single_classifier(
    data: list[dict],
    dimension: str,
    output_dir: str,
    base_model: str = "microsoft/deberta-v3-base",
    max_length: int = 512,
    epochs: int = 3,
    batch_size: int = 16,
    learning_rate: float = 2e-5,
) -> dict:
    """
    Train one classifier (context_relevance, answer_faithfulness, or answer_relevance).
    """
    print(f"\n{'='*70}")
    print(f"Training: {dimension}")
    print(f"{'='*70}")

    # Stratified split
    train_data, val_data = train_test_split(
        data, test_size=0.1, random_state=42,
        stratify=[d[dimension] for d in data],
    )

    print(f"Train: {len(train_data)} examples")
    print(f"Val:   {len(val_data)} examples")
    train_pos = sum(1 for d in train_data if d[dimension] == 1)
    print(f"Train positive ratio: {train_pos / len(train_data):.2f}")

    tokenizer = AutoTokenizer.from_pretrained(base_model)
    model = AutoModelForSequenceClassification.from_pretrained(
        base_model, num_labels=2,
    )

    train_ds = PairClassificationDataset(train_data, tokenizer, dimension, max_length)
    val_ds = PairClassificationDataset(val_data, tokenizer, dimension, max_length)

    training_args = TrainingArguments(
        output_dir=output_dir,
        num_train_epochs=epochs,
        per_device_train_batch_size=batch_size,
        per_device_eval_batch_size=batch_size * 2,
        learning_rate=learning_rate,
        weight_decay=0.01,
        warmup_ratio=0.1,
        eval_strategy="steps",
        eval_steps=200,
        save_strategy="steps",
        save_steps=200,
        load_best_model_at_end=True,
        metric_for_best_model="auc_roc",
        greater_is_better=True,
        save_total_limit=2,
        logging_steps=50,
        fp16=torch.cuda.is_available(),
        dataloader_num_workers=4,
        report_to="none",
    )

    trainer = Trainer(
        model=model,
        args=training_args,
        train_dataset=train_ds,
        eval_dataset=val_ds,
        compute_metrics=compute_metrics,
        callbacks=[EarlyStoppingCallback(early_stopping_patience=3)],
    )

    result = trainer.train()
    eval_result = trainer.evaluate()

    # Save best model
    best_dir = f"{output_dir}/best"
    trainer.save_model(best_dir)
    tokenizer.save_pretrained(best_dir)

    # Detailed validation report
    val_preds = trainer.predict(val_ds)
    pred_labels = np.argmax(val_preds.predictions, axis=-1)
    true_labels = val_preds.label_ids

    print(f"\nClassification Report for {dimension}:")
    print(classification_report(
        true_labels, pred_labels, target_names=["negative", "positive"],
    ))

    metrics = {
        "dimension": dimension,
        "train_samples": len(train_data),
        "val_samples": len(val_data),
        "train_loss": result.training_loss,
        "val_accuracy": eval_result["eval_accuracy"],
        "val_f1": eval_result["eval_f1"],
        "val_auc_roc": eval_result["eval_auc_roc"],
        "model_path": best_dir,
    }

    # Save metrics
    with open(f"{output_dir}/metrics.json", "w") as f:
        json.dump(metrics, f, indent=2)

    return metrics


def train_all_classifiers(
    synthetic_data_path: str,
    output_base: str = "./ares_models",
    base_model: str = "microsoft/deberta-v3-base",
) -> dict:
    """Train all three ARES classifiers."""
    with open(synthetic_data_path) as f:
        data = json.load(f)

    print(f"Loaded {len(data)} synthetic examples")

    all_metrics = {}
    for dimension in DIMENSIONS:
        metrics = train_single_classifier(
            data=data,
            dimension=dimension,
            output_dir=f"{output_base}/{dimension}",
            base_model=base_model,
        )
        all_metrics[dimension] = metrics

    # Summary
    print(f"\n{'='*70}")
    print("Training Summary")
    print(f"{'='*70}")
    for dim, m in all_metrics.items():
        print(
            f"  {dim}: AUC={m['val_auc_roc']:.4f}, "
            f"F1={m['val_f1']:.4f}, Acc={m['val_accuracy']:.4f}"
        )

    # Save combined metrics
    with open(f"{output_base}/all_metrics.json", "w") as f:
        json.dump(all_metrics, f, indent=2)

    return all_metrics
```

### Expected Training Times

| Hardware | Time per Classifier (5K examples) | Total (3 classifiers) |
|---|---|---|
| CPU only | 3-5 hours | 9-15 hours |
| 1x T4 (16GB) | 25-35 min | 75-105 min |
| 1x A10 (24GB) | 15-20 min | 45-60 min |
| 1x A100 (80GB) | 8-12 min | 24-36 min |

---

## Step 4: Collect Human Annotations

### Annotation Interface

```python
"""
Minimal annotation interface for collecting ARES human labels.
"""
import json
import random
from pathlib import Path


class AnnotationCollector:
    """
    Collect human annotations for the PPI labeled set.

    Presents RAG pipeline outputs one at a time and collects
    binary judgments for context relevance, answer faithfulness,
    and answer relevance.
    """

    def __init__(
        self,
        pipeline_outputs: list[dict],
        num_to_annotate: int = 100,
        output_path: str = "./annotations.json",
    ):
        self.output_path = output_path
        self.all_outputs = pipeline_outputs

        # Select random subset for annotation
        random.seed(42)
        indices = random.sample(
            range(len(pipeline_outputs)),
            min(num_to_annotate, len(pipeline_outputs)),
        )
        self.to_annotate = [(i, pipeline_outputs[i]) for i in indices]
        self.annotations = self._load_existing()

    def _load_existing(self) -> dict:
        """Load any existing annotations to allow resuming."""
        if Path(self.output_path).exists():
            with open(self.output_path) as f:
                data = json.load(f)
            print(f"Loaded {len(data)} existing annotations")
            return data
        return {}

    def annotate_batch_cli(self) -> None:
        """Run CLI-based annotation (for simple setups)."""
        remaining = [
            (idx, ex) for idx, ex in self.to_annotate
            if str(idx) not in self.annotations
        ]
        print(f"\n{len(remaining)} examples remaining to annotate")
        print("For each example, enter 1 (yes) or 0 (no).")
        print("Enter 'q' to quit and save progress.\n")

        for item_num, (idx, ex) in enumerate(remaining):
            print(f"\n{'='*60}")
            print(f"Example {item_num + 1}/{len(remaining)} (index {idx})")
            print(f"{'='*60}")
            print(f"\nQUERY: {ex['query']}")
            print(f"\nCONTEXT (first 500 chars):\n{ex['context'][:500]}...")
            print(f"\nANSWER:\n{ex['answer']}")

            labels = {}
            for dimension, question in [
                ("context_relevance", "Is the context relevant to the query?"),
                ("answer_faithfulness", "Is the answer faithful to (supported by) the context?"),
                ("answer_relevance", "Does the answer actually address the query?"),
            ]:
                while True:
                    val = input(f"\n{question} (1=yes, 0=no, q=quit): ").strip()
                    if val == "q":
                        self._save()
                        print(f"Saved {len(self.annotations)} annotations.")
                        return
                    if val in ("0", "1"):
                        labels[dimension] = int(val)
                        break
                    print("Please enter 0, 1, or q.")

            self.annotations[str(idx)] = labels

            # Auto-save every 10 annotations
            if (item_num + 1) % 10 == 0:
                self._save()
                print(f"\nAuto-saved ({len(self.annotations)} total)")

        self._save()
        print(f"\nAnnotation complete. {len(self.annotations)} total annotations saved.")

    def annotate_with_llm_assist(
        self,
        model: str = "claude-sonnet-4-20250514",
    ) -> None:
        """
        LLM-assisted annotation: LLM suggests labels, human confirms or overrides.

        This speeds up annotation by 3-5x while maintaining human quality.
        """
        import anthropic
        client = anthropic.Anthropic()

        remaining = [
            (idx, ex) for idx, ex in self.to_annotate
            if str(idx) not in self.annotations
        ]

        for item_num, (idx, ex) in enumerate(remaining):
            # Get LLM suggestion
            response = client.messages.create(
                model=model,
                max_tokens=100,
                messages=[{
                    "role": "user",
                    "content": (
                        "Rate this RAG output on three dimensions (1=yes, 0=no):\n\n"
                        f"Query: {ex['query']}\n\n"
                        f"Context: {ex['context'][:1500]}\n\n"
                        f"Answer: {ex['answer']}\n\n"
                        "Respond in exactly this format:\n"
                        "context_relevance=X\n"
                        "answer_faithfulness=X\n"
                        "answer_relevance=X"
                    ),
                }],
            )

            # Parse suggestion
            suggestion = {}
            for line in response.content[0].text.strip().split("\n"):
                if "=" in line:
                    key, val = line.split("=")
                    key = key.strip()
                    val = val.strip()
                    if key in ("context_relevance", "answer_faithfulness", "answer_relevance"):
                        suggestion[key] = int(val)

            print(f"\n{'='*60}")
            print(f"Example {item_num + 1}/{len(remaining)}")
            print(f"Query: {ex['query'][:100]}")
            print(f"LLM suggests: {suggestion}")

            confirm = input("Accept? (y=accept, n=manual, q=quit): ").strip()
            if confirm == "q":
                self._save()
                return
            elif confirm == "y":
                self.annotations[str(idx)] = suggestion
            else:
                # Manual override
                labels = {}
                for dim in ["context_relevance", "answer_faithfulness", "answer_relevance"]:
                    val = input(f"  {dim} (0/1): ").strip()
                    labels[dim] = int(val)
                self.annotations[str(idx)] = labels

            if (item_num + 1) % 10 == 0:
                self._save()

        self._save()

    def _save(self) -> None:
        with open(self.output_path, "w") as f:
            json.dump(self.annotations, f, indent=2)

    def get_labeled_data(self) -> tuple[list[int], dict]:
        """
        Return labeled indices and labels formatted for PPI.

        Returns:
            (labeled_indices, labels_by_dimension)
        """
        indices = sorted(int(k) for k in self.annotations.keys())
        labels = {
            "context_relevance": np.array([
                self.annotations[str(i)]["context_relevance"] for i in indices
            ], dtype=float),
            "answer_faithfulness": np.array([
                self.annotations[str(i)]["answer_faithfulness"] for i in indices
            ], dtype=float),
            "answer_relevance": np.array([
                self.annotations[str(i)]["answer_relevance"] for i in indices
            ], dtype=float),
        }
        return indices, labels


# Need numpy for get_labeled_data
import numpy as np
```

---

## Step 5: Run PPI Estimation

### Putting It All Together

```python
"""
Complete ARES evaluation pipeline: score with classifiers + PPI.
"""
import json

import numpy as np
import torch
from scipy import stats
from transformers import AutoModelForSequenceClassification, AutoTokenizer


def load_classifier(model_dir: str, device: str = "cpu"):
    """Load a trained ARES classifier."""
    model = AutoModelForSequenceClassification.from_pretrained(model_dir)
    tokenizer = AutoTokenizer.from_pretrained(model_dir)
    model.eval()
    model.to(device)
    return model, tokenizer


def score_examples(
    model,
    tokenizer,
    text_pairs: list[tuple[str, str]],
    device: str = "cpu",
    batch_size: int = 32,
) -> np.ndarray:
    """Score text pairs with a classifier, returning P(positive)."""
    all_probs = []

    for i in range(0, len(text_pairs), batch_size):
        batch = text_pairs[i : i + batch_size]
        texts_a = [p[0] for p in batch]
        texts_b = [p[1] for p in batch]

        enc = tokenizer(
            texts_a, texts_b,
            max_length=512, padding=True, truncation=True,
            return_tensors="pt",
        ).to(device)

        with torch.no_grad():
            logits = model(**enc).logits
            probs = torch.softmax(logits, dim=-1)[:, 1]
            all_probs.extend(probs.cpu().numpy())

    return np.array(all_probs)


def ppi_estimate(
    all_scores: np.ndarray,
    human_labels: np.ndarray,
    labeled_indices: list[int],
    threshold: float = 0.5,
    alpha: float = 0.05,
) -> dict:
    """Compute PPI estimate with confidence interval."""
    N = len(all_scores)
    n = len(human_labels)

    all_binary = (all_scores >= threshold).astype(float)
    labeled_binary = all_binary[labeled_indices]

    theta_f = np.mean(all_binary)
    correction = np.mean(human_labels - labeled_binary)
    theta_ppi = theta_f + correction

    residuals = human_labels - labeled_binary
    var_total = np.var(residuals, ddof=1) / n + np.var(all_binary, ddof=1) / N

    z = stats.norm.ppf(1 - alpha / 2)
    ci_half = z * np.sqrt(var_total)

    return {
        "estimate": float(np.clip(theta_ppi, 0, 1)),
        "ci_lower": float(np.clip(theta_ppi - ci_half, 0, 1)),
        "ci_upper": float(np.clip(theta_ppi + ci_half, 0, 1)),
        "ci_width": float(2 * ci_half),
        "classifier_raw": float(theta_f),
        "correction": float(correction),
    }


def run_ares_evaluation(
    pipeline_outputs: list[dict],
    classifiers_dir: str,
    annotations_path: str,
    device: str = "cpu",
) -> dict:
    """
    Full ARES evaluation pipeline.

    Args:
        pipeline_outputs: list of dicts with 'query', 'context', 'answer'
        classifiers_dir: directory containing trained classifiers
        annotations_path: path to human annotations JSON

    Returns:
        dict with PPI estimates and confidence intervals for each dimension
    """
    # Load annotations
    with open(annotations_path) as f:
        raw_annotations = json.load(f)

    labeled_indices = sorted(int(k) for k in raw_annotations.keys())

    dimensions_config = {
        "context_relevance": ("query", "context"),
        "answer_faithfulness": ("context", "answer"),
        "answer_relevance": ("query", "answer"),
    }

    results = {}

    for dimension, (field_a, field_b) in dimensions_config.items():
        print(f"\nScoring {dimension}...")

        model, tokenizer = load_classifier(
            f"{classifiers_dir}/{dimension}/best", device
        )

        pairs = [
            (ex[field_a], ex[field_b]) for ex in pipeline_outputs
        ]
        scores = score_examples(model, tokenizer, pairs, device)

        human_labels = np.array([
            raw_annotations[str(i)][dimension] for i in labeled_indices
        ], dtype=float)

        result = ppi_estimate(scores, human_labels, labeled_indices)
        results[dimension] = result

        print(
            f"  {dimension}: {result['estimate']:.3f} "
            f"[{result['ci_lower']:.3f}, {result['ci_upper']:.3f}]"
        )

    print(f"\n{'='*50}")
    print("ARES Evaluation Results")
    print(f"{'='*50}")
    for dim, r in results.items():
        print(
            f"  {dim}: {r['estimate']:.3f} "
            f"95% CI [{r['ci_lower']:.3f}, {r['ci_upper']:.3f}] "
            f"(width: {r['ci_width']:.3f})"
        )

    return results
```

---

## Troubleshooting

### Common Issues

| Issue | Symptom | Fix |
|---|---|---|
| Classifier AUC < 0.70 | Poor discrimination | Increase synthetic data to 10K+, check for label noise |
| Wide CI (> 0.15) | Too few human labels | Annotate more examples (target 100+) |
| PPI correction > 0.15 | Large classifier bias | Classifier may not match domain; retrain with more synthetic data or better LLM |
| GPU OOM during training | CUDA out of memory | Reduce batch_size to 8, reduce max_length to 256 |
| Slow synthetic generation | Rate limiting | Use batch API, add delays, or switch to cheaper model for negatives |
| Classifier overfits | Val loss increases after epoch 1 | Reduce epochs to 2, increase dropout, use early stopping |

### Validation Checklist

Before trusting ARES results, verify:

1. All three classifiers have val AUC-ROC > 0.75 (ideally > 0.85)
2. At least 50 human annotations are collected (100+ recommended)
3. PPI correction magnitude is < 0.10 for each dimension (indicates classifier is reasonably calibrated)
4. CI widths are < 0.10 (narrow enough to distinguish between pipeline versions)
5. Synthetic data has balanced positive/negative ratio (40-60% positive)

---

## References

- Saad-Falcon, J. et al. "ARES: An Automated Evaluation Framework for Retrieval-Augmented Generation Systems." arXiv 2023.
- Angelopoulos, A. et al. "Prediction-Powered Inference." Science, 2023.
- ares-ai package: https://pypi.org/project/ares-ai/
- DeBERTa-v3: https://huggingface.co/microsoft/deberta-v3-base
- HuggingFace Trainer: https://huggingface.co/docs/transformers/main_classes/trainer
