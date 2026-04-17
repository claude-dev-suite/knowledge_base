# R2R -- Architecture, Auto-KG, and Hybrid Search

## Overview

R2R (RAG to Riches) is an open-source RAG engine designed for production deployment. Unlike library-based frameworks (LangChain, LlamaIndex) where you compose retrieval pipelines in code, R2R is a complete server with a REST API, CLI, and Python/TypeScript SDKs. It handles document ingestion, chunking, embedding, vector storage, knowledge graph construction, hybrid search, and LLM generation out of the box.

R2R's key differentiators are automatic knowledge graph construction from ingested documents, built-in hybrid search (vector + keyword + knowledge graph), multi-tenant user management, and agentic RAG workflows. It runs as a Docker service backed by PostgreSQL (with pgvector) and Hatchet for workflow orchestration.

This guide covers R2R's architecture, the automatic knowledge graph pipeline, hybrid search mechanics, and how R2R compares to other RAG frameworks.

---

## Architecture

R2R runs as a set of services orchestrated by Docker Compose:

```
                         +------------------+
                         |   R2R REST API   |
                         |   (FastAPI)      |
                         +--------+---------+
                                  |
                    +-------------+-------------+
                    |             |             |
              +-----v---+  +-----v---+  +-----v----+
              | Ingest   |  | Search  |  | RAG      |
              | Pipeline |  | Engine  |  | Engine   |
              +-----+---+  +----+----+  +-----+----+
                    |            |              |
              +-----v---+  +----v----+   +-----v----+
              | Chunker  |  | Vector  |   |  LLM     |
              | Embedder |  | + BM25  |   | Provider |
              | KG Build |  | + KG    |   |          |
              +-----+---+  +----+----+   +----------+
                    |            |
              +-----v------------v----+
              |  PostgreSQL + pgvector |
              |  (documents, vectors,  |
              |   knowledge graph,     |
              |   user management)     |
              +------------------------+
```

### Core Services

| Service | Purpose |
|---------|---------|
| **R2R API** | FastAPI server exposing REST endpoints for ingestion, search, RAG, and management |
| **Ingestion Pipeline** | Parses documents, chunks text, generates embeddings, builds knowledge graph entities |
| **Search Engine** | Hybrid search combining vector similarity, BM25 keyword matching, and knowledge graph traversal |
| **RAG Engine** | Orchestrates retrieval + generation with configurable prompts and model selection |
| **PostgreSQL** | Stores documents, embeddings (pgvector), knowledge graph triples, user data, and conversation history |
| **Hatchet** | Workflow orchestration for long-running ingestion and KG construction tasks |

### Document Lifecycle

```
Upload file/text -> Parse -> Chunk -> Embed -> Store vectors
                                  \-> Extract entities -> Build KG triples -> Store in graph
```

1. **Parsing**: R2R extracts text from PDFs, DOCX, HTML, Markdown, CSV, JSON, images (OCR), and audio (transcription).
2. **Chunking**: Text is split into overlapping chunks using configurable strategies (recursive, semantic, or fixed-size).
3. **Embedding**: Chunks are embedded using OpenAI, local models, or any OpenAI-compatible endpoint.
4. **Knowledge Graph**: Entities and relationships are extracted using an LLM and stored as triples in PostgreSQL.
5. **Storage**: Vectors go to pgvector, chunks to PostgreSQL, KG triples to a graph table.

---

## Automatic Knowledge Graph (Auto-KG)

R2R's most distinctive feature is automatic knowledge graph construction. When you ingest documents, R2R optionally extracts entities and relationships using an LLM, creating a traversable knowledge graph alongside the vector index.

### How Auto-KG Works

```python
# When ingesting documents with KG enabled:
# 1. Each chunk is sent to an LLM with a structured extraction prompt
# 2. The LLM extracts (subject, predicate, object) triples
# 3. Triples are stored in PostgreSQL and linked to source chunks

# Example: from the text
# "PostgreSQL supports vector similarity search through the pgvector extension."

# Extracted triples:
# (PostgreSQL, supports, vector similarity search)
# (pgvector, is_extension_of, PostgreSQL)
# (pgvector, enables, vector similarity search)
```

### KG-Enhanced Search

When you search with KG enabled, R2R:

1. Retrieves relevant entities from the query
2. Traverses the knowledge graph for related entities
3. Fetches chunks associated with those entities
4. Merges with vector search results
5. Generates an answer using all retrieved context

```
Query: "How does pgvector work with PostgreSQL?"

Vector search results:
  [1] "pgvector adds vector data types and similarity search to PostgreSQL..."
  [2] "Install pgvector: CREATE EXTENSION vector..."

KG traversal:
  pgvector --is_extension_of--> PostgreSQL
  pgvector --enables--> vector similarity search
  PostgreSQL --supports--> HNSW indexing

KG-associated chunks:
  [3] "HNSW indexes in pgvector provide approximate nearest neighbor search..."

Combined context -> LLM -> Answer
```

### Configuring KG Extraction

```python
from r2r import R2RClient

client = R2RClient("http://localhost:7272")

# Ingest with KG extraction enabled
response = client.documents.create(
    file_path="database_docs.pdf",
    metadata={"source": "official-docs"},
    ingestion_config={
        "chunking_config": {
            "strategy": "recursive",
            "chunk_size": 512,
            "chunk_overlap": 50,
        },
    },
    run_with_orchestration=True,  # async processing via Hatchet
)

# Trigger KG construction for all documents
client.graphs.build(
    collection_id="default",
    settings={
        "extraction_model": "gpt-4o-mini",
        "max_entities_per_chunk": 10,
        "max_relations_per_chunk": 15,
    },
)
```

---

## Hybrid Search

R2R combines three retrieval strategies:

### Vector Search

Semantic similarity using pgvector:

```python
# Vector-only search
results = client.retrieval.search(
    query="How does pgvector handle HNSW indexes?",
    search_settings={
        "use_vector_search": True,
        "use_fulltext_search": False,
        "use_kg_search": False,
        "search_limit": 10,
        "ef_search": 100,  # HNSW search parameter
    },
)
```

### Full-Text Search (BM25)

Keyword matching using PostgreSQL full-text search:

```python
# BM25-only search
results = client.retrieval.search(
    query="pgvector HNSW index configuration",
    search_settings={
        "use_vector_search": False,
        "use_fulltext_search": True,
        "use_kg_search": False,
        "search_limit": 10,
    },
)
```

### Knowledge Graph Search

Entity and relationship traversal:

```python
# KG-only search
results = client.retrieval.search(
    query="What databases support vector search?",
    search_settings={
        "use_vector_search": False,
        "use_fulltext_search": False,
        "use_kg_search": True,
        "kg_search_settings": {
            "kg_search_type": "local",  # or "global" for community-level
        },
    },
)
```

### Hybrid (All Three Combined)

```python
# Full hybrid search (recommended for production)
results = client.retrieval.search(
    query="How does pgvector handle HNSW indexes?",
    search_settings={
        "use_vector_search": True,
        "use_fulltext_search": True,
        "use_kg_search": True,
        "search_limit": 10,
        "hybrid_settings": {
            "full_text_weight": 1.0,
            "semantic_weight": 5.0,
            "full_text_limit": 200,
            "rrf_k": 50,
        },
    },
)

for result in results["results"]["chunk_search_results"]:
    print(f"Score: {result['score']:.4f}")
    print(f"Text: {result['text'][:200]}...")
    print(f"Metadata: {result['metadata']}")
    print("---")
```

---

## Multi-Tenant Architecture

R2R has built-in user and collection management:

```python
# Create users
client.users.create(email="team-a@company.com", password="secure")
client.users.create(email="team-b@company.com", password="secure")

# Create collections (tenants)
team_a_collection = client.collections.create(
    name="Team A Knowledge Base",
    description="Internal documentation for Team A",
)

team_b_collection = client.collections.create(
    name="Team B Knowledge Base",
    description="Internal documentation for Team B",
)

# Assign documents to collections
client.collections.add_document(
    collection_id=team_a_collection["id"],
    document_id=doc_id,
)

# Search within a specific collection
results = client.retrieval.search(
    query="deployment procedures",
    search_settings={
        "filters": {
            "collection_ids": [team_a_collection["id"]],
        },
    },
)
```

---

## RAG Generation

### Basic RAG

```python
# RAG: retrieve + generate in one call
response = client.retrieval.rag(
    query="How do I configure HNSW indexes in pgvector?",
    rag_generation_config={
        "model": "openai/gpt-4o-mini",
        "temperature": 0.0,
        "max_tokens": 500,
    },
    search_settings={
        "use_vector_search": True,
        "use_fulltext_search": True,
        "search_limit": 5,
    },
)

print(response["results"]["completion"]["choices"][0]["message"]["content"])
```

### Streaming RAG

```python
# Streaming response
for chunk in client.retrieval.rag(
    query="Explain pgvector indexing strategies",
    rag_generation_config={
        "model": "openai/gpt-4o-mini",
        "stream": True,
    },
):
    if chunk.get("choices"):
        print(chunk["choices"][0].get("delta", {}).get("content", ""), end="")
```

### Agentic RAG

R2R supports agentic workflows where the agent decides when and what to retrieve:

```python
# Agent mode: the LLM decides when to search
response = client.retrieval.agent(
    message={
        "role": "user",
        "content": "Compare HNSW and IVFFlat indexes in pgvector. "
                   "Which is better for 1M vectors?",
    },
    rag_generation_config={
        "model": "openai/gpt-4o",
        "temperature": 0.0,
    },
    search_settings={
        "use_vector_search": True,
        "use_fulltext_search": True,
    },
)
```

---

## Configuration

R2R is configured via `r2r.toml` or environment variables:

```toml
# r2r.toml
[database]
provider = "postgres"

[embedding]
provider = "openai"
model = "text-embedding-3-small"
dimension = 512
batch_size = 128

[chunking]
strategy = "recursive"
chunk_size = 512
chunk_overlap = 50

[completion]
provider = "openai"
model = "gpt-4o-mini"

[kg]
provider = "postgres"
extraction_model = "gpt-4o-mini"
```

### Environment Variables

```bash
# Required
export OPENAI_API_KEY="sk-..."
export R2R_POSTGRES_HOST="localhost"
export R2R_POSTGRES_PORT="5432"
export R2R_POSTGRES_DBNAME="r2r"
export R2R_POSTGRES_USER="r2r"
export R2R_POSTGRES_PASSWORD="secure-password"

# Optional
export R2R_EMBEDDING_MODEL="text-embedding-3-small"
export R2R_EMBEDDING_DIMENSION="512"
export R2R_LLM_MODEL="gpt-4o-mini"
```

---

## When to Use R2R

| Scenario | R2R | Library frameworks (LangChain, LlamaIndex) |
|----------|-----|---------------------------------------------|
| Need a complete RAG server | Best fit | Must build your own server |
| Knowledge graph from documents | Built-in Auto-KG | Must implement separately |
| Multi-tenant isolation | Built-in | Must implement separately |
| Custom retrieval logic | Limited (plugins) | Full flexibility |
| Quick prototyping | Moderate (needs Docker) | Quick (pip install) |
| Agentic RAG workflows | Built-in | Possible with more code |
| Production deployment | Designed for it | Requires more infrastructure |

---

## Common Pitfalls

1. **Not running Docker services**: R2R requires PostgreSQL and optionally Hatchet. Trying to use the Python client without the server running gives connection errors.

2. **Skipping KG construction**: Auto-KG is not enabled by default on ingestion. You must call `graphs.build()` after ingesting documents to construct the knowledge graph.

3. **Ignoring collection scoping**: Without collection filters, searches return results across all documents for all users. Always scope searches to the appropriate collection in multi-tenant setups.

4. **Using large embedding dimensions**: `text-embedding-3-small` at full 1536 dimensions increases storage and slows search. Use `dimension=512` for a good quality/performance tradeoff.

5. **Not monitoring ingestion jobs**: Large document ingestion runs asynchronously via Hatchet. Check job status with `client.documents.list()` and monitor for failures.

---

## References

- R2R documentation: https://r2r-docs.sciphi.ai/
- R2R GitHub: https://github.com/SciPhi-AI/R2R
- R2R Python SDK: https://pypi.org/project/r2r/
- SciPhi AI blog: https://www.sciphi.ai/blog
