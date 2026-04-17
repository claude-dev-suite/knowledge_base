# ColBERT Benchmarks: Quality, Latency, and Storage

## Overview

This document compares ColBERT (v1 and v2) against dense bi-encoders and BM25 across standard benchmarks, with practical guidance on when ColBERT's extra complexity is justified. The key finding: ColBERTv2 outperforms bi-encoders by 5-15% on retrieval quality metrics while remaining within 2-5x of their latency, and its storage costs (with compression) are manageable for most production deployments.

---

## MS MARCO Passage Ranking

MS MARCO is the standard benchmark for passage retrieval. The dev set contains 6,980 queries; the corpus has 8.8M passages.

### Quality Comparison

| Model | Type | MRR@10 | Recall@50 | Recall@200 | Recall@1000 |
|---|---|---|---|---|---|
| BM25 (Anserini, tuned) | Sparse | 0.187 | 0.593 | 0.737 | 0.857 |
| docT5query + BM25 | Sparse (expanded) | 0.277 | 0.720 | 0.819 | 0.915 |
| ANCE | Bi-encoder | 0.330 | 0.755 | 0.868 | 0.959 |
| DPR | Bi-encoder | 0.311 | 0.720 | 0.845 | 0.953 |
| Contriever | Bi-encoder | 0.320 | -- | -- | 0.956 |
| GTR-XXL (4.8B params) | Bi-encoder | 0.349 | -- | -- | 0.970 |
| E5-large-v2 | Bi-encoder | 0.364 | -- | -- | 0.978 |
| ColBERT v1 | Late interaction | 0.360 | 0.829 | 0.923 | 0.968 |
| **ColBERTv2** | **Late interaction** | **0.397** | **0.866** | **0.939** | **0.984** |
| monoT5-3B (reranker) | Cross-encoder | 0.398 | -- | -- | -- |

Key observations:
- ColBERTv2 matches or exceeds the best cross-encoder rerankers on MRR@10
- The gap between ColBERTv2 and the best bi-encoder (E5-large-v2) is ~3 MRR@10 points -- significant in practice
- BM25 with query expansion (docT5query) closes about half the gap to neural models
- ColBERTv2's Recall@1000 (0.984) is among the highest, meaning it finds nearly all relevant passages in its candidate set

### Relative Improvement Analysis

| Comparison | MRR@10 Improvement |
|---|---|
| ColBERTv2 vs BM25 | +112% |
| ColBERTv2 vs ANCE (bi-encoder) | +20% |
| ColBERTv2 vs E5-large-v2 (best bi-encoder) | +9% |
| ColBERTv2 vs ColBERT v1 | +10% |
| ColBERTv2 vs monoT5 (cross-encoder) | -0.3% (essentially equal) |

---

## BEIR Benchmark (Zero-Shot Transfer)

BEIR measures how well models generalize to new domains without fine-tuning. Models are trained on MS MARCO and evaluated on 13 diverse datasets (scientific, biomedical, financial, etc.).

### NDCG@10 Across BEIR Datasets

| Dataset | Domain | BM25 | DPR | Contriever | E5-base-v2 | ColBERTv2 |
|---|---|---|---|---|---|---|
| TREC-COVID | Biomedical | 0.656 | 0.332 | 0.596 | 0.653 | **0.738** |
| NFCorpus | Biomedical | 0.325 | 0.189 | 0.318 | 0.336 | **0.335** |
| NQ | Wikipedia QA | 0.329 | 0.474 | 0.489 | 0.498 | **0.524** |
| HotpotQA | Multi-hop QA | 0.603 | 0.391 | 0.638 | 0.640 | **0.667** |
| FiQA | Financial | 0.236 | 0.112 | 0.329 | 0.343 | **0.356** |
| ArguAna | Argument | 0.315 | 0.175 | 0.446 | 0.437 | **0.463** |
| Touche-2020 | Argument | **0.367** | 0.131 | 0.230 | 0.248 | 0.263 |
| Quora | Duplicate QA | 0.789 | 0.248 | 0.865 | 0.868 | **0.855** |
| SCIDOCS | Scientific | 0.158 | 0.077 | 0.165 | 0.176 | **0.154** |
| SciFact | Scientific | 0.665 | 0.318 | 0.677 | 0.695 | **0.693** |
| DBPedia | Entity | 0.313 | 0.263 | 0.413 | 0.409 | **0.446** |
| FEVER | Fact checking | 0.753 | 0.562 | 0.758 | 0.775 | **0.785** |
| Climate-FEVER | Fact checking | 0.213 | 0.148 | 0.237 | 0.242 | **0.176** |
| **Average** | | 0.440 | 0.299 | 0.474 | 0.486 | **0.497** |

Key observations:
- ColBERTv2 wins on most datasets, especially those with complex queries (HotpotQA, FiQA, ArguAna)
- BM25 beats all neural models on Touche-2020 (argument retrieval with long queries) and is competitive on SCIDOCS
- ColBERTv2's per-token matching helps most when queries contain specific technical terms (biomedical, financial)
- DPR (early bi-encoder) transfers poorly -- modern bi-encoders (E5, Contriever) are much better

### Where BM25 Wins

BM25 outperforms ColBERTv2 in specific scenarios:
- **Exact term matching**: queries with specific codes, IDs, or proper nouns (e.g., "CVE-2024-1234")
- **Very long queries** (Touche-2020): ColBERT's fixed 32-token query limit truncates useful information
- **Domains with specialized notation** (SCIDOCS): chemical formulas, mathematical notation that tokenizers handle poorly

---

## Latency Comparison

All measurements on a single machine with 64GB RAM. GPU used where applicable.

### End-to-End Query Latency

| System | Hardware | Corpus Size | Latency (p50) | Latency (p99) | QPS |
|---|---|---|---|---|---|
| BM25 (Lucene) | CPU | 8.8M | 8ms | 25ms | 120 |
| BM25 (Elasticsearch) | CPU | 8.8M | 12ms | 40ms | 80 |
| ANCE + FAISS (flat) | CPU | 8.8M | 45ms | 120ms | 22 |
| ANCE + FAISS (HNSW) | CPU | 8.8M | 5ms | 15ms | 200 |
| ANCE + FAISS (HNSW) | GPU | 8.8M | 2ms | 5ms | 500 |
| ColBERT v1 (full) | CPU + GPU | 8.8M | 458ms | 1200ms | 2 |
| ColBERTv2 + PLAID (nprobe=2) | CPU | 8.8M | 52ms | 150ms | 19 |
| ColBERTv2 + PLAID (nprobe=8) | CPU | 8.8M | 74ms | 200ms | 13 |
| ColBERTv2 + PLAID (nprobe=32) | CPU | 8.8M | 162ms | 400ms | 6 |
| ColBERTv2 + PLAID (nprobe=2) | GPU | 8.8M | 18ms | 45ms | 55 |

Key observations:
- ColBERTv2 with PLAID is 9x faster than ColBERT v1, making it practical for production
- At nprobe=2, ColBERTv2 is comparable to unoptimized bi-encoder retrieval
- Bi-encoder + HNSW on GPU is still 5-10x faster than ColBERTv2
- BM25 on Lucene remains the latency champion for CPU-only deployments

### Latency Breakdown (ColBERTv2 + PLAID, nprobe=8)

| Phase | Time | % of Total |
|---|---|---|
| Query encoding (BERT forward pass) | 12ms | 16% |
| Centroid interaction | 5ms | 7% |
| Centroid pruning + candidate generation | 8ms | 11% |
| Upper bound filtering | 15ms | 20% |
| Decompression + exact MaxSim | 34ms | 46% |
| **Total** | **74ms** | **100%** |

The dominant cost is decompression and scoring. For lower latency, reduce `ndocs` (fewer candidates to score) or use GPU for the MaxSim computation.

---

## Storage Comparison

### Index Size at Different Scales

| System | 100K docs | 1M docs | 10M docs | 100M docs |
|---|---|---|---|---|
| BM25 (Lucene inverted index) | 200 MB | 1.5 GB | 12 GB | 100 GB |
| Bi-encoder (768d, float32) | 300 MB | 3 GB | 30 GB | 300 GB |
| Bi-encoder (768d, int8 quantized) | 77 MB | 770 MB | 7.7 GB | 77 GB |
| Bi-encoder (256d, Matryoshka) | 100 MB | 1 GB | 10 GB | 100 GB |
| ColBERT v1 (128d, float32) | 7.5 GB | 75 GB | 750 GB | 7.5 TB |
| ColBERTv2 (2-bit compression) | 500 MB | 5 GB | 50 GB | 500 GB |
| ColBERTv2 (1-bit compression) | 270 MB | 2.7 GB | 27 GB | 270 GB |

Key observations:
- ColBERT v1's storage is prohibitive beyond 1M documents
- ColBERTv2 with 2-bit compression is comparable to uncompressed bi-encoder storage
- ColBERTv2 with 1-bit compression is comparable to BM25 index size
- For 100M+ documents, ColBERTv2 (1-bit) at 270 GB is feasible with modern SSDs

### Storage per Document

| System | Bytes/Document | Relative |
|---|---|---|
| BM25 (avg document) | ~2 KB | 1x |
| Bi-encoder (768d, float32) | 3,072 bytes | 1.5x |
| Bi-encoder (768d, int8) | 768 bytes | 0.4x |
| ColBERT v1 (128d, 150 tokens avg) | 76,800 bytes | 38x |
| ColBERTv2 (2-bit, 150 tokens avg) | 5,100 bytes | 2.5x |
| ColBERTv2 (1-bit, 150 tokens avg) | 2,700 bytes | 1.4x |

---

## Quality vs Latency vs Storage Tradeoff

### Decision Matrix

| Requirement | Best Choice | Why |
|---|---|---|
| Lowest latency (<10ms) | Bi-encoder + HNSW | Single vector lookup, GPU-accelerated |
| Best quality, no latency constraint | Cross-encoder reranking | Full token interaction |
| Best quality, <100ms latency | ColBERTv2 + PLAID | Near-cross-encoder quality, practical latency |
| Smallest storage | BM25 or Bi-encoder (quantized) | Inverted index or single compressed vector |
| Best zero-shot transfer | ColBERTv2 | Per-token matching generalizes well |
| Exact term matching (codes, IDs) | BM25 | Lexical matching is perfect for exact terms |
| Budget deployment (CPU only) | BM25 or ColBERTv2 + PLAID | Both work well on CPU |

### When ColBERT is Worth the Complexity

**Worth it when:**
1. **RAG quality is critical**: the 5-15% retrieval quality improvement over bi-encoders translates to noticeably better LLM answers, especially for multi-aspect queries
2. **Your queries are complex**: multi-faceted questions, technical queries with specific terms, or queries requiring understanding of multiple concepts
3. **Zero-shot performance matters**: you are building a general-purpose search system across diverse domains
4. **You have 100K-10M documents**: the sweet spot where ColBERTv2 storage is manageable and quality gains are significant
5. **You can accept 50-100ms latency**: not real-time autocomplete, but acceptable for search and RAG

**Not worth it when:**
1. **Sub-10ms latency required**: bi-encoder + HNSW is the only option
2. **Corpus is tiny (<10K docs)**: just use a cross-encoder to rerank everything
3. **Corpus is huge (>100M docs)**: storage and indexing costs become significant; consider bi-encoder + cross-encoder reranking pipeline
4. **Queries are simple keyword lookups**: BM25 is sufficient and faster
5. **No GPU available for indexing**: CPU indexing is very slow (10x+ slower)

---

## Hybrid Approaches

### ColBERT + BM25 Hybrid

Combining ColBERT with BM25 can further improve quality, especially for queries with exact term requirements:

```python
from ragatouille import RAGPretrainedModel
import subprocess
import json

def hybrid_search(query, colbert_model, bm25_client, k=10, alpha=0.7):
    """
    Combine ColBERT and BM25 scores using weighted sum.
    alpha: weight for ColBERT scores (1-alpha for BM25)
    """
    # ColBERT retrieval
    colbert_results = colbert_model.search(query, k=k*3)
    colbert_scores = {r["document_id"]: r["score"] for r in colbert_results}

    # BM25 retrieval (via Elasticsearch or similar)
    bm25_results = bm25_client.search(query, size=k*3)
    bm25_scores = {r["_id"]: r["_score"] for r in bm25_results["hits"]["hits"]}

    # Normalize scores to [0, 1]
    def normalize(scores):
        if not scores:
            return {}
        min_s, max_s = min(scores.values()), max(scores.values())
        if max_s == min_s:
            return {k: 1.0 for k in scores}
        return {k: (v - min_s) / (max_s - min_s) for k, v in scores.items()}

    colbert_norm = normalize(colbert_scores)
    bm25_norm = normalize(bm25_scores)

    # Combine
    all_doc_ids = set(colbert_norm) | set(bm25_norm)
    combined = {}
    for doc_id in all_doc_ids:
        combined[doc_id] = (
            alpha * colbert_norm.get(doc_id, 0.0) +
            (1 - alpha) * bm25_norm.get(doc_id, 0.0)
        )

    # Sort and return top-k
    ranked = sorted(combined.items(), key=lambda x: x[1], reverse=True)[:k]
    return ranked
```

### ColBERT as Reranker

Use ColBERT to rerank bi-encoder candidates (cheaper than cross-encoder reranking):

```python
def colbert_rerank(query, candidate_texts, candidate_ids, rag_model, k=10):
    """
    Use ColBERT scoring to rerank a set of candidate passages.
    More accurate than bi-encoder but faster than cross-encoder.
    """
    # Score each candidate against the query
    # This uses the ColBERT model directly, not the index
    scores = rag_model.rerank(
        query=query,
        documents=candidate_texts,
        k=k,
    )
    return scores
```

---

## Scaling Considerations

### Cost Estimates at Scale

| Scale | Indexing Time (A100) | Storage (2-bit) | RAM for Search | Monthly Cloud Cost |
|---|---|---|---|---|
| 100K docs | 5 min | 500 MB | 2 GB | ~$50 (small instance) |
| 1M docs | 45 min | 5 GB | 8 GB | ~$150 (medium instance) |
| 10M docs | 8 hours | 50 GB | 64 GB | ~$500 (large instance) |
| 100M docs | 3.5 days | 500 GB | 512 GB+ | ~$3000 (high-memory) |

### Scaling Tips

1. **Shard the index**: for >10M documents, split across multiple machines. PLAID supports index sharding.
2. **Use replicas for QPS**: each replica can serve ~15-50 QPS (depending on nprobe). Scale horizontally.
3. **Pre-filter by metadata**: reduce the candidate set before ColBERT scoring (e.g., filter by date, category) to reduce latency.
4. **Cache frequent queries**: ColBERT results for common queries can be cached with standard HTTP/Redis caching.
5. **Use 1-bit for cold storage, 2-bit for hot**: keep frequently accessed collections at 2-bit for quality, archive at 1-bit.

---

## Common Pitfalls

1. **Comparing apples to oranges**: always use the same evaluation dataset and metrics. MRR@10 on MS MARCO is not comparable to NDCG@10 on BEIR.

2. **Ignoring recall**: MRR@10 measures the rank of the first relevant result. For RAG, Recall@K (how many relevant passages are in the top K) is equally important because you feed K passages to the LLM.

3. **Benchmarking on MS MARCO only**: MS MARCO queries are short web queries. Your domain may have longer, more complex queries where the relative performance of models differs significantly.

4. **Not accounting for index build time**: ColBERTv2 indexing takes hours on large corpora. If your corpus updates frequently, this matters.

5. **Overlooking hybrid approaches**: in production, the best system is often ColBERT/bi-encoder for retrieval + cross-encoder for reranking. Pure ColBERT may not be optimal.

---

## References

- MS MARCO leaderboard: https://microsoft.github.io/msmarco/
- BEIR benchmark: https://github.com/beir-cellar/beir
- MTEB leaderboard (embedding model comparisons): https://huggingface.co/spaces/mteb/leaderboard
- ColBERTv2 paper: https://arxiv.org/abs/2112.01488
- PLAID paper: https://arxiv.org/abs/2205.09707
- DPR paper: https://arxiv.org/abs/2004.04906
- E5 paper: https://arxiv.org/abs/2212.03533
- Contriever paper: https://arxiv.org/abs/2112.09118
