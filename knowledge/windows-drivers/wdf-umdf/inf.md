# Windows Drivers - UMDF v2 - INF Directives

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/specifying-wdf-directives-in-inf-files
> **Skill**: dev-suite skill `windows/wdf-umdf` — see SKILL.md for the always-loaded quick reference.

## What this covers

The `[<install>.NT.Wdf]` section directives that distinguish a UMDF v2 INF from a KMDF or legacy INF: `UmdfDispatcher`, `UmdfServiceOrder`, `UmdfFileObjectPolicy`, `UmdfMethodNeitherAction`, kernel-mode client policies, and the WUDFRd reflector service install. Read when writing or debugging an INF that PnP refuses to install.

## Deep dive

### The minimal UMDF v2 INF skeleton

```inf
[Version]
Signature   = "$WINDOWS NT$"
Class       = Sample
ClassGuid   = {78A1C341-4539-11d3-B88D-00C04FAD5171}
Provider    = %ProviderName%
DriverVer   = 06/21/2006,1.0.0.0
CatalogFile = MyDevice.cat
PnpLockdown = 1                                ; PnP-locked since Win10

[Manufacturer]
%MfgName% = Standard,NT$ARCH$.10.0...19041

[Standard.NT$ARCH$.10.0...19041]
%Device.DeviceDesc% = MyDevice_Install, ROOT\MyDevice

;-- Install ---------------------------------------------------------
[MyDevice_Install.NT]
CopyFiles = MyDevice.UmdfCopy

[MyDevice_Install.NT.Services]
AddService = WUDFRd, 0x000001fa, WUDFRD_ServiceInstall

[MyDevice_Install.NT.Wdf]
UmdfDispatcher                = NativeUSB           ; or NativeIDD, NativeHID, NativeNetAdapter
UmdfServiceOrder              = MyUmdfDriver
UmdfKernelModeClientPolicy    = AllowKernelModeClients
UmdfFileObjectPolicy          = AllowNullAndUnknownFileObjects
UmdfMethodNeitherAction       = Copy
UmdfFsContextUsePolicy        = CannotUseFsContexts
UmdfHostProcessSharing        = ProcessSharingDisabled
UmdfDirectHardwareAccess      = DenyDirectHardwareAccess

[MyUmdfDriver]
UmdfLibraryVersion = $UMDFVERSION$
ServiceBinary      = %13%\MyDevice.dll
DriverCLSID        = {01234567-89AB-CDEF-0123-456789ABCDEF}

[WUDFRD_ServiceInstall]
DisplayName    = "WUDF Reflector"
ServiceType    = 1
StartType      = 3
ErrorControl   = 1
ServiceBinary  = %12%\WUDFRd.sys
LoadOrderGroup = Base
```

`%12%` is `\Windows\System32\drivers`; `%13%` is `\Windows\System32\drivers\UMDF` (or DriverStore equivalent under PnP-lockdown).

### What each Wdf directive does

| Directive | Effect |
|-----------|--------|
| `UmdfDispatcher` | Tells WUDFRd which dispatcher to use. `NativeUSB` for WinUSB transport, `NativeIDD` for indirect displays, `NativeHID` for HID minidrivers, `NativeNetAdapter` for NetAdapter UMDF, blank for generic. |
| `UmdfServiceOrder` | List of UMDF service names in load order (most drivers ship one). |
| `UmdfKernelModeClientPolicy` | `AllowKernelModeClients` or `RejectKernelModeClients`. Controls whether a kernel driver can `IoGetDeviceObjectPointer` your device. Reject by default for security; allow if you have a kernel companion that talks to you. |
| `UmdfFileObjectPolicy` | `AllowNullAndUnknownFileObjects` lets requests without an associated `WDFFILEOBJECT` reach your queue. Required if a kernel driver creates I/O without a Win32 handle. |
| `UmdfMethodNeitherAction` | `Copy` (default) makes the framework copy `METHOD_NEITHER` buffers safely; `Pass` gives you the raw user pointer (rarely correct). |
| `UmdfFsContextUsePolicy` | `CanUseFsContexts` lets you stash your own pointer in `FILE_OBJECT->FsContext`; usually `CannotUseFsContexts`. |
| `UmdfHostProcessSharing` | `ProcessSharingDisabled` (each device gets its own host) vs `ProcessSharingEnabled` (multiple devices in one host). Default is disabled — better isolation. |
| `UmdfDirectHardwareAccess` | `DenyDirectHardwareAccess` (default) vs `RejectInterruptAccess` etc. Mostly historical for v1. |
| `UmdfRegisterAccessMode` / `UmdfPoolSizeOverride` | Tuning knobs almost no one needs. |

### `UmdfDispatcher` choices that matter

- **Blank** — your driver gets the raw IRP-equivalent. Used for generic software-only drivers.
- **`NativeUSB`** — the framework wires in WinUSB on the kernel side; your `WdfUsbTargetDeviceCreateWithParameters` call works. This is what you want for any USB device.
- **`NativeIDD`** — Indirect Display Class Extension; your driver implements the `IddCx*` callbacks.
- **`NativeHID`** — HID class extension `HidCx*`; for HID transport mini-drivers.
- **`NativeNetAdapter`** — NetAdapter Class Extension; for user-mode NIC drivers.

Mismatch between dispatcher and your code's expectations causes `WdfDeviceCreate` to fail with `STATUS_INVALID_DEVICE_REQUEST`.

### CatalogFile + signing

`CatalogFile = MyDevice.cat` is required — generate the cat with `inf2cat /driver:DriverPackage /os:10_X64,10_X86,10_ARM64`. Sign the cat with a real or test certificate via `signtool sign /v /fd SHA256 /a /f Test.pfx /t http://timestamp.digicert.com MyDevice.cat`. Without test signing on, Windows will refuse to load.

### Validation

Run `infverif /v MyDevice.inf` (ships with the WDK) to validate. Common failures:

- `1284` — section names mismatch architecture decoration
- `1296` — `Wdf` section missing required directive
- `2083` — `PnpLockdown=1` missing on a PnP INF
- `1009` — class GUID not registered

For a deep lint use `infverif /h` (hardware) — required for WHQL.

### Multiple INFs sharing one driver

Use `UmdfServiceOrder` to load multiple UMDF DLLs into the same host. Define a separate `[ServiceXxx]` section per service:

```inf
[MyDevice_Install.NT.Wdf]
UmdfServiceOrder = MyTransport, MyProfile

[MyTransport]
UmdfLibraryVersion = $UMDFVERSION$
ServiceBinary      = %13%\transport.dll

[MyProfile]
UmdfLibraryVersion = $UMDFVERSION$
ServiceBinary      = %13%\profile.dll
```

Order is bottom-up at load: `MyProfile` initializes after `MyTransport`.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Missing `[install].NT.Wdf` section | INF treated as legacy/KMDF, UMDF not loaded | Always include the section, even if empty (with at least `UmdfLibraryVersion`) |
| `ServiceBinary = %12%\MyDevice.dll` | Wrong directory for UMDF DLLs | `%13%` for UMDF |
| Forgetting `PnpLockdown = 1` | INF rejected on Win10+ | Set in `[Version]` |
| `UmdfDispatcher` mismatched with code | `WdfDeviceCreate` fails | Pick the dispatcher matching your transport |
| Unsigned INF or `.cat` | "Driver not digitally signed" prompt or hard fail | Sign with `signtool` after `inf2cat` |
| Re-using same `DriverCLSID` across drivers | COM clash, install ambiguity | Generate fresh GUID per driver via `uuidgen` |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/install/inf-version-section
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/installing-a-umdf-version-2-driver
- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/infverif
