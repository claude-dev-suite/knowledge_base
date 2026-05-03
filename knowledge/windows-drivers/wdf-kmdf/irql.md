# Windows Drivers - KMDF - IRQL Discipline

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/managing-hardware-priorities
> **Skill**: dev-suite skill `windows/wdf-kmdf` — see SKILL.md for the always-loaded quick reference.

## What this covers

What Interrupt Request Level (IRQL) actually controls on x64/ARM64 Windows, when each level is in effect, what `KeRaiseIrql`/`KeLowerIrql` cost, and the practical contract for each WDF callback. Read when SDV flags an IRQL violation, when you see `IRQL_NOT_LESS_OR_EQUAL` (`0xA`), or when designing an ISR/DPC pair.

## Deep dive

### IRQL is per-CPU, software-tracked

IRQL is stored in the per-CPU `KPRCB` (`Irql`). On x64 it maps to the APIC TPR/CR8. Raising IRQL masks any interrupt at or below the new level on the current CPU — other CPUs are unaffected. A thread running at `DISPATCH_LEVEL` cannot be preempted by the scheduler (the scheduler runs at `DISPATCH_LEVEL`), so it cannot block.

### The five levels you care about

| IRQL | Numeric | What runs | Forbidden |
|------|---------|-----------|-----------|
| `PASSIVE_LEVEL` | 0 | Most driver code, all user mode | — |
| `APC_LEVEL` | 1 | Special kernel APCs, page-fault handler completion | Acquiring same lock from APC |
| `DISPATCH_LEVEL` | 2 | DPCs, scheduler, spinlock holders | Page faults, blocking, paged pool, `KeWaitForSingleObject(timeout > 0)` |
| `Device IRQL` (DIRQL) | 3..26 | Your `EvtInterruptIsr` | Almost everything; talk to hardware then `WdfInterruptQueueDpcForIsr` |
| `HIGH_LEVEL` | 31 | NMI, machine check | — |

### Which WDF callback runs at what IRQL

| Callback | IRQL |
|----------|------|
| `EvtDriverDeviceAdd`, `EvtDevicePrepareHardware`, `EvtDeviceReleaseHardware`, `EvtDeviceD0Entry`/`Exit` | `PASSIVE_LEVEL` |
| `EvtIoDeviceControl`, `EvtIoRead`, `EvtIoWrite` (default) | `<= DISPATCH_LEVEL` (parallel queue) or `PASSIVE_LEVEL` if `ExecutionLevel = WdfExecutionLevelPassive` |
| `EvtInterruptIsr` | DIRQL |
| `EvtInterruptDpc`, `EvtInterruptWorkItem` | `DISPATCH_LEVEL` and `PASSIVE_LEVEL` respectively |
| `EvtTimerFunc` | `DISPATCH_LEVEL` (default) or `PASSIVE_LEVEL` if `WDF_TIMER_CONFIG.UseHighResolutionTimer = WdfFalse` and `AutomaticSerialization = WdfFalse` plus passive execution level |
| `EvtRequestCancel` | `DISPATCH_LEVEL` |

To force callbacks to PASSIVE on a given object:

```c
WDF_OBJECT_ATTRIBUTES_INIT(&attrs);
attrs.ExecutionLevel = WdfExecutionLevelPassive;
attrs.SynchronizationScope = WdfSynchronizationScopeDevice;
WdfIoQueueCreate(device, &qcfg, &attrs, &queue);
```

This costs you a context switch per request (the framework reposts the callback to a system worker thread).

### ISR / DPC pattern

```c
BOOLEAN MyEvtInterruptIsr(_In_ WDFINTERRUPT Interrupt, _In_ ULONG MessageID)
{
    PMY_DEVICE_CONTEXT ctx = GetDeviceContext(WdfInterruptGetDevice(Interrupt));
    ULONG status = READ_REGISTER_ULONG(&ctx->Regs->IntStatus);
    if (!(status & MY_INT_PENDING)) return FALSE;     // not ours

    WRITE_REGISTER_ULONG(&ctx->Regs->IntAck, status); // ack quickly
    ctx->LatchedStatus = status;
    WdfInterruptQueueDpcForIsr(Interrupt);
    return TRUE;
}

VOID MyEvtInterruptDpc(_In_ WDFINTERRUPT Interrupt, _In_ WDFOBJECT AssociatedObject)
{
    UNREFERENCED_PARAMETER(AssociatedObject);
    PMY_DEVICE_CONTEXT ctx = GetDeviceContext(WdfInterruptGetDevice(Interrupt));
    // Now at DISPATCH_LEVEL — process latched status, complete requests
    ProcessIoCompletion(ctx, ctx->LatchedStatus);
}
```

Hold the spinlock for the device registers via `WdfInterruptAcquireLock` from passive-level paths so you serialize against the ISR.

### Raising and lowering IRQL manually

Almost never needed in WDF. If you must (e.g. wrapping a legacy WDM API):

```c
KIRQL old;
KeRaiseIrql(DISPATCH_LEVEL, &old);
// ...
KeLowerIrql(old);
```

You may not raise above the CPU's current IRQL with `KeRaiseIrqlToDpcLevel` if already above DISPATCH. Lowering past where you raised is a bugcheck.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `KeWaitForSingleObject(..., NonZeroTimeout)` at DISPATCH | Wait routines require schedulable thread | Queue a workitem, wait at PASSIVE |
| Touching paged data at DISPATCH | Page fault → bugcheck `0xA` | `MmLockPagableCodeSection`, or move data to `NonPagedPoolNx` |
| Long ISR | Other devices on the same line stall | Latch + DPC; do work at DISPATCH_LEVEL |
| Forgetting `_IRQL_requires_max_(...)` SAL | SDV cannot prove caller correctness | Annotate every helper |
| Returning from ISR with `TRUE` when not yours | Steals the IRQ from the real owner | Strict register-read first |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/scheduling--thread-context--and-irql
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/handling-hardware-interrupts
- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/bug-check-0xa--irql-not-less-or-equal
