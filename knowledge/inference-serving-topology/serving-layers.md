# LLM Inference Serving Topology

## Overview

Production model serving is organized into three layers. Naming which layer you
are designing keeps the conversation precise and the system composable.

```
+-------------------------------------------------------------+
| Orchestration:  Kubernetes + KEDA / Ray Serve / llm-d       |  scale, health, placement
+-------------------------------------------------------------+
| Serving:        Triton / KServe / LiteLLM / Envoy AI GW     |  routing, batching policy, API, metrics
+-------------------------------------------------------------+
| Engine:         vLLM / SGLang / TensorRT-LLM                |  KV-cache, continuous batching, the model
+-------------------------------------------------------------+
```

## The layers

- **Engine** executes the model on accelerators and owns the paged **KV-cache**,
  **continuous (in-flight) batching**, and quantization. This is where
  throughput and latency are won.
- **Serving** handles request routing, batching policy, the API contract,
  metrics, and rate limiting. A common 2026 pairing is **vLLM as the token
  engine wrapped by Triton as the production shell**.
- **Orchestration** handles autoscaling, health, and placement; **Ray Serve** is
  used when you need distributed strategies across many GPUs/nodes.

## Levers that decide the topology

- **KV-cache** is the LLM memory bottleneck → paged KV (vLLM), cache reuse, and
  quantized KV extend max batch and context.
- **Continuous batching** (versus static) is essential for throughput.
- **Prefill–decode disaggregation** separates the compute-bound prefill from the
  memory-bound decode onto different pools to improve utilization at scale.
- **Parallelism** — tensor, pipeline, expert (MoE), data-parallel attention — is
  chosen by model size relative to GPU memory.
- **SLO targets:** state the time-to-first-token (low hundreds of ms) and
  inter-token latency (tens of ms) goals; they drive batching and parallelism.

## Scale ladder

Single GPU + vLLM → multi-GPU single node → Ray Serve/KServe multi-node with
KEDA autoscaling → disaggregated prefill/decode and multi-region. Do not climb a
tier without a load or latency reason.
