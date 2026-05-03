# Windows Drivers - Driver Debugging - Bug Check Catalog

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-code-reference2
> **Skill**: dev-suite skill `windows/driver-debugging` — see SKILL.md for the always-loaded quick reference.

## What this covers

Extended catalog of bugcheck codes commonly hit by driver code, with the meaning of each parameter and the typical bug pattern that produces it. SKILL.md lists the top ~9; this file covers an additional ~25 with the parameter decoding table you need at the keyboard.

## Deep dive

### IRQL / scheduling family

| Code | Name | Param interpretation | Usual bug |
|------|------|----------------------|-----------|
| `0x0A` | IRQL_NOT_LESS_OR_EQUAL | P1=addr, P2=IRQL, P3=read/write, P4=faulting IP | Touched paged data at >= DISPATCH_LEVEL or freed memory |
| `0xD1` | DRIVER_IRQL_NOT_LESS_OR_EQUAL | Same as 0x0A but always a driver | Pageable code/data at DISPATCH_LEVEL — check `MmIsAddressValid` patterns |
| `0xC5` | DRIVER_CORRUPTED_EXPOOL | P1=addr, P2=IRQL, P3=r/w, P4=IP | Pool corrupted to the point of being unreadable |
| `0xCA` | PNP_DETECTED_FATAL_ERROR | P1=substatus | Driver returned an invalid code from a PnP IRP |
| `0xCB` | DRIVER_LEFT_LOCKED_PAGES_IN_PROCESS | P1=callerAddr, P2=count | Forgot `MmUnlockPages` before unload/process exit |

### Pool / memory family

| Code | Name | Param interpretation | Usual bug |
|------|------|----------------------|-----------|
| `0xC1` | SPECIAL_POOL_DETECTED_MEMORY_CORRUPTION | P1=addr, P2=corruption type | Verifier's special pool tripped — guard page write |
| `0xC2` | BAD_POOL_CALLER | P1=op (e.g. 7=double-free), P2=tag, P3=ptr | Wrong pool free, double-free, wrong tag |
| `0xC4` | DRIVER_VERIFIER_DETECTED_VIOLATION | P1=rule (see verifier doc), P2..P4=context | Verifier rule fired |
| `0xC9` | DRIVER_VERIFIER_IOMANAGER_VIOLATION | P1=violation type | I/O misuse — most often incomplete IRP |
| `0xCC` | PAGE_FAULT_IN_FREED_SPECIAL_POOL | P1=addr | Use-after-free, with special pool guard fired |
| `0x1A` | MEMORY_MANAGEMENT | P1=substatus (e.g. 0x41284) | Mm subsystem invariant broken; often hardware (RAM) or page-table corruption |
| `0x4E` | PFN_LIST_CORRUPT | P1=substatus | Driver wrote past an MDL or freed pages out from under Mm |

### Power / PnP / Plug and Play family

| Code | Name | Param interpretation | Usual bug |
|------|------|----------------------|-----------|
| `0x9F` | DRIVER_POWER_STATE_FAILURE | P1=subtype (1..C), P2=device object | Driver didn't complete a Sx/Dx IRP within timeout — see the subtype table |
| `0x44` | MULTIPLE_IRP_COMPLETE_REQUESTS | P1=IRP | Same IRP completed twice |
| `0x48` | CANCEL_STATE_IN_COMPLETED_IRP | P1=IRP | Cancel routine still set after `IoCompleteRequest` |
| `0xA0` | INTERNAL_POWER_ERROR | P1=substatus | Power manager internal — often device left in invalid state |
| `0xEF` | CRITICAL_PROCESS_DIED | P1=EPROCESS | A protected user-mode process exited; sometimes triggered by driver |

### File system / filter family

| Code | Name | Param interpretation | Usual bug |
|------|------|----------------------|-----------|
| `0x24` | NTFS_FILE_SYSTEM | P1..P4 = NTFS-internal source line+context | NTFS internal; usually disk corruption or filter bug |
| `0x27` | RDR_FILE_SYSTEM | Same | SMB redirector |
| `0x50` | PAGE_FAULT_IN_NONPAGED_AREA | P1=addr, P2=r/w, P3=IP, P4=type | Touched freed nonpaged memory or bad address |
| `0x7E` | SYSTEM_THREAD_EXCEPTION_NOT_HANDLED | P1=ExceptionCode, P2=ExceptionAddr, P3=ExceptionRecord, P4=ContextRecord | Use `.exr P3` and `.cxr P4` to get the real fault |
| `0x1E` | KMODE_EXCEPTION_NOT_HANDLED | Same as 0x7E but for any thread | `.exr` / `.cxr` then `kP` |

### Code-integrity / security family

| Code | Name | Param interpretation | Usual bug |
|------|------|----------------------|-----------|
| `0x139` | KERNEL_SECURITY_CHECK_FAILURE | P1=subtype (e.g. 3=stack canary, 5=CFG, 9=LIST_ENTRY) | Buffer overflow, list corruption |
| `0x18B` | SECURE_KERNEL_ERROR | P1=subtype | VTL1 / VBS check failed — driver violated HVCI |
| `0x1AA` | EXCEPTION_ON_INVALID_STACK | P1=fault addr, P2=stack base | Stack pointer pointed outside thread's stack — usually corruption |

### How to use the table

1. Read the bugcheck name in `!analyze -v` first line.
2. Look up the code in this table.
3. Note the parameter interpretation — `!analyze` prints them as `Arg1..Arg4`.
4. For `0x7E` / `0x1E` always run `.exr <Arg3>` and `.cxr <Arg4>` to switch to the actual exception context.
5. For `0xC4` (Verifier), read parameter 1 and look it up in the Driver Verifier rule table.
6. Confirm with `kP` and module-level inspection (`lm vm <driver>`).

### Special handling

- **`0x9F` subtype 0x3**: device object stuck in a power transition. `!devstack` on `Arg2`, then `!irp` on the pending IRP.
- **`0x139` subtype 0x9**: a `LIST_ENTRY` was corrupted. `kP` to find the touching code; usually a missing lock or use-after-free.
- **`0xCA`**: substatus `0x6` = invalid IRP completion. `!irp <pending irp>` to see what the driver returned.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Reading `Arg2` of `0xD1` as the bad address | It's the IRQL, not the address (Arg1 is the address) | Always read the parameter decoding from the bugcheck doc |
| Diagnosing `0x1E` from the top stack frame alone | Top is the OS exception path; the real fault is in the trap context | Use `.cxr` then `kP` |
| Treating `0xC2` and `0xC4` as the same | `0xC2` = wrong API usage; `0xC4` = Verifier-detected | Check the bugcheck name |
| Repeating "page fault in nonpaged area" without checking the address | Could be UAF, invalid pointer, or hardware | Use `!pool <addr>` and `!address <addr>` |
| Skipping `0xCB` after a unload | Looks unrelated, actually a driver leaking locked MDLs | Search the driver for `MmProbeAndLockPages` without matching `MmUnlockPages` |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-code-reference2
- https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-0x9f--driver-power-state-failure
- https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-0xc4--driver-verifier-detected-violation
- https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-0x139--kernel-security-check-failure
