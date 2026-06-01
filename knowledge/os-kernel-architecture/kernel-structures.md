# Kernel Structures: Monolithic, Microkernel, Hybrid, Unikernel

## Overview

The kernel structure decides what runs in privileged (kernel) space versus
unprivileged (user) space. That single choice cascades into performance, fault
isolation, security, verifiability, and how the system is extended. It is the
first architectural decision when designing or evaluating an OS.

## The structures

```
Monolithic                 Microkernel                Unikernel
+-------------------+      +----------------+         +------------------+
| app | app | app   |      | app | FS | net |  user   |  application     |
+-------------------+      +----------------+         |  + libOS         |
| FS net drv sched  | krn  | IPC sched vm   |  kernel  |  (one address    |
| vm  ipc  ...      |      +----------------+         |   space)         |
+-------------------+      (drivers/FS/net are        +------------------+
 (all services in        user-space servers reached    runs on a hypervisor
  kernel space)          via fast IPC)                  as a single image
```

- **Monolithic (Linux, classic BSD):** all core services run in kernel space.
  No IPC cost between services; mature and fast. But the trusted computing base
  (TCB) is huge and a fault anywhere can panic the whole system.
- **Microkernel (seL4, QNX, L4 family):** the kernel provides only address
  spaces, threads/scheduling, and IPC. Drivers, file systems, and the network
  stack are ordinary user-space servers. Strong fault isolation (a crashed
  driver can be restarted), a small verifiable TCB (seL4 is formally verified),
  at the cost of IPC on hot paths — which is why microkernel IPC latency is the
  make-or-break metric.
- **Hybrid (XNU/macOS, Windows NT):** a monolithic-ish core with some services
  structured as servers. A pragmatic middle ground; boundaries can be blurry.
- **Unikernel (MirageOS, Unikraft):** the application is compiled together with
  a minimal library OS into a single image in one address space, booted directly
  on a hypervisor. Tiny attack surface and fast boot, but a single application
  with weak internal isolation — ideal for single-purpose cloud/edge appliances.
- **Exokernel (research):** the kernel only securely multiplexes hardware;
  policy lives in application-level library OSes.

## Choosing

| Driver | Favors |
|--------|--------|
| Maximum throughput, rich ecosystem | Monolithic (Linux) |
| Provable isolation, restartable drivers, small TCB | Microkernel (seL4/QNX) |
| Single-purpose cloud/edge appliance | Unikernel |
| Commercial desktop/mobile OS | Hybrid |

The recurring trade-off is **isolation/assurance versus IPC overhead and
engineering complexity**. Safety- and security-critical systems pay the IPC cost
for isolation and verifiability; performance-first general-purpose systems keep
services in-kernel.
