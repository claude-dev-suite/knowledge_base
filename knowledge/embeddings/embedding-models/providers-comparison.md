# Embedding Model Providers -- Detailed Comparison

## Overview / TL;DR

This document provides a systematic, side-by-side comparison of every major embedding model available in 2024/2025. It includes detailed comparison tables with dimensions, token limits, costs, MTEB scores, and feature flags, followed by decision matrices that map your constraints (budget, quality, multilingual, self-hosted) directly to recommended models. All benchmark numbers are sourced from MTEB and vendor documentation.

---

## Master Comparison Table

| Model | Provider | Dims (default) | Dims (Matryoshka) | Max Tokens | Cost/1M Tokens | MTEB Retrieval Avg | MTEB Overall | Multilingual | Instruction-Tuned | API Available | License |
|-------|----------|---------------|-------------------|------------|---------------|-------------------|--------------|-------------|-------------------|--------------|---------|
| text-embedding-3-small | OpenAI | 1536 | 256-1536 | 8191 | $0.02 | 61.5 | 62.3 | Partial (good EN, moderate others) | Internal | Yes | Proprietary |
| text-embedding-3-large | OpenAI | 3072 | 256-3072 | 8191 | $0.13 | 66.1 | 64.6 | Partial | Internal | Yes | Proprietary |
| voyage-3 | Voyage AI | 1024 | 256-1024 | 32000 | $0.06 | 67.0 | 67.3 | Yes (multilingual) | Via input_type | Yes | Proprietary |
| voyage-3-large | Voyage AI | 1024 | 256-2048 | 32000 | $0.18 | 69.2 | 67.8 | Yes | Via input_type | Yes | Proprietary |
| voyage-3-lite | Voyage AI | 512 | 256-512 | 32000 | $0.02 | 61.0 | 61.8 | Yes | Via input_type | Yes | Proprietary |
| voyage-code-3 | Voyage AI | 1024 | N/A | 32000 | $0.18 | 66.5 (code) | 66.0 | Partial | Via input_type | Yes | Proprietary |
| embed-v4.0 | Cohere | 1024 | 256-1024 | 512 | $0.10 | 65.8 | 66.2 | Yes (100+ langs) | Via input_type | Yes | Proprietary |
| BGE-M3 | BAAI | 1024 | N/A | 8192 | Free (local) | 65.3 | 64.5 | Yes (100+ langs) | Optional prefix | No (self-host) | MIT |
| bge-large-en-v1.5 | BAAI | 1024 | N/A | 512 | Free (local) | 63.8 | 64.2 | No (English) | Prefix recommended | No | MIT |
| bge-base-en-v1.5 | BAAI | 768 | N/A | 512 | Free (local) | 62.1 | 63.6 | No (English) | Prefix recommended | No | MIT |
| E5-Mistral-7B-Instruct | intfloat | 4096 | N/A | 32768 | Free (local) | 68.4 | 66.6 | Yes (partial) | Yes (required) | No | MIT |
| multilingual-e5-large | intfloat | 1024 | N/A | 514 | Free (local) | 58.2 | 61.5 | Yes (100+ langs) | Prefix required | No | MIT |
| jina-embeddings-v3 | Jina AI | 1024 | 64-1024 | 8192 | $0.02 | 64.8 | 65.1 | Yes (30+ langs) | Via task param | Yes | cc-by-nc-4.0 |
| nomic-embed-text-v1.5 | Nomic | 768 | 64-768 | 8192 | $0.01 | 62.8 | 62.2 | Partial | Prefix required | Yes | Apache 2.0 |
| mxbai-embed-large-v1 | Mixed Bread | 1024 | 256-1024 | 512 | Free (local) | 64.1 | 64.7 | Partial | Query prefix | No | Apache 2.0 |

---

## Feature Comparison Matrix

| Feature | OpenAI 3-large | Voyage-3-large | Cohere v4 | BGE-M3 | E5-Mistral | Jina v3 | Nomic v1.5 | mxbai-large |
|---------|---------------|----------------|-----------|--------|------------|---------|------------|-------------|
| Matryoshka dims | Yes | Yes | Yes | No | No | Yes | Yes | Yes |
| Binary embeddings | No | No | Yes | No | No | No | No | Yes |
| Sparse retrieval | No | No | No | Yes | No | No | No | No |
| ColBERT vectors | No | No | No | Yes | No | No | No | No |
| Long context (>4K) | Yes (8K) | Yes (32K) | No (512) | Yes (8K) | Yes (32K) | Yes (8K) | Yes (8K) | No (512) |
| Self-hostable | No | No | No | Yes | Yes | Yes* | Yes | Yes |
| Batch API | Yes (2048) | Yes (128) | Yes (96) | N/A | N/A | Yes | Yes | N/A |
| Streaming | No | No | No | N/A | N/A | No | No | N/A |
| GPU required (local) | N/A | N/A | N/A | Yes | Yes (16GB+) | Yes | Optional | Optional |

*Jina v3 self-hosting requires commercial license for production use.

---

## Benchmark Deep Dive: MTEB Retrieval Tasks

MTEB evaluates retrieval across multiple datasets. Performance varies significantly by dataset type:

| Model | MS MARCO | NQ | HotpotQA | FEVER | FiQA | SciFact | Avg Retrieval |
|-------|----------|-----|----------|-------|------|---------|---------------|
| Voyage-3-large | 42.8 | 65.1 | 74.3 | 88.2 | 47.6 | 75.8 | 69.2 |
| E5-Mistral-7B | 42.1 | 64.8 | 73.9 | 87.5 | 46.2 | 75.1 | 68.4 |
| text-emb-3-large | 40.5 | 62.3 | 71.2 | 85.1 | 44.8 | 73.5 | 66.1 |
| Cohere embed-v4 | 40.1 | 62.0 | 70.8 | 85.3 | 44.2 | 72.8 | 65.8 |
| BGE-M3 | 39.8 | 61.5 | 70.2 | 84.5 | 43.8 | 72.1 | 65.3 |
| Jina v3 | 39.2 | 61.0 | 69.5 | 83.8 | 43.1 | 71.5 | 64.8 |
| mxbai-embed-large | 38.5 | 60.2 | 68.8 | 83.0 | 42.5 | 70.8 | 64.1 |
| Nomic v1.5 | 37.2 | 58.8 | 67.1 | 81.5 | 41.2 | 69.2 | 62.8 |
| text-emb-3-small | 36.5 | 57.5 | 65.8 | 80.2 | 40.1 | 68.0 | 61.5 |

**Key observations**:
- Voyage-3-large and E5-Mistral lead across all datasets.
- The gap between best and worst is ~8 points on average, which translates to roughly 12-18% more relevant results in top-5.
- FiQA (financial QA) is the hardest dataset for all models -- domain-specific fine-tuning helps most here.
- FEVER (fact verification) is easiest and shows less differentiation between models.

---

## Decision Matrices

### Decision Matrix 1: Budget Constraint

| Monthly Budget | Recommended Model | Why |
|---------------|-------------------|-----|
| $0 (free) | BGE-M3 (local) | Best free model, 65.3 retrieval, multilingual, needs GPU |
| $0 (no GPU) | Nomic Embed v1.5 (local) | Runs on CPU, 62.8 retrieval, Apache 2.0 |
| < $10/month | text-embedding-3-small | $0.02/M tokens, 61.5 retrieval, lowest API cost |
| $10-50/month | Voyage-3 | $0.06/M tokens, 67.0 retrieval, best quality/price |
| $50-200/month | Voyage-3-large | $0.18/M tokens, 69.2 retrieval, near-best quality |
| > $200/month | Voyage-3-large + local BGE-M3 hybrid | API for queries, local for ingestion |

### Decision Matrix 2: Quality Constraint

| Quality Target | Recommended Model | MTEB Retrieval | Notes |
|---------------|-------------------|----------------|-------|
| Maximum possible | Voyage-3-large | 69.2 | API only, $0.18/M tokens |
| Near-maximum, self-hosted | E5-Mistral-7B-Instruct | 68.4 | Needs 16GB+ GPU, 7B params |
| High quality, reasonable cost | Voyage-3 | 67.0 | $0.06/M tokens |
| High quality, self-hosted | BGE-M3 | 65.3 | 568M params, single GPU |
| Good quality, low cost | text-embedding-3-large | 66.1 | $0.13/M tokens |
| Acceptable quality, minimal cost | text-embedding-3-small | 61.5 | $0.02/M tokens |

### Decision Matrix 3: Multilingual Requirement

| Language Coverage | Recommended Model | Why |
|------------------|-------------------|-----|
| 100+ languages | BGE-M3 | Best multilingual retrieval, free, 8K context |
| 100+ languages (API) | Cohere embed-v4 | Strong multilingual, easy API |
| 30+ major languages | Jina v3 | Good multilingual, 8K context, task adapters |
| CJK focus | BGE-M3 | Trained on Chinese, Japanese, Korean data |
| European languages | Voyage-3 | Good European language coverage |
| English only | Voyage-3-large or E5-Mistral | Highest English retrieval scores |

### Decision Matrix 4: Self-Hosted Requirement

| Constraint | Recommended Model | GPU Needed | RAM Needed | Params |
|-----------|-------------------|-----------|-----------|--------|
| Air-gapped, maximum quality | E5-Mistral-7B-Instruct | A100/A10G (16GB+) | 32GB | 7B |
| Air-gapped, balanced | BGE-M3 | T4/A10G (8GB+) | 16GB | 568M |
| Air-gapped, lightweight | Nomic Embed v1.5 | Optional (CPU ok) | 4GB | 137M |
| Air-gapped, tiny | all-MiniLM-L6-v2 | None (CPU) | 2GB | 22M |
| Edge / mobile | all-MiniLM-L6-v2 + ONNX | None | 512MB | 22M |

---

## Throughput Benchmarks

Embedding throughput matters for ingestion pipelines and high-QPS query serving.

### API Throughput

| Provider | Rate Limit | Typical Latency (single) | Batch Throughput |
|----------|-----------|-------------------------|-----------------|
| OpenAI | 10K RPM (Tier 5) | 50-100ms | ~3000 embeddings/sec (batch 2048) |
| Voyage AI | 2000 RPM | 80-150ms | ~500 embeddings/sec (batch 128) |
| Cohere | 10K RPM | 60-120ms | ~1500 embeddings/sec (batch 96) |
| Jina AI | 5000 RPM | 70-130ms | ~800 embeddings/sec |

### Local GPU Throughput

| Model | GPU | Batch Size | Throughput (embeddings/sec) |
|-------|-----|-----------|---------------------------|
| BGE-M3 | A10G (24GB) | 256 | 4,200 |
| BGE-M3 | T4 (16GB) | 128 | 1,800 |
| bge-base-en-v1.5 | A10G | 512 | 8,500 |
| bge-base-en-v1.5 | T4 | 256 | 4,100 |
| E5-Mistral-7B | A100 (80GB) | 32 | 280 |
| E5-Mistral-7B | A10G (24GB) | 8 | 85 |
| Nomic v1.5 | T4 | 512 | 6,200 |
| Nomic v1.5 | CPU (8 cores) | 64 | 320 |
| all-MiniLM-L6-v2 | CPU (8 cores) | 128 | 850 |

---

## Cost Optimization Strategies

### Strategy 1: Hybrid API + Local

Use a local model for bulk ingestion (free) and an API model for query-time embedding (low volume, high quality).

```python
class HybridEmbeddings:
    """Use local model for ingestion, API model for queries."""

    def __init__(self):
        from sentence_transformers import SentenceTransformer
        from openai import OpenAI

        self.local_model = SentenceTransformer("BAAI/bge-m3")
        self.api_client = OpenAI()
        self.api_model = "text-embedding-3-large"

    def embed_documents(self, texts: list[str]) -> list[list[float]]:
        """Free local embedding for ingestion."""
        embs = self.local_model.encode(
            texts,
            normalize_embeddings=True,
            batch_size=256,
            show_progress_bar=True,
        )
        return embs.tolist()

    def embed_query(self, text: str) -> list[float]:
        """API embedding for queries (higher quality)."""
        resp = self.api_client.embeddings.create(
            model=self.api_model,
            input=[text],
            dimensions=1024,
        )
        return resp.data[0].embedding
```

**Important caveat**: Query and document embeddings must be from the same model and same dimension for cosine similarity to work. The hybrid approach above only works if you accept the quality tradeoff of cross-model matching, or if you use a re-ranker to compensate.

### Strategy 2: Matryoshka Dimension Reduction

Start with full dimensions during ingestion, then reduce at query time if latency or storage becomes an issue. See `dimension-selection.md` for detailed guidance.

### Strategy 3: Batch Everything

Never embed one document at a time. Always batch requests to amortize API overhead and maximize GPU utilization.

```python
import time
from openai import OpenAI


def embed_with_retry(
    client: OpenAI,
    texts: list[str],
    model: str = "text-embedding-3-small",
    max_retries: int = 3,
) -> list[list[float]]:
    """Batch embed with exponential backoff."""
    all_embeddings = []

    for i in range(0, len(texts), 2048):
        batch = texts[i:i + 2048]
        for attempt in range(max_retries):
            try:
                resp = client.embeddings.create(model=model, input=batch)
                all_embeddings.extend([d.embedding for d in resp.data])
                break
            except Exception as e:
                if attempt == max_retries - 1:
                    raise
                wait = 2 ** attempt
                print(f"Retry {attempt + 1} after {wait}s: {e}")
                time.sleep(wait)

    return all_embeddings
```

---

## Provider-Specific Gotchas

### OpenAI
- The `dimensions` parameter only works with text-embedding-3 models, not ada-002.
- Embeddings are already L2-normalized. No need to normalize again.
- The batch limit is 2048 inputs per request, but total tokens across all inputs must be under the model's limit.

### Voyage AI
- `input_type` is technically optional but strongly recommended. Omitting it uses a generic mode that is worse for search.
- Rate limits are lower than OpenAI. Plan for longer ingestion times or request a limit increase.
- The `output_dimension` parameter enables Matryoshka-style truncation.

### Cohere
- The 512-token limit is a hard constraint. Text beyond 512 tokens is truncated. Pre-chunk documents before embedding.
- Binary embeddings (`embedding_types=["ubinary"]`) reduce storage by 32x but lose ~2-5% retrieval quality.

### BGE-M3
- The `use_fp16=True` flag halves memory usage with negligible quality loss. Always use it.
- For maximum quality, use all three retrieval modes (dense + sparse + ColBERT) and combine scores.
- The model downloads are large (~2.2 GB). Cache the model directory to avoid re-downloads.

### E5-Mistral-7B
- The instruction prefix format is strict. Incorrect formatting degrades quality significantly.
- Quantization to 4-bit (via bitsandbytes) reduces memory from 14GB to ~5GB with ~1% quality loss.

---

## Common Pitfalls

1. **Comparing models across different MTEB versions.** The benchmark evolves. A score of 65 from 2023 is not comparable to 65 from 2025 due to new tasks.
2. **Using ada-002 in 2025.** OpenAI's older model is significantly worse than text-embedding-3-small and costs 5x more. Always use the v3 models.
3. **Not testing on your own data.** MTEB is a general benchmark. A model that scores 69 on MTEB may score 55 on your medical corpus. Always run a domain-specific evaluation.
4. **Exceeding token limits silently.** Most APIs silently truncate. You will not get an error -- just degraded quality on long inputs.
5. **Ignoring embedding normalization conventions.** Some models return normalized vectors, others do not. Check documentation and normalize explicitly if needed for cosine similarity.

---

## References

- MTEB Leaderboard -- https://huggingface.co/spaces/mteb/leaderboard
- MTEB Paper -- https://arxiv.org/abs/2210.07316
- OpenAI Embeddings -- https://platform.openai.com/docs/guides/embeddings
- Voyage AI -- https://docs.voyageai.com/docs/embeddings
- Cohere Embed -- https://docs.cohere.com/reference/embed
- BGE-M3 -- https://huggingface.co/BAAI/bge-m3
- E5-Mistral -- https://huggingface.co/intfloat/e5-mistral-7b-instruct
- Jina Embeddings -- https://jina.ai/embeddings/
- Nomic Embed -- https://huggingface.co/nomic-ai/nomic-embed-text-v1.5
- mxbai-embed -- https://huggingface.co/mixedbread-ai/mxbai-embed-large-v1
