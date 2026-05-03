# Windows Drivers - UMDF v2 - Reflector & Host Architecture

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/architecture-of-umdf-version-2
> **Skill**: dev-suite skill `windows/wdf-umdf` — see SKILL.md for the always-loaded quick reference.

## What this covers

What `WUDFRd.sys` does in the kernel, how it marshals IRPs to your DLL inside `WUDFHost.exe`, the host-process restart policy, and the security boundary the reflector enforces. Read when designing communication between user-mode and kernel-mode parts of a driver, when debugging IRP timeouts, or when the host keeps recycling.

## Deep dive

### The reflector's role

`WUDFRd.sys` is a kernel WDM driver that PnP loads on top of (or in place of) the function driver's stack. From the I/O Manager's perspective it IS the kernel function driver: it owns the device stack, exposes the device interface, and receives every IRP destined for the device.

When an IRP arrives, the reflector:

1. Validates the IRP against the host's process state (host alive? draining for shutdown?).
2. Captures user-mode buffers using the appropriate I/O method (Buffered, Direct, or Neither — based on `UmdfMethodNeitherAction`).
3. Posts an ALPC message to the host process containing a marshaled descriptor of the request.
4. Waits for completion (or times out per the host's restart policy).

In the host process, the WUDF runtime (`WUDFx02000.dll`) receives the ALPC message, reconstructs a `WDFREQUEST` object, dispatches it to your callback (`EvtIoDeviceControl` etc.), then sends the completion back to the reflector.

### Host process model

By default each device gets its own `WUDFHost.exe` process. With `UmdfHostProcessSharing = ProcessSharingEnabled` in the INF, multiple devices of the same driver collapse into one host. Defaults to disabled because:

- Crash isolation: one driver bug only kills its own device.
- Memory accounting: each host's working set is attributable to its device in Task Manager.
- Bug-class containment: heap corruption can't traverse devices.

You see the host with `tasklist /svc | findstr WUDFHost` or in Process Explorer where `WUDFHost.exe` runs as the `LocalService` account by default. Your driver runs with a much smaller token than `System` — `CreateFile` against a secured kernel object may fail with `STATUS_ACCESS_DENIED` if you didn't request impersonation.

### IRP marshalling — what gets copied

| I/O Method | What the reflector does |
|-----------|-------------------------|
| `METHOD_BUFFERED` | Allocates a kernel buffer, copies user data in, copies result back on completion. Safe and slow for large buffers. |
| `METHOD_IN_DIRECT` / `METHOD_OUT_DIRECT` | Locks user pages with an MDL, maps into the host process's address space. Fast for large buffers. |
| `METHOD_NEITHER` | Behavior controlled by `UmdfMethodNeitherAction`: `Copy` (treat as buffered) or `Pass` (give the host the raw user pointer; the user-mode driver must validate). Use `Copy` unless you have a strong reason. |

### Restart policy

When the host crashes (unhandled exception, AV, fast-fail), the reflector:

1. Marks the device as "not started".
2. Waits the configured `HostProcessRestartIntervalSec` (default 10s).
3. Re-launches `WUDFHost.exe`, re-loads the driver DLL.
4. Re-issues `WdfDriverCreate` -> `EvtDriverDeviceAdd` -> `EvtDevicePrepareHardware` -> `EvtDeviceD0Entry`.
5. Pending requests in the queues at crash time are completed with `STATUS_DEVICE_REMOVED`.

Configuration registry keys under `HKLM\System\CurrentControlSet\Services\WUDFHost\Parameters`:

| Value | Meaning |
|-------|---------|
| `HostProcessDbgBreakOnStart` (DWORD) | If 1, break before driver load. Combine with `windbg -p` |
| `HostProcessDbgBreakOnError` (DWORD) | If 1, break on unhandled host failure instead of restart |
| `HostProcessRestartIntervalSec` (DWORD) | Delay between restarts |
| `HostProcessMaxRestartCount` (DWORD) | After N restarts the device is permanently failed |

### Cross-process security

The reflector enforces:

- The user-mode driver cannot directly call into other host processes.
- File handles created in the host are not visible to other hosts unless explicitly shared.
- The host runs with a **lowered integrity** token by default; many `Zw*` operations would fail anyway because they require `SYSTEM`.

If your driver needs higher privileges (e.g. write to a protected registry key), do it in a kernel companion driver. Don't try to elevate the host.

### Communication patterns

| Pattern | Mechanism |
|---------|-----------|
| App ⇄ UMDF driver | Standard `CreateFile` + `DeviceIoControl`. Reflector marshals. |
| UMDF driver ⇄ kernel companion | UMDF opens the companion's device interface; uses `DeviceIoControl` with inverted-call IOCTLs |
| UMDF driver ⇄ Win32 service | Named pipe, ALPC, COM — anything Win32 supports |
| Notification kernel→UMDF | Companion exposes IOCTL_PEND_NOTIFICATION; UMDF posts overlapped IOCTLs and waits |

### Diagnostics

`!wudfkd.dumphost <DeviceInstanceID>` in WinDbg connected to the kernel attaches you to the corresponding host process state. ETW provider `Microsoft-Windows-DriverFrameworks-UserMode` logs lifecycle events; capture with `wpr -start GeneralProfile -filemode` or `xperf`.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Assuming WUDFHost runs as SYSTEM | It runs as LocalService by default | Use service-account-appropriate APIs; impersonate the caller for ACL checks |
| Sharing handles via global state across hosts | Each host is isolated | Use Win32 named-object marshalling explicitly |
| Long-held requests blocking restart | Reflector waits for in-flight before drain | Implement cancel handlers and complete promptly on shutdown |
| Counting on host PID stability | PID changes on restart | Use device interface GUID + symbolic link instead |
| Setting `UmdfMethodNeitherAction = Pass` and dereferencing the pointer | Invalid in host's address space | `Copy` (default) unless you've explicitly probed the buffer |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/services-that-the-reflector-provides
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/processes-and-threads-of-umdf-drivers
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/sharing-power-information-with-the-host-process
