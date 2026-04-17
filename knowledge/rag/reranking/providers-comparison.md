# Reranker Providers Comparison

## TL;DR

This article provides a head-to-head comparison of the major reranking models and APIs available as of 2025: Cohere Rerank v3.5, Voyage rerank-2, Jina Reranker v2, BAAI bge-reranker-v2-m3, and the ms-marco-MiniLM family. We compare cost, latency, quality (BEIR benchmarks), multilingual support, max input length, and provide runnable API code for each. The recommendation matrix at the end maps common scenarios to the best-fit reranker.

---

## Provider Overview

| Provider | Model | Type | Params | Max Tokens | Multilingual | Pricing Model |
|----------|-------|------|--------|-----------|-------------|---------------|
| Cohere | rerank-v3.5 | API | Undisclosed | 4096 | Yes (100+ languages) | Per search unit |
| Voyage | rerank-2 | API | Undisclosed | 8000 | Yes (moderate) | Per query |
| Jina | jina-reranker-v2-base-multilingual | API + weights | 278M | 1024 | Yes (100+ languages) | Per query |
| BAAI | bge-reranker-v2-m3 | Open weights | 568M | 8192 | Yes (100+ languages) | Free (self-hosted) |
| MS MARCO | ms-marco-MiniLM-L-6-v2 | Open weights | 22M | 512 | English only | Free (self-hosted) |
| MS MARCO | ms-marco-MiniLM-L-12-v2 | Open weights | 33M | 512 | English only | Free (self-hosted) |

---

## Quality Benchmarks

### BEIR Benchmark (NDCG@10, zero-shot)

BEIR (Benchmarking IR) is the standard heterogeneous retrieval benchmark covering 18 diverse datasets. Numbers below are for the reranker applied on top of a strong first-stage retriever (typically BM25 or bi-encoder top-100).

| Model | BEIR Avg | MS MARCO | NQ | FiQA | SciFact | TREC-COVID | NFCorpus |
|-------|---------|----------|-----|------|---------|------------|----------|
| No reranker (BM25 baseline) | 0.428 | 0.228 | 0.329 | 0.236 | 0.665 | 0.594 | 0.325 |
| ms-marco-MiniLM-L-6-v2 | 0.481 | 0.390 | 0.479 | 0.301 | 0.689 | 0.748 | 0.342 |
| ms-marco-MiniLM-L-12-v2 | 0.489 | 0.397 | 0.491 | 0.312 | 0.695 | 0.756 | 0.349 |
| Jina reranker-v2 | 0.515 | 0.421 | 0.518 | 0.348 | 0.721 | 0.789 | 0.371 |
| bge-reranker-v2-m3 | 0.523 | 0.428 | 0.529 | 0.362 | 0.734 | 0.801 | 0.378 |
| Voyage rerank-2 | 0.528 | 0.433 | 0.535 | 0.371 | 0.739 | 0.809 | 0.384 |
| Cohere rerank-v3.5 | 0.531 | 0.437 | 0.541 | 0.378 | 0.742 | 0.814 | 0.389 |

Note: exact numbers vary by first-stage retriever and evaluation methodology. These are representative of the relative ordering observed across multiple independent benchmarks.

### Key Observations

1. **Cohere and Voyage lead** by a small margin (~1-2% NDCG over bge-reranker-v2-m3)
2. **bge-reranker-v2-m3 is the best open-weights option**, within 2% of commercial APIs
3. **MiniLM models trail by 5-10%** but are 20-50x faster
4. **Domain-specific performance varies**: on SciFact (scientific), the gap narrows; on FiQA (finance), it widens
5. **All rerankers significantly beat no-reranker baseline** (8-10% NDCG improvement)

---

## Cost Comparison

### API Pricing (as of Q1 2025)

| Provider | Pricing | Cost per 1K queries (20 docs each) | Monthly cost at 10K queries/day |
|----------|---------|-----------------------------------|-------------------------------|
| Cohere rerank-v3.5 | $2.00 per 1K search units | $2.00 | ~$600 |
| Voyage rerank-2 | $0.05 per 1M tokens | ~$0.50 | ~$150 |
| Jina reranker-v2 | $0.02 per 1K search units | $0.02 | ~$6 |

### Self-Hosted Cost (GPU)

| Model | GPU Required | Monthly GPU Cost (AWS) | Cost per 1K queries | Break-even vs API |
|-------|-------------|----------------------|--------------------|--------------------|
| ms-marco-MiniLM-L-6-v2 | CPU only | $50 (c5.xlarge) | ~$0.002 | Always cheaper |
| ms-marco-MiniLM-L-12-v2 | CPU only | $50 (c5.xlarge) | ~$0.003 | Always cheaper |
| bge-reranker-v2-m3 | T4 (16GB) | $250 (g4dn.xlarge) | ~$0.01 | <25K queries/day vs Cohere |
| Jina reranker-v2 | T4 (16GB) | $250 (g4dn.xlarge) | ~$0.01 | ~40K queries/day vs Jina API |

### Cost Decision Matrix

```
If queries/day < 1,000:
    Use Jina API (cheapest managed option, ~$0.60/month)

If queries/day 1,000 - 10,000:
    Self-host MiniLM on CPU ($50/month) OR use Jina API (~$6-60/month)

If queries/day 10,000 - 100,000:
    Self-host bge-reranker-v2-m3 on GPU (~$250/month)
    OR use Jina API (~$60-600/month)

If queries/day > 100,000:
    Self-host bge-reranker-v2-m3 on GPU cluster
    Multiple GPUs with batching: ~$750/month for 3x T4
```

---

## Latency Comparison

### API Latency (20 documents, average query length)

| Provider | P50 Latency | P99 Latency | Notes |
|----------|------------|------------|-------|
| Cohere rerank-v3.5 | ~250ms | ~800ms | Includes network overhead |
| Voyage rerank-2 | ~200ms | ~600ms | Generally faster API |
| Jina reranker-v2 | ~180ms | ~550ms | Competitive latency |

### Self-Hosted Latency (20 documents, GPU inference)

| Model | GPU | P50 Latency | P99 Latency | Throughput |
|-------|-----|------------|------------|------------|
| ms-marco-MiniLM-L-6-v2 | CPU (4 core) | ~100ms | ~180ms | ~200 queries/sec |
| ms-marco-MiniLM-L-6-v2 | T4 GPU | ~15ms | ~30ms | ~1500 queries/sec |
| ms-marco-MiniLM-L-12-v2 | T4 GPU | ~25ms | ~50ms | ~800 queries/sec |
| bge-reranker-v2-m3 | T4 GPU | ~120ms | ~250ms | ~170 queries/sec |
| bge-reranker-v2-m3 | A10G GPU | ~60ms | ~120ms | ~350 queries/sec |
| Jina reranker-v2 | T4 GPU | ~80ms | ~160ms | ~250 queries/sec |

### Latency vs. K (Number of Documents to Rerank)

```python
# Approximate latency scaling (ms) for self-hosted models on T4 GPU
def estimate_latency(model: str, k: int) -> float:
    base_ms_per_doc = {
        "ms-marco-MiniLM-L-6-v2": 0.75,
        "ms-marco-MiniLM-L-12-v2": 1.25,
        "bge-reranker-v2-m3": 6.0,
        "jina-reranker-v2": 4.0,
    }
    overhead_ms = 5.0  # Model loading, tokenization, etc.
    return overhead_ms + base_ms_per_doc.get(model, 5.0) * k

# Examples:
# MiniLM-L-6, K=20:  5 + 0.75*20 = 20ms
# MiniLM-L-6, K=100: 5 + 0.75*100 = 80ms
# bge-v2-m3, K=20:   5 + 6*20 = 125ms
# bge-v2-m3, K=100:  5 + 6*100 = 605ms
```

---

## API Code for Each Provider

### Cohere Rerank v3.5

```python
import cohere

co = cohere.ClientV2(api_key="your-cohere-key")


def cohere_rerank(
    query: str,
    documents: list[str],
    top_n: int = 5,
) -> list[dict]:
    """Rerank using Cohere Rerank v3.5."""
    response = co.rerank(
        model="rerank-v3.5",
        query=query,
        documents=documents,
        top_n=top_n,
        return_documents=True,
    )
    return [
        {
            "index": r.index,
            "score": r.relevance_score,
            "text": r.document.text,
        }
        for r in response.results
    ]


# Usage
results = cohere_rerank(
    query="How to configure row-level security in PostgreSQL?",
    documents=[
        "PostgreSQL RLS allows row-level access control policies...",
        "MySQL provides user-level GRANT and REVOKE statements...",
        "Row Level Security in PostgreSQL enables per-row filtering...",
    ],
    top_n=2,
)
for r in results:
    print(f"  [{r['index']}] score={r['score']:.4f}: {r['text'][:80]}...")
```

### Voyage Rerank-2

```python
import voyageai

vo = voyageai.Client(api_key="your-voyage-key")


def voyage_rerank(
    query: str,
    documents: list[str],
    top_k: int = 5,
) -> list[dict]:
    """Rerank using Voyage rerank-2."""
    response = vo.rerank(
        query=query,
        documents=documents,
        model="rerank-2",
        top_k=top_k,
    )
    return [
        {
            "index": r.index,
            "score": r.relevance_score,
            "text": documents[r.index],
        }
        for r in response.results
    ]


# Usage
results = voyage_rerank(
    query="How to configure row-level security in PostgreSQL?",
    documents=[
        "PostgreSQL RLS allows row-level access control policies...",
        "MySQL provides user-level GRANT and REVOKE statements...",
        "Row Level Security in PostgreSQL enables per-row filtering...",
    ],
    top_k=2,
)
```

### Jina Reranker v2

```python
import requests


def jina_rerank(
    query: str,
    documents: list[str],
    top_n: int = 5,
    api_key: str = "your-jina-key",
) -> list[dict]:
    """Rerank using Jina Reranker v2."""
    response = requests.post(
        "https://api.jina.ai/v1/rerank",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
        },
        json={
            "model": "jina-reranker-v2-base-multilingual",
            "query": query,
            "documents": documents,
            "top_n": top_n,
        },
    )
    response.raise_for_status()
    data = response.json()
    return [
        {
            "index": r["index"],
            "score": r["relevance_score"],
            "text": documents[r["index"]],
        }
        for r in data["results"]
    ]


# Usage
results = jina_rerank(
    query="How to configure row-level security in PostgreSQL?",
    documents=[
        "PostgreSQL RLS allows row-level access control policies...",
        "MySQL provides user-level GRANT and REVOKE statements...",
        "Row Level Security in PostgreSQL enables per-row filtering...",
    ],
    top_n=2,
)
```

### BAAI bge-reranker-v2-m3 (Self-Hosted)

```python
from sentence_transformers import CrossEncoder


def bge_rerank(
    query: str,
    documents: list[str],
    top_n: int = 5,
    model_name: str = "BAAI/bge-reranker-v2-m3",
) -> list[dict]:
    """Rerank using BGE reranker v2 m3 (local inference)."""
    model = CrossEncoder(model_name, max_length=1024)
    pairs = [(query, doc) for doc in documents]
    scores = model.predict(pairs, show_progress_bar=False)

    scored = [
        {"index": i, "score": float(s), "text": doc}
        for i, (doc, s) in enumerate(zip(documents, scores))
    ]
    scored.sort(key=lambda x: x["score"], reverse=True)
    return scored[:top_n]


# For production: load model once, reuse across queries
class BGEReranker:
    def __init__(self, model_name: str = "BAAI/bge-reranker-v2-m3"):
        self.model = CrossEncoder(model_name, max_length=1024)

    def rerank(self, query: str, documents: list[str], top_n: int = 5) -> list[dict]:
        pairs = [(query, doc) for doc in documents]
        scores = self.model.predict(pairs, batch_size=32, show_progress_bar=False)
        scored = [
            {"index": i, "score": float(s), "text": doc}
            for i, (doc, s) in enumerate(zip(documents, scores))
        ]
        scored.sort(key=lambda x: x["score"], reverse=True)
        return scored[:top_n]


# Initialize once
reranker = BGEReranker()

# Use in request handler
results = reranker.rerank(
    query="How to configure row-level security?",
    documents=candidate_texts,
    top_n=5,
)
```

### ms-marco-MiniLM (Self-Hosted)

```python
from sentence_transformers import CrossEncoder


class MiniLMReranker:
    """Lightweight reranker for latency-sensitive applications."""

    def __init__(self, variant: str = "L-6"):
        """
        Args:
            variant: "L-6" (22M params, faster) or "L-12" (33M params, better quality)
        """
        model_name = f"cross-encoder/ms-marco-MiniLM-{variant}-v2"
        self.model = CrossEncoder(model_name, max_length=512)

    def rerank(self, query: str, documents: list[str], top_n: int = 5) -> list[dict]:
        pairs = [(query, doc) for doc in documents]
        scores = self.model.predict(pairs, batch_size=64, show_progress_bar=False)
        scored = [
            {"index": i, "score": float(s), "text": doc}
            for i, (doc, s) in enumerate(zip(documents, scores))
        ]
        scored.sort(key=lambda x: x["score"], reverse=True)
        return scored[:top_n]


# Fast variant
fast_reranker = MiniLMReranker(variant="L-6")

# Quality variant
quality_reranker = MiniLMReranker(variant="L-12")
```

---

## Unified Reranker Interface

For production systems that may switch providers, use an abstraction layer:

```python
from abc import ABC, abstractmethod
from typing import Protocol


class RerankerResult:
    def __init__(self, index: int, score: float, text: str):
        self.index = index
        self.score = score
        self.text = text


class Reranker(ABC):
    @abstractmethod
    def rerank(
        self, query: str, documents: list[str], top_n: int = 5
    ) -> list[RerankerResult]:
        ...


class CohereReranker(Reranker):
    def __init__(self, api_key: str, model: str = "rerank-v3.5"):
        import cohere
        self.client = cohere.ClientV2(api_key=api_key)
        self.model = model

    def rerank(self, query, documents, top_n=5):
        resp = self.client.rerank(
            model=self.model, query=query, documents=documents,
            top_n=top_n, return_documents=True,
        )
        return [
            RerankerResult(r.index, r.relevance_score, r.document.text)
            for r in resp.results
        ]


class LocalCrossEncoderReranker(Reranker):
    def __init__(self, model_name: str = "cross-encoder/ms-marco-MiniLM-L-6-v2"):
        from sentence_transformers import CrossEncoder
        self.model = CrossEncoder(model_name)

    def rerank(self, query, documents, top_n=5):
        pairs = [(query, doc) for doc in documents]
        scores = self.model.predict(pairs, show_progress_bar=False)
        results = [
            RerankerResult(i, float(s), doc)
            for i, (doc, s) in enumerate(zip(documents, scores))
        ]
        results.sort(key=lambda r: r.score, reverse=True)
        return results[:top_n]


# Factory
def create_reranker(provider: str, **kwargs) -> Reranker:
    if provider == "cohere":
        return CohereReranker(api_key=kwargs["api_key"])
    elif provider == "local-minilm":
        return LocalCrossEncoderReranker("cross-encoder/ms-marco-MiniLM-L-6-v2")
    elif provider == "local-bge":
        return LocalCrossEncoderReranker("BAAI/bge-reranker-v2-m3")
    else:
        raise ValueError(f"Unknown provider: {provider}")
```

---

## Recommendation Matrix

| Scenario | Recommended Reranker | Why |
|----------|---------------------|-----|
| Prototyping / development | ms-marco-MiniLM-L-6-v2 | Free, fast, no API key needed |
| Production, English, latency <100ms | ms-marco-MiniLM-L-6-v2 on GPU | Fastest self-hosted option |
| Production, English, quality-first | bge-reranker-v2-m3 or Cohere | Best quality, acceptable latency |
| Multilingual production | bge-reranker-v2-m3 or Jina v2 | Best multilingual open models |
| Budget-constrained, API-only | Jina reranker-v2 | Cheapest API ($0.02/1K) |
| Enterprise, highest quality | Cohere rerank-v3.5 | Consistently top benchmark scores |
| Air-gapped / on-premises | bge-reranker-v2-m3 | Best self-hosted quality |
| Very long documents (>1K tokens) | bge-reranker-v2-m3 (8192 max) | Longest context window |
| Mobile / edge deployment | ms-marco-MiniLM-L-6-v2 (ONNX) | 22M params, runs on CPU |

---

## Migration Guide: Switching Providers

When switching rerankers, keep in mind:

1. **Score scales differ.** MiniLM outputs raw logits (-10 to +10), Cohere outputs 0.0-1.0, BGE outputs logits. Do not use absolute score thresholds -- use relative ranking.
2. **Re-run evaluations.** Switching rerankers can change optimal first-stage K and final N.
3. **Test edge cases.** Multilingual, very short queries, very long documents -- each model handles these differently.
4. **Update latency budgets.** A switch from MiniLM to BGE adds ~100ms per query.

---

## Common Pitfalls

1. **Choosing based on benchmarks alone.** BEIR numbers are zero-shot. Your domain may differ significantly. Always evaluate on your own data.
2. **Ignoring the cost of API rerankers at scale.** Cohere at 100K queries/day is ~$6,000/month. Self-hosting bge-reranker on a single GPU is ~$250/month.
3. **Using MiniLM for non-English content.** MiniLM models are English-only. They produce garbage scores for other languages. Use BGE or Jina for multilingual.
4. **Not accounting for max token limits.** MiniLM truncates at 512 tokens. If your documents average 800 tokens, 35% of content is invisible to the reranker. Use BGE (8192 tokens) for long documents.
5. **Running API rerankers synchronously.** Add timeout handling and fallback to unreranked results for API-based rerankers. Network failures should not break your search pipeline.

---

## References

- Cohere Rerank: https://docs.cohere.com/reference/rerank
- Voyage Rerank: https://docs.voyageai.com/docs/reranker
- Jina Reranker: https://jina.ai/reranker/
- BAAI bge-reranker: https://huggingface.co/BAAI/bge-reranker-v2-m3
- Sentence Transformers: https://www.sbert.net/docs/cross_encoder/pretrained_models.html
- BEIR Benchmark: https://github.com/beir-cellar/beir
- MTEB Leaderboard: https://huggingface.co/spaces/mteb/leaderboard
