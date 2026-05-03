# Windows Drivers - HID Input Filter - Internal IOCTLs

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/hid-collections
> **Skill**: dev-suite skill `windows/hid-input-filter` — see SKILL.md for the always-loaded quick reference.

## What this covers

Full catalog of `IOCTL_HID_*` codes, their buffering method, who issues each, and
how a filter inspects/modifies their buffers safely. The SKILL lists the headline
seven; this file covers the long tail and the buffer-method nuance that matters
when you reach into the IRP at completion time.

## Deep dive

### Complete IOCTL catalog

Defined in `hidclass.h` and `hidport.h`:

| IOCTL | Method | Direction | Notes |
|-------|--------|-----------|-------|
| `IOCTL_HID_GET_DEVICE_DESCRIPTOR` | NEITHER | top→bot | Returns 9-byte HID descriptor (`HID_DESCRIPTOR`) |
| `IOCTL_HID_GET_REPORT_DESCRIPTOR` | NEITHER | top→bot | Variable-length raw report descriptor |
| `IOCTL_HID_GET_DEVICE_ATTRIBUTES` | NEITHER | top→bot | Fills `HID_DEVICE_ATTRIBUTES` (VID/PID/Version) |
| `IOCTL_HID_READ_REPORT` | NEITHER | top→bot pended | Most common — pends in minidriver until report arrives |
| `IOCTL_HID_WRITE_REPORT` | NEITHER | top→bot | Output report; first byte is report ID |
| `IOCTL_HID_GET_INPUT_REPORT` | NEITHER | top→bot | Polls; not the same as READ_REPORT |
| `IOCTL_HID_SET_OUTPUT_REPORT` | NEITHER | top→bot | Synchronous output; bypasses INTERRUPT pipe |
| `IOCTL_HID_GET_FEATURE` | OUT_DIRECT | top→bot | Configuration read |
| `IOCTL_HID_SET_FEATURE` | IN_DIRECT | top→bot | Configuration write (e.g. firefly LED) |
| `IOCTL_HID_GET_STRING` | NEITHER | top→bot | String index in `Type1Input` |
| `IOCTL_HID_GET_INDEXED_STRING` | NEITHER | top→bot | UTF-16 string by index |
| `IOCTL_HID_GET_MS_GENRE_DESCRIPTOR` | NEITHER | top→bot | MS extension; rare |
| `IOCTL_HID_ACTIVATE_DEVICE` | NEITHER | bot→dev | Minidriver-internal |
| `IOCTL_HID_DEACTIVATE_DEVICE` | NEITHER | bot→dev | Minidriver-internal |
| `IOCTL_HID_SEND_IDLE_NOTIFICATION_REQUEST` | NEITHER | top→bot | Power policy |

### Buffer methods and how to read them

**METHOD_NEITHER** is dominant in the HID stack. The pointers live in
`Irp->UserBuffer` (output) and `Parameters.DeviceIoControl.Type3InputBuffer`
(input). Because no probing is done by the I/O manager, *only call from
PASSIVE_LEVEL or ensure the request originates from kernel mode*. HID class
clients (Mouclass, Kbdclass, RawInput) all live in kernel mode, so HID filters can
trust the pointers — but a filter that exposes a user-mode interface of its own
must `ProbeForRead`/`ProbeForWrite`.

```c
VOID OnReadReportCompleted(WDFREQUEST Request, WDFIOTARGET Target,
                           PWDF_REQUEST_COMPLETION_PARAMS Params, WDFCONTEXT Ctx)
{
    if (!NT_SUCCESS(Params->IoStatus.Status)) {
        WdfRequestComplete(Request, Params->IoStatus.Status);
        return;
    }
    PIRP irp = WdfRequestWdmGetIrp(Request);
    PVOID  buf = irp->UserBuffer;                        // METHOD_NEITHER
    SIZE_T len = (SIZE_T)Params->IoStatus.Information;   // bytes filled
    InspectOrRewriteReport(buf, len);
    WdfRequestCompleteWithInformation(Request, STATUS_SUCCESS, len);
}
```

**METHOD_IN_DIRECT / METHOD_OUT_DIRECT** for feature reports give you an
`Irp->MdlAddress`; map with `MmGetSystemAddressForMdlSafe(mdl, NormalPagePriority)`.

### Pended IRPs and cancellation

`IOCTL_HID_READ_REPORT` is the only IOCTL routinely pended for seconds at a time.
Class clients keep multiple in flight; on shutdown they cancel all of them. A
filter that completes IRPs in a completion routine doesn't need to handle cancel.
A filter that *holds* an IRP (e.g. for inverted call) must register a cancel
routine with `WdfRequestMarkCancelable` / `WdfRequestUnmarkCancelable`.

### IRP completion patterns

Three common patterns:

1. **Pass-through (default)** — `WdfRequestSend` with `SEND_AND_FORGET`. The
   filter is invisible for that IOCTL.
2. **Pre-process and forward** — modify input buffer, format, send, forget. Used
   for SET_FEATURE rewrites (e.g. force a specific LED brightness).
3. **Forward and post-process** — set a completion routine, send, on completion
   inspect/modify output, then complete upward. Standard for READ_REPORT
   inspection.

A fourth pattern, "fail it locally," is sometimes useful — for example to pretend
a device is detached (complete `READ_REPORT` with `STATUS_DEVICE_NOT_CONNECTED`).
Class clients will tear down their handle and the device disappears from
RawInput.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Touching `Irp->UserBuffer` in a completion routine without checking status | Minidriver may have completed with error and zero-length data | Branch on `Params->IoStatus.Status` first |
| Returning success but wrong `Information` count | RawInput truncates or ignores the report | Set `WdfRequestCompleteWithInformation(req, STATUS_SUCCESS, validLen)` |
| Probing METHOD_NEITHER buffers from kernel callers | Causes assertion in checked builds | Probe only when `Irp->RequestorMode == UserMode` |
| Calling `MmGetSystemAddressForMdlSafe` without checking NULL | Low-memory path returns NULL — bugcheck if dereferenced | Always test, fail with `STATUS_INSUFFICIENT_RESOURCES` |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/buffer-descriptions-for-i-o-control-codes
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/forwarding-i-o-requests
- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/hidclass/
