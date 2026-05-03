# Windows Drivers - Driver Debugging - Driver Verifier

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/driver-verifier
> **Skill**: dev-suite skill `windows/driver-debugging` — see SKILL.md for the always-loaded quick reference.

## What this covers

Every Driver Verifier rule, what each `/flags` bit enables, and when to choose `/standard` vs a custom mask. Fetch when SKILL.md's "use the standard set" advice is not specific enough — usually because Verifier is firing too often, not firing at all, or you need to enable an experimental rule like Code Integrity Checks.

## Deep dive

### Rule catalog and bit values

| Bit (hex) | Rule | Catches |
|-----------|------|---------|
| `0x00001` | Special pool | Pool overruns / use-after-free (each alloc on its own page) |
| `0x00002` | Force IRQL checking | Pageable code/data accessed at IRQL >= DISPATCH |
| `0x00004` | Low resources simulation | Random alloc failures (NDIS, registry, FS) |
| `0x00008` | Pool tracking | Leaked pool allocations at unload |
| `0x00010` | I/O Verification | IRP misuse: pending without `IoMarkIrpPending`, completing twice, etc. |
| `0x00020` | Deadlock detection | Lock-order inversions across spinlocks / mutexes |
| `0x00080` | DMA verification | Incorrect map-register / common-buffer use |
| `0x00100` | Security checks | RACE conditions on user-mode pointers |
| `0x00800` | Misc checks | Misc kernel API misuse |
| `0x02000` | Force pending I/O | Random `STATUS_PENDING` returns from lower drivers |
| `0x04000` | IRP logging | Records all IRPs for `!verifier 0x40 <drv>` later |
| `0x08000` | Invariant MDL checking (stack) | Stack MDL written after lock-down |
| `0x10000` | Invariant MDL checking (driver) | Driver-allocated MDL written after lock-down |
| `0x20000` | Power framework delay fuzzing | Random power-callback delays |
| `0x40000` | Port / miniport verification | Port-driver contract violations (storport, ndis) |
| `0x80000` | Systematic low resources simulation | Deterministic low-mem at numbered alloc sites |
| `0x200000` | Code integrity checks | Self-modifying code, executable pool |
| `0x209BB` | "Production-like" mask used in WHQL | Combines the everyday ones above |

`/standard` enables `0x00001 | 0x00002 | 0x00008 | 0x00010 | 0x00020 | 0x00080 | 0x00800` — pool, IRQL, leaks, I/O, deadlock, DMA, misc. It does **not** enable low-resources simulation by default.

### Useful command lines

```cmd
:: Enable the standard set on one driver, persist across reboots
verifier /standard /driver MyDriver.sys

:: Enable a custom mask
verifier /flags 0x209BB /driver MyDriver.sys

:: Enable on the entire driver collection (heavy — boot may take minutes)
verifier /flags 0x209BB /all

:: Query current verifier state without rebooting
verifier /query
verifier /querysettings

:: Add or remove drivers to a running session (no reboot for additions on Win10+)
verifier /volatile /adddriver Foo.sys
verifier /volatile /removedriver Bar.sys

:: Turn off Verifier (requires reboot)
verifier /reset
```

### Choosing flags by scenario

| Scenario | Flags |
|----------|-------|
| Day-to-day dev cycle | `/standard` |
| Hunting a leak | `/standard /flags +0x8` (already on) and unload the driver — `!verifier 0x3 <drv>` lists outstanding allocations |
| Reproducing a stress crash | `/flags 0x209BB` |
| Hunting a race / deadlock | `/standard /flags +0x20` (already on, just add aggressive testing) |
| Hunting a memory corruption that special pool isn't catching | Add `/flags +0x10000` (driver MDL) and `+0x8000` (stack MDL) |
| Power transition crashes | `/standard /flags +0x20000` |

### Reading a Verifier-induced bugcheck

`!analyze -v` on a `0xC4` will print a `VERIFIER_DETECTED_VIOLATION` parameter value. Common ones:

| Param 1 | Rule | Meaning |
|---------|------|---------|
| `0x60` | Pool | Quota mismatch on pool free |
| `0x68` | Pool | Allocation type mismatch (NonPaged freed as Paged) |
| `0x91` | Code | Driver attempted to execute non-executable memory |
| `0xC8` | DMA | DMA buffer freed while still mapped |
| `0xE6` | Pool | Pool overrun detected at end of buffer |

The full table is at the bug-check reference; bookmark `bug-checks.md` in this KB.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Verifier on the only kernel debugger target with no live debugger attached | Bugcheck reboots immediately, you lose the repro | Always attach `kd` first, or set `bcdedit /set autoreboot off` |
| `/flags 0x4` ("Low resources") on first boot | System won't even reach desktop | Add it on a second pass after `/standard` is clean |
| Forgetting to disable Verifier before benchmarking | Perf numbers off by 10–20% | `verifier /reset` then reboot |
| Using `/all` on a workstation | Boot becomes painfully slow, drivers third-party will crash | Use `/driver` with an explicit list |
| Ignoring the Verifier informational popups in Event Log | Useful warnings about IRQL / pool issues you missed | Filter Event Viewer → System for source = "Driver Verifier" |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/driver-verifier-options
- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/driver-verifier-flags
- https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-0xc4--driver-verifier-detected-violation
