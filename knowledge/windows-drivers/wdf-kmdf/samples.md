# Windows Drivers - KMDF - Sample Tour

> **Source**: https://github.com/microsoft/Windows-driver-samples/tree/main/general/echo/kmdf
> **Skill**: dev-suite skill `windows/wdf-kmdf` — see SKILL.md for the always-loaded quick reference.

## What this covers

A guided tour of the most useful samples in `microsoft/Windows-driver-samples`, what each one teaches, and which file to read first. Read when starting a new driver and looking for a known-good template, or when researching a specific scenario (HID, USB, ACPI, virtual battery, multi-queue forwarding).

## Deep dive

### echo / kmdf — the canonical starter

`general/echo/kmdf/` is the smallest non-trivial KMDF sample.

| File | Purpose |
|------|---------|
| `Driver.c` | `DriverEntry`, `EvtDriverDeviceAdd`, WPP init/cleanup |
| `Device.c` | Device context, `EvtDevicePrepareHardware`, queue creation |
| `Queue.c` | `EvtIoRead`, `EvtIoWrite`, `EvtIoDeviceControl` echoing data through a buffer |
| `Public.h` | IOCTL codes shared with the user-mode test app |
| `echoapp/echoapp.c` | Win32 console test that opens `\\.\Echo` and round-trips data |

What it teaches: IOCTL plumbing, request retrieval, buffer methods, lifecycle of a software-only driver. Read this first if you've never written a WDF driver.

### autosync — automatic synchronization

`general/echo/autosync/` shows the same problem with `WdfSynchronizationScopeQueue` and a sequential dispatcher — useful when learning when to drop automatic sync for performance.

### toaster — the bus driver classic

`general/toaster/` is a multi-driver tutorial:

- `toastmon` — function driver
- `toastpkg` — INF/CAT package
- `bus/` — ROOT enumerator that creates virtual child PDOs
- `func/` — function driver for the virtual children
- `filter/` — upper filter sample

Read `bus/dynambus/` first — it shows `WdfPdoInitAllocate` and dynamic child enumeration with `WdfPdoMarkMissing`. This is the canonical reference for building a virtual bus driver.

### simbatt — virtual battery via ACPI

`acpi/simbatt/` simulates a battery. Notable for:

- Implementing the standard `BATTERY_MINIPORT_INFO` interface
- Calling `BatteryClassInitializeDevice` from `EvtDevicePrepareHardware`
- Returning `BATTERY_STATUS` / `BATTERY_INFORMATION` from `EvtIoDeviceControl`

If your driver exposes a class-driver-defined contract (battery, HID, NDIS), simbatt's pattern of "WDF wraps a callback table from the inbox class driver" is the pattern to copy.

### eventdrv — kernel-to-user notification

`general/eventdrv/` demonstrates the inverted-call pattern: user mode sends an IOCTL that the driver parks on a manual queue; when an internal event fires, the driver retrieves and completes the request. This is the right way to push notifications from kernel to user — never use `KeSetEvent` on a user-handed event handle, which has security and lifetime problems.

Key file: `eventdrv.c::EvtIoDeviceControl` and the per-device `NotificationQueue`. The companion app `EventTest.exe` overlapped-issues an IOCTL and waits on its event handle.

### nondrv — non-PnP control device driver

`general/nondrv/` shows a non-PnP driver (no AddDevice). It calls `WdfControlDeviceInitAllocate` from `DriverEntry` to create a control device directly. Use this template for diagnostic-only kernel services that aren't tied to a hardware enumerator.

### USB — usbsamp / kmdf_fx2 / osrusbfx2

`usb/` directory has progressively complex USB function drivers:

- `usbsamp/` — generic bulk in/out
- `kmdf_fx2/` — Cypress FX2 dev board, including isochronous endpoints
- `osrusbfx2/` — OSR FX2 board with switches/LEDs/IOCTL forwarding

`osrusbfx2` is the right starting point for HID-style device drivers because it shows the full WDFUSBDEVICE / WDFUSBPIPE / WDFREQUEST chain plus selective suspend.

### NDIS / Storage / Filter

For network or storage you'll work in NDIS or StorPort, both of which sit alongside WDF rather than on top of it. Relevant samples:

- `network/ndis/netvmini/` — virtual NIC
- `storage/miniports/storahci/` — modern AHCI miniport
- `filesys/miniFilter/passthrough/` — file system mini-filter (FltMgr, not WDF)

Don't try to fit those into a KMDF model; they have their own object frameworks.

### How to actually use the repo

1. `git clone --depth 1 https://github.com/microsoft/Windows-driver-samples`
2. Open `general/echo/kmdf/kmdfecho.sln` in Visual Studio with the WDK installed.
3. Set `Configuration = Debug, Platform = x64`. Build.
4. The output `.sys` + `.inf` + `.cat` go to `x64/Debug/kmdfecho/`.
5. On the test VM: enable test signing (`bcdedit /set testsigning on`, reboot), then `pnputil /add-driver kmdfecho.inf /install`.
6. Watch traces with `tracelog -start MyTrace -guid file.guid -f trace.etl -flag 0xFF -level 5`, reproduce, `tracelog -stop`, decode with `tracefmt`.

### License and reuse

The samples are MIT-licensed. Copy entire directories into your own repo, but be careful to keep the per-sample `LICENSE` and update copyright headers. Do NOT vendor the entire `Windows-driver-samples` repo into your driver — it pulls thousands of unrelated files and slows scans.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Building samples without WDK installed | `wdfkmdrv.lib not found` | Install WDK matching your VS version (EWDK works too) |
| Editing samples in-place and losing track | Hard to upgrade later | Fork the sample subdir into your own repo |
| Following an old sample (pre-`ExAllocatePool2`) | Deprecated APIs ship in your driver | Cross-check every alloc against current docs |
| Treating `osrusbfx2` as a generic template | It's tightly coupled to the OSR FX2 board | Strip board-specific code first |

## See also

- https://github.com/microsoft/Windows-driver-samples
- https://learn.microsoft.com/en-us/windows-hardware/drivers/develop/sample-kmdf-drivers
- https://learn.microsoft.com/en-us/windows-hardware/drivers/develop/getting-started-with-windows-drivers
