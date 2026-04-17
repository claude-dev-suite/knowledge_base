# ColBERT as Reranker

## TL;DR

ColBERT (Contextualized Late Interaction over BERT) uses a "late interaction" mechanism: query and document are encoded independently into per-token embeddings, then relevance is computed via MaxSim -- the maximum similarity between each query token and all document tokens, summed across query tokens. This gives ColBERT accuracy close to cross-encoders (full attention) while retaining most of the efficiency of bi-encoders (pre-computable document representations). ColBERT can serve as both a retriever and a reranker. As a reranker, it sits between bi-encoder retrieval and cross-encoder reranking in the quality/speed tradeoff. This article covers the MaxSim mechanism, RAGatouille's rerank API, when ColBERT reranking beats cross-encoders, and multi-stage pipelines combining ColBERT with cross-encoders.

---

## The Late Interaction Mechanism

### How ColBERT Differs from Bi-Encoders and Cross-Encoders

```
Bi-Encoder:     query -> [Enc] -> 1 vector     }
                                                } -> dot(q_vec, d_vec) = score
                doc   -> [Enc] -> 1 vector      }

Cross-Encoder:  [CLS] query [SEP] doc [SEP] -> [Enc] -> score
                (full attention between all tokens)

ColBERT:        query -> [Enc] -> m token vectors  }
                                                    } -> MaxSim(Q, D) = score
                doc   -> [Enc] -> n token vectors   }
```

### MaxSim Scoring

Given query token embeddings `Q = {q_1, q_2, ..., q_m}` and document token embeddings `D = {d_1, d_2, ..., d_n}`:

```
MaxSim(Q, D) = SUM_{i=1}^{m} MAX_{j=1}^{n} cos(q_i, d_j)
```

For each query token, find the document token most similar to it, take that similarity, and sum across all query tokens.

**Intuition**: each query token "matches" to its best-aligned document token. A document is relevant if every important query token finds a good match somewhere in the document.

### Mathematical Example

```
Query: "configure RLS PostgreSQL"
  q1 = embed("configure")  = [0.3, 0.8, 0.1, ...]
  q2 = embed("RLS")        = [0.1, 0.2, 0.9, ...]
  q3 = embed("PostgreSQL")  = [0.7, 0.1, 0.3, ...]

Document: "Row Level Security in PostgreSQL databases allows..."
  d1 = embed("Row")       d2 = embed("Level")    d3 = embed("Security")
  d4 = embed("in")        d5 = embed("PostgreSQL") d6 = embed("databases")
  d7 = embed("allows")    ...

MaxSim computation:
  For q1 ("configure"): max similarity across all d_j -> best match d7 ("allows") = 0.62
  For q2 ("RLS"):        max similarity across all d_j -> best match d3 ("Security") = 0.87
  For q3 ("PostgreSQL"): max similarity across all d_j -> best match d5 ("PostgreSQL") = 0.99

  Score = 0.62 + 0.87 + 0.99 = 2.48
```

### Why Late Interaction Works

1. **Token-level matching** captures fine-grained relevance (like cross-encoders)
2. **Independent encoding** allows pre-computing document token embeddings (like bi-encoders)
3. **MaxSim** is differentiable and cheap to compute at inference time
4. **No full attention between query and document** -- O(m * n) similarity comparisons vs O((m+n)^2) attention in cross-encoders

---

## ColBERT v2 and PLAID

### ColBERTv2 Improvements

ColBERTv2 (2022) introduced:

1. **Residual compression**: document token embeddings are quantized using centroids + residuals, reducing storage from 128 floats to ~2 bytes per token embedding
2. **Denoised supervision**: distilled training from a cross-encoder teacher
3. **Cross-encoder distillation**: trains ColBERT to match cross-encoder relevance scores

### PLAID Engine

PLAID (Performance-optimized Late Interaction Driver) is ColBERTv2's retrieval engine:

1. **Candidate generation**: use centroid-based filtering to identify candidate documents
2. **Decompression**: decompress only candidate document embeddings
3. **MaxSim scoring**: compute full MaxSim only on candidates

This makes ColBERT retrieval viable at millions of documents with sub-second latency.

---

## RAGatouille: ColBERT Made Easy

RAGatouille is the standard Python library for using ColBERT in RAG pipelines.

### Installation

```bash
pip install ragatouille
```

### Indexing and Retrieval

```python
from ragatouille import RAGPretrainedModel

# Load ColBERTv2
rag = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")

# Index documents
documents = [
    "PostgreSQL Row Level Security allows per-row access control policies.",
    "MySQL user management uses GRANT and REVOKE privilege statements.",
    "Row Level Security in PostgreSQL enables fine-grained authorization.",
    "Database connection pooling improves performance under high load.",
    "PostgreSQL supports CREATE POLICY for RLS configuration.",
]

# Build index (creates PLAID index on disk)
rag.index(
    collection=documents,
    index_name="my_docs",
    max_document_length=256,
    split_documents=True,  # Auto-chunk long docs into passages
)

# Retrieve
results = rag.search("How to configure RLS in PostgreSQL?", k=3)
for r in results:
    print(f"Score: {r['score']:.4f} | {r['content'][:80]}...")
```

### ColBERT as Reranker with RAGatouille

RAGatouille supports using ColBERT purely as a reranker (without building a full index):

```python
from ragatouille import RAGPretrainedModel

rag = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")

# Rerank mode: score pre-retrieved candidates
query = "How to configure row-level security in PostgreSQL?"
candidates = [
    "PostgreSQL RLS allows row-level access control policies...",
    "MySQL user management uses GRANT and REVOKE statements...",
    "CREATE POLICY enables RLS configuration in PostgreSQL...",
    "Database connection pooling configuration guide...",
    "PostgreSQL supports row-level security through policies...",
]

# Rerank returns candidates sorted by ColBERT score
reranked = rag.rerank(query=query, documents=candidates, k=3)

for r in reranked:
    print(f"Score: {r['score']:.4f} | {r['content'][:80]}...")
```

### Integrating ColBERT Reranking into a Pipeline

```python
from ragatouille import RAGPretrainedModel
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

# First stage: standard bi-encoder retrieval
vectorstore = Chroma.from_documents(documents, OpenAIEmbeddings())
retriever = vectorstore.as_retriever(search_kwargs={"k": 30})

# ColBERT reranker
colbert = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")


async def retrieve_and_rerank(query: str, top_n: int = 5) -> list[dict]:
    # Stage 1: bi-encoder retrieval
    candidates = retriever.invoke(query)

    # Stage 2: ColBERT reranking
    candidate_texts = [doc.page_content for doc in candidates]
    reranked = colbert.rerank(
        query=query,
        documents=candidate_texts,
        k=top_n,
    )

    return [
        {
            "content": r["content"],
            "score": r["score"],
            "metadata": candidates[r.get("result_index", 0)].metadata
            if r.get("result_index") is not None else {},
        }
        for r in reranked
    ]
```

---

## ColBERT Reranking vs. Cross-Encoder Reranking

### Quality Comparison

| Method | BEIR Avg NDCG@10 | MS MARCO MRR@10 | Latency (20 docs) |
|--------|------------------|-----------------|-------------------|
| Bi-encoder only | 0.443 | 0.334 | 0ms |
| ColBERT reranking | 0.504 | 0.397 | ~50ms |
| MiniLM-L-6 cross-encoder | 0.481 | 0.390 | ~100ms |
| MiniLM-L-12 cross-encoder | 0.489 | 0.397 | ~200ms |
| bge-reranker-v2-m3 | 0.523 | 0.428 | ~300ms |
| Cohere rerank-v3.5 | 0.531 | 0.437 | ~400ms |

### When ColBERT Reranking Beats Cross-Encoders

1. **Latency-constrained applications**: ColBERT reranking is 2-6x faster than cross-encoders because document token embeddings can be pre-computed and cached.

2. **Large candidate sets (K > 50)**: ColBERT's cost for reranking K documents is dominated by the MaxSim computation (fast matrix operations), not transformer forward passes.

```
Cross-encoder latency:  K * forward_pass_time  (e.g., 100 * 10ms = 1000ms)
ColBERT rerank latency: query_forward_pass + K * maxsim_time  (e.g., 10ms + 100 * 0.5ms = 60ms)
```

3. **When document representations can be cached**: in a static corpus, pre-compute all document token embeddings. Only the query needs a forward pass at query time.

4. **When used as a mid-stage reranker**: ColBERT can serve as a fast filter between bi-encoder retrieval and a final cross-encoder, reducing the number of expensive cross-encoder passes.

### When Cross-Encoders Beat ColBERT

1. **Small candidate sets (K < 20)**: the fixed overhead of ColBERT's query encoding is amortized over fewer documents, reducing the latency advantage.

2. **Maximum accuracy required**: cross-encoders with full bidirectional attention still have a 2-5% NDCG edge over ColBERT, especially on nuanced queries requiring deep reasoning about the query-document relationship.

3. **When document pre-computation is not feasible**: rapidly changing corpora where documents cannot be pre-encoded negate ColBERT's main speed advantage.

---

## Multi-Stage Pipeline: ColBERT + Cross-Encoder

The optimal pipeline for maximum quality with acceptable latency:

```
User Query
    |
    v
[Bi-Encoder] -> top-200 candidates (fastest, coarsest)
    |
    v
[ColBERT Reranker] -> top-20 (fast token-level matching)
    |
    v
[Cross-Encoder Reranker] -> top-5 (slowest, most accurate)
    |
    v
[LLM Generation]
```

### Implementation

```python
from ragatouille import RAGPretrainedModel
from sentence_transformers import CrossEncoder


class ThreeStageRetriever:
    def __init__(
        self,
        vector_store,
        colbert_model: str = "colbert-ir/colbertv2.0",
        cross_encoder_model: str = "BAAI/bge-reranker-v2-m3",
        stage1_k: int = 200,
        stage2_k: int = 20,
        stage3_k: int = 5,
    ):
        self.vector_store = vector_store
        self.colbert = RAGPretrainedModel.from_pretrained(colbert_model)
        self.cross_encoder = CrossEncoder(cross_encoder_model)
        self.stage1_k = stage1_k
        self.stage2_k = stage2_k
        self.stage3_k = stage3_k

    def retrieve(self, query: str) -> list[dict]:
        # Stage 1: Bi-encoder retrieval (top-200)
        candidates = self.vector_store.similarity_search(
            query, k=self.stage1_k
        )
        if not candidates:
            return []

        # Stage 2: ColBERT reranking (200 -> 20)
        candidate_texts = [doc.page_content for doc in candidates]
        colbert_results = self.colbert.rerank(
            query=query,
            documents=candidate_texts,
            k=self.stage2_k,
        )

        # Stage 3: Cross-encoder reranking (20 -> 5)
        stage2_texts = [r["content"] for r in colbert_results]
        pairs = [(query, text) for text in stage2_texts]
        ce_scores = self.cross_encoder.predict(pairs, batch_size=32)

        # Combine and sort
        final = []
        for result, score in zip(colbert_results, ce_scores):
            final.append({
                "content": result["content"],
                "colbert_score": result["score"],
                "cross_encoder_score": float(score),
            })
        final.sort(key=lambda x: x["cross_encoder_score"], reverse=True)
        return final[: self.stage3_k]


# Usage
retriever = ThreeStageRetriever(vectorstore)
results = retriever.retrieve("How to configure row-level security in PostgreSQL?")
```

### Latency Breakdown

```
Stage 1 (bi-encoder, top-200):       ~40ms
Stage 2 (ColBERT rerank, 200->20):   ~60ms
Stage 3 (cross-encoder, 20->5):      ~250ms
                                      ------
Total:                                ~350ms

Compare to without ColBERT stage:
Stage 1 (bi-encoder, top-200):       ~40ms
Stage 2 (cross-encoder, 200->5):     ~2500ms  (200 * 12.5ms per pair)
                                      ------
Total:                                ~2540ms
```

The ColBERT mid-stage reduces end-to-end latency by 7x while retaining nearly identical quality, because the cross-encoder only processes 20 documents instead of 200.

---

## ColBERT Fine-Tuning for Domain-Specific Reranking

### When to Fine-Tune

- Your domain vocabulary differs significantly from the pre-training data (medical, legal, scientific)
- You have >1000 query-relevant-document triples
- ColBERT zero-shot performance is below your quality threshold

### Fine-Tuning with RAGatouille

```python
from ragatouille import RAGTrainer

trainer = RAGTrainer(
    model_name="my-domain-colbert",
    pretrained_model_name="colbert-ir/colbertv2.0",
    language_code="en",
)

# Training data: list of (query, positive_passage, negative_passage) triples
training_triples = [
    (
        "configure RLS in PostgreSQL",
        "CREATE POLICY enables row-level security in PostgreSQL...",
        "MySQL GRANT statement manages user-level privileges...",
    ),
    # ... more triples
]

# Prepare training data
trainer.prepare_training_data(
    raw_data=training_triples,
    data_out_path="./colbert_training_data/",
    num_new_negatives=10,  # Mine additional hard negatives
    mine_hard_negatives=True,
)

# Train
trainer.train(
    batch_size=32,
    nbits=2,  # Compression level (2-bit quantization)
    maxsteps=500000,
    use_ib_negatives=True,  # In-batch negatives for efficiency
    learning_rate=5e-6,
    output_path="./trained_colbert/",
)
```

---

## Storage and Memory Considerations

### Per-Document Storage

ColBERT stores per-token embeddings for each document. For a 128-dimensional ColBERT with 2-bit quantization:

```
Storage per document:
  avg_tokens = 150 (for a ~200 word chunk)
  bytes_per_token = 128 dims * 2 bits / 8 = 32 bytes (with ColBERTv2 compression)
  total_per_doc = 150 * 32 = 4,800 bytes ≈ 4.7 KB

Compare to bi-encoder:
  1 embedding * 768 dims * 4 bytes = 3,072 bytes ≈ 3 KB

ColBERT uses ~1.5x the storage of a bi-encoder.
```

For 1M documents:
- Bi-encoder: ~3 GB
- ColBERT (compressed): ~4.7 GB
- ColBERT (uncompressed, 128d float32): ~75 GB

### Memory for Reranking

When using ColBERT purely as a reranker (not as a retriever), document token embeddings are computed on-the-fly and do not need persistent storage. Memory usage is dominated by the model weights:

| Model | Weights (FP16) | Inference Memory (batch=32) |
|-------|---------------|---------------------------|
| ColBERTv2 (110M) | ~220MB | ~1.5GB |
| ColBERTv2 on CPU | ~440MB (FP32) | ~2GB |

---

## Common Pitfalls

1. **Confusing ColBERT retrieval with ColBERT reranking.** Retrieval requires a pre-built PLAID index. Reranking works on any candidate list without an index. Different use cases, different setup.
2. **Not pre-computing document embeddings for static corpora.** If your documents do not change frequently, pre-compute and cache token embeddings. This turns reranking into a matrix multiplication, not a transformer forward pass per document.
3. **Using ColBERT with very short queries (1-2 tokens).** MaxSim with 1-2 query tokens provides minimal token-level matching signal. ColBERT's advantage over bi-encoders diminishes for very short queries.
4. **Ignoring compression.** Uncompressed ColBERT embeddings for 1M documents require ~75GB. ColBERTv2 compression reduces this to ~5GB with minimal quality loss.
5. **Assuming ColBERT always beats cross-encoders.** On nuanced relevance judgments (negation, qualification), cross-encoders with full attention still win. ColBERT's MaxSim is a sum of independent per-token matches -- it does not capture cross-token dependencies.
6. **Using RAGatouille rerank on very large candidate sets (>1000).** Without pre-computed document embeddings, ColBERT reranking requires a forward pass per document. For K>1000, build a PLAID index instead.

---

## References

- Khattab & Zaharia. "ColBERT: Efficient and Effective Passage Search via Contextualized Late Interaction over BERT" (SIGIR 2020)
- Santhanam et al. "ColBERTv2: Effective and Efficient Retrieval via Lightweight Late Interaction" (NAACL 2022)
- Santhanam et al. "PLAID: An Efficient Engine for Late Interaction Retrieval" (CIKM 2022)
- RAGatouille: https://github.com/bclavie/RAGatouille
- ColBERT repository: https://github.com/stanford-futuredata/ColBERT
