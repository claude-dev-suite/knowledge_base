# TEI and Triton -- Deployment: Docker, ONNX, FP16, and Batching Configuration

## Overview

This guide covers production deployment of embedding serving infrastructure using TEI and Triton. It walks through Docker Compose configurations, ONNX model optimization, FP16 quantization, batching tuning, Kubernetes deployment, and load balancing.

---

## TEI Docker Deployment

### Docker Compose for TEI

```yaml
# docker-compose.yml
version: "3.8"

services:
  tei-embeddings:
    image: ghcr.io/huggingface/text-embeddings-inference:latest
    ports:
      - "8080:80"
    volumes:
      - ./model-cache:/data
    environment:
      - HUGGING_FACE_HUB_TOKEN=${HF_TOKEN}
    command:
      - --model-id=BAAI/bge-large-en-v1.5
      - --port=80
      - --max-batch-tokens=16384
      - --max-concurrent-requests=512
      - --max-client-batch-size=128
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  tei-reranker:
    image: ghcr.io/huggingface/text-embeddings-inference:latest
    ports:
      - "8081:80"
    volumes:
      - ./model-cache:/data
    command:
      - --model-id=BAAI/bge-reranker-v2-m3
      - --port=80
      - --max-batch-tokens=8192
      - --max-concurrent-requests=256
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: unless-stopped
```

### CPU-Only Deployment

```yaml
# docker-compose.cpu.yml
version: "3.8"

services:
  tei-embeddings-cpu:
    image: ghcr.io/huggingface/text-embeddings-inference:cpu-latest
    ports:
      - "8080:80"
    volumes:
      - ./model-cache:/data
    command:
      - --model-id=BAAI/bge-small-en-v1.5
      - --port=80
      - --max-batch-tokens=8192
      - --max-concurrent-requests=128
      - --max-client-batch-size=64
    deploy:
      resources:
        limits:
          cpus: "8"
          memory: 4G
    restart: unless-stopped
```

---

## ONNX Model Optimization

### Export a HuggingFace Model to ONNX

```python
from optimum.onnxruntime import ORTModelForFeatureExtraction
from transformers import AutoTokenizer
from pathlib import Path

model_name = "BAAI/bge-large-en-v1.5"
output_dir = Path("./onnx_models/bge-large-en-v1.5")
output_dir.mkdir(parents=True, exist_ok=True)

# Export to ONNX
model = ORTModelForFeatureExtraction.from_pretrained(
    model_name,
    export=True,
    provider="CUDAExecutionProvider",  # or "CPUExecutionProvider"
)
tokenizer = AutoTokenizer.from_pretrained(model_name)

model.save_pretrained(output_dir)
tokenizer.save_pretrained(output_dir)

print(f"ONNX model exported to: {output_dir}")
print(f"Model size: {sum(f.stat().st_size for f in output_dir.glob('*.onnx')) / 1e6:.1f} MB")
```

### FP16 Quantization

```python
from optimum.onnxruntime import ORTOptimizer, ORTModelForFeatureExtraction
from optimum.onnxruntime.configuration import OptimizationConfig

model_path = "./onnx_models/bge-large-en-v1.5"
optimized_path = "./onnx_models/bge-large-en-v1.5-fp16"

# Load the ONNX model
model = ORTModelForFeatureExtraction.from_pretrained(model_path)

# Create optimizer
optimizer = ORTOptimizer.from_pretrained(model_path)

# Optimize with FP16
optimization_config = OptimizationConfig(
    optimization_level=99,       # maximum optimization
    fp16=True,                   # convert to FP16
    optimize_for_gpu=True,       # GPU-specific optimizations
)

optimizer.optimize(
    save_dir=optimized_path,
    optimization_config=optimization_config,
)

print(f"Original model: {sum(f.stat().st_size for f in Path(model_path).glob('*.onnx')) / 1e6:.1f} MB")
print(f"Optimized model: {sum(f.stat().st_size for f in Path(optimized_path).glob('*.onnx')) / 1e6:.1f} MB")
```

### INT8 Quantization (CPU Deployment)

```python
from optimum.onnxruntime import ORTQuantizer
from optimum.onnxruntime.configuration import AutoQuantizationConfig

model_path = "./onnx_models/bge-large-en-v1.5"
quantized_path = "./onnx_models/bge-large-en-v1.5-int8"

quantizer = ORTQuantizer.from_pretrained(model_path)

# Dynamic INT8 quantization (no calibration data needed)
qconfig = AutoQuantizationConfig.avx512_vnni(
    is_static=False,
    per_channel=True,
)

quantizer.quantize(
    save_dir=quantized_path,
    quantization_config=qconfig,
)

# INT8 models are ~4x smaller and ~2-3x faster on CPU
```

### Model Size Comparison

| Model | Format | Size | GPU Throughput | CPU Throughput |
|-------|--------|------|----------------|----------------|
| `bge-large-en-v1.5` | PyTorch FP32 | 1.3 GB | 1x (baseline) | 1x (baseline) |
| `bge-large-en-v1.5` | ONNX FP32 | 1.3 GB | 1.2x | 1.5x |
| `bge-large-en-v1.5` | ONNX FP16 | 650 MB | 1.8x | N/A (GPU only) |
| `bge-large-en-v1.5` | ONNX INT8 | 325 MB | N/A | 2.5x |
| `bge-small-en-v1.5` | ONNX FP16 | 65 MB | 4x | N/A |
| `bge-small-en-v1.5` | ONNX INT8 | 33 MB | N/A | 5x |

---

## Batching Configuration

### TEI Batching Parameters

```bash
docker run --gpus all -p 8080:80 \
  ghcr.io/huggingface/text-embeddings-inference:latest \
  --model-id BAAI/bge-large-en-v1.5 \
  --max-batch-tokens 16384 \       # max total tokens in a batch
  --max-concurrent-requests 512 \  # max requests in flight
  --max-client-batch-size 128 \    # max items per client request
  --auto-truncate \                # truncate inputs exceeding max_length
  --port 80
```

### Understanding TEI Batching

```
Client sends: ["text1" (50 tokens), "text2" (100 tokens), "text3" (200 tokens)]

TEI batching with max-batch-tokens=16384:
  Batch 1: ["text1", "text2", "text3"]  # 350 tokens total, fits in budget
  -> Padded to max length in batch (200 tokens)
  -> Effective tokens processed: 3 * 200 = 600

If client sends 100 texts averaging 100 tokens each:
  Total tokens needed: 100 * 100 = 10,000 < 16,384
  -> All 100 texts fit in one batch
  -> Padded to longest text in batch
```

### Optimal Batch Size Selection

```python
def estimate_optimal_batch_params(
    model_name: str,
    gpu_memory_gb: float,
    avg_sequence_length: int = 128,
    model_hidden_size: int = 1024,
    model_layers: int = 24,
) -> dict:
    """Estimate optimal TEI batching parameters for a given GPU."""
    # Approximate model memory (FP16)
    params_estimate = model_hidden_size * model_hidden_size * model_layers * 4  # rough
    model_memory_gb = params_estimate / 1e9 * 2  # FP16 = 2 bytes per param

    # Available memory for batching
    available_gb = gpu_memory_gb - model_memory_gb - 0.5  # 0.5 GB overhead

    # Estimate memory per token (activation memory)
    bytes_per_token = model_hidden_size * model_layers * 2 * 4  # rough estimate, FP16
    gb_per_token = bytes_per_token / 1e9

    # Max batch tokens
    max_batch_tokens = int(available_gb / gb_per_token)

    # Conservative: use 70% of theoretical max
    recommended_batch_tokens = int(max_batch_tokens * 0.7)

    return {
        "model_memory_gb": round(model_memory_gb, 1),
        "available_memory_gb": round(available_gb, 1),
        "max_batch_tokens": max_batch_tokens,
        "recommended_batch_tokens": recommended_batch_tokens,
        "max_concurrent_at_avg_length": recommended_batch_tokens // avg_sequence_length,
    }


# Example: BGE-large on A10G (24GB)
params = estimate_optimal_batch_params(
    model_name="bge-large",
    gpu_memory_gb=24,
    avg_sequence_length=128,
    model_hidden_size=1024,
    model_layers=24,
)
print(f"Recommended max_batch_tokens: {params['recommended_batch_tokens']}")
print(f"Max concurrent requests at avg 128 tokens: {params['max_concurrent_at_avg_length']}")
```

---

## Triton Deployment with ONNX

### Prepare Model Repository

```python
import shutil
from pathlib import Path


def prepare_triton_model_repo(
    onnx_model_path: str,
    repo_path: str,
    model_name: str = "embedding_model",
    max_batch_size: int = 64,
):
    """Create a Triton model repository from an ONNX model."""
    repo = Path(repo_path)
    model_dir = repo / model_name / "1"
    model_dir.mkdir(parents=True, exist_ok=True)

    # Copy ONNX model
    onnx_files = list(Path(onnx_model_path).glob("*.onnx"))
    for f in onnx_files:
        shutil.copy2(f, model_dir / "model.onnx")

    # Write config
    config = f"""name: "{model_name}"
platform: "onnxruntime_onnx"
max_batch_size: {max_batch_size}

input [
  {{
    name: "input_ids"
    data_type: TYPE_INT64
    dims: [-1]
  }},
  {{
    name: "attention_mask"
    data_type: TYPE_INT64
    dims: [-1]
  }},
  {{
    name: "token_type_ids"
    data_type: TYPE_INT64
    dims: [-1]
  }}
]

output [
  {{
    name: "last_hidden_state"
    data_type: TYPE_FP32
    dims: [-1, -1]
  }}
]

instance_group [
  {{
    count: 2
    kind: KIND_GPU
    gpus: [0]
  }}
]

dynamic_batching {{
  preferred_batch_size: [8, 16, 32, {max_batch_size}]
  max_queue_delay_microseconds: 50000
}}
"""
    with open(repo / model_name / "config.pbtxt", "w") as f:
        f.write(config)

    print(f"Model repository created at: {repo}")


prepare_triton_model_repo(
    onnx_model_path="./onnx_models/bge-large-en-v1.5-fp16",
    repo_path="./triton_models",
)
```

### Triton Docker Compose

```yaml
version: "3.8"

services:
  triton:
    image: nvcr.io/nvidia/tritonserver:24.05-py3
    ports:
      - "8000:8000"   # HTTP
      - "8001:8001"   # gRPC
      - "8002:8002"   # Metrics
    volumes:
      - ./triton_models:/models
    command: tritonserver --model-repository=/models --log-verbose=1
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/v2/health/ready"]
      interval: 30s
      timeout: 10s
      retries: 5
```

### Triton Client

```python
import tritonclient.http as httpclient
import numpy as np
from transformers import AutoTokenizer

# Initialize client
client = httpclient.InferenceServerClient(url="localhost:8000")

# Check server and model health
print(f"Server ready: {client.is_server_ready()}")
print(f"Model ready: {client.is_model_ready('embedding_model')}")

# Tokenize input
tokenizer = AutoTokenizer.from_pretrained("BAAI/bge-large-en-v1.5")
encoded = tokenizer(
    ["What is semantic search?", "Vector databases store embeddings"],
    padding=True,
    truncation=True,
    max_length=512,
    return_tensors="np",
)

# Create inference request
inputs = [
    httpclient.InferInput("input_ids", encoded["input_ids"].shape, "INT64"),
    httpclient.InferInput("attention_mask", encoded["attention_mask"].shape, "INT64"),
    httpclient.InferInput("token_type_ids", encoded["token_type_ids"].shape, "INT64"),
]

inputs[0].set_data_from_numpy(encoded["input_ids"].astype(np.int64))
inputs[1].set_data_from_numpy(encoded["attention_mask"].astype(np.int64))
inputs[2].set_data_from_numpy(encoded["token_type_ids"].astype(np.int64))

outputs = [httpclient.InferRequestedOutput("last_hidden_state")]

# Run inference
result = client.infer("embedding_model", inputs, outputs=outputs)
hidden_states = result.as_numpy("last_hidden_state")

# Mean pooling
attention_mask = encoded["attention_mask"]
mask_expanded = np.expand_dims(attention_mask, axis=-1)
sum_embeddings = np.sum(hidden_states * mask_expanded, axis=1)
sum_mask = np.clip(np.sum(mask_expanded, axis=1), a_min=1e-9, a_max=None)
embeddings = sum_embeddings / sum_mask

# Normalize
norms = np.linalg.norm(embeddings, axis=1, keepdims=True)
embeddings = embeddings / norms

print(f"Embedding shape: {embeddings.shape}")
print(f"First embedding (first 5 dims): {embeddings[0][:5]}")
```

---

## Kubernetes Deployment

### TEI on Kubernetes

```yaml
# tei-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tei-embeddings
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tei-embeddings
  template:
    metadata:
      labels:
        app: tei-embeddings
    spec:
      containers:
        - name: tei
          image: ghcr.io/huggingface/text-embeddings-inference:latest
          args:
            - --model-id=BAAI/bge-large-en-v1.5
            - --port=80
            - --max-batch-tokens=16384
            - --max-concurrent-requests=512
          ports:
            - containerPort: 80
          resources:
            limits:
              nvidia.com/gpu: 1
              memory: 8Gi
            requests:
              nvidia.com/gpu: 1
              memory: 4Gi
          livenessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 60
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 30
            periodSeconds: 10
          volumeMounts:
            - name: model-cache
              mountPath: /data
      volumes:
        - name: model-cache
          persistentVolumeClaim:
            claimName: model-cache-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: tei-embeddings
spec:
  selector:
    app: tei-embeddings
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

---

## Common Pitfalls

1. **Not pre-downloading models**: First startup downloads the model from HuggingFace Hub, which can take minutes. In Kubernetes, this causes readiness probe failures. Pre-download models into a PVC or bake them into the Docker image.

2. **GPU memory overcommitment**: Running TEI and other GPU workloads on the same GPU without memory limits causes OOM kills. Always set GPU memory limits and monitor usage.

3. **Ignoring sequence length distribution**: If most inputs are 50 tokens but `max_length` is 512, padding wastes 90% of computation. Analyze your input distribution and set `max_length` accordingly.

4. **Not using gRPC for high throughput**: HTTP adds overhead per request. For >1000 req/sec, use Triton's gRPC endpoint which is 2-3x more efficient.

5. **Missing health checks in production**: Without readiness probes, load balancers route traffic to uninitialized servers. Always configure health/readiness checks with appropriate initial delays.

6. **Not caching model downloads**: Each container restart re-downloads the model without a persistent volume. Mount a PVC or use `--volume` to cache models across restarts.

---

## References

- TEI documentation: https://huggingface.co/docs/text-embeddings-inference/
- Triton documentation: https://docs.nvidia.com/deeplearning/triton-inference-server/
- ONNX Runtime optimization: https://onnxruntime.ai/docs/performance/
- Optimum: https://huggingface.co/docs/optimum/onnxruntime/usage_guides/optimization
