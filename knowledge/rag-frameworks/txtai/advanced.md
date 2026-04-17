# txtai -- Advanced: Graph, Agents, Edge Deployment, and SQLite Backend

## Overview

This guide covers txtai's advanced capabilities: knowledge graphs built from embeddings, LLM-driven agents with tool use, deployment on edge devices and resource-constrained environments, and deep configuration of the SQLite storage backend. These patterns build on the basics in `getting-started.md` and the architecture in `overview.md`.

---

## Knowledge Graphs

txtai can build a knowledge graph from indexed documents, enabling relationship-based queries alongside semantic search.

### Building a Graph

```python
from txtai import Embeddings

# Enable graph support
embeddings = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",
    content=True,
    graph={
        "approximate": False,     # exact vs approximate graph construction
        "minscore": 0.3,          # minimum similarity to create an edge
        "topics": {
            "algorithm": "louvain",  # community detection algorithm
            "terms": 4,              # number of topic terms per community
        },
    },
)

# Index documents -- graph is built automatically during indexing
documents = [
    (0, {"text": "Python is a programming language used for data science", "category": "language"}),
    (1, {"text": "NumPy provides numerical computing arrays for Python", "category": "library"}),
    (2, {"text": "Pandas is a data manipulation library built on NumPy", "category": "library"}),
    (3, {"text": "Scikit-learn provides machine learning algorithms for Python", "category": "ml"}),
    (4, {"text": "TensorFlow is a deep learning framework by Google", "category": "ml"}),
    (5, {"text": "PyTorch is a deep learning framework used in research", "category": "ml"}),
    (6, {"text": "FAISS is a library for efficient similarity search by Facebook", "category": "search"}),
    (7, {"text": "txtai uses FAISS for vector indexing and similarity search", "category": "search"}),
    (8, {"text": "Sentence transformers encode text into dense vector embeddings", "category": "nlp"}),
    (9, {"text": "BERT is a transformer model for natural language understanding", "category": "nlp"}),
]

embeddings.index(documents)
```

### Querying the Graph

```python
# Find related documents through graph traversal
# The graph connects documents with high semantic similarity

# Get the graph object
graph = embeddings.graph

# Find neighbors of a specific document
node_id = 0  # "Python is a programming language..."
neighbors = list(graph.backend.neighbors(node_id))
print(f"Neighbors of doc {node_id}:")
for neighbor in neighbors:
    result = embeddings.search(f"SELECT text FROM txtai WHERE id = {neighbor}")
    if result:
        print(f"  [{neighbor}] {result[0]['text'][:80]}")

# Community detection -- find topic clusters
communities = graph.topics
if communities:
    for topic_id, topic in communities.items():
        print(f"\nTopic {topic_id}: {topic.get('terms', [])}")
        for doc_id in topic.get("ids", [])[:3]:
            result = embeddings.search(f"SELECT text FROM txtai WHERE id = {doc_id}")
            if result:
                print(f"  - {result[0]['text'][:80]}")
```

### Graph-Enhanced RAG

```python
from txtai import Embeddings
from txtai.pipeline import LLM


def graph_enhanced_rag(
    embeddings: Embeddings,
    llm: LLM,
    question: str,
    k: int = 3,
    graph_hops: int = 1,
) -> dict:
    """RAG with graph expansion: retrieve documents + their graph neighbors."""
    # Step 1: Semantic search for seed documents
    results = embeddings.search(question, limit=k)
    seed_ids = set()
    for row in results:
        if isinstance(row, dict) and "id" in row:
            seed_ids.add(row["id"])

    # Step 2: Expand through graph neighbors
    expanded_ids = set(seed_ids)
    graph = embeddings.graph
    if graph and graph.backend:
        for doc_id in seed_ids:
            try:
                neighbors = list(graph.backend.neighbors(doc_id))
                for hop in range(graph_hops):
                    next_neighbors = set()
                    for n in neighbors:
                        next_neighbors.update(graph.backend.neighbors(n))
                    neighbors = list(next_neighbors - expanded_ids)
                expanded_ids.update(neighbors[:k])
            except Exception:
                pass

    # Step 3: Fetch all expanded documents
    all_texts = []
    for doc_id in expanded_ids:
        result = embeddings.search(f"SELECT text FROM txtai WHERE id = {doc_id}")
        if result:
            all_texts.append(result[0]["text"])

    # Step 4: Generate answer
    context = "\n".join(f"[{i+1}] {t}" for i, t in enumerate(all_texts))
    prompt = f"""Answer based on the context below. Cite passage numbers.

Context:
{context}

Question: {question}
Answer:"""

    answer = llm(prompt, maxlength=300)
    return {
        "answer": answer,
        "seed_docs": len(seed_ids),
        "expanded_docs": len(expanded_ids),
        "total_context": len(all_texts),
    }
```

---

## Agents

txtai Agents use LLMs to autonomously plan and execute tasks. Agents have access to tools (including the embeddings index) and decide when and how to use them.

### Basic Agent

```python
from txtai import Agent, Embeddings

# Create embeddings with indexed data
embeddings = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",
    content=True,
)
embeddings.index(documents)

# Create an agent with search capability
agent = Agent(
    embeddings=embeddings,
    llm="openai/gpt-4o-mini",   # or a local model path
    system="""You are a helpful research assistant.
Use the search tool to find relevant information before answering.
Always cite your sources.""",
)

# Ask a question -- the agent decides when to search
response = agent("What tools does txtai provide for NLP tasks?")
print(response)
```

### Agent with Custom Tools

```python
from txtai import Agent, Embeddings
import json


def calculate(expression: str) -> str:
    """Evaluate a mathematical expression.
    Args:
        expression: A mathematical expression to evaluate (e.g., '2 + 2', '100 * 0.15')
    Returns:
        The result of the calculation
    """
    try:
        result = eval(expression, {"__builtins__": {}}, {})
        return str(result)
    except Exception as e:
        return f"Error: {e}"


def get_current_date() -> str:
    """Get the current date and time.
    Returns:
        Current date in ISO format
    """
    from datetime import datetime
    return datetime.now().isoformat()


# Agent with multiple tools
agent = Agent(
    embeddings=embeddings,
    llm="openai/gpt-4o-mini",
    tools=[calculate, get_current_date],
    system="You are a helpful assistant with access to search, calculation, and date tools.",
)

# The agent uses tools as needed
response = agent("How many documents are about machine learning, "
                 "and what percentage is that of the total 10 documents?")
print(response)
```

### Multi-Step Agent Workflow

```python
from txtai import Agent, Embeddings
from txtai.pipeline import Summary


def summarize_text(text: str) -> str:
    """Summarize a piece of text.
    Args:
        text: The text to summarize
    Returns:
        A brief summary
    """
    summarizer = Summary()
    return summarizer(text, maxlength=50)


agent = Agent(
    embeddings=embeddings,
    llm="openai/gpt-4o-mini",
    tools=[summarize_text],
    system="""You are a research assistant. For each question:
1. Search the knowledge base for relevant information
2. Summarize the key findings
3. Provide a clear, concise answer""",
)

# Multi-step reasoning
response = agent(
    "Find information about deep learning frameworks and summarize "
    "the key differences between them."
)
print(response)
```

---

## Edge Deployment

txtai is designed to run on resource-constrained devices: Raspberry Pi, mobile devices, and offline environments.

### Minimal Memory Configuration

```python
from txtai import Embeddings

# Use a small model for edge deployment
embeddings = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",  # 80MB model
    content=True,
    backend="numpy",   # numpy backend uses less memory than FAISS
    quantize=True,     # quantize vectors to reduce memory
)

# Index documents
embeddings.index(documents)

# Save for deployment
embeddings.save("./edge_index")
```

### Optimized Model Selection for Edge

| Model | Size | Dimensions | Quality | Use Case |
|-------|------|------------|---------|----------|
| `all-MiniLM-L6-v2` | 80 MB | 384 | Good | General purpose, balanced |
| `all-MiniLM-L12-v2` | 120 MB | 384 | Better | Higher quality, moderate size |
| `paraphrase-MiniLM-L3-v2` | 60 MB | 384 | OK | Minimum size, acceptable quality |
| `BAAI/bge-small-en-v1.5` | 130 MB | 384 | Best | Best quality in small package |

### Offline Deployment

```python
import os
from pathlib import Path

# Step 1: Pre-download model on a machine with internet
from sentence_transformers import SentenceTransformer
model = SentenceTransformer("sentence-transformers/all-MiniLM-L6-v2")
model.save("./models/all-MiniLM-L6-v2")

# Step 2: Build and save the index
from txtai import Embeddings

embeddings = Embeddings(
    path="./models/all-MiniLM-L6-v2",  # local model path
    content=True,
)
embeddings.index(documents)
embeddings.save("./deployment/index")

# Step 3: Package for deployment
# Copy ./models/ and ./deployment/ to the target device

# Step 4: On the target device (no internet needed)
embeddings = Embeddings()
embeddings.load("./deployment/index")

results = embeddings.search("semantic search", limit=3)
```

### Docker for Edge

```dockerfile
# Dockerfile.edge
FROM python:3.11-slim

WORKDIR /app

# Install minimal dependencies
RUN pip install --no-cache-dir txtai

# Copy pre-built model and index
COPY models/ /app/models/
COPY index/ /app/index/
COPY server.py /app/

EXPOSE 8000

CMD ["python", "server.py"]
```

```python
# server.py
from txtai import Embeddings
from http.server import HTTPServer, BaseHTTPRequestHandler
import json
import urllib.parse

embeddings = Embeddings()
embeddings.load("/app/index")


class SearchHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        parsed = urllib.parse.urlparse(self.path)
        params = urllib.parse.parse_qs(parsed.query)

        if parsed.path == "/search":
            query = params.get("query", [""])[0]
            limit = int(params.get("limit", ["5"])[0])

            results = embeddings.search(query, limit=limit)
            response = [{"score": s, "text": t} for s, t in results]

            self.send_response(200)
            self.send_header("Content-Type", "application/json")
            self.end_headers()
            self.wfile.write(json.dumps(response).encode())
        elif parsed.path == "/health":
            self.send_response(200)
            self.send_header("Content-Type", "application/json")
            self.end_headers()
            self.wfile.write(json.dumps({"status": "ok"}).encode())
        else:
            self.send_response(404)
            self.end_headers()


server = HTTPServer(("0.0.0.0", 8000), SearchHandler)
print("Server running on port 8000")
server.serve_forever()
```

---

## SQLite Backend Deep Dive

txtai uses SQLite as its default content store. Understanding and tuning SQLite is important for production deployments.

### How txtai Uses SQLite

```
Embeddings index:
  - Vector store (FAISS/numpy/annoy): stores embedding vectors
  - SQLite database: stores document content, metadata, and search scoring data
  - Both are saved together when calling embeddings.save()

File structure after save:
  index/
    config              # JSON configuration
    documents           # SQLite database (content + metadata)
    embeddings          # FAISS/numpy index (vectors)
    scoring/            # BM25 scoring data (if hybrid enabled)
```

### Custom SQLite Configuration

```python
from txtai import Embeddings

embeddings = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",
    content=True,
    # SQLite configuration
    sqlite={
        "wal": True,          # Write-Ahead Logging (better concurrent reads)
        "journal": "wal",     # journal mode
        "cache_size": 10000,  # pages in cache (default: ~2000)
    },
)
```

### Direct SQLite Access

```python
import sqlite3
from pathlib import Path

# After saving an index, you can inspect the SQLite database directly
embeddings.save("./my_index")

# Open the SQLite database
db_path = Path("./my_index/documents")
conn = sqlite3.connect(str(db_path))
cursor = conn.cursor()

# List tables
cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")
tables = cursor.fetchall()
print(f"Tables: {[t[0] for t in tables]}")

# Count documents
cursor.execute("SELECT COUNT(*) FROM sections")
count = cursor.fetchone()[0]
print(f"Documents: {count}")

# Sample data
cursor.execute("SELECT id, text FROM sections LIMIT 3")
for row in cursor.fetchall():
    print(f"  [{row[0]}] {row[1][:80]}...")

conn.close()
```

### Backup and Migration

```python
import shutil
from datetime import datetime


def backup_index(index_path: str, backup_dir: str = "./backups"):
    """Create a timestamped backup of a txtai index."""
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    backup_path = f"{backup_dir}/index_{timestamp}"
    shutil.copytree(index_path, backup_path)
    print(f"Backup created: {backup_path}")
    return backup_path


def restore_index(backup_path: str, index_path: str):
    """Restore a txtai index from backup."""
    if Path(index_path).exists():
        shutil.rmtree(index_path)
    shutil.copytree(backup_path, index_path)
    print(f"Restored index from: {backup_path}")


# Usage
backup_path = backup_index("./my_index")
# ... later ...
# restore_index(backup_path, "./my_index")
```

---

## Advanced Embeddings Configuration

### Multiple Vector Backends

```python
# FAISS (default, best for medium to large collections)
embeddings_faiss = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",
    content=True,
    backend="faiss",
)

# Hnswlib (faster search, more memory)
embeddings_hnsw = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",
    content=True,
    backend="hnsw",
    hnsw={
        "efconstruction": 200,
        "m": 16,
        "efsearch": 100,
    },
)

# Annoy (fast, read-only after build)
embeddings_annoy = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",
    content=True,
    backend="annoy",
    annoy={
        "ntrees": 100,
        "searchk": 1000,
    },
)

# NumPy (smallest memory footprint, brute-force search)
embeddings_numpy = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",
    content=True,
    backend="numpy",
)
```

### External Vector Databases

```python
# Connect to an external vector store
embeddings = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",
    content=True,
    # External backend configuration
    backend="custom",
    # Supported external backends: Elasticsearch, pgvector, etc.
)
```

### Quantization for Smaller Indexes

```python
embeddings = Embeddings(
    path="sentence-transformers/all-MiniLM-L6-v2",
    content=True,
    quantize=True,    # int8 quantization of vectors
    backend="faiss",
    faiss={
        "quantize": True,
        "components": "IVF100,PQ32",  # FAISS IVF + product quantization
        "nprobe": 10,
    },
)

# Quantized indexes are 4-8x smaller with minimal quality loss
# Good for edge deployment where storage is constrained
```

---

## Testing

```python
import pytest
from txtai import Embeddings


@pytest.fixture
def embeddings():
    """Create a test embeddings instance."""
    emb = Embeddings(
        path="sentence-transformers/all-MiniLM-L6-v2",
        content=True,
    )
    test_data = [
        (0, {"text": "Python programming language", "category": "lang"}),
        (1, {"text": "JavaScript web development", "category": "lang"}),
        (2, {"text": "Machine learning algorithms", "category": "ml"}),
    ]
    emb.index(test_data)
    return emb


def test_search_returns_results(embeddings):
    results = embeddings.search("programming", limit=2)
    assert len(results) > 0


def test_sql_search(embeddings):
    results = embeddings.search(
        "SELECT text FROM txtai WHERE similar('programming') AND category = 'lang' LIMIT 5"
    )
    assert len(results) > 0
    assert all("programming" in r["text"].lower() or "language" in r["text"].lower()
               for r in results[:1])


def test_count(embeddings):
    assert embeddings.count() == 3


def test_upsert(embeddings):
    embeddings.upsert([(3, {"text": "New document", "category": "new"})])
    assert embeddings.count() == 4


def test_delete(embeddings):
    embeddings.delete([0])
    assert embeddings.count() == 2


def test_save_and_load(embeddings, tmp_path):
    save_path = str(tmp_path / "test_index")
    embeddings.save(save_path)

    loaded = Embeddings()
    loaded.load(save_path)

    assert loaded.count() == embeddings.count()
    results = loaded.search("programming", limit=1)
    assert len(results) > 0
```

---

## Common Pitfalls

1. **Graph construction on small datasets**: Graph community detection needs at least 20-30 documents to produce meaningful clusters. With fewer documents, communities may be a single giant cluster.

2. **Agent tool descriptions**: Agents use function docstrings to understand tools. Poor or missing docstrings cause agents to misuse or ignore tools. Always write clear, specific docstrings.

3. **Edge deployment model size**: The model is loaded entirely into RAM. On devices with 1-2 GB RAM, use the smallest model possible (`paraphrase-MiniLM-L3-v2` at 60 MB).

4. **SQLite WAL mode on network drives**: WAL mode does not work correctly on network-mounted filesystems (NFS, SMB). Use standard journal mode for shared storage.

5. **Quantization + FAISS IVF training**: IVF indexes need to be trained on a sample of vectors. With very small datasets (<1000), FAISS IVF may underperform simple flat indexes. Use flat search for small collections.

6. **Agent API key requirements**: Unlike basic embeddings (which run locally), agents typically need an LLM API key (OpenAI, Anthropic) for reasoning. Local agent LLMs are possible but quality is lower.

---

## References

- txtai documentation: https://neuml.github.io/txtai/
- txtai GitHub: https://github.com/neuml/txtai
- txtai graph: https://neuml.github.io/txtai/embeddings/graph/
- txtai agents: https://neuml.github.io/txtai/agent/
- NeuML blog: https://medium.com/neuml
