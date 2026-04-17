# Reranking for RAG -- Comprehensive Guide

## TL;DR

Reranking is a second-stage retrieval step that re-scores a candidate set of documents using a more powerful (and slower) model. The first stage (bi-encoder retrieval) trades accuracy for speed by encoding query and documents independently. The second stage (cross-encoder reranking) achieves higher accuracy by encoding query and document together with full attention, but is too expensive to run over the entire corpus. The standard pattern is: retrieve top-K candidates (K=20-100) with a fast bi-encoder, then rerank to select the final top-N (N=3-10) with a cross-encoder. This two-stage approach gives you the speed of bi-encoders with the accuracy of cross-encoders.

---

## The Two-Encoder Paradigm

### Bi-Encoders (First Stage)

Bi-encoders encode query and document independently into fixed-size embeddings, then compute similarity (cosine, dot product) between the two vectors.

```
Query  --> [Encoder] --> q_vec (768d)
                                        cosine(q_vec, d_vec) = score
Doc    --> [Encoder] --> d_vec (768d)
```

**Advantages**:
- Documents are encoded offline (once), stored in a vector index
- Query encoding is a single forward pass (~5-10ms)
- Approximate nearest neighbor search scales to billions of documents
- Total retrieval latency: 10-50ms

**Limitations**:
- Query and document are encoded independently -- no token-level interaction
- Cannot capture fine-grained relevance signals (negation, qualification, word order)
- Compression to a single vector loses information

### Cross-Encoders (Second Stage)

Cross-encoders concatenate query and document into a single input and process them jointly through a transformer with full self-attention.

```
[CLS] query tokens [SEP] document tokens [SEP]
                    |
              [Transformer with full cross-attention]
                    |
              relevance score (scalar)
```

**Advantages**:
- Full token-level interaction between query and document
- Can capture negation ("without OAuth" vs "with OAuth")
- Can capture specificity and qualification
- Consistently 5-15% better NDCG than bi-encoders

**Limitations**:
- Must run inference for every (query, document) pair: O(N) forward passes
- Each inference is 10-50ms depending on model size
- Cannot pre-compute document representations
- Too slow for full-corpus search (1M docs x 20ms = 5.5 hours per query)

### Why Two Stages?

| Approach | Latency (1M corpus, top-10) | NDCG@10 |
|----------|---------------------------|---------|
| Pure bi-encoder | ~30ms | 0.44 (BEIR avg) |
| Pure cross-encoder | ~5.5 hours | 0.52 (BEIR avg) |
| Bi-encoder top-100 + cross-encoder rerank | ~2.1 seconds | 0.50 |
| Bi-encoder top-20 + cross-encoder rerank | ~450ms | 0.49 |

The two-stage approach captures 85-95% of cross-encoder quality at <1% of the latency.

---

## The Retrieve-Then-Rerank Pipeline

### Architecture

```
User Query
    |
    v
[Embedding Model] --> query vector
    |
    v
[Vector Search] --> top-K candidates (K=20-100)
    |
    v
[Cross-Encoder Reranker] --> re-scored candidates
    |
    v
[Take top-N] (N=3-10)
    |
    v
[LLM Generation]
```

### Implementation with LangChain

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

# Stage 1: Vector retrieval
vectorstore = Chroma.from_documents(documents, OpenAIEmbeddings())
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})

# Stage 2: Cross-encoder reranking
cross_encoder = HuggingFaceCrossEncoder(model_name="cross-encoder/ms-marco-MiniLM-L-6-v2")
reranker = CrossEncoderReranker(model=cross_encoder, top_n=5)

# Combined retriever
compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=base_retriever,
)

docs = compression_retriever.invoke("How to configure row-level security in PostgreSQL?")
```

### Implementation with LlamaIndex

```python
from llama_index.core import VectorStoreIndex
from llama_index.core.postprocessor import SentenceTransformerRerank

# Build index
index = VectorStoreIndex.from_documents(documents)

# Create reranker
reranker = SentenceTransformerRerank(
    model="cross-encoder/ms-marco-MiniLM-L-6-v2",
    top_n=5,
)

# Query with reranking
query_engine = index.as_query_engine(
    similarity_top_k=20,  # Retrieve 20, rerank to 5
    node_postprocessors=[reranker],
)

response = query_engine.query("How to configure RLS?")
```

---

## Choosing K (Input) and N (Output)

### The K-to-N Ratio

| K (retrieved) | N (after rerank) | Ratio | Use Case |
|--------------|-----------------|-------|----------|
| 20 | 5 | 4:1 | Standard RAG, low latency budget |
| 50 | 5 | 10:1 | High-recall needs, medium latency budget |
| 100 | 10 | 10:1 | Complex queries, large context windows |
| 200 | 20 | 10:1 | Long-form generation, comprehensive answers |

**Guidelines**:
- Minimum K should be 3-5x N to give the reranker enough candidates
- Higher K increases recall but adds reranker latency (linear in K)
- Diminishing returns above K=100 for most use cases
- If K is too small, the reranker cannot fix first-stage misses

### Latency Estimation

```python
def estimate_rerank_latency(
    k: int,
    avg_doc_tokens: int = 200,
    model: str = "ms-marco-MiniLM-L-6-v2",
) -> float:
    """Estimate reranking latency in milliseconds."""
    # Approximate ms per (query, doc) pair by model size
    ms_per_pair = {
        "ms-marco-MiniLM-L-6-v2": 5,      # 22M params
        "ms-marco-MiniLM-L-12-v2": 10,     # 33M params
        "bge-reranker-v2-m3": 15,           # 568M params
        "cohere-rerank-v3.5": 20,           # API call overhead
        "jina-reranker-v2": 12,             # API
    }
    per_pair = ms_per_pair.get(model, 15)
    return k * per_pair  # Sequential; halve if batched on GPU


# Examples
print(estimate_rerank_latency(20))   # ~100ms with MiniLM
print(estimate_rerank_latency(100))  # ~500ms with MiniLM
print(estimate_rerank_latency(20, model="bge-reranker-v2-m3"))  # ~300ms
```

---

## Cross-Encoder Models

### Open-Source Models (Hugging Face)

```python
from sentence_transformers import CrossEncoder

# Small and fast (recommended starting point)
model = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2", max_length=512)

# Pairs to score
pairs = [
    ("How to configure RLS?", "PostgreSQL RLS allows row-level access control..."),
    ("How to configure RLS?", "MySQL user management involves GRANT statements..."),
]

scores = model.predict(pairs)
# [0.98, 0.12] -- higher score = more relevant
```

### Batch Reranking Function

```python
from sentence_transformers import CrossEncoder
import numpy as np


def rerank(
    query: str,
    documents: list[dict],
    model_name: str = "cross-encoder/ms-marco-MiniLM-L-6-v2",
    top_n: int = 5,
    content_key: str = "content",
) -> list[dict]:
    """
    Rerank documents using a cross-encoder.

    Args:
        query: The search query
        documents: List of dicts with at least a content_key field
        model_name: Cross-encoder model to use
        top_n: Number of documents to return
        content_key: Key in document dict containing the text

    Returns:
        Top-N documents sorted by relevance, with 'rerank_score' added
    """
    model = CrossEncoder(model_name, max_length=512)

    pairs = [(query, doc[content_key]) for doc in documents]
    scores = model.predict(pairs, show_progress_bar=False)

    # Attach scores and sort
    for doc, score in zip(documents, scores):
        doc["rerank_score"] = float(score)

    ranked = sorted(documents, key=lambda d: d["rerank_score"], reverse=True)
    return ranked[:top_n]


# Usage
candidates = [
    {"id": "1", "content": "PostgreSQL RLS policies control row access..."},
    {"id": "2", "content": "MySQL privilege system uses GRANT/REVOKE..."},
    {"id": "3", "content": "Row Level Security in PostgreSQL enables..."},
]

results = rerank("How to set up RLS in PostgreSQL?", candidates, top_n=2)
for r in results:
    print(f"  {r['id']}: score={r['rerank_score']:.4f}")
```

---

## API-Based Rerankers

### Cohere Rerank

```python
import cohere

co = cohere.ClientV2(api_key="your-key")

response = co.rerank(
    model="rerank-v3.5",
    query="How to configure row-level security?",
    documents=[
        "PostgreSQL RLS policies control row access per user...",
        "MySQL user management involves GRANT statements...",
        "Row Level Security enables fine-grained access...",
    ],
    top_n=2,
    return_documents=True,
)

for result in response.results:
    print(f"Index: {result.index}, Score: {result.relevance_score:.4f}")
    print(f"Text: {result.document.text[:100]}...")
```

### Voyage Rerank

```python
import voyageai

vo = voyageai.Client(api_key="your-key")

reranking = vo.rerank(
    query="How to configure row-level security?",
    documents=[
        "PostgreSQL RLS policies control row access per user...",
        "MySQL user management involves GRANT statements...",
        "Row Level Security enables fine-grained access...",
    ],
    model="rerank-2",
    top_k=2,
)

for result in reranking.results:
    print(f"Index: {result.index}, Score: {result.relevance_score:.4f}")
```

### Jina Reranker

```python
import requests

response = requests.post(
    "https://api.jina.ai/v1/rerank",
    headers={
        "Authorization": "Bearer your-key",
        "Content-Type": "application/json",
    },
    json={
        "model": "jina-reranker-v2-base-multilingual",
        "query": "How to configure row-level security?",
        "documents": [
            "PostgreSQL RLS policies control row access per user...",
            "MySQL user management involves GRANT statements...",
        ],
        "top_n": 2,
    },
)

for result in response.json()["results"]:
    print(f"Index: {result['index']}, Score: {result['relevance_score']:.4f}")
```

---

## When to Add Reranking

### Reranking clearly helps:

1. **Domain-specific content** where bi-encoder embeddings are not fine-tuned for your domain
2. **Queries requiring nuance** -- negation, qualification, comparison
3. **Legal/compliance** where precision is critical (wrong document = costly error)
4. **Multi-hop questions** where relevance depends on the relationship between query and document
5. **When first-stage recall is high but precision is low** -- many candidates, few truly relevant

### Reranking may not help:

1. **Simple factual queries** where the top-1 bi-encoder result is already correct
2. **Very small corpora** (<500 documents) where brute-force cross-encoder is feasible
3. **Latency-critical applications** where 100-500ms reranking overhead is unacceptable
4. **When bi-encoder is already fine-tuned** on your domain data

### Decision Framework

```python
def should_add_reranking(
    current_precision_at_5: float,
    current_recall_at_20: float,
    latency_budget_ms: int,
    corpus_is_domain_specific: bool,
) -> bool:
    """Heuristic: should you add a reranking stage?"""
    # If precision is already high, reranking won't help much
    if current_precision_at_5 > 0.85:
        return False

    # If recall at K is low, reranking can't fix missing documents
    if current_recall_at_20 < 0.50:
        return False  # Fix first-stage retrieval first

    # Need latency headroom
    if latency_budget_ms < 200:
        return False  # Use a smaller model or skip reranking

    # High-value use cases
    if corpus_is_domain_specific and current_precision_at_5 < 0.70:
        return True

    return current_precision_at_5 < 0.75  # General threshold
```

---

## Combining Reranking with Hybrid Search

The optimal RAG pipeline is: hybrid retrieval (BM25 + vector) followed by reranking.

```python
from langchain.retrievers import EnsembleRetriever, ContextualCompressionRetriever
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder
from langchain_community.retrievers import BM25Retriever

# Stage 1a: Vector retriever
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 30})

# Stage 1b: BM25 retriever
bm25_retriever = BM25Retriever.from_documents(documents, k=30)

# Stage 1: Hybrid fusion
hybrid_retriever = EnsembleRetriever(
    retrievers=[vector_retriever, bm25_retriever],
    weights=[0.5, 0.5],
)

# Stage 2: Cross-encoder reranking
cross_encoder = HuggingFaceCrossEncoder(model_name="BAAI/bge-reranker-v2-m3")
reranker = CrossEncoderReranker(model=cross_encoder, top_n=5)

# Combined pipeline
final_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=hybrid_retriever,
)

docs = final_retriever.invoke("How to configure PostgreSQL row-level security?")
```

---

## Performance Benchmarks

### Model Quality (BEIR Average NDCG@10)

| Model | Params | BEIR Avg | Latency (20 docs) | Cost |
|-------|--------|----------|--------------------|------|
| No reranker (bi-encoder only) | -- | 0.443 | 0ms | $0 |
| ms-marco-MiniLM-L-6-v2 | 22M | 0.481 | ~100ms | Free |
| ms-marco-MiniLM-L-12-v2 | 33M | 0.489 | ~200ms | Free |
| bge-reranker-v2-m3 | 568M | 0.523 | ~300ms | Free |
| Cohere rerank-v3.5 | Unknown | 0.531 | ~400ms | $2/1K queries |
| Voyage rerank-2 | Unknown | 0.528 | ~350ms | $0.05/1K queries |
| Jina reranker-v2 | 278M | 0.515 | ~250ms | $0.02/1K queries |

Note: numbers are approximate and vary by hardware and query length.

### Latency Breakdown for Typical RAG Pipeline

```
Query embedding:        15ms
Vector search (top-50): 20ms
BM25 search (top-50):   10ms
RRF fusion:              1ms
Cross-encoder rerank:  250ms (50 docs x 5ms with MiniLM)
                       -----
Total:                 296ms
```

---

## Common Pitfalls

1. **Reranking too few candidates.** If K is too small (e.g., reranking top-5), the reranker has no room to improve ordering. Use K >= 3x your desired output N.
2. **Using a reranker without evaluating its impact.** Always A/B test reranking vs. no reranking on your eval set. Some domains see minimal improvement.
3. **Ignoring document length.** Cross-encoders have max token limits (typically 512). Documents longer than this are truncated, potentially losing relevant content. Pre-truncate or use passage-level reranking.
4. **Not batching reranker inference.** Running one pair at a time wastes GPU. Batch all (query, doc) pairs in a single predict call.
5. **Reranking when first-stage recall is poor.** Reranking re-orders candidates but cannot retrieve new ones. If relevant documents are not in the top-K candidates, reranking cannot help. Fix first-stage retrieval first.
6. **Using a cross-encoder as the only retrieval stage.** Cross-encoders are O(N) per query where N is the corpus size. They are rerankers, not retrievers.
7. **Choosing the most expensive reranker by default.** ms-marco-MiniLM captures 80% of the quality improvement at 10% of the cost of API-based rerankers. Start small and upgrade only if eval metrics justify it.

---

## Production Checklist

- [ ] First-stage retriever fetches 3-5x more candidates than the reranker outputs
- [ ] Reranker model is chosen based on eval metrics, not marketing claims
- [ ] Document truncation strategy is defined for documents exceeding max token length
- [ ] Reranker inference is batched (all pairs in one forward pass)
- [ ] End-to-end latency (retrieval + reranking) is within the latency budget
- [ ] A/B test confirms reranking improves answer quality on your eval set
- [ ] Reranker output includes scores for potential confidence thresholding
- [ ] Monitoring tracks reranker latency, score distribution, and rank changes
- [ ] Fallback to unreranked results if reranker fails (timeout, error)
- [ ] GPU memory is allocated appropriately for the chosen model size

---

## References

- Nogueira & Cho. "Passage Re-ranking with BERT" (2019)
- Thakur et al. "BEIR: A Heterogeneous Benchmark for Zero-shot Evaluation of Information Retrieval Models" (NeurIPS 2021)
- Cohere Rerank: https://docs.cohere.com/docs/rerank-2
- Sentence Transformers cross-encoders: https://www.sbert.net/docs/cross_encoder/usage/usage.html
- BAAI bge-reranker: https://huggingface.co/BAAI/bge-reranker-v2-m3
- LangChain reranking: https://python.langchain.com/docs/how_to/contextual_compression/
