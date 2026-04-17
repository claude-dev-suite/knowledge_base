# txtai -- Lightweight Embeddings and Pipelines

## Overview

txtai is a Python framework for building AI-powered semantic search, LLM orchestration, and language model workflows. Created by NeuML, txtai emphasizes simplicity: a complete semantic search index runs in under 10 lines of code, and the same framework scales from laptop prototypes to production deployments. Unlike heavier frameworks (LangChain, LlamaIndex), txtai has minimal dependencies and can run entirely locally without API keys using built-in Hugging Face model support.

txtai's core abstraction is the Embeddings index -- a vector database that runs in-process backed by SQLite, FAISS, or external stores. Around this core, txtai provides Pipelines (text generation, summarization, transcription, translation), Workflows (DAG-based multi-step processing), Agents (autonomous LLM-driven task execution), and a Graph database for knowledge representation.

This guide covers txtai's architecture, the Embeddings API, pipeline system, and when txtai is the right choice.

---

## Architecture

```
                    +------------------+
                    |     Agents       |
                    | (LLM-driven      |
                    |  autonomous tasks)|
                    +--------+---------+
                             |
                    +--------v---------+
                    |    Workflows      |
                    | (DAG processing   |
                    |  chains)          |
                    +--------+---------+
                             |
              +--------------+---------------+
              |              |               |
    +---------v---+  +-------v------+  +-----v-------+
    | Embeddings  |  |  Pipelines   |  |   Graph     |
    | (semantic   |  | (generation, |  | (knowledge  |
    |  search)    |  |  summarize,  |  |  graph)     |
    +------+------+  |  transcribe) |  +-------------+
           |         +--------------+
    +------v------+
    |  Storage    |
    | (SQLite +   |
    |  FAISS/     |
    |  Annoy/     |
    |  external)  |
    +-------------+
```

### Core Components

| Component | Purpose |
|-----------|---------|
| **Embeddings** | In-process vector database with semantic search, SQL support, and hybrid search |
| **Pipelines** | Single-task processors: generation, summarization, translation, transcription, extraction |
| **Workflows** | Multi-step DAGs that connect pipelines and custom functions |
| **Agents** | LLM-driven autonomous task execution with tool use |
| **Graph** | Knowledge graph built from embeddings relationships |

---

## Embeddings

The Embeddings class is txtai's core. It creates, stores, and queries a semantic search index:

### Basic Usage

```python
from txtai import Embeddings

# Create an embeddings index
embeddings = Embeddings()

# Index data (list of strings or dicts)
data = [
    "txtai is a lightweight semantic search framework",
    "It supports FAISS, Annoy, and Hnswlib for vector indexing",
    "Pipelines provide text generation, summarization, and translation",
    "Workflows connect multiple pipelines into processing DAGs",
    "txtai can run entirely locally without API keys",
    "The Embeddings class is the core abstraction for semantic search",
    "SQLite is used as the default content store",
    "Agents use LLMs to autonomously execute tasks with tools",
]

embeddings.index(data)

# Search
results = embeddings.search("How does txtai store vectors?", limit=3)
for score, text in results:
    print(f"  [{score:.4f}] {text}")
```

### Embeddings with Configuration

```python
from txtai import Embeddings

# Full configuration
embeddings = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",  # embedding model
    content=True,        # store content alongside vectors (uses SQLite)
    backend="faiss",     # vector backend: faiss, annoy, hnsw, numpy
    hybrid=True,         # enable hybrid search (vector + BM25)
    scoring={            # BM25 scoring configuration
        "method": "bm25",
        "terms": True,
    },
)

# Index with IDs and metadata
data = [
    {"id": "doc-001", "text": "txtai supports semantic search", "category": "search"},
    {"id": "doc-002", "text": "Pipelines run NLP tasks", "category": "pipeline"},
    {"id": "doc-003", "text": "Workflows chain operations", "category": "workflow"},
]

embeddings.index([(d["id"], d) for d in data])

# Search with SQL-like filtering
results = embeddings.search(
    "SELECT id, text, score FROM txtai WHERE similar('search framework') AND category = 'search'",
    limit=5,
)
for row in results:
    print(f"  [{row['score']:.4f}] {row['id']}: {row['text']}")
```

### SQL Support

txtai's Embeddings class supports SQL queries that combine semantic search with metadata filtering:

```python
embeddings = Embeddings(path="sentence-transformers/all-MiniLM-L6-v2", content=True)

# Index documents with metadata
data = [
    (0, {"text": "Python is great for data science", "language": "python", "year": 2024}),
    (1, {"text": "JavaScript powers the web", "language": "javascript", "year": 2024}),
    (2, {"text": "Rust provides memory safety", "language": "rust", "year": 2024}),
    (3, {"text": "Python web frameworks include Django and Flask", "language": "python", "year": 2023}),
]

embeddings.index(data)

# Pure semantic search
results = embeddings.search("web development", limit=3)

# Semantic search + metadata filter
results = embeddings.search(
    "SELECT text, score FROM txtai WHERE similar('web development') AND language = 'python'",
    limit=3,
)

# Aggregation queries
results = embeddings.search(
    "SELECT language, COUNT(*) as count FROM txtai GROUP BY language"
)

# Combined semantic + keyword
results = embeddings.search(
    "SELECT text, score FROM txtai WHERE similar('systems programming') AND text LIKE '%memory%'",
    limit=5,
)
```

### Persistence

```python
# Save to disk
embeddings.save("./my_index")

# Load from disk
embeddings = Embeddings()
embeddings.load("./my_index")

# Upsert (update or insert)
new_data = [
    (4, {"text": "Go is designed for concurrent systems", "language": "go", "year": 2024}),
]
embeddings.upsert(new_data)

# Delete
embeddings.delete([0, 1])  # delete by IDs

# Count
print(f"Documents in index: {embeddings.count()}")
```

### Hybrid Search

txtai supports combining vector search with BM25 keyword search:

```python
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

# Hybrid search automatically combines vector similarity and BM25 scores
results = embeddings.search("Python web frameworks", limit=5)

# The results blend semantic understanding with keyword matching
# "Python web frameworks include Django and Flask" ranks highest because
# it matches both semantically and through keyword overlap
```

---

## Pipelines

Pipelines are single-task processors built on Hugging Face models:

### Text Generation (RAG)

```python
from txtai import Embeddings, Pipelines
from txtai.pipeline import LLM

# Create an LLM pipeline
llm = LLM("microsoft/Phi-3-mini-4k-instruct")

# Generate text
response = llm(
    "Explain what txtai is in one paragraph.",
    maxlength=200,
)
print(response)

# RAG: combine embeddings search with generation
embeddings = Embeddings(path="sentence-transformers/all-MiniLM-L6-v2", content=True)
embeddings.index(data)

# Retrieve context
results = embeddings.search("How does txtai work?", limit=3)
context = "\n".join(text for _, text in results)

# Generate answer with context
prompt = f"""Based on the following context, answer the question.

Context:
{context}

Question: How does txtai work?
Answer:"""

answer = llm(prompt, maxlength=300)
print(answer)
```

### Summarization

```python
from txtai.pipeline import Summary

summary = Summary()

text = """
txtai is an open-source embeddings database for semantic search,
LLM orchestration and language model workflows. It provides
in-process vector search backed by SQLite and FAISS, along with
pipelines for text generation, summarization, translation, and
transcription. Workflows connect these pipelines into multi-step
processing chains. The framework is designed to be lightweight
and can run entirely on a local machine without API keys.
"""

result = summary(text, maxlength=50)
print(result)
```

### Translation

```python
from txtai.pipeline import Translation

translate = Translation()

# English to French
result = translate("txtai is a semantic search framework", "fr")
print(result)  # "txtai est un cadre de recherche semantique"

# Auto-detect source language
result = translate("txtai es un marco de busqueda semantica", "en")
print(result)  # "txtai is a semantic search framework"
```

### Transcription

```python
from txtai.pipeline import Transcription

transcribe = Transcription()

# Transcribe audio file
text = transcribe("meeting_recording.wav")
print(text)

# With configuration
transcribe = Transcription(path="openai/whisper-large-v3")
text = transcribe("recording.mp3")
```

### Extraction (NER, QA)

```python
from txtai.pipeline import Extractor

# Question-answering extraction
extractor = Extractor(embeddings=embeddings, path="distilbert-base-cased-distilled-squad")

# Ask questions about the indexed data
answers = extractor([
    {"query": "What is txtai?", "question": "What is txtai?"},
    {"query": "storage backend", "question": "What storage does txtai use?"},
])

for answer in answers:
    print(f"Q: {answer['question']}")
    print(f"A: {answer['answer']}")
```

---

## Configuration via YAML

txtai can be configured entirely through YAML:

```yaml
# config.yml
embeddings:
  path: sentence-transformers/all-MiniLM-L6-v2
  content: true
  backend: faiss
  hybrid: true
  scoring:
    method: bm25
    terms: true

# Optional: enable API server
api:
  host: 0.0.0.0
  port: 8000
```

```python
from txtai import Application

# Load from YAML config
app = Application("config.yml")

# Use the configured embeddings
app.add(data)
results = app.search("semantic search", limit=5)
```

### API Server

txtai includes a built-in FastAPI server:

```bash
# Start the API server
python -m txtai.api config.yml

# Or with uvicorn directly
uvicorn txtai.api:app --host 0.0.0.0 --port 8000
```

```bash
# Index documents
curl -X POST http://localhost:8000/add \
  -H "Content-Type: application/json" \
  -d '[{"id": "0", "text": "txtai semantic search"}]'

# Search
curl "http://localhost:8000/search?query=search+framework&limit=5"
```

---

## When to Use txtai

| Scenario | txtai | LangChain/LlamaIndex |
|----------|-------|---------------------|
| Lightweight local prototype | Best fit | Heavier setup |
| No API keys needed | Built-in local models | Usually need API keys |
| SQLite-backed persistence | Native | Requires integration |
| Edge/embedded deployment | Designed for it | Not practical |
| Complex agent workflows | Supported (simpler) | More mature |
| Many LLM integrations | Fewer | Many more |
| Production vector DB | FAISS/in-process | External DBs (Pinecone, Qdrant) |
| SQL over semantic search | Native | Not available |

---

## Common Pitfalls

1. **Not setting `content=True`**: Without `content=True`, the Embeddings index only stores vectors, not the original text. Searches return IDs and scores but not text content.

2. **Using the wrong embedding model**: The default model works for English text. For multilingual content, use `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`.

3. **Forgetting to call `embeddings.save()`**: The in-memory index is lost when the process exits. Always persist the index after building it.

4. **Index vs. upsert confusion**: `index()` rebuilds the entire index from scratch. `upsert()` adds or updates individual records. Use `index()` for initial build, `upsert()` for incremental updates.

5. **SQL syntax errors**: txtai's SQL support uses its own parser. Not all standard SQL is supported. Test queries incrementally.

6. **Model download on first use**: Hugging Face models are downloaded on first use. First execution is slow. Pre-download models in Docker images or CI pipelines.

---

## References

- txtai documentation: https://neuml.github.io/txtai/
- txtai GitHub: https://github.com/neuml/txtai
- NeuML blog: https://medium.com/neuml
- txtai examples: https://github.com/neuml/txtai/tree/master/examples
