# Haystack 2.x -- Getting Started: Indexing and RAG Pipeline

## Overview

This guide walks through building a complete RAG system with Haystack 2.x, from installing the framework to running indexing and query pipelines. By the end you will have a working system that indexes documents from files, stores embeddings in a vector database, and answers questions with cited sources.

---

## Installation

### Basic Install

```bash
# Core Haystack
pip install haystack-ai

# Common integrations
pip install sentence-transformers   # local embedding models
pip install "anthropic-haystack"    # Anthropic Claude support
pip install "ollama-haystack"       # local LLMs via Ollama
```

### Production Install with a Vector Store

```bash
# With Qdrant
pip install haystack-ai qdrant-haystack sentence-transformers

# With pgvector
pip install haystack-ai pgvector-haystack sentence-transformers

# With Elasticsearch
pip install haystack-ai elasticsearch-haystack sentence-transformers

# With Pinecone
pip install haystack-ai pinecone-haystack
```

### Verify Installation

```python
import haystack
print(f"Haystack version: {haystack.__version__}")

from haystack import Pipeline, Document
from haystack.document_stores.in_memory import InMemoryDocumentStore

# Quick sanity check
store = InMemoryDocumentStore()
store.write_documents([Document(content="Hello, Haystack!")])
assert store.count_documents() == 1
print("Installation OK")
```

### Environment Variables

```bash
# For OpenAI models
export OPENAI_API_KEY="sk-..."

# For Anthropic models
export ANTHROPIC_API_KEY="sk-ant-..."

# For HuggingFace models (optional, for gated models)
export HF_TOKEN="hf_..."
```

---

## Part 1: The Indexing Pipeline

The indexing pipeline converts raw files into embedded documents stored in a vector database. This runs once (or on update) before the query pipeline.

### Step 1: Set Up the Document Store

```python
from haystack.document_stores.in_memory import InMemoryDocumentStore

# For development: in-memory store
document_store = InMemoryDocumentStore(
    embedding_similarity_function="cosine",
)
```

For production, use a persistent store:

```python
# Qdrant (recommended for most use cases)
from haystack_integrations.document_stores.qdrant import QdrantDocumentStore

document_store = QdrantDocumentStore(
    url="http://localhost:6333",
    index="knowledge_base",
    embedding_dim=384,              # must match your embedding model
    recreate_index=False,           # True only on first run
    hnsw_config={"m": 16, "ef_construct": 128},
    return_embedding=False,
)

# pgvector (if you already have PostgreSQL)
from haystack_integrations.document_stores.pgvector import PgvectorDocumentStore

document_store = PgvectorDocumentStore(
    connection_string="postgresql://user:pass@localhost:5432/vectordb",
    table_name="haystack_docs",
    embedding_dimension=384,
    vector_function="cosine_similarity",
    recreate_table=False,
    hnsw_recreate=False,
)
```

### Step 2: Build the Indexing Pipeline

```python
from haystack import Pipeline
from haystack.components.converters import (
    TextFileToDocument,
    PyPDFToDocument,
    MarkdownToDocument,
)
from haystack.components.preprocessors import DocumentCleaner, DocumentSplitter
from haystack.components.embedders import SentenceTransformersDocumentEmbedder
from haystack.components.writers import DocumentWriter
from haystack.components.routers import FileTypeRouter
from haystack.components.joiners import DocumentJoiner

indexing_pipeline = Pipeline()

# File type router: routes files to the correct converter
indexing_pipeline.add_component(
    "file_router",
    FileTypeRouter(mime_types=["text/plain", "application/pdf", "text/markdown"]),
)

# File converters
indexing_pipeline.add_component("txt_converter", TextFileToDocument())
indexing_pipeline.add_component("pdf_converter", PyPDFToDocument())
indexing_pipeline.add_component("md_converter", MarkdownToDocument())

# Join all converted documents into a single stream
indexing_pipeline.add_component("joiner", DocumentJoiner())

# Clean documents (remove whitespace artifacts, headers/footers)
indexing_pipeline.add_component(
    "cleaner",
    DocumentCleaner(
        remove_empty_lines=True,
        remove_extra_whitespaces=True,
    ),
)

# Split into chunks
indexing_pipeline.add_component(
    "splitter",
    DocumentSplitter(
        split_by="sentence",
        split_length=5,         # 5 sentences per chunk
        split_overlap=1,        # 1 sentence overlap between chunks
        split_threshold=3,      # minimum 3 sentences for a valid chunk
    ),
)

# Embed chunks
indexing_pipeline.add_component(
    "embedder",
    SentenceTransformersDocumentEmbedder(
        model="BAAI/bge-small-en-v1.5",
        batch_size=64,
        normalize_embeddings=True,
        progress_bar=True,
    ),
)

# Write to document store
indexing_pipeline.add_component(
    "writer",
    DocumentWriter(
        document_store=document_store,
        policy="overwrite",     # overwrite existing docs with same ID
    ),
)

# Connect the pipeline
indexing_pipeline.connect("file_router.text/plain", "txt_converter.sources")
indexing_pipeline.connect("file_router.application/pdf", "pdf_converter.sources")
indexing_pipeline.connect("file_router.text/markdown", "md_converter.sources")
indexing_pipeline.connect("txt_converter.documents", "joiner.documents")
indexing_pipeline.connect("pdf_converter.documents", "joiner.documents")
indexing_pipeline.connect("md_converter.documents", "joiner.documents")
indexing_pipeline.connect("joiner.documents", "cleaner.documents")
indexing_pipeline.connect("cleaner.documents", "splitter.documents")
indexing_pipeline.connect("splitter.documents", "embedder.documents")
indexing_pipeline.connect("embedder.documents", "writer.documents")
```

### Step 3: Run the Indexing Pipeline

```python
from pathlib import Path

# Collect all files to index
data_dir = Path("./data")
files = (
    list(data_dir.glob("**/*.txt"))
    + list(data_dir.glob("**/*.pdf"))
    + list(data_dir.glob("**/*.md"))
)

print(f"Indexing {len(files)} files...")

result = indexing_pipeline.run(
    {"file_router": {"sources": [str(f) for f in files]}}
)

written = result["writer"]["documents_written"]
print(f"Indexed {written} document chunks")
print(f"Total documents in store: {document_store.count_documents()}")
```

### Step 4: Verify the Index

```python
from haystack.document_stores.in_memory import InMemoryDocumentStore

# Check document count
count = document_store.count_documents()
print(f"Documents in store: {count}")

# Sample a document to verify embeddings exist
from haystack.document_stores.types import FilterPolicy

filters = {"field": "meta.source", "operator": "==", "value": "docs"}
sample = document_store.filter_documents(filters=None)[:3]

for doc in sample:
    print(f"Content: {doc.content[:100]}...")
    print(f"Embedding: {len(doc.embedding)} dimensions")
    print(f"Meta: {doc.meta}")
    print("---")
```

---

## Part 2: The RAG Query Pipeline

The query pipeline takes a user question, retrieves relevant chunks, and generates an answer.

### Step 1: Build the Query Pipeline

```python
from haystack import Pipeline
from haystack.components.embedders import SentenceTransformersTextEmbedder
from haystack.components.retrievers.in_memory import InMemoryEmbeddingRetriever
from haystack.components.builders import PromptBuilder
from haystack.components.generators import OpenAIGenerator

# For Qdrant, use:
# from haystack_integrations.components.retrievers.qdrant import QdrantEmbeddingRetriever

rag_pipeline = Pipeline()

# Embed the query
rag_pipeline.add_component(
    "text_embedder",
    SentenceTransformersTextEmbedder(
        model="BAAI/bge-small-en-v1.5",
        normalize_embeddings=True,
    ),
)

# Retrieve relevant documents
rag_pipeline.add_component(
    "retriever",
    InMemoryEmbeddingRetriever(
        document_store=document_store,
        top_k=5,
        scale_score=True,   # normalize scores to [0, 1]
    ),
)

# Build the prompt
rag_pipeline.add_component(
    "prompt",
    PromptBuilder(
        template="""
You are a knowledgeable assistant. Answer the question using ONLY the provided context.
If the context does not contain enough information to answer, say "I don't have enough information to answer this question."

Include specific references to the source documents when possible.

Context:
{% for doc in documents %}
[{{ loop.index }}] {{ doc.content }}
{% if doc.meta.get("source") %}(Source: {{ doc.meta["source"] }}){% endif %}
---
{% endfor %}

Question: {{ question }}

Answer:
"""
    ),
)

# Generate the answer
rag_pipeline.add_component(
    "llm",
    OpenAIGenerator(
        model="gpt-4o-mini",
        generation_kwargs={"temperature": 0.0, "max_tokens": 500},
    ),
)

# Connect the pipeline
rag_pipeline.connect("text_embedder.embedding", "retriever.query_embedding")
rag_pipeline.connect("retriever.documents", "prompt.documents")
rag_pipeline.connect("prompt.prompt", "llm.prompt")
```

### Step 2: Run Queries

```python
def ask(question: str) -> dict:
    """Run a RAG query and return the answer with sources."""
    result = rag_pipeline.run(
        {
            "text_embedder": {"text": question},
            "prompt": {"question": question},
        }
    )

    answer = result["llm"]["replies"][0]
    meta = result["llm"]["meta"][0]
    sources = [
        {
            "content": doc.content[:200],
            "score": doc.score,
            "source": doc.meta.get("source", "unknown"),
        }
        for doc in result["retriever"]["documents"]
    ]

    return {
        "answer": answer,
        "sources": sources,
        "tokens_used": meta.get("usage", {}),
    }


# Test it
response = ask("What architecture does Haystack 2.x use?")
print(f"Answer: {response['answer']}\n")
print("Sources:")
for s in response["sources"]:
    print(f"  [{s['score']:.3f}] {s['content'][:80]}...")
```

### Step 3: Using Anthropic Instead of OpenAI

```python
from haystack_integrations.components.generators.anthropic import AnthropicGenerator

rag_pipeline_anthropic = Pipeline()

rag_pipeline_anthropic.add_component(
    "text_embedder",
    SentenceTransformersTextEmbedder(model="BAAI/bge-small-en-v1.5"),
)
rag_pipeline_anthropic.add_component(
    "retriever",
    InMemoryEmbeddingRetriever(document_store=document_store, top_k=5),
)
rag_pipeline_anthropic.add_component(
    "prompt",
    PromptBuilder(
        template="""Answer the question using only the context below.

Context:
{% for doc in documents %}
- {{ doc.content }}
{% endfor %}

Question: {{ question }}
Answer:"""
    ),
)
rag_pipeline_anthropic.add_component(
    "llm",
    AnthropicGenerator(
        model="claude-sonnet-4-20250514",
        generation_kwargs={"temperature": 0.0, "max_tokens": 500},
    ),
)

rag_pipeline_anthropic.connect("text_embedder.embedding", "retriever.query_embedding")
rag_pipeline_anthropic.connect("retriever.documents", "prompt.documents")
rag_pipeline_anthropic.connect("prompt.prompt", "llm.prompt")
```

---

## Part 3: Hybrid Retrieval Pipeline

Combine BM25 (keyword) and embedding (semantic) retrieval for better results:

```python
from haystack import Pipeline
from haystack.components.embedders import SentenceTransformersTextEmbedder
from haystack.components.retrievers.in_memory import (
    InMemoryBM25Retriever,
    InMemoryEmbeddingRetriever,
)
from haystack.components.joiners import DocumentJoiner
from haystack.components.builders import PromptBuilder
from haystack.components.generators import OpenAIGenerator

hybrid_pipeline = Pipeline()

# Query embedding
hybrid_pipeline.add_component(
    "text_embedder",
    SentenceTransformersTextEmbedder(model="BAAI/bge-small-en-v1.5"),
)

# Two retrievers
hybrid_pipeline.add_component(
    "bm25_retriever",
    InMemoryBM25Retriever(document_store=document_store, top_k=5),
)
hybrid_pipeline.add_component(
    "embedding_retriever",
    InMemoryEmbeddingRetriever(document_store=document_store, top_k=5),
)

# Merge results with reciprocal rank fusion
hybrid_pipeline.add_component(
    "joiner",
    DocumentJoiner(
        join_mode="reciprocal_rank_fusion",
        top_k=5,
    ),
)

# Prompt and LLM
hybrid_pipeline.add_component(
    "prompt",
    PromptBuilder(
        template="""Answer using ONLY the context below.

Context:
{% for doc in documents %}
[{{ loop.index }}] {{ doc.content }}
{% endfor %}

Question: {{ question }}
Answer:"""
    ),
)
hybrid_pipeline.add_component(
    "llm",
    OpenAIGenerator(model="gpt-4o-mini", generation_kwargs={"temperature": 0.0}),
)

# Connect
hybrid_pipeline.connect("text_embedder.embedding", "embedding_retriever.query_embedding")
hybrid_pipeline.connect("bm25_retriever.documents", "joiner.documents")
hybrid_pipeline.connect("embedding_retriever.documents", "joiner.documents")
hybrid_pipeline.connect("joiner.documents", "prompt.documents")
hybrid_pipeline.connect("prompt.prompt", "llm.prompt")

# Run hybrid query
result = hybrid_pipeline.run(
    {
        "text_embedder": {"text": "What is the pipeline architecture?"},
        "bm25_retriever": {"query": "What is the pipeline architecture?"},
        "prompt": {"question": "What is the pipeline architecture?"},
    }
)
print(result["llm"]["replies"][0])
```

---

## Part 4: Serving the Pipeline

### FastAPI Server

```python
# server.py
from fastapi import FastAPI
from pydantic import BaseModel
import uvicorn

from haystack import Pipeline
from haystack.components.embedders import SentenceTransformersTextEmbedder
from haystack.components.retrievers.in_memory import InMemoryEmbeddingRetriever
from haystack.components.builders import PromptBuilder
from haystack.components.generators import OpenAIGenerator
from haystack.document_stores.in_memory import InMemoryDocumentStore

app = FastAPI(title="Haystack RAG API")


class QueryRequest(BaseModel):
    question: str
    top_k: int = 5


class QueryResponse(BaseModel):
    answer: str
    sources: list[dict]


# Initialize pipeline at startup
document_store = InMemoryDocumentStore()
# ... (load documents into store)

rag_pipeline = Pipeline()
rag_pipeline.add_component(
    "embedder", SentenceTransformersTextEmbedder(model="BAAI/bge-small-en-v1.5")
)
rag_pipeline.add_component(
    "retriever", InMemoryEmbeddingRetriever(document_store=document_store, top_k=5)
)
rag_pipeline.add_component(
    "prompt", PromptBuilder(template="Context: {% for d in documents %}{{ d.content }}{% endfor %}\n\nQ: {{ question }}\nA:")
)
rag_pipeline.add_component(
    "llm", OpenAIGenerator(model="gpt-4o-mini")
)
rag_pipeline.connect("embedder.embedding", "retriever.query_embedding")
rag_pipeline.connect("retriever.documents", "prompt.documents")
rag_pipeline.connect("prompt.prompt", "llm.prompt")


@app.post("/query", response_model=QueryResponse)
async def query(request: QueryRequest):
    result = rag_pipeline.run(
        {
            "embedder": {"text": request.question},
            "prompt": {"question": request.question},
        }
    )
    return QueryResponse(
        answer=result["llm"]["replies"][0],
        sources=[
            {"content": d.content[:200], "score": d.score}
            for d in result["retriever"]["documents"]
        ],
    )


if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## Project Structure

Recommended layout for a Haystack RAG project:

```
haystack-rag/
  pyproject.toml
  src/
    __init__.py
    config.py                  # document store and model configuration
    pipelines/
      __init__.py
      indexing.py              # indexing pipeline
      query.py                 # query pipeline (RAG)
      hybrid.py                # hybrid retrieval pipeline
    components/
      __init__.py
      custom_cleaner.py        # custom document cleaner
      reranker.py              # reranking component
    server.py                  # FastAPI server
  pipelines/
    indexing.yaml              # serialized indexing pipeline
    query.yaml                 # serialized query pipeline
  data/
    raw/                       # source documents
    processed/                 # intermediate outputs
  tests/
    test_indexing.py
    test_query.py
    test_components.py
  scripts/
    index.py                   # run indexing
    query.py                   # interactive query CLI
```

---

## Common Pitfalls

1. **Using `haystack` instead of `haystack-ai`**: The pip package for Haystack 2.x is `haystack-ai`, not `haystack` (which is the old 1.x version). Installing the wrong package gives confusing import errors.

2. **Mismatched embedding models between indexing and query**: The document embedder and text embedder must use the same model. Mixing models produces meaningless similarity scores.

3. **Not warming up components**: Local embedding models need warm-up. Inside a pipeline this happens automatically, but when using components standalone, call `component.warm_up()` first.

4. **Forgetting to provide all unconnected inputs**: When running a pipeline, every component input that is not connected to another component's output must be provided in the `run()` call. The error messages for missing inputs are not always clear.

5. **Using InMemoryDocumentStore in production**: The in-memory store is for development only. It does not persist between restarts and cannot handle large datasets. Use Qdrant, pgvector, or Elasticsearch for production.

6. **Not setting `recreate_index=False` after first run**: If you leave `recreate_index=True` on a Qdrant or pgvector store, it drops and recreates the collection on every pipeline instantiation, losing all indexed data.

---

## References

- Haystack documentation: https://docs.haystack.deepset.ai/docs/intro
- Haystack tutorials: https://haystack.deepset.ai/tutorials
- Haystack integrations: https://haystack.deepset.ai/integrations
- Haystack GitHub: https://github.com/deepset-ai/haystack
