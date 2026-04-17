# Matryoshka Representation Learning -- Comprehensive Guide

## Overview / TL;DR

Matryoshka Representation Learning (MRL) trains embedding models so that prefixes of the full embedding vector are themselves useful embeddings at lower dimensions. Named after Russian nesting dolls, an MRL-trained model produces a 3072-dimensional embedding where the first 1024, 512, 256, or even 64 dimensions form valid, meaningful embeddings. This eliminates the traditional trade-off between quality and efficiency -- you train once at full dimensions and deploy at whatever dimension your constraints allow. This guide covers the paper's key insight, how MRL training works, which models support it, and practical deployment patterns.

---

## The Problem MRL Solves

Traditional embedding models produce fixed-size vectors. If you need 256-dim embeddings for a latency-constrained application, you must:

1. Train (or find) a model that natively produces 256-dim embeddings.
2. Accept that 256-dim models are inherently less capable than 1024-dim models.
3. If you later need 1024-dim for a precision-critical use case, train a separate model and re-embed everything.

This leads to:
- **Model proliferation**: Different models for different use cases.
- **Re-embedding cost**: Switching dimensions means re-embedding the entire corpus.
- **Suboptimal quality**: A model trained at 256 dims cannot capture as much information as one trained at 1024 dims.

MRL solves all three problems. A single model produces embeddings that work at any prefix length, with quality that gracefully degrades as dimensions decrease.

---

## How MRL Training Works

### The Key Insight

The paper's insight is simple but powerful: if you add loss terms that evaluate the embedding at multiple truncation points during training, the model learns to pack the most important information into the first dimensions.

### Training Procedure

1. Start with a standard embedding model architecture.
2. During each training step, compute the full embedding (e.g., 1024 dims).
3. Truncate the embedding to multiple prefix lengths: [64, 128, 256, 512, 1024].
4. Compute the training loss at each truncation point.
5. Sum the losses (optionally with weights).
6. Backpropagate through the summed loss.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F


class MatryoshkaLoss(nn.Module):
    """Multi-resolution loss for Matryoshka Representation Learning.

    Computes the training loss at multiple embedding dimensions
    and sums them so the model learns useful representations at every size.
    """

    def __init__(
        self,
        base_loss: nn.Module,
        dimensions: list[int] = [64, 128, 256, 512, 1024],
        weights: list[float] | None = None,
    ):
        super().__init__()
        self.base_loss = base_loss
        self.dimensions = sorted(dimensions)
        self.weights = weights or [1.0] * len(dimensions)

    def forward(
        self,
        embeddings_a: torch.Tensor,  # (batch, full_dims)
        embeddings_b: torch.Tensor,  # (batch, full_dims)
        labels: torch.Tensor,
    ) -> torch.Tensor:
        total_loss = torch.tensor(0.0, device=embeddings_a.device)

        for dim, weight in zip(self.dimensions, self.weights):
            # Truncate to prefix
            trunc_a = F.normalize(embeddings_a[:, :dim], p=2, dim=1)
            trunc_b = F.normalize(embeddings_b[:, :dim], p=2, dim=1)

            # Compute loss at this dimension
            loss = self.base_loss(trunc_a, trunc_b, labels)
            total_loss += weight * loss

        return total_loss / sum(self.weights)
```

### Training with sentence-transformers

sentence-transformers supports MRL training natively via `MatryoshkaLoss`:

```python
from sentence_transformers import SentenceTransformer, InputExample, losses
from torch.utils.data import DataLoader


def train_matryoshka_model(
    base_model: str = "BAAI/bge-base-en-v1.5",
    train_examples: list[InputExample] = None,
    output_path: str = "./matryoshka-model",
    dimensions: list[int] = [64, 128, 256, 512, 768],
    epochs: int = 3,
    batch_size: int = 64,
):
    """Fine-tune an embedding model with Matryoshka loss.

    This wraps any base loss (e.g., MNRL) with multi-resolution evaluation
    so the model learns useful prefixes at every dimension in the list.
    """
    model = SentenceTransformer(base_model)

    train_dataloader = DataLoader(
        train_examples,
        shuffle=True,
        batch_size=batch_size,
    )

    # Wrap MNRL with Matryoshka
    base_loss = losses.MultipleNegativesRankingLoss(model=model)
    train_loss = losses.MatryoshkaLoss(
        model=model,
        loss=base_loss,
        matryoshka_dims=dimensions,
        matryoshka_weights=[1] * len(dimensions),  # Equal weight per dimension
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

### Why It Works: Information Packing

Without MRL, embedding dimensions encode information in a distributed fashion -- important information may be spread across all dimensions. With MRL, the multi-scale loss forces the model to prioritize: the most important semantic distinctions are encoded in the first few dimensions, with progressively finer distinctions in later dimensions.

Think of it as progressive encoding:
- **Dims 1-64**: Topic-level distinctions (is this about medicine, law, or engineering?).
- **Dims 65-256**: Subtopic distinctions (within medicine: cardiology vs oncology).
- **Dims 257-512**: Fine-grained distinctions (within cardiology: diagnosis vs treatment).
- **Dims 513-1024+**: Very fine distinctions (specific drugs, dosages, side effects).

---

## Models with MRL Support

### Production-Ready MRL Models

| Model | Full Dims | MRL Range | Access Method | Notes |
|-------|----------|-----------|---------------|-------|
| text-embedding-3-small | 1536 | 256-1536 | API `dimensions` param | OpenAI |
| text-embedding-3-large | 3072 | 256-3072 | API `dimensions` param | OpenAI |
| Voyage-3 | 1024 | 256-1024 | API `output_dimension` param | Voyage AI |
| Voyage-3-large | 2048 | 256-2048 | API `output_dimension` param | Voyage AI |
| nomic-embed-text-v1.5 | 768 | 64-768 | Manual truncation + normalize | Open source |
| mxbai-embed-large-v1 | 1024 | 256-1024 | Manual truncation + normalize | Open source |
| Jina v3 | 1024 | 64-1024 | API `dimensions` param | Jina AI |

### Non-MRL Models (Truncation Will Fail)

| Model | Dims | Why No MRL |
|-------|------|-----------|
| BGE-M3 | 1024 | Trained without multi-scale loss |
| bge-base-en-v1.5 | 768 | Standard training |
| bge-large-en-v1.5 | 1024 | Standard training |
| E5-Mistral-7B | 4096 | Standard training |
| all-MiniLM-L6-v2 | 384 | Standard training |

**Important**: If you truncate a non-MRL model's embeddings, quality will degrade much more sharply than with an MRL model. For non-MRL models, the only way to get lower dimensions is to train a smaller model or apply PCA/random projection after embedding (both inferior to native MRL).

---

## Quality at Different Dimensions

### Empirical Results

The following table shows MTEB retrieval scores at different Matryoshka dimensions, demonstrating the graceful degradation characteristic of MRL models:

**text-embedding-3-large (native 3072)**:

| Dims | MTEB Retrieval | % of Full | Storage per 1M | Speed Improvement |
|------|---------------|-----------|----------------|-------------------|
| 3072 | 66.1 | 100.0% | 11.4 GB | 1.0x (baseline) |
| 2048 | 65.8 | 99.5% | 7.6 GB | 1.5x |
| 1536 | 65.5 | 99.1% | 5.7 GB | 2.0x |
| 1024 | 65.0 | 98.3% | 3.8 GB | 3.0x |
| 768 | 64.5 | 97.6% | 2.9 GB | 4.0x |
| 512 | 63.8 | 96.5% | 1.9 GB | 6.0x |
| 256 | 61.5 | 93.0% | 1.0 GB | 12.0x |

**nomic-embed-text-v1.5 (native 768)**:

| Dims | MTEB Retrieval | % of Full | Storage per 1M | Speed Improvement |
|------|---------------|-----------|----------------|-------------------|
| 768 | 62.8 | 100.0% | 2.9 GB | 1.0x |
| 512 | 62.1 | 98.9% | 1.9 GB | 1.5x |
| 384 | 61.2 | 97.5% | 1.4 GB | 2.0x |
| 256 | 60.5 | 96.3% | 1.0 GB | 3.0x |
| 128 | 57.2 | 91.1% | 0.5 GB | 6.0x |
| 64 | 52.1 | 83.0% | 0.25 GB | 12.0x |

### The Quality Cliff

Every MRL model has a "quality cliff" -- a dimension below which quality drops sharply. This cliff depends on the model architecture and training data:

| Model | Quality Cliff | Recommendation |
|-------|-------------|----------------|
| text-embedding-3-large | ~256 dims | Do not go below 256 |
| text-embedding-3-small | ~256 dims | Do not go below 256 |
| nomic-embed-text-v1.5 | ~128 dims | Do not go below 128 |
| mxbai-embed-large-v1 | ~256 dims | Do not go below 256 |
| Jina v3 | ~128 dims | Do not go below 128 |

Above the cliff, quality degrades at roughly 0.5-1.5% per halving of dimensions. Below the cliff, quality can drop 5-10% per halving.

---

## Deployment Patterns

### Pattern 1: Static Dimension Selection

Choose one dimension at deployment time and use it consistently for all queries and documents.

```python
from openai import OpenAI

client = OpenAI()

# Decision: 512 dims for our use case (good quality, low storage)
EMBEDDING_DIMS = 512

def embed_for_indexing(texts: list[str]) -> list[list[float]]:
    """Embed documents at the chosen dimension for indexing."""
    response = client.embeddings.create(
        model="text-embedding-3-large",
        input=texts,
        dimensions=EMBEDDING_DIMS,
    )
    return [item.embedding for item in response.data]


def embed_for_query(text: str) -> list[float]:
    """Embed a query at the same dimension."""
    response = client.embeddings.create(
        model="text-embedding-3-large",
        input=[text],
        dimensions=EMBEDDING_DIMS,
    )
    return response.data[0].embedding
```

### Pattern 2: Multi-Resolution Index

Store embeddings at full dimension and truncate at query time for speed, then re-score at full dimension for precision.

```python
import numpy as np


class MultiResolutionSearch:
    """Two-stage search using Matryoshka embeddings.

    Stage 1: Fast search with truncated (256-dim) embeddings.
    Stage 2: Re-score candidates with full (1024-dim) embeddings.
    """

    def __init__(self, full_embeddings: np.ndarray, doc_ids: list[str]):
        """
        Args:
            full_embeddings: (n_docs, full_dims) normalized embeddings.
            doc_ids: Corresponding document IDs.
        """
        self.full_embeddings = full_embeddings
        self.doc_ids = doc_ids

        # Pre-compute truncated embeddings for fast search
        self.truncated = full_embeddings[:, :256].copy()
        norms = np.linalg.norm(self.truncated, axis=1, keepdims=True)
        self.truncated /= np.maximum(norms, 1e-12)

    def search(
        self,
        query_embedding: np.ndarray,
        top_k: int = 10,
        candidate_pool: int = 500,
    ) -> list[tuple[str, float]]:
        """Two-stage Matryoshka search.

        Stage 1 uses 256 dims for ~12x faster candidate generation.
        Stage 2 uses full dims for precise re-scoring of candidates.
        """
        # Stage 1: Fast search with 256 dims
        q_truncated = query_embedding[:256]
        q_truncated = q_truncated / np.linalg.norm(q_truncated)
        fast_scores = np.dot(self.truncated, q_truncated)
        candidate_indices = np.argpartition(fast_scores, -candidate_pool)[-candidate_pool:]

        # Stage 2: Re-score with full dims
        q_full = query_embedding / np.linalg.norm(query_embedding)
        candidate_embs = self.full_embeddings[candidate_indices]
        precise_scores = np.dot(candidate_embs, q_full)

        # Sort and return top-k
        top_within_candidates = np.argsort(precise_scores)[::-1][:top_k]
        results = []
        for idx in top_within_candidates:
            original_idx = candidate_indices[idx]
            results.append((
                self.doc_ids[original_idx],
                float(precise_scores[idx]),
            ))

        return results
```

### Pattern 3: Adaptive Dimensions by Query Type

Use different dimensions based on query characteristics.

```python
def select_dimensions(query: str, latency_budget_ms: float = 50.0) -> int:
    """Select embedding dimensions based on query and latency budget.

    Short, simple queries work fine with fewer dimensions.
    Complex, nuanced queries benefit from more dimensions.
    """
    query_tokens = len(query.split())

    # Latency-constrained: use fewer dims
    if latency_budget_ms < 10:
        return 256

    # Short, specific queries: fewer dims suffice
    if query_tokens < 5:
        return 512

    # Medium queries: standard dims
    if query_tokens < 15:
        return 768

    # Long, complex queries: more dims for nuance
    return 1024
```

---

## How to Verify MRL Support

If you are unsure whether a model supports Matryoshka truncation, run this test:

```python
import numpy as np
from sentence_transformers import SentenceTransformer


def verify_matryoshka_support(
    model_name: str,
    test_dims: list[int] = [64, 128, 256, 512],
) -> dict:
    """Test whether a model produces useful embeddings when truncated.

    A model with MRL support will maintain high similarity between
    semantically similar texts even at low dimensions.
    A model without MRL will show sharp quality drops.
    """
    model = SentenceTransformer(model_name, trust_remote_code=True)
    full_dim = model.get_sentence_embedding_dimension()

    # Test pairs: semantically similar vs different
    similar_a = "Machine learning algorithms learn patterns from data."
    similar_b = "ML models find structure in training datasets."
    different = "The restaurant serves excellent Italian cuisine."

    embs = model.encode([similar_a, similar_b, different], normalize_embeddings=True)

    results = {"full_dim": full_dim, "dims": {}}

    for dim in [full_dim] + test_dims:
        truncated = embs[:, :dim]
        norms = np.linalg.norm(truncated, axis=1, keepdims=True)
        truncated = truncated / np.maximum(norms, 1e-12)

        sim_positive = float(np.dot(truncated[0], truncated[1]))
        sim_negative = float(np.dot(truncated[0], truncated[2]))
        margin = sim_positive - sim_negative

        results["dims"][dim] = {
            "sim_positive": sim_positive,
            "sim_negative": sim_negative,
            "margin": margin,
        }
        print(f"  dims={dim:4d}: sim_pos={sim_positive:.4f}, sim_neg={sim_negative:.4f}, margin={margin:.4f}")

    # Check for MRL: margin should degrade gracefully
    full_margin = results["dims"][full_dim]["margin"]
    min_margin = min(d["margin"] for d in results["dims"].values())

    if min_margin > full_margin * 0.5:
        print(f"\nModel likely supports MRL (min margin {min_margin:.4f} > 50% of full {full_margin:.4f})")
    else:
        print(f"\nModel likely does NOT support MRL (min margin {min_margin:.4f} << full {full_margin:.4f})")

    return results


# Test models
verify_matryoshka_support("nomic-ai/nomic-embed-text-v1.5")  # Has MRL
verify_matryoshka_support("BAAI/bge-base-en-v1.5")           # No MRL
```

---

## Common Pitfalls

1. **Truncating non-MRL models.** If the model was not trained with Matryoshka loss, truncating embeddings produces poor results. Always verify MRL support first.
2. **Forgetting to re-normalize after truncation.** Truncated embeddings are NOT unit-length even if the full embedding was. Always L2-normalize after truncation.
3. **Going below the quality cliff.** Just because a model supports 64-dim truncation does not mean 64 dims are useful for your task. Test quality at your target dimension.
4. **Using different dimensions for queries and documents.** Both must use the same dimension. A 256-dim query cannot be compared to a 1024-dim document.
5. **Assuming MRL is free.** MRL training adds a small overhead (~5-10% slower training) and can reduce full-dimension quality by 0.5-1% compared to standard training. This is usually a worthwhile trade-off.

---

## References

- Kusupati et al., "Matryoshka Representation Learning" (2022) -- https://arxiv.org/abs/2205.13147
- OpenAI Embeddings: dimensions parameter -- https://platform.openai.com/docs/guides/embeddings
- Nomic Embed Matryoshka -- https://blog.nomic.ai/posts/nomic-embed-matryoshka-v1.5
- sentence-transformers MatryoshkaLoss -- https://www.sbert.net/docs/package_reference/losses.html#matryoshkaloss
- HuggingFace Matryoshka Blog -- https://huggingface.co/blog/matryoshka
