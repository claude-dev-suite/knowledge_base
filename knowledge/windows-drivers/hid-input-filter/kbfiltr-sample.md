# Windows Drivers - HID Input Filter - kbfiltr Sample

> **Source**: https://github.com/microsoft/Windows-driver-samples/tree/main/input/kbfiltr
> **Skill**: dev-suite skill `windows/hid-input-filter` — see SKILL.md for the always-loaded quick reference.

## What this covers

Tour of the canonical Microsoft kbfiltr KMDF sample: where each callback lives,
how the connect/disconnect dance with Kbdclass works, and the scan-code rewrite
pattern. The SKILL gives a generic filter skeleton; this file walks the specific
Kbdclass mechanics.

## Deep dive

### Files and entry points

The sample is small — five real files:

| File | Role |
|------|------|
| `kbfiltr.c` | DriverEntry, EvtDeviceAdd, IOCTL dispatch |
| `kbfiltr.h` | Context structs, IOCTL definitions |
| `kbfiltr.inx` | INF (becomes kbfiltr.inf at install time) |
| `wmi.c` | WMI provider (optional, often stripped) |
| `kbfiltr.ctl` | Control-app interface IOCTLs |

### The Kbdclass connect IRP

Kbdclass talks to its lower stack via a single private IOCTL during start:
`IOCTL_INTERNAL_KEYBOARD_CONNECT`. The IRP carries a `CONNECT_DATA` struct with a
function pointer (`ClassService`) and a callback context (`ClassDeviceObject`). Any
filter must:

1. Intercept the IOCTL on the way down.
2. Save the original `ClassService` (e.g. `Kbdclass!KeyboardClassServiceCallback`)
   in its device context.
3. Replace `ClassService` with its own `KbFilter_ServiceCallback`.
4. Forward the IRP. Kbdclass sees the *filter's* callback as the upstream entry
   point.

```c
NTSTATUS KbFilter_DispatchPassThrough(WDFREQUEST Request) {
    PCONNECT_DATA connect;
    NTSTATUS s = WdfRequestRetrieveInputBuffer(Request, sizeof(*connect),
                                               &connect, NULL);
    if (NT_SUCCESS(s) && CurrentIoControlCode == IOCTL_INTERNAL_KEYBOARD_CONNECT) {
        ctx->UpperConnectData = *connect;        // Kbdclass's real callback
        connect->ClassDeviceObject = WdfDeviceWdmGetDeviceObject(device);
        connect->ClassService      = KbFilter_ServiceCallback;
    }
    return SendDownTheStack(Request);
}
```

When the underlying keyboard fires an interrupt, the chain becomes:

```
i8042prt / hidclass → KbFilter_ServiceCallback → Kbdclass → RawInput
```

Inside `KbFilter_ServiceCallback`, the filter receives an array of
`KEYBOARD_INPUT_DATA` structs (NOT raw HID reports — Kbdclass already
synthesized them):

```c
typedef struct _KEYBOARD_INPUT_DATA {
    USHORT UnitId;
    USHORT MakeCode;        // PS/2-style scan code
    USHORT Flags;           // KEY_BREAK, KEY_E0, KEY_E1
    USHORT Reserved;
    ULONG  ExtraInformation;
} KEYBOARD_INPUT_DATA;
```

### Scan-code rewriting

The sample's `KbFilter_ServiceCallback` walks the array and can drop, modify, or
reorder events before invoking the upstream callback:

```c
VOID KbFilter_ServiceCallback(
    PDEVICE_OBJECT DeviceObject,
    PKEYBOARD_INPUT_DATA InputDataStart,
    PKEYBOARD_INPUT_DATA InputDataEnd,
    PULONG InputDataConsumed)
{
    PDEVICE_EXTENSION ext = ((WDFDEVICE_FILTER_EXT*)DeviceObject->DeviceExtension)->Ext;
    for (PKEYBOARD_INPUT_DATA p = InputDataStart; p < InputDataEnd; ++p) {
        // Example: swap CapsLock (0x3A) with LeftCtrl (0x1D)
        if (p->MakeCode == 0x3A) p->MakeCode = 0x1D;
        else if (p->MakeCode == 0x1D) p->MakeCode = 0x3A;
    }
    (*(PSERVICE_CALLBACK_ROUTINE)ext->UpperConnectData.ClassService)(
        ext->UpperConnectData.ClassDeviceObject,
        InputDataStart, InputDataEnd, InputDataConsumed);
}
```

`InputDataConsumed` lets the filter *swallow* events: set it shorter than the
range to drop trailing entries. Set it to zero to drop all.

### IOCTL surface for a control app

`kbfiltr.ctl` defines IOCTLs the user-mode test app uses to inject scan codes
(`IOCTL_KBFILTR_SYNTHESIZE_SCAN_CODE`). The driver builds a synthetic
`KEYBOARD_INPUT_DATA` array and calls the upstream Kbdclass callback. This is the
reverse direction — synthesizing keystrokes that look like they came from the
keyboard.

Note: in 2024+, the supported way to inject keyboard input is **VHF**, not
poking Kbdclass. The kbfiltr sample is older and remains illustrative for the
*intercept* path.

### INF specifics

```inf
[KbFilter_AddReg]
HKR,,"UpperFilters",0x00010008,"kbfiltr"     ; REG_MULTI_SZ append
```

`0x00010008` = `FLG_ADDREG_TYPE_MULTI_SZ | FLG_ADDREG_APPEND` — appends without
clobbering existing class filters.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Calling `UpperConnectData.ClassService` from raised IRQL with paged data | Kbdclass callback runs at DISPATCH; passing paged buffers crashes | Keep input data in nonpaged context |
| Forgetting to save `UpperConnectData` before forwarding | Kbdclass overwrites it on the down call | Capture before the IRP completes |
| Filtering by VK code | The callback sees scan codes (MakeCode), not VKs — VK mapping happens in user mode | Match on PS/2 make codes + KEY_E0/E1 flags |
| Replacing `ClassDeviceObject` but not `ClassService` | Kbdclass calls its own callback with your device object → bugcheck | Replace the pair atomically |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/kbdmou/ns-kbdmou-_keyboard_input_data
- https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/keyboard-and-mouse-class-drivers
- https://github.com/microsoft/Windows-driver-samples/blob/main/input/kbfiltr/sys/kbfiltr.c
