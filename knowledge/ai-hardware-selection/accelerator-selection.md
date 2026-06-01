# Selecting Accelerators for AI Workloads

## Overview

Choosing AI hardware is usually framed around raw FLOPS, but for LLM inference
the binding constraints are memory and bandwidth. Size the memory first, then
choose the accelerator family, then validate cost per token at the target
latency.

## Memory first

For inference, weights plus KV-cache must fit in accelerator memory, and decode
is memory-bandwidth-bound. Estimate:

```
weights ≈ params × bytes/param
  e.g. 70B params × 2 bytes (FP16) ≈ 140 GB  -> multi-GPU or quantize
KV-cache grows with context length × batch size (add it on top)
```

Only after memory and bandwidth fit does raw compute (TOPS/FLOPS) become the
deciding factor.

## Accelerator families

| Type | Strength | Use |
|------|----------|-----|
| GPU (NVIDIA H/B-series, AMD MI) | Flexible, huge ecosystem, HBM | Training and inference — the default |
| TPU | Dense matmul, pod-scale interconnect | Large-scale training/inference on GCP |
| NPU | Performance per watt at low power | Edge, mobile, AI-PC inference |
| FPGA | Custom low-latency dataflow | Niche fixed ultra-low-latency pipelines |
| CPU | Always available, fine for small/batch | Small models, embeddings, light load |

## Other levers

- **Interconnect** (NVLink, InfiniBand) bounds multi-accelerator scaling and is
  decisive for training and tensor parallelism.
- **Precision support** (FP8, INT4) multiplies effective throughput and capacity.
- **Cost per token (or request) at the target latency** is the honest metric —
  it folds in utilization, power/cooling, and cloud-versus-owned TCO.
- **Training versus inference:** training needs FLOPS, interconnect, and memory;
  inference needs memory capacity/bandwidth and low latency.

## Guidance

Default to NVIDIA GPUs sized by model VRAM for flexibility and training; use NPUs
for low-power edge inference; TPU pods for hyperscale training on GCP; and FPGAs
only for a justified fixed ultra-low-latency pipeline.
