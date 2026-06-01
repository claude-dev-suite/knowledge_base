# Hybrid Edge-Cloud AI: Local-First with Escalation

## Overview

A hybrid design pairs a small, fast local model with a large, capable cloud
model, getting low latency, privacy, and cost for the common case while
preserving cloud quality for hard cases. It fits when neither pure-edge nor
pure-cloud is right on its own.

## Patterns

- **Local-first with cloud escalation:** run a small on-device model and escalate
  to the cloud only when needed — low confidence, long context, or a hard query.
  Most requests stay local (fast, cheap, private); the difficult minority gets
  cloud quality.
- **Model cascade:** a cheap model runs first; if its confidence is below a
  threshold, a larger model runs, and so on. Tune thresholds to a cost/quality
  target (this also applies entirely within the cloud).
- **Speculative / draft-verify:** a small model drafts tokens and a large model
  verifies them — primarily a latency optimization that composes with the above.
- **Split computation:** preprocessing or feature extraction on the device,
  heavy inference in the cloud (classic for vision and audio).

## Decision drivers

- **Escalation trigger quality** is the make-or-break: confidence score, input
  complexity/length, task type, or explicit user action.
- **Privacy boundary:** decide what may leave the device — sometimes only
  embeddings or redacted text escalate.
- **Connectivity:** if the system must degrade gracefully offline, the local
  model is the floor.
- **Cost model:** the fraction of traffic that escalates times the cloud cost,
  versus the local hardware cost.

## Failure and consistency

Define behavior when the cloud is unreachable — serve the local result and flag
it, queue, or refuse — and avoid silent quality cliffs. Cache cloud results on
the device for repeated queries.

## Guidance

Hybrid suits voice assistants, laptop/phone copilots, field/IoT devices with
intermittent connectivity, and privacy-sensitive apps with occasional hard
queries. If essentially all traffic needs the big model, just use cloud serving;
if essentially none does, go pure edge.
