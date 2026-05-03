# Windows Drivers - HID Input Filter - firefly Sample

> **Source**: https://github.com/microsoft/Windows-driver-samples/tree/main/hid/firefly
> **Skill**: dev-suite skill `windows/hid-input-filter` — see SKILL.md for the always-loaded quick reference.

## What this covers

Tour of the `firefly` KMDF sample: a HID filter for the Razer Copperhead-class
mouse that toggles its LED via *feature reports*. Demonstrates the SET_FEATURE
direction of the IOCTL plumbing, INF installation as a per-device upper filter,
and a user-mode control surface. The SKILL focuses on intercepting input; this
file covers the orthogonal feature-report-write path.

## Deep dive

### Why firefly exists

The original Razer Copperhead exposed a vendor feature report for LED control. No
generic Windows mechanism fits — the keyboard/mouse class drivers ignore the
vendor TLC. firefly attaches as a per-device upper filter and provides a custom
control device that user-mode applets open with `CreateFile` to send feature
reports.

### Two device objects

The driver creates:

1. The **filter device** — attached above HIDClass for the mouse PDO. Its job is
   to learn the mouse's `WDFIOTARGET` (the lower I/O target it can send feature
   reports to).
2. A **control device** — a separate WDF device with a symlink (e.g.
   `\\.\Firefly`) that user mode opens to send `IOCTL_FIREFLY_SET_BRIGHTNESS`.

```c
// Create the user-mode control device once at DriverEntry-time
WDFDEVICE control;
PWDFDEVICE_INIT init = WdfControlDeviceInitAllocate(driver, &SDDL_DEVOBJ_SYS_ALL_ADM_ALL);
WdfDeviceInitSetDeviceType(init, FILE_DEVICE_UNKNOWN);
WdfDeviceInitSetExclusive(init, FALSE);
WdfDeviceCreate(&init, &attrs, &control);
WdfDeviceCreateSymbolicLink(control, &symlinkName);
```

### Sending a feature report from kernel mode

The filter exposes a callback the control device invokes. Internally it builds a
preparsed-data lookup, fills a `HID_XFER_PACKET`, and sends `IOCTL_HID_SET_FEATURE`
down its own filter stack:

```c
NTSTATUS Firefly_SetFeature(WDFDEVICE Filter, UCHAR ReportId, BOOLEAN On) {
    PDEVICE_CONTEXT ctx = GetContext(Filter);
    HIDP_CAPS caps;
    HidP_GetCaps(ctx->PreparsedData, &caps);

    UCHAR report[FEATURE_REPORT_SIZE] = { ReportId };
    report[1] = On ? 0x01 : 0x00;        // device-specific format

    HID_XFER_PACKET packet = {
        .reportBuffer  = report,
        .reportBufferLen = sizeof(report),
        .reportId      = ReportId
    };
    WDF_MEMORY_DESCRIPTOR mem;
    WDF_MEMORY_DESCRIPTOR_INIT_BUFFER(&mem, &packet, sizeof(packet));
    return WdfIoTargetSendInternalIoctlSynchronously(
        WdfDeviceGetIoTarget(Filter),
        NULL,
        IOCTL_HID_SET_FEATURE,
        &mem, NULL, NULL, NULL);
}
```

`IOCTL_HID_SET_FEATURE` carries a `HID_XFER_PACKET`, not a raw byte array — the
HID subsystem uses `HID_XFER_PACKET` as a uniform envelope across read/write
report variants.

### Per-device INF (production-grade)

```inf
[Standard.NT$ARCH$.10.0...19041]
%DeviceDesc% = Firefly_Inst, HID\Vid_1532&Pid_0101&Col01

[Firefly_Inst.NT]
Include = input.inf
Needs   = HID_Inst.NT
AddReg  = Firefly_Inst.AddReg

[Firefly_Inst.AddReg]
HKR,,"UpperFilters",0x00010000,"firefly"
```

The `Include`/`Needs` pair lets your filter ride on top of the standard HID device
install — your filter just adds itself, doesn't redo the function-driver setup.

### Why this isn't a class filter

If you registered `firefly` on the HID class GUID, it would attach to every HID
device on the box and try to send vendor feature reports — bricking some, hanging
others. **Per-device** filters are essential for vendor-specific quirks. The
sample's hardware-id filter (`HID\Vid_1532&Pid_0101&Col01`) targets exactly one
TLC of one VID/PID.

### Concurrency

A user app may call `IOCTL_FIREFLY_SET_BRIGHTNESS` while the device is suspending.
Wrap feature-report submission in `WdfDeviceStopIdle` / `WdfDeviceResumeIdle` so
the device powers up first:

```c
WdfDeviceStopIdle(Filter, TRUE);    // wait for D0
NTSTATUS s = Firefly_SetFeature(Filter, REPORT_ID_LED, on);
WdfDeviceResumeIdle(Filter);
```

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Sending feature reports without first powering the device | D3 → no USB transfers | `WdfDeviceStopIdle` before, `WdfDeviceResumeIdle` after |
| Hard-coding VID/PID in code, not INF | Driver loads on every HID device, then noops | Filter binding in INF; keep code generic |
| Reading user-mode buffer at DISPATCH | Page fault possible | Mark control-device queue `DispatchSequential` and run at PASSIVE |
| Forgetting `WDFIOTARGET` of the filter | Sending the IOCTL down the wrong stack | Use `WdfDeviceGetIoTarget(filter)` — never the control device's |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/sending-a-hid-feature-report
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/controlling-device-access-in-kmdf-drivers
- https://github.com/microsoft/Windows-driver-samples/blob/main/hid/firefly/sys/firefly.c
