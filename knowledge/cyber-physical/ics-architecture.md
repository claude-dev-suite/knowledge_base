# Cyber-Physical / ICS-SCADA Architecture (general)

## Overview

A cyber-physical system couples computation with physical processes through
sensors and actuators in closed control loops, where a software fault can have
physical and safety consequences. This article generalizes the product-specific
industrial control material (DCS, IEC 61131, ISA formats) into engine-agnostic
architecture guidance.

## The Purdue reference model

Industrial control architecture is organized into levels, and the boundary
between IT and OT (operational technology) is the critical trust boundary.

```
L5/L4  Enterprise / MES / ERP                         (IT)
-------------------- IT/OT boundary (DMZ) --------------------
L3     Operations, historians, engineering workstations
L2     SCADA / HMI supervisory control
L1     Controllers: PLC / RTU / DCS
L0     Field devices: sensors, actuators, drives  (physical process)
```

## Defining constraints

- **Determinism and real time:** control loops have hard cycle times; jitter
  degrades control quality. This pulls in real-time scheduling concerns.
- **Safety:** safety-instrumented systems are kept independent from basic
  control, with fail-safe or fail-operational design, redundancy (e.g. triple
  modular redundancy), and standards such as IEC 61508/61511.
- **Availability over confidentiality:** unlike IT, OT prioritizes uptime and
  safety. You cannot freely patch or reboot a running plant, which shapes how
  updates and security controls are applied.

## IT/OT convergence and OT security

The traditional air gap is largely a myth: plants are connected for analytics
and remote operations. Design for connectivity rather than assuming isolation.

- **IEC 62443** is the reference standard for industrial security.
- Apply **zone-and-conduit segmentation**, an IT/OT DMZ, and least privilege;
  increasingly, **zero-trust principles for OT** authenticate every cross-zone
  flow.
- Legacy field devices often have weak or no authentication. Compensate with
  network segmentation, protocol-aware firewalls, and monitoring rather than
  assuming endpoint security.

## Guidance

Treat the IT/OT boundary as the primary trust boundary and segment by Purdue
level. Keep safety functions independent and redundant. Design for determinism
and availability first, and assume connectivity — secure it per IEC 62443 rather
than relying on isolation.
