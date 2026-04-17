# RAGatouille -- Getting Started: Index, Search, and LangChain Integration

## Overview

This guide walks through setting up RAGatouille from installation to a working RAG pipeline. It covers indexing a document collection, running searches, integrating with LangChain and LlamaIndex, and building a complete question-answering system with ColBERT retrieval.

---

## Installation

### Basic Install

```bash
# Core package
pip install ragatouille

# With specific PyTorch version for CUDA
pip install ragatouille torch==2.3.1 --extra-index-url https://download.pytorch.org/whl/cu121

# For LangChain integration
pip install ragatouille langchain langchain-openai

# Verify installation
python -c "from ragatouille import RAGPretrainedModel; print('OK')"
```

### Environment Setup

```bash
# For LLM generation (used in the RAG pipeline, not for retrieval)
export OPENAI_API_KEY="sk-..."

# RAGatouille/ColBERT itself does not need API keys
# The model runs locally using PyTorch
```

### Hardware Requirements

| Workload | CPU | GPU | RAM | Notes |
|----------|-----|-----|-----|-------|
| Indexing <10K docs | OK (slow) | Recommended | 8 GB | CPU: ~50 docs/sec |
| Indexing >10K docs | Not practical | Required | 16 GB+ | GPU: ~1000 docs/sec |
| Search | OK | Optional | 4 GB | PLAID is CPU-efficient |
| Reranking | OK | Optional | 4 GB | Lightweight scoring |

---

## Step 1: Prepare Your Documents

```python
from pathlib import Path


def load_text_files(directory: str) -> tuple[list[str], list[str]]:
    """Load text files and return (contents, filenames)."""
    documents = []
    doc_ids = []
    for path in sorted(Path(directory).glob("**/*.txt")):
        text = path.read_text(encoding="utf-8")
        if text.strip():
            documents.append(text)
            doc_ids.append(path.stem)
    return documents, doc_ids


def load_markdown_files(directory: str) -> tuple[list[str], list[dict]]:
    """Load markdown files with metadata."""
    documents = []
    metadatas = []
    for path in sorted(Path(directory).glob("**/*.md")):
        text = path.read_text(encoding="utf-8")
        if text.strip():
            documents.append(text)
            metadatas.append({
                "source": str(path),
                "filename": path.name,
                "category": path.parent.name,
            })
    return documents, metadatas


# Example: load documents
documents, doc_ids = load_text_files("./data/docs")
print(f"Loaded {len(documents)} documents")
print(f"Average length: {sum(len(d.split()) for d in documents) / len(documents):.0f} words")
```

### Preparing Documents for Optimal Retrieval

```python
def prepare_documents(
    raw_documents: list[str],
    min_length: int = 50,
    max_length: int = 2000,
) -> list[str]:
    """Clean and filter documents for indexing."""
    prepared = []
    for doc in raw_documents:
        # Basic cleaning
        text = " ".join(doc.split())  # normalize whitespace

        # Skip too-short documents
        if len(text.split()) < min_length:
            continue

        # Truncate very long documents (RAGatouille will also split them)
        words = text.split()
        if len(words) > max_length:
            text = " ".join(words[:max_length])

        prepared.append(text)

    print(f"Prepared {len(prepared)}/{len(raw_documents)} documents")
    return prepared


documents = prepare_documents(documents)
```

---

## Step 2: Create the Index

```python
from ragatouille import RAGPretrainedModel

# Initialize with a pretrained ColBERT model
rag = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")

# Build the PLAID index
index_path = rag.index(
    index_name="my_knowledge_base",
    collection=documents,
    document_ids=doc_ids if doc_ids else None,
    document_metadatas=None,
    split_documents=True,       # auto-split long documents into passages
    max_document_length=256,    # max tokens per passage after splitting
    use_faiss=True,             # use FAISS for fast centroid search
)

print(f"Index built at: {index_path}")
```

### Indexing with Metadata

```python
documents = [
    "ColBERT uses late interaction for retrieval.",
    "PLAID compresses token embeddings efficiently.",
    "RAGatouille provides a simple Python API for ColBERT.",
]

metadatas = [
    {"source": "colbert-paper", "topic": "architecture"},
    {"source": "plaid-paper", "topic": "indexing"},
    {"source": "ragatouille-docs", "topic": "api"},
]

index_path = rag.index(
    index_name="docs_with_metadata",
    collection=documents,
    document_metadatas=metadatas,
    split_documents=True,
)
```

### Monitoring Index Build Progress

```python
import time

# For large collections, track indexing time
start = time.time()

index_path = rag.index(
    index_name="large_collection",
    collection=documents,  # could be 100K+ documents
    split_documents=True,
    max_document_length=256,
)

elapsed = time.time() - start
print(f"Indexed {len(documents)} documents in {elapsed:.1f}s")
print(f"Throughput: {len(documents) / elapsed:.0f} docs/sec")
```

---

## Step 3: Search

```python
from ragatouille import RAGPretrainedModel

# Load from existing index
rag = RAGPretrainedModel.from_index(
    ".ragatouille/colbert/indexes/my_knowledge_base"
)

# Single query search
results = rag.search(
    query="How does ColBERT compute relevance?",
    k=5,
)

for i, result in enumerate(results):
    print(f"\n--- Result {i + 1} ---")
    print(f"Score:       {result['score']:.4f}")
    print(f"Content:     {result['content'][:200]}...")
    print(f"Document ID: {result['document_id']}")
    if result.get('document_metadata'):
        print(f"Metadata:    {result['document_metadata']}")
```

### Batch Search

```python
# Search multiple queries at once (more efficient than looping)
queries = [
    "What is late interaction retrieval?",
    "How does PLAID indexing work?",
    "What are the benefits of ColBERT over dense retrieval?",
]

batch_results = rag.search(
    query=queries,
    k=5,
)

for query, results in zip(queries, batch_results):
    print(f"\nQuery: {query}")
    for r in results[:3]:
        print(f"  [{r['score']:.3f}] {r['content'][:80]}...")
```

### Search with Score Filtering

```python
def search_with_threshold(
    rag: RAGPretrainedModel,
    query: str,
    k: int = 10,
    min_score: float = 15.0,
) -> list[dict]:
    """Search and filter results below a score threshold."""
    results = rag.search(query=query, k=k)
    filtered = [r for r in results if r["score"] >= min_score]
    return filtered


results = search_with_threshold(rag, "ColBERT architecture", min_score=20.0)
print(f"Found {len(results)} results above threshold")
```

---

## Step 4: Build a Complete RAG Pipeline

### With LangChain

```python
from ragatouille import RAGPretrainedModel
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough, RunnableLambda
from langchain_openai import ChatOpenAI

# Load the ColBERT index
rag = RAGPretrainedModel.from_index(
    ".ragatouille/colbert/indexes/my_knowledge_base"
)

# Convert to LangChain retriever
retriever = rag.as_langchain_retriever(k=5)

# Build the RAG chain
prompt = ChatPromptTemplate.from_template(
    """You are a knowledgeable assistant. Answer the question using ONLY
the provided context. If the context does not contain the answer,
say "I don't have enough information."

Context:
{context}

Question: {question}

Answer:"""
)

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)


def format_docs(docs):
    return "\n\n".join(
        f"[{i+1}] {doc.page_content}" for i, doc in enumerate(docs)
    )


chain = (
    {
        "context": retriever | format_docs,
        "question": RunnablePassthrough(),
    }
    | prompt
    | llm
    | StrOutputParser()
)

# Query the RAG pipeline
answer = chain.invoke("How does ColBERT score documents?")
print(answer)
```

### With LangChain and Source Tracking

```python
from langchain_core.runnables import RunnableParallel


def build_rag_chain_with_sources(rag_model, k: int = 5):
    """Build a LangChain RAG chain that returns both the answer and source documents."""
    retriever = rag_model.as_langchain_retriever(k=k)

    prompt = ChatPromptTemplate.from_template(
        """Answer the question based on the context below. Cite passage numbers.

Context:
{context}

Question: {question}

Answer (cite [1], [2], etc.):"""
    )

    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

    # Retrieve and format in parallel
    def get_context_and_docs(question):
        docs = retriever.invoke(question)
        context = "\n\n".join(
            f"[{i+1}] {doc.page_content}" for i, doc in enumerate(docs)
        )
        return {"context": context, "docs": docs, "question": question}

    def generate_answer(inputs):
        result = prompt | llm | StrOutputParser()
        answer = result.invoke({
            "context": inputs["context"],
            "question": inputs["question"],
        })
        return {
            "answer": answer,
            "sources": [
                {
                    "content": doc.page_content[:200],
                    "metadata": doc.metadata,
                }
                for doc in inputs["docs"]
            ],
        }

    chain = RunnableLambda(get_context_and_docs) | RunnableLambda(generate_answer)
    return chain


chain = build_rag_chain_with_sources(rag, k=5)
result = chain.invoke("What is PLAID indexing?")
print(f"Answer: {result['answer']}\n")
print("Sources:")
for s in result["sources"]:
    print(f"  - {s['content'][:100]}...")
```

### Standalone Pipeline (No LangChain)

```python
from ragatouille import RAGPretrainedModel
import openai


class ColBERTRAGPipeline:
    """Standalone RAG pipeline using ColBERT retrieval and OpenAI generation."""

    def __init__(
        self,
        index_path: str,
        model: str = "gpt-4o-mini",
        k: int = 5,
    ):
        self.rag = RAGPretrainedModel.from_index(index_path)
        self.client = openai.OpenAI()
        self.model = model
        self.k = k

    def query(self, question: str) -> dict:
        # Retrieve with ColBERT
        results = self.rag.search(query=question, k=self.k)

        # Format context
        context_parts = []
        for i, r in enumerate(results):
            context_parts.append(f"[{i+1}] (score: {r['score']:.2f}) {r['content']}")
        context = "\n\n".join(context_parts)

        # Generate answer
        response = self.client.chat.completions.create(
            model=self.model,
            temperature=0.0,
            messages=[
                {
                    "role": "system",
                    "content": "Answer questions using ONLY the provided context. "
                               "Cite passage numbers in brackets.",
                },
                {
                    "role": "user",
                    "content": f"Context:\n{context}\n\nQuestion: {question}",
                },
            ],
        )

        return {
            "answer": response.choices[0].message.content,
            "sources": results,
            "tokens": response.usage.total_tokens,
        }


# Usage
pipeline = ColBERTRAGPipeline(
    index_path=".ragatouille/colbert/indexes/my_knowledge_base",
    model="gpt-4o-mini",
    k=5,
)

result = pipeline.query("How does late interaction work in ColBERT?")
print(result["answer"])
```

---

## Step 5: Incremental Updates

### Adding Documents to an Existing Index

```python
rag = RAGPretrainedModel.from_index(
    ".ragatouille/colbert/indexes/my_knowledge_base"
)

new_documents = [
    "ColBERTv2 achieves state-of-the-art retrieval performance on MS MARCO.",
    "The PLAID engine supports billion-scale document collections.",
    "Late interaction models can be fine-tuned on domain-specific data.",
]

new_metadatas = [
    {"source": "benchmark-results", "date": "2025-01"},
    {"source": "plaid-scalability", "date": "2025-01"},
    {"source": "finetuning-guide", "date": "2025-01"},
]

rag.add_to_index(
    new_collection=new_documents,
    new_document_metadatas=new_metadatas,
)

# Verify
results = rag.search("ColBERTv2 MS MARCO performance", k=3)
print(f"Found {len(results)} results after update")
```

### Deleting and Rebuilding

```python
import shutil

# Delete an index
index_path = ".ragatouille/colbert/indexes/my_knowledge_base"
shutil.rmtree(index_path, ignore_errors=True)

# Rebuild from scratch
rag = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")
rag.index(
    index_name="my_knowledge_base",
    collection=all_documents,  # full corpus
    split_documents=True,
)
```

---

## Step 6: Serving with FastAPI

```python
# server.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from ragatouille import RAGPretrainedModel
import openai

app = FastAPI(title="ColBERT RAG API")


class SearchRequest(BaseModel):
    query: str
    k: int = 5


class RAGRequest(BaseModel):
    question: str
    k: int = 5


class SearchResult(BaseModel):
    content: str
    score: float
    document_id: str


class RAGResponse(BaseModel):
    answer: str
    sources: list[SearchResult]


# Initialize at startup
rag = RAGPretrainedModel.from_index(
    ".ragatouille/colbert/indexes/my_knowledge_base"
)
oai_client = openai.OpenAI()


@app.post("/search", response_model=list[SearchResult])
async def search(request: SearchRequest):
    results = rag.search(query=request.query, k=request.k)
    return [
        SearchResult(
            content=r["content"],
            score=r["score"],
            document_id=str(r.get("document_id", "")),
        )
        for r in results
    ]


@app.post("/ask", response_model=RAGResponse)
async def ask(request: RAGRequest):
    # Retrieve
    results = rag.search(query=request.question, k=request.k)

    if not results:
        raise HTTPException(status_code=404, detail="No relevant documents found")

    context = "\n".join(f"[{i+1}] {r['content']}" for i, r in enumerate(results))

    # Generate
    response = oai_client.chat.completions.create(
        model="gpt-4o-mini",
        temperature=0.0,
        messages=[
            {"role": "system", "content": "Answer using only the provided context."},
            {"role": "user", "content": f"Context:\n{context}\n\nQuestion: {request.question}"},
        ],
    )

    return RAGResponse(
        answer=response.choices[0].message.content,
        sources=[
            SearchResult(
                content=r["content"],
                score=r["score"],
                document_id=str(r.get("document_id", "")),
            )
            for r in results
        ],
    )


# Run: uvicorn server:app --host 0.0.0.0 --port 8000
```

---

## Common Pitfalls

1. **Forgetting to set `split_documents=True`**: Without splitting, long documents are truncated to `max_document_length` tokens. This silently loses content. Always enable splitting unless your documents are already chunked.

2. **Loading model instead of index**: `from_pretrained()` loads the model for indexing. `from_index()` loads an existing index for search. Calling `search()` on a `from_pretrained()` instance without indexing first raises an error.

3. **Index path confusion**: The default index location is `.ragatouille/colbert/indexes/{index_name}/`. When loading, use the full path including the index name directory.

4. **Not using batch search**: Calling `search()` in a loop is much slower than passing a list of queries. Always batch queries when possible.

5. **GPU memory during indexing**: ColBERT loads the full BERT model during indexing. If GPU memory is insufficient, reduce `max_document_length` or index in smaller batches.

6. **Expecting metadata in search results**: Metadata is only returned if you provided `document_metadatas` during indexing. Without it, `document_metadata` in results is None.

---

## References

- RAGatouille documentation: https://github.com/AnswerDotAI/RAGatouille
- ColBERT GitHub: https://github.com/stanford-futuredata/ColBERT
- Ben Clavie (RAGatouille author) blog: https://ben.clavie.eu/
- LangChain integration: https://python.langchain.com/docs/integrations/retrievers/ragatouille/
