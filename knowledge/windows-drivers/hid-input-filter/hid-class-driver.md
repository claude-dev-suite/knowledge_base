# Windows Drivers - HID Input Filter - HIDClass.sys

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/hid-class-driver
> **Skill**: dev-suite skill `windows/hid-input-filter` — see SKILL.md for the always-loaded quick reference.

## What this covers

The internal contract of `HIDClass.sys`: how it presents two faces (a top-edge to
class clients/RawInput, a bottom-edge to transport minidrivers), which IOCTLs cross
each edge, and where filter drivers can and can't intercept. The SKILL shows the
filter pattern; this file explains *why* the pattern works.

## Deep dive

### Two-edged driver

HIDClass is unusual: it sits between transport-specific minidrivers (HIDUSB,
HIDBTH, HIDI2C, ...) and class clients (Mouclass, Kbdclass, RawInput, your
WinUSB-like user app). It implements:

- A **bottom edge** that calls into the minidriver via `HID_MINIDRIVER_REGISTRATION`
  and the `IOCTL_HID_*` private interface.
- A **top edge** that exposes per-TLC PDOs to PnP. Class clients open these PDOs
  with `CreateFile` / `IoGetDeviceObjectPointer` and submit the same `IOCTL_HID_*`
  codes — but those go *down* into HIDClass, which translates and forwards.

```
   ReadFile / IOCTL_HID_READ_REPORT (top edge, IRP_MJ_DEVICE_CONTROL)
                          │
                  HIDClass.sys (PDO per TLC)
                          │
   IOCTL_HID_READ_REPORT (bottom edge, IRP_MJ_INTERNAL_DEVICE_CONTROL)
                          │
                  HIDUSB.sys (minidriver FDO)
                          │
                       USB stack
```

A filter installed *above* HIDClass on the HID class GUID sees the **bottom-edge**
IOCTLs (`IRP_MJ_INTERNAL_DEVICE_CONTROL` carrying `IOCTL_HID_READ_REPORT` etc.). A
filter installed *above* the per-TLC PDO (above Mouclass / inside the function
stack of the child) sees a different surface — usually class-specific IOCTLs.

### Report Descriptor parsing

On `START_DEVICE`, HIDClass issues `IOCTL_HID_GET_REPORT_DESCRIPTOR` to the
minidriver, then walks the descriptor with the public `HidP_*` API
(`HidP_GetCaps`, `HidP_GetButtonCaps`, `HidP_GetValueCaps`,
`HidP_GetUsageValue`, `HidP_GetData`). The parsed `HIDP_PREPARSED_DATA` is what
class clients receive when they call `HidD_GetPreparsedData`.

A filter that wants to *modify* reports (e.g. invert Y, swap buttons) must use the
same parser to find the right offset in the variable-length report — never assume
byte 1 is X, byte 2 is Y. Real descriptors interleave, pad, and use multi-byte
fields.

```c
HIDP_CAPS caps;
HidP_GetCaps(preparsedData, &caps);
HIDP_VALUE_CAPS valueCaps[16];
USHORT count = ARRAYSIZE(valueCaps);
HidP_GetValueCaps(HidP_Input, valueCaps, &count, preparsedData);
for (USHORT i = 0; i < count; ++i) {
    if (valueCaps[i].UsagePage == 0x01 && valueCaps[i].NotRange.Usage == 0x30) {
        // X axis - rewrite via HidP_SetUsageValue
        HidP_SetUsageValue(HidP_Input, 0x01, 0, 0x30, newX,
                           preparsedData, report, reportLength);
    }
}
```

### Async report queue

HIDClass maintains per-collection queues of pending input reports. If reports
arrive faster than the class client drains, the queue depth is governed by the
`MaxInputReport` registry value (default 32). Filters that complete reads quickly
help keep the queue short; filters that synchronously call out to user-mode
through inverted call should keep that path lock-free.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Filter installed on `Mouse` class but expects HID-format reports | Mouclass already converted to `MOUSE_INPUT_DATA` | Install on HID class instead, or read `MOUSE_INPUT_DATA` shape |
| Hand-decoding report bytes by offset | Descriptor changes between firmware revs break the filter | Use `HidP_*` against the device's preparsed data |
| Modifying a report during DPC | Report buffer may be paged or guarded | Modify in completion routine at IRQL ≤ DISPATCH, never above |
| Assuming all HID devices have report IDs | Single-collection devices often skip report ID byte 0 | Branch on `caps.NumberInputReportIDs` |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/parsing-a-device-s-report-descriptor
- https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/preparsed-data
- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/hidpi/
