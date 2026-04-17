# Embedding Models -- Comprehensive Landscape Guide

## Overview / TL;DR

Embedding models convert text into dense numerical vectors that capture semantic meaning. The choice of embedding model determines the ceiling of your RAG pipeline's retrieval quality -- no amount of re-ranking or query transformation can recover information lost by a poor embedding. This guide covers every major embedding model family (OpenAI, Voyage, Cohere, BGE, E5, Jina, Nomic, mxbai), their architectural differences (instruction-tuned vs not, query/document asymmetric modes), dimension ranges (256 to 4096), cost structures, and how to interpret MTEB leaderboard scores to select the right model for your use case.

---

## Why the Embedding Model Matters

The embedding model is the foundation of semantic search. It determines:

1. **Retrieval quality** -- whether the right chunks are retrieved for a given query.
2. **Storage cost** -- higher-dimensional embeddings consume more memory and disk.
3. **Latency** -- larger models are slower at inference, both locally and via API.
4. **Multilingual capability** -- some models handle 100+ languages, others are English-only.
5. **Domain fit** -- general-purpose models may underperform on specialized corpora (legal, medical, code).

A model that scores 65 on MTEB retrieval tasks versus one that scores 60 translates to roughly 8-12% more relevant documents in the top-5 results -- a meaningful difference in production.

---

## Model Families

### OpenAI text-embedding-3 Series

OpenAI's third-generation embedding models, released January 2024. Two variants: `text-embedding-3-small` (1536 dims, cheap) and `text-embedding-3-large` (3072 dims, higher quality). Both support Matryoshka dimension reduction via the `dimensions` parameter.

**Key features**:
- Native Matryoshka support: truncate to 256, 512, 1024, or any dimension at query time.
- 8191 max input tokens.
- No instruction prefix required (the model internally handles query vs document).
- Affordable at scale.

```python
from openai import OpenAI

client = OpenAI()

# Full dimensions (3072)
response = client.embeddings.create(
    model="text-embedding-3-large",
    input="How do I configure Kubernetes pod autoscaling?",
)
embedding_full = response.data[0].embedding  # len == 3072

# Reduced dimensions (256) -- Matryoshka truncation
response = client.embeddings.create(
    model="text-embedding-3-large",
    input="How do I configure Kubernetes pod autoscaling?",
    dimensions=256,
)
embedding_256 = response.data[0].embedding  # len == 256
```

**Batch embedding** (efficient for ingestion):

```python
from openai import OpenAI

client = OpenAI()


def embed_batch(texts: list[str], model: str = "text-embedding-3-small", dimensions: int | None = None) -> list[list[float]]:
    """Embed a batch of texts. OpenAI supports up to 2048 inputs per request."""
    all_embeddings = []
    batch_size = 2048

    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        kwargs = {"model": model, "input": batch}
        if dimensions is not None:
            kwargs["dimensions"] = dimensions
        response = client.embeddings.create(**kwargs)
        all_embeddings.extend([item.embedding for item in response.data])

    return all_embeddings


# Embed 10,000 chunks
chunks = ["chunk text here..."] * 10000
embeddings = embed_batch(chunks, model="text-embedding-3-small", dimensions=512)
```

### Voyage-3 Series

Voyage AI (acquired by Anthropic) offers a family of models tuned for different domains. The `voyage-3` general model and `voyage-3-large` are competitive with the best on MTEB.

**Key features**:
- Instruction-based asymmetric embedding (different prefixes for queries vs documents).
- `voyage-code-3` specifically tuned for code retrieval.
- `voyage-3-lite` for budget-conscious applications.
- Up to 32,000 token context for long documents.
- Supports a `output_dimension` parameter for Matryoshka-style truncation.

```python
import voyageai

client = voyageai.Client()

# Embed queries (retrieval optimized)
query_embeddings = client.embed(
    texts=["How does Kubernetes autoscaling work?"],
    model="voyage-3-large",
    input_type="query",
).embeddings

# Embed documents
doc_embeddings = client.embed(
    texts=[
        "Kubernetes Horizontal Pod Autoscaler automatically scales...",
        "The HPA controller adjusts replica counts based on metrics...",
    ],
    model="voyage-3-large",
    input_type="document",
).embeddings

# Reduced dimensions
reduced = client.embed(
    texts=["query text"],
    model="voyage-3-large",
    input_type="query",
    output_dimension=512,
).embeddings
```

### Cohere embed-v4

Cohere's latest embedding model with built-in search type optimization. Supports `search_document` and `search_query` input types, plus `classification` and `clustering`.

**Key features**:
- 1024 default dimensions, compressible to 256/512.
- 512 max input tokens (shorter than competitors).
- Explicit `input_type` parameter for asymmetric search.
- Strong multilingual performance.
- Supports binary embeddings for extreme compression.

```python
import cohere

co = cohere.ClientV2()

# Embed documents for search indexing
doc_response = co.embed(
    texts=[
        "Kubernetes autoscaling adjusts pod replicas based on CPU utilization.",
        "Docker containers share the host OS kernel for efficiency.",
    ],
    model="embed-v4.0",
    input_type="search_document",
    embedding_types=["float"],
)
doc_embeddings = doc_response.embeddings.float_

# Embed queries for retrieval
query_response = co.embed(
    texts=["How does K8s autoscaling work?"],
    model="embed-v4.0",
    input_type="search_query",
    embedding_types=["float"],
)
query_embedding = query_response.embeddings.float_[0]
```

### BGE-M3 (BAAI General Embedding)

Open-source model from BAAI (Beijing Academy of AI). Supports dense, sparse, and ColBERT-style multi-vector retrieval in a single model. Multilingual across 100+ languages.

**Key features**:
- 1024 dimensions (dense), plus sparse and multi-vector outputs.
- 8192 max input tokens.
- Free to run locally (Apache 2.0 license).
- Multi-granularity: dense for semantic, sparse for keyword, multi-vector for fine-grained.

```python
from FlagEmbedding import BGEM3FlagModel

model = BGEM3FlagModel("BAAI/bge-m3", use_fp16=True)

# Encode with all three retrieval methods
output = model.encode(
    ["How does Kubernetes autoscaling work?"],
    return_dense=True,
    return_sparse=True,
    return_colbert_vecs=True,
)

dense_embedding = output["dense_vecs"][0]     # shape: (1024,)
sparse_embedding = output["lexical_weights"]   # dict of token_id -> weight
colbert_vecs = output["colbert_vecs"]          # token-level embeddings

# Simple dense-only usage with sentence-transformers
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("BAAI/bge-m3")
embeddings = model.encode(
    ["query text", "document text"],
    normalize_embeddings=True,
)
```

### E5-Mistral-7B-Instruct

A large instruction-tuned embedding model built on the Mistral-7B base. Requires instruction prefixes. High quality but resource-intensive (7B parameters, needs GPU with 16+ GB VRAM).

**Key features**:
- 4096 dimensions (highest among common models).
- Instruction-tuned: requires specific task prefixes.
- 32,768 max input tokens.
- Near state-of-the-art on MTEB.

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("intfloat/e5-mistral-7b-instruct")

# Queries require instruction prefix
queries = [
    "Instruct: Given a technical question, retrieve relevant documentation\nQuery: How does Kubernetes autoscaling work?"
]

# Documents use no prefix (or a generic one)
documents = [
    "Kubernetes HPA automatically adjusts replica counts based on observed CPU or custom metrics.",
]

query_embeddings = model.encode(queries, normalize_embeddings=True)
doc_embeddings = model.encode(documents, normalize_embeddings=True)
```

### Jina Embeddings v3

Jina AI's latest model with 8192 token context, multilingual support, and task-specific LoRA adapters.

**Key features**:
- 1024 dimensions (default), Matryoshka truncation support.
- 8192 max input tokens.
- Task-specific LoRA adapters: `retrieval.query`, `retrieval.passage`, `text-matching`, `classification`, `separation`.
- Multilingual across 30+ languages.
- Late chunking support (see `late-chunking/` KB entry).

```python
from transformers import AutoModel

model = AutoModel.from_pretrained("jinaai/jina-embeddings-v3", trust_remote_code=True)

# Query embedding with task-specific adapter
query_embeddings = model.encode(
    ["How does Kubernetes autoscaling work?"],
    task="retrieval.query",
)

# Document embedding
doc_embeddings = model.encode(
    ["Kubernetes HPA adjusts pod replicas based on CPU utilization."],
    task="retrieval.passage",
)

# Via Jina API
import requests

response = requests.post(
    "https://api.jina.ai/v1/embeddings",
    headers={"Authorization": "Bearer jina_YOUR_KEY"},
    json={
        "model": "jina-embeddings-v3",
        "task": "retrieval.query",
        "input": ["How does K8s autoscaling work?"],
        "dimensions": 512,
    },
)
embedding = response.json()["data"][0]["embedding"]
```

### Nomic Embed

Open-source, fully auditable embedding model with Matryoshka support. Available via API or self-hosted.

**Key features**:
- 768 dimensions (default), truncatable to 256/512 via Matryoshka.
- 8192 max input tokens.
- Requires `search_query:` or `search_document:` prefix.
- Fully open-source (weights + training code + data).
- Competitive with proprietary models at lower cost.

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("nomic-ai/nomic-embed-text-v1.5", trust_remote_code=True)

# Queries require prefix
queries = ["search_query: How does Kubernetes autoscaling work?"]
docs = ["search_document: Kubernetes HPA adjusts pod replicas based on CPU."]

query_emb = model.encode(queries, normalize_embeddings=True)
doc_emb = model.encode(docs, normalize_embeddings=True)

# Via Nomic API
from nomic import embed

output = embed.text(
    texts=["How does Kubernetes autoscaling work?"],
    model="nomic-embed-text-v1.5",
    task_type="search_query",
    dimensionality=256,
)
```

### mxbai-embed-large

Mixed Bread AI's open-source model. Strong MTEB performance with Matryoshka support.

**Key features**:
- 1024 dimensions, Matryoshka truncation to 512/256.
- 512 max input tokens.
- Instruction prefix for queries: `Represent this sentence for searching relevant passages:`.
- Apache 2.0 license.

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("mixedbread-ai/mxbai-embed-large-v1")

# Query (with instruction)
query_emb = model.encode(
    ["Represent this sentence for searching relevant passages: How does K8s autoscaling work?"],
    normalize_embeddings=True,
)

# Document (no instruction)
doc_emb = model.encode(
    ["Kubernetes HPA adjusts pod replicas based on CPU utilization."],
    normalize_embeddings=True,
)
```

---

## Instruction-Tuned vs Standard Models

### What "Instruction-Tuned" Means

Instruction-tuned embedding models expect a task prefix that tells the model how to interpret the input. This allows a single model to serve multiple tasks (search, classification, clustering) by adjusting behavior at inference time.

| Model | Instruction Required? | Query Prefix | Document Prefix |
|-------|----------------------|--------------|-----------------|
| OpenAI text-embedding-3 | No (internal) | None | None |
| Voyage-3 | Via `input_type` param | `input_type="query"` | `input_type="document"` |
| Cohere embed-v4 | Via `input_type` param | `input_type="search_query"` | `input_type="search_document"` |
| BGE-M3 | Optional | `"Represent this sentence..."` | None |
| E5-Mistral | Yes (required) | `"Instruct: ...\nQuery: ..."` | None or generic |
| Jina v3 | Via `task` param | `task="retrieval.query"` | `task="retrieval.passage"` |
| Nomic Embed | Yes (prefix) | `"search_query: ..."` | `"search_document: ..."` |
| mxbai-embed | Yes (query only) | `"Represent this sentence..."` | None |

### Asymmetric vs Symmetric Search

**Asymmetric search**: Queries and documents are structurally different. A short query ("What is autoscaling?") should match a long explanatory passage. Use query/document modes.

**Symmetric search**: Both inputs are similar in structure. Two sentences that are paraphrases should be close. Use the same mode for both.

For RAG, you almost always want **asymmetric search** -- your queries are short questions and your documents are longer passages.

---

## Dimension Ranges and Tradeoffs

| Model | Default Dims | Min (Matryoshka) | Max | Bytes per Vector |
|-------|-------------|------------------|-----|-----------------|
| text-embedding-3-small | 1536 | 256 | 1536 | 6,144 |
| text-embedding-3-large | 3072 | 256 | 3072 | 12,288 |
| Voyage-3 | 1024 | 256 | 1024 | 4,096 |
| Voyage-3-large | 1024 | 256 | 2048 | 4,096-8,192 |
| Cohere embed-v4 | 1024 | 256 | 1024 | 4,096 |
| BGE-M3 | 1024 | N/A | 1024 | 4,096 |
| E5-Mistral-7B | 4096 | N/A | 4096 | 16,384 |
| Jina v3 | 1024 | 64 | 1024 | 4,096 |
| Nomic Embed v1.5 | 768 | 64 | 768 | 3,072 |
| mxbai-embed-large | 1024 | 256 | 1024 | 4,096 |

**Storage math**: Each float32 value = 4 bytes. A 1024-dim vector = 4,096 bytes = 4 KB. One million vectors at 1024 dims = ~3.8 GB (just vectors, before index overhead).

See `dimension-selection.md` in this directory for detailed dimension optimization guidance.

---

## Cost Comparison (2025 API Pricing)

| Model | Price per 1M Tokens | Price to Embed 100K Chunks (avg 300 tokens) | Monthly Cost at 10K queries/day |
|-------|--------------------|--------------------------------------------|-------------------------------|
| text-embedding-3-small | $0.02 | $0.60 | $0.18 |
| text-embedding-3-large | $0.13 | $3.90 | $1.17 |
| Voyage-3 | $0.06 | $1.80 | $0.54 |
| Voyage-3-large | $0.18 | $5.40 | $1.62 |
| Voyage-code-3 | $0.18 | $5.40 | $1.62 |
| Cohere embed-v4 | $0.10 | $3.00 | $0.90 |
| Jina v3 (API) | $0.02 | $0.60 | $0.18 |
| Local (any open-source) | ~$0.00 | ~$0.00 | GPU instance cost |

**GPU cost for local models**: A single A10G GPU instance on AWS ($0.76/hr) can embed ~5,000 chunks/second with BGE-M3. That is 432M chunks/day, meaning the per-chunk cost is effectively $0.0000015 -- negligible versus API pricing.

---

## MTEB Leaderboard Interpretation

The MTEB (Massive Text Embedding Benchmark) leaderboard evaluates models across multiple task categories. For RAG, focus on the **Retrieval** subset, not the overall average.

### Key MTEB Categories

| Category | Relevance to RAG | What It Measures |
|----------|------------------|------------------|
| Retrieval | Critical | Query-document matching (your core RAG task) |
| STS (Semantic Similarity) | Moderate | Sentence pair similarity scoring |
| Clustering | Low | Grouping similar documents |
| Classification | Low | Text categorization via embeddings |
| Reranking | Moderate | Ordering candidates by relevance |
| PairClassification | Low | Binary pair relationships |

### How to Read MTEB Scores

1. **Filter to Retrieval tasks** on the leaderboard. A model ranked #1 overall but #15 on retrieval is a poor choice for RAG.
2. **Check model size**. A 7B-parameter model scoring 68 is impressive but may be impractical for your infrastructure. A 110M model scoring 65 is better if latency matters.
3. **Check sequence length**. Models evaluated on MTEB with 512-token limits may underperform on your 2000-token chunks.
4. **Check the specific datasets**. If your domain is similar to one of the MTEB retrieval datasets (NQ, HotpotQA, FEVER, MS MARCO), the scores are directly relevant. If your domain is medical or legal text, MTEB scores are a rough proxy.
5. **Beware of contamination**. Some models train on MTEB datasets. Cross-reference with non-MTEB benchmarks like BEIR.

### Approximate MTEB Retrieval Scores (2025)

| Model | MTEB Retrieval Avg | Overall Avg | Parameters |
|-------|-------------------|-------------|------------|
| Voyage-3-large | 69.2 | 67.8 | Unknown (API) |
| E5-Mistral-7B-Instruct | 68.4 | 66.6 | 7B |
| text-embedding-3-large | 66.1 | 64.6 | Unknown (API) |
| Cohere embed-v4 | 65.8 | 66.2 | Unknown (API) |
| BGE-M3 | 65.3 | 64.5 | 568M |
| Jina v3 | 64.8 | 65.1 | 570M |
| mxbai-embed-large | 64.1 | 64.7 | 335M |
| Nomic Embed v1.5 | 62.8 | 62.2 | 137M |
| text-embedding-3-small | 61.5 | 62.3 | Unknown (API) |

Note: Scores shift as the leaderboard updates. Always check the live leaderboard at https://huggingface.co/spaces/mteb/leaderboard for current numbers.

---

## Unified Embedding Interface

A production system should abstract the embedding provider behind a common interface so you can swap models without changing application code.

```python
from abc import ABC, abstractmethod
import numpy as np


class EmbeddingProvider(ABC):
    """Unified interface for embedding models."""

    @abstractmethod
    def embed_query(self, text: str) -> list[float]:
        """Embed a single query for retrieval."""
        ...

    @abstractmethod
    def embed_documents(self, texts: list[str]) -> list[list[float]]:
        """Embed a batch of documents for indexing."""
        ...

    @property
    @abstractmethod
    def dimensions(self) -> int:
        """Return the embedding dimension."""
        ...


class OpenAIEmbeddings(EmbeddingProvider):
    def __init__(self, model: str = "text-embedding-3-small", dims: int | None = None):
        from openai import OpenAI
        self.client = OpenAI()
        self.model = model
        self._dims = dims or (1536 if "small" in model else 3072)

    def embed_query(self, text: str) -> list[float]:
        kwargs = {"model": self.model, "input": [text]}
        if self._dims:
            kwargs["dimensions"] = self._dims
        resp = self.client.embeddings.create(**kwargs)
        return resp.data[0].embedding

    def embed_documents(self, texts: list[str]) -> list[list[float]]:
        all_embs = []
        for i in range(0, len(texts), 2048):
            batch = texts[i:i + 2048]
            kwargs = {"model": self.model, "input": batch}
            if self._dims:
                kwargs["dimensions"] = self._dims
            resp = self.client.embeddings.create(**kwargs)
            all_embs.extend([d.embedding for d in resp.data])
        return all_embs

    @property
    def dimensions(self) -> int:
        return self._dims


class VoyageEmbeddings(EmbeddingProvider):
    def __init__(self, model: str = "voyage-3", dims: int | None = None):
        import voyageai
        self.client = voyageai.Client()
        self.model = model
        self._dims = dims or 1024

    def embed_query(self, text: str) -> list[float]:
        kwargs = {"texts": [text], "model": self.model, "input_type": "query"}
        if self._dims:
            kwargs["output_dimension"] = self._dims
        return self.client.embed(**kwargs).embeddings[0]

    def embed_documents(self, texts: list[str]) -> list[list[float]]:
        all_embs = []
        for i in range(0, len(texts), 128):
            batch = texts[i:i + 128]
            kwargs = {"texts": batch, "model": self.model, "input_type": "document"}
            if self._dims:
                kwargs["output_dimension"] = self._dims
            result = self.client.embed(**kwargs)
            all_embs.extend(result.embeddings)
        return all_embs

    @property
    def dimensions(self) -> int:
        return self._dims


class LocalEmbeddings(EmbeddingProvider):
    def __init__(self, model_name: str = "BAAI/bge-m3", dims: int | None = None):
        from sentence_transformers import SentenceTransformer
        self.model = SentenceTransformer(model_name, trust_remote_code=True)
        self._dims = dims or self.model.get_sentence_embedding_dimension()

    def embed_query(self, text: str) -> list[float]:
        emb = self.model.encode([text], normalize_embeddings=True)[0]
        if self._dims and self._dims < len(emb):
            emb = emb[:self._dims]
            emb = emb / np.linalg.norm(emb)
        return emb.tolist()

    def embed_documents(self, texts: list[str]) -> list[list[float]]:
        embs = self.model.encode(texts, normalize_embeddings=True, batch_size=256)
        if self._dims and self._dims < embs.shape[1]:
            embs = embs[:, :self._dims]
            norms = np.linalg.norm(embs, axis=1, keepdims=True)
            embs = embs / norms
        return embs.tolist()

    @property
    def dimensions(self) -> int:
        return self._dims
```

---

## Model Selection Decision Framework

```
What is your primary constraint?
  |
  +---> Budget (minimize API cost)
  |     --> text-embedding-3-small ($0.02/M tokens)
  |         or local BGE-M3 (free, needs GPU)
  |
  +---> Quality (maximize retrieval accuracy)
  |     --> Voyage-3-large or E5-Mistral-7B-Instruct
  |         (check MTEB retrieval scores for your domain)
  |
  +---> Multilingual
  |     --> BGE-M3 (100+ languages, free)
  |         or Cohere embed-v4 (API, strong multilingual)
  |
  +---> Code retrieval
  |     --> Voyage-code-3 (specialized for code)
  |         or E5-Mistral-7B-Instruct (strong general + code)
  |
  +---> Self-hosted / air-gapped
  |     --> BGE-M3 (Apache 2.0, 568M params, runs on single GPU)
  |         or Nomic Embed (fully open, 137M params, runs on CPU)
  |
  +---> Long documents (>2K tokens per chunk)
  |     --> Jina v3 (8192 tokens) or Voyage-3 (32K tokens)
  |         Avoid: Cohere embed-v4 (512 token limit)
  |
  +---> Low latency / edge deployment
        --> Nomic Embed v1.5 (137M params, fast on CPU)
            or text-embedding-3-small (API, fast)
```

---

## Common Pitfalls

1. **Mixing query and document modes.** If a model supports asymmetric search, embedding both queries and documents with the same mode degrades retrieval by 5-15%. Always use the correct mode for each.
2. **Ignoring the token limit.** Text beyond the model's max tokens is silently truncated. A 10,000-token document embedded with a 512-token model loses 95% of its content. Chunk first, embed second.
3. **Choosing by overall MTEB score instead of retrieval score.** A model can rank high on classification and clustering but poorly on retrieval. Filter the leaderboard to retrieval tasks.
4. **Using the same model for all domains without evaluation.** A model that excels on general web text may underperform on medical literature or legal contracts. Always evaluate on your specific corpus.
5. **Not normalizing embeddings before cosine similarity.** Some models return unnormalized vectors. Cosine similarity on unnormalized vectors gives different results than on L2-normalized vectors. Check the model's documentation.
6. **Forgetting that re-embedding is required when switching models.** Embeddings from different models are incompatible. Switching models means re-embedding your entire corpus.

---

## Production Checklist

- [ ] Embedding model selected based on domain evaluation, not just MTEB scores
- [ ] Query/document modes correctly applied for asymmetric search
- [ ] Batch embedding implemented for ingestion efficiency
- [ ] Embedding dimension chosen based on quality/storage tradeoff analysis
- [ ] Token limits verified against actual chunk sizes
- [ ] Fallback provider configured in case primary API is unavailable
- [ ] Embedding costs estimated for both ingestion and query-time
- [ ] Model version pinned to avoid silent updates changing embedding space
- [ ] Embeddings normalized (L2) before storage if required by similarity metric
- [ ] Provider abstraction layer allows model swaps without application changes

---

## References

- MTEB Leaderboard -- https://huggingface.co/spaces/mteb/leaderboard
- OpenAI Embeddings Guide -- https://platform.openai.com/docs/guides/embeddings
- Voyage AI Documentation -- https://docs.voyageai.com/
- Cohere Embed API -- https://docs.cohere.com/reference/embed
- BGE-M3 Paper -- https://arxiv.org/abs/2402.03216
- E5-Mistral Paper -- https://arxiv.org/abs/2401.00368
- Jina Embeddings v3 -- https://jina.ai/embeddings/
- Nomic Embed -- https://blog.nomic.ai/posts/nomic-embed-text-v1
- mxbai-embed -- https://huggingface.co/mixedbread-ai/mxbai-embed-large-v1
