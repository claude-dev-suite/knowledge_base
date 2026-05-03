# Windows Drivers - Driver Debugging - !wdfkd Reference

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/wdfkd-debugger-extensions
> **Skill**: dev-suite skill `windows/driver-debugging` — see SKILL.md for the always-loaded quick reference.

## What this covers

Detailed reference for the `wdfkd.dll` extensions — what each command shows, how to walk a WDF object hierarchy from `WDFDRIVER` down to `WDFREQUEST`, and how to debug power transitions, queue stalls, and reference leaks. Fetch when SKILL.md's three-line WDF table is not enough.

## Deep dive

### Bootstrap

```
.load wdfkd
!wdfkd.help                          ; full list (50+ commands)
!wdfkd.wdfldr                        ; verify Wdf01000 (KMDF) or WUDFx02000 is loaded
!wdfkd.wdfsearch <pattern>           ; find a command by keyword
```

### Top-down hierarchy walk

```
!wdfkd.wdfdriverinfo MyDriver          ; KMDF metadata about your driver
!wdfkd.wdfdriverinfo MyDriver 0xff     ; everything: handles, devices, queues, refs

!wdfkd.wdfdevice <wdfdevice>           ; one device — pnp/power state, callbacks, queues
!wdfkd.wdfdevice <wdfdevice> 0xff      ; verbose: pending IRPs, child list, IO targets

!wdfkd.wdfqueue <wdfqueue>             ; queue state, request count, dispatch type
!wdfkd.wdfqueue <wdfqueue> 0xff        ; per-request list

!wdfkd.wdfrequest <wdfrequest>         ; one request: cancellation, IRP, IRQL, target
!wdfkd.wdfrequest <wdfrequest> 0xff    ; full IRP chain
```

### Power and PnP debugging

```
!wdfkd.wdfdevicepower <wdfdevice>     ; current Sx/Dx state, last transition
!wdfkd.wdfpnp <wdfdevice>              ; pnp state machine current node + history
!wdfkd.wdfdevicequeues <wdfdevice>    ; all queues hanging off this device (default + manual)
!wdfkd.wdfintwakeable                  ; devices that can wake the system
!wdfkd.wdfwmiinstance                  ; WMI instances exposed by WDF
```

The state-machine history is gold for "why is my device stuck in `D3hot`" — every transition with timestamp and trigger is recorded.

### Object reference debugging (leak hunting)

```
!wdfkd.wdfobject <wdfobj>              ; ref count, parent, child list, attributes
!wdfkd.wdfhandle <handle>              ; handle → object resolution
!wdfkd.wdfdriverinfo MyDriver 0x80     ; all live objects of the driver
!wdfkd.wdftagtracker <wdfobj>          ; if WdfDriverGlobalsObjectAttributes had ContextSizeOverride / tag tracking on
```

For tag tracking to be meaningful, build with `WPP_USE_OBJECT_TAGS` and call `WdfObjectReferenceWithTag` — every `Add/ReleaseRef` is logged with file/line.

### Common diagnostic commands

| Command | Use |
|---------|-----|
| `!wdfkd.wdflogdump <wdfdriver>` | Dump the WDF in-flight recorder log (very useful — turns silent failures into a transcript) |
| `!wdfkd.wdfkd` | The high-level overview command — start here on a fresh dump |
| `!wdfkd.wdfshowtargetstate` | I/O targets: opened, started, stopped, reset |
| `!wdfkd.wdftriage` | Auto-detects the driver, dumps a triage summary |
| `!wdfkd.wdftmffile +<file>` | Provide a TMF file for in-flight recorder messages without symbols |

### A typical investigation: "my queue stopped dispatching"

```
.load wdfkd
.reload /f Wdf01000.sys
!wdfkd.wdftriage                          ; lists candidate WDF drivers
!wdfkd.wdfdriverinfo MyDriver 0xff        ; find your WDFDEVICE handles
!wdfkd.wdfdevice <wdfdev>                 ; check power state — Sx/Dx
!wdfkd.wdfdevicequeues <wdfdev>           ; list queues
!wdfkd.wdfqueue <wdfqueue> 0xff           ; check IsPowerManaged, IsAcceptingRequests
!wdfkd.wdflogdump <wdfdriver>             ; recent EvtIo* invocations and returns
```

If `IsPowerManaged=TRUE` and the device is in `D3`, the queue **will not dispatch** until you transition back to `D0` — that's the bug class to look at first.

### UMDF differences

For UMDF 2 drivers, the host is `WUDFHost.exe`. The same commands work but you need a user-mode dump or a process-context switch:

```
!process 0 7 WUDFHost.exe                ; find the host PID
.process /i <eprocess>                    ; switch context
.reload /user
!wdfkd.wdfldr                             ; should now show WUDFx02000.dll
```

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Calling `!wdfkd.wdfdevice` with a `PDEVICE_OBJECT` instead of `WDFDEVICE` | Different handle types — silent garbage | Use `!wdfkd.wdfdriverinfo MyDriver 0xff` to enumerate proper WDFDEVICE handles |
| Not loading `Wdf01000.sys` symbols | Output is just hex addresses | `.reload /f Wdf01000.sys` |
| Confusing `WDFDEVICE` (handle) with the underlying object pointer | `wdfdevice` accepts handles, `wdfobject` accepts pointers | Use `!wdfkd.wdfhandle <h>` to resolve |
| Assuming the IFR (in-flight recorder) is on by default | It's on for chk builds, off for fre by default | Build with `WPP_INIT_TRACING(...)` and ship the TMF |
| Reading `IsAcceptingRequests=FALSE` and panicking | Often correct — queue stopped during PnP transition | Check the power/pnp state first |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/-wdfkd-wdftriage
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/using-the-framework-s-event-logger
- https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/-wdfkd-wdflogdump
