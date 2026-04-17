# Semantic Deduplication -- Comprehensive Guide

## Overview / TL;DR

Semantic deduplication identifies and removes near-duplicate documents or chunks from a RAG corpus. Duplicates waste index space, inflate storage costs, and cause redundant results that degrade user experience (the same information appearing as 3 of the top 5 results). There are three types of duplicates: exact (identical text), near-exact (minor formatting or whitespace differences), and paraphrase (same meaning, different words). This guide covers detection approaches for each type, provides a decision framework for choosing the right algorithm, and explains how to integrate dedup into production ingestion pipelines.

---

## Why Duplicates Hurt RAG

### Problem 1: Wasted Index Space

Duplicates consume storage and memory without adding information. A 10M-vector index where 15% are duplicates wastes 1.5M vectors worth of storage (5.7 GB at 1024 dims).

### Problem 2: Redundant Retrieval Results

When a query matches a concept that appears in 5 duplicate chunks, all 5 may appear in the top-10 results, pushing genuinely different relevant documents out:

```
Query: "How do I configure Kubernetes autoscaling?"

Without dedup (top 5):
  1. [score 0.92] HPA configuration guide (chunk 1)
  2. [score 0.91] HPA configuration guide (chunk 1, from different source)  <-- duplicate
  3. [score 0.90] HPA configuration guide (slightly rephrased)  <-- paraphrase
  4. [score 0.88] VPA configuration guide  <-- UNIQUE, useful
  5. [score 0.87] HPA configuration guide (another source)  <-- duplicate

With dedup (top 5):
  1. [score 0.92] HPA configuration guide
  2. [score 0.88] VPA configuration guide
  3. [score 0.85] Cluster autoscaler guide
  4. [score 0.83] Karpenter node autoscaling
  5. [score 0.80] Custom metrics autoscaling
```

The deduped index provides 5 distinct pieces of information versus 2.

### Problem 3: Biased Embeddings

If duplicates cluster around certain topics, the vector space becomes biased. Similarity search returns over-represented topics disproportionately.

### Problem 4: Increased LLM Cost

Passing duplicate chunks as context to the LLM wastes input tokens. At $3/M tokens for Claude Sonnet, 3 redundant chunks per query at 500 tokens each = $0.0045 wasted per query. At 100K queries/month, that is $450/month in wasted LLM cost.

---

## Types of Duplicates

### Type 1: Exact Duplicates

Identical text content. Caused by ingesting the same document multiple times, or the same content appearing on multiple pages.

**Detection**: Hash comparison (MD5, SHA-256). O(n) time, 100% accurate.

```python
import hashlib


def detect_exact_duplicates(documents: list[str]) -> dict:
    """Find exact duplicate documents using content hashing.

    Returns:
        Dictionary mapping hash -> list of document indices.
        Groups with >1 index contain duplicates.
    """
    hash_groups = {}
    for i, doc in enumerate(documents):
        # Normalize whitespace before hashing
        normalized = " ".join(doc.split())
        doc_hash = hashlib.sha256(normalized.encode()).hexdigest()

        if doc_hash not in hash_groups:
            hash_groups[doc_hash] = []
        hash_groups[doc_hash].append(i)

    duplicates = {h: indices for h, indices in hash_groups.items() if len(indices) > 1}

    n_dupes = sum(len(indices) - 1 for indices in duplicates.values())
    print(f"Found {n_dupes} exact duplicates in {len(documents)} documents")

    return duplicates
```

### Type 2: Near-Exact Duplicates

Almost identical text with minor differences: extra whitespace, different formatting, added/removed headers, trailing punctuation.

**Detection**: Normalized hash comparison, or character-level similarity (edit distance).

```python
import re
import hashlib


def detect_near_exact_duplicates(
    documents: list[str],
    similarity_threshold: float = 0.95,
) -> dict:
    """Find near-exact duplicates using aggressive text normalization.

    Normalizes case, whitespace, punctuation, and common formatting
    before hashing.
    """
    def normalize_text(text: str) -> str:
        text = text.lower()
        text = re.sub(r'\s+', ' ', text)  # Collapse whitespace
        text = re.sub(r'[^\w\s]', '', text)  # Remove punctuation
        text = text.strip()
        return text

    hash_groups = {}
    for i, doc in enumerate(documents):
        normalized = normalize_text(doc)
        doc_hash = hashlib.sha256(normalized.encode()).hexdigest()

        if doc_hash not in hash_groups:
            hash_groups[doc_hash] = []
        hash_groups[doc_hash].append(i)

    duplicates = {h: indices for h, indices in hash_groups.items() if len(indices) > 1}

    n_dupes = sum(len(indices) - 1 for indices in duplicates.values())
    print(f"Found {n_dupes} near-exact duplicates in {len(documents)} documents")

    return duplicates
```

### Type 3: Paraphrase Duplicates

Same meaning, different words. "The cat sat on the mat" and "A feline was sitting upon the rug" are paraphrase duplicates. These require semantic comparison.

**Detection**: Embedding similarity above a threshold (0.90-0.95 cosine). The most expensive but catches the most subtle duplicates.

---

## Detection Approaches Overview

| Approach | Duplicate Types | Speed | Accuracy | Memory |
|----------|----------------|-------|----------|--------|
| Content hash | Exact | Very fast (O(n)) | 100% | Low |
| Normalized hash | Near-exact | Very fast (O(n)) | 95%+ | Low |
| MinHash + LSH | Near-exact | Fast (O(n)) | 90-95% | Medium |
| SimHash | Near-exact | Fast (O(n)) | 85-90% | Low |
| Embedding cosine | Paraphrase | Medium (O(n^2) or O(n log n)) | 95%+ | High |
| Embedding + DBSCAN | Paraphrase | Medium | 90-95% | High |

See `algorithms.md` in this directory for detailed implementation of each approach.

---

## Threshold Selection

The dedup threshold determines the aggressiveness of deduplication:

| Cosine Threshold | What Gets Removed | Risk |
|-----------------|-------------------|------|
| 0.99 | Near-exact copies only | Very conservative, may miss paraphrases |
| 0.95 | Near-exact + close paraphrases | Recommended default |
| 0.90 | Most paraphrases | May remove related but distinct content |
| 0.85 | Broad dedup | Risk of removing genuinely different content |
| 0.80 | Aggressive dedup | High risk of information loss |

### Empirical Threshold Selection

```python
import numpy as np
from sentence_transformers import SentenceTransformer


def analyze_similarity_distribution(
    documents: list[str],
    model_name: str = "BAAI/bge-base-en-v1.5",
    n_samples: int = 50000,
) -> dict:
    """Analyze the pairwise similarity distribution to select a dedup threshold.

    The threshold should be above the "natural" similarity level (same topic,
    different content) and below the "duplicate" similarity level (same content,
    different words).
    """
    model = SentenceTransformer(model_name)
    embeddings = model.encode(documents, normalize_embeddings=True, batch_size=256)

    # Sample random pairs
    n = len(embeddings)
    idx_a = np.random.randint(0, n, size=n_samples)
    idx_b = np.random.randint(0, n, size=n_samples)
    mask = idx_a != idx_b
    idx_a, idx_b = idx_a[mask], idx_b[mask]

    similarities = np.sum(embeddings[idx_a] * embeddings[idx_b], axis=1)

    # Analyze distribution
    percentiles = [90, 95, 99, 99.5, 99.9]
    results = {
        "mean": float(similarities.mean()),
        "std": float(similarities.std()),
        "max": float(similarities.max()),
    }

    print(f"Similarity distribution:")
    print(f"  Mean: {results['mean']:.4f}")
    print(f"  Std:  {results['std']:.4f}")
    print(f"  Max:  {results['max']:.4f}")
    print(f"\nPercentiles:")

    for p in percentiles:
        val = float(np.percentile(similarities, p))
        results[f"p{p}"] = val
        print(f"  P{p}: {val:.4f}")

    print(f"\nRecommended threshold: {results['p99']:.4f} (99th percentile)")
    print(f"  This removes pairs more similar than 99% of random pairs.")

    return results
```

---

## Dedup at Chunk Level vs Document Level

| Level | What It Does | Best For |
|-------|-------------|----------|
| Document-level | Dedup full documents before chunking | Identical documents from different sources |
| Chunk-level | Dedup individual chunks after chunking | Overlapping chunks, repeated sections |
| Both | Document dedup first, then chunk dedup | Most thorough, recommended for production |

### Document-Level Dedup

Fast and should always be done first. Catches full-document duplicates before expensive chunking and embedding.

### Chunk-Level Dedup

More nuanced. Catches:
- Overlapping chunks (from chunk overlap parameter).
- Common boilerplate text (headers, footers, copyright notices).
- Repeated sections across documents (shared templates, copy-pasted content).

```python
def two_level_dedup(
    documents: list[dict],  # [{"id": "...", "text": "...", "chunks": [...]}]
    model_name: str = "BAAI/bge-base-en-v1.5",
    doc_threshold: float = 0.98,
    chunk_threshold: float = 0.95,
) -> dict:
    """Two-level deduplication: document then chunk.

    Returns:
        Dictionary with dedup statistics and the cleaned chunk list.
    """
    # Level 1: Document dedup (hash-based, fast)
    doc_hashes = {}
    unique_docs = []
    doc_dupes = 0

    for doc in documents:
        normalized = " ".join(doc["text"].split()).lower()
        doc_hash = hashlib.sha256(normalized.encode()).hexdigest()

        if doc_hash not in doc_hashes:
            doc_hashes[doc_hash] = doc["id"]
            unique_docs.append(doc)
        else:
            doc_dupes += 1

    print(f"Document dedup: {doc_dupes} duplicates removed, {len(unique_docs)} unique")

    # Level 2: Chunk dedup (embedding-based, slower)
    all_chunks = []
    for doc in unique_docs:
        for chunk in doc["chunks"]:
            all_chunks.append({"text": chunk, "doc_id": doc["id"]})

    model = SentenceTransformer(model_name)
    chunk_texts = [c["text"] for c in all_chunks]
    embeddings = model.encode(chunk_texts, normalize_embeddings=True, batch_size=256)

    # Find and remove duplicate chunks
    # (see algorithms.md for scalable approaches)
    keep_mask = np.ones(len(all_chunks), dtype=bool)
    chunk_dupes = 0

    for i in range(len(all_chunks)):
        if not keep_mask[i]:
            continue
        for j in range(i + 1, len(all_chunks)):
            if not keep_mask[j]:
                continue
            sim = np.dot(embeddings[i], embeddings[j])
            if sim > chunk_threshold:
                keep_mask[j] = False
                chunk_dupes += 1

    unique_chunks = [c for c, keep in zip(all_chunks, keep_mask) if keep]
    print(f"Chunk dedup: {chunk_dupes} duplicates removed, {len(unique_chunks)} unique")

    return {
        "doc_duplicates": doc_dupes,
        "chunk_duplicates": chunk_dupes,
        "unique_chunks": unique_chunks,
        "total_removed": doc_dupes + chunk_dupes,
    }
```

---

## Impact Assessment

### Typical Dedup Rates by Corpus Type

| Corpus Type | Exact Dupes | Near-Exact Dupes | Paraphrase Dupes | Total |
|-------------|------------|------------------|------------------|-------|
| Web scrape | 5-15% | 3-8% | 5-10% | 13-33% |
| Documentation site | 2-5% | 5-10% | 3-5% | 10-20% |
| Email corpus | 10-20% | 5-10% | 2-5% | 17-35% |
| Code repository | 3-8% | 2-5% | 1-3% | 6-16% |
| Academic papers | 1-3% | 1-3% | 2-5% | 4-11% |
| News articles | 5-10% | 3-5% | 5-8% | 13-23% |

---

## Common Pitfalls

1. **Skipping dedup entirely.** Even a "clean" corpus has 5-10% duplicates from overlapping chunks alone. Always dedup.
2. **Setting thresholds too low (too aggressive).** A threshold of 0.85 will remove genuinely different content that happens to discuss similar topics. Start at 0.95 and adjust based on inspection.
3. **Not inspecting removed duplicates.** Always sample and manually review a portion of removed "duplicates" to verify they are actually redundant.
4. **Deduping after embedding instead of before.** Document-level dedup should happen before embedding to save compute. Chunk-level dedup requires embeddings.
5. **Treating boilerplate as duplicates.** Common headers/footers/disclaimers appear in many documents. These should be stripped during preprocessing, not caught by dedup.

---

## References

- datasketch Library (MinHash, LSH) -- https://github.com/ekzhu/datasketch
- FAISS for Similarity Search -- https://github.com/facebookresearch/faiss
- SimHash Paper -- https://dl.acm.org/doi/10.1145/509907.509965
- NearDuplicates Survey -- https://arxiv.org/abs/2303.04760
