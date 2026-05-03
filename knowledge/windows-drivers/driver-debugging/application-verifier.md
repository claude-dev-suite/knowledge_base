# Windows Drivers - Driver Debugging - Application Verifier

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/application-verifier
> **Skill**: dev-suite skill `windows/driver-debugging` — see SKILL.md for the always-loaded quick reference.

## What this covers

Application Verifier (`appverif.exe`) for UMDF host processes (`WUDFHost.exe`) and the user-mode tests / services that talk to a driver. Fault injection options, leak detection, lock-checking layers, and how to consume the bugs in WinDbg. Fetch when SKILL.md's "appverif /verify" line isn't enough.

## Deep dive

### Two ways to drive AppVerifier

```cmd
:: GUI
appverif

:: CLI: enable on a specific image (persisted in Image File Execution Options)
appverif /verify WUDFHost.exe
appverif /verify mytestapp.exe /faults
appverif /verify mysvc.exe /loglevel error

:: Disable
appverif /disable * /for WUDFHost.exe

:: List
appverif /logtoxml C:\appverif.xml
```

Settings persist via `HKLM\Software\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\<image>\GlobalFlag` plus an `VerifierFlags` value — wiping that key undoes everything.

### Available test layers

| Group | Layer | What it instruments |
|-------|-------|---------------------|
| Basics | Heaps | Page-heap with guard pages on every allocation |
| Basics | Handles | Detects close-of-bad / double-close |
| Basics | Locks | Critical section lifecycle and recursion |
| Basics | Memory | VirtualAlloc / VirtualFree mismatches |
| Basics | TLS | TLS index lifecycle |
| Basics | Exceptions | Catches first-chance exceptions in unexpected paths |
| Basics | SRWLock | Slim reader-writer lock misuse |
| Basics | Threadpool | Threadpool object lifecycle |
| LuaPriv | Privileges | Detects calls that require admin and would fail under standard user |
| Networking | NetworkingAPI | WinSock / WinHTTP misuse |
| Cuzz | Cuzz | Concurrent-execution scheduler — randomises thread interleavings to surface races |
| LowRes | LowResourceSimulation | Random allocation failures + delays |

For UMDF, **Heaps**, **Handles**, **Locks**, **Memory**, and **Exceptions** are the must-haves. Add **LowResourceSimulation** with `/faults` once the easy bugs are out.

### Fault injection in detail

```cmd
:: Inject 5% allocation failures, 10% file I/O failures, after a 30s warm-up,
:: at any of the listed APIs
appverif /verify mytestapp.exe /faults 30 5 10 ^
    HeapAlloc CreateFileW ReadFile RegOpenKeyExW
```

Format: `/faults <warmup_seconds> <fault_probability_percent> [API API ...]`. The list of injectable APIs is documented at the upstream link. Random by default — set `HKLM\Software\Microsoft\AppVerifier\Image\<image>\FaultInjectionSeed` for reproducibility.

### Catching bugs live in WinDbg

```cmd
:: Attach to the UMDF host
windbg -p $(Get-Process WUDFHost -ErrorAction SilentlyContinue | Select -First 1 -Expand Id)

:: Tell the engine to break on AppVerifier stops
sxe -c "!avrf" av

:: Inside WinDbg after a stop:
!avrf                       ; explain the current stop
!avrf -threads              ; per-thread layer state
!avrf -logo C:\avrf.log     ; flush the log
!avrf -hp                   ; heap-related details (page heap)
!htrace <handle>            ; handle history (with Handles layer on)
```

The breakpoints triggered by AppVerifier are precise — the failing thread's stack is the actual buggy one, no heuristic needed.

### Picking a config from a UMDF crash

| Symptom | Layer to add |
|---------|--------------|
| Random crash with bad heap blocks | Heaps |
| `INVALID_HANDLE_VALUE` operations succeeding | Handles |
| Hangs only under load | Cuzz |
| Crashes only on tablets / low-mem devices | LowResourceSimulation |
| `EnterCriticalSection` on uninitialised lock | Locks |
| Works as admin, fails as user | LuaPriv |

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Forgetting AppVerifier is persistent | The image keeps running with page-heap after you walk away | `appverif /disable * /for <image>` when done |
| Enabling on `svchost.exe` | All hosted services get verified — system instability | Target the specific UMDF host or your own exe |
| Heaps + LowRes at the same time | Allocations almost always fail; nothing makes progress | Stage them: Heaps first, then add LowRes |
| Cuzz on a release build with /Ox | Optimised stacks are noisy; hard to read | Run Cuzz on a `/Od /Zi` chk build |
| Mixing Driver Verifier and AppVerifier without separating their reports | Hard to know which fired | Capture the bugcheck (DV) and the `!avrf` output (AV) separately |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/application-verifier-stop-codes
- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/using-application-verifier-with-a-umdf-2-driver
- https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/-avrf
