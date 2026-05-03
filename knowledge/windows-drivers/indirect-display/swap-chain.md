# Windows Drivers - Indirect Display - Swap Chain Loop

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/iddcx/nf-iddcx-iddcxswapchainreleaseandacquirebuffer
> **Skill**: dev-suite skill `windows/indirect-display` — see SKILL.md for the always-loaded quick reference.

## What this covers

Detailed mechanics of the swap-chain consumer loop: `IddCxSwapChainAcquireBuffer`
vs `IddCxSwapChainReleaseAndAcquireBuffer`, the `IDARG_OUT_RELEASEANDACQUIREBUFFER`
struct, frame pacing strategies, and how to keep the GPU pipeline non-blocking.
The SKILL has the canonical loop; this file explains every line.

## Deep dive

### Acquire vs release-and-acquire

There are two acquire APIs:

- `IddCxSwapChainAcquireBuffer` — used once at the very start to get the *first*
  frame.
- `IddCxSwapChainReleaseAndAcquireBuffer` — releases the frame you previously
  acquired (calling FinishedProcessingFrame implicitly is **not** a release; you
  still need ReleaseAndAcquire) and gets the next.

In practice you call `ReleaseAndAcquire` for every iteration after the first.
Many drivers skip the standalone `Acquire` because IddCx tolerates an initial
`ReleaseAndAcquire` on a fresh swap chain (it just won't release anything).

### Output struct

```c
typedef struct IDARG_OUT_RELEASEANDACQUIREBUFFER {
    UINT64       MetaData_PresentationFrameNumber;
    LARGE_INTEGER PresentationTime;
    LARGE_INTEGER LastDisplayedTime;       // for frame-drop telemetry
    HANDLE       MetaData_Reserved;
    IDDCX_BUFFER Buffer;                   // contains an IDXGIResource via NT handle
} IDARG_OUT_RELEASEANDACQUIREBUFFER;
```

`Buffer` carries an NT handle to a shared D3D resource. To use it on your local
D3D11 device:

```cpp
HRESULT hr = D3D11Device->OpenSharedResource1(buf.Buffer.MetaData.pSharedHandle,
                                              IID_PPV_ARGS(&texture));
// texture is a IDXGIResource1 / ID3D11Texture2D ready to use as input
```

Modern IddCx (1.5+) hands you the resource via `Buffer.MetaData.pSharedHandle`
directly. Older versions used a `IDXGIResource1*` pointer in another field; check
the `Buffer.MetaData.Type`.

### Frame pacing

The framework fires the swap-chain's "new frame" event at the cadence DWM expects
(matches the declared monitor refresh rate). The consumer loop:

```cpp
// Setup
HANDLE newFrameEvent = nullptr;
IDARG_IN_SETMODE setMode{};
setMode.AvailableBufferEvent = newFrameEvent;
IddCxSwapChainSetDevice(swapChain, &setDevice);

for (;;) {
    DWORD wait = WaitForSingleObject(newFrameEvent, 100);
    if (wait == WAIT_TIMEOUT) {
        // No new frame in 100ms — check stop event, log
        if (WaitForSingleObject(stopEvent, 0) == WAIT_OBJECT_0) break;
        continue;
    }
    IDARG_OUT_RELEASEANDACQUIREBUFFER buf{};
    HRESULT hr = IddCxSwapChainReleaseAndAcquireBuffer(swapChain, &buf);
    if (hr == E_PENDING) continue;          // race: wait again
    if (FAILED(hr)) break;
    EncodeAndQueue(buf);                     // <-- non-blocking
    IDARG_IN_FINISHEDPROCESSINGFRAME fin{};
    IddCxSwapChainFinishedProcessingFrame(swapChain, &fin);
}
```

Two distinct sequences matter:

1. **Acquire → Process → Finish** is one frame's lifetime from the framework's
   perspective. `Finish` tells DWM "I've consumed the surface, you can recycle
   it." Failing to call `Finish` will deadlock the next acquire after the buffer
   pool drains (typically 3 buffers).
2. **Acquire → Release** marks the surface as no longer in use by your driver.
   `ReleaseAndAcquireBuffer` does both in one call.

### Why E_PENDING happens

DWM presents at most as fast as the declared refresh. If you call
`ReleaseAndAcquireBuffer` faster than DWM presents, you get `E_PENDING`. Always
wait on the new-frame event before calling — this is the framework's flow control
mechanism.

### Throughput tips

- **Stay on GPU**: open the shared handle, encode with NVENC's
  `NV_ENC_INPUT_RESOURCE_TYPE_DIRECTX`, never `Map` the texture to CPU.
- **One D3D11 device per monitor thread**: cross-monitor sharing causes immediate
  context contention.
- **Pre-allocate encoder buffers**: per-frame allocation kills cache locality and
  allocator throughput.
- **Decouple encode from network**: encoder fills a ring, separate sender thread
  drains.
- **Use CPU side only for cursor blending**, if at all — cursor surface is small.

### Mode change handling

`EvtIddCxAdapterCommitModes` may fire while the loop is running. The framework
will then unassign and reassign the swap chain. Your loop should treat
`FAILED(hr)` other than `E_PENDING` as "exit cleanly" — the framework will create
a fresh swap chain and call `EvtIddCxMonitorAssignSwapChain` again with new
dimensions.

### Frame-drop diagnostics

`buf.LastDisplayedTime` reflects the wall-clock time DWM last saw your "Finish".
If it lags `buf.PresentationTime` by more than one frame interval, you're
dropping. Log periodically — most production IDD drivers expose a counter via
ETW.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `Sleep(16)` instead of waiting on the new-frame event | Drift, jitter, missed frames | Always `WaitForSingleObject(newFrameEvent, ...)` |
| Synchronous network send in the loop | Blocks the next acquire | Encode + enqueue; sender on its own thread |
| Forgetting `FinishedProcessingFrame` | Buffer pool drains, framework deadlocks | Pair every acquire with finish |
| Recreating D3D11 device per mode change | Multi-second hitch | Recreate only encoder/swap-chain views, keep device |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/iddcx/nf-iddcx-iddcxswapchainfinishedprocessingframe
- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/iddcx/nf-iddcx-iddcxswapchainsetdevice
- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/iddcx/ns-iddcx-iddarg_out_releaseandacquirebuffer
