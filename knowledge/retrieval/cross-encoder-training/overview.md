# Cross-Encoder Training -- Overview

## What Is a Cross-Encoder?

A cross-encoder is a transformer model that takes a query-document pair as a single input and outputs a relevance score. Unlike bi-encoders (which encode query and document separately), cross-encoders perform full attention between query and document tokens, allowing them to capture fine-grained interactions like negation, coreference, and contextual word sense.

```
Bi-encoder:
  Query -> Encoder -> q_vec    }
  Doc   -> Encoder -> d_vec    } -> dot_product(q_vec, d_vec) = score

Cross-encoder:
  [CLS] query tokens [SEP] document tokens [SEP]
  -> Encoder -> [CLS] representation -> Linear -> score
```

Cross-encoders achieve the highest quality among neural ranking models but cannot be used for first-stage retrieval (they require running the full model on every candidate). Their role is as a **reranker**: take 20-100 candidates from a fast first-stage retriever (BM25, dense, SPLADE) and reorder them.

---

## Why Fine-Tune a Cross-Encoder?

Off-the-shelf cross-encoders (e.g., `cross-encoder/ms-marco-MiniLM-L-12-v2`, `BAAI/bge-reranker-v2-m3`) are trained on MS MARCO, a general web search dataset. They perform well on general queries but may underperform on domain-specific data:

| Domain | Off-the-Shelf NDCG@10 | Fine-Tuned NDCG@10 | Improvement |
|---|---|---|---|
| General web (MS MARCO) | 0.390 | 0.390 (already trained) | -- |
| Legal (CaseHOLD) | 0.520 | 0.620 | +19% |
| Biomedical (TREC-COVID) | 0.680 | 0.780 | +15% |
| Financial (FiQA) | 0.360 | 0.440 | +22% |
| Code search (CodeSearchNet) | 0.410 | 0.530 | +29% |
| E-commerce (WANDS) | 0.450 | 0.560 | +24% |

These improvements come from adapting the model to domain-specific vocabulary, relevance patterns, and document structure.

---

## Training Data Requirements

### What You Need

Cross-encoder training requires labeled query-document pairs with relevance labels. The minimum format:

```
query | document | label
```

Where label is either:
- **Binary**: 0 (not relevant) or 1 (relevant)
- **Graded**: 0 (not relevant), 1 (somewhat relevant), 2 (relevant), 3 (highly relevant)

### How Much Data?

| Data Volume | Expected Quality | Notes |
|---|---|---|
| 100-500 pairs | Marginal improvement | Barely enough for fine-tuning, risk of overfitting |
| 1K-5K pairs | Solid domain adaptation | Minimum recommended for production use |
| 10K-50K pairs | Strong domain performance | Sweet spot for most domains |
| 100K+ pairs | Diminishing returns | Quality plateaus, more data does not help much |
| 500K+ (MS MARCO scale) | Full training from scratch | Only needed if starting from a base LM, not a pretrained reranker |

### Sources of Training Data

1. **Manual annotation**: most reliable but expensive. Use tools like Label Studio or Prodigy. Budget: ~$0.10-0.50 per judgment.

2. **Click-through data**: if you have a search system, clicks provide implicit relevance labels. Noisy but abundant. Use click-through rate (CTR) or dwell time as proxy relevance.

3. **LLM-generated labels**: use Claude or GPT-4 to judge relevance for query-document pairs. Cost: ~$0.001-0.01 per judgment. Quality approaches human annotation for clear-cut cases.

4. **Distillation from strong reranker**: use an LLM reranker (RankGPT) on your data to generate soft labels or rankings. Train the cross-encoder to match these rankings.

5. **Existing datasets**: for common domains, labeled datasets may already exist (MS MARCO for web search, BioASQ for biomedical, FiQA for finance).

---

## The Role of Hard Negatives

### Why Hard Negatives Matter

A "hard negative" is a document that looks relevant but is not. Training with only random negatives teaches the model to distinguish obviously irrelevant documents (easy) but not subtly irrelevant ones (hard).

```
Query: "Python asyncio connection pool"

Easy negative: "The history of the Roman Empire spans several centuries..."
  -> Trivially irrelevant, the model learns nothing useful

Hard negative (BM25): "Python provides synchronous database connections
  through psycopg2 and SQLAlchemy's default engine..."
  -> Shares key terms but discusses sync, not async

Hard negative (dense): "Node.js async/await patterns for database
  connection management with pg-pool..."
  -> Semantically similar but wrong language/ecosystem
```

### Sources of Hard Negatives

1. **BM25 retrieval**: retrieve top-100 documents for each query using BM25. Documents ranked 10-100 that are not labeled relevant make excellent hard negatives (they match keywords but miss the intent).

2. **Dense retrieval**: encode the query with a dense model and retrieve nearest neighbors. Non-relevant neighbors are hard negatives for the semantic dimension.

3. **Cross-model negatives**: use one retriever's failures as another's hard negatives. BM25's top results that a dense model ranks low (and vice versa) are complementary hard negatives.

4. **In-batch negatives**: during training, positive documents from other queries in the same batch serve as negatives. Simple and effective but not as hard as mined negatives.

### How Many Hard Negatives?

| Hard Negatives per Query | Quality Impact | Training Cost |
|---|---|---|
| 1 | Moderate improvement | Baseline |
| 3-5 | Strong improvement | 3-5x data size |
| 10-15 | Near-optimal | 10-15x data size |
| 30+ | Diminishing returns | High memory usage |

The sweet spot is 5-10 hard negatives per query, mixing BM25 and dense-retrieved negatives.

---

## Architecture Choices

### Model Selection

| Model | Params | Latency (20 docs, GPU) | Quality (MS MARCO MRR@10) | Recommended For |
|---|---|---|---|---|
| MiniLM-L-6-v2 | 22M | ~12ms | 0.369 | Maximum throughput |
| MiniLM-L-12-v2 | 33M | ~25ms | 0.390 | Balanced |
| DistilBERT | 66M | ~30ms | 0.385 | Good baseline |
| BERT-base | 110M | ~45ms | 0.395 | Quality focus |
| DeBERTa-v3-base | 86M | ~50ms | 0.405 | Best quality at base size |
| BGE-reranker-v2-m3 | 568M | ~120ms | 0.420 | Best off-the-shelf |
| DeBERTa-v3-large | 304M | ~150ms | 0.425 | Best quality (self-trained) |

For fine-tuning, start with `cross-encoder/ms-marco-MiniLM-L-12-v2` (already has MS MARCO knowledge) and adapt to your domain. If starting from scratch, DeBERTa-v3-base offers the best quality-per-parameter.

### Max Sequence Length

Cross-encoders concatenate query + document tokens. The maximum length determines how much text the model can consider:

- **256 tokens**: handles queries up to ~30 words + passages up to ~180 words. Fast but may truncate long passages.
- **512 tokens**: the standard choice. Handles most query-passage pairs without truncation.
- **1024+ tokens**: for long documents. Requires models trained at this length (or careful position embedding extension). Significantly slower.

```python
# Estimate token budget for your data
def estimate_token_budget(
    queries: list[str],
    documents: list[str],
    tokenizer,
) -> dict:
    """
    Analyze token length distribution to choose max_length.
    """
    import numpy as np

    q_lengths = [len(tokenizer.encode(q)) for q in queries]
    d_lengths = [len(tokenizer.encode(d)) for d in documents]

    # Combined length (query + [SEP] + document + [SEP])
    combined = [q + d + 3 for q, d in zip(q_lengths, d_lengths)]

    return {
        "query_tokens": {
            "mean": round(np.mean(q_lengths), 1),
            "p95": int(np.percentile(q_lengths, 95)),
            "max": max(q_lengths),
        },
        "doc_tokens": {
            "mean": round(np.mean(d_lengths), 1),
            "p95": int(np.percentile(d_lengths, 95)),
            "max": max(d_lengths),
        },
        "combined_tokens": {
            "mean": round(np.mean(combined), 1),
            "p95": int(np.percentile(combined, 95)),
            "max": max(combined),
        },
        "recommended_max_length": (
            256 if np.percentile(combined, 95) <= 256
            else 512 if np.percentile(combined, 95) <= 512
            else 1024
        ),
    }
```

---

## Loss Functions

### Binary Cross-Entropy (BCE)

The standard loss for binary relevance labels:

```
L = -[ y * log(sigma(s)) + (1-y) * log(1 - sigma(s)) ]
```

Where s is the model's raw score and sigma is the sigmoid function. Each query-document pair is scored independently.

**Best for**: binary relevance labels (relevant / not relevant).

### Margin Ranking Loss

Directly optimizes the margin between positive and negative document scores:

```
L = max(0, margin - (s_pos - s_neg))
```

The model learns to produce scores where positives are at least `margin` points above negatives.

**Best for**: training with hard negatives where you want clear separation.

### Knowledge Distillation (KL Divergence / MSE)

When using a teacher model (e.g., LLM reranker) to generate soft labels:

```
L = MSE(student_score, teacher_score)
  or
L = KL(softmax(student_scores), softmax(teacher_scores))
```

**Best for**: distilling LLM reranking quality into a fast cross-encoder.

### Listwise Losses (LambdaLoss, ListMLE)

Consider the full ranking of candidates for a query, not just individual pairs:

```
L = -SUM_i log( exp(s_pi(i)) / SUM_{j>=i} exp(s_pi(j)) )
```

Where pi is the ground-truth permutation. This directly optimizes ranking metrics like NDCG.

**Best for**: when you have graded relevance labels and want to optimize ranking quality.

---

## Training Hyperparameters

### Recommended Starting Point

| Hyperparameter | Value | Notes |
|---|---|---|
| Learning rate | 2e-5 to 3e-5 | Lower for larger models |
| Batch size | 16-64 pairs | Larger if memory allows |
| Epochs | 2-5 | More for small datasets, less for large |
| Warmup | 10% of total steps | Standard for transformer fine-tuning |
| Weight decay | 0.01 | Standard AdamW |
| Max sequence length | 512 | Reduce to 256 if passages are short |
| Hard negatives per query | 5-10 | Mix BM25 and dense |
| Gradient accumulation | 2-4 steps | If batch size is memory-limited |

### Learning Rate Sensitivity

Cross-encoder fine-tuning is sensitive to learning rate. Too high causes catastrophic forgetting of pretrained knowledge; too low causes underfitting.

```python
# Recommended learning rate grid search
learning_rates = [1e-5, 2e-5, 3e-5, 5e-5]
# Evaluate each on a validation set (10% of training data)
# Choose the learning rate with the best NDCG@10 on validation
```

---

## When to Fine-Tune vs. Use Off-the-Shelf

### Use off-the-shelf when:
- Your domain is close to web search (the model's training data)
- You have fewer than 500 labeled pairs
- You need a quick baseline before investing in annotation
- Your retrieval quality is already bottlenecked by the first-stage retriever, not the reranker

### Fine-tune when:
- Your domain has specialized vocabulary (medical, legal, code)
- Off-the-shelf models score below your quality threshold
- You have 1K+ labeled query-document pairs (or can generate them with an LLM)
- Reranking quality directly impacts your product (e.g., RAG answer quality, search revenue)

### Distill from LLM when:
- You want LLM-level reranking quality at cross-encoder speed
- You have access to a strong LLM (Claude Sonnet, GPT-4o) for generating training data
- Your volume justifies the one-time investment in distillation

---

## Common Pitfalls

1. **Fine-tuning without hard negatives**: training on random negatives barely improves the model. Always mine hard negatives from BM25 and/or dense retrieval.

2. **Too many epochs on small data**: cross-encoders overfit quickly on small datasets (< 5K pairs). Use early stopping on a validation set and rarely exceed 3 epochs.

3. **Ignoring the base model**: starting from a general-purpose BERT is much slower to converge than starting from `cross-encoder/ms-marco-MiniLM-L-12-v2` which already understands relevance.

4. **Not evaluating on held-out data**: always split your data into train/validation/test. Report metrics on the test set only.

5. **Truncating long passages silently**: if your passages are longer than max_length, the model silently loses information. Either chunk passages or increase max_length.

6. **Comparing fine-tuned cross-encoders to retrieval models**: cross-encoders are rerankers, not retrievers. Compare them against other rerankers (off-the-shelf cross-encoders, Cohere Rerank, LLM rerankers), not against BM25 or dense retrieval directly.

---

## References

- Nogueira, R. and Cho, K. "Passage Re-ranking with BERT." arXiv 2019.
- Thakur, N. et al. "BEIR: A Heterogeneous Benchmark for Zero-shot Evaluation of Information Retrieval Models." NeurIPS 2021.
- Reimers, N. and Gurevych, I. "Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks." EMNLP 2019.
- Wang, L. et al. "Improving Text Embeddings with Large Language Models." arXiv 2024.
- Gao, L. and Callan, J. "Is Your Cross-Encoder Robust? Measuring the Robustness of Cross-Encoder Rerankers." ECIR 2024.
- sentence-transformers CrossEncoder documentation: https://www.sbert.net/docs/cross_encoder/usage/usage.html
