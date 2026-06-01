# Security Architecture and Threat Modeling

## Overview

Security architecture designs protection into the shape of a system rather than
bolting it on afterward. It begins with a threat model and is guided by a few
durable principles, then reaches for platform and hardware mechanisms where the
threat justifies them.

## Threat modeling

1. **What are we protecting?** Assets and data classifications.
2. **From whom?** Threat actors and their capabilities.
3. **Trust boundaries.** Draw them: wherever data or control crosses between
   differently-trusted components. Each crossing is where you authenticate,
   authorize, validate input, and encrypt.
4. **Enumerate threats per boundary** with STRIDE — Spoofing, Tampering,
   Repudiation, Information disclosure, Denial of service, Elevation of
   privilege — rank by risk, and design a mitigation for each.

Produce a trust-boundary diagram plus a threat/mitigation table as part of the
architecture decision record.

## Principles

- **Defense in depth:** independent layers so one failure is not catastrophic.
- **Least privilege / least authority:** minimal scope per component;
  capability-based rather than ambient authority where possible.
- **Minimize attack surface and TCB:** fewer entry points and a smaller trusted
  base — which favors microkernel and microVM isolation.
- **Fail securely:** deny by default; errors must not open access.
- **Secure by design and by default:** safe defaults, no secrets in code,
  encryption in transit and at rest, no hand-rolled crypto.

## Zero-trust

Drop implicit network trust. Authenticate and authorize **every** request based
on identity, device posture, and policy regardless of network location;
micro-segment; assume breach. This replaces the "hard perimeter, soft interior"
model.

## Platform and hardware security

- **TEE / enclaves** (SGX, TrustZone, SEV-SNP, TDX) isolate and seal sensitive
  computation from the OS or host — for untrusted-host and confidential-computing
  scenarios.
- **Secure boot / chain of trust:** each boot stage verifies the next from a
  hardware root of trust; measured boot plus attestation establishes remote
  trust.
- **Key management:** HSM/KMS, rotation, and envelope encryption.

## When to invoke

New system or platform designs, multi-tenant or untrusted-input systems,
regulated data, and anything internet-facing or handling secrets/PII.
