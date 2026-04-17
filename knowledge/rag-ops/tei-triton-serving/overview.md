# TEI and Triton Serving -- Architecture for Self-Hosted Embeddings

## Overview

Text Embeddings Inference (TEI) is HuggingFace's high-performance serving runtime for embedding and reranking models. NVIDIA Triton Inference Server is a general-purpose model serving platform that supports multiple backends (ONNX Runtime, TensorRT, PyTorch, TensorFlow). Together, they form the backbone of self-hosted embedding infrastructure for RAG systems that cannot rely on external APIs due to cost, latency, data privacy, or compliance requirements.

This guide covers the architecture of both systems, when to use each, and how they compare to API-based embedding services.

---

## Why Self-Host Embeddings

### Cost at Scale

| Approach | Cost per 1M embeddings (1536-dim) | Monthly cost at 100M embeddings |
|----------|-----------------------------------|---------------------------------|
| OpenAI `text-embedding-3-small` | $0.02 | $2,000 |
| OpenAI `text-embedding-3-large` | $0.13 | $13,000 |
| Voyage `voyage-3` | $0.06 | $6,000 |
| Self-hosted (A10G GPU) | ~$0.005 (amortized) | ~$500 + GPU cost |
| Self-hosted (CPU, 8 cores) | ~$0.01 (amortized) | ~$300 + compute cost |

At 100M+ embeddings per month (common for large RAG systems with frequent re-indexing), self-hosting saves 80-95% compared to API providers.

### Other Reasons to Self-Host

- **Data privacy**: Embedding requests contain raw text. Self-hosting keeps data within your network.
- **Latency**: Local inference eliminates network round trips (2-10ms vs 50-200ms per request).
- **No rate limits**: API providers throttle at high volumes. Self-hosted servers handle your full throughput.
- **Offline operation**: Self-hosted embedding works without internet connectivity.

---

## HuggingFace Text Embeddings Inference (TEI)

### Architecture

TEI is a Rust-based server optimized specifically for embedding and reranking models:

```
                    +-------------------+
                    |   TEI Server      |
                    |   (Rust runtime)  |
                    +--------+----------+
                             |
              +--------------+--------------+
              |              |              |
    +---------v---+  +-------v-----+  +----v-------+
    | Token       |  | Batching    |  | Model      |
    | Router      |  | Engine      |  | Backend    |
    | (queue +    |  | (dynamic    |  | (ONNX /    |
    |  priority)  |  |  batching)  |  |  PyTorch / |
    +-------------+  +-------------+  |  Candle)   |
                                      +------------+
```

Key architectural features:

- **Token-based routing**: Routes requests to the most efficient batch based on sequence length
- **Dynamic batching**: Groups incoming requests into optimal batches (configurable max batch size and max waiting time)
- **Multiple backends**: Candle (Rust, default), ONNX Runtime, PyTorch
- **Automatic padding optimization**: Pads sequences to the longest in the batch, not the model max length

### Supported Model Types

| Type | Purpose | Example Models |
|------|---------|----------------|
| Embedding | Dense vector encoding | `BAAI/bge-large-en-v1.5`, `sentence-transformers/all-MiniLM-L6-v2` |
| Reranking | Cross-encoder scoring | `BAAI/bge-reranker-v2-m3`, `cross-encoder/ms-marco-MiniLM-L-12-v2` |
| Sequence Classification | Text classification | Any HF classifier |
| SPLADE | Sparse embedding | `naver/splade-cocondenser-ensembledistil` |

### Basic Docker Deployment

```bash
# Run TEI with a BGE embedding model
docker run --gpus all -p 8080:80 \
  -v $PWD/data:/data \
  ghcr.io/huggingface/text-embeddings-inference:latest \
  --model-id BAAI/bge-large-en-v1.5 \
  --port 80

# CPU-only deployment
docker run -p 8080:80 \
  -v $PWD/data:/data \
  ghcr.io/huggingface/text-embeddings-inference:cpu-latest \
  --model-id BAAI/bge-small-en-v1.5 \
  --port 80
```

### API Usage

```python
import httpx

TEI_URL = "http://localhost:8080"

# Single embedding
response = httpx.post(
    f"{TEI_URL}/embed",
    json={"inputs": "What is semantic search?"},
)
embedding = response.json()  # list of floats

# Batch embeddings
response = httpx.post(
    f"{TEI_URL}/embed",
    json={
        "inputs": [
            "First document to embed",
            "Second document to embed",
            "Third document to embed",
        ]
    },
)
embeddings = response.json()  # list of list of floats

# Reranking
response = httpx.post(
    f"{TEI_URL}/rerank",
    json={
        "query": "What is semantic search?",
        "texts": [
            "Semantic search uses embeddings to find similar content",
            "Python is a programming language",
            "Vector databases store high-dimensional embeddings",
        ],
    },
)
rerank_scores = response.json()  # list of {index, score}

# OpenAI-compatible endpoint
response = httpx.post(
    f"{TEI_URL}/v1/embeddings",
    json={
        "input": ["Text to embed"],
        "model": "BAAI/bge-large-en-v1.5",
    },
)
result = response.json()
embedding = result["data"][0]["embedding"]
```

---

## NVIDIA Triton Inference Server

### Architecture

Triton is a model-agnostic serving platform:

```
                    +-------------------+
                    |   Triton Server   |
                    |   (C++ runtime)   |
                    +--------+----------+
                             |
              +--------------+--------------+
              |              |              |
    +---------v---+  +-------v-----+  +----v-------+
    | HTTP/gRPC   |  | Scheduler   |  | Model      |
    | Frontend    |  | (dynamic    |  | Repository |
    |             |  |  batcher,   |  | (versioned)|
    +-------------+  |  sequence   |  +-----+------+
                     |  batcher)   |        |
                     +------+------+  +-----v------+
                            |         | Backends:  |
                     +------v------+  | - ONNX RT  |
                     | Instance    |  | - TensorRT |
                     | Manager     |  | - PyTorch  |
                     | (GPU/CPU    |  | - Custom   |
                     |  allocation)|  +------------+
                     +-------------+
```

Key features:

- **Multi-model serving**: Host multiple models on the same GPU with shared memory
- **Model ensembles**: Chain models (e.g., tokenizer -> embedding -> post-process) in a pipeline
- **Dynamic batching**: Configurable batch sizes and queue timeouts
- **Instance groups**: Multiple model instances per GPU, or spread across GPUs
- **Model versioning**: Serve multiple versions simultaneously for A/B testing
- **gRPC and HTTP**: Both protocols supported for client communication

### Model Repository Structure

```
model_repository/
  embedding_model/
    config.pbtxt          # model configuration
    1/                    # version 1
      model.onnx          # ONNX model file
    2/                    # version 2 (optional)
      model.onnx
  tokenizer/
    config.pbtxt
    1/
      model.py            # Python backend for tokenization
  ensemble/
    config.pbtxt          # pipeline: tokenizer -> embedding -> pooling
    1/
      (empty)             # ensemble has no model files
```

### Configuration Example

```protobuf
# model_repository/embedding_model/config.pbtxt
name: "embedding_model"
platform: "onnxruntime_onnx"
max_batch_size: 64

input [
  {
    name: "input_ids"
    data_type: TYPE_INT64
    dims: [-1]  # variable sequence length
  },
  {
    name: "attention_mask"
    data_type: TYPE_INT64
    dims: [-1]
  }
]

output [
  {
    name: "embeddings"
    data_type: TYPE_FP32
    dims: [-1]  # embedding dimension
  }
]

instance_group [
  {
    count: 2           # 2 instances per GPU
    kind: KIND_GPU
    gpus: [0]
  }
]

dynamic_batching {
  preferred_batch_size: [8, 16, 32]
  max_queue_delay_microseconds: 100000  # 100ms max wait
}

optimization {
  execution_accelerators {
    gpu_execution_accelerator: [
      {
        name: "tensorrt"
        parameters {
          key: "precision_mode"
          value: "FP16"
        }
      }
    ]
  }
}
```

### Docker Deployment

```bash
# Start Triton with a model repository
docker run --gpus all -p 8000:8000 -p 8001:8001 -p 8002:8002 \
  -v $PWD/model_repository:/models \
  nvcr.io/nvidia/tritonserver:24.05-py3 \
  tritonserver --model-repository=/models

# Ports:
# 8000 - HTTP inference
# 8001 - gRPC inference
# 8002 - Metrics (Prometheus)
```

---

## TEI vs. Triton: When to Use Each

| Aspect | TEI | Triton |
|--------|-----|--------|
| Setup complexity | Low (one command) | Medium (config files, model repo) |
| Embedding models | Best (purpose-built) | Good (with ONNX export) |
| Multi-model serving | One model per instance | Multiple models, shared GPU |
| Model types | Embedding, reranking, SPLADE | Any (vision, NLP, speech, custom) |
| Batching | Automatic, optimized for text | Configurable, general-purpose |
| Backend options | Candle, ONNX | ONNX, TensorRT, PyTorch, custom |
| OpenAI API compatibility | Built-in | Requires custom frontend |
| Monitoring | Basic metrics | Prometheus + detailed metrics |
| GPU memory efficiency | Good | Better (multi-model sharing) |
| Throughput (embedding) | Higher (specialized) | Slightly lower (general overhead) |

**Use TEI when**: You only need embedding/reranking serving and want minimal setup.

**Use Triton when**: You serve multiple model types, need TensorRT optimization, or require advanced scheduling features.

---

## Model Optimization

### ONNX Export

```python
from sentence_transformers import SentenceTransformer
from optimum.onnxruntime import ORTModelForFeatureExtraction
from transformers import AutoTokenizer

model_name = "BAAI/bge-large-en-v1.5"
output_dir = "./onnx_model"

# Export to ONNX
ort_model = ORTModelForFeatureExtraction.from_pretrained(
    model_name,
    export=True,
)
tokenizer = AutoTokenizer.from_pretrained(model_name)

ort_model.save_pretrained(output_dir)
tokenizer.save_pretrained(output_dir)
```

### FP16 Quantization

```python
from optimum.onnxruntime import ORTQuantizer, ORTModelForFeatureExtraction
from optimum.onnxruntime.configuration import AutoQuantizationConfig

model_path = "./onnx_model"
output_path = "./onnx_model_fp16"

# Quantize to FP16
quantizer = ORTQuantizer.from_pretrained(model_path)
qconfig = AutoQuantizationConfig.avx512_vnni(is_static=False, per_channel=True)

quantizer.quantize(
    save_dir=output_path,
    quantization_config=qconfig,
)
```

---

## Production Configuration

### TEI Production Setup

```bash
docker run --gpus all -p 8080:80 \
  -v $PWD/data:/data \
  --restart unless-stopped \
  --name tei-embeddings \
  ghcr.io/huggingface/text-embeddings-inference:latest \
  --model-id BAAI/bge-large-en-v1.5 \
  --max-batch-tokens 16384 \
  --max-concurrent-requests 512 \
  --max-client-batch-size 128 \
  --port 80
```

### Health Monitoring

```python
import httpx


def check_tei_health(url: str = "http://localhost:8080") -> dict:
    """Check TEI server health and readiness."""
    health = httpx.get(f"{url}/health").json()
    info = httpx.get(f"{url}/info").json()
    return {
        "healthy": health.get("status") == "ok",
        "model": info.get("model_id"),
        "max_batch_size": info.get("max_client_batch_size"),
        "model_type": info.get("model_type"),
    }


status = check_tei_health()
print(f"TEI Status: {status}")
```

---

## Common Pitfalls

1. **GPU memory underestimation**: Large embedding models (BGE-large, E5-large) need 2-4 GB GPU memory. Running on a GPU with insufficient memory causes OOM crashes. Check model size before deployment.

2. **Not warming up the server**: First requests after startup are slow due to model loading and JIT compilation. Send a few dummy requests during startup health checks.

3. **Batch size too large**: Very large batches exhaust GPU memory. Start with `max-batch-tokens=8192` and increase gradually while monitoring memory.

4. **Mismatched tokenizers**: When using Triton with custom preprocessing, the tokenizer must exactly match the model's expected tokenization. Mismatches produce garbage embeddings.

5. **Ignoring sequence length**: TEI pads to the longest sequence in the batch. One very long input in a batch of short inputs wastes computation. Bin inputs by length for optimal throughput.

6. **Not setting resource limits**: In Kubernetes, always set memory and GPU limits. Without limits, a misbehaving model can starve other workloads.

---

## References

- TEI documentation: https://huggingface.co/docs/text-embeddings-inference/
- TEI GitHub: https://github.com/huggingface/text-embeddings-inference
- Triton documentation: https://docs.nvidia.com/deeplearning/triton-inference-server/
- ONNX Runtime: https://onnxruntime.ai/
- Optimum: https://huggingface.co/docs/optimum/
