# Windows Drivers - UMDF v2 - Overview

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/getting-started-with-umdf-version-2
> **Skill**: dev-suite skill `windows/wdf-umdf` — see SKILL.md for the always-loaded quick reference.

## What this covers

The architecture of UMDF v2 — host process, reflector, dispatcher — and the decision tree for choosing UMDF v2 over KMDF (or vice versa). Includes the v1-to-v2 migration story and the IDD/HID/Sensor/MFT scenarios that are UMDF-only. Read when standing up a new device class or when deciding whether your driver belongs in user mode at all.

## Deep dive

### What "UMDF v2" actually is

A UMDF v2 driver is a regular Win32 DLL exporting `DriverEntry`. It's loaded by the WUDF host service (`WUDFHost.exe`), one host process per device by default, and the host calls into your DLL using the same `Wdf*` API surface KMDF drivers use. The framework runtime is `WUDFx02000.dll` (mirroring KMDF's `Wdf01000.sys`).

The path between the kernel I/O Manager and your DLL is:

```
App  ─▶ I/O Manager ─▶ WUDFRd.sys (Reflector, kernel)
                              │
                              ▼  (ALPC + shared section)
                        WUDFHost.exe (host process, user mode)
                              │
                              ▼
                        WUDFx02000.dll  (UMDF runtime)
                              │
                              ▼
                        YourDriver.dll  (you)
```

Each IRP becomes a `WDFREQUEST` delivered to your callback. The reflector enforces process-isolation and tears down the host on driver crash without a bugcheck.

### Why UMDF v1 is dead

UMDF v1 was COM-based: drivers implemented `IDriverEntry`, `IPnpCallback`, `IQueueCallbackDeviceIoControl`, etc. as COM objects with reference counting. v2 dropped COM in favor of the WDF object model used by KMDF — same handles, same callbacks. Microsoft has not added features to v1 since Windows 10. If you encounter a v1 driver, port it.

### When UMDF is required

| Driver class | Why UMDF |
|--------------|----------|
| Indirect Display Driver (IDD) | Inbox `IddCx` only loads in WUDFHost |
| Sensor Class Extension (`SensorsCx`) | Inbox class extension is UMDF-only |
| MFT (Media Foundation Transform) for capture | Runs as part of an MF pipeline, fundamentally user mode |
| Many printer / scanner v4 drivers | WUDF is the supported path |
| Bluetooth profile drivers (`BthHfp`, etc.) | Profile drivers run on UMDF over the inbox kernel stack |

### When UMDF is recommended (but optional)

- Anything talking to USB through WinUSB
- Anything backed by a Win32 service or library you would rather not port to kernel
- HID-over-Bluetooth peripherals where the transport is already user-reachable
- "Software-only" drivers that just need an IOCTL surface

If your code links against `<filesystem>`, `<thread>`, `<chrono>`, OpenSSL, libcurl, or anything you'd find on vcpkg, UMDF is your answer.

### When KMDF is required

- You're in the I/O path of a kernel-only stack (file system filter, NDIS LWF, StorPort miniport).
- You need to be at DISPATCH_LEVEL (e.g. servicing a real hardware ISR you own).
- You need direct MMIO/port I/O to memory-mapped registers without going through a companion driver.

### Companion driver pattern

When UMDF can't reach kernel-only resources, ship a thin KMDF companion that exposes a private inverted-call IOCTL channel. Your UMDF driver opens the kernel device, sends commands, and consumes async notifications:

```cpp
// UMDF: open kernel companion
HANDLE h = CreateFileW(L"\\\\.\\MyKernelHelper", GENERIC_READ | GENERIC_WRITE,
                      FILE_SHARE_READ | FILE_SHARE_WRITE, nullptr,
                      OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, nullptr);
DeviceIoControl(h, IOCTL_HELPER_DO_THING, &in, sizeof(in), &out, sizeof(out), &bytes, nullptr);
```

Keep the kernel surface as small as possible — every IOCTL is an attack surface. The UMDF driver applies policy; the kernel companion just executes the unavoidable kernel-only step.

### Crash semantics

If your UMDF DLL crashes (access violation, unhandled C++ exception), the reflector tears down WUDFHost.exe. PnP marks the device as "not functioning" briefly, then attempts a restart. The user sees a transient device disappearance, not a bugcheck. This is the killer reliability advantage over KMDF.

You can configure the reflector's restart policy via `HKLM\System\CurrentControlSet\Services\WUDFHost\Parameters`:

- `HostProcessDbgBreakOnError` — break into debugger instead of restart
- `HostProcessRestartIntervalSec` — minimum interval between restarts
- `HostProcessMaxRestartCount` — give up after N restarts

### Tooling story

UMDF gets the full user-mode tooling stack: Visual Studio diagnostics tools, ETW with TraceLogging, Application Verifier, AddressSanitizer (limited). Driver Verifier (kernel) does not apply; the closest analog is Application Verifier with the right plugins enabled.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Trying to call `KeXxx` kernel APIs | Won't link | Use Win32 / WDF user-mode equivalents |
| Sharing process state across device instances | One host can serve multiple devices | Per-device context only |
| Treating WUDFHost as long-lived for caching | Reflector may recycle it | Reload from registry/file in `EvtDevicePrepareHardware` |
| Long blocking work in callbacks | Serializes other requests | Spawn worker thread, complete request when done |
| Choosing UMDF for a real ISR-driven device | UMDF can't service hardware interrupts | Either KMDF or UMDF + KMDF companion |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/overview-of-the-umdf
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/choosing-which-driver-model-to-use
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/architecture-of-umdf-version-2
