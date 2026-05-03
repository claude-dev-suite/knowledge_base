# Windows Drivers - KMDF - Synchronization

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/using-automatic-synchronization
> **Skill**: dev-suite skill `windows/wdf-kmdf` — see SKILL.md for the always-loaded quick reference.

## What this covers

WDF's automatic synchronization (`SynchronizationScope` + `ExecutionLevel`), when to fall back to explicit `WDFSPINLOCK`/`WDFWAITLOCK`, the cost of each primitive, interrupt synchronization with `WdfInterruptAcquireLock`, and the lock ordering that prevents deadlocks across queue/timer callbacks. Read when SDV reports a lock leak or when a callback fires concurrently with itself unexpectedly.

## Deep dive

### Automatic synchronization scopes

`WDF_OBJECT_ATTRIBUTES.SynchronizationScope` controls which callbacks the framework serializes:

| Scope | Effect |
|-------|--------|
| `WdfSynchronizationScopeNone` | No synchronization — your callbacks may run concurrently |
| `WdfSynchronizationScopeQueue` | All callbacks on a queue object are serialized against each other |
| `WdfSynchronizationScopeDevice` | All callbacks on the device AND its queues are serialized via a single device lock |

Combined with `ExecutionLevel`:

| Level | Effect |
|-------|--------|
| `WdfExecutionLevelInheritFromParent` | Default — pick parent's level |
| `WdfExecutionLevelPassive` | Framework reposts to passive worker if needed |
| `WdfExecutionLevelDispatch` | Run at <= DISPATCH; locks are spinlocks |

`Device` scope + `Passive` level gives you a single `WDFWAITLOCK` covering everything; convenient for low-rate devices, but a contention bottleneck on high-IOPS hardware.

### Explicit primitives

```c
// Spinlock — DISPATCH max
WDFSPINLOCK lock;
WDF_OBJECT_ATTRIBUTES attrs;
WDF_OBJECT_ATTRIBUTES_INIT(&attrs);
attrs.ParentObject = device;
WdfSpinLockCreate(&attrs, &lock);

WdfSpinLockAcquire(lock);     // raises to DISPATCH on this CPU
// short, non-blocking critical section
WdfSpinLockRelease(lock);
```

`WdfSpinLockAcquire` is a regular spinlock — about 30-100 cycles uncontended. Holding it raises IRQL to `DISPATCH_LEVEL`, so all the usual restrictions apply.

```c
// Wait lock — PASSIVE only, may block
WDFWAITLOCK waitLock;
WdfWaitLockCreate(&attrs, &waitLock);

NTSTATUS s = WdfWaitLockAcquire(waitLock, NULL);  // NULL = wait forever
// blocking work allowed
WdfWaitLockRelease(waitLock);
```

`WdfWaitLockAcquire` accepts an optional timeout (`PLONGLONG`, 100ns ticks; negative = relative). If the timeout expires you get `STATUS_TIMEOUT`; check it.

### Interrupt synchronization

The framework gives you a special spinlock per `WDFINTERRUPT` that raises to the interrupt's DIRQL — the only way to safely touch shared state with an ISR from passive code:

```c
WdfInterruptAcquireLock(interrupt);     // raises to DIRQL on this CPU
ctx->SharedRegister |= MY_BIT;
WRITE_REGISTER_ULONG(&ctx->Regs->Ctrl, ctx->SharedRegister);
WdfInterruptReleaseLock(interrupt);
```

Or wrap the work as an `EvtInterruptSynchronize` callback:

```c
BOOLEAN MyToggleBit(_In_ WDFINTERRUPT Interrupt, _In_ WDFCONTEXT Context)
{
    PMY_DEVICE_CONTEXT ctx = (PMY_DEVICE_CONTEXT)Context;
    ctx->SharedRegister |= MY_BIT;
    return TRUE;
}
WdfInterruptSynchronize(interrupt, MyToggleBit, ctx);
```

### Lock ordering rules

KMDF follows strict ordering to avoid deadlocks; you must too. Standard order:

1. Driver lock (rarely used)
2. Device lock
3. Queue lock
4. Request lock
5. Per-data spinlocks you create (assign your own ranking, document it)
6. Interrupt lock (highest IRQL — implicitly last)

Never acquire a higher-ranked lock while holding a lower-ranked one. SDV's `LockRanking` rule enforces this if you annotate `_Acquires_lock_` / `_Releases_lock_`.

### KEVENT and other low-level primitives

Sometimes you need a `KEVENT` (e.g. waiting for an async response in a passive thread):

```c
KEVENT done;
KeInitializeEvent(&done, NotificationEvent, FALSE);
// signal from completion routine: KeSetEvent(&done, IO_NO_INCREMENT, FALSE);
KeWaitForSingleObject(&done, Executive, KernelMode, FALSE, NULL);
```

This is fine at PASSIVE, never at DISPATCH. Prefer WDFTIMER + workitem if you might be at DISPATCH.

### Reader/writer

KMDF doesn't ship a R/W lock. For read-heavy workloads, use `EX_PUSH_LOCK`:

```c
EX_PUSH_LOCK pushLock;
ExInitializePushLock(&pushLock);

KeEnterCriticalRegion();
ExAcquirePushLockShared(&pushLock);
// many readers
ExReleasePushLockShared(&pushLock);
KeLeaveCriticalRegion();
```

Push locks are PASSIVE-only and require disabling APCs (`KeEnterCriticalRegion`).

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Acquiring `WDFWAITLOCK` from a parallel-queue callback | Callback may run at DISPATCH | Either use spinlock or set `ExecutionLevel = WdfExecutionLevelPassive` |
| Holding spinlock across `WdfRequestComplete` | Complete may invoke completion routines and reentrantly take more locks | Drop the lock first |
| Acquiring locks in different orders in two paths | Deadlock | Pick a global order; annotate with SAL `_Acquires_lock_` |
| Using `KeWaitForSingleObject` with non-NULL timeout at DISPATCH | Bugcheck (`SCHEDULER_AT_DISPATCH`) | Move to PASSIVE via workitem |
| Forgetting `KeEnterCriticalRegion` before push-lock | APC during push-lock = deadlock | Always pair `Enter`/`Leave` |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/synchronizing-interrupt-code
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/using-framework-locks
- https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/introduction-to-spin-locks
