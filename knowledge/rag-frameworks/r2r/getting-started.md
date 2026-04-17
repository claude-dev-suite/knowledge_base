# R2R -- Getting Started: CLI, Python Client, and Docker Deployment

## Overview

This guide walks through deploying R2R, ingesting documents, running searches, and building RAG queries using the CLI, Python SDK, and Docker. By the end you will have a running R2R instance with documents indexed and searchable through hybrid retrieval.

---

## Installation Options

### Option 1: Docker (Recommended for Production)

```bash
# Install the R2R CLI
pip install r2r

# Deploy R2R with Docker Compose (includes PostgreSQL, pgvector, Hatchet)
r2r serve --docker

# This starts:
# - R2R API server on port 7272
# - PostgreSQL + pgvector on port 5432
# - Hatchet orchestrator on port 7077
```

### Option 2: Docker Compose Manually

```bash
# Clone the repository
git clone https://github.com/SciPhi-AI/R2R.git
cd R2R

# Configure environment
cp .env.example .env
# Edit .env with your API keys

# Start all services
docker compose up -d

# Verify services are running
docker compose ps
curl http://localhost:7272/v3/health
```

### Option 3: Python Package (Development Only)

```bash
pip install r2r

# Start R2R without Docker (requires local PostgreSQL with pgvector)
r2r serve --host 0.0.0.0 --port 7272
```

### Environment Setup

```bash
# Required: LLM provider API key
export OPENAI_API_KEY="sk-..."

# Optional: for Anthropic models
export ANTHROPIC_API_KEY="sk-ant-..."

# Database (auto-configured in Docker mode)
export R2R_POSTGRES_HOST="localhost"
export R2R_POSTGRES_PORT="5432"
export R2R_POSTGRES_DBNAME="r2r"
export R2R_POSTGRES_USER="r2r"
export R2R_POSTGRES_PASSWORD="your-password"
```

### Verify Installation

```bash
# Check R2R server health
r2r health

# Or via curl
curl http://localhost:7272/v3/health

# Expected response:
# {"results": {"response": "ok"}}
```

---

## Part 1: Using the CLI

### Ingesting Documents

```bash
# Ingest a single file
r2r documents create --file-path ./docs/architecture.pdf

# Ingest a directory of files
r2r documents create --file-path ./docs/

# Ingest with metadata
r2r documents create \
  --file-path ./docs/api-guide.md \
  --metadata '{"source": "internal", "team": "engineering", "version": "2.0"}'

# Ingest raw text
r2r documents create \
  --raw-text "R2R is a production RAG engine with built-in knowledge graph support." \
  --metadata '{"source": "manual-entry"}'

# List ingested documents
r2r documents list

# Check document status
r2r documents retrieve --id <document-id>
```

### Searching

```bash
# Vector search
r2r retrieval search --query "How does hybrid search work?"

# Search with filters
r2r retrieval search \
  --query "deployment guide" \
  --search-settings '{"filters": {"metadata.team": {"$eq": "engineering"}}}'

# Search with specific settings
r2r retrieval search \
  --query "pgvector indexing" \
  --search-settings '{
    "use_vector_search": true,
    "use_fulltext_search": true,
    "search_limit": 10
  }'
```

### RAG Queries

```bash
# Basic RAG query
r2r retrieval rag --query "How do I configure HNSW indexes?"

# RAG with model selection
r2r retrieval rag \
  --query "Compare vector search strategies" \
  --rag-generation-config '{"model": "openai/gpt-4o", "temperature": 0}'

# Streaming RAG
r2r retrieval rag \
  --query "Explain the R2R architecture" \
  --stream
```

### Knowledge Graph Operations

```bash
# Build knowledge graph from ingested documents
r2r graphs build --collection-id default

# Check KG status
r2r graphs status --collection-id default

# Search with KG enabled
r2r retrieval search \
  --query "What entities are related to PostgreSQL?" \
  --search-settings '{"use_kg_search": true}'
```

### User Management

```bash
# Create a user
r2r users create --email "user@example.com" --password "secure-pass"

# List users
r2r users list

# Create a collection
r2r collections create --name "Engineering Docs"

# Add document to collection
r2r collections add-document --collection-id <id> --document-id <doc-id>
```

---

## Part 2: Python SDK

### Client Setup

```python
from r2r import R2RClient

# Connect to local R2R server
client = R2RClient("http://localhost:7272")

# Or with authentication
client = R2RClient(
    base_url="http://localhost:7272",
)
client.users.login(email="admin@example.com", password="admin-password")

# Health check
health = client.system.health()
print(f"Server status: {health['results']['response']}")
```

### Document Ingestion

```python
from r2r import R2RClient
from pathlib import Path

client = R2RClient("http://localhost:7272")

# Ingest a single file
result = client.documents.create(
    file_path="./docs/architecture.pdf",
    metadata={
        "source": "internal-docs",
        "department": "engineering",
        "version": "3.0",
    },
    run_with_orchestration=True,  # async processing
)
doc_id = result["results"]["document_id"]
print(f"Document ID: {doc_id}")

# Ingest raw text
result = client.documents.create(
    raw_text="R2R supports automatic knowledge graph construction from documents.",
    metadata={"source": "manual", "topic": "features"},
)

# Ingest multiple files
docs_dir = Path("./docs")
for file_path in docs_dir.glob("**/*.md"):
    result = client.documents.create(
        file_path=str(file_path),
        metadata={
            "source": "docs",
            "filename": file_path.name,
            "category": file_path.parent.name,
        },
    )
    print(f"Ingested: {file_path.name} -> {result['results']['document_id']}")

# List all documents
docs = client.documents.list()
print(f"Total documents: {len(docs['results'])}")
```

### Chunking Configuration

```python
# Custom chunking strategy
result = client.documents.create(
    file_path="./docs/long-document.pdf",
    ingestion_config={
        "chunking_config": {
            "strategy": "recursive",      # recursive, semantic, or fixed
            "chunk_size": 512,            # tokens per chunk
            "chunk_overlap": 50,          # overlap between chunks
        },
        "embedding_config": {
            "model": "text-embedding-3-small",
            "dimension": 512,
        },
    },
)
```

### Search

```python
# Basic search
results = client.retrieval.search(
    query="How does hybrid search combine results?",
)

for chunk in results["results"]["chunk_search_results"]:
    print(f"Score: {chunk['score']:.4f}")
    print(f"Text: {chunk['text'][:200]}...")
    print(f"Doc ID: {chunk['document_id']}")
    print("---")

# Hybrid search with custom weights
results = client.retrieval.search(
    query="pgvector HNSW configuration",
    search_settings={
        "use_vector_search": True,
        "use_fulltext_search": True,
        "use_kg_search": False,
        "search_limit": 10,
        "hybrid_settings": {
            "full_text_weight": 1.0,
            "semantic_weight": 5.0,
            "rrf_k": 50,
        },
    },
)

# Filtered search (metadata-based)
results = client.retrieval.search(
    query="deployment procedures",
    search_settings={
        "use_vector_search": True,
        "filters": {
            "metadata.department": {"$eq": "engineering"},
            "metadata.version": {"$gte": "2.0"},
        },
    },
)
```

### RAG (Retrieve + Generate)

```python
# Basic RAG
response = client.retrieval.rag(
    query="What are the best practices for configuring R2R in production?",
    rag_generation_config={
        "model": "openai/gpt-4o-mini",
        "temperature": 0.0,
        "max_tokens": 1000,
    },
    search_settings={
        "use_vector_search": True,
        "use_fulltext_search": True,
        "search_limit": 5,
    },
)

answer = response["results"]["completion"]["choices"][0]["message"]["content"]
print(f"Answer: {answer}")

# Access search results used for generation
search_results = response["results"]["search_results"]
print(f"\nSources used: {len(search_results['chunk_search_results'])}")

# Streaming RAG
for chunk in client.retrieval.rag(
    query="Explain the R2R ingestion pipeline",
    rag_generation_config={
        "model": "openai/gpt-4o-mini",
        "stream": True,
    },
):
    content = chunk.get("choices", [{}])[0].get("delta", {}).get("content", "")
    if content:
        print(content, end="", flush=True)
print()
```

### Knowledge Graph

```python
# Build knowledge graph for a collection
client.graphs.build(
    collection_id="default",
    settings={
        "extraction_model": "gpt-4o-mini",
        "max_entities_per_chunk": 10,
        "max_relations_per_chunk": 15,
    },
    run_with_orchestration=True,
)

# Search using knowledge graph
results = client.retrieval.search(
    query="What technologies are related to vector search?",
    search_settings={
        "use_kg_search": True,
        "use_vector_search": True,
        "kg_search_settings": {
            "kg_search_type": "local",
        },
    },
)

# Access KG results
if results["results"].get("kg_search_results"):
    for entity in results["results"]["kg_search_results"]:
        print(f"Entity: {entity}")
```

### Conversations (Multi-Turn RAG)

```python
# Create a conversation
conversation = client.conversations.create()
conv_id = conversation["results"]["id"]

# First turn
response = client.retrieval.agent(
    message={
        "role": "user",
        "content": "What embedding models does R2R support?",
    },
    conversation_id=conv_id,
    rag_generation_config={"model": "openai/gpt-4o-mini"},
)
print(response["results"]["completion"]["choices"][0]["message"]["content"])

# Follow-up (maintains context)
response = client.retrieval.agent(
    message={
        "role": "user",
        "content": "Which one would you recommend for a small dataset?",
    },
    conversation_id=conv_id,
    rag_generation_config={"model": "openai/gpt-4o-mini"},
)
print(response["results"]["completion"]["choices"][0]["message"]["content"])

# List conversations
conversations = client.conversations.list()
```

---

## Part 3: Docker Production Deployment

### Custom Docker Compose

```yaml
# docker-compose.yml
version: "3.8"

services:
  r2r:
    image: ragtoriches/prod:latest
    ports:
      - "7272:7272"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - R2R_POSTGRES_HOST=postgres
      - R2R_POSTGRES_PORT=5432
      - R2R_POSTGRES_DBNAME=r2r
      - R2R_POSTGRES_USER=r2r
      - R2R_POSTGRES_PASSWORD=${DB_PASSWORD}
      - R2R_CONFIG_PATH=/app/config/r2r.toml
    volumes:
      - ./config:/app/config
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

  postgres:
    image: pgvector/pgvector:pg16
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=r2r
      - POSTGRES_USER=r2r
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U r2r"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  hatchet:
    image: ghcr.io/hatchet-dev/hatchet/hatchet-lite:latest
    ports:
      - "7077:7077"
    environment:
      - DATABASE_URL=postgres://r2r:${DB_PASSWORD}@postgres:5432/r2r
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

volumes:
  pgdata:
```

### Configuration File

```toml
# config/r2r.toml

[app]
project_name = "production-rag"

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

### Deploying

```bash
# Set environment variables
export OPENAI_API_KEY="sk-..."
export DB_PASSWORD="secure-password"

# Start services
docker compose up -d

# Check logs
docker compose logs -f r2r

# Verify
curl http://localhost:7272/v3/health
```

### Backup and Restore

```bash
# Backup PostgreSQL data
docker compose exec postgres pg_dump -U r2r r2r > backup.sql

# Restore
cat backup.sql | docker compose exec -T postgres psql -U r2r r2r
```

---

## Complete Example: Building a Knowledge Base

```python
from r2r import R2RClient
from pathlib import Path
import time

# Connect
client = R2RClient("http://localhost:7272")

# Step 1: Ingest documents
docs_dir = Path("./knowledge_base")
doc_ids = []

for file_path in docs_dir.glob("**/*"):
    if file_path.suffix in {".md", ".pdf", ".txt", ".html", ".docx"}:
        try:
            result = client.documents.create(
                file_path=str(file_path),
                metadata={
                    "source": "knowledge_base",
                    "category": file_path.parent.name,
                    "filename": file_path.name,
                },
            )
            doc_ids.append(result["results"]["document_id"])
            print(f"Ingested: {file_path.name}")
        except Exception as e:
            print(f"Failed: {file_path.name}: {e}")

print(f"\nIngested {len(doc_ids)} documents")

# Step 2: Build knowledge graph
print("\nBuilding knowledge graph...")
client.graphs.build(
    collection_id="default",
    settings={"extraction_model": "gpt-4o-mini"},
    run_with_orchestration=True,
)

# Step 3: Wait for processing
print("Waiting for processing to complete...")
time.sleep(30)  # adjust based on corpus size

# Step 4: Search
print("\nRunning test searches...")
test_queries = [
    "What are the main architecture components?",
    "How is authentication handled?",
    "What databases are supported?",
]

for query in test_queries:
    results = client.retrieval.search(
        query=query,
        search_settings={
            "use_vector_search": True,
            "use_fulltext_search": True,
            "search_limit": 3,
        },
    )
    print(f"\nQ: {query}")
    for r in results["results"]["chunk_search_results"][:2]:
        print(f"  [{r['score']:.3f}] {r['text'][:100]}...")

# Step 5: RAG query
response = client.retrieval.rag(
    query="Summarize the system architecture in 3 bullet points.",
    rag_generation_config={"model": "openai/gpt-4o-mini", "temperature": 0.0},
)
print(f"\nRAG Answer:\n{response['results']['completion']['choices'][0]['message']['content']}")
```

---

## Common Pitfalls

1. **Docker not running**: Most R2R operations require the server to be running. Always start with `r2r serve --docker` or `docker compose up -d` before using the client.

2. **PostgreSQL not ready**: On first start, PostgreSQL needs time to initialize. If R2R fails to connect, wait 30 seconds and retry, or use Docker Compose healthchecks.

3. **Forgetting to build the knowledge graph**: Document ingestion does not automatically build the KG. You must explicitly call `graphs.build()` after ingestion.

4. **Large file ingestion timeouts**: PDF and DOCX files over 50MB may timeout during ingestion. Use `run_with_orchestration=True` for async processing of large files.

5. **Not filtering by collection**: In multi-tenant setups, searches without collection filters return results from all tenants. Always specify `collection_ids` in search settings.

6. **Default embedding dimensions**: The default `text-embedding-3-small` uses 1536 dimensions. Set `dimension=512` in config for better performance with minimal quality loss.

---

## References

- R2R documentation: https://r2r-docs.sciphi.ai/
- R2R GitHub: https://github.com/SciPhi-AI/R2R
- R2R Python SDK: https://pypi.org/project/r2r/
- Docker deployment guide: https://r2r-docs.sciphi.ai/self-hosting/docker
