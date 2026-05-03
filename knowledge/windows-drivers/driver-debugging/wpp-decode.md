# Windows Drivers - Driver Debugging - WPP Decode Workflow

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/tracefmt
> **Skill**: dev-suite skill `windows/driver-debugging` — see SKILL.md for the always-loaded quick reference.

## What this covers

End-to-end WPP (Software Tracing) decode workflow: extracting TMF files from a `.pdb`, running `tracefmt` against an `.etl`, alternative routes via `traceview` and the in-flight recorder, and how to find the GUID for a third-party provider you didn't write. Fetch when SKILL.md's two-line tracefmt example doesn't explain why your decode is empty.

## Deep dive

### Why WPP exists

WPP wraps `DbgPrint` with structured tracing: each `TraceEvents(...)` call is compiled to an event with a numeric ID. The format strings are stripped from the binary into a TMF (Trace Message Format) file embedded in the PDB. Decoding requires the original PDB or a TMF that matches.

### Build-side: what makes a driver WPP-capable

In sources/.vcxproj, WPP preprocessing adds these:

```xml
<ItemDefinitionGroup>
  <WppPreProcessor>
    <ScanConfigurationData>tracing.h</ScanConfigurationData>
    <FunctionName>FUNC TraceEvents(LEVEL,FLAGS,MSG,...)</FunctionName>
    <KernelMode>true</KernelMode>
    <PreserveExtension>true</PreserveExtension>
    <RecordersConfiguration>tracing.h</RecordersConfiguration>
  </WppPreProcessor>
</ItemDefinitionGroup>
```

Output: per-source-file `.tmh` headers, plus TMF data linked into the `.pdb`.

### Runtime: starting and capturing

```cmd
:: 1. Find the provider GUID (yours is in a tracing.h define, e.g. WPP_DEFINE_CONTROL_GUID)
::    Or list registered providers:
logman query providers

:: 2. Start a session
tracelog -start MyTrace -guid #11111111-2222-3333-4444-555555555555 ^
    -f C:\trace.etl -flag 0xFFFF -level 5

::    Alternative form using a GUID file (one GUID per line)
echo {11111111-2222-3333-4444-555555555555} 0xFFFF 5 > guids.txt
tracelog -start MyTrace -guid guids.txt -f C:\trace.etl

:: 3. Reproduce the bug

:: 4. Stop
tracelog -stop MyTrace
```

For continuous capture (circular buffer):

```cmd
tracelog -start MyTrace -guid guids.txt -f C:\trace.etl ^
    -flag 0xFFFF -level 5 -b 64 -min 4 -max 16 -ft 1 -cir 64
```

`-cir 64` = 64 MB ring buffer; oldest events overwrite newest.

### Decoding with tracefmt

```cmd
:: Single PDB
tracefmt -p C:\symbols\MyDriver.pdb C:\trace.etl -o C:\trace.txt

:: Symbol server path (preferred — pulls TMFs for OS providers too)
tracefmt -i srv*C:\symbols*https://msdl.microsoft.com/download/symbols ^
    -p C:\symbols\MyDriver.pdb C:\trace.etl -o C:\trace.txt

:: Multiple TMF files in a directory
tracefmt -p C:\tmf C:\trace.etl -o C:\trace.txt

:: With absolute timestamps and CSV-friendly output
tracefmt -tmf C:\tmf -displayonly -p C:\symbols ^
    -nosummary -noheaders C:\trace.etl > trace.csv
```

A typical decoded line:

```
[0]0064.0078::04/18/2026-12:33:01.123 [MyDriver]EvtIoRead Entry: Length=0x1000 Queue=0xffffabcd00010000
[0]0064.0078::04/18/2026-12:33:01.124 [MyDriver]EvtIoRead Exit: Status=STATUS_SUCCESS BytesRead=0x1000
```

Format: `[CPU]ProcessId.ThreadId::Timestamp [Module]MessageText`.

### When tracefmt prints "No TMF found for message"

This means the GUID matched but the format strings are missing. Causes:

1. PDB stripped of TMF section (e.g. `link /pdbstripped`) — get the unstripped PDB.
2. Driver was built with `WPP_DEFINE_BIT()` differently — TMF and ETL come from different builds.
3. Provider is a non-WPP ETW provider — use `tracerpt` (XML) instead.

To extract a TMF without an ETL:

```cmd
tracepdb -f C:\symbols\MyDriver.pdb -p C:\tmf
```

Now `C:\tmf\<guid>.tmf` exists; pass that to `tracefmt -p C:\tmf`.

### TraceView (legacy GUI)

```cmd
traceview.exe
:: File -> Create New Log Session
:: Provider: GUID Browser, paste your GUID
:: Source: PDB or TMF directory
:: Live decoding window
```

Useful for live triage; saves `.ctl` configurations you can re-run with `traceview -e <ctl>`.

### Modern alternative: TraceLogging / WPR

If your driver uses `TraceLoggingProvider.h` instead of WPP, capture with WPR and view in WPA — no TMF needed.

```cmd
wpr -start GeneralProfile -filemode
:: repro
wpr -stop C:\trace.etl
wpa C:\trace.etl
```

### In-flight recorder (no manual capture needed)

KMDF maintains an in-memory ring of recent WPP messages, dumpable from a kernel debugger or kernel dump:

```
.load wdfkd
!wdfkd.wdflogdump <WDFDRIVER>
```

This gives you the last few thousand messages from the moment of the bugcheck — invaluable when you didn't have tracelog running.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Decoding with the wrong PDB | TMF mismatch — every line says "No TMF found" | Pin the build's PDB next to the SYS via build IDs |
| Forgetting `-flag` and `-level` | Driver only emits the messages your build defined; defaults capture nothing | Use `-flag 0xFFFF -level 5` for full verbosity |
| Provider GUID mismatch | Tracelog session started, but no events | Verify with `logman query providers <guid>` |
| ETL file grows unbounded in lab | Disk full mid-repro | Always use `-cir N` for ring-buffered captures |
| Mixing WPP and TraceLogging in the same provider | Decoders can't mix and match | Pick one per provider |
| Stripped PDB shipped to the public symbol server | Customers cannot decode | Keep the unstripped PDB on your symbol server |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/tracelog
- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/traceview
- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/tracepdb
- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/wpp-software-tracing
