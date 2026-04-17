# Elasticsearch Vector Search Deep Guide

## Overview

Elasticsearch 8.x provides native vector search via the `dense_vector` field type with HNSW indexing, kNN search API, hybrid search (BM25 + kNN with Reciprocal Rank Fusion), ELSER learned sparse retrieval, custom scoring with `script_score`, nested vectors, int8 scalar quantization, and the retriever API. This guide covers the full spectrum of Elasticsearch vector capabilities for production use.

For basic Elasticsearch setup, see the official documentation at <https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html>.

---

## Dense Vector Field Type

### Basic Configuration

```json
PUT /documents
{
  "mappings": {
    "properties": {
      "title": { "type": "text" },
      "content": { "type": "text" },
      "category": { "type": "keyword" },
      "year": { "type": "integer" },
      "embedding": {
        "type": "dense_vector",
        "dims": 1536,
        "index": true,
        "similarity": "cosine",
        "index_options": {
          "type": "hnsw",
          "m": 16,
          "ef_construction": 128
        }
      }
    }
  }
}
```

### Similarity Metrics

| Similarity | ES Parameter | Score Formula | Use Case |
|-----------|-------------|---------------|----------|
| Cosine | `cosine` | `(1 + cosine_sim) / 2` | Text embeddings (default) |
| Dot product | `dot_product` | `(1 + dot_product) / 2` | Pre-normalized vectors |
| L2 norm | `l2_norm` | `1 / (1 + l2_norm^2)` | Image embeddings |
| Max inner product | `max_inner_product` | Complex (see docs) | Asymmetric embeddings |

### Int8 Scalar Quantization (ES 8.12+)

```json
PUT /documents_quantized
{
  "mappings": {
    "properties": {
      "embedding": {
        "type": "dense_vector",
        "dims": 1536,
        "index": true,
        "similarity": "cosine",
        "index_options": {
          "type": "int8_hnsw",
          "m": 16,
          "ef_construction": 128,
          "confidence_interval": 0.99
        }
      }
    }
  }
}
```

Int8 quantization reduces memory by ~4x with <1% recall loss. The `confidence_interval` parameter controls outlier clipping (0.99 means clip values above the 99th percentile).

---

## kNN Search API

### Basic kNN Search

```json
POST /documents/_search
{
  "knn": {
    "field": "embedding",
    "query_vector": [0.1, 0.2, ...],
    "k": 10,
    "num_candidates": 100
  },
  "_source": ["title", "content", "category"]
}
```

### kNN with Pre-Filtering

```json
POST /documents/_search
{
  "knn": {
    "field": "embedding",
    "query_vector": [0.1, 0.2, ...],
    "k": 10,
    "num_candidates": 100,
    "filter": {
      "bool": {
        "must": [
          { "term": { "category": "engineering" } },
          { "range": { "year": { "gte": 2023 } } }
        ]
      }
    }
  }
}
```

**Important**: kNN filters in Elasticsearch are **pre-filters** (applied before the approximate search), unlike post-filters in some other vector databases. This means highly selective filters can degrade recall because fewer candidates are available for the HNSW traversal. Increase `num_candidates` when using restrictive filters.

### Python Client

```python
from elasticsearch import Elasticsearch
import numpy as np

es = Elasticsearch(
    "https://localhost:9200",
    api_key="your-api-key",
    ca_certs="/path/to/ca.crt",
)

# Index a document
es.index(
    index="documents",
    id="doc-1",
    document={
        "title": "Vector Search in Elasticsearch",
        "content": "Elasticsearch supports approximate kNN search...",
        "category": "technology",
        "year": 2025,
        "embedding": np.random.rand(1536).tolist(),
    },
)

# kNN search
query_vector = np.random.rand(1536).tolist()
results = es.search(
    index="documents",
    knn={
        "field": "embedding",
        "query_vector": query_vector,
        "k": 10,
        "num_candidates": 100,
    },
    source=["title", "category"],
)

for hit in results["hits"]["hits"]:
    print(f"{hit['_score']:.4f} - {hit['_source']['title']}")
```

---

## Hybrid Search (BM25 + kNN with RRF)

### Reciprocal Rank Fusion (ES 8.9+)

RRF combines BM25 text scores with kNN vector scores without requiring score normalization.

```json
POST /documents/_search
{
  "retriever": {
    "rrf": {
      "retrievers": [
        {
          "standard": {
            "query": {
              "multi_match": {
                "query": "vector database performance",
                "fields": ["title^2", "content"]
              }
            }
          }
        },
        {
          "knn": {
            "field": "embedding",
            "query_vector": [0.1, 0.2, ...],
            "k": 50,
            "num_candidates": 200
          }
        }
      ],
      "rank_window_size": 100,
      "rank_constant": 60
    }
  },
  "size": 10
}
```

### RRF Formula

```
RRF_score = sum(1 / (rank_constant + rank_i)) for each retriever
```

Where `rank_constant` (default 60) dampens the influence of rank position. Higher values make the ranking more uniform; lower values favor top-ranked results.

### Python Client Hybrid Search

```python
# Hybrid search with RRF
results = es.search(
    index="documents",
    retriever={
        "rrf": {
            "retrievers": [
                {
                    "standard": {
                        "query": {
                            "multi_match": {
                                "query": "vector database performance",
                                "fields": ["title^2", "content"],
                            }
                        }
                    }
                },
                {
                    "knn": {
                        "field": "embedding",
                        "query_vector": query_vector,
                        "k": 50,
                        "num_candidates": 200,
                    }
                },
            ],
            "rank_window_size": 100,
            "rank_constant": 60,
        }
    },
    size=10,
)

for hit in results["hits"]["hits"]:
    print(f"{hit['_score']:.4f} - {hit['_source']['title']}")
```

### Weighted kNN + BM25 (Without RRF)

```json
POST /documents/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "multi_match": {
            "query": "vector database",
            "fields": ["title^2", "content"],
            "boost": 0.3
          }
        }
      ]
    }
  },
  "knn": {
    "field": "embedding",
    "query_vector": [0.1, 0.2, ...],
    "k": 50,
    "num_candidates": 200,
    "boost": 0.7
  },
  "size": 10
}
```

**Note**: the `boost`-based approach requires careful score calibration because BM25 and kNN scores are on different scales. RRF is generally preferred because it operates on ranks, not raw scores.

---

## ELSER (Elastic Learned Sparse EncodeR)

ELSER is Elasticsearch's trained sparse retrieval model that generates learned sparse representations, similar to SPLADE.

### Setup

```json
PUT /_ml/trained_models/.elser_model_2
{
  "input": { "field_names": ["text_field"] }
}

POST /_ml/trained_models/.elser_model_2/deployment/_start
{
  "number_of_allocations": 2,
  "threads_per_allocation": 1
}
```

### Index Mapping with ELSER

```json
PUT /documents_elser
{
  "mappings": {
    "properties": {
      "content": { "type": "text" },
      "content_tokens": {
        "type": "sparse_vector"
      }
    }
  }
}
```

### Ingest Pipeline for ELSER

```json
PUT /_ingest/pipeline/elser-pipeline
{
  "processors": [
    {
      "inference": {
        "model_id": ".elser_model_2",
        "input_output": [
          {
            "input_field": "content",
            "output_field": "content_tokens"
          }
        ]
      }
    }
  ]
}
```

### ELSER Search

```json
POST /documents_elser/_search
{
  "query": {
    "sparse_vector": {
      "field": "content_tokens",
      "inference_id": ".elser_model_2",
      "query": "how do vector databases work"
    }
  }
}
```

### ELSER + kNN Hybrid

```json
POST /documents/_search
{
  "retriever": {
    "rrf": {
      "retrievers": [
        {
          "standard": {
            "query": {
              "sparse_vector": {
                "field": "content_tokens",
                "inference_id": ".elser_model_2",
                "query": "vector database performance"
              }
            }
          }
        },
        {
          "knn": {
            "field": "embedding",
            "query_vector": [0.1, 0.2, ...],
            "k": 50,
            "num_candidates": 200
          }
        }
      ],
      "rank_window_size": 100
    }
  },
  "size": 10
}
```

---

## Script Score for Custom Scoring

For custom distance functions or re-ranking logic, use `script_score`.

```json
POST /documents/_search
{
  "query": {
    "script_score": {
      "query": {
        "bool": {
          "filter": { "term": { "category": "engineering" } }
        }
      },
      "script": {
        "source": "cosineSimilarity(params.query_vector, 'embedding') + 1.0",
        "params": {
          "query_vector": [0.1, 0.2, ...]
        }
      }
    }
  }
}
```

### Custom Scoring Functions

```json
{
  "script": {
    "source": """
      double vector_score = cosineSimilarity(params.qvec, 'embedding') + 1.0;
      double recency = 1.0 / (1.0 + Math.abs(doc['year'].value - params.current_year));
      return vector_score * 0.8 + recency * 0.2;
    """,
    "params": {
      "qvec": [0.1, 0.2, ...],
      "current_year": 2025
    }
  }
}
```

**Warning**: `script_score` performs a brute-force scan and does not use the HNSW index. It is 10-100x slower than kNN for large indexes. Use it only with restrictive pre-filters or on small datasets.

---

## Nested Vectors

For documents with multiple vector fields (e.g., paragraph-level embeddings).

```json
PUT /documents_nested
{
  "mappings": {
    "properties": {
      "title": { "type": "text" },
      "paragraphs": {
        "type": "nested",
        "properties": {
          "text": { "type": "text" },
          "embedding": {
            "type": "dense_vector",
            "dims": 1536,
            "index": true,
            "similarity": "cosine"
          }
        }
      }
    }
  }
}
```

### Nested kNN Search

```json
POST /documents_nested/_search
{
  "knn": {
    "field": "paragraphs.embedding",
    "query_vector": [0.1, 0.2, ...],
    "k": 10,
    "num_candidates": 100,
    "inner_hits": {
      "_source": ["paragraphs.text"],
      "size": 3
    }
  }
}
```

---

## Retriever API (ES 8.14+)

The retriever API provides a composable search pipeline for combining multiple retrieval strategies.

### Multi-Stage Retriever

```json
POST /documents/_search
{
  "retriever": {
    "rrf": {
      "retrievers": [
        {
          "standard": {
            "query": {
              "match": { "content": "vector search optimization" }
            }
          }
        },
        {
          "knn": {
            "field": "embedding",
            "query_vector": [0.1, 0.2, ...],
            "k": 50,
            "num_candidates": 200
          }
        },
        {
          "standard": {
            "query": {
              "sparse_vector": {
                "field": "content_tokens",
                "inference_id": ".elser_model_2",
                "query": "vector search optimization"
              }
            }
          }
        }
      ],
      "rank_window_size": 200,
      "rank_constant": 60
    }
  },
  "size": 10
}
```

This combines BM25, dense vector, and ELSER sparse vector retrieval with a single RRF fusion step.

---

## Bulk Indexing with Vectors

```python
from elasticsearch import Elasticsearch, helpers
import numpy as np

es = Elasticsearch("https://localhost:9200", api_key="your-api-key")

def generate_actions(documents, embeddings):
    """Generate bulk index actions."""
    for i, (doc, emb) in enumerate(zip(documents, embeddings)):
        yield {
            "_index": "documents",
            "_id": f"doc-{i}",
            "_source": {
                "title": doc["title"],
                "content": doc["content"],
                "category": doc["category"],
                "year": doc.get("year", 2025),
                "embedding": emb.tolist(),
            },
        }

# Bulk index
documents = [{"title": f"Doc {i}", "content": f"Content {i}", "category": "tech"} for i in range(100_000)]
embeddings = np.random.rand(100_000, 1536).astype(np.float32)

success, errors = helpers.bulk(
    es,
    generate_actions(documents, embeddings),
    chunk_size=500,
    request_timeout=120,
    raise_on_error=False,
)
print(f"Indexed {success} documents, {len(errors)} errors")
```

### Force Merge for kNN Performance

After bulk indexing, force-merge segments for optimal kNN search performance.

```python
# Force merge to 1 segment per shard (run after bulk indexing is complete)
es.indices.forcemerge(
    index="documents",
    max_num_segments=1,
    wait_for_completion=True,
    request_timeout=3600,
)
```

**Why force merge matters**: HNSW graphs are built per segment. With many small segments, each search must traverse multiple independent graphs and merge results. A single segment means a single, well-connected HNSW graph with better recall and lower latency.

**Force merge impact** (1M vectors, 1536 dims):

| Segments | Recall@10 | p50 (ms) | p99 (ms) |
|----------|----------|---------|---------|
| 50 (default after bulk) | 0.945 | 8.5 | 25.0 |
| 10 | 0.972 | 5.2 | 15.0 |
| 5 | 0.983 | 3.8 | 11.0 |
| 1 (force merged) | 0.991 | 2.5 | 8.0 |

---

## Query Patterns

### kNN with Boost and BM25

```python
# Hybrid: BM25 + kNN with boost weights
results = es.search(
    index="documents",
    query={
        "bool": {
            "should": [
                {
                    "multi_match": {
                        "query": "vector database performance",
                        "fields": ["title^3", "content"],
                        "boost": 0.3,
                    }
                },
            ],
            "filter": [
                {"term": {"category": "technology"}},
                {"range": {"year": {"gte": 2023}}},
            ],
        }
    },
    knn={
        "field": "embedding",
        "query_vector": query_vector,
        "k": 50,
        "num_candidates": 200,
        "boost": 0.7,
    },
    size=10,
)
```

### Batch kNN Search (msearch)

```python
# Multiple kNN queries in one request
body = []
for qvec in query_vectors:
    body.append({"index": "documents"})
    body.append({
        "knn": {
            "field": "embedding",
            "query_vector": qvec.tolist(),
            "k": 10,
            "num_candidates": 100,
        },
        "size": 10,
    })

results = es.msearch(body=body)
for i, response in enumerate(results["responses"]):
    hits = response["hits"]["hits"]
    print(f"Query {i}: {len(hits)} results")
```

---

## Common Pitfalls

1. **Not running force-merge after bulk indexing**: multiple segments mean multiple independent HNSW graphs with degraded recall and higher latency. Always force-merge to 1 segment per shard after bulk loads.

2. **Using `script_score` for large-scale vector search**: script_score is brute-force and does not use the HNSW index. Use the kNN API instead.

3. **Setting `num_candidates` too low with pre-filters**: aggressive filters reduce the candidate pool. If you filter to 1% of the data, set `num_candidates` to at least 10x `k`.

4. **Not accounting for shard count in kNN**: each shard returns `k` results independently, and results are merged. More shards can improve throughput but may reduce recall if `num_candidates` per shard is too low.

5. **Mixing index-time and query-time similarity**: the similarity metric is set at index creation and cannot be changed. Ensure it matches your embedding model's expected distance function.

6. **Forgetting to allocate ML nodes for ELSER**: ELSER inference requires ML nodes with sufficient memory (at least 4 GB per model deployment). Without ML nodes, ELSER inference fails.

7. **Using nested vectors without `inner_hits`**: without `inner_hits`, you get the parent document but not which specific nested vector matched, making it impossible to identify the relevant paragraph.

8. **Ignoring quantization for cost optimization**: int8_hnsw reduces memory by ~4x with minimal recall loss. For large-scale deployments, this directly translates to fewer nodes and lower cost.

---

## References

- Elasticsearch kNN search: https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html
- Elasticsearch dense_vector: https://www.elastic.co/guide/en/elasticsearch/reference/current/dense-vector.html
- ELSER documentation: https://www.elastic.co/guide/en/elasticsearch/reference/current/semantic-search-elser.html
- Retriever API: https://www.elastic.co/guide/en/elasticsearch/reference/current/retrievers-overview.html
- Elasticsearch Python client: https://elasticsearch-py.readthedocs.io/
- RRF paper: Cormack, G.V., Clarke, C.L.A., and Buettcher, S. "Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods." SIGIR 2009.
