# On-Device and Edge AI Inference

## Overview

Edge inference runs models on the device — microcontrollers, mobile/PC NPUs,
or edge boxes — instead of calling a cloud API. The architecture is shaped by
tight memory, energy, and thermal budgets, and is chosen when latency, privacy,
offline operation, or per-unit cost demand it.

## Why on-device

- **Latency:** on-device inference returns in tens of milliseconds versus
  200–500 ms cloud round-trips — decisive for voice, AR, and control loops.
- **Privacy:** data never leaves the device.
- **Offline and cost:** works without connectivity and incurs no per-call cloud
  cost.

## Hardware tiers

| Tier | Silicon | Typical model |
|------|---------|---------------|
| MCU / TinyML | Cortex-M + tiny NPU (sub-$1 class) | KB–MB models: keyword spotting, anomaly detection |
| Mobile / AI-PC | Phone or laptop NPU (tens of TOPS) | Quantized 3–8B LLMs, vision |
| Edge box | Jetson Orin/Thor, Coral, Hailo (40+ TOPS) | 7–13B LLMs, multi-camera vision |

## Architectural levers

- **Quantization** is the key enabler: FP16 → INT8 → INT4 trades accuracy for
  memory, throughput, and energy. Most edge LLMs run INT4/INT8; validate the
  accuracy loss against the task.
- **Model selection** is sized to the accelerator's **memory bandwidth**, not
  just its TOPS; at batch size 1 the KV-cache dominates LLM memory.
- **Runtime:** TFLite/LiteRT, ONNX Runtime, ExecuTorch, llama.cpp/Ollama, or a
  vendor SDK (TensorRT on Jetson) — chosen to target the specific accelerator.
- **Budgets:** state the TOPS, RAM, and energy-per-inference budget up front;
  they bound every other decision.

## Guidance

Choose on-device for hard latency, offline, privacy, or per-unit-cost
requirements; cloud serving for large models, variable load, and centralized
updates; and a **hybrid** design (small local model with cloud escalation) when
you need both.
