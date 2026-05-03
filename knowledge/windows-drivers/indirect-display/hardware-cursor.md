# Windows Drivers - Indirect Display - Hardware Cursor

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/iddcx/nf-iddcx-iddcxmonitorsetuphardwarecursor
> **Skill**: dev-suite skill `windows/indirect-display` — see SKILL.md for the always-loaded quick reference.

## What this covers

How to opt into separate cursor surfaces, react to position-change and shape-change
events, and feed the cursor to a remote receiver for sub-frame latency. The SKILL
mentions the API; this file shows the full event-loop pattern and shape format
quirks.

## Deep dive

### Why separate cursor

If the cursor is composited into the frame, every cursor movement triggers a full
frame redraw. For a network-streamed display this is a huge bandwidth and latency
loss: cursor moves at 1000Hz, your encoder runs at 60fps, and the cursor lags
visibly. Separating the cursor lets the receiver render it locally at near-zero
latency, sourced from a tiny ARGB sprite the driver hands over once and updates
on shape change only.

### Setup

Inside `EvtIddCxMonitorAssignSwapChain` (or shortly after), call:

```cpp
IDDCX_CURSOR_CAPS caps = {};
caps.Size              = sizeof(caps);
caps.ColorXorCursorSupport = IDDCX_XOR_CURSOR_SUPPORT_NONE;  // we don't do XOR
caps.AlphaCursorSupport    = TRUE;                           // ARGB cursors only
caps.MaxX                  = 256;
caps.MaxY                  = 256;

HANDLE cursorEvent = CreateEvent(nullptr, FALSE, FALSE, nullptr);

IDARG_IN_SETUP_HWCURSOR setup = {};
setup.CursorInfo               = caps;
setup.hNewCursorDataAvailable  = cursorEvent;

NTSTATUS s = IddCxMonitorSetupHardwareCursor(monitor, &setup);
```

Two events the framework signals:

1. `cursorEvent` — fires when **cursor shape** changes. Call
   `IddCxMonitorQueryHardwareCursor` to fetch the new bitmap.
2. There is no separate position event — querying the cursor returns position too;
   for low latency, run a dedicated cursor thread that polls position via the
   shape-update or via a second timer.

### Querying cursor data

```cpp
IDARG_IN_QUERY_HWCURSOR queryIn  = {};
IDARG_OUT_QUERY_HWCURSOR queryOut = {};
queryIn.LastShapeId  = mctx->LastCursorShapeId;
queryIn.pShapeBuffer = mctx->ShapeBuf;
queryIn.ShapeBufferSizeInBytes = sizeof(mctx->ShapeBuf);

HRESULT hr = IddCxMonitorQueryHardwareCursor(monitor, &queryIn, &queryOut);
if (queryOut.IsCursorShapeUpdated) {
    mctx->LastCursorShapeId = queryOut.ShapeInfo.ShapeId;
    SendCursorShape(mctx->ShapeBuf,
                    queryOut.ShapeInfo.Width,
                    queryOut.ShapeInfo.Height,
                    queryOut.ShapeInfo.XHotSpot,
                    queryOut.ShapeInfo.YHotSpot,
                    queryOut.ShapeInfo.Type);
}
SendCursorPosition(queryOut.X, queryOut.Y, queryOut.IsCursorVisible);
```

`ShapeId` is a monotonically-increasing UINT64. Pass the last seen ID; the
framework only refills `pShapeBuffer` if the cursor changed. This avoids copying
the bitmap every poll.

### Shape format

`ShapeInfo.Type` is one of:

| Type | Format |
|------|--------|
| `IDDCX_CURSOR_SHAPE_TYPE_MONOCHROME` | 1bpp AND mask + 1bpp XOR mask, doubled height |
| `IDDCX_CURSOR_SHAPE_TYPE_COLOR` | 32bpp BGRA (alpha is 0 or 255) |
| `IDDCX_CURSOR_SHAPE_TYPE_MASKED_COLOR` | 32bpp BGRA + AND mask |
| `IDDCX_CURSOR_SHAPE_TYPE_ALPHACURSOR` | 32bpp BGRA premultiplied alpha |

Only declare alpha support if you can render premultiplied; the OS will deliver
mostly alpha cursors on Win10/11 desktops with modern themes. Monochrome cursors
appear on legacy apps and console-style sessions.

### Cursor pacing

The framework signals the cursor event much faster than the swap-chain new-frame
event — typically every cursor pixel of motion. Don't process this on the
swap-chain thread; spawn a dedicated cursor thread:

```cpp
DWORD WINAPI CursorThread(LPVOID arg) {
    auto* m = (MonitorCtx*)arg;
    HANDLE evts[] = { m->CursorEvent, m->StopEvent };
    for (;;) {
        DWORD w = WaitForMultipleObjects(2, evts, FALSE, 16);
        if (w == WAIT_OBJECT_0 + 1) break;
        QueryAndSendCursor(m);
    }
    return 0;
}
```

The 16ms timeout polls position in case the framework signals less often than the
cursor moves (rare, but happens during slow motion).

### Hot-spot handling

`XHotSpot`/`YHotSpot` is the in-cursor offset (0..Width-1, 0..Height-1) that
should align with the reported `X`/`Y`. Always pass both to the receiver — Linux
and macOS receivers may need offset compensation when their cursor compositor
expects top-left origin.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Polling cursor on swap-chain thread | Cursor lags up to 16ms behind frame | Dedicated cursor thread |
| Sending shape on every position update | Wastes bandwidth | Only send when `IsCursorShapeUpdated`; cache `ShapeId` |
| Ignoring `IsCursorVisible` | Receiver shows ghost cursor when DWM hides | Hide on receiver when false |
| Allocating bitmap buffer per query | Heap churn at 1kHz | Reuse a fixed `MaxX*MaxY*4` buffer |
| Treating monochrome cursor as 32bpp | Garbage rendering | Switch on `Type`; convert mono to ARGB on driver side if needed |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/iddcx/nf-iddcx-iddcxmonitorqueryhardwarecursor
- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/iddcx/ne-iddcx-iddcx_cursor_shape_type
- https://learn.microsoft.com/en-us/windows-hardware/drivers/display/cursor-handling-overview
