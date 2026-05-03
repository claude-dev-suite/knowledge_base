# Windows Drivers - HID Input Filter - Virtual HID Framework (VHF)

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/virtual-hid-framework--vhf-
> **Skill**: dev-suite skill `windows/hid-input-filter` — see SKILL.md for the always-loaded quick reference.

## What this covers

Detailed VHF lifecycle: how a kernel client creates a synthetic HID source on the
PnP tree, when the framework calls back for output/feature requests, and the
buffering rules for `VhfReadReportSubmit`. The SKILL gives a 6-line skeleton; this
file covers async report I/O, feature-report sinks, and lifetime concerns that
trip people up.

## Deep dive

### What VHF actually creates

`VhfCreate` makes the framework instantiate a virtual `HIDClass` PDO under the
caller's WDF device. From PnP's perspective the device is a real HID device —
RawInput, Mouclass/Kbdclass, and any other class client see it normally. There is
no underlying transport (no USB/I2C); reports are produced and consumed by the
caller in software.

Use cases:

- Keystroke / mouse injection (replacement for `SendInput` when you need driver-
  level credibility, e.g. gaming peripherals that watch for SendInput hooks).
- Mapping a non-HID input source (sensor over network, kiosk button board) into
  the HID stack so existing tools work.
- Test harnesses that fake a Razer-quality mouse or a touch controller.

### Lifetime

```c
PUCHAR descriptor = MyReportDescriptorBytes;
USHORT descSize   = sizeof(MyReportDescriptorBytes);

VHF_CONFIG cfg;
VHF_CONFIG_INIT(&cfg, WdfDeviceWdmGetDeviceObject(device), descSize, descriptor);
cfg.OperationContextSize        = sizeof(MY_OP_CTX);
cfg.EvtVhfAsyncOperationGetFeature = OnGetFeature;     // optional sinks
cfg.EvtVhfAsyncOperationSetFeature = OnSetFeature;
cfg.EvtVhfAsyncOperationWriteReport= OnWriteReport;
cfg.EvtVhfReadyForNextReadReport   = OnReadyForNextRead;

VHFHANDLE h;
NTSTATUS s = VhfCreate(&cfg, &h);
if (NT_SUCCESS(s)) s = VhfStart(h);
// ...later, in shutdown:
VhfDelete(h, TRUE);   // TRUE = wait for all pending callbacks to finish
```

The `OperationContextSize` carves out per-async-operation storage that the
framework hands back as `VhfOperationHandle`'s context — useful for tracking
which feature request to complete.

### Submitting input reports

Use `VhfReadReportSubmit` whenever you have a new input report ready. The
framework copies the buffer; you can free it on return.

```c
HID_XFER_PACKET pkt;
RtlZeroMemory(&pkt, sizeof(pkt));
pkt.reportBuffer    = mouseReport;        // 5 bytes: ID, btns, dx, dy, wheel
pkt.reportBufferLen = sizeof(mouseReport);
pkt.reportId        = mouseReport[0];
NTSTATUS s = VhfReadReportSubmit(g_VhfHandle, &pkt);
```

Submission is **non-blocking** — internally VHF queues the report and signals
upstream class clients. There is no flow-control if you over-submit; reports are
dropped silently when the upper-edge queue fills. To pace correctly, listen for
`EvtVhfReadyForNextReadReport` (signals "I drained at least one") before
submitting the next one.

### Output and feature reports (async sinks)

When a class client calls `IOCTL_HID_SET_OUTPUT_REPORT` or `IOCTL_HID_SET_FEATURE`
on the virtual device, VHF invokes the configured `EvtVhfAsync*` callback.
Operations are *async* — the callback receives a `VhfOperationHandle` you must
complete with `VhfAsyncOperationComplete`:

```c
VOID OnSetFeature(PVOID Ctx, VHFOPERATIONHANDLE Op, ULONG_PTR Cookie,
                  PHID_XFER_PACKET Packet)
{
    UNREFERENCED_PARAMETER(Cookie);
    if (Packet->reportId == REPORT_ID_LED) {
        SetLedAsync(Packet->reportBuffer[1]);   // your hardware
    }
    VhfAsyncOperationComplete(Op, STATUS_SUCCESS);
}
```

You can complete inline (synchronously) or hand `Op` to a workitem and complete
later. Either way the ULONG_PTR `Cookie` is yours to use.

### Shutdown ordering

`VhfDelete(h, TRUE)`:

1. Stops the framework from accepting new I/O.
2. Cancels pending read/feature requests (class clients see `STATUS_CANCELLED`).
3. Waits for all in-flight `Evt*` callbacks to return — that's why `TRUE`.
4. Releases the synthetic PDO.

If you're mid-callback when shutdown starts, the framework blocks on the wait.
Never call `VhfDelete` from inside a VHF callback — guaranteed deadlock.

### Driver requirements

- **Min OS**: Windows 10 1607 (Redstone 1) for Vhf.sys.
- **WDM or KMDF**: VHF takes a `PDEVICE_OBJECT`, so it works for either, but
  KMDF callers should pass `WdfDeviceWdmGetDeviceObject(device)`.
- **INF**: List `Vhf.sys` as a service dependency:
  ```inf
  [MyDriver.NT.Services]
  AddService = MyDriver,0x00000002,Service_Inst
  AddService = vhf,,VhfServiceInst
  [VhfServiceInst]
  DisplayName = "Virtual HID Framework"
  ServiceType = 1
  StartType   = 3
  ServiceBinary = %12%\vhf.sys
  ```

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Submitting reports faster than RawInput drains | Drops occur silently | Pace via `EvtVhfReadyForNextReadReport` |
| Forgetting `VhfStart` after `VhfCreate` | Device exists but stays disabled | Call `VhfStart` before first submit |
| Calling `VhfDelete` from inside a callback | Self-wait deadlock | Defer to a separate cleanup workitem |
| Building a descriptor with array Input items for keyboards | VK lookup goes through Kbdclass — needs a usage table that matches your scancode synthesis | Mirror Microsoft's standard 101-key descriptor unless you have a reason |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/vhf/
- https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/creating-a-virtual-hid-device-with-vhf
- https://github.com/microsoft/Windows-driver-samples/tree/main/hid/vhidmini2
