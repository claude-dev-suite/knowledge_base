# ColBERT Late Interaction Retrieval

## Overview

ColBERT (Contextualized Late Interaction over BERT) is a neural retrieval model that combines the expressiveness of cross-encoders with the efficiency of bi-encoders through a "late interaction" mechanism. Instead of compressing an entire document into a single vector, ColBERT produces one embedding per token, then scores query-document relevance using cheap per-token operations (MaxSim) that can be partially precomputed offline.

This architecture sits between two extremes:
- **Bi-encoders** (e.g., Sentence-BERT): single vector per text, fast but lossy
- **Cross-encoders** (e.g., MS-MARCO MiniLM): joint query-document attention, accurate but O(N) at query time

ColBERT achieves 95-99% of cross-encoder quality while remaining practical for retrieval over millions of documents.

---

## How Late Interaction Works

### The Core Idea

In a bi-encoder, you compress "The quick brown fox jumps over the lazy dog" into a single 768-dimensional vector. Fine-grained token-level information is destroyed. In a cross-encoder, you concatenate query and document tokens and run full attention -- accurate but you must do this for every candidate document at query time.

ColBERT takes a middle path:

1. **Encode query and document independently** through BERT, producing per-token embeddings
2. **At scoring time**, compute a lightweight interaction between these two sets of token embeddings

```
Query:    "what is information retrieval?"
          -> [q1, q2, q3, q4]   (4 token embeddings, each 128-dim)

Document: "Information retrieval is the process of obtaining relevant data..."
          -> [d1, d2, d3, ..., d20]  (20 token embeddings, each 128-dim)

Score = sum over query tokens of max similarity to any document token
      = sum_i( max_j( q_i . d_j ) )
```

### MaxSim Operation

The scoring function is called MaxSim. For each query token embedding q_i, find the document token embedding d_j with the highest cosine similarity, then sum these maximum similarities:

```
Score(Q, D) = SUM_{i=1}^{|Q|}  MAX_{j=1}^{|D|}  (q_i * d_j^T)
```

This is the key insight: each query token "finds its best match" in the document. A query about "python decorators" will have the "python" token match document tokens about Python, and the "decorators" token match document tokens about decorators -- independently and precisely.

### Why This Works Better Than Single-Vector

Single-vector models must compress all semantic content into one point in embedding space. ColBERT preserves the full token-level semantics:

- **Polysemy handling**: the word "bank" gets different embeddings depending on context (river bank vs. financial bank) because BERT contextualizes each token
- **Multi-aspect queries**: "python async database connection pooling" has four distinct aspects, each independently matched
- **Exact term matching**: specific terms like function names or error codes get their own embeddings rather than being averaged away

---

## Architecture Details

### Encoding

Both query and document encoders share the same BERT backbone (typically bert-base-uncased or a distilled variant). The output is projected to a lower dimension (128 by default) to reduce storage:

```python
# Simplified ColBERT encoding
class ColBERT(nn.Module):
    def __init__(self, bert_model="bert-base-uncased", dim=128):
        super().__init__()
        self.bert = AutoModel.from_pretrained(bert_model)
        self.linear = nn.Linear(768, dim, bias=False)

    def encode_query(self, input_ids, attention_mask):
        # [Q] token prepended, padded to fixed length (32 tokens)
        outputs = self.bert(input_ids, attention_mask)
        embeddings = self.linear(outputs.last_hidden_state)
        return F.normalize(embeddings, dim=-1)

    def encode_document(self, input_ids, attention_mask):
        # [D] token prepended, punctuation tokens optionally filtered
        outputs = self.bert(input_ids, attention_mask)
        embeddings = self.linear(outputs.last_hidden_state)
        # Filter out padding and punctuation embeddings
        mask = self._get_skiplist_mask(input_ids)
        return F.normalize(embeddings[mask], dim=-1)
```

Key differences between query and document encoding:
- **Query**: prepends a special `[Q]` token, pads to a fixed length (32 tokens by default) with `[MASK]` tokens. The MASK tokens act as "soft expansion" -- BERT fills them with relevant context, improving recall
- **Document**: prepends a special `[D]` token, no padding, punctuation tokens are removed (they add storage without improving relevance)

### Offline Indexing and Online Search

The decomposability of late interaction enables a two-phase approach:

**Offline (index time):**
1. Encode all documents through the document encoder
2. Store per-token embeddings in an index (one 128-dim vector per token)
3. Build an approximate nearest-neighbor index over all token embeddings

**Online (query time):**
1. Encode the query through the query encoder (produces ~32 token embeddings)
2. For each query token, find the top-k nearest document tokens via ANN
3. Gather candidate documents from these token matches
4. Compute exact MaxSim scores for candidate documents
5. Return top results

This is much faster than running the full model on every document, because step 2 uses precomputed document embeddings.

---

## Comparison with Other Retrieval Models

| Property | BM25 | Bi-Encoder | ColBERT | Cross-Encoder |
|---|---|---|---|---|
| Document representation | Term frequencies | Single vector | Per-token vectors | None (online only) |
| Query-time computation | Term matching | Single dot product | MaxSim (multi dot products) | Full transformer pass |
| Semantic understanding | None | Good | Very good | Best |
| Latency (1M docs) | ~5ms | ~10ms | ~50-100ms | Infeasible |
| Storage per document | Inverted index | 1 vector (768-3072 dim) | N vectors (128 dim each) | None |
| Offline indexing | Yes | Yes | Yes | No |
| Handles polysemy | No | Partially | Yes | Yes |

---

## Storage Considerations

ColBERT's per-token representation is its main cost. A typical document of 200 tokens stored at 128 dimensions in float32:

```
200 tokens * 128 dims * 4 bytes = 102,400 bytes per document = 100 KB
```

For 1 million documents: ~100 GB of raw embeddings. This is 50-100x more than a bi-encoder (which stores one vector per document).

ColBERTv2 addresses this with residual compression (see `paper-v2-plaid.md`), reducing storage by 6-32x while maintaining quality.

---

## When to Use ColBERT

**Use ColBERT when:**
- You need better retrieval quality than bi-encoders but cannot afford cross-encoder latency on the full corpus
- Your queries are complex, multi-aspect, or contain specific technical terms
- You have the storage budget (or can use ColBERTv2 compression)
- You are building a RAG system where retrieval quality directly impacts generation quality

**Do not use ColBERT when:**
- Simple keyword matching (BM25) is sufficient -- e.g., exact code searches, ID lookups
- You have very tight latency requirements (<10ms) and a large corpus
- Storage is extremely constrained and you cannot use compression
- Your corpus is small enough (<10K documents) that cross-encoder re-ranking of all candidates is feasible

---

## Quick Start with RAGatouille

The fastest way to use ColBERT in Python is through the RAGatouille library (see `ragatouille-guide.md` for details):

```python
from ragatouille import RAGPretrainedModel

# Load pretrained ColBERTv2
rag = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")

# Index documents
rag.index(
    collection=[
        "ColBERT uses late interaction for efficient retrieval.",
        "BM25 is a traditional term-matching algorithm.",
        "Cross-encoders process query and document jointly.",
    ],
    index_name="my_index",
)

# Search
results = rag.search("how does late interaction work?", k=3)
for r in results:
    print(f"Score: {r['score']:.4f} | {r['content'][:80]}")
```

---

## Common Pitfalls

1. **Ignoring storage costs**: raw ColBERT embeddings for a large corpus can exceed available disk/RAM. Always estimate storage before indexing and consider ColBERTv2 compression.

2. **Using ColBERT for short documents**: the per-token advantage diminishes when documents are very short (1-2 sentences). A bi-encoder with re-ranking may be more cost-effective.

3. **Not filtering punctuation tokens**: including punctuation in document embeddings wastes storage and adds noise. The default ColBERT implementation filters these, but custom implementations may not.

4. **Fixed query length padding**: ColBERT pads queries to 32 tokens with [MASK]. If your queries are consistently much longer, you may need to increase this (at the cost of more MaxSim operations per document).

5. **Expecting real-time index updates**: adding new documents requires encoding and adding to the ANN index. Batch updates are much more efficient than single-document inserts.

---

## References

- Khattab, O. and Zaharia, M. "ColBERT: Efficient and Effective Passage Search via Contextualized Late Interaction over BERT." SIGIR 2020.
- Santhanam, K. et al. "ColBERTv2: Effective and Efficient Retrieval via Lightweight Late Interaction." NAACL 2022.
- RAGatouille library: https://github.com/AnswerDotAI/RAGatouille
- ColBERT repository: https://github.com/stanford-futuredata/ColBERT
- PLAID engine: https://arxiv.org/abs/2205.09707
