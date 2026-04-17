# ChromaDB Advanced -- Deployment and Operations

## Overview

This guide covers ChromaDB deployment options: local development with pip, shared deployments with Docker, managed via Chroma Cloud, client/server architecture, migration patterns, backup strategies, multi-tenant patterns, and authentication configuration. ChromaDB 0.5+ is designed to be simple to deploy and operate, but production use requires understanding the persistence model, server modes, and scaling boundaries.

---

## Local Development (pip install)

### Basic Installation

```bash
# Install ChromaDB
pip install chromadb

# With specific embedding function support
pip install chromadb[openai]      # OpenAI embedding function
pip install chromadb[cohere]      # Cohere embedding function
pip install chromadb[huggingface] # HuggingFace embedding function
```

### Persistent Client (Default)

```python
import chromadb

# Data persisted to local directory
client = chromadb.PersistentClient(path="/data/chromadb")

# Create a collection
collection = client.get_or_create_collection(
    name="documents",
    metadata={"hnsw:space": "cosine"}
)

# Add documents
collection.add(
    ids=["doc1", "doc2"],
    documents=["Hello world", "Machine learning overview"],
    metadatas=[{"source": "test"}, {"source": "wiki"}]
)

# Data is persisted automatically
# Restart the process and data is still there:
client2 = chromadb.PersistentClient(path="/data/chromadb")
collection2 = client2.get_collection("documents")
print(collection2.count())  # 2
```

### Ephemeral Client (Testing)

```python
import chromadb

# In-memory only, data lost when process exits
client = chromadb.EphemeralClient()

collection = client.create_collection("test")
collection.add(ids=["1"], documents=["Test document"])

# When the process exits, all data is gone
```

---

## Docker Deployment (Shared Access)

### Basic Docker Run

```bash
# Pull the latest ChromaDB image
docker pull chromadb/chroma:latest

# Run with persistent storage
docker run -d \
    --name chromadb \
    -p 8000:8000 \
    -v /data/chromadb:/chroma/chroma \
    -e ANONYMIZED_TELEMETRY=FALSE \
    -e CHROMA_SERVER_AUTHN_PROVIDER= \
    chromadb/chroma:latest
```

### Docker Compose

```yaml
# docker-compose.yml
version: "3.8"

services:
  chromadb:
    image: chromadb/chroma:latest
    container_name: chromadb
    ports:
      - "8000:8000"
    volumes:
      - chromadb_data:/chroma/chroma
    environment:
      - ANONYMIZED_TELEMETRY=FALSE
      - IS_PERSISTENT=TRUE
      - CHROMA_SERVER_AUTHN_PROVIDER=chromadb.auth.token_authn.TokenAuthenticationServerProvider
      - CHROMA_SERVER_AUTHN_CREDENTIALS=your-secret-token-here
      - CHROMA_SERVER_AUTHN_CREDENTIALS_PROVIDER=chromadb.auth.token_authn.TokenAuthConfigurationServerProvider
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/v1/heartbeat"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  chromadb_data:
    driver: local
```

### Connect from Client

```python
import chromadb

# Connect to the Docker server
client = chromadb.HttpClient(
    host="localhost",
    port=8000,
    headers={"Authorization": "Bearer your-secret-token-here"}
)

# Verify connection
heartbeat = client.heartbeat()
print(f"Connected to ChromaDB, heartbeat: {heartbeat}")

# Use exactly like a local client
collection = client.get_or_create_collection("documents")
```

---

## Chroma Cloud (Managed)

Chroma Cloud is the managed service by the Chroma team. It handles persistence, scaling, and operations.

### Setup

```python
import chromadb

# Connect to Chroma Cloud
client = chromadb.CloudClient(
    tenant="my-organization",
    database="production",
    api_key="your-api-key"
)

# Use normally
collection = client.get_or_create_collection(
    name="documents",
    metadata={"hnsw:space": "cosine"}
)
```

### Chroma Cloud vs Self-Hosted

| Feature | Self-Hosted (Docker) | Chroma Cloud |
|---|---|---|
| Management | You manage | Managed |
| Scaling | Manual | Automatic |
| Backup | Manual | Automatic |
| Cost | Infrastructure only | Per-request pricing |
| Max vectors | Limited by hardware | Plan-dependent |
| Multi-region | DIY | Available |
| SLA | None | Plan-dependent |

---

## Client/Server Mode Architecture

### Server Startup Options

```bash
# Start with custom configuration
chroma run \
    --host 0.0.0.0 \
    --port 8000 \
    --path /data/chromadb \
    --log-path /var/log/chromadb.log

# Or via Python
python -m chromadb.server \
    --host 0.0.0.0 \
    --port 8000 \
    --path /data/chromadb
```

### Server Configuration File

```yaml
# chroma_config.yaml
chroma_server_host: "0.0.0.0"
chroma_server_http_port: 8000
persist_directory: "/data/chromadb"
anonymized_telemetry: false
chroma_server_authn_provider: "chromadb.auth.token_authn.TokenAuthenticationServerProvider"
chroma_server_authn_credentials: "your-secret-token"
```

### Multi-Process Architecture

```
                    Load Balancer
                    /           \
                   /             \
            Client A          Client B
               |                  |
               v                  v
          ChromaDB Server (single instance)
               |
               v
          Persistent Storage (/data/chromadb)
```

**Important**: ChromaDB does not support horizontal scaling of the server process. A single Chroma server handles all requests. For higher throughput, vertically scale the server (more CPU, RAM) or shard data across multiple Chroma instances at the application level.

---

## Migration Patterns

### Ephemeral to Persistent

```python
import chromadb

# Source: ephemeral client with data
ephemeral = chromadb.EphemeralClient()
src_collection = ephemeral.get_collection("documents")

# Destination: persistent client
persistent = chromadb.PersistentClient(path="/data/chromadb")
dst_collection = persistent.get_or_create_collection(
    name="documents",
    metadata=src_collection.metadata
)

# Migrate data in batches
BATCH_SIZE = 1000
total = src_collection.count()

for offset in range(0, total, BATCH_SIZE):
    batch = src_collection.get(
        limit=BATCH_SIZE,
        offset=offset,
        include=["documents", "metadatas", "embeddings"]
    )

    if not batch["ids"]:
        break

    dst_collection.add(
        ids=batch["ids"],
        documents=batch["documents"],
        metadatas=batch["metadatas"],
        embeddings=batch["embeddings"]
    )
    print(f"Migrated {min(offset + BATCH_SIZE, total)}/{total}")

print(f"Migration complete: {dst_collection.count()} documents")
```

### Local to Client/Server

```python
import chromadb

# Source: local persistent client
local = chromadb.PersistentClient(path="/data/chromadb")

# Destination: remote server
remote = chromadb.HttpClient(host="chromadb-server", port=8000)

# Migrate all collections
for src_coll_info in local.list_collections():
    src_collection = local.get_collection(src_coll_info.name)
    dst_collection = remote.get_or_create_collection(
        name=src_coll_info.name,
        metadata=src_collection.metadata
    )

    total = src_collection.count()
    BATCH_SIZE = 500  # Smaller batches for network transfer

    for offset in range(0, total, BATCH_SIZE):
        batch = src_collection.get(
            limit=BATCH_SIZE,
            offset=offset,
            include=["documents", "metadatas", "embeddings"]
        )

        if not batch["ids"]:
            break

        dst_collection.add(
            ids=batch["ids"],
            documents=batch["documents"] if batch["documents"][0] else None,
            metadatas=batch["metadatas"] if batch["metadatas"][0] else None,
            embeddings=batch["embeddings"]
        )

    print(f"Collection '{src_coll_info.name}': {dst_collection.count()} docs")
```

### Version Upgrade Migration (0.4.x to 0.5.x)

ChromaDB 0.5 introduced a new storage format. The migration is automatic on first startup.

```bash
# Before upgrading
# 1. Backup your data directory
cp -r /data/chromadb /data/chromadb-backup-v04

# 2. Upgrade ChromaDB
pip install --upgrade chromadb

# 3. Start with the same persist directory
# ChromaDB will automatically migrate the data format
python -c "
import chromadb
client = chromadb.PersistentClient(path='/data/chromadb')
print(f'Collections: {len(client.list_collections())}')
print('Migration successful')
"
```

---

## Backup Patterns

### File-System Backup

```bash
#!/bin/bash
# backup-chromadb.sh

CHROMA_DIR="/data/chromadb"
BACKUP_DIR="/backups/chromadb"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Stop writes if possible (or accept a slightly inconsistent snapshot)
mkdir -p "$BACKUP_DIR"

# Create compressed backup
tar -czf "$BACKUP_DIR/chromadb_${TIMESTAMP}.tar.gz" \
    -C "$(dirname $CHROMA_DIR)" \
    "$(basename $CHROMA_DIR)"

# Cleanup old backups (keep last 7)
ls -t "$BACKUP_DIR"/chromadb_*.tar.gz | tail -n +8 | xargs -r rm

echo "Backup created: chromadb_${TIMESTAMP}.tar.gz"
```

### Programmatic Backup

```python
import chromadb
import json
from pathlib import Path
from datetime import datetime


def backup_to_jsonl(client: chromadb.ClientAPI, backup_dir: str):
    """Export all collections to JSONL files."""
    backup_path = Path(backup_dir) / datetime.now().strftime("%Y%m%d_%H%M%S")
    backup_path.mkdir(parents=True, exist_ok=True)

    manifest = {"collections": [], "timestamp": datetime.now().isoformat()}

    for coll_info in client.list_collections():
        collection = client.get_collection(coll_info.name)
        total = collection.count()
        coll_file = backup_path / f"{coll_info.name}.jsonl"

        with open(coll_file, "w") as f:
            for offset in range(0, total, 1000):
                batch = collection.get(
                    limit=1000,
                    offset=offset,
                    include=["documents", "metadatas", "embeddings"]
                )

                for i, doc_id in enumerate(batch["ids"]):
                    record = {
                        "id": doc_id,
                        "document": batch["documents"][i] if batch["documents"] else None,
                        "metadata": batch["metadatas"][i] if batch["metadatas"] else None,
                        "embedding": batch["embeddings"][i] if batch["embeddings"] else None,
                    }
                    f.write(json.dumps(record) + "\n")

        manifest["collections"].append({
            "name": coll_info.name,
            "metadata": collection.metadata,
            "count": total,
            "file": coll_file.name
        })

    with open(backup_path / "manifest.json", "w") as f:
        json.dump(manifest, f, indent=2)

    print(f"Backup complete: {backup_path}")
    return str(backup_path)


def restore_from_jsonl(client: chromadb.ClientAPI, backup_dir: str):
    """Restore collections from JSONL backup."""
    backup_path = Path(backup_dir)
    with open(backup_path / "manifest.json") as f:
        manifest = json.load(f)

    for coll_info in manifest["collections"]:
        collection = client.get_or_create_collection(
            name=coll_info["name"],
            metadata=coll_info.get("metadata")
        )

        coll_file = backup_path / coll_info["file"]
        batch_ids, batch_docs, batch_metas, batch_embs = [], [], [], []

        with open(coll_file) as f:
            for line in f:
                record = json.loads(line)
                batch_ids.append(record["id"])
                batch_docs.append(record["document"])
                batch_metas.append(record["metadata"])
                batch_embs.append(record["embedding"])

                if len(batch_ids) >= 500:
                    collection.upsert(
                        ids=batch_ids,
                        documents=batch_docs if batch_docs[0] else None,
                        metadatas=batch_metas if batch_metas[0] else None,
                        embeddings=batch_embs if batch_embs[0] else None,
                    )
                    batch_ids, batch_docs, batch_metas, batch_embs = [], [], [], []

            if batch_ids:
                collection.upsert(
                    ids=batch_ids,
                    documents=batch_docs if batch_docs[0] else None,
                    metadatas=batch_metas if batch_metas[0] else None,
                    embeddings=batch_embs if batch_embs[0] else None,
                )

        print(f"Restored '{coll_info['name']}': {collection.count()} docs")
```

---

## Multi-Tenant Patterns

### Collection-Per-Tenant

The simplest multi-tenant pattern: each tenant gets its own collection.

```python
import chromadb


class MultiTenantChroma:
    """Multi-tenant ChromaDB with collection-per-tenant isolation."""

    def __init__(self, path: str = "/data/chromadb"):
        self.client = chromadb.PersistentClient(path=path)

    def get_tenant_collection(self, tenant_id: str):
        """Get or create a collection for a specific tenant."""
        collection_name = f"tenant_{tenant_id}"
        return self.client.get_or_create_collection(
            name=collection_name,
            metadata={"hnsw:space": "cosine", "tenant_id": tenant_id}
        )

    def add_documents(self, tenant_id: str, ids: list, documents: list,
                      metadatas: list = None, embeddings: list = None):
        """Add documents for a specific tenant."""
        collection = self.get_tenant_collection(tenant_id)
        collection.add(
            ids=ids,
            documents=documents,
            metadatas=metadatas,
            embeddings=embeddings
        )

    def search(self, tenant_id: str, query_embeddings: list,
               n_results: int = 10, where: dict = None):
        """Search within a specific tenant's collection."""
        collection = self.get_tenant_collection(tenant_id)
        return collection.query(
            query_embeddings=query_embeddings,
            n_results=n_results,
            where=where
        )

    def delete_tenant(self, tenant_id: str):
        """Delete all data for a tenant."""
        collection_name = f"tenant_{tenant_id}"
        try:
            self.client.delete_collection(collection_name)
        except ValueError:
            pass  # Collection does not exist
```

**Limitations of collection-per-tenant**:
- ChromaDB creates a separate HNSW index per collection. Hundreds of collections consume significant memory.
- Cross-tenant search requires querying each collection individually.
- Recommended for up to ~50-100 tenants. Beyond that, use metadata filtering.

### Metadata-Based Tenancy

For many tenants, store all data in one collection and filter by tenant metadata.

```python
class MetadataMultiTenant:
    """Multi-tenant ChromaDB with metadata-based isolation."""

    def __init__(self, path: str = "/data/chromadb"):
        self.client = chromadb.PersistentClient(path=path)
        self.collection = self.client.get_or_create_collection(
            name="shared_documents",
            metadata={"hnsw:space": "cosine"}
        )

    def add_documents(self, tenant_id: str, ids: list, documents: list,
                      metadatas: list = None, embeddings: list = None):
        """Add documents with tenant_id in metadata."""
        if metadatas is None:
            metadatas = [{}] * len(ids)

        # Inject tenant_id into metadata
        for meta in metadatas:
            meta["tenant_id"] = tenant_id

        # Prefix IDs to avoid cross-tenant collision
        prefixed_ids = [f"{tenant_id}:{doc_id}" for doc_id in ids]

        self.collection.add(
            ids=prefixed_ids,
            documents=documents,
            metadatas=metadatas,
            embeddings=embeddings
        )

    def search(self, tenant_id: str, query_embeddings: list,
               n_results: int = 10, where: dict = None):
        """Search filtered to a specific tenant."""
        # Build tenant filter
        tenant_filter = {"tenant_id": {"$eq": tenant_id}}

        if where:
            combined_filter = {"$and": [tenant_filter, where]}
        else:
            combined_filter = tenant_filter

        return self.collection.query(
            query_embeddings=query_embeddings,
            n_results=n_results,
            where=combined_filter
        )
```

---

## Authentication

### Token Authentication

```yaml
# docker-compose.yml with token auth
services:
  chromadb:
    image: chromadb/chroma:latest
    environment:
      - CHROMA_SERVER_AUTHN_PROVIDER=chromadb.auth.token_authn.TokenAuthenticationServerProvider
      - CHROMA_SERVER_AUTHN_CREDENTIALS=secret-token-12345
      - CHROMA_SERVER_AUTHN_CREDENTIALS_PROVIDER=chromadb.auth.token_authn.TokenAuthConfigurationServerProvider
```

```python
# Client with token auth
client = chromadb.HttpClient(
    host="localhost",
    port=8000,
    headers={"Authorization": "Bearer secret-token-12345"}
)
```

### Basic Authentication

```yaml
# docker-compose.yml with basic auth
services:
  chromadb:
    image: chromadb/chroma:latest
    environment:
      - CHROMA_SERVER_AUTHN_PROVIDER=chromadb.auth.basic_authn.BasicAuthenticationServerProvider
      - CHROMA_SERVER_AUTHN_CREDENTIALS_FILE=/chroma/auth/credentials.htpasswd
    volumes:
      - ./credentials.htpasswd:/chroma/auth/credentials.htpasswd:ro
```

```bash
# Generate credentials file
htpasswd -Bbn admin your-password > credentials.htpasswd
```

```python
# Client with basic auth
from chromadb.config import Settings

client = chromadb.HttpClient(
    host="localhost",
    port=8000,
    settings=Settings(
        chroma_client_auth_provider="chromadb.auth.basic_authn.BasicAuthClientProvider",
        chroma_client_auth_credentials="admin:your-password"
    )
)
```

### Reverse Proxy with TLS

```nginx
# nginx.conf
server {
    listen 443 ssl;
    server_name chromadb.example.com;

    ssl_certificate /etc/ssl/certs/chromadb.crt;
    ssl_certificate_key /etc/ssl/private/chromadb.key;

    location / {
        proxy_pass http://chromadb:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Increase timeout for large batch operations
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }
}
```

---

## Performance Tuning

### Memory Requirements

| Vectors | Dimensions | Estimated RAM | Notes |
|---|---|---|---|
| 10K | 1536 | 200 MB | Minimal |
| 50K | 1536 | 500 MB | Comfortable |
| 100K | 1536 | 1 GB | Default settings |
| 500K | 1536 | 4-5 GB | Tune HNSW params |
| 1M | 1536 | 8-10 GB | Near practical limit |

### Docker Resource Limits

```yaml
services:
  chromadb:
    image: chromadb/chroma:latest
    deploy:
      resources:
        limits:
          memory: 8G
          cpus: "4"
        reservations:
          memory: 4G
          cpus: "2"
```

---

## Common Pitfalls

1. **Assuming horizontal scaling**: ChromaDB server is single-instance. You cannot run multiple servers against the same storage directory. For higher throughput, scale vertically or shard at the application level.

2. **Using ephemeral client in production**: EphemeralClient loses all data on process restart. Always use PersistentClient or HttpClient in production.

3. **Creating too many collections**: each collection has its own HNSW index. Hundreds of collections with large datasets will exhaust memory. Use metadata filtering for multi-tenancy beyond ~50 tenants.

4. **Not setting auth in Docker deployments**: by default, the ChromaDB Docker image has no authentication. Anyone who can reach port 8000 can read/write all data. Always configure auth.

5. **Large batch sizes over HTTP**: sending batches of 10K+ documents over HTTP to a remote server can timeout. Use batch sizes of 500-1000 for HttpClient.

6. **Not backing up the persist directory**: ChromaDB stores data in the persist directory. If that disk is lost, all data is gone. Schedule regular backups.

7. **Mixing embedding functions**: if you create a collection with OpenAI embeddings and later query with Cohere embeddings, results are meaningless. The embedding function is not stored with the collection.

---

## References

- ChromaDB documentation: https://docs.trychroma.com/
- ChromaDB Docker image: https://hub.docker.com/r/chromadb/chroma
- Chroma Cloud: https://www.trychroma.com/cloud
- ChromaDB authentication: https://docs.trychroma.com/deployment/auth
- ChromaDB GitHub: https://github.com/chroma-core/chroma
