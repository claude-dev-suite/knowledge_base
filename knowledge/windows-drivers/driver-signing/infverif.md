# Windows Drivers - Driver Signing - infverif

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/infverif
> **Skill**: dev-suite skill `windows/driver-signing` — see SKILL.md for the always-loaded quick reference.

## What this covers

`infverif.exe` full reference for INF compliance checks: the `/v` (verbose), `/w` (universal driver), `/u` (Universal Windows), and `/h` (HLK) modes, the rules each one applies, and how to interpret the most common error codes. Fetch when SKILL.md's "run infverif" line bounces with errors you don't recognise.

## Deep dive

### Modes

| Flag | Rule set | When to use |
|------|----------|-------------|
| `/v` | Verbose: print everything | Always combine with another mode |
| `/w` | Universal driver compliance (Win10 1607+) | Default for any new driver targeting Win10/11 |
| `/u` | Universal Windows Driver (UWD) — even stricter, Windows 10 IoT / Mobile / HoloLens | Cross-edition drivers |
| `/h` | HLK / WHQL submission rules | Pre-flight before HLK package |

```cmd
infverif /v /w MyDriver.inf
infverif /v /u MyDriver.inf
infverif /v /h MyDriver.inf
```

`/w` is what attestation-signed drivers need. `/u` is a superset for the niche scenario of identical driver across editions. `/h` is for drivers going through HLK.

### Rule categories under /w (universal)

1. **Forbidden directives**: `Include`, `Needs`, `Reboot`, `LogConfig`, `RegistryKeysToHide`, `BootCriticalDriver`
2. **Forbidden values in legal directives**: `CopyFiles` to system directories outside DriverStore, paths starting with `%SystemRoot%`
3. **Architecture decoration required**: every section referenced from `[Manufacturer]` must have `.NT$ARCH$.10.0...19041` (or numerically newer) decoration
4. **Catalog directive required**: `CatalogFile=<name>.cat` must be present
5. **DriverVer date**: not in the future, not before year 2000
6. **No co-installer entries**: `CoInstaller32=` and `CoInstaller64=` rejected
7. **Service entries**: `ServiceType` numeric only (kernel = 1, file system = 2), `StartType` < 4
8. **Class GUID**: must match the Class name from the canonical class registry
9. **File destination dirs**: only `12` (\Drivers), `13` (\DriverStore), `10` (\Windows), and a few others — most others rejected

### Output format

```
$ infverif /v /w MyDriver.inf

Inf2Cat passed.
Detail:
  Architecture: x64
  CatalogFile: MyDriver.cat
  DriverVer: 04/18/2026,1.2.3.4
  Section: [MyDriver_Inst.NT$ARCH$.10.0...19041]
  ...

Errors:
  HResult E000000A0 (PASS)  Section [DefaultInstall] is allowed but discouraged
  HResult E0000234  Co-installer not allowed in universal driver INF
  HResult E0000201  Section name [Standard.NTAMD64] missing version decoration
```

`E0000xxx` codes are persistent across versions; bookmark the codes you hit.

### Common error codes decoded

| Code | Meaning | Fix |
|------|---------|-----|
| `E0000201` | Section missing `.10.0...nnnnn` decoration | Add the version range, e.g. `[Std.NTAMD64.10.0...19041]` |
| `E0000234` | Co-installer present | Remove `CoInstaller32=`/`CoInstaller64=`, refactor logic into the driver |
| `E0000236` | INF references a file not in `[SourceDisksFiles]` | Add the file or remove the reference |
| `E0000242` | DriverVer in future | Set today's date |
| `E0000244` | DriverVer too old (<2000) | Set today's date |
| `E0000247` | Class is not a known Windows device class | Use a registered class GUID — see the class registry |
| `E0000250` | CatalogFile missing | Add `CatalogFile=MyDriver.cat` to `[Version]` |
| `E0000253` | Service start type out of range | Use 0 (boot), 1 (system), 2 (auto), 3 (demand) |
| `E0000260` | Forbidden directive `Reboot` | Trigger reboot via PnP if necessary, not in INF |
| `E0000262` | Forbidden registry destination | Use `HKR` (relative) or one of the allowed roots |
| `E0000270` | DDInstall section calls `[DefaultInstall]` | Remove — not allowed in universal |
| `E0000275` | File destination dir not in allowed list | Use 12 (Drivers) or 13 (DriverStore) |

### Discouraged-but-allowed warnings (E000000xx, "PASS")

These are warnings — submission still succeeds but Microsoft may flag them in review. Examples:

- `E000000A0` — `[DefaultInstall]` section present (should not be needed for PnP drivers)
- `E000000A1` — Section names with double dots, ambiguous matching
- `E000000B3` — Multiple service definitions with overlapping start types

### Verifying multiple INFs in a directory

```cmd
:: Run against every INF in folder
for %f in (C:\drvbuild\*.inf) do infverif /v /w "%f"

:: PowerShell equivalent
Get-ChildItem C:\drvbuild\*.inf | ForEach-Object { infverif /v /w $_.FullName }
```

### Use in CI

```yaml
- name: Validate INF universal compliance
  run: |
    & "$env:WindowsSdkVerBinPath\x86\infverif.exe" /v /w build\MyDriver.inf
    if ($LASTEXITCODE -ne 0) { throw "infverif failed" }
```

infverif returns 0 on full pass, non-zero if any error rule fires (warnings still return 0).

### Class GUID registry

The full canonical class GUID list is at: https://learn.microsoft.com/en-us/windows-hardware/drivers/install/system-defined-device-setup-classes-available-to-vendors

Custom classes need `[ClassInstall32]` with a `ClassGuid=` you control — but Microsoft strongly prefers re-use of standard classes (HID, Net, Display, etc.). Custom classes raise review scrutiny.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Forgot `/w` flag | Runs default rules, doesn't catch universal-only issues | Always pass `/w` for new drivers |
| Co-installer entries from a legacy template | Auto-rejected | Move logic into `EvtDevicePrepareHardware` or a Windows service |
| `[DefaultInstall]` section copied from sample | Warning + bad practice | Drop unless you're targeting non-PnP scenarios |
| Service type = 1 (kernel) with start type 0 (boot) | Allowed but flagged for review | Use 2 (auto) or 3 (demand) unless you have a compelling boot reason |
| Custom class GUID without `[ClassInstall32]` | Class not registered | Either register or use a standard class |
| INF passes `/w` but fails `/h` | HLK adds device-specific compliance rules | Run `/h` before HLK submission, fix per code |
| Different rules between WDK versions | Microsoft tightens checks each release | Pin WDK version in CI; update only deliberately |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/install/using-a-universal-inf-file
- https://learn.microsoft.com/en-us/windows-hardware/drivers/install/inf-section-directives-prohibited-in-universal-inf-files
- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/infverif-validation-rules
