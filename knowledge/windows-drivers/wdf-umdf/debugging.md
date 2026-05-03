# Windows Drivers - UMDF v2 - Debugging

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/how-to-enable-debugging-of-a-umdf-driver
> **Skill**: dev-suite skill `windows/wdf-umdf` — see SKILL.md for the always-loaded quick reference.

## What this covers

How to attach to `WUDFHost.exe` for live debugging, configure host-process break-on-start, capture full-process dumps on driver crash, run Application Verifier with the right plugin set, and inspect the WDF object tree from the user-mode debugger. Read when WUDFHost keeps crashing, when a driver freezes mid-IO, or when chasing memory bugs.

## Deep dive

### Live attach

`WUDFHost.exe` is a normal Win32 process — attach with WinDbg or Visual Studio:

```powershell
# Find the host PID for your device
tasklist /svc | findstr WUDFHost
# Or via PowerShell with device interface
Get-PnpDeviceProperty -InstanceId 'ROOT\MyDevice\0000' -KeyName DEVPKEY_Device_DriverProcessId
```

Then in WinDbg: `File > Attach to a Process` and pick the right WUDFHost PID. You get full source-level debugging, locals, breakpoints — everything missing from KMDF live debugging unless you have a second machine for kernel debug.

### Breaking before driver load

Set:

```
HKLM\System\CurrentControlSet\Services\WUDFHost\Parameters
  HostProcessDbgBreakOnStart = 1   (DWORD)
```

When PnP starts your device, the host launches and immediately calls `DbgBreakPoint`. WinDbg catches it (configure `Image File Execution Options\WUDFHost.exe\Debugger = "C:\path\to\windbgx.exe"` to auto-attach).

For per-driver isolation use:

```
HKLM\System\CurrentControlSet\Services\WUDFHost\Parameters\<MyDriverDLL>
  HostProcessDbgBreakOnDriverLoad = 1
```

### Crash dumps

Force a full dump on host crash via Windows Error Reporting:

```
HKLM\Software\Microsoft\Windows\Windows Error Reporting\LocalDumps\WUDFHost.exe
  DumpType = 2          (2 = full)
  DumpCount = 10
  DumpFolder = C:\Crashes
```

After a crash, `C:\Crashes\WUDFHost.exe.<pid>.dmp` contains the full host snapshot. Open in WinDbg:

```
0:000> !analyze -v
0:000> .ecxr
0:000> kP
0:000> dt MyDriver!_MY_DEVICE_CONTEXT @rcx
```

### Application Verifier (the UMDF Driver Verifier)

Application Verifier is the user-mode equivalent of Driver Verifier. Enable for `WUDFHost.exe`:

```cmd
appverif -enable Heaps Handles Locks Threadpool TLS Exceptions DangerousAPIs -for WUDFHost.exe
```

| Plugin | What it catches |
|--------|------------------|
| `Heaps` | Use-after-free, double-free, heap overflow at the page level |
| `Handles` | Closing already-closed handles, mismatched APIs |
| `Locks` | Recursive acquire of non-recursive locks, leaked critical sections |
| `Threadpool` | Misuse of `TpAlloc*`/`TpRelease*` |
| `TLS` | Leaks of TLS slots |
| `Exceptions` | Catches first-chance access violations, fast-fail |
| `DangerousAPIs` | Flags `Sleep(0)` busy waits, `TerminateThread`, etc. |

Then reproduce the issue. Verifier breaks into the debugger at the offending call with a clear message.

### WDF debugger extensions

The `wdfkd.dll` (kernel) and `wudfkd.dll` (user-mode) extensions ship with WinDbg:

```
0:000> .load wudfkd
0:000> !wudfkd.help
0:000> !wdfdriverinfo MyDriver 0x3
0:000> !wdfdevice 0x000001A2BC123450
0:000> !wdfqueue 0x000001A2BC234560
0:000> !wdfrequest 0x000001A2BC345670
0:000> !wdflogdump MyDriver           ; pulls the In-Flight Recorder
```

`!wdflogdump` is invaluable in a crash dump — it shows the last several hundred WPP messages that were buffered in the driver, even if no live ETW session was running.

### ETW for live timing

ETW's `Microsoft-Windows-DriverFrameworks-UserMode` provider logs every WDF object lifecycle event. Capture with WPR:

```cmd
wpr -start GeneralProfile -start "c:\path\Custom-WDF-Trace.wprp"
:: Reproduce the bug
wpr -stop C:\trace.etl
:: Open in WPA (Windows Performance Analyzer)
```

Ship a `.wprp` profile listing your driver's WPP GUID + UMDF framework GUIDs and Verbose level. Engineers running into your driver in the field can capture with one command.

### TraceLogging vs WPP

TraceLogging is the modern C++ idiom for ETW; works in UMDF (and any user-mode code):

```cpp
#include <TraceLoggingProvider.h>
#include <TraceLoggingActivity.h>

TRACELOGGING_DECLARE_PROVIDER(g_provider);
TRACELOGGING_DEFINE_PROVIDER(g_provider, "MyDriver",
    (0x12345678, 0x1234, 0x1234, 0x12, 0x34, 0x12, 0x34, 0x56, 0x78, 0x9a, 0xbc));

TraceLoggingRegister(g_provider);
TraceLoggingWrite(g_provider, "RequestHandled",
    TraceLoggingValue(ioctl, "Code"),
    TraceLoggingValue(status, "Status"));
TraceLoggingUnregister(g_provider);
```

No `.tmh` step, no preprocessor. Self-describing event format. Decode with `tracerpt -of EVTX` or `wpa.exe`.

### Symbol setup

Set `_NT_SYMBOL_PATH=srv*c:\symbols*https://msdl.microsoft.com/download/symbols;c:\my-pdbs` so the debugger resolves both Microsoft and your driver's `.pdb`. Source-indexed PDBs let WinDbg pull the exact source revision from your VCS — wire `srctool.exe` into your build.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Attaching to wrong WUDFHost PID | Multiple hosts running | `tasklist /svc | findstr WUDFHost` and match service name |
| Missing PDB so `!analyze -v` shows offsets only | Debugger can't resolve | Configure `_NT_SYMBOL_PATH`; ship matched PDB |
| Application Verifier left enabled in production | Performance regression and fast-fails | `appverif -disable * -for WUDFHost.exe` after debugging |
| Forgetting `WPP_INIT_TRACING` so IFR is empty | Nothing to dump on crash | Init WPP early in `DriverEntry` |
| Treating first-chance exceptions as bugs | C++ uses exceptions for control flow sometimes | Enable break only for second-chance unless investigating something specific |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/debugging-using-windbg
- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/application-verifier
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/getting-started-with-the-kernel-mode-driver-framework
