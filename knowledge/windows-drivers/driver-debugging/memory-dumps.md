# Windows Drivers - Driver Debugging - Memory Dump Varieties

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/varieties-of-kernel-mode-dump-files
> **Skill**: dev-suite skill `windows/driver-debugging` — see SKILL.md for the always-loaded quick reference.

## What this covers

The five kernel-mode dump variants Windows can produce, the difference in what each contains, the registry / GUI ways to configure them on the target, the size cost, and the analysis flow that matches each. Fetch when "set Kernel dump" advice from SKILL.md is not detailed enough — usually because you need either a tiny minidump (CI artefact) or a complete dump (hardest bugs).

## Deep dive

### Dump variants

| Variant | Size (typical) | What's in it | When to use |
|---------|----------------|--------------|-------------|
| **Small memory dump (Minidump)** | 256 KB | Stop code, params, faulting thread context, loaded module list | CI/triage, mass-collected from users |
| **Kernel memory dump** | RAM in use by kernel + drivers (often 100 MB – 1 GB) | All kernel-mode memory, no user-mode pages | Default for driver bugs |
| **Complete memory dump** | Full RAM (32+ GB on modern boxes) | Every byte of physical memory | Hard bugs needing user-mode pages or arbitrary physical inspection |
| **Active memory dump** | RAM minus pages clearly unrelated (e.g. Hyper-V guest backing) | Practical complete dump on hyperscale hosts | Hosts with massive RAM where Complete is impractical |
| **Automatic memory dump** | Same content as Kernel | Kernel dump but page file is auto-sized to fit | Default in modern Windows; hands-off |

### Configuring on the target

GUI: System → Advanced system settings → Startup and Recovery → Settings → "Write debugging information".

Registry (script-friendly):

```cmd
:: 0=None, 1=Complete, 2=Kernel, 3=Small, 7=Automatic
reg add "HKLM\SYSTEM\CurrentControlSet\Control\CrashControl" /v CrashDumpEnabled /t REG_DWORD /d 1 /f

:: Path (full + small)
reg add "HKLM\SYSTEM\CurrentControlSet\Control\CrashControl" /v DumpFile /t REG_EXPAND_SZ /d "%SystemRoot%\MEMORY.DMP" /f
reg add "HKLM\SYSTEM\CurrentControlSet\Control\CrashControl" /v MinidumpDir /t REG_EXPAND_SZ /d "%SystemRoot%\Minidump" /f

:: Always overwrite
reg add "HKLM\SYSTEM\CurrentControlSet\Control\CrashControl" /v Overwrite /t REG_DWORD /d 1 /f

:: Don't auto-reboot — keep the BSOD on screen
reg add "HKLM\SYSTEM\CurrentControlSet\Control\CrashControl" /v AutoReboot /t REG_DWORD /d 0 /f

:: Active dump (Win10 1607+): set FilterPages
reg add "HKLM\SYSTEM\CurrentControlSet\Control\CrashControl" /v FilterPages /t REG_DWORD /d 1 /f
```

For Complete dumps, the page file on `%SystemDrive%` must be **at least RAM + 1 MB**. If not, Windows silently falls back to Kernel.

### Triggering dumps on demand

```cmd
:: Live system: use NotMyFault from Sysinternals to force a known bugcheck
notmyfault64.exe /crash

:: Or from WinDbg attached:
.crash                        ; immediate bugcheck
.dump /ma C:\full.dmp         ; user-mode full dump (target must be a process)
.dump /f C:\kernel.dmp        ; kernel dump (in a kernel session)
```

For VMs, hypervisor-managed crash dumps are also an option — `Stop-VM -TurnOff` followed by `Save-VMState` can yield a full memory snapshot you can later open with `livekd -hvl`.

### Hardware NMI as a last-resort trigger

Enable on the target:

```cmd
reg add "HKLM\SYSTEM\CurrentControlSet\Control\CrashControl" /v NMICrashDump /t REG_DWORD /d 1 /f
```

Then press the NMI button (server BMC) or use `ipmitool chassis power diag`. The system bugchecks `0x80` and writes the configured dump variant — invaluable for hangs that cannot be broken into otherwise.

### Analysis flow

```
:: Open any kernel dump
windbg -z C:\Windows\MEMORY.DMP

.symfix C:\sym
.sympath+ C:\build\out\symbols
.reload /f
!analyze -v
```

For a **Minidump** the analysis is constrained — you have the faulting thread's stack and registers, but you cannot dereference arbitrary kernel pointers. Most of the standard `!process`, `!irp`, `!pool` commands will fail with "memory access error".

For a **Kernel dump** the inspection is full for kernel pointers; you cannot inspect user-mode process memory.

For a **Complete dump** anything in physical RAM is inspectable. `!process 0 17 myapp.exe` will dump the full process tree including handles.

### Size budgeting

| RAM size | Kernel dump (typical) | Complete dump | Page file required for Complete |
|----------|------------------------|----------------|----------------------------------|
| 8 GB | 200–500 MB | 8 GB | 8 GB + 1 MB |
| 16 GB | 400 MB – 1 GB | 16 GB | 16 GB + 1 MB |
| 64 GB | 1–3 GB | 64 GB | 64 GB + 1 MB |
| 256 GB | 2–6 GB | 256 GB | 256 GB + 1 MB — usually impractical, use Active |

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Configured Complete dump but page file too small | Silent fallback to Kernel | Verify after a test crash that file size matches RAM |
| `Overwrite=0` and `MEMORY.DMP` already exists | Subsequent crashes write nothing | Set `Overwrite=1` on lab boxes |
| Reading a Kernel dump with user-mode `!analyze` heuristics | Wrong tool, confusing output | Open in kernel mode |
| Minidump only contains the faulting thread | If your bug is on another thread, you're stuck | Configure Kernel/Complete on at least one repro machine |
| `!analyze -v` fails with "PROCESS_NAME: <unknown>" | Kernel page tables not in dump (pure minidump) | Capture a Kernel dump |
| Auto-reboot enabled in CI VM | Loops continuously without producing useful state | Disable AutoReboot, capture, then reboot manually |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/configure-system-failure-and-recovery-options
- https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/active-memory-dump
- https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/forcing-a-system-crash-from-the-keyboard
