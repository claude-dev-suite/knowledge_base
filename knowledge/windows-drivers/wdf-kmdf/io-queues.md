# Windows Drivers - KMDF - I/O Queues

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/managing-i-o-queues
> **Skill**: dev-suite skill `windows/wdf-kmdf` — see SKILL.md for the always-loaded quick reference.

## What this covers

The internal mechanics of WDF I/O queues — how parallel/sequential dispatchers serialize, how manual queues let you implement inverted-call patterns, the ordering guarantees during PnP transitions, and what `EvtIoStop`/`EvtIoResume` mean for a request that was already handed to your callback. Read when designing an asynchronous device or when requests "vanish" during sleep/resume.

## Deep dive

### Dispatch types in detail

| Type | When the framework calls your callback | Concurrency |
|------|----------------------------------------|-------------|
| `WdfIoQueueDispatchSequential` | After previous request completes | One callback active per queue |
| `WdfIoQueueDispatchParallel` | As soon as a request arrives | Many callbacks concurrent — your code MUST be reentrant |
| `WdfIoQueueDispatchManual` | Never — you call `WdfIoQueueRetrieveNextRequest` | You schedule pulling |

Sequential queues use a single internal "in-flight" slot. The next request is delivered exactly when the previous one is completed (`WdfRequestComplete*`). If you forward to another queue or to an I/O target, the slot opens immediately — sequential does NOT mean "until the device responds".

### Multiple queues per device

You almost always want at least two queues:

```c
// Default queue — accepts everything not steered elsewhere
WDF_IO_QUEUE_CONFIG_INIT_DEFAULT_QUEUE(&q, WdfIoQueueDispatchParallel);
q.EvtIoDeviceControl = MyEvtIoDeviceControl;
WdfIoQueueCreate(device, &q, WDF_NO_OBJECT_ATTRIBUTES, NULL);

// Manual queue for "long poll" / inverted-call IOCTLs
WDF_IO_QUEUE_CONFIG_INIT(&q, WdfIoQueueDispatchManual);
WDFQUEUE manualQ;
WdfIoQueueCreate(device, &q, WDF_NO_OBJECT_ATTRIBUTES, &manualQ);
GetDeviceContext(device)->NotifyQueue = manualQ;
```

Then steer by IoControlCode:

```c
case IOCTL_MYDRV_WAIT_FOR_EVENT:
    status = WdfRequestForwardToIoQueue(Request, GetDeviceContext(device)->NotifyQueue);
    if (!NT_SUCCESS(status))
        WdfRequestComplete(Request, status);
    return;   // do NOT complete here
```

When the device produces an event, retrieve and complete:

```c
WDFREQUEST req;
NTSTATUS s = WdfIoQueueRetrieveNextRequest(ctx->NotifyQueue, &req);
if (NT_SUCCESS(s)) {
    PMY_EVENT out;
    WdfRequestRetrieveOutputBuffer(req, sizeof(*out), (PVOID*)&out, NULL);
    *out = currentEvent;
    WdfRequestCompleteWithInformation(req, STATUS_SUCCESS, sizeof(*out));
}
```

### Power-managed queues

By default, queues are power-managed: KMDF stops dispatching when the device leaves D0 and resumes on D0 entry. Set `q.PowerManaged = WdfFalse` for queues whose requests must flow even when the device is off (rare; e.g. a queue that returns cached state).

### EvtIoStop and the cancelable state

When PnP needs to stop the queue (sleep, surprise-remove), KMDF invokes `EvtIoStop` for each in-flight request. You have three legal responses:

```c
VOID MyEvtIoStop(_In_ WDFQUEUE Queue, _In_ WDFREQUEST Request, _In_ ULONG ActionFlags)
{
    UNREFERENCED_PARAMETER(Queue);

    if (ActionFlags & WdfRequestStopActionSuspend) {
        // Option A: tell framework "I will complete it eventually"
        WdfRequestStopAcknowledge(Request, FALSE /* don't requeue */);
    } else if (ActionFlags & WdfRequestStopActionPurge) {
        // Option B: cancel it now
        WdfRequestComplete(Request, STATUS_CANCELLED);
    }
    // Option C: silently drop reference — framework times out and bugchecks. Don't.
}
```

`WdfRequestStopAcknowledge(req, TRUE)` requeues at the head when power returns — useful for I/O you can simply re-issue.

### Cancellation

Mark a request cancelable BEFORE you park it:

```c
WdfRequestMarkCancelableEx(Request, MyEvtRequestCancel);
// later, retrieve before completing to avoid race
NTSTATUS s = WdfRequestUnmarkCancelable(Request);
if (s == STATUS_CANCELLED) return;       // cancel callback owns it
WdfRequestComplete(Request, STATUS_SUCCESS);
```

`MyEvtRequestCancel` runs at `DISPATCH_LEVEL`; it must complete the request itself.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Holding `WDFREQUEST` past return without marking cancelable | App `CancelIoEx` deadlocks; PnP stop times out | `WdfRequestMarkCancelableEx` + matching `Unmark` |
| Sequential queue + forwarding to another queue | "Sequential" only blocks until you forward | Use a per-device counter or block on a `KEVENT` if true serialization needed |
| Calling `WdfRequestComplete` from cancel routine without `Unmark` | Double-complete bugcheck | Check `Unmark` return value before completing in the normal path |
| Not setting `PowerManaged = FALSE` on diagnostic queues | Queue stalls during S3, app appears hung | Mark non-power-managed for control-plane queues |
| Forgetting that a parallel queue dispatches concurrently | Race conditions in callback | Either use sequential or take a `WDFSPINLOCK` |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/request-handlers
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/dispatching-methods-for-i-o-requests
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/canceling-i-o-requests
