# Embedding Fine-Tuning -- Comprehensive Guide

## Overview / TL;DR

Fine-tuning an embedding model adapts a general-purpose model to your specific domain, improving retrieval quality by 5-15% on average (measured by Hit@10 and NDCG@10). This guide covers when and why to fine-tune, data requirements (1K-100K training pairs), cost and time estimation, base model selection, and the end-to-end workflow from data preparation to deployment. Fine-tuning is the single highest-ROI optimization you can make when retrieval quality on your domain-specific corpus plateaus with off-the-shelf models.

---

## When to Fine-Tune

### Strong Signals That Fine-Tuning Will Help

1. **Domain-specific vocabulary**: Your corpus uses specialized terminology (medical codes, legal citations, financial instruments) that general models did not see during training.
2. **Retrieval quality plateau**: You have optimized chunking, tried multiple off-the-shelf models, and added re-ranking, but retrieval quality on your evaluation set is still unsatisfactory.
3. **Semantic gaps**: Your evaluation shows that relevant documents score low because the model does not understand domain-specific synonym relationships (e.g., "MI" = "myocardial infarction" = "heart attack").
4. **Unique query patterns**: Your users ask questions in a specific style that differs from general web search (e.g., structured medical queries, legal case citations, code search patterns).

### When NOT to Fine-Tune

1. **Small corpus (<1000 documents)**: The improvement from fine-tuning will be minimal, and you risk overfitting. Focus on better chunking and prompt engineering instead.
2. **No evaluation dataset**: Without a labeled evaluation set, you cannot measure whether fine-tuning helped. Build the eval set first.
3. **General-purpose queries**: If your queries and documents look like web search, off-the-shelf models are already optimized for this.
4. **Budget/time constraints**: Fine-tuning requires data preparation, training, evaluation, and deployment. If you need results in hours, not weeks, use re-ranking as a faster alternative.

---

## Expected Improvement

Based on published results and production deployments:

| Domain | Base Model | Fine-Tuned | Improvement (NDCG@10) |
|--------|-----------|-----------|----------------------|
| Medical literature | BGE-M3 (65.3) | Domain-tuned (72.8) | +7.5 points (+11.5%) |
| Legal contracts | text-emb-3-large (66.1) | Domain-tuned (73.5) | +7.4 points (+11.2%) |
| Financial reports | Voyage-3 (67.0) | Domain-tuned (72.1) | +5.1 points (+7.6%) |
| Code search | bge-base-en (62.1) | Code-tuned (69.8) | +7.7 points (+12.4%) |
| Technical documentation | bge-large-en (63.8) | Domain-tuned (68.2) | +4.4 points (+6.9%) |
| E-commerce product search | bge-base-en (62.1) | Domain-tuned (70.5) | +8.4 points (+13.5%) |
| Customer support tickets | bge-base-en (62.1) | Domain-tuned (68.9) | +6.8 points (+11.0%) |

**General rule**: The more specialized your domain, the more fine-tuning helps. General technical documentation sees 5-7% improvement; highly specialized domains (medical, legal) see 10-15%.

---

## Data Requirements

### Minimum Viable Dataset

| Dataset Size | Expected Quality | Use Case |
|-------------|-----------------|----------|
| 1K pairs | Marginal improvement (2-5%) | Proof of concept, quick experiment |
| 5K pairs | Moderate improvement (5-8%) | Small-domain specialization |
| 10K pairs | Good improvement (8-12%) | Production fine-tuning |
| 50K pairs | Near-optimal (10-15%) | High-stakes domains |
| 100K+ pairs | Diminishing returns | Only if you have the data naturally |

### Data Format: Training Triples

The standard format for retrieval fine-tuning is **(query, positive, negative)** triples:

```python
# Training data format
training_data = [
    {
        "query": "What are the side effects of metformin?",
        "positive": "Metformin commonly causes gastrointestinal side effects including nausea, diarrhea, and abdominal pain. Lactic acidosis is a rare but serious side effect.",
        "negative": "Metformin is an oral diabetes medication that helps control blood sugar levels. It is typically the first-line treatment for type 2 diabetes.",
    },
    {
        "query": "How to configure NGINX reverse proxy?",
        "positive": "To set up NGINX as a reverse proxy, use the proxy_pass directive in the location block: location / { proxy_pass http://backend_server; }",
        "negative": "NGINX is a high-performance web server that can handle thousands of concurrent connections. It was created by Igor Sysoev in 2004.",
    },
]
```

### Generating Training Data

#### Method 1: From Existing Query Logs

```python
import json
from dataclasses import dataclass


@dataclass
class TrainingTriple:
    query: str
    positive: str
    negative: str


def create_triples_from_logs(
    query_logs: list[dict],  # [{"query": "...", "clicked_doc": "...", "shown_docs": [...]}]
    corpus: list[str],
) -> list[TrainingTriple]:
    """Create training triples from search/click logs.

    Clicked documents are positives. Shown-but-not-clicked are hard negatives.
    """
    triples = []

    for log in query_logs:
        query = log["query"]
        positive = log["clicked_doc"]

        # Hard negative: shown but not clicked
        shown_not_clicked = [
            doc for doc in log["shown_docs"]
            if doc != positive
        ]

        if shown_not_clicked:
            negative = shown_not_clicked[0]  # Most confusing negative
            triples.append(TrainingTriple(query=query, positive=positive, negative=negative))

    return triples
```

#### Method 2: Synthetic Generation with LLM

```python
import anthropic


def generate_synthetic_queries(
    documents: list[str],
    queries_per_doc: int = 3,
) -> list[dict]:
    """Generate synthetic queries for documents using Claude.

    This is the most practical approach when you do not have query logs.
    """
    client = anthropic.Anthropic()
    training_pairs = []

    for doc in documents:
        response = client.messages.create(
            model="claude-haiku-4-20250514",
            max_tokens=1000,
            messages=[{
                "role": "user",
                "content": (
                    f"Given the following document, generate {queries_per_doc} realistic "
                    f"search queries that this document would be a good answer for. "
                    f"Make queries diverse (different angles, specificity levels). "
                    f"Return one query per line, nothing else.\n\n"
                    f"Document:\n{doc[:2000]}"
                ),
            }],
        )

        queries = [
            q.strip() for q in response.content[0].text.strip().split("\n")
            if q.strip()
        ]

        for query in queries[:queries_per_doc]:
            training_pairs.append({
                "query": query,
                "positive": doc,
            })

    return training_pairs


def add_hard_negatives(
    training_pairs: list[dict],
    corpus: list[str],
    embedding_model_name: str = "BAAI/bge-base-en-v1.5",
    n_negatives: int = 7,
) -> list[dict]:
    """Add hard negatives using the base embedding model.

    Hard negatives are documents that are similar but not relevant.
    See hard-negative-mining/ KB entry for advanced strategies.
    """
    from sentence_transformers import SentenceTransformer
    import numpy as np

    model = SentenceTransformer(embedding_model_name)

    # Embed all corpus documents
    corpus_embs = model.encode(corpus, normalize_embeddings=True, batch_size=256)

    for pair in training_pairs:
        # Embed the query
        query_emb = model.encode([pair["query"]], normalize_embeddings=True)[0]

        # Find most similar documents (excluding the positive)
        scores = np.dot(corpus_embs, query_emb)
        ranked = np.argsort(scores)[::-1]

        negatives = []
        for idx in ranked:
            if corpus[idx] != pair["positive"] and len(negatives) < n_negatives:
                negatives.append(corpus[idx])

        pair["negatives"] = negatives

    return training_pairs
```

---

## Cost and Time Estimation

### Training Cost (GPU)

| Base Model | Parameters | GPU Required | Training Time (10K triples) | Estimated Cost |
|-----------|-----------|-------------|---------------------------|---------------|
| bge-base-en-v1.5 | 109M | T4 (16GB) | 15-30 min | $0.15-0.30 |
| bge-large-en-v1.5 | 335M | A10G (24GB) | 30-60 min | $0.40-0.80 |
| BGE-M3 | 568M | A10G (24GB) | 45-90 min | $0.60-1.20 |
| E5-Mistral-7B | 7B | A100 (80GB) | 4-8 hours | $16-32 |

### Data Preparation Cost (LLM for Synthetic Queries)

| Corpus Size | Queries per Doc | Total Queries | Claude Haiku Cost |
|------------|----------------|---------------|-------------------|
| 1,000 docs | 3 | 3,000 | ~$1.50 |
| 10,000 docs | 3 | 30,000 | ~$15 |
| 100,000 docs | 2 | 200,000 | ~$100 |

### Total Project Cost

| Scenario | Data Prep | Training | Evaluation | Total |
|----------|----------|---------|-----------|-------|
| Quick experiment (1K pairs, base model) | $0.50 | $0.15 | $0.05 | ~$1 |
| Production (10K pairs, large model) | $15 | $1.20 | $0.50 | ~$17 |
| High-stakes (50K pairs, M3) | $50 | $3 | $2 | ~$55 |

---

## Base Model Selection for Fine-Tuning

| Requirement | Recommended Base | Why |
|------------|-----------------|-----|
| Lowest cost, fastest training | bge-base-en-v1.5 (109M) | Small, fast, good starting quality |
| Best English quality | bge-large-en-v1.5 (335M) | Higher capacity, better starting point |
| Multilingual | BGE-M3 (568M) | 100+ languages, strong baseline |
| Code | bge-base with code data | Add code-specific training data |
| Maximum quality | E5-Mistral-7B (with LoRA) | Highest capacity, best potential |

---

## End-to-End Workflow

### Step 1: Prepare Evaluation Set First

Before any training, create a held-out evaluation set. This is non-negotiable.

```python
import random


def split_training_eval(
    training_data: list[dict],
    eval_fraction: float = 0.1,
    min_eval_size: int = 100,
) -> tuple[list[dict], list[dict]]:
    """Split data into training and evaluation sets.

    The eval set must be large enough for statistically meaningful results.
    """
    random.shuffle(training_data)
    eval_size = max(int(len(training_data) * eval_fraction), min_eval_size)
    eval_size = min(eval_size, len(training_data) // 2)  # Never use more than half

    eval_data = training_data[:eval_size]
    train_data = training_data[eval_size:]

    print(f"Training set: {len(train_data)} examples")
    print(f"Evaluation set: {len(eval_data)} examples")

    return train_data, eval_data
```

### Step 2: Train the Model

See `training-guide.md` in this directory for complete training code with loss functions and training arguments.

### Step 3: Evaluate

See `evaluation.md` in this directory for evaluation methodology (Hit@K, MRR, NDCG@10, A/B testing).

### Step 4: Deploy

```python
from sentence_transformers import SentenceTransformer
import numpy as np


def deploy_finetuned_model(
    model_path: str,
    corpus: list[str],
    output_embeddings_path: str,
):
    """Deploy a fine-tuned model: re-embed the entire corpus.

    CRITICAL: After fine-tuning, you MUST re-embed your entire corpus.
    The fine-tuned model produces vectors in a different space than the base model.
    Old embeddings are INCOMPATIBLE with the new model.
    """
    model = SentenceTransformer(model_path)

    print(f"Re-embedding {len(corpus)} documents...")
    embeddings = model.encode(
        corpus,
        normalize_embeddings=True,
        batch_size=256,
        show_progress_bar=True,
    )

    np.save(output_embeddings_path, embeddings)
    print(f"Saved embeddings: {embeddings.shape}")

    return embeddings
```

---

## Fine-Tuning Alternatives

If full fine-tuning is too heavy, consider these lighter alternatives:

### Alternative 1: Adapter/LoRA Fine-Tuning

Train only a small adapter module (2-5% of parameters). Faster and cheaper.

```python
from peft import LoraConfig, get_peft_model
from transformers import AutoModel

base_model = AutoModel.from_pretrained("BAAI/bge-large-en-v1.5")

lora_config = LoraConfig(
    r=16,                     # LoRA rank
    lora_alpha=32,            # Scaling factor
    target_modules=["query", "value"],  # Which layers to adapt
    lora_dropout=0.1,
    bias="none",
)

model = get_peft_model(base_model, lora_config)
trainable_params = sum(p.numel() for p in model.parameters() if p.requires_grad)
total_params = sum(p.numel() for p in model.parameters())
print(f"Trainable: {trainable_params:,} / {total_params:,} ({100*trainable_params/total_params:.1f}%)")
```

### Alternative 2: Re-ranking Instead of Fine-Tuning

Add a cross-encoder re-ranker fine-tuned on your domain. This avoids re-embedding and can be deployed alongside any base embedding model.

### Alternative 3: Contextual Retrieval

Prepend context to each chunk using an LLM (Anthropic's contextual retrieval approach). This enriches chunks with domain context without changing the embedding model.

---

## Common Pitfalls

1. **Skipping the evaluation set.** Without held-out evaluation data, you have no way to know if fine-tuning helped or caused overfitting.
2. **Too few negatives.** Using random negatives only teaches the model obvious distinctions. Hard negatives (similar but irrelevant) are critical for meaningful improvement.
3. **Overfitting on small datasets.** With fewer than 1K training pairs, the model may memorize rather than generalize. Use early stopping and monitor eval loss.
4. **Forgetting to re-embed.** The most common deployment mistake: using old embeddings with a new model. They are incompatible.
5. **Fine-tuning the wrong model.** Starting with a weak base model (e.g., all-MiniLM) limits the ceiling. Start with the strongest model your GPU can handle.
6. **Not comparing against the base model on the same eval set.** Always run the base model on your eval set as a baseline before and after fine-tuning.

---

## References

- Sentence-Transformers Training -- https://www.sbert.net/docs/training/overview.html
- BGE Fine-Tuning -- https://github.com/FlagOpen/FlagEmbedding/tree/master/FlagEmbedding/baai_general_embedding
- Matryoshka Fine-Tuning -- https://huggingface.co/blog/matryoshka
- LoRA: Low-Rank Adaptation -- https://arxiv.org/abs/2106.09685
- Mining Hard Negatives for Retrieval -- https://arxiv.org/abs/2112.07899
- Anthropic Contextual Retrieval -- https://www.anthropic.com/news/contextual-retrieval
