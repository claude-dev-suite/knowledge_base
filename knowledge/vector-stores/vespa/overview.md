# Vespa Deep Guide

## Overview

Vespa is an open-source platform for building search, recommendation, and AI applications at scale. Unlike purpose-built vector databases, Vespa provides first-class hybrid retrieval: BM25 text search, approximate nearest neighbor (ANN) vector search, and learned late interaction models (ColBERT) can all be combined in a single ranking expression. Vespa's architecture separates content nodes (data storage and retrieval) from container nodes (query processing and document feeding), with config servers coordinating the cluster. Ranking happens in multiple phases (match, first-phase, second-phase, global-phase), and tensor expressions enable arbitrary ML computations inline. This guide covers the architecture, schema definition, ranking, hybrid retrieval, and tensor expressions.

For basic Vespa deployment and document operations, see the companion deployment guide. This document focuses on Vespa's data model, ranking system, and hybrid retrieval capabilities.

---

## Architecture

### Component Overview

```
Vespa Cluster
+----------------------------------------------------------+
|                                                           |
|  Config Servers (3 nodes, ZooKeeper-based)               |
|  - Application package management                        |
|  - Cluster state coordination                            |
|  - Schema distribution                                   |
|                                                           |
|  Container Cluster (stateless)                           |
|  +-----------------------------------------------------+ |
|  | - HTTP API (document, query, feed)                   | |
|  | - Query parsing and dispatch                         | |
|  | - Result processing (global-phase ranking)           | |
|  | - Embedding inference (ONNX models)                  | |
|  | - Document processing pipeline                       | |
|  +-----------------------------------------------------+ |
|                                                           |
|  Content Cluster (stateful)                              |
|  +-----------------------------------------------------+ |
|  | Node 1          Node 2          Node 3              | |
|  | - Proton        - Proton        - Proton            | |
|  |   (search      (search        (search              | |
|  |    engine)      engine)        engine)              | |
|  | - Match phase   - Match phase  - Match phase        | |
|  | - First/second  - First/second - First/second       | |
|  |   phase rank    phase rank     phase rank           | |
|  +-----------------------------------------------------+ |
|  | Distribution: automatic, groups, redundancy         | |
+----------------------------------------------------------+
```

### Key Architectural Properties

| Property | Description |
|---|---|
| **Stateless containers** | Scale independently of content. Handle query parsing, document processing, and global-phase ranking. |
| **Stateful content** | Store documents and indexes. Each node runs Proton (Vespa's native search engine). |
| **Real-time** | Documents are searchable within milliseconds of feeding. No batch reindexing required. |
| **Multi-phase ranking** | Coarse-to-fine ranking reduces computation. Expensive ML models run only on top candidates. |
| **In-process tensors** | Tensor operations execute directly in the search engine, no external model serving needed. |
| **Self-healing** | Automatic data redistribution when nodes are added/removed. |

### Proton: The Search Engine

Proton is Vespa's native C++ search engine running on each content node. It manages:

- **Document store**: raw document storage with compression.
- **Attribute store**: in-memory columnar storage for filtering, grouping, and ranking.
- **Index**: inverted indexes for text fields, HNSW graphs for tensor fields.
- **Ranking**: executes rank expressions using document features and query features.

---

## Schema Definition (.sd Files)

Vespa schemas define document types, fields, indexes, and ranking profiles using `.sd` (schema definition) files.

### Basic Schema

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

        field metadata type map<string, string> {
            indexing: summary
        }
    }

    fieldset default {
        fields: title, content
    }
}
```

### Field Indexing Modes

| Mode | Description | Use For |
|---|---|---|
| `index` | Inverted index (text search) | BM25 text retrieval |
| `attribute` | In-memory columnar | Filtering, grouping, sorting, ranking |
| `summary` | Stored for retrieval | Returning in search results |

### Tensor Field Types

```
# Dense vector (fixed dimensions)
field embedding type tensor<float>(x[1536]) { ... }

# Dense vector with bfloat16 (half memory)
field embedding type tensor<bfloat16>(x[1536]) { ... }

# Dense vector with int8 (quarter memory)
field embedding type tensor<int8>(x[1536]) { ... }

# Multi-vector (for ColBERT late interaction)
field token_embeddings type tensor<float>(token{}, x[128]) { ... }

# Sparse tensor (for learned sparse representations)
field sparse_embedding type tensor<float>(term{}) { ... }

# 2D tensor (for cross-attention patterns)
field attention_matrix type tensor<float>(row[32], col[32]) { ... }
```

### Distance Metrics

| Metric | Schema Name | Formula | Best For |
|---|---|---|---|
| Cosine | `angular` | 1 - cosine_similarity | Text embeddings |
| Euclidean | `euclidean` | L2 distance | Spatial data |
| Inner product | `dotproduct` | Negative dot product | Pre-normalized vectors |
| Hamming | `hamming` | Hamming distance | Binary vectors |

---

## Ranking Phases

Vespa's multi-phase ranking is its most powerful feature. It allows progressively more expensive computations on smaller candidate sets.

### Phase Overview

```
All documents in index
        |
        v
   Match Phase          (boolean matching: text + filter)
   ~millions of docs
        |
        v
   First Phase          (cheap rank expression on ALL matched docs)
   ~thousands ranked
        |
        v
   Second Phase         (expensive rank expression on top N)
   ~hundreds re-ranked
        |
        v
   Global Phase         (cross-node ranking on top M)
   ~tens re-ranked      (runs on container, sees all results)
        |
        v
   Final Results
```

### Ranking Profile Definition

```
schema document {
    # ... document fields ...

    rank-profile hybrid-search inherits default {
        inputs {
            query(query_embedding) tensor<float>(x[1536])
            query(query_tokens) tensor<float>(token{}, x[128])
        }

        # First phase: fast scoring on all matched documents
        first-phase {
            expression: closeness(field, embedding)
        }

        # Second phase: re-rank top 100 with expensive features
        second-phase {
            expression {
                0.4 * closeness(field, embedding) +
                0.3 * bm25(title) +
                0.2 * bm25(content) +
                0.1 * freshness(created_at)
            }
            rerank-count: 100
        }

        # Match features to return in results
        match-features {
            closeness(field, embedding)
            bm25(title)
            bm25(content)
            freshness(created_at)
        }
    }

    rank-profile colbert-reranking inherits hybrid-search {
        # Override second-phase with ColBERT late interaction
        second-phase {
            expression {
                0.5 * max_sim(query(query_tokens), attribute(token_embeddings)) +
                0.3 * closeness(field, embedding) +
                0.2 * bm25(content)
            }
            rerank-count: 50
        }
    }
}
```

### Phase Cost Model

| Phase | Evaluated On | Typical Cost | Use For |
|---|---|---|---|
| Match | Index traversal | O(matched docs) | Boolean filters, text matching |
| First-phase | All matched docs | Microseconds/doc | Simple dot products, single features |
| Second-phase | Top N (rerank-count) | Milliseconds/doc | Multi-feature scoring, tensor ops |
| Global-phase | Top M (across nodes) | Milliseconds/doc | Cross-document features (diversity, MMR) |

---

## Hybrid Retrieval

### BM25 + ANN in One Query

```python
import requests
import json

query = {
    "yql": (
        "select * from document where "
        "({targetHits: 100}nearestNeighbor(embedding, query_embedding)) "
        "or userQuery()"
    ),
    "query": "machine learning transformers",
    "input.query(query_embedding)": json.dumps(query_vector),
    "ranking": "hybrid-search",
    "hits": 10
}

response = requests.get(
    "http://localhost:8080/search/",
    params=query
)
results = response.json()
```

### How Hybrid Works in Vespa

Unlike systems that run vector and text search separately then merge, Vespa combines them in a single query execution:

1. **Match phase**: the `nearestNeighbor` operator retrieves top-N ANN candidates. The `userQuery()` retrieves BM25 matches. The `or` combines both sets.
2. **First-phase**: a cheap ranking expression scores all matched documents.
3. **Second-phase**: an expensive expression (potentially including ColBERT) re-ranks the top candidates.

This eliminates the need for external result merging (like Reciprocal Rank Fusion) though RRF is also supported in the global phase.

### ColBERT Late Interaction

ColBERT computes a similarity score by performing a MaxSim operation between query token embeddings and document token embeddings.

```
schema document {
    document document {
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

        field colbert_tokens type tensor<float>(token{}, x[128]) {
            indexing: summary | attribute
        }
    }

    rank-profile colbert inherits default {
        inputs {
            query(query_embedding) tensor<float>(x[1536])
            query(query_tokens) tensor<float>(token{}, x[128])
        }

        function max_sim() {
            expression {
                sum(
                    reduce(
                        sum(
                            query(query_tokens) * attribute(colbert_tokens),
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
            expression: closeness(field, embedding)
        }

        second-phase {
            expression {
                0.6 * max_sim() +
                0.2 * closeness(field, embedding) +
                0.2 * bm25(content)
            }
            rerank-count: 50
        }
    }
}
```

### Reciprocal Rank Fusion in Global Phase

```
rank-profile rrf-hybrid inherits default {
    inputs {
        query(query_embedding) tensor<float>(x[1536])
    }

    function vector_score() {
        expression: closeness(field, embedding)
    }

    function text_score() {
        expression: 0.6 * bm25(title) + 0.4 * bm25(content)
    }

    first-phase {
        expression: vector_score() + text_score()
    }

    global-phase {
        expression {
            reciprocal_rank(vector_score(), 60) +
            reciprocal_rank(text_score(), 60)
        }
        rerank-count: 100
    }
}
```

---

## Tensor Expressions

Vespa's tensor expression language enables arbitrary ML computations inline in ranking.

### Common Tensor Operations

```
# Dot product (closeness is a shorthand)
sum(query(query_embedding) * attribute(embedding), x)

# Cosine similarity
sum(query(q) * attribute(d), x) / (
    sqrt(sum(query(q) * query(q), x)) *
    sqrt(sum(attribute(d) * attribute(d), x))
)

# L2 distance
sqrt(sum(pow(query(q) - attribute(d), 2), x))

# MaxSim (ColBERT)
sum(
    reduce(
        sum(query(qt) * attribute(dt), x),
        max, token
    ),
    token
)

# Weighted combination
0.5 * closeness(field, embedding) +
0.3 * bm25(title) +
0.2 * freshness(timestamp)
```

### Custom Freshness Function

```
rank-profile with-freshness inherits default {
    inputs {
        query(query_embedding) tensor<float>(x[1536])
        query(now) long
    }

    function freshness_score() {
        expression {
            # Exponential decay: half-life of 30 days
            exp(-0.693 * (query(now) - attribute(created_at)) / 2592000)
        }
    }

    first-phase {
        expression {
            0.7 * closeness(field, embedding) +
            0.2 * bm25(content) +
            0.1 * freshness_score()
        }
    }
}
```

---

## Embedding Inference In-Cluster

Vespa can run ONNX models directly in the container node, embedding documents and queries without external API calls.

### ONNX Model Configuration

```xml
<!-- services.xml -->
<container id="default" version="1.0">
    <component id="e5-small"
               class="ai.vespa.embedding.huggingface.HuggingFaceEmbedder"
               bundle="model-integration">
        <config name="embedding.huggingface.huggingface-embedder">
            <transformerModelPath>models/e5-small-v2.onnx</transformerModelPath>
            <tokenizerPath>models/tokenizer.json</tokenizerPath>
            <transformerMaxTokens>512</transformerMaxTokens>
        </config>
    </component>
</container>
```

```
# Schema with embedder reference
schema document {
    document document {
        field content type string {
            indexing: summary | index
            index: enable-bm25
        }

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
    }
}
```

With this configuration:
- **On document feed**: the `content` field is automatically embedded by the `e5-small` model. No external embedding API call needed.
- **On query**: the query text is embedded in the container node before being dispatched to content nodes.

### Supported Embedders

| Embedder | Class | Use Case |
|---|---|---|
| HuggingFace | `HuggingFaceEmbedder` | Any ONNX-exported HuggingFace model |
| ColBERT | `ColBertEmbedder` | ColBERT-style multi-vector models |
| SPLADE | `SpladeEmbedder` | Learned sparse representation models |
| Custom | Implement `Embedder` interface | Any custom model |

---

## Document Feeding

### vespa-feed-client (High-Performance)

```bash
# Feed a JSONL file at high throughput
vespa feed documents.jsonl \
    --target https://vespa-cluster:8080 \
    --connections 128 \
    --max-streams-per-connection 256
```

### JSONL Document Format

```json
{"put": "id:namespace:document::doc1", "fields": {"title": "Introduction to ML", "content": "Machine learning is...", "category": "ml", "created_at": 1700000000}}
{"put": "id:namespace:document::doc2", "fields": {"title": "Deep Learning", "content": "Neural networks are...", "category": "dl", "created_at": 1700100000}}
```

### Python Feeding

```python
import requests
import json


def feed_document(doc_id: str, fields: dict, endpoint: str = "http://localhost:8080"):
    """Feed a single document to Vespa."""
    url = f"{endpoint}/document/v1/namespace/document/docid/{doc_id}"
    response = requests.post(
        url,
        headers={"Content-Type": "application/json"},
        data=json.dumps({"fields": fields})
    )
    return response.json()


def feed_batch(documents: list[dict], endpoint: str = "http://localhost:8080"):
    """Feed documents in batch using the /document/v1/ API."""
    results = []
    for doc in documents:
        result = feed_document(
            doc_id=doc["id"],
            fields=doc["fields"],
            endpoint=endpoint
        )
        results.append(result)
    return results


# With PyVespa (official Python client)
from vespa.application import Vespa
from vespa.io import VespaResponse

app = Vespa(url="http://localhost", port=8080)

with app.syncio() as session:
    for doc in documents:
        response: VespaResponse = session.feed_data_point(
            schema="document",
            data_id=doc["id"],
            fields=doc["fields"]
        )
        if not response.is_successful():
            print(f"Failed: {response.json}")
```

---

## Query Patterns

### Nearest Neighbor with Filter

```python
query = {
    "yql": (
        "select * from document where "
        "({targetHits: 100}nearestNeighbor(embedding, query_embedding)) "
        "and category contains 'ml' "
        "and created_at > 1700000000"
    ),
    "input.query(query_embedding)": json.dumps(query_vector),
    "ranking": "hybrid-search",
    "hits": 10
}
```

### Approximate vs Exact Nearest Neighbor

```python
# Approximate (uses HNSW, fast)
"({targetHits: 100}nearestNeighbor(embedding, query_embedding))"

# Exact brute-force (slower, 100% recall)
"({targetHits: 100, approximate: false}nearestNeighbor(embedding, query_embedding))"
```

### Grouping Results

```python
# Group search results by category
query = {
    "yql": (
        "select * from document where "
        "({targetHits: 200}nearestNeighbor(embedding, query_embedding)) "
        "| all(group(category) each(max(5) each(output(summary()))))"
    ),
    "input.query(query_embedding)": json.dumps(query_vector),
    "ranking": "hybrid-search"
}
```

---

## Common Pitfalls

1. **Not using multi-phase ranking**: running an expensive ML model in first-phase on all matched documents kills latency. Move expensive computations to second-phase with a small `rerank-count`.

2. **Setting targetHits too low**: `targetHits` controls how many candidates the HNSW index explores. Setting it to 10 when you need 10 results gives poor recall. Use 50-200 for good recall at the ANN stage.

3. **Ignoring tensor type precision**: `tensor<float>` uses 4 bytes/dim. `tensor<bfloat16>` uses 2 bytes/dim with <1% quality loss. `tensor<int8>` uses 1 byte/dim. Choose based on your scale.

4. **Not force-flushing after bulk feed**: Vespa indexes documents in real-time, but large bulk feeds may have a lag. Use the `/document/v1/` API with `?route=default` and monitor the `content.proton.documentdb.documents.active` metric.

5. **Using OR for hybrid without understanding the match set**: `nearestNeighbor OR userQuery()` creates the union of ANN and BM25 candidates. This can be very large. Ensure your first-phase rank expression is cheap.

6. **Not pre-normalizing vectors for dotproduct**: the `dotproduct` distance metric assumes vectors are L2-normalized. Using non-normalized vectors gives incorrect similarity scores.

7. **Forgetting to deploy the application package after schema changes**: Vespa requires explicit `vespa deploy` after any schema modification. Changes to `.sd` files, `services.xml`, or model files are not picked up automatically.

---

## References

- Vespa documentation: https://docs.vespa.ai/
- Vespa ranking reference: https://docs.vespa.ai/en/ranking.html
- Vespa tensor guide: https://docs.vespa.ai/en/tensor-user-guide.html
- Vespa nearest neighbor search: https://docs.vespa.ai/en/nearest-neighbor-search.html
- ColBERT in Vespa: https://docs.vespa.ai/en/colbert.html
- PyVespa: https://pyvespa.readthedocs.io/
- Vespa blog: https://blog.vespa.ai/
