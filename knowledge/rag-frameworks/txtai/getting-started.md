# txtai -- Getting Started: Index, Search, and Workflows

## Overview

This guide walks through installing txtai, building a semantic search index, running searches with SQL and hybrid modes, creating processing workflows, and building a complete RAG pipeline. By the end you will have a local RAG system that runs without any API keys.

---

## Installation

```bash
# Core package (embeddings + search)
pip install txtai

# With all pipelines (generation, summarization, translation, transcription)
pip install "txtai[pipeline]"

# With API server support
pip install "txtai[api]"

# With graph support
pip install "txtai[graph]"

# Everything
pip install "txtai[all]"

# Verify
python -c "from txtai import Embeddings; print('txtai installed')"
```

### No API Keys Required

txtai runs entirely locally using Hugging Face models. No OpenAI, Anthropic, or other API keys are needed for basic use:

```python
from txtai import Embeddings

# This downloads and runs a local model -- no API key needed
embeddings = Embeddings(path="sentence-transformers/all-MiniLM-L6-v2")
embeddings.index(["txtai runs locally", "No API keys required"])

results = embeddings.search("local models", limit=1)
print(results)  # [(score, text)]
```

---

## Step 1: Build a Semantic Search Index

### Basic Index

```python
from txtai import Embeddings

# Create embeddings index with content storage
embeddings = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",
    content=True,  # store original text in SQLite
)

# Sample knowledge base
documents = [
    "txtai is a lightweight embeddings database for semantic search",
    "FAISS provides efficient vector similarity search at scale",
    "SQLite stores document content alongside vector embeddings",
    "Hybrid search combines BM25 keyword matching with vector similarity",
    "Pipelines run NLP tasks like summarization, translation, and generation",
    "Workflows chain multiple pipelines into processing DAGs",
    "Agents use LLMs to autonomously plan and execute tasks",
    "The Embeddings class supports SQL queries over semantic search results",
    "txtai can run on edge devices without internet connectivity",
    "Graph databases model relationships between indexed documents",
]

# Index the documents
embeddings.index(documents)
print(f"Indexed {embeddings.count()} documents")
```

### Index with IDs and Metadata

```python
# Index with explicit IDs and metadata
data = [
    (0, {"text": "txtai semantic search framework", "category": "search", "lang": "en"}),
    (1, {"text": "FAISS vector indexing library", "category": "search", "lang": "en"}),
    (2, {"text": "LLM text generation pipeline", "category": "generation", "lang": "en"}),
    (3, {"text": "Whisper audio transcription", "category": "audio", "lang": "en"}),
    (4, {"text": "BM25 keyword scoring algorithm", "category": "search", "lang": "en"}),
    (5, {"text": "SQLite embedded database engine", "category": "storage", "lang": "en"}),
]

embeddings = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",
    content=True,
)
embeddings.index(data)
```

### Index from Files

```python
from pathlib import Path


def index_text_files(embeddings: Embeddings, directory: str):
    """Index all text and markdown files from a directory."""
    data = []
    for i, path in enumerate(sorted(Path(directory).glob("**/*.md"))):
        text = path.read_text(encoding="utf-8")
        if text.strip():
            data.append((i, {
                "text": text,
                "filename": path.name,
                "category": path.parent.name,
                "path": str(path),
            }))

    embeddings.index(data)
    print(f"Indexed {len(data)} files")


embeddings = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",
    content=True,
)
index_text_files(embeddings, "./docs")
```

---

## Step 2: Search

### Basic Semantic Search

```python
# Simple text search
results = embeddings.search("vector similarity search", limit=5)

for score, text in results:
    print(f"  [{score:.4f}] {text}")
```

### SQL-Powered Search

```python
# Semantic search with SQL
results = embeddings.search(
    "SELECT id, text, score FROM txtai WHERE similar('vector search') LIMIT 5"
)
for row in results:
    print(f"  [{row['score']:.4f}] {row['id']}: {row['text']}")

# Semantic search with metadata filter
results = embeddings.search(
    "SELECT text, score FROM txtai WHERE similar('search') AND category = 'search' LIMIT 5"
)

# Aggregate queries
results = embeddings.search(
    "SELECT category, COUNT(*) as count FROM txtai GROUP BY category ORDER BY count DESC"
)
for row in results:
    print(f"  {row['category']}: {row['count']} documents")

# Full-text search (no vector, keyword only)
results = embeddings.search(
    "SELECT text FROM txtai WHERE text LIKE '%SQLite%'"
)
```

### Hybrid Search

```python
# Enable hybrid search (vector + BM25)
embeddings = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",
    content=True,
    hybrid=True,
    scoring={
        "method": "bm25",
        "terms": True,
        "normalize": True,
    },
)

embeddings.index(data)

# Hybrid search combines semantic similarity with keyword matching
results = embeddings.search("SQLite database storage", limit=5)

# The hybrid score blends:
# - Vector similarity (semantic understanding)
# - BM25 score (keyword matching)
# This is useful when you need both exact term matching and semantic understanding
```

### Batch Search

```python
queries = [
    "How does txtai store data?",
    "What NLP tasks are supported?",
    "Can txtai run offline?",
]

# Search all queries at once
batch_results = embeddings.batchsearch(queries, limit=3)

for query, results in zip(queries, batch_results):
    print(f"\nQuery: {query}")
    for score, text in results:
        print(f"  [{score:.4f}] {text}")
```

---

## Step 3: Persistence

### Save and Load

```python
# Save index to disk
embeddings.save("./my_search_index")

# Load index from disk
loaded = Embeddings()
loaded.load("./my_search_index")

# Verify
results = loaded.search("semantic search", limit=1)
print(f"Loaded index with {loaded.count()} documents")
```

### Incremental Updates

```python
# Load existing index
embeddings = Embeddings()
embeddings.load("./my_search_index")

# Add new documents (upsert = update or insert)
new_data = [
    (100, {"text": "New feature: graph-based knowledge representation", "category": "graph"}),
    (101, {"text": "Agent framework for autonomous task execution", "category": "agent"}),
]
embeddings.upsert(new_data)

# Delete documents
embeddings.delete([0, 1])  # delete by ID

# Save updated index
embeddings.save("./my_search_index")
```

---

## Step 4: Building a RAG Pipeline

### Local RAG (No API Keys)

```python
from txtai import Embeddings
from txtai.pipeline import LLM

# Create embeddings index
embeddings = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",
    content=True,
)
embeddings.index(documents)

# Create local LLM pipeline
llm = LLM("microsoft/Phi-3-mini-4k-instruct")


def rag_query(question: str, k: int = 3) -> str:
    """Local RAG: retrieve + generate."""
    # Retrieve relevant context
    results = embeddings.search(question, limit=k)
    context = "\n".join(f"- {text}" for _, text in results)

    # Generate answer with context
    prompt = f"""Answer the question based ONLY on the following context.
If the context does not contain the answer, say "I don't know."

Context:
{context}

Question: {question}

Answer:"""

    answer = llm(prompt, maxlength=300)
    return answer


# Test the RAG pipeline
answer = rag_query("What is txtai?")
print(f"Answer: {answer}")

answer = rag_query("How does hybrid search work?")
print(f"Answer: {answer}")
```

### RAG with OpenAI (When Quality Matters)

```python
from txtai import Embeddings
import openai


def rag_query_openai(
    embeddings: Embeddings,
    question: str,
    model: str = "gpt-4o-mini",
    k: int = 5,
) -> dict:
    """RAG pipeline using OpenAI for generation."""
    # Retrieve
    results = embeddings.search(question, limit=k)
    context = "\n".join(f"[{i+1}] {text}" for i, (_, text) in enumerate(results))

    # Generate
    client = openai.OpenAI()
    response = client.chat.completions.create(
        model=model,
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
        "sources": [{"score": s, "text": t[:200]} for s, t in results],
        "tokens": response.usage.total_tokens,
    }


result = rag_query_openai(embeddings, "What storage backends does txtai support?")
print(f"Answer: {result['answer']}")
print(f"Tokens used: {result['tokens']}")
```

---

## Step 5: Workflows

Workflows chain multiple operations into a processing pipeline:

### Basic Workflow

```python
from txtai import Embeddings
from txtai.workflow import Workflow, Task


def clean_text(texts):
    """Clean and normalize text."""
    return [" ".join(t.split()).lower().strip() for t in texts]


def filter_short(texts):
    """Remove texts shorter than 20 characters."""
    return [t for t in texts if len(t) >= 20]


# Define workflow
workflow = Workflow([
    Task(clean_text),
    Task(filter_short),
])

# Run workflow
raw_texts = [
    "  This is   a  messy  text  with extra   spaces  ",
    "Short",
    "This is a proper sentence with enough content to keep.",
]

results = list(workflow(raw_texts))
for text in results:
    print(f"  '{text}'")
```

### Workflow with Pipelines

```python
from txtai import Embeddings
from txtai.pipeline import Summary, Translation, LLM
from txtai.workflow import Workflow, Task

# Create pipelines
summarizer = Summary()
translator = Translation()

# Workflow: summarize then translate to French
workflow = Workflow([
    Task(lambda texts: [summarizer(t, maxlength=50) for t in texts]),
    Task(lambda texts: [translator(t, "fr") for t in texts]),
])

long_texts = [
    "txtai is an open-source framework that provides semantic search, "
    "LLM orchestration, and language model workflows. It supports local "
    "models, SQL queries over vector search, and runs on edge devices. "
    "The framework includes pipelines for text generation, summarization, "
    "translation, and transcription.",
]

results = list(workflow(long_texts))
print(results[0])  # summarized and translated to French
```

### RAG Workflow

```python
from txtai import Embeddings
from txtai.pipeline import LLM
from txtai.workflow import Workflow, Task

embeddings = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",
    content=True,
)
embeddings.index(documents)

llm = LLM("microsoft/Phi-3-mini-4k-instruct")


def retrieve_context(questions):
    """Retrieve context for each question."""
    contexts = []
    for q in questions:
        results = embeddings.search(q, limit=3)
        context = "\n".join(text for _, text in results)
        contexts.append((q, context))
    return contexts


def generate_answers(qa_pairs):
    """Generate answers using retrieved context."""
    answers = []
    for question, context in qa_pairs:
        prompt = f"Context: {context}\n\nQuestion: {question}\nAnswer:"
        answer = llm(prompt, maxlength=200)
        answers.append({"question": question, "answer": answer})
    return answers


# RAG workflow
rag_workflow = Workflow([
    Task(retrieve_context),
    Task(generate_answers),
])

questions = [
    "What is txtai?",
    "How does hybrid search work?",
]

results = list(rag_workflow(questions))
for r in results:
    print(f"Q: {r['question']}")
    print(f"A: {r['answer']}\n")
```

---

## Step 6: API Server

### Configuration

```yaml
# api_config.yml
embeddings:
  path: sentence-transformers/all-MiniLM-L6-v2
  content: true
  hybrid: true
  scoring:
    method: bm25
    terms: true

api:
  host: 0.0.0.0
  port: 8000
```

### Start the Server

```bash
# Start API server
python -m txtai.api api_config.yml

# Or with uvicorn
CONFIG=api_config.yml uvicorn "txtai.api:app" --host 0.0.0.0 --port 8000
```

### Use the API

```bash
# Add documents
curl -X POST http://localhost:8000/add \
  -H "Content-Type: application/json" \
  -d '[
    {"id": "0", "text": "txtai semantic search"},
    {"id": "1", "text": "vector similarity indexing"},
    {"id": "2", "text": "BM25 keyword search"}
  ]'

# Index
curl -X POST http://localhost:8000/index

# Search
curl "http://localhost:8000/search?query=search+framework&limit=3"

# SQL search
curl -X POST http://localhost:8000/search \
  -H "Content-Type: application/json" \
  -d '{"query": "SELECT text, score FROM txtai WHERE similar('\''search'\'') LIMIT 3"}'
```

### Python Client

```python
import httpx

client = httpx.Client(base_url="http://localhost:8000")

# Add and index
client.post("/add", json=[
    {"id": "0", "text": "txtai semantic search"},
    {"id": "1", "text": "vector indexing with FAISS"},
])
client.post("/index")

# Search
response = client.get("/search", params={"query": "search", "limit": 3})
results = response.json()
for r in results:
    print(f"  [{r['score']:.4f}] {r['text']}")
```

---

## Project Structure

```
txtai-rag-project/
  config.yml              # txtai configuration
  index/                  # persisted embeddings index
  data/
    documents/            # source documents to index
  src/
    indexer.py            # document indexing script
    rag.py                # RAG pipeline
    server.py             # API server customization
    workflows/
      ingestion.py        # document processing workflow
      rag_workflow.py     # RAG workflow definition
  tests/
    test_search.py
    test_rag.py
```

---

## Common Pitfalls

1. **Forgetting `content=True`**: Without this flag, searches return only IDs and scores, not text. Always enable content storage for RAG use cases.

2. **Using `index()` instead of `upsert()` for updates**: `index()` rebuilds the entire index. For adding new documents to an existing index, use `upsert()`.

3. **Not saving after indexing**: The in-memory index is lost on process exit. Call `embeddings.save()` after building or updating the index.

4. **SQL syntax for txtai**: txtai SQL is not standard SQL. The `similar()` function is txtai-specific. Test queries incrementally.

5. **Large model downloads**: First use downloads models from Hugging Face. This can take minutes and several GB of disk space. Pre-download models in CI/Docker builds.

6. **Mixing local and API LLMs**: The `LLM` pipeline class supports both local models and API-based models. Ensure you pass the correct model identifier format for your intended backend.

---

## References

- txtai documentation: https://neuml.github.io/txtai/
- txtai GitHub: https://github.com/neuml/txtai
- txtai examples: https://github.com/neuml/txtai/tree/master/examples
- NeuML blog: https://medium.com/neuml
