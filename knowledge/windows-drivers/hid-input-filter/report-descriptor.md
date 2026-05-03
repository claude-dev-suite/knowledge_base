# Windows Drivers - HID Input Filter - Report Descriptor

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/getting-hid-reports
> **Skill**: dev-suite skill `windows/hid-input-filter` - see SKILL.md for the always-loaded quick reference.

## What this covers

The actual byte-for-byte structure of HID Report Descriptors and how to navigate
them with the `HidP_*` API. The SKILL points at descriptors as an anti-pattern to
hard-code; this file gives the parser internals you need when you intercept and
rewrite reports.

## Deep dive

### Item types

A report descriptor is a stream of items, each 1 to 5 bytes. The leading byte is
`{tag, type, size}`:

```
Bits 7-4: tag    (e.g. Usage Page, Usage, Logical Min)
Bits 3-2: type   (Main=0, Global=1, Local=2)
Bits 1-0: size   (0,1,2,4 bytes of data follow)
```

Three categories:

- **Main items**: `Input`, `Output`, `Feature`, `Collection`, `End Collection`.
  These are the only ones that consume the descriptor state machine and produce
  fields in the report.
- **Global items**: `Usage Page`, `Logical Min/Max`, `Physical Min/Max`,
  `Report Size`, `Report Count`, `Report ID`. Persist across Main items until
  changed.
- **Local items**: `Usage`, `Usage Min/Max`. Reset after every Main item.

### Worked example: minimal mouse

```
05 01           Usage Page (Generic Desktop)
09 02           Usage (Mouse)
A1 01           Collection (Application)
  09 01         Usage (Pointer)
  A1 00         Collection (Physical)
    05 09       Usage Page (Buttons)
    19 01       Usage Min (Button 1)
    29 03       Usage Max (Button 3)
    15 00       Logical Min (0)
    25 01       Logical Max (1)
    95 03       Report Count (3)
    75 01       Report Size (1)
    81 02       Input (Data,Variable,Absolute)        // 3 button bits
    95 01       Report Count (1)
    75 05       Report Size (5)
    81 03       Input (Const,Variable,Absolute)       // 5 padding bits
    05 01       Usage Page (Generic Desktop)
    09 30       Usage (X)
    09 31       Usage (Y)
    15 81       Logical Min (-127)
    25 7F       Logical Max ( 127)
    75 08       Report Size (8)
    95 02       Report Count (2)
    81 06       Input (Data,Variable,Relative)        // X, Y deltas
  C0            End Collection
C0              End Collection
```

Result: a 3-byte input report - byte 0 is buttons + padding, byte 1 = X delta,
byte 2 = Y delta.

### Parsing strategy in a filter

Three valid approaches, in order of robustness:

1. **`HidP_*` against preparsed data** - call once at attach, cache offsets for
   the usages you care about, rewrite via `HidP_SetUsageValue` /
   `HidP_SetButtons`.
2. **Manual descriptor walk** - parse with your own state machine; needed if the
   minidriver gives you a non-standard descriptor (rare).
3. **Hard-coded offsets per VID/PID** - fastest but brittle; only acceptable when
   you control the firmware.

```c
// Locate X axis caps in input report
USHORT count = 8;
HIDP_VALUE_CAPS valueCaps[8];
HidP_GetSpecificValueCaps(HidP_Input, 0x01, 0, 0x30,
                          valueCaps, &count, preparsedData);
// Use HidP_SetUsageValue with the discovered caps to rewrite the field
```

In practice, prefer the high-level `HidP_GetUsageValue` / `HidP_SetUsageValue`
functions - they take the report buffer and report ID, walk the preparsed data,
and read or write the field by usage rather than by offset.

### Padding and report ID byte

When `HidP_GetCaps().NumberInputReportIDs > 1`, every report has a report-ID byte
0; otherwise reports start directly with data. Always branch on that, not on a
header pattern. Many bugs come from filters assuming byte 0 is a report ID on
single-collection mice (it isn't).

### Multi-collection compound devices

Gaming peripherals routinely emit 3-5 TLCs (mouse + macro keyboard + consumer
controls + vendor + DFU). Each TLC has its own report set with independent IDs.
HIDClass enumerates them as separate PDOs; your filter must attach to each PDO
that carries data of interest.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Computing a report-byte offset from raw item types | Local-item resets and global persistence are easy to get wrong | Use `HidP_GetValueCaps`/`HidP_GetButtonCaps` and let the API track state |
| Writing fields with `RtlCopyMemory` then forwarding | Bit-packed fields cross byte boundaries | Use `HidP_SetUsageValue` which handles bit packing |
| Same code for variable and array Input items | Array items use index to usage mapping (e.g. keyboard scancodes) | Branch on `IsRange` / `IsButton` in HIDP_BUTTON_CAPS |
| Caching preparsed data across firmware updates | The descriptor can change between versions | Re-parse on `EvtDevicePrepareHardware` |

## See also

- https://www.usb.org/document-library/device-class-definition-hid-111
- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/hidpi/nf-hidpi-hidp_setusagevalue
- https://learn.microsoft.com/en-us/windows-hardware/drivers/hid/parsing-a-device-s-report-descriptor
