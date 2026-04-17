# SPLADE -- Learned Sparse Retrieval

## Overview

SPLADE (SParse Lexical AnD Expansion model) is a family of neural retrieval models that produce **sparse** query and document representations -- vectors with mostly zero entries, where each dimension corresponds to a vocabulary token. Unlike BM25, which relies on exact term matching and hand-crafted statistics, SPLADE learns which terms matter and which terms to **expand** (inject related terms that do not appear in the original text). Unlike dense retrieval models (e.g., DPR, Contriever), SPLADE preserves the interpretability and exact-match strengths of sparse representations while adding semantic understanding.

The result is a model that combines the best properties of lexical and neural retrieval: it matches exact terms like BM25, understands synonyms and paraphrases like a dense model, and produces representations you can actually read and debug.

---

## How Sparse Vectors Differ from BM25

### BM25: Statistics-Based Sparse Vectors

BM25 represents documents as sparse vectors over the vocabulary, where each non-zero entry is a function of term frequency, inverse document frequency, and document length. The key limitation: a BM25 vector has non-zero entries **only for tokens that actually appear in the text**.

```
Query:  "how to fix a memory leak in Python"
BM25 vector (non-zero entries only):
  "how": 0.02,  "fix": 1.45,  "memory": 2.81,
  "leak": 3.12, "python": 1.90

Document: "Debugging RAM issues in CPython garbage collection"
BM25 vector:
  "debugging": 2.50, "ram": 3.20, "issues": 0.80,
  "cpython": 4.10, "garbage": 3.60, "collection": 1.90
```

BM25 gives this query-document pair a score of **zero** because they share no exact terms, even though the document is clearly relevant.

### SPLADE: Learned Sparse Vectors with Term Expansion

SPLADE passes text through a masked language model (MLM head of BERT/DistilBERT) to produce a sparse vector where:

1. **Original terms** receive learned weights (replacing BM25's TF-IDF formula)
2. **Expanded terms** -- tokens not in the original text but semantically related -- receive non-zero weights

```
Query:  "how to fix a memory leak in Python"
SPLADE vector (showing top entries):
  "memory": 3.10,  "leak": 3.45,  "python": 2.80,
  "fix": 1.90,     "debug": 1.20, "garbage": 0.85,
  "ram": 0.70,     "cpython": 0.55, "oom": 0.40,
  "allocation": 0.35, "profiling": 0.30

Document: "Debugging RAM issues in CPython garbage collection"
SPLADE vector:
  "debugging": 2.80, "ram": 3.50,   "cpython": 3.20,
  "garbage": 3.10,   "collection": 2.40, "memory": 1.80,
  "leak": 0.90,      "python": 1.50, "debug": 1.20,
  "issue": 1.10,     "gc": 0.85,    "heap": 0.60
```

Now the query and document share terms like "memory", "leak", "python", "garbage", "ram", "debug", and "cpython" -- even though most of these overlaps come from **expansion**, not from tokens present in both original texts.

### The Vocabulary-Aligned Representation

Both BM25 and SPLADE produce vectors of dimension V (vocabulary size, typically 30,522 for BERT's WordPiece). The crucial differences:

| Property | BM25 | SPLADE |
|---|---|---|
| Non-zero entries per document | Only tokens present in text | Present tokens + expanded terms |
| Weight computation | TF-IDF formula | Learned (MLM head + log-saturation) |
| Semantic understanding | None | Via MLM contextual predictions |
| Interpretability | Full (each entry = a real token) | Full (each entry = a real token) |
| Training required | No | Yes (contrastive or distillation) |
| Can match synonyms | No | Yes (via expansion) |
| Average sparsity (% zero) | ~99.5% | ~99.0-99.5% |

---

## The SPLADE Architecture

### From MLM Logits to Sparse Weights

SPLADE uses a transformer encoder (typically DistilBERT or BERT-base) with its masked language model head. For an input text, the model:

1. Tokenizes and encodes the text through the transformer, producing contextual embeddings for each token position
2. Passes each token embedding through the MLM head, producing a logit vector of dimension V (one score per vocabulary token) for each input position
3. Aggregates across positions using max-pooling: for each vocabulary token j, take the maximum logit across all input positions
4. Applies a log-saturation function to prevent any single term from dominating

```python
import torch
import torch.nn as nn
from transformers import AutoModelForMaskedLM, AutoTokenizer


class SPLADEEncoder(nn.Module):
    """
    Minimal SPLADE encoder that produces sparse vocabulary-sized vectors.
    """

    def __init__(self, model_name: str = "distilbert-base-uncased"):
        super().__init__()
        self.model = AutoModelForMaskedLM.from_pretrained(model_name)

    def forward(
        self,
        input_ids: torch.Tensor,
        attention_mask: torch.Tensor,
    ) -> torch.Tensor:
        """
        Returns a sparse vector of shape (batch_size, vocab_size).
        Each entry is a non-negative weight for the corresponding token.
        """
        # Step 1-2: get MLM logits for every input position
        # Shape: (batch_size, seq_len, vocab_size)
        output = self.model(input_ids=input_ids, attention_mask=attention_mask)
        logits = output.logits

        # Step 3: max-pool across sequence positions
        # Mask out padding positions by setting their logits to -inf
        logits[attention_mask.unsqueeze(-1).expand_as(logits) == 0] = float("-inf")
        max_logits, _ = torch.max(logits, dim=1)  # (batch_size, vocab_size)

        # Step 4: log-saturation to produce non-negative weights
        # log(1 + ReLU(x)) grows slowly, preventing any term from dominating
        sparse_vec = torch.log1p(torch.relu(max_logits))  # (batch_size, vocab_size)

        return sparse_vec
```

### Why Max-Pooling?

Each input position produces a distribution over the entire vocabulary via the MLM head. The token at position 3 ("RAM") might assign high probability to vocabulary tokens like "memory", "ram", "heap", "storage". The token at position 7 ("CPython") might assign high probability to "python", "cpython", "interpreter".

Max-pooling across positions means: "for each vocabulary token, take the highest prediction from any input position." This naturally captures expansion -- the model "predicts" related terms at positions where they are contextually appropriate.

### Log-Saturation: log(1 + ReLU(x))

The log-saturation function serves two purposes:

1. **ReLU** zeros out negative logits (tokens the model considers irrelevant), maintaining sparsity
2. **log(1 + x)** compresses large values, preventing a single high-confidence term from overwhelming the dot-product score

Without log-saturation, a document about "Python" might have a weight of 50 for "python" and 2 for everything else, making the representation effectively single-dimensional.

---

## SPLADE++ and FLOPS Regularization

### The Sparsity Problem

The raw SPLADE architecture produces vectors that are too dense. Without regularization, the model learns to assign small but non-zero weights to hundreds or thousands of vocabulary tokens. This destroys the efficiency advantage of sparse retrieval -- if vectors are not sparse, you cannot use inverted indexes efficiently.

### FLOPS Regularization

SPLADE++ introduces FLOPS (FLOating Point operations Per Second) regularization, which penalizes the expected number of floating-point operations needed to compute query-document dot products:

```
L_FLOPS = SUM_{j=1}^{V} ( E_q[w_j^q] * E_d[w_j^d] )
```

Where:
- `w_j^q` is the weight of vocabulary token j in the query representation
- `w_j^d` is the weight of vocabulary token j in the document representation
- `E_q` and `E_d` are expectations over queries and documents in the training batch

The intuition: if a vocabulary token has high average weight in both queries and documents, it contributes to many dot products and costs many FLOPs. Penalizing this encourages the model to keep weights low (or zero) for most tokens.

```python
def flops_loss(
    query_vecs: torch.Tensor,
    doc_vecs: torch.Tensor,
) -> torch.Tensor:
    """
    FLOPS regularization loss.
    query_vecs: (batch_size, vocab_size) -- sparse query representations
    doc_vecs:   (batch_size, vocab_size) -- sparse document representations
    """
    # Average activation per vocabulary token across the batch
    avg_query = torch.mean(query_vecs, dim=0)   # (vocab_size,)
    avg_doc = torch.mean(doc_vecs, dim=0)        # (vocab_size,)

    # FLOPS = sum of products of average activations
    flops = torch.sum(avg_query * avg_doc)
    return flops


def splade_training_loss(
    query_vecs: torch.Tensor,
    doc_pos_vecs: torch.Tensor,
    doc_neg_vecs: torch.Tensor,
    lambda_q: float = 3e-4,
    lambda_d: float = 1e-4,
) -> torch.Tensor:
    """
    Combined contrastive + FLOPS regularization loss.
    """
    # Contrastive loss (InfoNCE / in-batch negatives)
    scores_pos = torch.sum(query_vecs * doc_pos_vecs, dim=-1)     # (batch,)
    scores_neg = torch.sum(
        query_vecs.unsqueeze(1) * doc_neg_vecs.unsqueeze(0), dim=-1,
    )  # (batch, batch) -- in-batch negatives
    all_scores = torch.cat(
        [scores_pos.unsqueeze(1), scores_neg], dim=1,
    )  # (batch, 1+batch)
    labels = torch.zeros(query_vecs.size(0), dtype=torch.long, device=query_vecs.device)
    contrastive = nn.CrossEntropyLoss()(all_scores, labels)

    # FLOPS regularization -- separate for queries and documents
    # Penalize query sparsity and document sparsity independently
    flops_q = torch.mean(torch.sum(query_vecs ** 2, dim=-1))
    flops_d = torch.mean(torch.sum(doc_pos_vecs ** 2, dim=-1))

    return contrastive + lambda_q * flops_q + lambda_d * flops_d
```

### SPLADE++ Variants

| Variant | Description | Avg non-zero entries | MRR@10 (MS MARCO) |
|---|---|---|---|
| SPLADE-max | Max-pooling, no distillation | ~120-180 per doc | 0.340 |
| SPLADE-doc | Expansion only on documents, queries are bag-of-words | ~200 per doc | 0.322 |
| SPLADE++ Self-Distillation | Distillation from a strong SPLADE to a sparser one | ~40-90 per doc | 0.338 |
| SPLADE++ CoCondenser | Initialized from CoCondenser, distillation + FLOPS | ~30-60 per doc | 0.340 |
| Efficient SPLADE V | Aggressive FLOPS, very sparse | ~20-40 per doc | 0.330 |

The CoCondenser variant is the most widely used today. It achieves top-tier retrieval quality with only 30-60 non-zero entries per document, making it practical for inverted-index-based retrieval.

---

## Term Expansion in Practice

### What Does Expansion Look Like?

To make SPLADE's behavior concrete, here is what expansion produces for different types of text:

```python
from transformers import AutoModelForMaskedLM, AutoTokenizer
import torch


def get_splade_representation(text: str, model, tokenizer, top_k: int = 20):
    """
    Encode text with SPLADE and return the top-k weighted terms.
    """
    inputs = tokenizer(text, return_tensors="pt", truncation=True, max_length=256)
    with torch.no_grad():
        output = model(**inputs)
    logits = output.logits

    # Max-pool and log-saturate
    attention_mask = inputs["attention_mask"]
    logits[attention_mask.unsqueeze(-1).expand_as(logits) == 0] = float("-inf")
    max_logits, _ = torch.max(logits, dim=1)
    sparse_vec = torch.log1p(torch.relu(max_logits)).squeeze(0)

    # Decode top-k terms
    top_indices = torch.topk(sparse_vec, k=top_k).indices.tolist()
    top_weights = torch.topk(sparse_vec, k=top_k).values.tolist()

    terms = []
    for idx, weight in zip(top_indices, top_weights):
        token = tokenizer.decode([idx]).strip()
        terms.append((token, round(weight, 3)))
    return terms


# Example outputs (from naver/splade-cocondenser-ensembledistil):
#
# Input: "kubernetes pod eviction"
# Top terms: [
#   ("kubernetes", 3.85), ("pod", 3.42), ("eviction", 3.91),
#   ("k8s", 1.80), ("container", 1.45), ("node", 1.20),
#   ("cluster", 1.05), ("docker", 0.80), ("deployment", 0.75),
#   ("scheduling", 0.60), ("orchestration", 0.55), ("kube", 0.50),
# ]
#
# Input: "React useEffect cleanup function"
# Top terms: [
#   ("react", 3.50), ("useeffect", 3.20), ("cleanup", 3.10),
#   ("function", 2.40), ("hook", 1.80), ("component", 1.30),
#   ("lifecycle", 1.10), ("unmount", 0.95), ("effect", 0.90),
#   ("render", 0.75), ("javascript", 0.60), ("return", 0.50),
# ]
```

### Expansion Strengths

1. **Synonym bridging**: "car" expands to include "vehicle", "automobile", "auto"
2. **Abbreviation linking**: "k8s" appears in expansion of "kubernetes"
3. **Concept association**: "memory leak" expands to "garbage", "heap", "allocation", "oom"
4. **Hierarchical terms**: "PostgreSQL" expands to "database", "sql", "postgres", "relational"

### Expansion Risks

1. **Topic drift**: aggressive expansion can add weakly related terms that cause false matches
2. **Ambiguity propagation**: "python" (language) might expand to "snake", "reptile" in some contexts
3. **Stopword leakage**: common terms like "the", "is", "and" may receive small but non-zero weights

FLOPS regularization directly controls these risks by forcing the model to be selective about which terms to expand.

---

## Scoring: Sparse Dot Product

The query-document relevance score is simply the dot product of the two sparse vectors:

```
Score(Q, D) = SUM_{j in V}  w_j^Q * w_j^D
```

Because both vectors are sparse (only 30-200 non-zero entries out of 30,522), this sum involves very few terms. The computation is equivalent to intersecting two posting lists in an inverted index, which is exactly why SPLADE is compatible with traditional search infrastructure.

```python
def score_sparse(
    query_vec: dict[int, float],
    doc_vec: dict[int, float],
) -> float:
    """
    Dot product of two sparse vectors represented as {token_id: weight}.
    Only iterates over the intersection of non-zero entries.
    """
    score = 0.0
    # Iterate over the smaller vector for efficiency
    if len(query_vec) > len(doc_vec):
        query_vec, doc_vec = doc_vec, query_vec

    for token_id, q_weight in query_vec.items():
        if token_id in doc_vec:
            score += q_weight * doc_vec[token_id]

    return score
```

This is identical in structure to how BM25 scoring works in Lucene -- iterate over query terms, look up the posting list, multiply weights. SPLADE can therefore be deployed on existing inverted index infrastructure with minimal changes.

---

## Training Pipeline

### Data Requirements

SPLADE is typically trained on MS MARCO passage ranking:
- 8.8M passages
- 500K training queries with one positive passage each
- Hard negatives mined from BM25 or a dense retriever

### Training Stages

1. **Initialize** from a pretrained MLM (BERT-base, DistilBERT, or CoCondenser)
2. **Stage 1: Contrastive training** with in-batch negatives and BM25 hard negatives, mild FLOPS regularization
3. **Stage 2: Distillation** from a cross-encoder teacher (e.g., cross-encoder/ms-marco-MiniLM-L-12-v2), stronger FLOPS regularization
4. **Stage 3 (optional): Self-distillation** from the Stage 2 model to a smaller or sparser student

```python
from transformers import AutoModelForMaskedLM, AutoTokenizer
from torch.utils.data import DataLoader
import torch


def train_splade_one_epoch(
    model: torch.nn.Module,
    dataloader: DataLoader,
    optimizer: torch.optim.Optimizer,
    lambda_q: float = 3e-4,
    lambda_d: float = 1e-4,
    device: str = "cuda",
):
    """
    One epoch of SPLADE training with contrastive loss + FLOPS regularization.
    Each batch contains (query, positive_doc, negative_doc) triplets.
    """
    model.train()
    total_loss = 0.0

    for batch in dataloader:
        query_ids = batch["query_ids"].to(device)
        query_mask = batch["query_mask"].to(device)
        pos_ids = batch["pos_ids"].to(device)
        pos_mask = batch["pos_mask"].to(device)
        neg_ids = batch["neg_ids"].to(device)
        neg_mask = batch["neg_mask"].to(device)

        # Encode
        q_vec = model(query_ids, query_mask)       # (B, V)
        p_vec = model(pos_ids, pos_mask)            # (B, V)
        n_vec = model(neg_ids, neg_mask)            # (B, V)

        # In-batch contrastive (positive + all negatives in batch)
        pos_scores = torch.sum(q_vec * p_vec, dim=-1, keepdim=True)   # (B, 1)
        neg_scores = torch.mm(q_vec, n_vec.t())                       # (B, B)
        all_scores = torch.cat([pos_scores, neg_scores], dim=1)       # (B, 1+B)
        labels = torch.zeros(q_vec.size(0), dtype=torch.long, device=device)
        ce_loss = torch.nn.CrossEntropyLoss()(all_scores, labels)

        # FLOPS regularization
        flops_q = torch.mean(torch.sum(q_vec ** 2, dim=-1))
        flops_d = torch.mean(torch.sum(p_vec ** 2, dim=-1))
        flops_d += torch.mean(torch.sum(n_vec ** 2, dim=-1))
        flops_d /= 2.0

        loss = ce_loss + lambda_q * flops_q + lambda_d * flops_d

        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

        total_loss += loss.item()

    return total_loss / len(dataloader)
```

---

## When to Use SPLADE

### SPLADE Wins

- **Domain-specific retrieval** where exact terminology matters (legal, medical, code): SPLADE preserves exact term matching while adding semantic expansion
- **Hybrid is not an option**: when you cannot run both BM25 and a dense retriever (infrastructure constraint), SPLADE alone covers both needs
- **Interpretability required**: you can inspect exactly which terms matched and with what weights, making debugging straightforward
- **Existing inverted index infrastructure**: SPLADE plugs into Elasticsearch, Qdrant (sparse vectors), Vespa, or any system with inverted index support

### SPLADE Loses

- **Extremely low latency requirements**: SPLADE query encoding requires a transformer forward pass (~5-15ms on GPU), whereas BM25 query processing is sub-millisecond
- **No GPU available for encoding**: while retrieval can use CPU-based inverted indexes, document and query encoding at scale requires GPU
- **Very short texts** (1-3 words): expansion has little context to work with, and BM25 or dense retrieval may be more reliable
- **Multilingual retrieval**: SPLADE models are primarily English; multilingual support is limited compared to multilingual dense models like mE5 or mContriever

---

## Comparison with Dense Retrieval

| Property | SPLADE | Dense (e.g., E5, Contriever) |
|---|---|---|
| Representation | Sparse, vocab-sized | Dense, 768-1024 dim |
| Exact term matching | Excellent | Weak (terms averaged into single vector) |
| Semantic matching | Good (via expansion) | Excellent (full semantic compression) |
| Index type | Inverted index | ANN (HNSW, IVF) |
| Storage per document | ~200-800 bytes (sparse entries) | ~3-4 KB (768 float32) |
| Query latency (index lookup) | ~5-20ms | ~5-20ms |
| Encoding latency (per query) | ~5-15ms (GPU) | ~5-15ms (GPU) |
| Interpretability | High (see which terms matched) | Low (opaque embedding) |
| Hybrid compatibility | Natural (already sparse) | Requires separate BM25 stage |
| Out-of-domain generalization | Good (lexical backbone) | Variable (depends on training data) |

### The Hybrid Alternative

Many production systems combine BM25 + dense retrieval (with reciprocal rank fusion or linear interpolation). SPLADE can be seen as a single model that approximates this hybrid: it does BM25-like exact matching and dense-like semantic matching in one representation. However, the best absolute quality often comes from a three-way hybrid: BM25 + dense + SPLADE.

---

## Common Pitfalls

1. **Forgetting to quantize weights**: raw float32 sparse vectors waste storage. In production, quantize SPLADE weights to float16 or even int8 with minimal quality loss.

2. **Indexing without FLOPS regularization**: models trained without FLOPS produce vectors that are too dense for efficient inverted index retrieval. Always use a SPLADE++ variant.

3. **Using SPLADE for re-ranking**: SPLADE is a first-stage retriever. For re-ranking, cross-encoders or ColBERT are more effective because they allow deeper query-document interaction.

4. **Ignoring the MLM head initialization**: SPLADE's quality depends heavily on the MLM head being well-calibrated. Using a randomly initialized MLM head (instead of the pretrained one) significantly degrades expansion quality.

5. **Not benchmarking against BM25 + dense hybrid**: SPLADE's advantage is simplicity (one model instead of two). If you are already running a hybrid system, adding SPLADE may provide diminishing returns.

6. **Assuming SPLADE handles all languages**: most SPLADE models are English-only (trained on MS MARCO English). For multilingual use cases, verify that a multilingual SPLADE checkpoint exists or train your own.

---

## Key Models

| Model | Base | Params | Avg BEIR NDCG@10 | MS MARCO MRR@10 |
|---|---|---|---|---|
| naver/splade-cocondenser-ensembledistil | DistilBERT | 66M | 0.500 | 0.340 |
| naver/splade-cocondenser-selfdistil | DistilBERT | 66M | 0.497 | 0.338 |
| naver/splade_v2_max | DistilBERT | 66M | 0.478 | 0.340 |
| naver/splade_v2_distil | DistilBERT | 66M | 0.471 | 0.335 |
| prithivida/Splade_PP_en_v1 | BERT-base | 110M | 0.505 | 0.342 |

---

## References

- Formal, T. et al. "SPLADE: Sparse Lexical and Expansion Model for First Stage Ranking." SIGIR 2021.
- Formal, T. et al. "SPLADE v2: Sparse Lexical and Expansion Model for Information Retrieval." arXiv 2021.
- Formal, T. et al. "From Distillation to Hard Negative Sampling: Making Sparse Neural IR Models More Effective." SIGIR 2022 (SPLADE++).
- Lassance, C. and Clinchant, S. "An Efficiency Study for SPLADE Models." SIGIR 2022.
- Gao, L. and Callan, J. "Unsupervised Corpus Aware Language Model Pre-training for Dense Passage Retrieval." ACL 2022 (CoCondenser).
- SPLADE repository: https://github.com/naver/splade
