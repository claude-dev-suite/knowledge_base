# Game-Engine Architecture (engine-agnostic)

## Overview

This article generalizes the Unity-specific game development material into
engine-agnostic architecture guidance. For Unity APIs and DOTS specifics, see
the dedicated gamedev documentation.

## The game loop

The engine's heartbeat processes input, updates the simulation, and renders. The
key decision is the timestep:

- Use a **fixed timestep** for the simulation to get deterministic physics and
  stable netcode, and **decouple** it from a variable render rate using
  interpolation. This is standard for anything requiring determinism
  (multiplayer, physics-driven gameplay).
- A purely variable timestep is simpler but makes physics and netcode
  non-deterministic.

## Data model: ECS vs scene graph

| | OOP / scene graph | ECS (entity-component-system) |
|---|---|---|
| Memory layout | Objects with inheritance | Data in contiguous component arrays |
| Cache behavior | Pointer-chasing, poor | Data-oriented, cache-friendly, SIMD-able |
| Logic | Methods on objects | Systems iterate over components |
| Best fit | Smaller/simpler games, tooling | Many entities, performance, parallelism |

ECS is the modern default for performance and parallelism (it aligns with
hardware-aware, data-oriented design); OOP scene graphs remain fine for simpler
titles.

## Render pipeline

- **Forward** shading (per object) versus **deferred** (G-buffer, shade per
  pixel — handles many lights) versus forward+/clustered variants. Choose by
  light count and platform; mobile and bandwidth-limited targets favor
  forward/tiled.
- A **render graph / frame graph** declares passes and their resource
  dependencies so the engine can schedule work and alias GPU memory — the modern
  way to structure rendering.

## Core subsystems

Physics, audio, animation, scene and asset streaming, and a deliberate **memory
strategy** (pools, arenas, and frame allocators rather than per-frame
allocation, for determinism and no GC stalls). Keep subsystems decoupled behind
clear interfaces.

## Netcode

- **Lockstep:** exchange only inputs and simulate deterministically on every
  client — tiny bandwidth, requires full determinism (RTS games).
- **Client prediction with server reconciliation:** responsive with an
  authoritative server (FPS games).
- **Rollback:** predict and re-simulate on correction (fighting games).

Authority and determinism (a fixed timestep) are the cross-cutting requirements
for all of these.

## Guidance

Prefer an off-the-shelf engine (Unity, Unreal, Godot) unless there is a specific
reason to build one. For a custom engine: a fixed-timestep loop, ECS for
entity-heavy or performance-critical games, a render graph, arena-based memory,
and a netcode model matched to the genre.
