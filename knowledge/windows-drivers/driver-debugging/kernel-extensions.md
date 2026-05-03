# Windows Drivers - Driver Debugging - Specialized Kernel Extensions

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/specialized-extensions
> **Skill**: dev-suite skill `windows/driver-debugging` — see SKILL.md for the always-loaded quick reference.

## What this covers

Catalog of the specialised debugger extensions beyond the core `nt` set: `ndiskd` (NDIS), `fltkd` (Filter Manager / minifilter), `gflags`, `!devnode` and PnP commands, `!storagekd`, `!usbkd`, plus how to find the right extension when SKILL.md only mentions a couple. Use when the symptom points to a specific subsystem (network, filesystem, USB, storage).

## Deep dive

### Loading and discovering

```
.load wdfkd.dll
.load ndiskd.dll
.load fltkd.dll
.chain                              ; show currently loaded extensions
!ndiskd.help                        ; per-extension help (most extensions follow this convention)
!fltkd.help
.extpath                            ; current search path
.extpath+ c:\windbg\winext\custom   ; add a custom path
```

Most extensions ship in `<WinDbg>\winext\` and `<WinDbg>\winxp\` (legacy). Vendor extensions (Intel ME, NVIDIA, etc.) come as separate downloads.

### Subsystem cheat sheet

| Subsystem | Extension | Headline commands |
|-----------|-----------|-------------------|
| WDF (KMDF/UMDF) | `wdfkd.dll` | `!wdfdriverinfo`, `!wdfdevice`, `!wdfqueue`, `!wdfrequest`, `!wdfobject`, `!wdfldr` |
| NDIS networking | `ndiskd.dll` | `!ndiskd.miniports`, `!ndiskd.protocols`, `!ndiskd.netadapter`, `!ndiskd.nbl`, `!ndiskd.pkt` |
| Filter Manager | `fltkd.dll` | `!fltkd.filters`, `!fltkd.volumes`, `!fltkd.frame`, `!fltkd.cbd` (callback data) |
| TCPIP | `tcpip.dll` ext | `!tcpip.tcb`, `!tcpip.endp`, `!tcpip.ip6packet` |
| Storage | `storagekd.dll` | `!storagekd.storeport`, `!storagekd.classpnp` |
| USB | `usbkd.dll` | `!usbkd.usbanalyze`, `!usbkd.dumpurb`, `!usbkd.usbtree` |
| HID | `hidkd.dll` | `!hidkd.devices`, `!hidkd.hidirp` |
| GDI / Win32k | `win32k.dll` ext | `!w32k`, `!gdikdx.gh`, `!w32k.thread` |
| PnP / device tree | built-in (`nt`) | `!devnode 0 1`, `!devstack`, `!drvobj`, `!arbiter` |
| Pool & memory | built-in (`nt`) | `!pool`, `!poolused`, `!vm`, `!address` |
| Locks | built-in (`nt`) | `!locks`, `!qlocks`, `!pushlocks` |
| Security | built-in (`nt`) | `!sd <addr>`, `!sid <addr>`, `!token <addr>` |

### PnP / device tree walking

```
!devnode 0 1                        ; whole tree from root, with names
!devnode 0 1 mydevice               ; subtree filtered by instance name
!devnode <devnode_addr> 7           ; verbose: caps, state, drivers
!devstack <PDEVICE_OBJECT>          ; stack for a single device
!drvobj <DRIVER_OBJECT>             ; entry points + dispatch table
!drvobj <DRIVER_OBJECT> 7           ; verbose: also list all device objects
!arbiter 9                          ; resource arbitration state
```

### NDIS network triage

```
!ndiskd.miniports                   ; list all miniport adapters with state
!ndiskd.netadapter <addr>           ; detailed adapter info
!ndiskd.nbl <NET_BUFFER_LIST>       ; decode an NBL chain
!ndiskd.help                        ; ~50 more commands
```

### Filter Manager triage

```
!fltkd.filters                      ; all loaded minifilters and their altitude
!fltkd.frame <frame_addr>           ; per-frame view
!fltkd.cbd <callback_data>          ; what operation the filter is handling
!fltkd.instances <filter>           ; per-volume attachments
```

### gflags — runtime kernel/user flag tweaks

`gflags.exe` is a separate utility (in WDK / SDK, not an extension), but commonly paired with debugger work. It manipulates `NtGlobalFlag` (kernel) and `GlobalFlag` (per-image):

```cmd
gflags /r +ksl                      ; kernel: enable kernel-stack-trace database
gflags /i myapp.exe +ust            ; user: enable user-mode-stack-trace database for myapp
gflags /i myapp.exe +hpa            ; enable full page-heap (slower, more thorough than appverif Heaps)
gflags /i WUDFHost.exe +loadflagsetdebuger
```

Combined with `!htrace`, `!heap -p -a <addr>`, and `!stacks`, gflags is the way to capture call sites for handle / heap leaks.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Loading the wrong-bitness extension | `.load` succeeds but commands silently misread structures | Use the matching `winext` for the target architecture |
| Calling `!ndiskd.help` and seeing zero commands | Symbol path missing or wrong NDIS version | `.reload /f ndis.sys` first |
| `!devnode` returning addresses that look like garbage | Symbols for `nt` / `pnp` not loaded | `.reload /f nt` |
| Mixing extension names: `!wdfkd.wdfdevice` vs `!wdfdevice` | Some are namespaced, some not | Always prefix with the extension name when ambiguous |
| Using `!process 0 17` and grepping the output | Output is megabytes; debugger may stall | Use `!process 0 7 myapp.exe` to filter |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/extensions-for-windows-drivers
- https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/-ndiskd-help
- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/gflags
- https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/-devnode
