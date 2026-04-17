# OpenSearch k-NN Deep Guide

## Overview

OpenSearch k-NN is a plugin that adds approximate nearest neighbor (ANN) search to OpenSearch. It supports three backend engines (Lucene, Faiss, nmslib), multiple algorithms (HNSW, IVF, IVF+PQ), radial search, neural search pipelines for inline model inference, and hybrid search combining BM25 with k-NN scoring. As of OpenSearch 2.13+, Lucene is the default engine and includes byte vector support, while Faiss provides billion-scale capabilities with product quantization. This guide covers the k-NN architecture, engine selection, index configuration, search patterns, and hybrid retrieval.

For basic OpenSearch operations (indices, mappings, queries), see the companion OpenSearch guide. This document focuses exclusively on k-NN vector search capabilities.

---

## Architecture

### k-NN Plugin Components

```
OpenSearch Node
+------------------------------------------+
|  OpenSearch Core                          |
|  +--------------------------------------+|
|  |  k-NN Plugin                         ||
|  |  - KNNVectorFieldMapper              ||
|  |  - KNNQueryBuilder                   ||
|  |  - NativeMemoryManager               ||
|  |  +----------------------------------+||
|  |  | Engine Layer                      |||
|  |  |  - Lucene (default, 2.13+)       |||
|  |  |  - Faiss  (billion-scale)        |||
|  |  |  - nmslib (legacy)               |||
|  |  +----------------------------------+||
|  +--------------------------------------+|
|                                           |
|  JVM Heap        Off-Heap / Native Memory |
|  (metadata)      (vector graphs/indexes)  |
+------------------------------------------+
```

### Engine Comparison

| Feature | Lucene | Faiss | nmslib |
|---|---|---|---|
| Default since | 2.13+ | - | Pre-2.13 |
| Algorithm | HNSW | HNSW, IVF, IVF+PQ | HNSW |
| Memory location | JVM heap | Native (off-heap) | Native (off-heap) |
| Max dimensions | 16,384 | 16,384 | 16,384 |
| Product quantization | No | Yes | No |
| Byte vectors (int8) | Yes (2.13+) | Yes | No |
| On-disk mode | Via OS page cache | IVF+PQ (partial) | No |
| Filter integration | Deep integration | Approximate | Approximate |
| Segment merging | Automatic | Automatic | Manual |
| Concurrent segment search | Yes (2.12+) | Yes | Yes |

### How k-NN Indexes Work

Each OpenSearch segment contains a separate HNSW graph (or IVF index for Faiss). When segments merge, the k-NN indexes merge as well. This means:

1. **Small segments = many small graphs**: frequent indexing without force merges creates many small HNSW graphs, each searched independently, then results merged.
2. **Force merge = one large graph**: `_forcemerge?max_num_segments=1` creates a single optimized graph per shard, improving search performance.
3. **Memory usage**: HNSW graphs live in native memory (Faiss/nmslib) or JVM heap (Lucene). OpenSearch manages loading/evicting graphs via a circuit breaker.

---

## Index Mappings

### Lucene Engine (Default, Recommended)

```json
PUT /documents
{
  "settings": {
    "index": {
      "knn": true,
      "number_of_shards": 3,
      "number_of_replicas": 1
    }
  },
  "mappings": {
    "properties": {
      "embedding": {
        "type": "knn_vector",
        "dimension": 1536,
        "method": {
          "engine": "lucene",
          "space_type": "cosinesimil",
          "name": "hnsw",
          "parameters": {
            "m": 16,
            "ef_construction": 128,
            "ef_search": 100
          }
        }
      },
      "content": {
        "type": "text",
        "analyzer": "standard"
      },
      "metadata": {
        "type": "object",
        "properties": {
          "category": { "type": "keyword" },
          "tenant_id": { "type": "keyword" },
          "created_at": { "type": "date" }
        }
      }
    }
  }
}
```

### Faiss Engine (Billion-Scale)

```json
PUT /large_documents
{
  "settings": {
    "index": {
      "knn": true,
      "knn.algo_param.ef_search": 100,
      "number_of_shards": 6,
      "number_of_replicas": 1
    }
  },
  "mappings": {
    "properties": {
      "embedding": {
        "type": "knn_vector",
        "dimension": 1536,
        "method": {
          "engine": "faiss",
          "space_type": "innerproduct",
          "name": "hnsw",
          "parameters": {
            "m": 32,
            "ef_construction": 256,
            "ef_search": 128,
            "encoder": {
              "name": "pq",
              "parameters": {
                "code_size": 48,
                "m": 48
              }
            }
          }
        }
      },
      "content": { "type": "text" },
      "metadata": {
        "properties": {
          "category": { "type": "keyword" }
        }
      }
    }
  }
}
```

### Faiss IVF+PQ (Maximum Compression)

```json
PUT /compressed_documents
{
  "settings": {
    "index": {
      "knn": true
    }
  },
  "mappings": {
    "properties": {
      "embedding": {
        "type": "knn_vector",
        "dimension": 768,
        "method": {
          "engine": "faiss",
          "space_type": "l2",
          "name": "ivf",
          "parameters": {
            "nlist": 1024,
            "nprobes": 32,
            "encoder": {
              "name": "pq",
              "parameters": {
                "code_size": 96,
                "m": 96
              }
            }
          }
        }
      }
    }
  }
}
```

### Space Types

| Space Type | Distance | Best For |
|---|---|---|
| `cosinesimil` | 1 - cosine_distance (similarity) | Text embeddings (not pre-normalized) |
| `l2` | Squared L2 distance | When magnitude matters |
| `innerproduct` | Inner product (negative for ordering) | Pre-normalized embeddings |
| `l1` | L1 / Manhattan distance | Sparse-like vectors |
| `linf` | L-infinity / Chebyshev | Special use cases |

---

## HNSW Parameters

### Build-Time Parameters

| Parameter | Default | Range | Effect |
|---|---|---|---|
| `m` | 16 | 2-100 | Connections per node. Higher = better recall, more memory |
| `ef_construction` | 100 | 2-2000 | Build beam width. Higher = better graph quality, slower build |

### Search-Time Parameters

| Parameter | Default | Range | Effect |
|---|---|---|---|
| `ef_search` | 100 | 2-2000 | Query beam width. Higher = better recall, slower search |

```
# Set ef_search at index level
PUT /documents/_settings
{
  "index": {
    "knn.algo_param.ef_search": 150
  }
}

# Or per-query via k-NN query parameter
GET /documents/_search
{
  "query": {
    "knn": {
      "embedding": {
        "vector": [0.1, 0.2, ...],
        "k": 10,
        "method_parameters": {
          "ef_search": 200
        }
      }
    }
  }
}
```

### Tuning Guidelines

| Dataset Size | m | ef_construction | ef_search | Notes |
|---|---|---|---|---|
| < 100K | 16 | 100 | 50-100 | Defaults work well |
| 100K - 1M | 16-24 | 128-200 | 100-150 | Moderate tuning |
| 1M - 10M | 24-32 | 200-400 | 150-300 | Memory becomes important |
| 10M - 100M | 32-48 | 256-512 | 200-400 | Consider Faiss with PQ |
| 100M+ | Use Faiss IVF+PQ | - | - | HNSW memory prohibitive |

---

## Basic k-NN Search

### Standard k-NN Query

```python
from opensearchpy import OpenSearch

client = OpenSearch(
    hosts=[{"host": "localhost", "port": 9200}],
    http_auth=("admin", "admin"),
    use_ssl=True,
    verify_certs=False
)

query_vector = [0.1, 0.2, 0.3]  # Your 1536-dim vector

response = client.search(
    index="documents",
    body={
        "size": 10,
        "query": {
            "knn": {
                "embedding": {
                    "vector": query_vector,
                    "k": 10
                }
            }
        },
        "_source": ["content", "metadata"]
    }
)

for hit in response["hits"]["hits"]:
    print(f"Score: {hit['_score']:.4f} - {hit['_source']['content'][:80]}")
```

### Filtered k-NN Search

```python
# Pre-filtered k-NN (Lucene engine applies filter during graph traversal)
response = client.search(
    index="documents",
    body={
        "size": 10,
        "query": {
            "knn": {
                "embedding": {
                    "vector": query_vector,
                    "k": 10,
                    "filter": {
                        "bool": {
                            "must": [
                                {"term": {"metadata.tenant_id": "tenant_123"}},
                                {"range": {"metadata.created_at": {"gte": "2024-01-01"}}}
                            ]
                        }
                    }
                }
            }
        }
    }
)
```

**Filter strategies by engine**:

| Engine | Filter Behavior | Mechanism |
|---|---|---|
| Lucene | Efficient pre-filtering | Integrated into HNSW traversal |
| Faiss | Post-filtering (approximate) | Search first, then filter results |
| nmslib | Post-filtering (approximate) | Search first, then filter results |

With Lucene, if the filter is highly selective (matches < 10% of documents), the search automatically falls back to exact k-NN on the filtered subset.

### Radial Search

Radial search returns all vectors within a distance threshold rather than top-K.

```python
# Find all documents within cosine distance 0.3 (similarity > 0.7)
response = client.search(
    index="documents",
    body={
        "query": {
            "knn": {
                "embedding": {
                    "vector": query_vector,
                    "max_distance": 0.3
                }
            }
        }
    }
)

# Or using minimum score
response = client.search(
    index="documents",
    body={
        "query": {
            "knn": {
                "embedding": {
                    "vector": query_vector,
                    "min_score": 0.7
                }
            }
        }
    }
)
```

---

## Neural Search Pipeline

OpenSearch's neural search pipeline provides inline model inference, allowing you to send text queries and have them automatically embedded by a model deployed in OpenSearch ML framework or connected via a remote connector.

### Setup with Remote Model (OpenAI)

```json
// Step 1: Create a connector to OpenAI
POST /_plugins/_ml/connectors/_create
{
  "name": "OpenAI Embedding Connector",
  "description": "Connector to OpenAI embedding API",
  "version": "1.0",
  "protocol": "http",
  "parameters": {
    "model": "text-embedding-3-small"
  },
  "credential": {
    "openAI_key": "sk-..."
  },
  "actions": [
    {
      "action_type": "predict",
      "method": "POST",
      "url": "https://api.openai.com/v1/embeddings",
      "headers": {
        "Authorization": "Bearer ${credential.openAI_key}"
      },
      "request_body": "{ \"input\": ${parameters.input}, \"model\": \"${parameters.model}\" }",
      "pre_process_function": "connector.pre_process.openai.embedding",
      "post_process_function": "connector.post_process.openai.embedding"
    }
  ]
}

// Step 2: Register and deploy the model
POST /_plugins/_ml/models/_register
{
  "name": "openai-text-embedding-3-small",
  "function_name": "remote",
  "connector_id": "<connector_id>",
  "description": "OpenAI text-embedding-3-small via connector"
}

POST /_plugins/_ml/models/<model_id>/_deploy

// Step 3: Create a search pipeline with neural query enricher
PUT /_search/pipeline/neural_pipeline
{
  "request_processors": [
    {
      "neural_query_enricher": {
        "default_model_id": "<model_id>",
        "neural_field_default_id": {
          "embedding": "<model_id>"
        }
      }
    }
  ]
}

// Step 4: Create index with default pipeline
PUT /documents
{
  "settings": {
    "index": {
      "knn": true,
      "default_pipeline": "neural_pipeline"
    }
  },
  "mappings": {
    "properties": {
      "content": { "type": "text" },
      "embedding": {
        "type": "knn_vector",
        "dimension": 1536,
        "method": {
          "engine": "lucene",
          "space_type": "cosinesimil",
          "name": "hnsw"
        }
      }
    }
  }
}
```

### Neural Search Query

```python
# Text query -- automatically embedded by the neural pipeline
response = client.search(
    index="documents",
    body={
        "query": {
            "neural": {
                "embedding": {
                    "query_text": "What is retrieval augmented generation?",
                    "model_id": "<model_id>",
                    "k": 10
                }
            }
        }
    }
)
```

---

## Hybrid Search (BM25 + k-NN)

### Weighted Sum Approach

OpenSearch supports combining BM25 text scores with k-NN vector scores using a normalization processor.

```json
// Create a search pipeline with score normalization
PUT /_search/pipeline/hybrid_pipeline
{
  "phase_results_processors": [
    {
      "normalization-processor": {
        "normalization": {
          "technique": "min_max"
        },
        "combination": {
          "technique": "arithmetic_mean",
          "parameters": {
            "weights": [0.3, 0.7]
          }
        }
      }
    }
  ]
}
```

```python
# Hybrid query combining BM25 and k-NN
response = client.search(
    index="documents",
    params={"search_pipeline": "hybrid_pipeline"},
    body={
        "size": 10,
        "query": {
            "hybrid": {
                "queries": [
                    {
                        "match": {
                            "content": {
                                "query": "machine learning transformers"
                            }
                        }
                    },
                    {
                        "knn": {
                            "embedding": {
                                "vector": query_vector,
                                "k": 20
                            }
                        }
                    }
                ]
            }
        }
    }
)
```

### Normalization Techniques

| Technique | Description | When to Use |
|---|---|---|
| `min_max` | Scale scores to [0, 1] based on min/max in results | Default, works well for most cases |
| `l2` | L2 normalization of score vectors | When score distributions are similar |

### Combination Techniques

| Technique | Description | When to Use |
|---|---|---|
| `arithmetic_mean` | Weighted average of normalized scores | Default, intuitive weight tuning |
| `geometric_mean` | Geometric mean (penalizes zeros) | When both signals must be present |
| `harmonic_mean` | Harmonic mean (closer to the smaller value) | When minimum signal quality matters |

---

## Script Scoring for Custom Logic

Script scoring allows custom distance functions when the built-in metrics are insufficient.

```python
# Custom scoring: combine vector similarity with metadata-based boost
response = client.search(
    index="documents",
    body={
        "size": 10,
        "query": {
            "script_score": {
                "query": {
                    "bool": {
                        "filter": {
                            "term": {"metadata.category": "engineering"}
                        }
                    }
                },
                "script": {
                    "source": """
                        float score = cosineSimilarity(params.query_vector, doc['embedding']);
                        float recency_boost = 1.0;
                        if (doc['metadata.created_at'].size() > 0) {
                            long age_days = (new Date().getTime() - doc['metadata.created_at'].value.toInstant().toEpochMilli()) / 86400000;
                            recency_boost = 1.0 / (1.0 + age_days / 30.0);
                        }
                        return score * 0.8 + recency_boost * 0.2;
                    """,
                    "params": {
                        "query_vector": query_vector
                    }
                }
            }
        }
    }
)
```

### Available Script Functions

| Function | Description |
|---|---|
| `cosineSimilarity(params.v, doc['field'])` | Cosine similarity (returns [-1, 1]) |
| `l2Squared(params.v, doc['field'])` | Squared L2 distance |
| `l1Norm(params.v, doc['field'])` | L1 / Manhattan distance |
| `dotProduct(params.v, doc['field'])` | Dot product |

**Performance note**: script scoring performs exact (brute-force) computation on the filtered document set. It does not use the HNSW index. Use it only with a selective pre-filter that narrows the candidate set to thousands, not millions.

---

## OpenSearch 2.13+ Features

### Byte Vectors (int8)

```json
PUT /byte_documents
{
  "mappings": {
    "properties": {
      "embedding": {
        "type": "knn_vector",
        "dimension": 1536,
        "data_type": "byte",
        "method": {
          "engine": "lucene",
          "space_type": "cosinesimil",
          "name": "hnsw"
        }
      }
    }
  }
}
```

Byte vectors store each dimension as an int8 (range: -128 to 127), reducing storage by 4x compared to float32. Embeddings must be quantized before indexing.

```python
import numpy as np

def quantize_to_byte(embedding: list[float]) -> list[int]:
    """Quantize float32 embedding to int8 range."""
    arr = np.array(embedding, dtype=np.float32)
    # Scale to [-128, 127] range
    max_abs = np.abs(arr).max()
    if max_abs > 0:
        scaled = arr / max_abs * 127
    else:
        scaled = arr
    return scaled.astype(np.int8).tolist()

# Index byte vectors
byte_vector = quantize_to_byte(float_embedding)
client.index(
    index="byte_documents",
    body={
        "embedding": byte_vector,
        "content": "Document text"
    }
)
```

### Concurrent Segment Search (2.12+)

```json
PUT /documents/_settings
{
  "index": {
    "search.concurrent_segment_search.enabled": true
  }
}
```

This enables parallel search across segments within a shard, improving latency for indices with many segments.

### k-NN Model Cache Warming

```json
PUT /documents/_settings
{
  "index": {
    "knn.model.cache.warm": true
  }
}
```

Pre-loads HNSW graphs into memory when a shard is opened, avoiding cold-start latency.

---

## Memory Estimation

### HNSW Memory Formula

```
Memory (bytes) = num_vectors * (dimension * bytes_per_dim + M * 2 * 4 + overhead)

Where:
  bytes_per_dim = 4 (float32) or 1 (byte)
  M = HNSW connections per node
  overhead ~ 48 bytes per vector (metadata)
```

### Practical Examples

| Vectors | Dimensions | Type | M | Estimated Memory |
|---|---|---|---|---|
| 1M | 1536 | float32 | 16 | ~6.3 GB |
| 1M | 1536 | byte | 16 | ~1.7 GB |
| 5M | 1536 | float32 | 16 | ~31.5 GB |
| 10M | 1536 | float32 | 24 | ~68 GB |
| 10M | 768 | float32 | 16 | ~34 GB |
| 50M | 1536 | float32 | 16 | ~315 GB |
| 10M | 1536 | float32 + PQ(48) | 16 | ~5 GB (Faiss IVF+PQ) |

### Memory Circuit Breaker

```json
// Set k-NN memory limit (percentage of JVM heap)
PUT /_cluster/settings
{
  "persistent": {
    "knn.memory.circuit_breaker.limit": "60%"
  }
}

// Monitor k-NN memory usage
GET /_plugins/_knn/stats
```

---

## Common Pitfalls

1. **Not enabling `index.knn: true`**: the `knn_vector` field type requires `index.knn: true` in index settings. Without it, vectors are stored but not indexed for ANN search.

2. **Using nmslib for new projects**: nmslib is legacy. Use Lucene (default) for most cases or Faiss for billion-scale with product quantization.

3. **Ignoring the memory circuit breaker**: HNSW graphs consume native memory (Faiss/nmslib) or heap (Lucene). If graphs exceed the circuit breaker limit, queries fail with `503 Service Unavailable`.

4. **Not force-merging after bulk indexing**: many small segments mean many small HNSW graphs, each searched independently. After bulk indexing, run `POST /index/_forcemerge?max_num_segments=1` to optimize.

5. **Post-filtering surprise with Faiss**: Faiss engine applies filters **after** the ANN search. If 90% of your data is filtered out, you may get far fewer than `k` results. Over-request (set `k` to 5-10x your target) or use Lucene engine for filtered workloads.

6. **Not accounting for replicas in memory**: each replica holds its own copy of the HNSW graph. With 3 replicas and 10M vectors, you need 4x the memory (1 primary + 3 replicas).

7. **Setting ef_search too low for hybrid search**: hybrid queries combine BM25 and k-NN scores. If k-NN recall is poor (low ef_search), the hybrid result quality suffers even with perfect BM25 scores.

---

## References

- OpenSearch k-NN plugin: https://opensearch.org/docs/latest/search-plugins/knn/
- OpenSearch neural search: https://opensearch.org/docs/latest/search-plugins/neural-search/
- OpenSearch hybrid search: https://opensearch.org/docs/latest/search-plugins/hybrid-search/
- Faiss documentation: https://github.com/facebookresearch/faiss
- nmslib: https://github.com/nmslib/nmslib
- OpenSearch ML framework: https://opensearch.org/docs/latest/ml-commons-plugin/
