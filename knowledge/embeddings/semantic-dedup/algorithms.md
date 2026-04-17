# Semantic Dedup Algorithms -- Detailed Implementation

## Overview / TL;DR

This guide provides complete implementations for every major deduplication algorithm: MinHash + LSH (datasketch library for scalable near-duplicate detection), SimHash (bit-level fingerprinting), embedding cosine + DBSCAN clustering, threshold selection via ROC on labeled pairs, scalable dedup with FAISS brute-force + threshold, and hierarchical approaches for large corpora (100M+ documents). Each algorithm includes runnable code, complexity analysis, and guidance on when to use it.

---

## Algorithm 1: MinHash + LSH

MinHash creates a compact signature of each document's n-gram set. LSH (Locality-Sensitive Hashing) enables approximate nearest neighbor search on these signatures, finding near-duplicates without comparing every pair.

**Best for**: Near-exact duplicates at scale. Fast, memory-efficient, but does not catch paraphrase duplicates.

```python
from datasketch import MinHash, MinHashLSH
import re


def minhash_dedup(
    documents: list[str],
    threshold: float = 0.8,
    num_perm: int = 128,
    ngram_size: int = 5,
) -> dict:
    """Deduplicate using MinHash + LSH.

    Complexity: O(n) for indexing, O(1) per query.
    Memory: ~128 bytes per document (for 128 permutations).

    Args:
        documents: List of document texts.
        threshold: Jaccard similarity threshold (0.0-1.0).
        num_perm: Number of permutations (higher = more accurate, more memory).
        ngram_size: Character n-gram size for shingling.

    Returns:
        Dictionary with duplicate groups and unique document indices.
    """
    def create_minhash(text: str) -> MinHash:
        """Create a MinHash signature from text."""
        m = MinHash(num_perm=num_perm)
        # Create character n-grams (shingles)
        text = re.sub(r'\s+', ' ', text.lower().strip())
        for i in range(len(text) - ngram_size + 1):
            shingle = text[i:i + ngram_size]
            m.update(shingle.encode('utf-8'))
        return m

    # Create LSH index
    lsh = MinHashLSH(threshold=threshold, num_perm=num_perm)
    minhashes = {}

    # Index all documents
    for i, doc in enumerate(documents):
        mh = create_minhash(doc)
        minhashes[i] = mh
        try:
            lsh.insert(f"doc_{i}", mh)
        except ValueError:
            # Duplicate detected during insertion
            pass

    # Find duplicate groups
    seen = set()
    duplicate_groups = []
    unique_indices = set(range(len(documents)))

    for i in range(len(documents)):
        if i in seen:
            continue

        # Query LSH for similar documents
        result = lsh.query(minhashes[i])
        group_indices = [int(r.split("_")[1]) for r in result]

        if len(group_indices) > 1:
            duplicate_groups.append(group_indices)
            # Keep only the first in each group
            for idx in group_indices[1:]:
                unique_indices.discard(idx)
                seen.add(idx)
        seen.add(i)

    n_removed = len(documents) - len(unique_indices)
    print(f"MinHash dedup: {n_removed} duplicates in {len(duplicate_groups)} groups")
    print(f"  Unique: {len(unique_indices)}/{len(documents)} ({len(unique_indices)/len(documents)*100:.1f}%)")

    return {
        "unique_indices": sorted(unique_indices),
        "duplicate_groups": duplicate_groups,
        "n_removed": n_removed,
    }
```

---

## Algorithm 2: SimHash

SimHash creates a single bit-vector fingerprint per document. Similar documents have similar fingerprints (low Hamming distance). Very fast and memory-efficient.

```python
import hashlib
import numpy as np
from collections import Counter


def simhash_dedup(
    documents: list[str],
    hash_bits: int = 64,
    max_hamming_distance: int = 3,
    ngram_size: int = 3,
) -> dict:
    """Deduplicate using SimHash fingerprinting.

    Complexity: O(n) for fingerprinting, O(n * buckets) for finding duplicates.
    Memory: 8 bytes per document (64-bit fingerprint).

    Args:
        documents: List of document texts.
        hash_bits: Number of bits in the fingerprint (64 or 128).
        max_hamming_distance: Maximum Hamming distance for duplicate.
        ngram_size: Word n-gram size.
    """
    def compute_simhash(text: str) -> int:
        """Compute SimHash fingerprint for a text."""
        words = text.lower().split()
        # Create word n-grams
        ngrams = []
        for i in range(len(words) - ngram_size + 1):
            ngrams.append(" ".join(words[i:i + ngram_size]))

        # Count n-gram frequencies
        freq = Counter(ngrams)

        # Compute weighted hash
        v = [0] * hash_bits
        for ngram, weight in freq.items():
            h = int(hashlib.md5(ngram.encode()).hexdigest(), 16)
            for i in range(hash_bits):
                if h & (1 << i):
                    v[i] += weight
                else:
                    v[i] -= weight

        # Convert to binary fingerprint
        fingerprint = 0
        for i in range(hash_bits):
            if v[i] > 0:
                fingerprint |= (1 << i)

        return fingerprint

    def hamming_distance(a: int, b: int) -> int:
        """Count differing bits between two integers."""
        return bin(a ^ b).count('1')

    # Compute fingerprints
    fingerprints = []
    for doc in documents:
        fp = compute_simhash(doc)
        fingerprints.append(fp)

    # Find duplicates by Hamming distance
    unique_indices = set(range(len(documents)))
    duplicate_groups = []

    for i in range(len(documents)):
        if i not in unique_indices:
            continue
        group = [i]
        for j in range(i + 1, len(documents)):
            if j not in unique_indices:
                continue
            dist = hamming_distance(fingerprints[i], fingerprints[j])
            if dist <= max_hamming_distance:
                group.append(j)
                unique_indices.discard(j)

        if len(group) > 1:
            duplicate_groups.append(group)

    n_removed = len(documents) - len(unique_indices)
    print(f"SimHash dedup: {n_removed} duplicates in {len(duplicate_groups)} groups")

    return {
        "unique_indices": sorted(unique_indices),
        "duplicate_groups": duplicate_groups,
        "n_removed": n_removed,
        "fingerprints": fingerprints,
    }
```

---

## Algorithm 3: Embedding Cosine + DBSCAN Clustering

Use embedding similarity and DBSCAN to cluster near-duplicate documents. DBSCAN naturally identifies groups of similar documents without requiring the number of clusters.

```python
import numpy as np
from sentence_transformers import SentenceTransformer
from sklearn.cluster import DBSCAN


def embedding_dbscan_dedup(
    documents: list[str],
    model_name: str = "BAAI/bge-base-en-v1.5",
    similarity_threshold: float = 0.95,
    min_samples: int = 2,
) -> dict:
    """Deduplicate using embedding cosine similarity + DBSCAN.

    DBSCAN groups documents that are mutually within the similarity threshold.
    One representative is kept from each cluster.

    Catches paraphrase duplicates that hash-based methods miss.
    """
    model = SentenceTransformer(model_name)

    # Embed all documents
    print(f"Embedding {len(documents)} documents...")
    embeddings = model.encode(
        documents,
        normalize_embeddings=True,
        batch_size=256,
        show_progress_bar=True,
    )

    # DBSCAN uses distance, not similarity. Convert threshold.
    # cosine_distance = 1 - cosine_similarity
    eps = 1.0 - similarity_threshold

    # Compute cosine distance matrix
    # For large corpora, use FAISS instead (see next algorithm)
    similarity_matrix = np.dot(embeddings, embeddings.T)
    distance_matrix = 1.0 - similarity_matrix

    # Run DBSCAN
    clustering = DBSCAN(
        eps=eps,
        min_samples=min_samples,
        metric="precomputed",
    ).fit(distance_matrix)

    labels = clustering.labels_
    n_clusters = len(set(labels)) - (1 if -1 in labels else 0)

    # Select representatives: keep the longest document in each cluster
    unique_indices = []
    duplicate_groups = []

    # Handle unclustered points (label = -1): these are unique
    for i, label in enumerate(labels):
        if label == -1:
            unique_indices.append(i)

    # Handle clusters: keep one representative per cluster
    for cluster_id in range(n_clusters):
        cluster_indices = [i for i, l in enumerate(labels) if l == cluster_id]
        if len(cluster_indices) > 1:
            duplicate_groups.append(cluster_indices)
        # Keep the longest document as representative
        representative = max(cluster_indices, key=lambda i: len(documents[i]))
        unique_indices.append(representative)

    unique_indices.sort()
    n_removed = len(documents) - len(unique_indices)

    print(f"DBSCAN dedup: {n_clusters} clusters, {n_removed} duplicates removed")
    print(f"  Unique: {len(unique_indices)}/{len(documents)}")

    return {
        "unique_indices": unique_indices,
        "duplicate_groups": duplicate_groups,
        "n_removed": n_removed,
        "n_clusters": n_clusters,
        "labels": labels.tolist(),
    }
```

---

## Algorithm 4: Scalable Dedup with FAISS

For large corpora (1M+ documents), the O(n^2) distance matrix approach is infeasible. Use FAISS for efficient similarity search.

```python
import numpy as np
from sentence_transformers import SentenceTransformer
import faiss


def faiss_dedup(
    documents: list[str],
    model_name: str = "BAAI/bge-base-en-v1.5",
    similarity_threshold: float = 0.95,
    batch_size: int = 1000,
    n_neighbors: int = 50,
) -> dict:
    """Scalable deduplication using FAISS similarity search.

    For each document, finds its nearest neighbors above the threshold.
    Uses union-find to group duplicates transitively.

    Scales to millions of documents.
    """
    model = SentenceTransformer(model_name)

    # Embed
    print(f"Embedding {len(documents)} documents...")
    embeddings = model.encode(
        documents,
        normalize_embeddings=True,
        batch_size=256,
        show_progress_bar=True,
    ).astype(np.float32)

    # Build FAISS index
    dim = embeddings.shape[1]
    index = faiss.IndexFlatIP(dim)
    index.add(embeddings)

    # Union-Find for grouping duplicates
    parent = list(range(len(documents)))

    def find(x):
        while parent[x] != x:
            parent[x] = parent[parent[x]]
            x = parent[x]
        return x

    def union(x, y):
        px, py = find(x), find(y)
        if px != py:
            # Keep the one with the longer document as root
            if len(documents[px]) >= len(documents[py]):
                parent[py] = px
            else:
                parent[px] = py

    # Batch similarity search
    print("Finding near-duplicates...")
    for start in range(0, len(documents), batch_size):
        end = min(start + batch_size, len(documents))
        batch_embs = embeddings[start:end]

        scores, indices = index.search(batch_embs, n_neighbors)

        for i, (score_row, idx_row) in enumerate(zip(scores, indices)):
            doc_idx = start + i
            for score, neighbor_idx in zip(score_row, idx_row):
                if neighbor_idx == doc_idx or neighbor_idx == -1:
                    continue
                if score >= similarity_threshold:
                    union(doc_idx, neighbor_idx)

    # Build groups from union-find
    groups = {}
    for i in range(len(documents)):
        root = find(i)
        if root not in groups:
            groups[root] = []
        groups[root].append(i)

    # Select unique documents (representatives of each group)
    unique_indices = []
    duplicate_groups = []
    for root, members in groups.items():
        unique_indices.append(root)
        if len(members) > 1:
            duplicate_groups.append(members)

    unique_indices.sort()
    n_removed = len(documents) - len(unique_indices)

    print(f"FAISS dedup: {n_removed} duplicates in {len(duplicate_groups)} groups")
    print(f"  Unique: {len(unique_indices)}/{len(documents)}")

    return {
        "unique_indices": unique_indices,
        "duplicate_groups": duplicate_groups,
        "n_removed": n_removed,
    }
```

---

## Algorithm 5: Threshold Selection via ROC

```python
import numpy as np
from sklearn.metrics import roc_curve, auc


def select_threshold_roc(
    labeled_pairs: list[dict],  # [{"doc_a": str, "doc_b": str, "is_duplicate": bool}]
    model_name: str = "BAAI/bge-base-en-v1.5",
) -> dict:
    """Select optimal dedup threshold using ROC analysis on labeled pairs.

    Requires a labeled dataset of document pairs with duplicate/not-duplicate labels.
    """
    model = SentenceTransformer(model_name)

    # Compute similarities for all pairs
    texts_a = [p["doc_a"] for p in labeled_pairs]
    texts_b = [p["doc_b"] for p in labeled_pairs]
    labels = np.array([1 if p["is_duplicate"] else 0 for p in labeled_pairs])

    embs_a = model.encode(texts_a, normalize_embeddings=True, batch_size=256)
    embs_b = model.encode(texts_b, normalize_embeddings=True, batch_size=256)

    similarities = np.sum(embs_a * embs_b, axis=1)

    # ROC curve
    fpr, tpr, thresholds = roc_curve(labels, similarities)
    roc_auc = auc(fpr, tpr)

    # Find optimal threshold (maximize Youden's J statistic)
    j_scores = tpr - fpr
    optimal_idx = np.argmax(j_scores)
    optimal_threshold = thresholds[optimal_idx]

    # Find threshold for specific precision targets
    precision_targets = {}
    for target_fpr in [0.01, 0.05, 0.10]:
        idx = np.argmin(np.abs(fpr - target_fpr))
        precision_targets[f"fpr_{target_fpr}"] = {
            "threshold": float(thresholds[idx]),
            "tpr": float(tpr[idx]),
            "fpr": float(fpr[idx]),
        }

    print(f"ROC AUC: {roc_auc:.4f}")
    print(f"Optimal threshold (Youden's J): {optimal_threshold:.4f}")
    print(f"  TPR: {tpr[optimal_idx]:.4f}, FPR: {fpr[optimal_idx]:.4f}")

    return {
        "roc_auc": float(roc_auc),
        "optimal_threshold": float(optimal_threshold),
        "optimal_tpr": float(tpr[optimal_idx]),
        "optimal_fpr": float(fpr[optimal_idx]),
        "precision_targets": precision_targets,
    }
```

---

## Algorithm 6: Hierarchical Dedup for Large Corpora

For 100M+ documents, even FAISS-based approaches can be slow. Use a hierarchical approach: coarse filtering (hash) then fine filtering (embedding).

```python
import hashlib
import numpy as np
from sentence_transformers import SentenceTransformer
from collections import defaultdict


def hierarchical_dedup(
    documents: list[str],
    model_name: str = "BAAI/bge-base-en-v1.5",
    hash_similarity: float = 0.8,
    embedding_threshold: float = 0.95,
    hash_ngram: int = 3,
    n_hash_bands: int = 20,
) -> dict:
    """Hierarchical dedup: hash-based coarse filter + embedding-based fine filter.

    Stage 1 (fast): MinHash to group candidates.
    Stage 2 (slow): Embedding similarity for candidate pairs only.

    This avoids embedding all N^2 pairs.
    """
    from datasketch import MinHash, MinHashLSH

    # Stage 1: MinHash coarse grouping
    print("Stage 1: MinHash candidate grouping...")
    lsh = MinHashLSH(threshold=hash_similarity, num_perm=128)
    minhashes = {}

    for i, doc in enumerate(documents):
        if i % 100000 == 0:
            print(f"  Hashing {i}/{len(documents)}")
        text = doc.lower().strip()
        m = MinHash(num_perm=128)
        for j in range(len(text) - hash_ngram + 1):
            m.update(text[j:j + hash_ngram].encode('utf-8'))
        minhashes[i] = m
        try:
            lsh.insert(f"d_{i}", m)
        except ValueError:
            pass

    # Find candidate groups
    candidate_pairs = set()
    for i in range(len(documents)):
        result = lsh.query(minhashes[i])
        indices = [int(r.split("_")[1]) for r in result]
        for j in indices:
            if i < j:
                candidate_pairs.add((i, j))

    print(f"  {len(candidate_pairs)} candidate pairs from MinHash")

    if not candidate_pairs:
        return {"unique_indices": list(range(len(documents))), "n_removed": 0}

    # Stage 2: Embedding verification
    print("Stage 2: Embedding verification...")
    # Collect all document indices that need embedding
    docs_to_embed = set()
    for i, j in candidate_pairs:
        docs_to_embed.add(i)
        docs_to_embed.add(j)

    docs_to_embed = sorted(docs_to_embed)
    idx_map = {doc_idx: pos for pos, doc_idx in enumerate(docs_to_embed)}

    model = SentenceTransformer(model_name)
    texts_to_embed = [documents[i] for i in docs_to_embed]
    embeddings = model.encode(
        texts_to_embed,
        normalize_embeddings=True,
        batch_size=256,
        show_progress_bar=True,
    )

    # Verify pairs with embedding similarity
    parent = list(range(len(documents)))

    def find(x):
        while parent[x] != x:
            parent[x] = parent[parent[x]]
            x = parent[x]
        return x

    def union(x, y):
        px, py = find(x), find(y)
        if px != py:
            parent[py] = px

    verified_dupes = 0
    for i, j in candidate_pairs:
        if i in idx_map and j in idx_map:
            sim = np.dot(embeddings[idx_map[i]], embeddings[idx_map[j]])
            if sim >= embedding_threshold:
                union(i, j)
                verified_dupes += 1

    print(f"  {verified_dupes} verified duplicate pairs")

    # Build unique set
    groups = defaultdict(list)
    for i in range(len(documents)):
        groups[find(i)].append(i)

    unique_indices = [min(group, key=lambda i: -len(documents[i])) for group in groups.values()]
    unique_indices.sort()

    n_removed = len(documents) - len(unique_indices)
    print(f"Hierarchical dedup: {n_removed} duplicates removed")
    print(f"  Unique: {len(unique_indices)}/{len(documents)}")

    return {
        "unique_indices": unique_indices,
        "n_removed": n_removed,
        "n_candidate_pairs": len(candidate_pairs),
        "n_verified_dupes": verified_dupes,
    }
```

---

## Algorithm Comparison

| Algorithm | Corpus Size | Speed | Memory | Catches Paraphrases |
|-----------|-------------|-------|--------|-------------------|
| Content hash | Any | O(n) | O(n) | No |
| MinHash + LSH | 10M+ | O(n) | O(n) | No |
| SimHash | 10M+ | O(n) | O(n) | No |
| DBSCAN + embeddings | <100K | O(n^2) | O(n^2) | Yes |
| FAISS + union-find | 1M-10M | O(n log n) | O(n) | Yes |
| Hierarchical | 10M+ | O(n + k^2) | O(n + k) | Yes |

k = number of candidate pairs from coarse stage.

---

## Common Pitfalls

1. **Using O(n^2) algorithms on large corpora.** DBSCAN with a full distance matrix requires O(n^2) memory and time. At 1M documents, that is 1 trillion comparisons. Use FAISS or hierarchical approaches.
2. **Not using a union-find for transitive dedup.** If A ~ B and B ~ C, then A ~ C. Without transitive grouping, you may keep both A and C.
3. **Setting thresholds without data.** Always analyze the similarity distribution of your specific corpus before choosing a threshold. Use ROC analysis if labeled data is available.
4. **Deduplicating across different document types.** A table of contents and the full document may have high similarity but serve different purposes. Consider metadata-aware dedup.
5. **Not preserving the best representative.** When removing duplicates, keep the most complete/recent/authoritative version, not just the first one encountered.

---

## References

- datasketch MinHash -- https://github.com/ekzhu/datasketch
- SimHash Paper (Charikar 2002) -- https://dl.acm.org/doi/10.1145/509907.509965
- FAISS -- https://github.com/facebookresearch/faiss
- DBSCAN Algorithm -- https://scikit-learn.org/stable/modules/generated/sklearn.cluster.DBSCAN.html
- Union-Find Data Structure -- https://en.wikipedia.org/wiki/Disjoint-set_data_structure
