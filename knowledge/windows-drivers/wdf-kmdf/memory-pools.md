# Windows Drivers - KMDF - Memory Pools

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/allocating-system-space-memory
> **Skill**: dev-suite skill `windows/wdf-kmdf` — see SKILL.md for the always-loaded quick reference.

## What this covers

The kernel pool allocators introduced with `ExAllocatePool2` (WDK 22000+), the bit semantics of `POOL_FLAG_*`, when to use lookaside lists for high-frequency small allocations, how `WDFMEMORY` wraps allocations into the WDF object lifetime, and what HVCI/CFG/Pool Hardening actually enforce. Read when migrating off deprecated APIs, when `!analyze` reports `BAD_POOL_*`, or when tuning a hot path.

## Deep dive

### Why the old APIs are gone

`ExAllocatePool`, `ExAllocatePoolWithTag`, and `ExAllocatePoolWithQuotaTag` were marked deprecated in WDK 22000 and produce SDV errors as of more recent kits. The replacement is a single function with a flags bitmap:

```c
PVOID p = ExAllocatePool2(POOL_FLAG_NON_PAGED, sizeof(MY_THING), 'rvDM');
```

`ExAllocatePool2` differs from the old call in three observable ways:

1. **Zeroed by default** — equivalent to `RtlZeroMemory` after allocation. Pass `POOL_FLAG_UNINITIALIZED` if you immediately fill the buffer (perf-only).
2. **Failure injection** — under Driver Verifier, allocations may return `NULL` deliberately to test failure paths.
3. **Pool hardening** — extra metadata makes use-after-free and overflow exploit primitives harder. Pool Hardening is on by default in 22H2+.

### POOL_FLAG_* catalog

| Flag | Meaning |
|------|---------|
| `POOL_FLAG_NON_PAGED` | Resident in physical memory; safe at DISPATCH |
| `POOL_FLAG_PAGED` | Pageable; PASSIVE only |
| `POOL_FLAG_NON_PAGED_EXECUTE` | NX disabled; **avoid** — fails HVCI attestation |
| `POOL_FLAG_UNINITIALIZED` | Skip zero-fill |
| `POOL_FLAG_CACHE_ALIGNED` | Aligned to system cache line; combine with size for SIMD work |
| `POOL_FLAG_RAISE_ON_FAILURE` | Raise structured exception instead of returning NULL — combine with SEH |
| `POOL_FLAG_USE_QUOTA` | Charge against caller's process quota |

`POOL_FLAG_NON_PAGED_EXECUTE` will block driver loading on devices with HVCI; if you generate code (JIT, hooking, anti-cheat) you need a separate code-integrity story.

Always free with the matching tag:

```c
ExFreePoolWithTag(p, 'rvDM');
```

### Lookaside lists for fixed-size hot allocations

If you allocate the same struct thousands of times per second (per-IO context, per-packet), use a lookaside:

```c
LOOKASIDE_LIST_EX g_MyLookaside;

NTSTATUS InitLookaside(VOID) {
    return ExInitializeLookasideListEx(
        &g_MyLookaside,
        NULL, NULL,                  // alloc/free hooks (optional)
        NonPagedPoolNx,
        0,                            // flags
        sizeof(MY_PER_IO),
        'rvDM',
        0);
}

PMY_PER_IO p = (PMY_PER_IO)ExAllocateFromLookasideListEx(&g_MyLookaside);
// ...
ExFreeToLookasideListEx(&g_MyLookaside, p);

// Cleanup at unload
ExDeleteLookasideListEx(&g_MyLookaside);
```

The lookaside avoids pool fragmentation and can serve the request from a per-CPU cache without locking.

### WDFMEMORY — pool tied to an object lifetime

```c
WDFMEMORY mem;
PVOID buffer;
WDF_OBJECT_ATTRIBUTES attrs;
WDF_OBJECT_ATTRIBUTES_INIT(&attrs);
attrs.ParentObject = device;          // freed when device dies

NTSTATUS s = WdfMemoryCreate(&attrs,
                              NonPagedPoolNx,
                              'rvDM',
                              0x1000,
                              &mem,
                              &buffer);
if (!NT_SUCCESS(s)) return s;
```

Use this for buffers that should die with the device, registry data, or anything you would have leaked if you forgot to free in `EvtDeviceReleaseHardware`. The framework also exposes `WdfMemoryAssignBuffer` so you can wrap an externally-allocated buffer.

### Tag conventions

A pool tag is exactly four ASCII bytes, displayed reversed by `!poolused`. Use ALL CAPS (so `'rvDM'` shows as `MDvr` in poolmon — a deliberate four-letter prefix per driver makes leaks obvious). Reserve one tag per allocation site for high-noise drivers (`MDv1`, `MDv2`, ...).

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Allocating `POOL_FLAG_PAGED` then touching at DISPATCH | Page fault, bugcheck `0xA` | Always `NON_PAGED` for DPC/ISR data |
| Forgetting `ExFreePoolWithTag` matching tag | Bugcheck `0xC2` (`BAD_POOL_CALLER`) | Same tag in alloc and free |
| Reusing a freed pointer in a lookaside cache | Use-after-free | Set the slot to `NULL` after free |
| Running on HVCI with `POOL_FLAG_NON_PAGED_EXECUTE` | Driver fails to load | Drop NX requirement; never JIT in kernel |
| Ignoring `RAISE_ON_FAILURE` exceptions | Stack unwind through unprepared code | Wrap in `__try`/`__except` if you set the flag |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/exallocatepool2
- https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/using-lookaside-lists
- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/pool-allocations
