# Real-Time Scheduling and Schedulability

## Overview

A real-time system is correct only if it produces the right result **at the
right time**. Architecture therefore centers on guaranteeing deadlines, not
maximizing average throughput. The inputs are each task's period (T), worst-case
execution time (C), and deadline (D).

## Hard, firm, soft

- **Hard:** a missed deadline is a system failure (flight control, airbag).
- **Firm:** a late result is useless but not catastrophic.
- **Soft:** late results degrade quality (media playback).

The classification sets how much pessimism (and certification) the design needs.

## Scheduling algorithms

- **Rate-Monotonic (RMS):** fixed priorities assigned by rate (shorter period =
  higher priority). A set of n tasks is schedulable if
  `Σ (Cᵢ/Tᵢ) ≤ n(2^(1/n) − 1)` (≈0.69 as n→∞). Simple, predictable, widely
  certified.
- **Earliest-Deadline-First (EDF):** dynamic priority by nearest deadline.
  Optimal for a single processor and schedulable up to 100% utilization, but
  behaves worse under overload and is harder to certify.

## The hazards

- **WCET (worst-case execution time)** is the foundation everything rests on, and
  it is hard: caches, branch prediction, DMA contention, and shared buses make
  the worst case far above the average. Measure and analyze; budget pessimism.
- **Priority inversion:** a high-priority task blocks on a resource held by a
  low-priority task that is itself preempted by a medium task. The Mars
  Pathfinder resets were this bug. Fix with **priority inheritance** or
  **priority ceiling** protocols on shared mutexes.
- **Interrupt latency:** keep ISRs short (top-half) and defer work to tasks;
  bound the longest interrupts-disabled window.

## Architectural posture

Prefer determinism over throughput: static allocation (no heap fragmentation),
bounded loops, no non-deterministic libraries on hot paths, and a scheduler whose
worst case you can prove. On tiny MCUs a bare-metal superloop plus ISRs may beat
an RTOS; as task count and connectivity grow, an RTOS (FreeRTOS, Zephyr,
ThreadX) earns its footprint.
