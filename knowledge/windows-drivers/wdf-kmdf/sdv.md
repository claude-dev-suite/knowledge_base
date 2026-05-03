# Windows Drivers - KMDF - Static Driver Verifier (SDV)

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/static-driver-verifier
> **Skill**: dev-suite skill `windows/wdf-kmdf` — see SKILL.md for the always-loaded quick reference.

## What this covers

How SDV models a KMDF driver, the rule packs you must pass before WHQL/HCK submission, the MSBuild flow (`/t:sdv`), the practical interpretation of SDV defects, and the suppression mechanism (`Sdv-default.xml`, `#pragma prefast`). Read when an SDV run flags a defect you don't understand or when integrating SDV into CI.

## Deep dive

### How SDV works

SDV is an abstract interpreter that exhaustively explores reachable states of your driver against a set of "rules" (model programs). For each rule it asks: is there ANY input under which the driver violates the rule? It uses your function entry points (DriverEntry, Evt callbacks, completion routines) as program entries.

Symbol resolution comes from your build's `.obj` files plus a set of `.lib` stubs that SDV swaps in for kernel APIs. SAL annotations on your functions tighten the search — the more accurate your SAL, the faster and more precise SDV's answer.

### Building for SDV

SDV requires a single-build, single-target driver. From an EWDK or Visual Studio Developer Command Prompt, in the driver's project directory:

```cmd
msbuild MyDriver.vcxproj ^
        /t:sdv /p:Inputs="/clean" ^
        /p:Configuration="Release" /p:Platform=x64

msbuild MyDriver.vcxproj ^
        /t:sdv /p:Inputs="/check:default.sdv" ^
        /p:Configuration="Release" /p:Platform=x64
```

This produces `sdv\SDV.DVL.xml` (the Driver Verification Log) which is a required artifact for HCK submission. To inspect findings interactively use `sdv /view`.

### Rule sets

| Rule pack | What it checks |
|-----------|---------------|
| `default.sdv` | Standard set; required for HCK |
| `wdf.sdv` | KMDF-specific (queue lifetime, request completion, IRQL on Evt callbacks) |
| `ndis.sdv` | NDIS miniports |
| `storport.sdv` | StorPort miniports |
| `mapped_io.sdv` | Buffer mapping safety |

Always run `default.sdv` and any framework-specific pack relevant to your driver class.

### Reading a defect

SDV defects look like:

```
Defect: IrqlRaiseLowerCheck
Function: MyEvtIoDeviceControl
Step 12: Acquired WDFSPINLOCK 'g_DataLock'
Step 17: KeWaitForSingleObject called at IRQL DISPATCH_LEVEL
```

The trace is a *witness* — a concrete path through your code that violates the rule. The fix is almost never to suppress; it's to refactor so the trace becomes infeasible. In the example, you'd hand off to a workitem before waiting.

### Common rule violations and fixes

| Rule | Typical cause | Fix |
|------|---------------|-----|
| `IrqlRaiseLowerCheck` | Wait at DISPATCH | Workitem to PASSIVE |
| `WdfRequestCompletion` | Path that exits without `WdfRequestComplete` | Audit every code path; consider `__analysis_assume(0)` only after full review |
| `RequestCompletedLocal` | Forward request then complete it | After `WdfRequestForwardToIoQueue`, return immediately |
| `EvtIoStopRule` | `EvtIoStop` doesn't acknowledge or complete | Add `WdfRequestStopAcknowledge` |
| `LockReleaseCheck` | Path leaves function with lock held | Mirror acquire/release on every branch |
| `MdlAfterReqCompletedIoctl` | Touching MDL after completion | Capture MDL pointer before complete |

### Suppressions (use sparingly)

Per-line:

```c
__pragma(prefast(suppress: __WARNING_RACE_CONDITION_AROUND_LOCKS,
        "False positive: lock acquired conditionally in caller; SAL on caller is sufficient"))
```

Project-level via a `Sdv-default.xml` or `Sdv-map.h` to suppress an entire rule:

```xml
<!-- Sdv-default.xml -->
<DriverData>
  <Defects>
    <Defect Id="DoubleCompletion" Suppressed="true" />
  </Defects>
</DriverData>
```

Every suppression must be paired with a comment explaining why the diagnostic is wrong. Reviewers will reject blank suppressions.

### Integrating into CI

SDV needs the EWDK or full WDK. Typical Azure Pipeline / GitHub Actions step on a Windows runner:

```yaml
- name: SDV
  run: |
    msbuild MyDriver.vcxproj /t:sdv /p:Inputs="/clean"
    msbuild MyDriver.vcxproj /t:sdv /p:Inputs="/check:default.sdv;wdf.sdv"
    msbuild MyDriver.vcxproj /t:dvl
- name: Upload DVL
  uses: actions/upload-artifact@v4
  with:
    name: dvl
    path: '**/SDV.DVL.xml'
```

Treat any new defect as a build break; gate merges on a clean DVL.

### CodeQL alongside SDV

CodeQL covers some of the same ground (unverified buffer sizes, missing completions) plus modern queries (cryptographic misuses, bounds checks). Run both in CI; they catch overlapping but non-identical issues. Microsoft publishes the must-pass query suite in `Windows-Driver-Developer-Supplemental-Tools`.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Running SDV on a debug build with whole-program optimization off | Different inlining, different defects | Always run on Release |
| Treating SDV warnings as informational | HCK gate fails on any defect | Resolve or suppress with justification BEFORE submission |
| Suppressing `WdfRequestCompletion` blanket | Hides real lifetime bugs | Refactor to single completion path |
| SDV runs forever | Excessive callback complexity | Split functions, narrow `_When_` annotations |
| Not regenerating DVL after fix | Stale artifact | Run `msbuild /t:dvl` after every clean check |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/static-driver-verifier-rules
- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/creating-a-driver-verification-log
- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/codeql-and-the-static-tools-logo-test
