# RAGatouille -- ColBERT Late-Interaction Retrieval

## Overview

RAGatouille is a Python library that makes ColBERT -- the late-interaction neural retrieval model -- accessible for RAG applications. ColBERT (Contextualized Late Interaction over BERT) represents both queries and documents as sets of token-level embeddings, then scores relevance by computing fine-grained interactions between these token embeddings at search time. This produces significantly better retrieval quality than single-vector embedding models (like `text-embedding-3-small` or `bge-small`) while remaining efficient through the PLAID indexing engine.

RAGatouille wraps the `colbert-ai` library with a simple API: three lines to index, one line to search. It handles model loading, PLAID index construction, and provides direct integrations with LangChain and LlamaIndex.

This guide covers how ColBERT works, the RAGatouille API, PLAID indexing, and when to choose late-interaction retrieval over single-vector approaches.

---

## How ColBERT Works

### Single-Vector vs. Late-Interaction

**Single-vector models** (OpenAI embeddings, BGE, E5) compress an entire document into one vector. Query-document similarity is a single dot product:

```
query  -> [0.1, 0.3, ...]  (1 vector)
doc    -> [0.2, 0.4, ...]  (1 vector)
score  = dot(query_vec, doc_vec)
```

**ColBERT late-interaction** keeps one vector per token. Scoring computes MaxSim -- the maximum similarity between each query token and all document tokens:

```
query  -> [[0.1, 0.3, ...], [0.2, 0.1, ...], ...]  (N vectors, one per query token)
doc    -> [[0.2, 0.4, ...], [0.3, 0.2, ...], ...]  (M vectors, one per doc token)

For each query token q_i:
  max_sim_i = max(dot(q_i, d_j) for all d_j in doc tokens)

score = sum(max_sim_i for all query tokens)
```

This fine-grained matching captures exact term overlap, partial matches, and semantic similarity simultaneously. ColBERT typically achieves 5-15% higher recall@10 than single-vector models on retrieval benchmarks.

### PLAID Indexing

Storing per-token embeddings for millions of documents would be prohibitively expensive. PLAID (Performance-optimized Late Interaction Driver) solves this with three techniques:

1. **Centroid-based compression**: Token embeddings are quantized to cluster centroids using k-means. The index stores centroid IDs (2 bytes) instead of full vectors (128-256 bytes).

2. **Residual compression**: The difference between the original embedding and its centroid is stored as a compressed residual (typically 16-32 bytes per token).

3. **Candidate generation**: At search time, PLAID first uses centroids for fast candidate generation (similar to IVF in FAISS), then decompresses residuals only for candidates to compute exact scores.

The result: ColBERT indexes are 5-10x larger than single-vector indexes but 100x smaller than storing full per-token embeddings, while maintaining near-exact retrieval quality.

### Available Models

| Model | Parameters | Dimensions | Max Length | Notes |
|-------|-----------|------------|------------|-------|
| `colbert-ir/colbertv2.0` | 110M | 128 | 512 | Original ColBERTv2, English |
| `answerdotai/answerai-colbert-small-v1` | 33M | 96 | 512 | Lightweight, good quality |
| Custom finetuned | Varies | Configurable | Configurable | Domain-specific training |

---

## RAGatouille API

### Installation

```bash
pip install ragatouille

# For GPU support (recommended for indexing and training)
pip install ragatouille torch --extra-index-url https://download.pytorch.org/whl/cu121
```

### Basic Indexing and Search

```python
from ragatouille import RAGPretrainedModel

# Load a pretrained ColBERT model
rag = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")

# Index documents
documents = [
    "ColBERT uses late interaction for fine-grained relevance matching.",
    "PLAID indexing compresses per-token embeddings using centroid quantization.",
    "RAGatouille wraps ColBERT with a simple Python API.",
    "Late-interaction models outperform single-vector models on retrieval benchmarks.",
    "ColBERTv2 was developed at Stanford and uses 128-dimensional token embeddings.",
    "The MaxSim operation computes the maximum similarity between query and document tokens.",
    "FAISS and HNSW are common indexing strategies for single-vector retrieval.",
    "Neural retrieval models can be fine-tuned on domain-specific data for better performance.",
]

# Create an index (this builds the PLAID index)
index_path = rag.index(
    index_name="my_index",
    collection=documents,
    document_ids=None,          # auto-generated if None
    document_metadatas=None,    # optional metadata dicts
    split_documents=True,       # split long documents into passages
    max_document_length=256,    # max tokens per passage
    use_faiss=True,             # use FAISS for centroid search
)

print(f"Index created at: {index_path}")
```

### Searching

```python
# Search the index
results = rag.search(
    query="How does ColBERT score documents?",
    k=5,
)

for result in results:
    print(f"Score: {result['score']:.4f}")
    print(f"Content: {result['content']}")
    print(f"Document ID: {result['document_id']}")
    print("---")
```

### Loading an Existing Index

```python
from ragatouille import RAGPretrainedModel

# Load from a previously created index
rag = RAGPretrainedModel.from_index(".ragatouille/colbert/indexes/my_index")

# Search immediately
results = rag.search("What is PLAID?", k=3)
```

### Adding Documents to an Existing Index

```python
# Add new documents to an existing index
new_documents = [
    "ColBERT can be used as a reranker by scoring query-document pairs directly.",
    "The residual compression in PLAID reduces storage by 4-8x compared to full embeddings.",
]

rag.add_to_index(
    new_collection=new_documents,
    new_document_ids=None,
    new_document_metadatas=None,
)
```

### Batch Search

```python
queries = [
    "How does ColBERT work?",
    "What is PLAID indexing?",
    "How do late-interaction models compare to single-vector?",
]

# Search all queries at once (more efficient than individual searches)
batch_results = rag.search(
    query=queries,
    k=5,
)

# batch_results is a list of lists
for query, results in zip(queries, batch_results):
    print(f"Query: {query}")
    for r in results[:2]:
        print(f"  [{r['score']:.3f}] {r['content'][:80]}...")
    print()
```

---

## Using ColBERT as a Reranker

ColBERT can rerank documents retrieved by a faster first-stage retriever (e.g., BM25 or a single-vector model):

```python
from ragatouille import RAGPretrainedModel

# Load the model (no index needed for reranking)
reranker = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")

# Documents from a first-stage retriever (e.g., BM25)
candidate_documents = [
    "ColBERT uses per-token embeddings for fine-grained matching.",
    "Python is a popular programming language.",
    "Late interaction computes MaxSim between query and document tokens.",
    "Machine learning models require training data.",
    "The PLAID engine makes ColBERT efficient at scale.",
]

# Rerank by ColBERT score
reranked = reranker.rerank(
    query="How does ColBERT compute relevance scores?",
    documents=candidate_documents,
    k=3,
)

for doc in reranked:
    print(f"Score: {doc['score']:.4f} | {doc['content'][:80]}")
```

### Two-Stage Retrieval Pipeline

```python
from ragatouille import RAGPretrainedModel


def two_stage_retrieval(
    query: str,
    first_stage_results: list[str],
    top_k_first: int = 20,
    top_k_final: int = 5,
) -> list[dict]:
    """
    Stage 1: Fast retriever (BM25/embedding) retrieves top_k_first candidates.
    Stage 2: ColBERT reranks candidates to select top_k_final.
    """
    reranker = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")

    # Stage 2: Rerank with ColBERT
    reranked = reranker.rerank(
        query=query,
        documents=first_stage_results[:top_k_first],
        k=top_k_final,
    )

    return reranked


# Example usage with BM25 first stage
import rank_bm25

corpus = [
    "ColBERT computes MaxSim between query and document token embeddings.",
    "BM25 is a bag-of-words retrieval algorithm.",
    "PLAID indexing compresses token embeddings for efficient storage.",
    "Neural models can capture semantic similarity beyond exact term matching.",
    "The late interaction paradigm separates encoding from scoring.",
]

tokenized_corpus = [doc.split() for doc in corpus]
bm25 = rank_bm25.BM25Okapi(tokenized_corpus)

query = "How does ColBERT score relevance?"
bm25_scores = bm25.get_scores(query.split())

# Sort by BM25 score
bm25_ranked = sorted(
    zip(corpus, bm25_scores), key=lambda x: x[1], reverse=True
)
first_stage = [doc for doc, _ in bm25_ranked[:20]]

# Rerank with ColBERT
final_results = two_stage_retrieval(query, first_stage)
```

---

## Integration with LangChain

```python
from ragatouille import RAGPretrainedModel

# Create the RAGatouille model and index
rag = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")
rag.index(
    index_name="langchain_docs",
    collection=documents,
)

# Convert to a LangChain retriever
langchain_retriever = rag.as_langchain_retriever(k=5)

# Use in a LangChain chain
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_openai import ChatOpenAI

template = """Answer the question based on the context:

Context: {context}

Question: {question}

Answer:"""

prompt = ChatPromptTemplate.from_template(template)
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

chain = (
    {"context": langchain_retriever, "question": RunnablePassthrough()}
    | prompt
    | llm
)

response = chain.invoke("How does ColBERT index documents?")
print(response.content)
```

---

## Integration with LlamaIndex

```python
from ragatouille import RAGPretrainedModel
from llama_index.core import VectorStoreIndex, Settings
from llama_index.core.schema import TextNode
from llama_index.llms.openai import OpenAI

# Create RAGatouille index
rag = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")
rag.index(index_name="llamaindex_docs", collection=documents)

# Use RAGatouille search results in LlamaIndex
Settings.llm = OpenAI(model="gpt-4o-mini")


def ragatouille_query(question: str, k: int = 5) -> str:
    """Query using RAGatouille retrieval + LlamaIndex synthesis."""
    # Retrieve with ColBERT
    results = rag.search(question, k=k)

    # Convert to LlamaIndex nodes
    nodes = [
        TextNode(
            text=r["content"],
            metadata={"colbert_score": r["score"], "doc_id": r["document_id"]},
        )
        for r in results
    ]

    # Build a temporary index for synthesis
    index = VectorStoreIndex(nodes=nodes)
    query_engine = index.as_query_engine(similarity_top_k=k)

    response = query_engine.query(question)
    return str(response)
```

---

## Index Management

### Index Configuration

```python
from ragatouille import RAGPretrainedModel

rag = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")

# Configure PLAID indexing parameters
index_path = rag.index(
    index_name="production_index",
    collection=documents,
    max_document_length=256,    # tokens per passage
    split_documents=True,       # auto-split long documents
    use_faiss=True,             # FAISS for centroid search
)
```

### Index Storage and Portability

```python
import shutil

# Default index location
# .ragatouille/colbert/indexes/{index_name}/

# Copy index for deployment
shutil.copytree(
    ".ragatouille/colbert/indexes/production_index",
    "/opt/app/indexes/production_index",
)

# Load from custom location
rag = RAGPretrainedModel.from_index("/opt/app/indexes/production_index")
```

### Index Size Estimation

The index size depends on the number of documents and tokens:

```python
def estimate_index_size(
    num_documents: int,
    avg_tokens_per_doc: int = 128,
    bytes_per_token: int = 34,     # centroid ID (2B) + residual (32B)
) -> str:
    """Estimate PLAID index size."""
    total_tokens = num_documents * avg_tokens_per_doc
    total_bytes = total_tokens * bytes_per_token
    total_gb = total_bytes / (1024 ** 3)

    return f"{num_documents:,} docs, ~{total_tokens:,} tokens = ~{total_gb:.2f} GB"


print(estimate_index_size(100_000))     # ~100K docs = ~0.40 GB
print(estimate_index_size(1_000_000))   # ~1M docs = ~4.02 GB
print(estimate_index_size(10_000_000))  # ~10M docs = ~40.2 GB
```

---

## Performance Characteristics

### Retrieval Quality Comparison

Based on MS MARCO and BEIR benchmarks:

| Model | Type | MRR@10 (MS MARCO) | Avg NDCG@10 (BEIR) |
|-------|------|-------------------|---------------------|
| BM25 | Keyword | 0.187 | 0.438 |
| `text-embedding-3-small` | Single-vector | 0.325 | 0.512 |
| `bge-large-en-v1.5` | Single-vector | 0.348 | 0.543 |
| ColBERTv2 | Late-interaction | 0.397 | 0.561 |

### Latency and Throughput

| Operation | Typical Latency | Notes |
|-----------|----------------|-------|
| Indexing (GPU) | ~1000 docs/sec | Depends on doc length |
| Indexing (CPU) | ~50 docs/sec | Not recommended for large collections |
| Search (PLAID) | 20-50ms per query | For 100K-1M doc collections |
| Reranking | 5-20ms per query | Depends on candidate count |
| Batch search | 5-10ms per query | Amortized over batch |

### When to Use ColBERT/RAGatouille

**Choose ColBERT when:**
- Retrieval quality is paramount (legal, medical, technical domains)
- You can tolerate 5-10x larger indexes than single-vector
- You have GPU resources for indexing (CPU is slow)
- Your queries require fine-grained term matching

**Choose single-vector when:**
- Index size is constrained
- You need sub-10ms latency at scale
- Your documents are short (< 100 tokens)
- Retrieval quality from BM25/embedding is sufficient

**Best of both worlds:**
- Use single-vector or BM25 for fast first-stage retrieval
- Rerank top candidates with ColBERT for quality

---

## Common Pitfalls

1. **Indexing on CPU**: ColBERT indexing on CPU is 20x slower than GPU. For collections over 10K documents, use a GPU machine for indexing, then copy the index to the serving machine.

2. **Not splitting long documents**: If `split_documents=False` and documents exceed `max_document_length`, they are silently truncated. Always enable splitting for documents longer than 256 tokens.

3. **Index path confusion**: RAGatouille stores indexes in `.ragatouille/colbert/indexes/` by default. Use `from_index()` with the full path when loading, not just the index name.

4. **Memory during indexing**: PLAID index construction loads all token embeddings into memory for k-means clustering. For very large collections (>5M docs), consider indexing in shards and merging.

5. **Stale index after adding documents**: When you call `add_to_index()`, the PLAID index structure may not be fully optimized for the new documents. For best quality, periodically rebuild the full index.

---

## References

- RAGatouille GitHub: https://github.com/AnswerDotAI/RAGatouille
- ColBERT paper: "ColBERT: Efficient and Effective Passage Search via Contextualized Late Interaction over BERT"
- ColBERTv2 paper: "ColBERTv2: Effective and Efficient Retrieval via Lightweight Late Interaction"
- PLAID paper: "PLAID: An Efficient Engine for Late Interaction Retrieval"
