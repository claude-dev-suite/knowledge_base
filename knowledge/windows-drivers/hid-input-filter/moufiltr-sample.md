# Windows Drivers - HID Input Filter - moufiltr Sample

> **Source**: https://github.com/microsoft/Windows-driver-samples/tree/main/input/moufiltr
> **Skill**: dev-suite skill `windows/hid-input-filter` — see SKILL.md for the always-loaded quick reference.

## What this covers

Walk-through of Microsoft's `moufiltr` KMDF sample — the mouse-class equivalent of
`kbfiltr`. Covers the connect IOCTL, `MOUSE_INPUT_DATA` semantics, button bit
manipulation, and the trade-offs of filtering above Mouclass vs above HIDClass for
mouse-only behavior.

## Deep dive

### Sample shape

| File | Role |
|------|------|
| `moufiltr.c` | KMDF entry, callback registration |
| `moufiltr.h` | Context, IOCTL defs |
| `moufiltr.inx` | INF — installs as upper class filter on `Mouse` GUID |
| `wmi.c` | Optional WMI provider |

### The Mouclass connect IOCTL

`IOCTL_INTERNAL_MOUSE_CONNECT` carries a `CONNECT_DATA` exactly like keyboard,
just with a different prototype for the service callback:

```c
typedef struct _MOUSE_INPUT_DATA {
    USHORT UnitId;
    USHORT Flags;            // MOUSE_MOVE_RELATIVE / _ABSOLUTE / _VIRTUAL_DESKTOP
    union {
        ULONG Buttons;
        struct { USHORT ButtonFlags; USHORT ButtonData; };
    };
    ULONG  RawButtons;
    LONG   LastX;
    LONG   LastY;
    ULONG  ExtraInformation;
} MOUSE_INPUT_DATA;
```

`ButtonFlags` are *transitions* — `MOUSE_LEFT_BUTTON_DOWN`, `_LEFT_BUTTON_UP`,
etc. — not steady-state buttons. This catches people: filtering "left-button held
+ X movement" requires tracking state across calls because Mouclass only sends
transitions.

### Service callback signature

```c
typedef VOID (*PSERVICE_CALLBACK_ROUTINE)(
    PVOID NormalContext,             // ClassDeviceObject
    PVOID InputDataStart,            // PMOUSE_INPUT_DATA[]
    PVOID InputDataEnd,
    PULONG InputDataConsumed);
```

The sample replaces it with `MouFilter_ServiceCallback` and forwards to the saved
upstream pointer:

```c
VOID MouFilter_ServiceCallback(
    PDEVICE_OBJECT DeviceObject,
    PMOUSE_INPUT_DATA InputDataStart,
    PMOUSE_INPUT_DATA InputDataEnd,
    PULONG InputDataConsumed)
{
    PDEVICE_EXTENSION ext = GetExt(DeviceObject);
    for (PMOUSE_INPUT_DATA p = InputDataStart; p < InputDataEnd; ++p) {
        // Example: invert vertical axis
        p->LastY = -p->LastY;
        // Example: turn middle-click into RBUTTON_DOWN
        if (p->ButtonFlags & MOUSE_MIDDLE_BUTTON_DOWN) {
            p->ButtonFlags &= ~MOUSE_MIDDLE_BUTTON_DOWN;
            p->ButtonFlags |=  MOUSE_RIGHT_BUTTON_DOWN;
        }
    }
    (*(PSERVICE_CALLBACK_ROUTINE)ext->UpperConnectData.ClassService)(
        ext->UpperConnectData.ClassDeviceObject,
        InputDataStart, InputDataEnd, InputDataConsumed);
}
```

### Above Mouclass vs above HIDClass

`moufiltr` lives above Mouclass — that means you only see *mouse* TLCs, after
parsing into deltas. You will **not** see:

- Touchpad pen / touch events (different class clients)
- Gaming-mouse "extra button" usages on the consumer page (RawInput-only)
- Reports that Mouclass dropped because `Logical Min/Max` clamping rejected them

If your goal is "intercept everything that looks like pointer motion regardless of
device kind", filter above HIDClass and parse the descriptor yourself. If your
goal is "rewrite mouse-only behavior with minimal code", `moufiltr` is the model.

### Coordinate semantics

`Flags` controls the meaning of `LastX`/`LastY`:

| Flag | LastX/Y meaning |
|------|------------------|
| `MOUSE_MOVE_RELATIVE` | Delta in mickeys since the last report |
| `MOUSE_MOVE_ABSOLUTE` | Absolute coordinate in [0..65535] mapped to primary screen |
| `MOUSE_VIRTUAL_DESKTOP` | Absolute coordinate mapped across all monitors |

Touchscreens and tablets typically send `_ABSOLUTE | _VIRTUAL_DESKTOP`. Mice send
`_RELATIVE`. A filter that scales motion must respect this — multiplying absolute
coords by a sensitivity factor is a bug.

### ExtraInformation field

`ExtraInformation` is a 32-bit cookie that flows from the driver up through
RawInput and `WM_MOUSEMOVE`. Apps can match on it via `GetMessageExtraInfo`. A
filter can use this to *tag* events so a cooperating user-mode app can distinguish
filter-injected from real:

```c
p->ExtraInformation = MY_FILTER_TAG;     // unique 32-bit cookie
```

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Treating `Buttons`/`ButtonFlags` as steady state | They are edge transitions only | Maintain your own button-state shadow if you need "is held" |
| Multiplying `LastX`/`LastY` regardless of `Flags` | Breaks touch and absolute-positioning devices | Check `Flags & MOUSE_MOVE_RELATIVE` first |
| Filtering wheel deltas as `LastX`/`LastY` | Wheel uses `ButtonData` with `MOUSE_WHEEL` flag | Branch on `MOUSE_WHEEL` / `MOUSE_HWHEEL` |
| Sample's INF installs class-wide on `Mouse` GUID | Affects every mouse and touchpad | Switch to per-device INF for production |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/kbdmou/ns-kbdmou-_mouse_input_data
- https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/mouse-input-overview
- https://github.com/microsoft/Windows-driver-samples/blob/main/input/moufiltr/moufiltr.c
