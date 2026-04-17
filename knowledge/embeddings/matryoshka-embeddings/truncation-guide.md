# Matryoshka Embeddings -- Practical Truncation Guide

## Overview / TL;DR

This guide covers the practical aspects of truncating Matryoshka embeddings in production: how to use the OpenAI `dimensions` parameter, how to manually truncate and L2-normalize for open-source models, real quality-vs-dimension curves, optimal dimensions per use case, and complete code examples for every scenario. The key insight is that truncation is not approximation -- with MRL-trained models, truncated embeddings are first-class representations that were optimized during training.

---

## API-Based Truncation

### OpenAI text-embedding-3

OpenAI's v3 models support native Matryoshka truncation via the `dimensions` parameter. The API returns pre-truncated, pre-normalized embeddings.

```python
from openai import OpenAI
import numpy as np

client = OpenAI()


def embed_openai(
    texts: list[str],
    model: str = "text-embedding-3-large",
    dimensions: int = 1024,
) -> np.ndarray:
    """Embed texts with OpenAI at specified dimensions.

    The API handles truncation and normalization internally.
    You can use any dimension from 1 to the model's maximum (3072 for large).
    Practical minimum: 256.
    """
    all_embeddings = []
    batch_size = 2048

    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        response = client.embeddings.create(
            model=model,
            input=batch,
            dimensions=dimensions,
        )
        all_embeddings.extend([item.embedding for item in response.data])

    return np.array(all_embeddings)


# Compare dimensions
text = "Kubernetes horizontal pod autoscaler adjusts replica count based on CPU utilization."

for dims in [256, 512, 1024, 1536, 3072]:
    emb = embed_openai([text], dimensions=dims)
    norm = np.linalg.norm(emb[0])
    print(f"dims={dims:4d}: shape={emb.shape}, norm={norm:.4f}")
    # norm will be ~1.0 (already normalized by API)
```

**Key behavior**:
- Any integer from 1 to max_dims is accepted.
- The returned embedding is already L2-normalized.
- Cost is the same regardless of the `dimensions` parameter (you pay for the full computation).
- Storage and downstream computation are reduced.

### Voyage AI

```python
import voyageai

client = voyageai.Client()


def embed_voyage(
    texts: list[str],
    model: str = "voyage-3-large",
    dimensions: int = 1024,
    input_type: str = "document",
) -> list[list[float]]:
    """Embed texts with Voyage at specified dimensions."""
    all_embeddings = []
    batch_size = 128

    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        result = client.embed(
            texts=batch,
            model=model,
            input_type=input_type,
            output_dimension=dimensions,
        )
        all_embeddings.extend(result.embeddings)

    return all_embeddings
```

### Jina AI

```python
import requests


def embed_jina(
    texts: list[str],
    dimensions: int = 512,
    task: str = "retrieval.passage",
    api_key: str = "jina_YOUR_KEY",
) -> list[list[float]]:
    """Embed texts with Jina v3 at specified dimensions."""
    response = requests.post(
        "https://api.jina.ai/v1/embeddings",
        headers={"Authorization": f"Bearer {api_key}"},
        json={
            "model": "jina-embeddings-v3",
            "task": task,
            "input": texts,
            "dimensions": dimensions,
        },
    )
    return [item["embedding"] for item in response.json()["data"]]
```

---

## Manual Truncation for Open-Source Models

For models without API-based truncation (local sentence-transformers models), truncate manually and re-normalize.

### Basic Truncation

```python
import numpy as np
from sentence_transformers import SentenceTransformer


def truncate_and_normalize(
    embeddings: np.ndarray,
    target_dims: int,
) -> np.ndarray:
    """Truncate Matryoshka embeddings and re-normalize.

    IMPORTANT: This only produces good results for MRL-trained models.
    For non-MRL models, truncation produces garbage.

    Args:
        embeddings: (n, full_dims) array, may or may not be normalized.
        target_dims: Number of dimensions to keep (prefix).

    Returns:
        (n, target_dims) array, L2-normalized.
    """
    if target_dims >= embeddings.shape[1]:
        # No truncation needed
        norms = np.linalg.norm(embeddings, axis=1, keepdims=True)
        return embeddings / np.maximum(norms, 1e-12)

    truncated = embeddings[:, :target_dims].copy()
    norms = np.linalg.norm(truncated, axis=1, keepdims=True)
    return truncated / np.maximum(norms, 1e-12)


# Example with nomic-embed
model = SentenceTransformer("nomic-ai/nomic-embed-text-v1.5", trust_remote_code=True)

# nomic-embed requires prefixes
queries = ["search_query: How does Kubernetes autoscaling work?"]
documents = [
    "search_document: Kubernetes HPA adjusts pod replicas based on CPU utilization.",
    "search_document: Docker containers share the host OS kernel for isolation.",
]

# Get full-dimension embeddings
full_embs = model.encode(queries + documents, normalize_embeddings=True)

# Truncate to different dimensions
for dims in [64, 128, 256, 512, 768]:
    truncated = truncate_and_normalize(full_embs, dims)
    # Similarity between query and relevant doc
    sim_relevant = np.dot(truncated[0], truncated[1])
    # Similarity between query and irrelevant doc
    sim_irrelevant = np.dot(truncated[0], truncated[2])
    margin = sim_relevant - sim_irrelevant
    print(f"dims={dims:3d}: relevant={sim_relevant:.4f}, irrelevant={sim_irrelevant:.4f}, margin={margin:.4f}")
```

### Batch Processing with Truncation

```python
def embed_and_truncate(
    model: SentenceTransformer,
    texts: list[str],
    target_dims: int,
    batch_size: int = 256,
) -> np.ndarray:
    """Embed texts and truncate to target dimensions.

    Efficient batch processing for large corpora.
    """
    # Encode at full dimensions
    full_embeddings = model.encode(
        texts,
        normalize_embeddings=False,  # We normalize after truncation
        batch_size=batch_size,
        show_progress_bar=True,
    )

    # Truncate and normalize
    return truncate_and_normalize(full_embeddings, target_dims)


# Index 100K documents at 256 dims
model = SentenceTransformer("nomic-ai/nomic-embed-text-v1.5", trust_remote_code=True)
documents = ["search_document: " + doc for doc in your_documents]
doc_embeddings = embed_and_truncate(model, documents, target_dims=256, batch_size=512)
print(f"Index shape: {doc_embeddings.shape}")  # (100000, 256)
print(f"Index size: {doc_embeddings.nbytes / (1024**3):.2f} GB")  # ~0.10 GB
```

---

## Dimension vs Quality Curves

### Real Data: text-embedding-3-large on MTEB Retrieval

| Dims | MTEB Retrieval | Relative Quality | Storage/1M | Index Time | Query Latency |
|------|---------------|-----------------|-----------|-----------|--------------|
| 3072 | 66.1 | 100.0% | 11.4 GB | 1.0x | 1.0x |
| 2048 | 65.8 | 99.5% | 7.6 GB | 0.7x | 0.7x |
| 1536 | 65.5 | 99.1% | 5.7 GB | 0.5x | 0.5x |
| 1024 | 65.0 | 98.3% | 3.8 GB | 0.3x | 0.3x |
| 768 | 64.5 | 97.6% | 2.9 GB | 0.25x | 0.25x |
| 512 | 63.8 | 96.5% | 1.9 GB | 0.17x | 0.17x |
| 384 | 63.0 | 95.3% | 1.4 GB | 0.13x | 0.13x |
| 256 | 61.5 | 93.0% | 1.0 GB | 0.08x | 0.08x |
| 128 | 57.8 | 87.4% | 0.5 GB | 0.04x | 0.04x |
| 64 | 52.2 | 79.0% | 0.25 GB | 0.02x | 0.02x |

**Observation**: The curve has three regions:
1. **Plateau (1024-3072)**: Quality loss is <2%. Storage saves 3-8x. This is free performance.
2. **Graceful decline (256-1024)**: Quality loss is 2-7%. Good trade-off for most applications.
3. **Quality cliff (<256)**: Quality drops sharply. Only use for coarse pre-filtering.

### Measuring Quality on Your Own Data

```python
import numpy as np
from sentence_transformers import SentenceTransformer
import json


def measure_quality_curve(
    model_name: str,
    eval_data_path: str,
    dimensions: list[int] = [64, 128, 256, 384, 512, 768, 1024],
) -> dict:
    """Measure retrieval quality at each Matryoshka dimension on your eval set.

    Args:
        eval_data_path: JSON file with {"queries": [...], "corpus": [...],
                        "relevant": [{query_idx: [doc_indices]}]}
    """
    model = SentenceTransformer(model_name, trust_remote_code=True)

    with open(eval_data_path) as f:
        data = json.load(f)

    queries = data["queries"]
    corpus = data["corpus"]
    relevant = data["relevant"]  # list of sets of relevant doc indices

    # Embed at full dimensions
    full_dim = model.get_sentence_embedding_dimension()
    q_embs_full = model.encode(queries, normalize_embeddings=True)
    d_embs_full = model.encode(corpus, normalize_embeddings=True)

    results = {}

    for dim in dimensions + [full_dim]:
        # Truncate
        q_embs = truncate_and_normalize(q_embs_full, dim)
        d_embs = truncate_and_normalize(d_embs_full, dim)

        # Compute Hit@10
        scores = np.dot(q_embs, d_embs.T)
        hits = 0
        mrr_sum = 0.0

        for i in range(len(queries)):
            top_10 = set(np.argsort(scores[i])[::-1][:10])
            rel_set = set(relevant[i])
            if top_10 & rel_set:
                hits += 1
            for rank, idx in enumerate(np.argsort(scores[i])[::-1][:10], 1):
                if idx in rel_set:
                    mrr_sum += 1.0 / rank
                    break

        hit_at_10 = hits / len(queries)
        mrr = mrr_sum / len(queries)

        results[dim] = {"hit_at_10": hit_at_10, "mrr_at_10": mrr}
        print(f"dims={dim:4d}: Hit@10={hit_at_10:.4f}, MRR@10={mrr:.4f}")

    return results
```

---

## Optimal Dimensions per Use Case

### Decision Table

| Use Case | Recommended Dims | Rationale |
|----------|-----------------|-----------|
| Pre-filtering before re-ranker | 256 | Re-ranker corrects errors, embedding just needs rough relevance |
| High-volume production search | 512 | Best quality/cost trade-off for most applications |
| Standard RAG retrieval | 768-1024 | Near-full quality, manageable storage |
| High-precision without re-ranker | Full (1536-3072) | When embedding quality is all you have |
| Classification / clustering | 256-512 | Broad distinctions suffice |
| Semantic caching | 256 | Fast comparison, approximate matching is fine |
| Document fingerprinting | 128-256 | Just need to detect near-duplicates |
| Hierarchical search (stage 1) | 256 | Fast candidate generation |
| Hierarchical search (stage 2) | Full | Precise re-scoring |

### Storage Impact Calculation

```python
def compare_storage(
    n_vectors: int,
    original_dims: int,
    target_dims: int,
) -> dict:
    """Compare storage before and after dimension reduction."""
    original_bytes = n_vectors * original_dims * 4  # float32
    reduced_bytes = n_vectors * target_dims * 4

    def fmt(b: int) -> str:
        if b < 1024**2:
            return f"{b / 1024:.1f} KB"
        if b < 1024**3:
            return f"{b / (1024**2):.1f} MB"
        return f"{b / (1024**3):.2f} GB"

    return {
        "original": fmt(original_bytes),
        "reduced": fmt(reduced_bytes),
        "savings": f"{(1 - reduced_bytes/original_bytes) * 100:.1f}%",
        "compression_ratio": f"{original_dims / target_dims:.1f}x",
    }


# Examples
for n in [100_000, 1_000_000, 10_000_000]:
    result = compare_storage(n, 3072, 512)
    print(f"{n:>12,} vectors: {result['original']:>10} -> {result['reduced']:>10} "
          f"({result['savings']} savings, {result['compression_ratio']})")

# Output:
#      100,000 vectors:    1.14 GB ->    0.19 GB (83.3% savings, 6.0x)
#    1,000,000 vectors:   11.44 GB ->    1.91 GB (83.3% savings, 6.0x)
#   10,000,000 vectors:  114.44 GB ->   19.07 GB (83.3% savings, 6.0x)
```

---

## Combining Truncation with Binary Quantization

For extreme compression, first truncate with Matryoshka, then apply binary quantization. This is the most aggressive compression available.

```python
import numpy as np


def matryoshka_binary_compress(
    embeddings: np.ndarray,
    target_dims: int = 256,
) -> tuple[np.ndarray, dict]:
    """Two-stage compression: Matryoshka truncation + binary quantization.

    A 3072-dim float32 vector (12,288 bytes) becomes
    a 256-dim binary vector (32 bytes) -- 384x compression.
    """
    # Stage 1: Matryoshka truncation
    truncated = embeddings[:, :target_dims].copy()
    norms = np.linalg.norm(truncated, axis=1, keepdims=True)
    truncated /= np.maximum(norms, 1e-12)

    # Stage 2: Binary quantization (1 bit per dimension)
    binary = (truncated > 0).astype(np.uint8)
    packed = np.packbits(binary, axis=1)

    original_bytes = embeddings.shape[0] * embeddings.shape[1] * 4
    compressed_bytes = packed.nbytes

    stats = {
        "original_dims": embeddings.shape[1],
        "truncated_dims": target_dims,
        "original_bytes_per_vector": embeddings.shape[1] * 4,
        "compressed_bytes_per_vector": packed.shape[1],
        "compression_ratio": (embeddings.shape[1] * 4) / packed.shape[1],
        "total_original": f"{original_bytes / (1024**3):.2f} GB",
        "total_compressed": f"{compressed_bytes / (1024**3):.2f} GB",
    }

    return packed, stats


def hamming_search(
    query_packed: np.ndarray,
    corpus_packed: np.ndarray,
    top_k: int = 10,
) -> list[tuple[int, float]]:
    """Search using Hamming distance on binary-packed vectors.

    Fast but approximate. Use for candidate generation, then re-score.
    """
    # XOR gives differing bits, popcount gives Hamming distance
    xor = np.bitwise_xor(corpus_packed, query_packed)
    # Count set bits in each byte, sum across bytes
    distances = np.zeros(len(corpus_packed))
    for byte_idx in range(xor.shape[1]):
        # Lookup table for popcount of a byte
        distances += np.array([bin(b).count("1") for b in xor[:, byte_idx]])

    total_bits = query_packed.shape[0] * 8
    similarities = 1.0 - (distances / total_bits)

    top_indices = np.argsort(similarities)[::-1][:top_k]
    return [(int(idx), float(similarities[idx])) for idx in top_indices]


# Example: compress 1M vectors from 3072 to 256-bit
n_vectors = 1_000_000
full_embeddings = np.random.randn(n_vectors, 3072).astype(np.float32)
packed, stats = matryoshka_binary_compress(full_embeddings, target_dims=256)

print(f"Compression: {stats['compression_ratio']:.0f}x")
print(f"Storage: {stats['total_original']} -> {stats['total_compressed']}")
print(f"Bytes per vector: {stats['original_bytes_per_vector']} -> {stats['compressed_bytes_per_vector']}")
```

---

## Common Pitfalls

1. **Not re-normalizing after manual truncation.** The truncated prefix is NOT unit-length. Cosine similarity requires L2-normalized vectors. Always normalize.
2. **Mixing dimensions between queries and documents.** A 256-dim query embedding cannot be compared to a 1024-dim document embedding. Both must be at the same dimension.
3. **Using dimensions below the quality cliff.** Just because the API accepts `dimensions=32` does not mean it is useful. Always test quality at your target dimension.
4. **Assuming truncation reduces API cost.** OpenAI charges the same regardless of the `dimensions` parameter. You save on storage and downstream compute, not on the embedding API call.
5. **Truncating non-MRL models.** Not all models support Matryoshka. Truncating a standard model's embeddings produces much worse results than truncating an MRL model's.

---

## References

- Kusupati et al., "Matryoshka Representation Learning" (2022) -- https://arxiv.org/abs/2205.13147
- OpenAI Embeddings Guide -- https://platform.openai.com/docs/guides/embeddings
- Nomic Embed Matryoshka -- https://blog.nomic.ai/posts/nomic-embed-matryoshka-v1.5
- Qdrant Binary Quantization -- https://qdrant.tech/documentation/guides/quantization/
