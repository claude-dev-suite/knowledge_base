# Cross-Encoder Theory for Reranking

## TL;DR

Cross-encoders process query and document as a single concatenated input through a transformer, enabling full bidirectional attention between all tokens. This token-level interaction captures fine-grained relevance signals -- negation, qualification, word order, entity co-reference -- that bi-encoders miss because they encode query and document independently. The cost is O(N) inference per query (one forward pass per candidate document), making cross-encoders impractical for full-corpus search but ideal as rerankers over a pre-filtered candidate set. This article covers the architecture, why they outperform bi-encoders, training methodology, computational analysis, and the distinction between BERT-based and instruction-tuned cross-encoders.

---

## Architecture Deep Dive

### Input Format

A cross-encoder takes a single input sequence containing both the query and the document:

```
[CLS] query_token_1 query_token_2 ... query_token_m [SEP] doc_token_1 doc_token_2 ... doc_token_n [SEP]
```

The `[CLS]` token's final hidden state is passed through a linear classification head to produce a relevance score:

```
score = sigmoid(W * h_CLS + b)
```

Or for regression-trained models:

```
score = W * h_CLS + b    (no sigmoid, raw logit)
```

### Why Full Attention Matters

In a bi-encoder, the query attention mask and document attention mask are separate. Query tokens can only attend to other query tokens; document tokens can only attend to other document tokens. The similarity signal comes entirely from the single dot product between the two pooled vectors.

In a cross-encoder, every query token attends to every document token and vice versa. This enables:

1. **Token-level matching**: the model sees that "RLS" in the query aligns with "Row Level Security" in the document.
2. **Negation handling**: "authentication WITHOUT OAuth" -- the model sees "without" modifying "OAuth" in the context of the document's content about OAuth.
3. **Qualification sensitivity**: "Python 3.12 ONLY, not 3.11" -- the model can distinguish documents about 3.12 from those about 3.11.
4. **Semantic entailment**: the model can determine whether the document actually answers the query, not just whether they are topically related.
5. **Ordering sensitivity**: "convert JSON to CSV" vs "convert CSV to JSON" -- bi-encoders often produce identical embeddings for these.

### Attention Visualization

```
            query: "configure  RLS  without  OAuth"
            -------  ----  -------  -----
doc token   attend   attend attend  attend
"Row"       0.12     0.31   0.05    0.02
"Level"     0.08     0.45   0.03    0.01
"Security"  0.15     0.38   0.04    0.03
"does"      0.02     0.01   0.08    0.05
"not"       0.03     0.02   0.41    0.06
"require"   0.05     0.03   0.35    0.22
"OAuth"     0.04     0.02   0.28    0.61
```

The model learns that "without OAuth" in the query matches "does not require OAuth" in the document -- a signal that bi-encoders cannot capture.

---

## Mathematical Comparison: Bi-Encoder vs Cross-Encoder

### Bi-Encoder Information Flow

```
Query: [CLS] q1 q2 ... qm [SEP]
    --> Transformer_Q --> pool(h_CLS) --> q_vec in R^d

Doc: [CLS] d1 d2 ... dn [SEP]
    --> Transformer_D --> pool(h_CLS) --> d_vec in R^d

Score = cosine(q_vec, d_vec)
```

Information bottleneck: all query information is compressed into a single d-dimensional vector (d=768 typically). All document information is compressed into another single vector. The relevance judgment must be made from these two vectors alone.

**Information capacity**: 2 * d * 32 bits = ~49KB for d=768 (at fp32). This is a severe bottleneck for complex relevance judgments.

### Cross-Encoder Information Flow

```
Input: [CLS] q1 q2 ... qm [SEP] d1 d2 ... dn [SEP]
    --> Transformer (L layers, H heads)
    --> h_CLS captures full cross-attention information
    --> Linear(h_CLS) --> score

Attention at each layer:
    Every token attends to every other token.
    Total attention entries: (m + n + 3)^2 per head per layer.
```

**Information capacity**: the `[CLS]` token's hidden state has been informed by full bidirectional attention over the concatenated sequence. With 12 layers and 12 heads, there are 144 attention operations, each processing the full (m+n+3) sequence. The information available for the relevance decision is orders of magnitude greater.

### Why the Gap Is Fundamental

The bi-encoder bottleneck is not fixable by scaling:

| Improvement | Helps bi-encoder? | Helps cross-encoder? |
|-------------|-------------------|---------------------|
| Larger embedding dim (768 -> 1536) | Partially (diminishing returns above ~1024) | N/A (already sees everything) |
| Better training data | Yes, but ceiling exists | Yes, and ceiling is higher |
| Larger model (110M -> 335M params) | Marginal improvement | Significant improvement |
| Multi-vector representations (ColBERT) | Yes, narrows the gap | N/A |
| Instruction tuning | Moderate improvement | Significant improvement |

The only way to close the gap within the bi-encoder paradigm is to move toward multi-vector representations (ColBERT), which trade some of the encoding independence for token-level interaction at scoring time. See `colbert-as-reranker.md`.

---

## Training Cross-Encoders

### Training Objective

Most cross-encoders are trained with binary cross-entropy loss on (query, document, relevance_label) triples:

```python
import torch
import torch.nn as nn
from transformers import AutoModelForSequenceClassification, AutoTokenizer

model = AutoModelForSequenceClassification.from_pretrained(
    "bert-base-uncased",
    num_labels=1,  # Regression (single score)
)
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")

# Training loop (simplified)
optimizer = torch.optim.AdamW(model.parameters(), lr=2e-5)
loss_fn = nn.BCEWithLogitsLoss()

for query, document, label in training_data:
    inputs = tokenizer(
        query,
        document,
        padding=True,
        truncation=True,
        max_length=512,
        return_tensors="pt",
    )
    logits = model(**inputs).logits.squeeze(-1)
    loss = loss_fn(logits, torch.tensor([label], dtype=torch.float))
    loss.backward()
    optimizer.step()
    optimizer.zero_grad()
```

### Training Data Sources

| Dataset | Size | Domain | Usage |
|---------|------|--------|-------|
| MS MARCO Passage | 8.8M queries, 530K labeled | Web search | Most common pretraining set |
| Natural Questions | 307K queries | Wikipedia | Factual Q&A |
| TREC Deep Learning | 200 queries, deep judgments | Web search | Fine-grained evaluation |
| SQuAD | 100K questions | Wikipedia | Reading comprehension |
| Domain-specific | Varies | Your domain | Fine-tuning for best results |

### Hard Negative Mining

The quality of negative examples is critical. Random negatives are too easy; the model needs "hard" negatives that are topically related but not relevant.

```python
from sentence_transformers import CrossEncoder, InputExample

# Mine hard negatives using a bi-encoder
# For each query, retrieve top-100 with bi-encoder, label top-1-5 as positive,
# top-20-50 as hard negatives

training_examples = []
for query, positive_doc, hard_negative_doc in mined_triples:
    # Positive pair
    training_examples.append(InputExample(texts=[query, positive_doc], label=1.0))
    # Hard negative pair
    training_examples.append(InputExample(texts=[query, hard_negative_doc], label=0.0))
```

### Fine-Tuning on Domain Data

```python
from sentence_transformers import CrossEncoder
from sentence_transformers.cross_encoder.evaluation import CERerankingEvaluator

# Start from a pre-trained cross-encoder
model = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2", num_labels=1)

# Prepare domain-specific training data
train_samples = [
    InputExample(texts=["configure RLS", "PostgreSQL RLS policies..."], label=1.0),
    InputExample(texts=["configure RLS", "MySQL user management..."], label=0.0),
    # ... more domain-specific examples
]

# Evaluation data
eval_samples = {
    "query1": {"positive": ["rel_doc1"], "negative": ["irrel_doc1", "irrel_doc2"]},
}
evaluator = CERerankingEvaluator(eval_samples, name="domain-eval")

# Train
model.fit(
    train_dataloader=DataLoader(train_samples, batch_size=16, shuffle=True),
    evaluator=evaluator,
    epochs=3,
    warmup_steps=100,
    output_path="./domain-cross-encoder",
)
```

---

## BERT-Based vs. Instruction-Tuned Cross-Encoders

### BERT-Based (Traditional)

Models like `ms-marco-MiniLM-L-6-v2` and `ms-marco-MiniLM-L-12-v2` are BERT/MiniLM architectures fine-tuned on MS MARCO with binary relevance labels.

**Characteristics**:
- Score output is a raw logit (not bounded to [0,1] without sigmoid)
- Trained on web search queries -- may not generalize well to domain-specific queries
- No instruction following -- the model sees query+doc and produces a score
- Fast inference due to small model sizes (22-33M params)

### Instruction-Tuned Cross-Encoders

Newer models like `bge-reranker-v2-m3` and `jina-reranker-v2` are trained with instruction-tuning methodology:

1. Pre-trained on diverse text pairs across many tasks (NLI, STS, retrieval, classification)
2. Fine-tuned with instructions that describe the relevance criteria
3. Often multilingual from the start

**Characteristics**:
- Better zero-shot generalization to new domains
- Can handle multilingual queries and documents
- Larger models (278M-568M params)
- Scores are often better calibrated

### Comparison

```python
from sentence_transformers import CrossEncoder

# BERT-based (fast, good for English web-style queries)
bert_ce = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

# Instruction-tuned (better generalization, multilingual)
bge_ce = CrossEncoder("BAAI/bge-reranker-v2-m3")

query = "Comment configurer la securite au niveau des lignes?"  # French
doc = "PostgreSQL Row Level Security allows per-row access control..."

bert_score = bert_ce.predict([(query, doc)])   # May struggle with French
bge_score = bge_ce.predict([(query, doc)])     # Handles multilingual well
```

### When to Use Which

| Scenario | Recommended Model Type |
|----------|----------------------|
| English-only, web-style queries | BERT-based (MiniLM) for speed |
| Multilingual queries | Instruction-tuned (bge-reranker-v2-m3) |
| Domain-specific (will fine-tune) | BERT-based as starting point |
| Zero-shot on new domain | Instruction-tuned |
| Latency-critical (<100ms for 20 docs) | BERT-based (MiniLM-L-6) |
| Quality-critical (latency secondary) | Instruction-tuned (bge-reranker-v2-m3) |

---

## Computational Cost Analysis

### FLOPs per (query, doc) Pair

For a transformer with:
- `L` layers, `H` attention heads, `d_model` hidden size
- Input sequence length `S = len(query) + len(doc) + 3` (CLS + 2 SEP tokens)

```
FLOPs per pair ≈ 2 * L * (4 * d_model^2 * S + 2 * d_model * S^2)
```

For MiniLM-L-6 (L=6, d=384, S=256):
```
≈ 2 * 6 * (4 * 384^2 * 256 + 2 * 384 * 256^2)
≈ 2 * 6 * (150,994,944 + 50,331,648)
≈ 2.4 billion FLOPs per pair
```

For BGE-reranker-v2-m3 (L=24, d=1024, S=256):
```
≈ 2 * 24 * (4 * 1024^2 * 256 + 2 * 1024 * 256^2)
≈ 2 * 24 * (1,073,741,824 + 134,217,728)
≈ 58 billion FLOPs per pair
```

The 24x difference in FLOPs explains the latency gap between small and large cross-encoders.

### Memory Requirements

| Model | Params | FP32 Memory | FP16 Memory | INT8 Memory |
|-------|--------|-------------|-------------|-------------|
| MiniLM-L-6-v2 | 22M | 88MB | 44MB | 22MB |
| MiniLM-L-12-v2 | 33M | 132MB | 66MB | 33MB |
| bge-reranker-v2-m3 | 568M | 2.3GB | 1.1GB | 570MB |
| jina-reranker-v2 | 278M | 1.1GB | 556MB | 278MB |

### Optimization: Quantization

```python
from sentence_transformers import CrossEncoder
import torch

# Load with FP16 for ~2x speedup on GPU
model = CrossEncoder(
    "BAAI/bge-reranker-v2-m3",
    device="cuda",
    automodel_args={"torch_dtype": torch.float16},
)

# Or use ONNX Runtime for CPU optimization
from optimum.onnxruntime import ORTModelForSequenceClassification

ort_model = ORTModelForSequenceClassification.from_pretrained(
    "cross-encoder/ms-marco-MiniLM-L-6-v2",
    export=True,
)
# ~1.5-2x faster inference on CPU
```

---

## The Retrieve-Then-Rerank Paradigm in Detail

### Why Not Just Use Cross-Encoders Directly?

| Corpus Size | Cross-Encoder Time (20ms/pair) | Bi-Encoder + Rerank (top-50) |
|-------------|-------------------------------|------------------------------|
| 1,000 docs | 20 seconds | 35ms + 1s = 1.035s |
| 10,000 docs | 3.3 minutes | 40ms + 1s = 1.04s |
| 100,000 docs | 33 minutes | 50ms + 1s = 1.05s |
| 1,000,000 docs | 5.5 hours | 60ms + 1s = 1.06s |

Cross-encoder time scales linearly with corpus size. Bi-encoder retrieval time scales sub-linearly (logarithmic with ANN indexes). The two-stage approach makes cross-encoder quality accessible at bi-encoder speed.

### Optimal Pipeline Design

```python
import asyncio
from sentence_transformers import CrossEncoder


class TwoStageRetriever:
    def __init__(
        self,
        vector_store,
        reranker_model: str = "cross-encoder/ms-marco-MiniLM-L-6-v2",
        first_stage_k: int = 50,
        final_k: int = 5,
    ):
        self.vector_store = vector_store
        self.reranker = CrossEncoder(reranker_model)
        self.first_stage_k = first_stage_k
        self.final_k = final_k

    async def retrieve(self, query: str) -> list[dict]:
        # Stage 1: Fast bi-encoder retrieval
        candidates = await self.vector_store.asimilarity_search(
            query, k=self.first_stage_k
        )

        if not candidates:
            return []

        # Stage 2: Cross-encoder reranking
        pairs = [(query, doc.page_content) for doc in candidates]
        scores = self.reranker.predict(pairs, batch_size=32)

        # Combine and sort
        scored = list(zip(candidates, scores))
        scored.sort(key=lambda x: x[1], reverse=True)

        return [
            {
                "content": doc.page_content,
                "metadata": doc.metadata,
                "rerank_score": float(score),
            }
            for doc, score in scored[: self.final_k]
        ]
```

---

## Limitations of Cross-Encoders

1. **Cannot pre-compute document representations.** Every query requires N new forward passes. This is the fundamental limitation.

2. **Max sequence length.** Most models have a 512-token limit. For a 30-token query, only 479 tokens of the document are visible. Longer documents require chunking or truncation.

3. **Score calibration.** Scores from different queries are not directly comparable. A score of 0.8 for query A does not mean the same thing as 0.8 for query B. Use relative ordering, not absolute thresholds.

4. **Training data bias.** MS MARCO-trained models are biased toward web search queries. They may underperform on domain-specific content without fine-tuning.

5. **Latency at scale.** Even reranking 100 documents takes 0.5-2 seconds depending on model size. For real-time applications, K must be carefully chosen.

---

## Common Pitfalls

1. **Treating cross-encoder scores as probabilities.** Raw logit outputs are not bounded. Apply sigmoid only if the model was trained with BCEWithLogitsLoss. Check the model card.
2. **Truncating the wrong end.** When query+doc exceeds max length, ensure the query is fully preserved and the document is truncated. Never truncate the query.
3. **Not batching inference.** Processing pairs one at a time wastes GPU utilization. Always batch: `model.predict(all_pairs, batch_size=32)`.
4. **Fine-tuning without hard negatives.** Random negatives are too easy. The model learns nothing useful. Mine hard negatives from your bi-encoder's top results.
5. **Assuming cross-encoder always beats bi-encoder.** On well-represented domains (web search, Wikipedia), fine-tuned bi-encoders with hard-negative training can approach cross-encoder quality. Measure on your data.
6. **Ignoring the first-stage recall ceiling.** A cross-encoder reranker can only reorder documents that were retrieved in the first stage. If recall@K is 60%, the reranker's maximum achievable recall is also 60%.

---

## References

- Nogueira & Cho. "Passage Re-ranking with BERT" (arXiv:1901.04085, 2019)
- Humeau et al. "Poly-encoders: Architectures and Pre-training Strategies for Fast and Accurate Multi-sentence Scoring" (ICLR 2020)
- Thakur et al. "BEIR: A Heterogeneous Benchmark for Zero-shot Evaluation of Information Retrieval Models" (NeurIPS 2021)
- Reimers & Gurevych. "Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks" (EMNLP 2019)
- BAAI bge-reranker: https://huggingface.co/BAAI/bge-reranker-v2-m3
- Sentence Transformers cross-encoder docs: https://www.sbert.net/docs/cross_encoder/usage/usage.html
