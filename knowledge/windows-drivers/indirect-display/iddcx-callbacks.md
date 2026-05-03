# Windows Drivers - Indirect Display - IddCx Callbacks

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/iddcx/
> **Skill**: dev-suite skill `windows/indirect-display` — see SKILL.md for the always-loaded quick reference.

## What this covers

The complete set of `EvtIddCx*` callbacks, the order in which IddCx invokes them,
and what state must be ready before each one returns. The SKILL summarises the
hot path; this file is the lifecycle reference.

## Deep dive

### Driver-level callbacks (registered with `IDD_CX_CLIENT_CONFIG`)

| Callback | When called | Required to set up before return |
|----------|-------------|----------------------------------|
| `EvtIddCxDeviceIoControl` | App `DeviceIoControl` | Optional — only if you expose IOCTLs |
| `EvtIddCxAdapterInitFinished` | After `IddCxAdapterInitAsync` returns success | Plug all initial monitors via `IddCxMonitorArrival` |
| `EvtIddCxAdapterCommitModes` | When desktop mode set is committed | Reconfigure scanout buffers if needed |
| `EvtIddCxParseMonitorDescription` | Each time IddCx needs to interpret a descriptor (EDID) | Fill `MonitorModeBufferOutputCount` and modes array |
| `EvtIddCxMonitorGetDefaultModes` | When OS asks for fallback modes | Populate up to N IDDCX_MONITOR_MODE entries |
| `EvtIddCxMonitorQueryTargetModes` | OS narrowing target-side timings | Filter to the subset you can produce |
| `EvtIddCxMonitorAssignSwapChain` | DWM has a swap-chain ready | Start the processor thread, take a ref on `IddCxSwapChain` |
| `EvtIddCxMonitorUnassignSwapChain` | DWM tearing down swap-chain | Signal stop, join thread, release D3D resources |

### Lifecycle order, in detail

1. `DriverEntry` → `WdfDriverCreate` → `IddCxDriverInitialize`
2. PnP `EvtDeviceAdd`:
   - `WdfDeviceCreate`
   - `IDD_CX_CLIENT_CONFIG` filled with the eight callbacks above
   - `IddCxDeviceInitConfig` → `IddCxDeviceInitialize`
3. PnP `EvtDevicePrepareHardware`:
   - Initialize your D3D11 device, encoder, network resources
4. `IddCxAdapterInitAsync(IDARG_IN_ADAPTER_INIT*)` — give the framework your
   adapter caps (max monitors, primary preferred surface format, vertex shader
   level, etc.)
5. `EvtIddCxAdapterInitFinished(WDFDEVICE, IDARG_IN_ADAPTER_INIT_FINISHED*)`
   — framework confirms; here you call `IddCxMonitorCreate` for each monitor and
   `IddCxMonitorArrival` to advertise them
6. For each monitor, framework calls `EvtIddCxParseMonitorDescription` (one call
   to get count, second call to fill modes array)
7. When DWM is ready: `EvtIddCxMonitorAssignSwapChain` → start consumer
8. Steady state: consumer loops on `IddCxSwapChainReleaseAndAcquireBuffer`
9. On mode change: `EvtIddCxAdapterCommitModes` → may unassign+reassign
10. On unplug or shutdown: `EvtIddCxMonitorUnassignSwapChain` → join thread →
    `IddCxMonitorDeparture` → `IddCxMonitorReleaseAndDestroy`

### Two-phase callbacks

`EvtIddCxParseMonitorDescription` and `EvtIddCxMonitorGetDefaultModes` use the
classic Windows two-call pattern. First call: `pMonitorModes == NULL`, you fill
`MonitorModeBufferOutputCount` with the count. Second call: array is allocated to
that size, fill it.

```c
NTSTATUS MyParseMonitorDescription(
    const IDARG_IN_PARSEMONITORDESCRIPTION* in,
    IDARG_OUT_PARSEMONITORDESCRIPTION* out)
{
    UINT count = ComputeModeCount(in->MonitorDescription.pData,
                                   in->MonitorDescription.DataSize);
    out->MonitorModeBufferOutputCount = count;
    if (in->pMonitorModes == NULL) {
        return STATUS_SUCCESS;          // first call: count only
    }
    if (in->MonitorModeBufferInputCount < count) {
        return STATUS_BUFFER_TOO_SMALL;
    }
    FillModes(in->pMonitorModes, count); // second call: fill array
    out->PreferredMonitorModeIdx = 0;
    return STATUS_SUCCESS;
}
```

### Async vs sync init

`IddCxAdapterInitAsync` and `IddCxMonitorCreate` are non-blocking. Your driver
must not present monitors as ready until `EvtIddCxAdapterInitFinished` fires —
calling `IddCxMonitorArrival` before that returns `STATUS_INVALID_DEVICE_STATE`.

### Threading rules

Each `EvtIddCx*` callback runs on a framework worker thread, *not* on the same
thread that registered the callback. Concurrent callbacks for different monitors
can fire in parallel. Per-monitor state needs its own lock (or single-threaded
serialization through a per-monitor work queue). Driver-global state (e.g. shared
encoder pool) needs cross-thread synchronization.

### Shutdown sequence

PnP remove fires `EvtDeviceReleaseHardware`. Before that, IddCx calls
`EvtIddCxMonitorUnassignSwapChain` for every active monitor. Pattern:

```cpp
void OnUnassignSwapChain(IDDCX_MONITOR Monitor) {
    auto* mctx = GetMonitorContext(Monitor);
    SetEvent(mctx->StopEvent);
    WaitForSingleObject(mctx->ThreadHandle, INFINITE);
    CloseHandle(mctx->ThreadHandle);
    mctx->SwapChain.reset();        // releases the COM ref
}
```

If your thread holds the swap-chain ref, framework cleanup waits forever — always
release on stop.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Calling `IddCxMonitorArrival` from `EvtDeviceAdd` | Adapter not initialized yet | Defer to `EvtIddCxAdapterInitFinished` |
| Returning `STATUS_SUCCESS` from `ParseMonitorDescription` first call without setting count | Framework allocates 0 modes, fails | Always set `MonitorModeBufferOutputCount` |
| Single global lock around all callbacks | Serializes multi-monitor work | Per-monitor lock; only adapter-wide state needs the global one |
| Joining the processor thread inside the callback when the thread is waiting on a callback to return | Deadlock | Use `SetEvent` + non-blocking wait pattern |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/iddcx/ne-iddcx-iddcx_version
- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/iddcx/nf-iddcx-iddcxmonitorcreate
- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/iddcx/nf-iddcx-iddcxadapterinitasync
