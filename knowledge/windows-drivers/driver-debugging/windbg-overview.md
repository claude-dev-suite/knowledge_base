# Windows Drivers - Driver Debugging - WinDbg Overview

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/
> **Skill**: dev-suite skill `windows/driver-debugging` — see SKILL.md for the always-loaded quick reference.

## What this covers

The differences between the WinDbg variants shipped today, the kernel-mode vs user-mode debugging models that sit underneath them, and the structure of the command language. Fetch this when you need to pick the right tool, or when an unfamiliar command syntax appears.

## Deep dive

### The three "WinDbg" binaries

| Binary | Source | UI | When to use |
|--------|--------|----|-------------|
| `windbg.exe` (classic) | Debugging Tools for Windows (in WDK / SDK) | MDI, win32 | Headless lab boxes, scripts that screen-scrape, when WinDbgX won't install |
| `WinDbg` (Preview / X) | Microsoft Store / WinGet (`Microsoft.WinDbg`) | Ribbon, dock panes, time travel UI | Default daily driver — supports TTD, settings sync, native dark mode |
| `kd.exe` | Same WDK install as classic | Pure command-line | CI, automation, remote SSH, dump triage scripts |

All three drive the same `dbgeng.dll` engine and accept the same commands. WinDbg-Preview adds Time Travel Debugging (TTD) recording for user-mode processes — kernel-mode TTD is not supported.

### Kernel-mode vs user-mode session model

- **User-mode**: attach to / launch a single process. Symbols, breakpoints, memory all scoped to that process.
  ```
  windbg -o notepad.exe              ; launch and attach (debug child too)
  windbg -p 4242                     ; attach to PID
  windbg -z C:\dump\app.dmp          ; open a user-mode minidump
  ```
- **Kernel-mode**: attach to the kernel of a target machine (or a kernel dump). You see *all* processes, the kernel, drivers, and the HAL.
  ```
  kd -k net:port=50000,key=1.2.3.4   ; live KDNET
  kd -z C:\Windows\MEMORY.DMP        ; post-mortem
  ```

You can switch process context inside a kernel session with `.process /i <eprocess>` then `.reload /user`.

### Command categories

| Prefix | Meaning | Example |
|--------|---------|---------|
| (none) | Built-in eval / step / breakpoint | `g`, `bp`, `kn` |
| `.` | Meta / debugger-state | `.reload /f`, `.sympath+ c:\sym`, `.dump /ma` |
| `!` | Extension command (DLL-provided) | `!analyze -v`, `!process 0 7`, `!wdfkd.wdfdriverinfo` |
| `dt` | Display typed structure | `dt nt!_EPROCESS @$proc` |
| `?` / `??` | MASM / C++ expression eval | `? @rcx + 0x18`, `?? sizeof(KDPC)` |

`?` is MASM (assembler) eval — addresses, registers, symbols. `??` is C++ — pointer arithmetic, member access, types. They use different operator precedence; pick `??` when you would otherwise reach for casts.

### Loading extensions

Extensions live in `winext\` next to the engine. Load with `.load`, list with `.chain`:

```
.load wdfkd.dll
.load ndiskd.dll
.chain
```

Most ship in the WDK; vendor extensions (e.g. `gflags`-style) come from third-party packages and must be on the extension search path (`.extpath+ c:\myext`).

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Mixing 32-bit and 64-bit debugger / target | `dbgeng` cannot translate pointers across bitness | Use the `x64` debugger for x64 targets, ARM64 debugger for ARM64 targets |
| Running classic WinDbg against an ARM64 dump | No SIMD register decoders, garbled call stacks | Use WinDbg-Preview (auto-selects engine) |
| Calling `!analyze` on a user dump in a kernel session (or vice-versa) | Wrong heuristics, confusing output | Open the dump in the matching mode — kernel for `.dmp`, user for `.mdmp` |
| Trusting cached symbols across builds | Wrong source line offsets | `.reload /f` after every rebuild; pin build-id with `srv*` |
| Using `windbg.exe -k com:...` over a flaky USB-serial | Drops connection, corrupted breakpoints | Use KDNET — Ethernet is far more robust |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/windbg-overview
- https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-commands
- https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-overview
