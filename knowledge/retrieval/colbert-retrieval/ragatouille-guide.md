# RAGatouille: ColBERT Made Easy

## Overview

RAGatouille is a Python library that wraps ColBERT/ColBERTv2 into a simple, high-level API for indexing, searching, and fine-tuning. It handles all the complexity of PLAID indexing, residual compression, and ANN search behind a clean interface. If you want to use ColBERT in a RAG system without dealing with the low-level ColBERT codebase, RAGatouille is the standard approach.

**Installation:**
```bash
pip install ragatouille
```

**Key dependency**: RAGatouille uses the `colbert-ai` package under the hood, which requires PyTorch. GPU is recommended for indexing but not required for search.

---

## Loading a Pretrained Model

```python
from ragatouille import RAGPretrainedModel

# Load the standard ColBERTv2 checkpoint (downloads ~400MB on first use)
rag = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")
```

Other available checkpoints:

| Checkpoint | Description | Size |
|---|---|---|
| `colbert-ir/colbertv2.0` | Standard ColBERTv2 trained on MS MARCO | 438 MB |
| `jinaai/jina-colbert-v2` | Jina's ColBERT variant with longer context | 560 MB |
| `answerdotai/answerai-colbert-small-v1` | Smaller, faster model | 134 MB |

```python
# Load alternative models
rag_jina = RAGPretrainedModel.from_pretrained("jinaai/jina-colbert-v2")
rag_small = RAGPretrainedModel.from_pretrained("answerdotai/answerai-colbert-small-v1")
```

---

## Indexing a Corpus

### Basic Indexing

```python
# A list of strings to index
corpus = [
    "ColBERT encodes documents into per-token embeddings for fine-grained matching.",
    "BM25 uses term frequency and document length normalization for relevance scoring.",
    "Dense retrieval models compress documents into single dense vectors.",
    "Re-ranking with cross-encoders improves precision on the top retrieved candidates.",
    "Hybrid search combines keyword matching with semantic vector search.",
]

# Create an index -- this encodes all documents and builds the PLAID index
index_path = rag.index(
    collection=corpus,
    index_name="my_search_index",
    max_document_length=256,       # max tokens per document
    split_documents=True,          # split long documents into passages
)

print(f"Index saved to: {index_path}")
# Typically: .ragatouille/colbert/indexes/my_search_index/
```

### Indexing with Document IDs and Metadata

```python
# Documents with custom IDs
documents = [
    "ColBERT uses late interaction for efficient neural retrieval.",
    "HNSW graphs provide approximate nearest neighbor search.",
    "Inverted file indexes cluster vectors for faster search.",
]

document_ids = ["doc_colbert_001", "doc_hnsw_002", "doc_ivf_003"]

# Document metadata (stored alongside, queryable after retrieval)
document_metadatas = [
    {"source": "paper", "year": 2020, "topic": "neural-retrieval"},
    {"source": "textbook", "year": 2018, "topic": "ann-search"},
    {"source": "paper", "year": 2019, "topic": "ann-search"},
]

index_path = rag.index(
    collection=documents,
    document_ids=document_ids,
    document_metadatas=document_metadatas,
    index_name="annotated_index",
    max_document_length=256,
    split_documents=False,         # documents are already passage-sized
)
```

### Indexing Large Corpora

For corpora with millions of documents, batch indexing is essential:

```python
import json

# Load documents from a JSONL file
def load_corpus(path):
    docs = []
    doc_ids = []
    with open(path) as f:
        for line in f:
            obj = json.loads(line)
            docs.append(obj["text"])
            doc_ids.append(obj["id"])
    return docs, doc_ids

corpus, ids = load_corpus("corpus.jsonl")
print(f"Loaded {len(corpus)} documents")

# Index with GPU acceleration
# RAGatouille automatically detects and uses GPU if available
index_path = rag.index(
    collection=corpus,
    document_ids=ids,
    index_name="large_corpus",
    max_document_length=180,       # shorter = faster + less storage
    split_documents=True,          # will create sub-passages
    use_faiss=True,                # use FAISS for ANN (default)
)
```

**Indexing speed benchmarks (single A100 GPU):**

| Corpus Size | Time | Index Size (2-bit) |
|---|---|---|
| 10K docs | ~30 seconds | ~50 MB |
| 100K docs | ~5 minutes | ~500 MB |
| 1M docs | ~45 minutes | ~5 GB |
| 8.8M docs (MS MARCO) | ~4 hours | ~40 GB |

---

## Searching

### Basic Search

```python
# Load an existing index
rag = RAGPretrainedModel.from_index(".ragatouille/colbert/indexes/my_search_index/")

# Search
results = rag.search(query="how does late interaction work?", k=5)

for result in results:
    print(f"Score: {result['score']:.4f}")
    print(f"Content: {result['content'][:100]}")
    print(f"Doc ID: {result.get('document_id', 'N/A')}")
    print(f"Metadata: {result.get('document_metadata', {})}")
    print("---")
```

### Batch Search

```python
# Search multiple queries at once (more efficient than individual searches)
queries = [
    "what is late interaction?",
    "how does BM25 scoring work?",
    "dense retrieval vs sparse retrieval",
]

batch_results = rag.search(query=queries, k=5)

# batch_results is a list of lists
for query, results in zip(queries, batch_results):
    print(f"\nQuery: {query}")
    for r in results[:3]:
        print(f"  {r['score']:.4f} | {r['content'][:60]}")
```

### Tuning Search Quality

```python
# The k parameter controls how many results to return
# Under the hood, PLAID parameters control quality/speed tradeoff

# For highest quality (slower):
results = rag.search(query="neural retrieval", k=10)

# RAGatouille sets reasonable PLAID defaults:
#   ncells=1 or 2 (number of centroid cells to search)
#   centroid_score_threshold=0.5
#   ndocs=256 (candidates for exact scoring)
```

---

## Updating an Existing Index

```python
# Load existing index
rag = RAGPretrainedModel.from_index(".ragatouille/colbert/indexes/my_search_index/")

# Add new documents to the index
new_docs = [
    "SPLADE uses sparse learned representations for efficient retrieval.",
    "Matryoshka embeddings allow flexible dimensionality reduction.",
]

new_ids = ["doc_splade_004", "doc_matryoshka_005"]

rag.add_to_index(
    new_collection=new_docs,
    new_document_ids=new_ids,
    new_document_metadatas=[
        {"source": "paper", "year": 2021},
        {"source": "paper", "year": 2022},
    ],
)
```

**Note:** Adding documents to an existing index re-encodes only the new documents but may trigger a partial re-index of the PLAID structure. For large batch additions, it is often faster to rebuild the entire index.

---

## Deleting Documents

```python
rag = RAGPretrainedModel.from_index(".ragatouille/colbert/indexes/my_search_index/")

# Delete by document ID
rag.delete_from_index(
    document_ids=["doc_splade_004", "doc_matryoshka_005"]
)
```

---

## Fine-Tuning on Domain Data

### When to Fine-Tune

The pretrained ColBERTv2 model is trained on MS MARCO (web search queries). Fine-tuning improves quality when:
- Your domain has specialized vocabulary (medical, legal, code)
- Your queries are structurally different from web search queries
- You have labeled training data (query, relevant passage pairs)
- Zero-shot ColBERTv2 underperforms BM25 on your evaluation set

### Training Data Format

```python
# Training data: list of (query, positive_passage, negative_passage) triples
# Negative passages should be hard negatives (similar but not relevant)

training_data = [
    {
        "query": "what causes memory leaks in Python?",
        "positive": "Memory leaks in Python commonly occur from circular references, "
                    "unclosed file handles, global variable accumulation, and "
                    "C extension objects not properly releasing memory.",
        "negative": "Python uses automatic garbage collection with reference counting "
                    "as its primary mechanism, supplemented by a generational collector "
                    "for handling circular references."
    },
    {
        "query": "how to configure HNSW parameters?",
        "positive": "The key HNSW parameters are m (connections per node, default 16), "
                    "ef_construction (build quality, default 64), and ef_search "
                    "(query-time quality, default 40). Higher values improve recall "
                    "but increase latency and memory.",
        "negative": "HNSW (Hierarchical Navigable Small World) is a graph-based "
                    "approximate nearest neighbor algorithm published in 2016."
    },
    # ... more triples
]
```

### Fine-Tuning with RAGatouille

```python
from ragatouille import RAGTrainer

trainer = RAGTrainer(
    model_name="my_domain_colbert",
    pretrained_model_name="colbert-ir/colbertv2.0",
    language_code="en",
)

# Prepare training data
trainer.prepare_training_data(
    raw_data=training_data,
    data_out_path="./training_data/",
    num_new_negatives=10,           # mine additional hard negatives per query
    mine_hard_negatives=True,       # use the base model to find hard negatives
)

# Fine-tune
trainer.train(
    batch_size=32,                  # per-GPU batch size
    nbits=2,                       # compression bits (match your index setting)
    maxsteps=500_000,              # total training steps
    use_ib_negatives=True,         # in-batch negatives (improves quality)
    learning_rate=5e-6,            # lower than base training
    dim=128,                       # embedding dimension
    doc_maxlen=256,                # max document tokens
    query_maxlen=32,               # max query tokens
    warmup_steps=2000,
    accumsteps=1,                  # gradient accumulation steps
)

# The trained model is saved to:
# ./training_data/my_domain_colbert/
```

### Using the Fine-Tuned Model

```python
# Load your fine-tuned model
rag = RAGPretrainedModel.from_pretrained("./training_data/my_domain_colbert/")

# Index and search as before
rag.index(collection=my_corpus, index_name="domain_index")
results = rag.search("domain-specific query", k=10)
```

### Fine-Tuning Tips

1. **Minimum training data**: 1000-5000 query-passage pairs produce noticeable improvement. Below 1000, fine-tuning may hurt by overfitting.

2. **Hard negative mining**: always enable `mine_hard_negatives=True`. Random negatives are too easy and do not teach the model to distinguish subtle relevance differences.

3. **Learning rate**: use 1e-6 to 1e-5. Higher rates risk catastrophic forgetting of the pretrained knowledge.

4. **Validation**: hold out 10-20% of your labeled data and evaluate MRR@10 periodically during training to detect overfitting.

5. **Training steps**: for small datasets (1K-10K pairs), 10K-50K steps are usually sufficient. For large datasets (100K+ pairs), train for 100K-500K steps.

---

## Integration with LangChain

RAGatouille integrates with LangChain as a retriever:

```python
from ragatouille import RAGPretrainedModel
from langchain_core.documents import Document

# Create and populate a RAGatouille index
rag = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")
rag.index(
    collection=["doc1 text", "doc2 text", "doc3 text"],
    index_name="langchain_demo",
)

# Convert to a LangChain retriever
retriever = rag.as_langchain_retriever(k=5)

# Use in a LangChain chain
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

llm = ChatOpenAI(model="gpt-4o-mini")

prompt = ChatPromptTemplate.from_template("""
Answer the question based on the following context:

Context:
{context}

Question: {question}

Answer:
""")

def format_docs(docs: list[Document]) -> str:
    return "\n\n".join(doc.page_content for doc in docs)

chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

answer = chain.invoke("how does late interaction work?")
print(answer)
```

### Using with LlamaIndex

```python
from ragatouille import RAGPretrainedModel
from llama_index.core import VectorStoreIndex, ServiceContext
from llama_index.core.schema import TextNode

# Build RAGatouille index
rag = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")
rag.index(collection=texts, index_name="llamaindex_demo")

# Use RAGatouille's search directly in a custom retriever
from llama_index.core.retrievers import BaseRetriever
from llama_index.core.schema import NodeWithScore, QueryBundle

class ColBERTRetriever(BaseRetriever):
    def __init__(self, rag_model: RAGPretrainedModel, k: int = 5):
        super().__init__()
        self._rag = rag_model
        self._k = k

    def _retrieve(self, query_bundle: QueryBundle) -> list[NodeWithScore]:
        results = self._rag.search(query_bundle.query_str, k=self._k)
        nodes = []
        for r in results:
            node = TextNode(text=r["content"])
            nodes.append(NodeWithScore(node=node, score=r["score"]))
        return nodes

retriever = ColBERTRetriever(rag, k=5)
```

---

## Serving with an API

### FastAPI Server

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from ragatouille import RAGPretrainedModel

app = FastAPI(title="ColBERT Search API")

# Load index at startup
rag = RAGPretrainedModel.from_index(".ragatouille/colbert/indexes/production_index/")

class SearchRequest(BaseModel):
    query: str
    k: int = 10

class SearchResult(BaseModel):
    content: str
    score: float
    document_id: str | None = None

class SearchResponse(BaseModel):
    results: list[SearchResult]
    query: str

@app.post("/search", response_model=SearchResponse)
async def search(request: SearchRequest):
    if not request.query.strip():
        raise HTTPException(status_code=400, detail="Query cannot be empty")

    results = rag.search(query=request.query, k=request.k)

    return SearchResponse(
        query=request.query,
        results=[
            SearchResult(
                content=r["content"],
                score=r["score"],
                document_id=r.get("document_id"),
            )
            for r in results
        ],
    )

@app.post("/search/batch")
async def batch_search(queries: list[str], k: int = 10):
    batch_results = rag.search(query=queries, k=k)
    return [
        {
            "query": q,
            "results": [
                {"content": r["content"], "score": r["score"]}
                for r in results
            ]
        }
        for q, results in zip(queries, batch_results)
    ]
```

Run with:
```bash
uvicorn colbert_api:app --host 0.0.0.0 --port 8000
```

### Docker Deployment

```dockerfile
FROM python:3.11-slim

WORKDIR /app
RUN pip install ragatouille fastapi uvicorn

COPY colbert_api.py .
COPY .ragatouille/ .ragatouille/

# ColBERT index is read-only at serving time
ENV RAGATOUILLE_INDEX_PATH=.ragatouille/colbert/indexes/production_index/

EXPOSE 8000
CMD ["uvicorn", "colbert_api:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Common Pitfalls

1. **Not installing PyTorch with CUDA**: indexing on CPU is 10-50x slower than GPU. Ensure `torch` is installed with CUDA support: `pip install torch --index-url https://download.pytorch.org/whl/cu121`

2. **Indexing with `split_documents=True` on pre-chunked text**: if your documents are already passage-sized (100-300 tokens), set `split_documents=False` to avoid splitting them further.

3. **Loading the full model for search-only**: `RAGPretrainedModel.from_index()` is lighter than `from_pretrained()` because it does not need the full model weights for search (only the index). Use `from_index()` in production.

4. **Not persisting the index**: the index directory under `.ragatouille/` must be preserved. If running in containers, mount the index as a volume.

5. **Mixing index and model versions**: if you fine-tune a new model, you must rebuild the index. An index built with the base ColBERTv2 checkpoint is incompatible with a fine-tuned model.

6. **Memory issues on large corpora**: ColBERTv2 indexes can be large. Monitor memory usage during indexing and at load time. For corpora >5M documents, ensure at least 32 GB RAM for the index.

---

## References

- RAGatouille repository: https://github.com/AnswerDotAI/RAGatouille
- RAGatouille documentation: https://github.com/AnswerDotAI/RAGatouille/wiki
- ColBERTv2 model: https://huggingface.co/colbert-ir/colbertv2.0
- LangChain ColBERT integration: https://python.langchain.com/docs/integrations/retrievers/ragatouille/
