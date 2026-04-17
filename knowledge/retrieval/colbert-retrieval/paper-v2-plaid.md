# ColBERTv2 and PLAID Engine

## Overview

ColBERTv2 (NAACL 2022) and PLAID (2022) solve ColBERT v1's biggest problem: storage. The original ColBERT stores one 128-dim float32 vector per document token, requiring ~75 GB for 1M documents. ColBERTv2 introduces residual compression that reduces this by 6-32x while retaining 95-99% of retrieval quality. PLAID (Performance-optimized Late Interaction using Deferred compression) further optimizes the retrieval engine with centroid-based pruning and deferred decompression, achieving sub-100ms latency.

---

## ColBERTv2: Residual Compression

### The Storage Problem

ColBERT v1 stores per-token embeddings as raw float32 vectors:

```
1M documents * ~150 tokens/doc * 128 dims * 4 bytes = ~75 GB
```

This is impractical for large-scale deployment. ColBERTv2 compresses these embeddings using a residual quantization scheme.

### Centroid-Based Encoding

The key insight: token embeddings from a trained ColBERT model cluster into groups of semantically similar tokens. ColBERTv2 exploits this structure:

**Step 1: Learn centroids.** Run k-means clustering on a sample of all token embeddings. The paper uses K = 2^16 = 65,536 centroids.

**Step 2: For each token embedding, store:**
- The centroid ID (2 bytes -- index into 65,536 centroids)
- A quantized residual vector (the difference between the true embedding and the centroid)

```
Original:    e = [0.123, -0.456, 0.789, ...]   (128 * 4 bytes = 512 bytes)

Compressed:  centroid_id = 4217               (2 bytes)
             residual = quantize(e - centroid[4217])  (128 bits = 16 bytes)

Total: 18 bytes per token (vs 512 bytes in v1) -> 28x compression
```

### Residual Quantization

The residual (e - centroid) is quantized to 1 bit per dimension:

```python
# Simplified residual quantization
def compress_token(embedding, centroids):
    # Find nearest centroid
    centroid_id = nearest_centroid(embedding, centroids)
    centroid = centroids[centroid_id]

    # Compute residual
    residual = embedding - centroid

    # Quantize each dimension to 1 bit (sign bit)
    # 0 = negative residual, 1 = positive residual
    quantized_residual = (residual > 0).astype(np.uint8)

    # Pack 128 bits into 16 bytes
    packed = np.packbits(quantized_residual)

    return centroid_id, packed

def decompress_token(centroid_id, packed_residual, centroids, bucket_weights):
    centroid = centroids[centroid_id]
    residual_signs = np.unpackbits(packed_residual).astype(np.float32)

    # Reconstruct: centroid + sign * magnitude_estimate
    residual_signs = residual_signs * 2 - 1  # Convert 0/1 to -1/+1
    reconstructed = centroid + residual_signs * bucket_weights[centroid_id]

    return reconstructed
```

### Compression Ratios

| Method | Bytes per token | Compression vs v1 | MRR@10 (MS MARCO) |
|---|---|---|---|
| ColBERT v1 (float32) | 512 | 1x | 0.360 |
| ColBERTv2 (2-bit residual) | 34 | 15x | 0.356 |
| ColBERTv2 (1-bit residual) | 18 | 28x | 0.353 |

**At 1M documents:**

| Method | Index Size | Quality |
|---|---|---|
| ColBERT v1 | ~75 GB | baseline |
| ColBERTv2 (2-bit) | ~5 GB | -1.1% MRR@10 |
| ColBERTv2 (1-bit) | ~2.7 GB | -1.9% MRR@10 |

---

## Improved Training in ColBERTv2

### Cross-Encoder Distillation

ColBERTv2 improves training by distilling from a cross-encoder teacher:

```
1. Train a cross-encoder on MS MARCO (monoT5 or similar)
2. For each query, score BM25 top-1000 with the cross-encoder
3. Use cross-encoder scores as soft labels for ColBERTv2 training
```

The loss becomes KL divergence between ColBERTv2 scores and cross-encoder scores:

```python
# Simplified distillation training
def distillation_loss(colbert_scores, teacher_scores, temperature=1.0):
    student_probs = F.softmax(colbert_scores / temperature, dim=-1)
    teacher_probs = F.softmax(teacher_scores / temperature, dim=-1)
    return F.kl_div(student_probs.log(), teacher_probs, reduction='batchmean')
```

### Denoised Hard Negatives

ColBERT v1 used BM25 negatives. ColBERTv2 uses a bootstrapped approach:

1. Train an initial ColBERT model
2. Use it to retrieve hard negatives (passages the model ranks high but are not relevant)
3. Filter out false negatives using the cross-encoder teacher
4. Retrain ColBERT with these cleaned hard negatives

This "denoising" step is important because BM25 negatives contain false negatives (passages that are actually relevant but not labeled). Training on these hurts the model.

---

## PLAID: The Retrieval Engine

PLAID (Performance-optimized Late Interaction using Deferred compression) is the retrieval engine designed to efficiently search ColBERTv2 indexes.

### The Retrieval Challenge

Even with compression, searching requires:
1. Finding candidate documents (cannot score all M documents)
2. Decompressing token embeddings for scoring
3. Computing MaxSim for each candidate

PLAID optimizes each step.

### Step 1: Centroid Interaction

Instead of decompressing all embeddings, PLAID first computes interactions between query tokens and centroids:

```python
def centroid_scoring(query_embeddings, centroids):
    """
    For each query token, score against all centroids.
    Returns: (Nq, K) matrix of query-centroid similarities
    """
    # query_embeddings: (Nq, 128)
    # centroids: (K, 128)  where K = 65536
    scores = query_embeddings @ centroids.T  # (Nq, K)
    return scores
```

This produces a (32, 65536) matrix -- the similarity of each query token to each centroid. This is cheap: 32 * 65536 dot products of 128-dim vectors.

### Step 2: Candidate Generation via Centroid Pruning

For each query token, keep only the top-nprobe centroids. Documents that contain tokens assigned to these centroids are candidates:

```python
def generate_candidates(centroid_scores, token_to_centroid, nprobe=32):
    """
    For each query token, find the top centroids.
    Candidate documents are those with tokens in these centroids.
    """
    candidate_doc_ids = set()

    for q_idx in range(centroid_scores.shape[0]):  # For each query token
        # Top centroids for this query token
        top_centroids = torch.topk(centroid_scores[q_idx], k=nprobe).indices

        # All document tokens assigned to these centroids
        for c_id in top_centroids:
            candidate_doc_ids.update(token_to_centroid[c_id])

    return candidate_doc_ids
```

### Step 3: Centroid-Based Upper Bound Filtering

Before decompressing any embeddings, PLAID estimates an upper bound on each candidate's MaxSim score using only centroid information:

```
For document D with tokens assigned to centroids c1, c2, ..., cN:
  Upper_bound(Q, D) = SUM_i MAX_j centroid_score(qi, cj)
```

Documents whose upper bound is below a threshold are pruned without decompression.

### Step 4: Deferred Decompression

Only for the surviving candidates (typically a few hundred out of millions), PLAID decompresses the actual token embeddings and computes exact MaxSim:

```python
def score_candidates(query_embeddings, candidate_doc_ids, compressed_index):
    scores = {}
    for doc_id in candidate_doc_ids:
        # Decompress only this document's embeddings
        doc_embeddings = decompress_document(doc_id, compressed_index)

        # Exact MaxSim
        sim_matrix = query_embeddings @ doc_embeddings.T  # (Nq, Nd)
        max_sims = sim_matrix.max(dim=1).values           # (Nq,)
        scores[doc_id] = max_sims.sum().item()

    return scores
```

### PLAID Performance Pipeline Summary

```
All documents (e.g., 8.8M)
    |
    v  [Centroid interaction: ~5ms]
Centroid scores computed
    |
    v  [Centroid pruning: ~2ms]
~100K candidate documents
    |
    v  [Upper bound filtering: ~10ms]
~500 candidate documents
    |
    v  [Decompress + exact MaxSim: ~30ms]
Final ranked results
    |
    v
Top-K results returned

Total: ~50ms per query
```

---

## Benchmarks: ColBERTv2 vs v1 vs Others

### MS MARCO Passage Ranking (dev set)

| Model | MRR@10 | Recall@50 | Recall@1000 | Index Size (8.8M passages) |
|---|---|---|---|---|
| BM25 | 0.187 | 0.593 | 0.857 | ~2 GB |
| ANCE (bi-encoder) | 0.330 | 0.755 | 0.959 | ~26 GB |
| ColBERT v1 | 0.360 | 0.829 | 0.968 | ~154 GB |
| ColBERTv2 (1-bit) | 0.397 | 0.866 | 0.984 | ~5.4 GB |
| ColBERTv2 (2-bit) | 0.398 | 0.868 | 0.984 | ~9.4 GB |

ColBERTv2 is better quality than v1 (thanks to distillation training) AND smaller (thanks to compression). The quality improvement comes entirely from better training, not the compression.

### BEIR Benchmark (Zero-Shot Transfer)

ColBERTv2 shows strong zero-shot transfer, outperforming BM25 on most BEIR datasets despite being trained only on MS MARCO:

| Dataset | BM25 | DPR | ColBERTv2 |
|---|---|---|---|
| TREC-COVID | 0.656 | 0.332 | 0.738 |
| NFCorpus | 0.325 | 0.189 | 0.335 |
| NQ | 0.329 | 0.474 | 0.524 |
| HotpotQA | 0.603 | 0.391 | 0.667 |
| FiQA | 0.236 | 0.112 | 0.356 |
| SciFact | 0.665 | 0.318 | 0.693 |
| Average (13 datasets) | 0.440 | 0.299 | 0.497 |

### Latency with PLAID

| System | Latency (ms) | MRR@10 |
|---|---|---|
| BM25 (Anserini) | 55 | 0.187 |
| ANCE + FAISS | 16 | 0.330 |
| ColBERT v1 | 458 | 0.360 |
| ColBERTv2 + PLAID (nprobe=2) | 52 | 0.391 |
| ColBERTv2 + PLAID (nprobe=8) | 74 | 0.397 |
| ColBERTv2 + PLAID (nprobe=32) | 162 | 0.398 |

PLAID with nprobe=2 achieves BM25-level latency at much higher quality.

---

## Implementation Details

### Building a ColBERTv2 Index

```python
from colbert import Indexer, Searcher
from colbert.infra import Run, RunConfig, ColBERTConfig

config = ColBERTConfig(
    nbits=2,                    # 1 or 2 bit residual compression
    kmeans_niters=4,            # k-means iterations for centroid learning
    doc_maxlen=180,             # max document tokens
    query_maxlen=32,            # max query tokens (with MASK padding)
    ncells=1,                   # PLAID cells to search (low = fast, high = accurate)
    centroid_score_threshold=0.5,  # prune candidates below this
    ndocs=8192,                 # candidates to fully score
)

with Run().context(RunConfig(nranks=1, experiment="msmarco")):
    indexer = Indexer(checkpoint="colbert-ir/colbertv2.0", config=config)
    indexer.index(
        name="msmarco.nbits=2",
        collection="/path/to/collection.tsv",
        overwrite=True,
    )
```

### Searching

```python
with Run().context(RunConfig(experiment="msmarco")):
    searcher = Searcher(index="msmarco.nbits=2", collection="/path/to/collection.tsv")

    results = searcher.search("what is information retrieval?", k=10)
    for passage_id, rank, score in zip(*results):
        print(f"Rank {rank} | Score {score:.4f} | PID {passage_id}")
```

### Configuration Knobs

| Parameter | Effect | Default | Range |
|---|---|---|---|
| `nbits` | Residual quantization bits | 2 | 1, 2, 4 |
| `ncells` | PLAID cells to search (nprobe) | 1 | 1-32 |
| `centroid_score_threshold` | Prune low-scoring centroids | 0.5 | 0.3-0.7 |
| `ndocs` | Max candidates for exact scoring | 8192 | 256-65536 |
| `doc_maxlen` | Max document token length | 180 | 64-512 |
| `query_maxlen` | Max query token length | 32 | 16-64 |

---

## Practical Deployment Considerations

### Memory vs Disk

The compressed index can be loaded from disk on demand or memory-mapped. For production:
- **Memory-mapped**: good for moderate QPS, no need to load entire index into RAM
- **Fully in RAM**: required for high QPS (>100 queries/sec)
- **GPU**: the centroid interaction and MaxSim scoring benefit from GPU acceleration

### Index Build Time

Building a ColBERTv2 index involves:
1. Encoding all documents through the model (~1000 docs/sec on a single GPU)
2. Running k-means clustering for centroids (~10 minutes for 8.8M passages)
3. Compressing all token embeddings (~30 minutes for 8.8M passages)

Total: ~3-4 hours for MS MARCO (8.8M passages) on a single GPU.

### Updating the Index

ColBERTv2 indexes are not trivially updatable:
- Adding documents requires encoding + compression + reinsertion into the PLAID index
- Batch updates (e.g., nightly) are much more efficient than real-time inserts
- For real-time updates, consider a hybrid approach: keep recent documents in a small bi-encoder index, periodically merge into the ColBERTv2 index

---

## Common Pitfalls

1. **Setting nprobe=1 and expecting top quality**: nprobe controls accuracy-speed tradeoff. nprobe=1 is fastest but may miss relevant candidates. Start with nprobe=8 and tune down only if latency is unacceptable.

2. **Using 1-bit compression for domain-specific data**: 1-bit residuals work well for general text but may lose too much precision for specialized domains (medical, legal). Test with 2-bit first.

3. **Not retraining centroids on your data**: the default centroids from the MS MARCO checkpoint may not be optimal for your domain. If you fine-tune the ColBERT model, re-index (which re-clusters) to get domain-appropriate centroids.

4. **Ignoring the centroid_score_threshold**: too high prunes good candidates, too low keeps too many (slow). The default (0.5) works for MS MARCO but may need tuning for other domains.

5. **Expecting exact reproduction of full-precision results**: compression is lossy. If you need exact ColBERT scores (e.g., for evaluation), use the uncompressed index.

---

## References

- Santhanam, K. et al. "ColBERTv2: Effective and Efficient Retrieval via Lightweight Late Interaction." NAACL 2022. https://arxiv.org/abs/2112.01488
- Santhanam, K. et al. "PLAID: An Efficient Engine for Late Interaction Retrieval." CIKM 2022. https://arxiv.org/abs/2205.09707
- ColBERT repository: https://github.com/stanford-futuredata/ColBERT
- ColBERTv2 model: https://huggingface.co/colbert-ir/colbertv2.0
