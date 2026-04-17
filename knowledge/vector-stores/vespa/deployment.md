# Vespa -- Deployment and Operations

## Overview

This guide covers deploying Vespa for vector search and hybrid retrieval: Docker single-node for development, Vespa Cloud (managed), self-hosted multi-node clusters, application package deployment, schema definition, document feeding with vespa-feed-client, in-cluster embedding with ONNX models, and monitoring. Vespa's deployment model is application-package-based: you define your schema, services, and models in a package, then deploy it as an atomic unit.

---

## Docker Single-Node (Development)

### Quick Start

```bash
# Pull the Vespa container image
docker pull vespaengine/vespa:latest

# Run Vespa (single-node, all services)
docker run -d \
    --name vespa \
    --hostname vespa-container \
    -p 8080:8080 \
    -p 19071:19071 \
    vespaengine/vespa:latest

# Wait for configuration service to be ready
curl -s --retry 30 --retry-delay 2 --retry-connrefused \
    http://localhost:19071/state/v1/health

# Deploy application package
vespa deploy --wait 300 app/
```

### Docker Compose

```yaml
# docker-compose.yml
version: "3.8"

services:
  vespa:
    image: vespaengine/vespa:latest
    container_name: vespa
    hostname: vespa-container
    ports:
      - "8080:8080"    # Search and feed API
      - "19071:19071"  # Config server
    volumes:
      - vespa_data:/opt/vespa/var
    environment:
      - VESPA_CONFIGSERVERS=vespa-container
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:19071/state/v1/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

volumes:
  vespa_data:
    driver: local
```

### Application Package Structure

```
app/
  schemas/
    document.sd          # Schema definition
  services.xml           # Service configuration
  models/                # ONNX models (optional)
    e5-small.onnx
    tokenizer.json
```

### Minimal services.xml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<services version="1.0">

  <!-- Container cluster: query processing, document processing, HTTP API -->
  <container id="default" version="1.0">
    <search />
    <document-api />
    <document-processing />
    <nodes>
      <node hostalias="node1" />
    </nodes>
  </container>

  <!-- Content cluster: data storage and search -->
  <content id="documents" version="1.0">
    <redundancy>1</redundancy>
    <documents>
      <document type="document" mode="index" />
    </documents>
    <nodes>
      <node hostalias="node1" distribution-key="0" />
    </nodes>
  </content>

</services>
```

### Minimal Schema

```
# schemas/document.sd
schema document {
    document document {
        field title type string {
            indexing: summary | index
            index: enable-bm25
        }

        field content type string {
            indexing: summary | index
            index: enable-bm25
        }

        field embedding type tensor<float>(x[1536]) {
            indexing: summary | attribute | index
            attribute {
                distance-metric: angular
            }
            index {
                hnsw {
                    max-links-per-node: 16
                    neighbors-to-explore-at-insert: 200
                }
            }
        }

        field category type string {
            indexing: summary | attribute
            attribute: fast-search
        }

        field created_at type long {
            indexing: summary | attribute
        }
    }

    fieldset default {
        fields: title, content
    }

    rank-profile default {
        inputs {
            query(query_embedding) tensor<float>(x[1536])
        }

        first-phase {
            expression: closeness(field, embedding)
        }
    }

    rank-profile hybrid inherits default {
        first-phase {
            expression {
                0.5 * closeness(field, embedding) +
                0.3 * bm25(title) +
                0.2 * bm25(content)
            }
        }
    }
}
```

### Deploy and Test

```bash
# Install Vespa CLI
brew install vespa-cli  # macOS
# or: curl -fsSL https://cli.vespa.ai/install.sh | bash

# Deploy the application package
vespa deploy --wait 300 app/

# Verify deployment
vespa status

# Feed a document
vespa document - <<'EOF'
{
    "put": "id:namespace:document::doc1",
    "fields": {
        "title": "Introduction to Vector Search",
        "content": "Vector search enables semantic similarity...",
        "embedding": [0.1, 0.2, 0.3],
        "category": "search",
        "created_at": 1700000000
    }
}
EOF

# Query
vespa query \
    'yql=select * from document where ({targetHits:10}nearestNeighbor(embedding, query_embedding))' \
    'input.query(query_embedding)=[0.1, 0.2, 0.3]' \
    'ranking=hybrid'
```

---

## Vespa Cloud (Managed)

### Setup

```bash
# Install Vespa CLI
brew install vespa-cli

# Log in to Vespa Cloud
vespa auth login

# Create application
vespa config set target cloud
vespa config set application my-tenant.my-app

# Deploy to Vespa Cloud
vespa deploy --wait 600 app/
```

### Vespa Cloud services.xml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<services version="1.0">

  <container id="default" version="1.0">
    <search />
    <document-api />
    <document-processing />
    <nodes count="2">
      <resources vcpu="4" memory="16Gb" disk="50Gb" />
    </nodes>
  </container>

  <content id="documents" version="1.0">
    <min-redundancy>2</min-redundancy>
    <documents>
      <document type="document" mode="index" />
    </documents>
    <nodes count="3">
      <resources vcpu="8" memory="32Gb" disk="200Gb" />
    </nodes>
  </content>

</services>
```

### Vespa Cloud Features

| Feature | Description |
|---|---|
| **Auto-scaling** | Content and container nodes scale based on load |
| **Zero-downtime deploys** | Application packages deploy without interrupting queries |
| **Multi-region** | Deploy to multiple regions for latency and redundancy |
| **Managed upgrades** | Vespa version upgrades handled automatically |
| **Integrated monitoring** | Metrics, logs, and alerts built in |
| **Encryption** | Data encrypted at rest and in transit by default |

### Vespa Cloud Pricing Guidance

| Configuration | Content Nodes | Container Nodes | Estimated Cost/mo |
|---|---|---|---|
| Dev/Test | 1 x (2 vCPU, 8 GB) | 1 x (2 vCPU, 4 GB) | $200-400 |
| Small Production | 3 x (4 vCPU, 16 GB) | 2 x (4 vCPU, 8 GB) | $800-1,500 |
| Medium Production | 3 x (8 vCPU, 64 GB) | 3 x (8 vCPU, 16 GB) | $3,000-5,000 |
| Large Production | 6 x (16 vCPU, 128 GB) | 4 x (16 vCPU, 32 GB) | $10,000-20,000 |

---

## Self-Hosted Multi-Node Cluster

### Architecture

```
Config Servers (3 nodes, ZooKeeper)
  - vespa-config-1 (19071)
  - vespa-config-2 (19071)
  - vespa-config-3 (19071)

Container Cluster (stateless, behind load balancer)
  - vespa-container-1 (8080)
  - vespa-container-2 (8080)

Content Cluster (stateful)
  - vespa-content-1 (19112)
  - vespa-content-2 (19112)
  - vespa-content-3 (19112)
```

### hosts.xml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<hosts>
  <host name="vespa-config-1"><alias>config1</alias></host>
  <host name="vespa-config-2"><alias>config2</alias></host>
  <host name="vespa-config-3"><alias>config3</alias></host>
  <host name="vespa-container-1"><alias>container1</alias></host>
  <host name="vespa-container-2"><alias>container2</alias></host>
  <host name="vespa-content-1"><alias>content1</alias></host>
  <host name="vespa-content-2"><alias>content2</alias></host>
  <host name="vespa-content-3"><alias>content3</alias></host>
</hosts>
```

### Multi-Node services.xml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<services version="1.0">

  <admin version="2.0">
    <configservers>
      <configserver hostalias="config1" />
      <configserver hostalias="config2" />
      <configserver hostalias="config3" />
    </configservers>
  </admin>

  <container id="default" version="1.0">
    <search />
    <document-api />
    <document-processing />
    <nodes>
      <node hostalias="container1" />
      <node hostalias="container2" />
    </nodes>
  </container>

  <content id="documents" version="1.0">
    <min-redundancy>2</min-redundancy>
    <documents>
      <document type="document" mode="index" />
    </documents>
    <nodes>
      <node hostalias="content1" distribution-key="0" />
      <node hostalias="content2" distribution-key="1" />
      <node hostalias="content3" distribution-key="2" />
    </nodes>
  </content>

</services>
```

### Systemd Service

```ini
# /etc/systemd/system/vespa.service
[Unit]
Description=Vespa Search Engine
After=network.target

[Service]
Type=forking
User=vespa
Group=vespa
ExecStart=/opt/vespa/bin/vespa-start-services
ExecStop=/opt/vespa/bin/vespa-stop-services
ExecStartPost=/bin/bash -c 'until curl -sf http://localhost:19071/state/v1/health; do sleep 2; done'
LimitNOFILE=65536
LimitMEMLOCK=infinity
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

---

## Schema Definition Deep Dive

### Advanced Schema with Multiple Ranking Profiles

```
schema document {
    document document {
        field doc_id type string {
            indexing: summary | attribute
            attribute: fast-search
        }

        field title type string {
            indexing: summary | index
            index: enable-bm25
            stemming: best
        }

        field content type string {
            indexing: summary | index
            index: enable-bm25
            stemming: best
        }

        field embedding type tensor<float>(x[1536]) {
            indexing: summary | attribute | index
            attribute {
                distance-metric: angular
            }
            index {
                hnsw {
                    max-links-per-node: 24
                    neighbors-to-explore-at-insert: 300
                }
            }
        }

        # ColBERT token-level embeddings
        field colbert_embedding type tensor<float>(token{}, x[128]) {
            indexing: summary | attribute
        }

        field category type string {
            indexing: summary | attribute
            attribute: fast-search
        }

        field tenant_id type string {
            indexing: summary | attribute
            attribute: fast-search
        }

        field language type string {
            indexing: summary | attribute
            attribute: fast-search
        }

        field created_at type long {
            indexing: summary | attribute
            attribute: fast-search
        }

        field chunk_index type int {
            indexing: summary | attribute
        }

        field parent_doc_id type string {
            indexing: summary | attribute
            attribute: fast-search
        }
    }

    fieldset default {
        fields: title, content
    }

    # Vector-only search
    rank-profile vector-search {
        inputs {
            query(query_embedding) tensor<float>(x[1536])
        }
        first-phase {
            expression: closeness(field, embedding)
        }
        match-features {
            closeness(field, embedding)
        }
    }

    # Hybrid: vector + BM25
    rank-profile hybrid-search {
        inputs {
            query(query_embedding) tensor<float>(x[1536])
        }
        first-phase {
            expression {
                0.5 * closeness(field, embedding) +
                0.3 * bm25(title) +
                0.2 * bm25(content)
            }
        }
        match-features {
            closeness(field, embedding)
            bm25(title)
            bm25(content)
        }
    }

    # Hybrid + ColBERT re-ranking
    rank-profile colbert-hybrid {
        inputs {
            query(query_embedding) tensor<float>(x[1536])
            query(query_tokens) tensor<float>(token{}, x[128])
        }

        function max_sim() {
            expression {
                sum(
                    reduce(
                        sum(
                            query(query_tokens) * attribute(colbert_embedding),
                            x
                        ),
                        max,
                        token
                    ),
                    token
                )
            }
        }

        first-phase {
            expression {
                0.5 * closeness(field, embedding) +
                0.3 * bm25(title) +
                0.2 * bm25(content)
            }
        }

        second-phase {
            expression {
                0.5 * max_sim() +
                0.3 * closeness(field, embedding) +
                0.1 * bm25(title) +
                0.1 * bm25(content)
            }
            rerank-count: 100
        }
    }

    # Freshness-boosted ranking
    rank-profile fresh-hybrid {
        inputs {
            query(query_embedding) tensor<float>(x[1536])
            query(now) long
        }

        function freshness() {
            expression {
                exp(-0.693 * max(0, query(now) - attribute(created_at)) / 2592000)
            }
        }

        first-phase {
            expression {
                0.4 * closeness(field, embedding) +
                0.3 * bm25(content) +
                0.3 * freshness()
            }
        }
    }
}
```

---

## Document Feeding

### vespa-feed-client (High Throughput)

```bash
# Install vespa-feed-client (Java)
# Included in Vespa CLI or download separately

# Feed from JSONL file
vespa feed documents.jsonl \
    --target https://vespa-endpoint:8080 \
    --connections 128 \
    --max-streams-per-connection 256

# Feed with progress monitoring
vespa feed documents.jsonl \
    --target http://localhost:8080 \
    --connections 64 \
    --speedtest
```

### JSONL Format

```json
{"put": "id:default:document::doc1", "fields": {"doc_id": "doc1", "title": "Introduction to ML", "content": "Machine learning is...", "embedding": [0.1, 0.2, 0.3], "category": "ml", "tenant_id": "t1", "created_at": 1700000000}}
{"put": "id:default:document::doc2", "fields": {"doc_id": "doc2", "title": "Deep Learning", "content": "Neural networks...", "embedding": [0.4, 0.5, 0.6], "category": "dl", "tenant_id": "t1", "created_at": 1700100000}}
{"update": "id:default:document::doc1", "fields": {"category": {"assign": "ml-updated"}}}
{"remove": "id:default:document::doc3"}
```

### PyVespa Feeding (Python)

```python
from vespa.application import Vespa
from vespa.io import VespaResponse
import json


def feed_documents(app_url: str, documents: list[dict]):
    """Feed documents to Vespa using PyVespa."""
    app = Vespa(url=app_url, port=8080)

    with app.syncio() as session:
        for doc in documents:
            response: VespaResponse = session.feed_data_point(
                schema="document",
                data_id=doc["doc_id"],
                fields=doc
            )
            if not response.is_successful():
                print(f"Failed to feed {doc['doc_id']}: {response.json}")


async def async_feed_documents(app_url: str, documents: list[dict]):
    """Async feeding for high throughput."""
    app = Vespa(url=app_url, port=8080)

    async with app.asyncio() as session:
        for doc in documents:
            response = await session.feed_data_point(
                schema="document",
                data_id=doc["doc_id"],
                fields=doc
            )
            if not response.is_successful():
                print(f"Failed: {doc['doc_id']}")


# HTTP API feeding (no PyVespa dependency)
import requests


def feed_via_http(endpoint: str, doc_id: str, fields: dict):
    """Feed a single document via HTTP API."""
    url = f"{endpoint}/document/v1/default/document/docid/{doc_id}"
    response = requests.post(
        url,
        headers={"Content-Type": "application/json"},
        json={"fields": fields}
    )
    return response.json()
```

---

## Embedding In-Cluster (ONNX Models)

### services.xml with Embedder

```xml
<?xml version="1.0" encoding="utf-8" ?>
<services version="1.0">

  <container id="default" version="1.0">
    <search />
    <document-api />
    <document-processing />

    <!-- HuggingFace embedder: embeds text on feed and query -->
    <component id="e5-small"
               class="ai.vespa.embedding.huggingface.HuggingFaceEmbedder"
               bundle="model-integration">
      <config name="embedding.huggingface.huggingface-embedder">
        <transformerModelPath>models/e5-small-v2.onnx</transformerModelPath>
        <tokenizerPath>models/tokenizer.json</tokenizerPath>
        <transformerMaxTokens>512</transformerMaxTokens>
        <transformerGpuDevice>-1</transformerGpuDevice>
      </config>
    </component>

    <!-- ColBERT embedder for token-level embeddings -->
    <component id="colbert"
               class="ai.vespa.embedding.colbert.ColBertEmbedder"
               bundle="model-integration">
      <config name="embedding.colbert.colbert-embedder">
        <transformerModelPath>models/colbert-v2.onnx</transformerModelPath>
        <tokenizerPath>models/colbert-tokenizer.json</tokenizerPath>
        <maxQueryTokens>64</maxQueryTokens>
        <maxDocumentTokens>512</maxDocumentTokens>
      </config>
    </component>

    <nodes>
      <node hostalias="node1" />
    </nodes>
  </container>

  <content id="documents" version="1.0">
    <redundancy>1</redundancy>
    <documents>
      <document type="document" mode="index" />
    </documents>
    <nodes>
      <node hostalias="node1" distribution-key="0" />
    </nodes>
  </content>

</services>
```

### Schema with Embed Expressions

```
schema document {
    document document {
        field title type string {
            indexing: summary | index
            index: enable-bm25
        }

        field content type string {
            indexing: summary | index
            index: enable-bm25
        }

        # Auto-embed content field using e5-small
        field embedding type tensor<float>(x[384]) {
            indexing: input content | embed e5-small | attribute | index
            attribute {
                distance-metric: angular
            }
            index {
                hnsw {
                    max-links-per-node: 16
                    neighbors-to-explore-at-insert: 200
                }
            }
        }

        # Auto-embed for ColBERT
        field colbert_embedding type tensor<float>(token{}, x[128]) {
            indexing: input content | embed colbert | attribute
        }

        field category type string {
            indexing: summary | attribute
            attribute: fast-search
        }
    }
}
```

### Exporting ONNX Models

```python
# Export HuggingFace model to ONNX for Vespa
from transformers import AutoModel, AutoTokenizer
from optimum.exporters.onnx import main_export

# Export e5-small-v2 to ONNX
main_export(
    model_name_or_path="intfloat/e5-small-v2",
    output="models/",
    task="feature-extraction",
    opset=13
)

# Rename for Vespa
import shutil
shutil.move("models/model.onnx", "models/e5-small-v2.onnx")

# Export tokenizer
tokenizer = AutoTokenizer.from_pretrained("intfloat/e5-small-v2")
tokenizer.save_pretrained("models/")
```

---

## Monitoring

### Health Checks

```bash
# Config server health
curl http://localhost:19071/state/v1/health

# Container cluster health
curl http://localhost:8080/state/v1/health

# Application status
vespa status

# Cluster state
curl http://localhost:19071/cluster/v2/
```

### Key Metrics

```bash
# Document count and memory usage
curl http://localhost:19071/metrics/v2/values | python3 -m json.tool

# Key metrics to watch:
# - content.proton.documentdb.documents.active   (document count)
# - content.proton.documentdb.memory_usage.total  (memory used)
# - content.proton.resource_usage.memory           (node memory %)
# - content.proton.resource_usage.disk             (node disk %)
# - queries.rate                                    (QPS)
# - query_latency.average                          (average latency)
# - feed.operations.rate                            (feed rate)
```

### PyVespa Monitoring

```python
from vespa.application import Vespa

app = Vespa(url="http://localhost", port=8080)

# Get application status
status = app.get_application_status()
print(f"Application: {status}")

# Search with trace for debugging
result = app.query(
    yql="select * from document where ({targetHits:10}nearestNeighbor(embedding, query_embedding))",
    body={
        "input.query(query_embedding)": [0.1, 0.2, 0.3],
        "ranking": "hybrid",
        "trace.level": 3  # Detailed trace
    }
)
print(f"Hits: {result.number_documents_retrieved}")
print(f"Coverage: {result.json.get('root', {}).get('coverage', {})}")
```

---

## Common Pitfalls

1. **Not deploying the application package after changes**: Vespa requires explicit `vespa deploy` after any schema, services.xml, or model change. Running `vespa deploy` is atomic and supports zero-downtime.

2. **Using too few `neighbors-to-explore-at-insert`**: this parameter controls HNSW build quality. Setting it too low (e.g., 50) creates a low-quality graph. Use 200+ for production.

3. **Forgetting to set `targetHits` in queries**: the `nearestNeighbor` operator requires `targetHits` to know how many candidates to retrieve from the HNSW graph. Without it, the query fails.

4. **Running expensive ranking in first-phase**: first-phase runs on ALL matched documents. ColBERT max-sim or complex tensor operations should be in second-phase with a bounded `rerank-count`.

5. **Not pre-warming the container after deployment**: after a fresh deployment, the first few queries may be slow as ONNX models load and HNSW indexes are mapped into memory. Run warmup queries before routing production traffic.

6. **Undersizing content nodes for HNSW**: HNSW indexes live in memory on content nodes. Ensure content nodes have enough RAM to hold all HNSW graphs plus working memory for ranking computations.

7. **Not using `attribute: fast-search` for filter fields**: without `fast-search`, attribute fields are scanned linearly during filtering. With it, a B-tree index is created for fast lookups.

---

## References

- Vespa documentation: https://docs.vespa.ai/
- Vespa Cloud: https://cloud.vespa.ai/
- Vespa CLI: https://docs.vespa.ai/en/vespa-cli.html
- PyVespa: https://pyvespa.readthedocs.io/
- vespa-feed-client: https://docs.vespa.ai/en/vespa-feed-client.html
- ONNX model integration: https://docs.vespa.ai/en/onnx.html
