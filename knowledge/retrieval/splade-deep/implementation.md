# SPLADE -- Implementation Guide

## Overview

This guide covers practical implementation of SPLADE sparse retrieval, from encoding documents with pretrained checkpoints to indexing and searching with production vector databases. All examples use Python with sentence-transformers, the Hugging Face transformers library, Qdrant, and Elasticsearch.

---

## Encoding with Pretrained SPLADE Models

### Using sentence-transformers

The sentence-transformers library provides built-in support for SPLADE models. This is the simplest way to get started:

```python
from sentence_transformers import SparseEncoder


def encode_with_sentence_transformers():
    """
    Encode queries and documents using sentence-transformers SPLADE support.
    Requires sentence-transformers >= 4.0 for SparseEncoder.
    """
    # Load the best general-purpose SPLADE model
    model = SparseEncoder("naver/splade-cocondenser-ensembledistil")

    # Encode documents -- produces sparse dictionaries
    documents = [
        "Python's asyncio library provides an event loop for concurrent I/O.",
        "Kubernetes manages container orchestration across cluster nodes.",
        "PostgreSQL supports JSONB columns for semi-structured data storage.",
        "React hooks like useState and useEffect manage component lifecycle.",
        "Redis provides in-memory data structures for caching and pub/sub.",
    ]

    # encode() returns a list of sparse representations
    # Each is a dict-like object: {token_id: weight}
    doc_embeddings = model.encode(documents, convert_to_sparse_tensor=True)

    # Encode a query
    query = "async programming in Python"
    query_embedding = model.encode([query], convert_to_sparse_tensor=True)

    print(f"Query non-zero entries: {query_embedding[0]._nnz()}")
    print(f"Doc non-zero entries (avg): "
          f"{sum(d._nnz() for d in doc_embeddings) / len(doc_embeddings):.0f}")

    return query_embedding, doc_embeddings


def decode_sparse_to_tokens(sparse_vec, tokenizer, top_k: int = 15):
    """
    Convert a SPLADE sparse vector back to human-readable tokens.
    Useful for debugging and understanding what the model expanded.
    """
    import torch

    if hasattr(sparse_vec, "to_dense"):
        dense = sparse_vec.to_dense()
    else:
        dense = sparse_vec

    top_indices = torch.topk(dense, k=min(top_k, dense.shape[-1])).indices
    top_values = torch.topk(dense, k=min(top_k, dense.shape[-1])).values

    terms = []
    for idx, val in zip(top_indices.squeeze().tolist(), top_values.squeeze().tolist()):
        if val > 0:
            token = tokenizer.decode([idx]).strip()
            terms.append((token, round(val, 3)))

    return sorted(terms, key=lambda x: x[1], reverse=True)
```

### Using transformers Directly (naver/splade-cocondenser)

For more control over encoding, use the model directly:

```python
import torch
from transformers import AutoModelForMaskedLM, AutoTokenizer


class SPLADEEncoder:
    """
    Production SPLADE encoder using naver/splade-cocondenser-ensembledistil.
    Supports batched encoding and GPU acceleration.
    """

    def __init__(
        self,
        model_name: str = "naver/splade-cocondenser-ensembledistil",
        device: str | None = None,
        max_length: int = 256,
    ):
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = AutoModelForMaskedLM.from_pretrained(model_name)
        self.max_length = max_length

        if device is None:
            device = "cuda" if torch.cuda.is_available() else "cpu"
        self.device = device
        self.model.to(self.device)
        self.model.eval()

    @torch.no_grad()
    def encode(
        self,
        texts: list[str],
        batch_size: int = 32,
    ) -> list[dict[int, float]]:
        """
        Encode a list of texts into sparse representations.
        Returns a list of {token_id: weight} dictionaries.
        """
        all_sparse = []

        for i in range(0, len(texts), batch_size):
            batch_texts = texts[i : i + batch_size]
            inputs = self.tokenizer(
                batch_texts,
                return_tensors="pt",
                padding=True,
                truncation=True,
                max_length=self.max_length,
            ).to(self.device)

            output = self.model(**inputs)
            logits = output.logits  # (B, seq_len, vocab_size)

            # Mask padding
            mask = inputs["attention_mask"].unsqueeze(-1)  # (B, seq_len, 1)
            logits = logits * mask + (~mask.bool()).float() * float("-inf")

            # Max-pool over sequence length, then log-saturate
            max_logits, _ = torch.max(logits, dim=1)  # (B, vocab_size)
            sparse_vecs = torch.log1p(torch.relu(max_logits))  # (B, vocab_size)

            # Convert to sparse dicts
            for vec in sparse_vecs:
                nonzero_indices = vec.nonzero(as_tuple=True)[0]
                sparse_dict = {}
                for idx in nonzero_indices:
                    idx_int = idx.item()
                    weight = vec[idx_int].item()
                    if weight > 0.0:
                        sparse_dict[idx_int] = weight
                all_sparse.append(sparse_dict)

        return all_sparse

    def encode_query(self, query: str) -> dict[int, float]:
        """Convenience method for single query encoding."""
        return self.encode([query])[0]

    def encode_documents(
        self,
        documents: list[str],
        batch_size: int = 32,
    ) -> list[dict[int, float]]:
        """Convenience method with document-appropriate batch size."""
        return self.encode(documents, batch_size=batch_size)

    def sparse_to_text(
        self,
        sparse_dict: dict[int, float],
        top_k: int = 20,
    ) -> list[tuple[str, float]]:
        """Decode sparse representation to human-readable (token, weight) pairs."""
        sorted_items = sorted(sparse_dict.items(), key=lambda x: x[1], reverse=True)
        result = []
        for token_id, weight in sorted_items[:top_k]:
            token = self.tokenizer.decode([token_id]).strip()
            result.append((token, round(weight, 3)))
        return result
```

---

## Indexing in Qdrant (Sparse Vectors)

Qdrant natively supports sparse vectors, making it an excellent choice for SPLADE deployment.

### Setting Up a Qdrant Collection

```python
from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance,
    NamedSparseVector,
    PointStruct,
    SparseIndexParams,
    SparseVector,
    SparseVectorParams,
    VectorParams,
)


def create_splade_collection(
    client: QdrantClient,
    collection_name: str = "documents",
):
    """
    Create a Qdrant collection configured for SPLADE sparse vectors.
    Optionally includes a dense vector field for hybrid search.
    """
    client.create_collection(
        collection_name=collection_name,
        vectors_config={},  # No dense vectors (add VectorParams here for hybrid)
        sparse_vectors_config={
            "splade": SparseVectorParams(
                index=SparseIndexParams(
                    on_disk=False,  # Keep in RAM for speed; True for large corpora
                ),
            ),
        },
    )
    print(f"Created collection '{collection_name}' with SPLADE sparse vectors")


def index_documents(
    client: QdrantClient,
    collection_name: str,
    encoder: "SPLADEEncoder",
    documents: list[dict],
    batch_size: int = 100,
):
    """
    Index documents with SPLADE sparse vectors in Qdrant.

    documents: list of {"id": int, "text": str, "metadata": dict}
    """
    texts = [doc["text"] for doc in documents]
    sparse_vecs = encoder.encode_documents(texts, batch_size=32)

    points = []
    for doc, svec in zip(documents, sparse_vecs):
        indices = list(svec.keys())
        values = list(svec.values())

        point = PointStruct(
            id=doc["id"],
            payload={
                "text": doc["text"],
                **doc.get("metadata", {}),
            },
            vector={
                "splade": SparseVector(
                    indices=indices,
                    values=values,
                ),
            },
        )
        points.append(point)

    # Upsert in batches
    for i in range(0, len(points), batch_size):
        batch = points[i : i + batch_size]
        client.upsert(collection_name=collection_name, points=batch)

    print(f"Indexed {len(points)} documents in '{collection_name}'")


def search_qdrant_splade(
    client: QdrantClient,
    collection_name: str,
    encoder: "SPLADEEncoder",
    query: str,
    top_k: int = 10,
) -> list[dict]:
    """
    Search Qdrant using SPLADE sparse vectors.
    """
    query_vec = encoder.encode_query(query)
    indices = list(query_vec.keys())
    values = list(query_vec.values())

    results = client.query_points(
        collection_name=collection_name,
        query=SparseVector(indices=indices, values=values),
        using="splade",
        limit=top_k,
        with_payload=True,
    )

    return [
        {
            "id": point.id,
            "score": point.score,
            "text": point.payload.get("text", ""),
            "metadata": {
                k: v for k, v in point.payload.items() if k != "text"
            },
        }
        for point in results.points
    ]
```

### Hybrid Search: SPLADE + Dense in Qdrant

```python
from qdrant_client.models import Prefetch, Query, FusionQuery, Fusion


def create_hybrid_collection(
    client: QdrantClient,
    collection_name: str = "hybrid_docs",
    dense_dim: int = 768,
):
    """
    Create a Qdrant collection with both dense and SPLADE sparse vectors.
    """
    client.create_collection(
        collection_name=collection_name,
        vectors_config={
            "dense": VectorParams(
                size=dense_dim,
                distance=Distance.COSINE,
            ),
        },
        sparse_vectors_config={
            "splade": SparseVectorParams(
                index=SparseIndexParams(on_disk=False),
            ),
        },
    )


def hybrid_search(
    client: QdrantClient,
    collection_name: str,
    dense_query: list[float],
    sparse_query: dict[int, float],
    top_k: int = 10,
) -> list[dict]:
    """
    Hybrid search using Reciprocal Rank Fusion (RRF) of dense + SPLADE.
    """
    sparse_indices = list(sparse_query.keys())
    sparse_values = list(sparse_query.values())

    results = client.query_points(
        collection_name=collection_name,
        prefetch=[
            Prefetch(
                query=dense_query,
                using="dense",
                limit=top_k * 2,
            ),
            Prefetch(
                query=SparseVector(
                    indices=sparse_indices,
                    values=sparse_values,
                ),
                using="splade",
                limit=top_k * 2,
            ),
        ],
        query=FusionQuery(fusion=Fusion.RRF),
        limit=top_k,
        with_payload=True,
    )

    return [
        {
            "id": point.id,
            "score": point.score,
            "text": point.payload.get("text", ""),
        }
        for point in results.points
    ]
```

---

## Indexing in Elasticsearch (sparse_vector)

Elasticsearch 8.x supports the `sparse_vector` field type, which can store SPLADE representations.

### Index Mapping

```python
from elasticsearch import Elasticsearch


def create_elasticsearch_index(
    es: Elasticsearch,
    index_name: str = "splade_documents",
):
    """
    Create an Elasticsearch index with a sparse_vector field for SPLADE.
    Requires Elasticsearch 8.11+ for sparse_vector field type.
    """
    mapping = {
        "mappings": {
            "properties": {
                "text": {"type": "text"},
                "splade_vector": {
                    "type": "sparse_vector",
                },
                "title": {"type": "keyword"},
                "category": {"type": "keyword"},
            },
        },
        "settings": {
            "number_of_shards": 1,
            "number_of_replicas": 0,
        },
    }

    es.indices.create(index=index_name, body=mapping)
    print(f"Created Elasticsearch index '{index_name}'")


def index_documents_elasticsearch(
    es: Elasticsearch,
    index_name: str,
    encoder: "SPLADEEncoder",
    documents: list[dict],
    batch_size: int = 500,
):
    """
    Index documents with SPLADE vectors in Elasticsearch.
    Uses bulk API for efficiency.
    """
    texts = [doc["text"] for doc in documents]
    sparse_vecs = encoder.encode_documents(texts, batch_size=32)

    actions = []
    for doc, svec in zip(documents, sparse_vecs):
        # Elasticsearch sparse_vector expects {token_string: weight}
        # Convert token IDs to actual token strings
        token_weights = {}
        for token_id, weight in svec.items():
            token = encoder.tokenizer.decode([token_id]).strip()
            if token and weight > 0.01:  # Filter very low weights
                token_weights[token] = round(weight, 4)

        actions.append({"index": {"_index": index_name, "_id": doc["id"]}})
        actions.append({
            "text": doc["text"],
            "splade_vector": token_weights,
            "title": doc.get("title", ""),
            "category": doc.get("category", ""),
        })

    # Bulk index
    for i in range(0, len(actions), batch_size * 2):
        batch = actions[i : i + batch_size * 2]
        es.bulk(body=batch, refresh=(i + batch_size * 2 >= len(actions)))

    print(f"Indexed {len(documents)} documents in '{index_name}'")


def search_elasticsearch_splade(
    es: Elasticsearch,
    index_name: str,
    encoder: "SPLADEEncoder",
    query: str,
    top_k: int = 10,
) -> list[dict]:
    """
    Search Elasticsearch using SPLADE sparse vectors.
    Uses the sparse_vector query type.
    """
    query_vec = encoder.encode_query(query)

    # Convert to token strings for Elasticsearch
    token_weights = {}
    for token_id, weight in query_vec.items():
        token = encoder.tokenizer.decode([token_id]).strip()
        if token and weight > 0.01:
            token_weights[token] = round(weight, 4)

    body = {
        "query": {
            "sparse_vector": {
                "field": "splade_vector",
                "query_vector": token_weights,
            },
        },
        "size": top_k,
    }

    response = es.search(index=index_name, body=body)

    return [
        {
            "id": hit["_id"],
            "score": hit["_score"],
            "text": hit["_source"]["text"],
        }
        for hit in response["hits"]["hits"]
    ]
```

---

## Batch Encoding Pipeline for Large Corpora

When encoding millions of documents, efficiency matters. Here is a production-ready pipeline:

```python
import json
import time
from pathlib import Path
from typing import Iterator

import torch
from transformers import AutoModelForMaskedLM, AutoTokenizer
from torch.utils.data import DataLoader, Dataset


class DocumentDataset(Dataset):
    """Dataset for batch encoding documents."""

    def __init__(
        self,
        documents: list[dict],
        tokenizer: AutoTokenizer,
        max_length: int = 256,
    ):
        self.documents = documents
        self.tokenizer = tokenizer
        self.max_length = max_length

    def __len__(self):
        return len(self.documents)

    def __getitem__(self, idx):
        doc = self.documents[idx]
        encoding = self.tokenizer(
            doc["text"],
            truncation=True,
            max_length=self.max_length,
            padding="max_length",
            return_tensors="pt",
        )
        return {
            "input_ids": encoding["input_ids"].squeeze(0),
            "attention_mask": encoding["attention_mask"].squeeze(0),
            "doc_id": doc["id"],
        }


def encode_corpus(
    documents: list[dict],
    model_name: str = "naver/splade-cocondenser-ensembledistil",
    batch_size: int = 64,
    output_path: str = "splade_embeddings.jsonl",
    device: str = "cuda",
    fp16: bool = True,
) -> Path:
    """
    Encode an entire corpus and save sparse vectors to JSONL.

    Each output line: {"id": ..., "indices": [...], "values": [...]}

    Performance on A100 (40GB):
    - ~1000 documents/second at batch_size=128, max_length=256
    - 1M documents in ~17 minutes
    """
    tokenizer = AutoTokenizer.from_pretrained(model_name)
    model = AutoModelForMaskedLM.from_pretrained(model_name).to(device)
    model.eval()

    if fp16:
        model.half()

    dataset = DocumentDataset(documents, tokenizer)
    dataloader = DataLoader(
        dataset,
        batch_size=batch_size,
        shuffle=False,
        num_workers=4,
        pin_memory=True,
    )

    output_file = Path(output_path)
    total_encoded = 0
    start_time = time.time()

    with open(output_file, "w") as f:
        for batch in dataloader:
            input_ids = batch["input_ids"].to(device)
            attention_mask = batch["attention_mask"].to(device)
            doc_ids = batch["doc_id"]

            with torch.no_grad(), torch.cuda.amp.autocast(enabled=fp16):
                output = model(input_ids=input_ids, attention_mask=attention_mask)
                logits = output.logits

                mask = attention_mask.unsqueeze(-1)
                logits = logits * mask + (~mask.bool()).float() * float("-inf")
                max_logits, _ = torch.max(logits, dim=1)
                sparse_vecs = torch.log1p(torch.relu(max_logits))

            # Write sparse representations
            for doc_id, vec in zip(doc_ids, sparse_vecs):
                nonzero = vec.nonzero(as_tuple=True)[0]
                indices = nonzero.cpu().tolist()
                values = vec[nonzero].cpu().float().tolist()

                # Filter very small values to save storage
                filtered = [
                    (idx, val) for idx, val in zip(indices, values) if val > 0.01
                ]
                if filtered:
                    idxs, vals = zip(*filtered)
                else:
                    idxs, vals = [], []

                f.write(json.dumps({
                    "id": doc_id if isinstance(doc_id, (int, str)) else doc_id.item(),
                    "indices": list(idxs),
                    "values": [round(v, 4) for v in vals],
                }) + "\n")

            total_encoded += len(doc_ids)
            if total_encoded % 10000 == 0:
                elapsed = time.time() - start_time
                rate = total_encoded / elapsed
                print(f"Encoded {total_encoded} docs | {rate:.0f} docs/sec")

    elapsed = time.time() - start_time
    print(
        f"Done: {total_encoded} documents in {elapsed:.1f}s "
        f"({total_encoded / elapsed:.0f} docs/sec)"
    )
    return output_file
```

---

## Weight Quantization for Production

Raw float32 SPLADE weights are wasteful. Quantization reduces storage by 2-4x with negligible quality loss:

```python
import numpy as np


def quantize_sparse_vector(
    sparse_dict: dict[int, float],
    method: str = "float16",
    min_weight: float = 0.01,
) -> dict[int, float]:
    """
    Quantize SPLADE sparse vector weights to reduce storage.

    Methods:
    - float16: 50% storage reduction, virtually no quality loss
    - int8: 75% storage reduction, < 0.5% quality loss on most benchmarks
    - binary: 87% storage reduction (weights become 0 or 1), ~2-5% quality loss
    """
    # Filter very small weights first
    filtered = {k: v for k, v in sparse_dict.items() if v > min_weight}

    if method == "float16":
        return {
            k: float(np.float16(v))
            for k, v in filtered.items()
        }

    elif method == "int8":
        if not filtered:
            return {}
        max_val = max(filtered.values())
        return {
            k: round(v / max_val * 127) / 127 * max_val
            for k, v in filtered.items()
        }

    elif method == "binary":
        threshold = np.median(list(filtered.values()))
        return {
            k: 1.0
            for k, v in filtered.items()
            if v >= threshold
        }

    else:
        raise ValueError(f"Unknown quantization method: {method}")


def estimate_storage(
    sparse_vecs: list[dict[int, float]],
    method: str = "float32",
) -> dict[str, float]:
    """
    Estimate storage requirements for a set of sparse vectors.
    """
    total_entries = sum(len(v) for v in sparse_vecs)
    avg_entries = total_entries / len(sparse_vecs) if sparse_vecs else 0

    bytes_per_entry = {
        "float32": 4 + 4,    # 4 bytes index + 4 bytes value
        "float16": 4 + 2,    # 4 bytes index + 2 bytes value
        "int8": 4 + 1,       # 4 bytes index + 1 byte value
        "binary": 4,         # 4 bytes index only (value is always 1)
    }

    bpe = bytes_per_entry.get(method, 8)
    total_bytes = total_entries * bpe
    avg_bytes = avg_entries * bpe

    return {
        "total_entries": total_entries,
        "avg_entries_per_doc": round(avg_entries, 1),
        "bytes_per_doc": round(avg_bytes, 1),
        "total_mb": round(total_bytes / 1024 / 1024, 2),
        "method": method,
    }
```

---

## End-to-End Example

```python
def main():
    """
    Complete example: encode, index in Qdrant, and search.
    """
    from qdrant_client import QdrantClient

    # 1. Initialize encoder
    encoder = SPLADEEncoder(
        model_name="naver/splade-cocondenser-ensembledistil",
        device="cuda",
    )

    # 2. Prepare documents
    documents = [
        {"id": 1, "text": "Python asyncio provides cooperative multitasking through coroutines and event loops."},
        {"id": 2, "text": "Kubernetes horizontal pod autoscaler adjusts replicas based on CPU utilization."},
        {"id": 3, "text": "PostgreSQL VACUUM reclaims storage from dead tuples after UPDATE and DELETE."},
        {"id": 4, "text": "React concurrent mode enables interruptible rendering for smoother user interfaces."},
        {"id": 5, "text": "Redis Cluster partitions data across multiple nodes using hash slots."},
        {"id": 6, "text": "SQLAlchemy async sessions use greenlet to bridge sync ORM with asyncio drivers."},
        {"id": 7, "text": "Celery workers process distributed task queues with configurable concurrency."},
        {"id": 8, "text": "FastAPI dependency injection resolves request-scoped resources automatically."},
    ]

    # 3. Set up Qdrant
    client = QdrantClient(url="http://localhost:6333")
    create_splade_collection(client, "demo_splade")

    # 4. Index
    index_documents(client, "demo_splade", encoder, documents)

    # 5. Search
    queries = [
        "async programming Python",
        "database maintenance and cleanup",
        "container scaling and load",
    ]

    for query in queries:
        results = search_qdrant_splade(client, "demo_splade", encoder, query, top_k=3)
        print(f"\nQuery: '{query}'")
        for r in results:
            print(f"  [{r['score']:.3f}] {r['text'][:80]}...")

        # Debug: show query expansion
        qvec = encoder.encode_query(query)
        terms = encoder.sparse_to_text(qvec, top_k=10)
        print(f"  Expanded terms: {terms}")


if __name__ == "__main__":
    main()
```

---

## Performance Tuning

### Encoding Speed

| Setting | Docs/sec (A100) | Docs/sec (T4) | Notes |
|---|---|---|---|
| batch_size=32, fp32 | ~600 | ~150 | Safe default |
| batch_size=64, fp16 | ~1200 | ~280 | Best throughput/memory tradeoff |
| batch_size=128, fp16 | ~1400 | OOM | Requires >= 24GB VRAM |
| ONNX Runtime, fp16 | ~1800 | ~400 | 30-50% faster than PyTorch |

### Index Size (per 1M documents, avg 50 non-zero entries)

| Storage Method | Index Size | Query Latency (p99) |
|---|---|---|
| float32 (raw) | ~400 MB | ~15ms |
| float16 | ~300 MB | ~15ms |
| int8 | ~250 MB | ~18ms |
| On-disk (Qdrant) | ~400 MB disk | ~25-40ms |

### Qdrant Configuration Tips

1. **Set `on_disk=False`** for collections under 5GB -- RAM-resident sparse indexes are significantly faster
2. **Use `batch_size=100`** for upserts -- Qdrant performs better with moderate batch sizes
3. **Enable WAL** for durability if you cannot afford re-indexing on crash
4. **Shard large collections** (>10M docs) across multiple Qdrant nodes

### Elasticsearch Configuration Tips

1. **Set `number_of_replicas=0`** during bulk indexing, then increase after
2. **Use `_bulk` API** with batches of 500-1000 documents
3. **Refresh interval**: set `refresh_interval=-1` during indexing, reset to `1s` after
4. **Heap size**: allocate 50% of RAM (up to 31GB) to JVM heap for large sparse_vector indexes

---

## Common Pitfalls

1. **Not filtering small weights before indexing**: SPLADE produces many entries with weights < 0.01 that add index size without improving quality. Always apply a minimum weight threshold.

2. **Using CPU for encoding at scale**: encoding 1M documents on CPU takes ~10 hours vs ~15 minutes on GPU. Always use GPU for batch encoding.

3. **Forgetting token ID to string conversion for Elasticsearch**: Qdrant accepts integer token IDs natively, but Elasticsearch sparse_vector requires string keys. Use the tokenizer to convert.

4. **Not benchmarking hybrid vs. SPLADE-only**: on some datasets, SPLADE alone matches or exceeds hybrid BM25+dense. Test before adding complexity.

5. **Ignoring max_length truncation**: documents longer than 256 tokens get truncated. For long documents, consider chunking with overlap (128-token stride) and max-pooling the chunk representations.

---

## References

- sentence-transformers SPLADE documentation: https://www.sbert.net/docs/sparse_encoder/usage/usage.html
- Qdrant sparse vectors guide: https://qdrant.tech/documentation/concepts/vectors/#sparse-vectors
- Elasticsearch sparse_vector field type: https://www.elastic.co/guide/en/elasticsearch/reference/current/sparse-vector.html
- naver/splade-cocondenser-ensembledistil: https://huggingface.co/naver/splade-cocondenser-ensembledistil
