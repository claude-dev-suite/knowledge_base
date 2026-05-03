# Windows Drivers - HID Input Filter - HID Architecture

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/hid-architecture
> **Skill**: dev-suite skill `windows/hid-input-filter` — see SKILL.md for the always-loaded quick reference.

## What this covers

The end-to-end Windows HID transport. The SKILL gives a vertical slice; this file
fills in the horizontal detail: every component that sits between the device and an
application calling `GetRawInputData`, including the user-mode RawInput dispatcher,
which class drivers consume which usage pages, and how PnP enumerates HID children.

## Deep dive

### Transport (mini)drivers

Each physical bus has its own HID transport driver, all of them implementing the
HIDClass *minidriver* contract documented in `hidport.h`. They all sit *under*
`HIDClass.sys` and translate bus-specific traffic into HID transfer packets.

| Transport | Driver | Bus enumerator |
|-----------|--------|----------------|
| USB | `HIDUSB.sys` | USB hub driver enumerates `USB\VID_xxxx&PID_yyyy&MI_zz` |
| Bluetooth | `HIDBTH.sys` (classic), `HIDPRO.sys` (LE) | BT profile driver |
| I2C | `HIDI2C.sys` | ACPI-enumerated, `ACPI\VID_xxxx&...` |
| GATT (BLE) | `Microsoft.Bluetooth.Profiles.HumanInterfaceDevice` UMDF | BLE GATT scan |

### HIDClass and child PDOs

When `HIDClass.sys` finishes loading on top of a transport, it parses the device's
Report Descriptor and creates a *physical device object* (PDO) for every top-level
collection (TLC). A composite gaming mouse with mouse + consumer-control + vendor
TLCs becomes three child PDOs, each with its own hardware ID like
`HID\VID_046D&PID_C52B&Col02`. PnP then loads the right class client on each PDO
based on usage page:

| Usage Page | Usage | Loaded by |
|------------|-------|-----------|
| 0x01 | 0x02 (Mouse) | `Mouclass.sys` |
| 0x01 | 0x06 (Keyboard) | `Kbdclass.sys` |
| 0x0D | 0x04 (Touch screen) | `HidTouch.sys` (built into HIDClass on modern Windows) |
| 0x0C | * (Consumer) | RawInput-only (no class client) |
| 0xFF00+ | vendor | RawInput / WinUSB-style userland |

This is why the SKILL recommends filtering at the HID class level for breadth — by
the time you reach Mouclass, the consumer page is already on a sibling stack.

### RawInput vs Mouclass / Kbdclass

`Win32k!RawInputThread` listens to all HID class clients and to Mouclass/Kbdclass.
For mouse and keyboard, RawInput sees *both* the parsed HID report and the
synthesized `MOUSE_INPUT_DATA` / `KEYBOARD_INPUT_DATA` struct produced by the class
driver — applications using `RIDEV_INPUTSINK | RIDEV_NOLEGACY` get HID data, while
legacy `WM_MOUSEMOVE` / `WM_KEYDOWN` flow only through GDI after Mouclass posts.

### IRP plumbing between layers

All inter-layer traffic is **internal IOCTLs** (`IRP_MJ_INTERNAL_DEVICE_CONTROL`),
never regular IOCTLs. Class clients (Mouclass, Kbdclass, your filter) submit
`IOCTL_HID_READ_REPORT` with no input buffer and a system buffer sized to
`HidP_GetCaps().InputReportByteLength`. The transport driver pends the IRP until
the device produces a report, then completes it asynchronously. RawInput keeps
several outstanding READ_REPORTs at all times to avoid losing reports.

```c
// HIDClass uses this loop internally; a filter mirrors it
for (i = 0; i < HID_OUTSTANDING_READS; ++i) {
    PIRP irp = IoAllocateIrp(stackSize, FALSE);
    IO_STACK_LOCATION* sl = IoGetNextIrpStackLocation(irp);
    sl->MajorFunction = IRP_MJ_INTERNAL_DEVICE_CONTROL;
    sl->Parameters.DeviceIoControl.IoControlCode = IOCTL_HID_READ_REPORT;
    sl->Parameters.DeviceIoControl.OutputBufferLength = caps.InputReportByteLength;
    irp->UserBuffer = ReportBuffer[i];
    IoSetCompletionRoutine(irp, OnReadCompleted, ctx, TRUE, TRUE, TRUE);
    IoCallDriver(LowerDevice, irp);
}
```

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Filtering at the wrong layer for consumer-control keys | Volume / media keys never reach Mouclass or Kbdclass — they only flow as HID + RawInput | Filter on the HID class above HIDClass.sys |
| Assuming one PDO per device | TLC fan-out creates N children; your filter must attach to each PDO that matters | Match the right `Col0x` hardware-id suffix in your INF |
| Treating I2C HID like USB HID | I2C HID has different power-management semantics (SET_POWER, SLEEP) at the transport layer | Don't intercept `IOCTL_HID_*_POWER` from above HIDClass |
| One outstanding READ_REPORT IRP | Reports get dropped under burst load | Keep 4-8 IRPs in flight, recycle on completion |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/hid-clients-supported-in-windows
- https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/top-level-collections
- https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/hid-transports
