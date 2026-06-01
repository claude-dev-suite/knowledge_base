# Virtualization and Isolation Models

## Overview

Isolation is a spectrum, and stronger isolation costs density, performance, and
boot time. The architectural decision is choosing the **weakest boundary that is
strong enough for the workload's trust level**.

## The spectrum

```
weaker / denser / faster  <------------------------------>  stronger / heavier
 containers   gVisor      microVM (Firecracker/Kata)    full VM (KVM/Xen/ESXi)
 shared kernel  syscall    own guest kernel, tiny        own kernel + full
 + namespaces   intercept  device model                  device model
```

| Boundary | Isolation | Density / overhead | Boot | Fits |
|----------|-----------|--------------------|------|------|
| Process + namespaces/cgroups (containers) | Weak (shared kernel) | Highest density | ms | Trusted internal services |
| gVisor (user-space kernel) | Syscall interception | Moderate | ms | Semi-trusted, k8s-friendly |
| microVM (Firecracker, Kata) | Own kernel, minimal devices | Low-ish; ~125 ms boot | fast | Untrusted multi-tenant (FaaS) |
| Full VM (type-1/type-2) | Strong, own kernel + devices | Highest per-VM | s | Strong tenant separation, mixed OS |

## Mechanics

- **Hypervisors:** type-1 run on bare metal (KVM, Xen, Hyper-V, ESXi); type-2
  run hosted (VirtualBox). Hardware assists (VT-x/AMD-V, EPT/NPT) are assumed.
- **Containers** are not a virtualization technology but a kernel feature
  composition: namespaces (pid/net/mnt/user/…), cgroups (resource limits),
  capabilities, and seccomp. They share the host kernel, so by themselves they
  are **not** a strong security boundary against kernel exploits.
- **microVMs** give VM-grade isolation at near-container speed and density by
  stripping the device model to the minimum (AWS Lambda runs on Firecracker).
- **Confidential computing** (SEV-SNP, TDX) encrypts guest memory from the host,
  for scenarios where even the host is untrusted.

## Guidance

Trusted internal microservices → containers with cgroups, seccomp, and rootless
user namespaces. Running untrusted or tenant code → at least a microVM (Firecracker/
Kata) or gVisor. Mixed operating systems or the strongest tenant separation →
full VMs on a type-1 hypervisor.
