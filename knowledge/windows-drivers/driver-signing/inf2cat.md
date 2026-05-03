# Windows Drivers - Driver Signing - inf2cat Reference

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/inf2cat
> **Skill**: dev-suite skill `windows/driver-signing` — see SKILL.md for the always-loaded quick reference.

## What this covers

`inf2cat.exe`: command-line options, the `/os:` target list (Win10, Server 2025, etc.), how the catalog binds to the INF, output format, and how to debug "INF is not valid" errors. Fetch when SKILL.md's `inf2cat /driver:... /os:10_X64` line bounces with an unhelpful error.

## Deep dive

### Where it lives

```
C:\Program Files (x86)\Windows Kits\10\bin\<sdk-version>\x86\inf2cat.exe
```

The 32-bit binary works for all targets; there is no 64-bit equivalent. It must be run from a directory writable by the user — it generates a `.cat` next to the INF.

### Basic usage

```cmd
inf2cat /driver:C:\drvbuild /os:10_X64
```

- `/driver:<dir>` — the directory containing the INF and all referenced files. **Not** a path to the INF — a path to the folder.
- `/os:<list>` — comma-separated list of target OS families.
- `/uselocaltime` — use local time in the catalog (default is UTC). Almost never needed.
- `/nocat` — validate INF only, don't write a catalog.
- `/verbose` — print every check.

### Supported `/os:` values

| Token | Meaning |
|-------|---------|
| `10_X64` | Windows 10/11 client x64 |
| `10_X86` | Windows 10/11 client x86 |
| `10_AU_X64` | Win10 Anniversary Update x64 (legacy alias) |
| `10_RS1_X64` | Win10 1607 x64 (legacy alias) |
| `10_ARM64` | Windows 10/11 ARM64 client |
| `Server10_X64` | Windows Server 2016+ x64 |
| `Server2025_X64` | Windows Server 2025 x64 |
| `ServerARM64` | Windows Server ARM64 |
| `8_X64`, `8_X86`, `8_ARM` | Windows 8 (legacy — rarely needed) |
| `7_X64`, `7_X86` | Windows 7 (legacy) |

You can chain multiple targets:

```cmd
inf2cat /driver:C:\drvbuild /os:10_X64,10_ARM64,Server10_X64,Server2025_X64
```

The output `.cat` will declare it is valid for **all** listed OSes. If any of the `[Manufacturer]` decorations don't match a listed target, inf2cat fails (see common errors below).

### Output catalog format

A `.cat` is a PKCS#7 SignedData blob containing a `MS_CTL` (Microsoft Certificate Trust List) inner structure. Each entry has:
- File name
- SHA-256 hash
- Page hashes (for the SYS, used by HVCI)
- OS attribute strings — one per `/os:` target

You can inspect with:

```cmd
:: Friendly view
signtool verify /v /pa MyDriver.cat

:: Raw view via certutil
certutil -dump MyDriver.cat
```

### Validation rules inf2cat applies

1. INF must parse cleanly (`infverif`-style checks).
2. Every file referenced in `[SourceDisksFiles]` must exist in `/driver:` directory.
3. `[Manufacturer]` section's TargetOS decoration must include all `/os:` targets.
4. Architecture decorations must match: `[Standard.NT$ARCH$.10.0...19041]` covers 10_X64, 10_X86, 10_ARM64 via `$ARCH$` substitution.
5. `CatalogFile=<basename>.cat` directive in `[Version]` is required.
6. `DriverVer=` date must parse (MM/DD/YYYY).
7. No deprecated directives (`Include`, `Needs`) for universal drivers.

### Common errors decoded

```
*** Signability test failed.
Errors:
22.9.7: DriverVer set to incorrect date (post-dated) in MyDriver.inf
```
Fix: set `DriverVer=` to today or earlier (`MM/DD/YYYY,X.Y.Z.W`).

```
22.1.4: Inf is missing CatalogFile= directive in [Version] section
```
Fix: add `CatalogFile=MyDriver.cat` to `[Version]`.

```
22.1.5: Section [Standard.NT$ARCH$.10.0...19041] not found in INF
```
Fix: your `[Manufacturer]` references this decoration but no `[Standard...]` section defines it.

```
22.9.4: 10_X64 target requires that all sections include the .NT$ARCH$ decoration
```
Fix: legacy section like `[Standard.NTAMD64]` is not allowed when targeting Win10 universal — must be `[Standard.NTAMD64.10.0...19041]` or `[Standard.NT$ARCH$.10.0...19041]`.

```
22.9.5: File MyDriver.sys does not exist or is not readable
```
Fix: file path in `[SourceDisksFiles]` doesn't match the actual file under `/driver:`.

### Verbose output for debugging

```cmd
inf2cat /driver:C:\drvbuild /os:10_X64 /verbose > inf2cat.log 2>&1
notepad inf2cat.log
```

The verbose log shows every file inf2cat hashed, every OS check, and the order in which they were applied — invaluable when an error message references a section you can't find.

### After inf2cat succeeds

The catalog is **unsigned**. You must sign it with the same EV (or test) cert used for the SYS:

```cmd
signtool sign /v /f cert.pfx /p pwd ^
    /tr http://timestamp.digicert.com /td sha256 /fd sha256 ^
    C:\drvbuild\MyDriver.cat
signtool verify /v /pa C:\drvbuild\MyDriver.cat
```

Any change to the SYS, INF, or any file in `[SourceDisksFiles]` invalidates the catalog hashes — re-run inf2cat then re-sign.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Run inf2cat from a read-only path | Tool fails to write the .cat | Copy package to a writable dir first |
| Pointed `/driver:` at the INF file, not the folder | Cryptic "INF not found" | Always pass the folder path |
| Used `/os:10_AU_X64` in 2026 | Legacy alias still works but produces older catalog format | Use the canonical `10_X64` |
| Modified the SYS after inf2cat | CAT hash mismatch — install fails | Re-run the full chain: build → inf2cat → sign cat |
| Forgot to bump `DriverVer` between submissions | Same .cat hash on two binaries | Always increment version |
| Targeted `10_X64` but INF has only `[Standard.NTAMD64]` (no decoration) | Section not matched, build fails | Add the version decoration `.10.0...19041` |
| Tried to use 64-bit cabarc / makecab tools to bundle | inf2cat doesn't produce a CAB — separate step | Use makecab after inf2cat |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/install/inf-version-section
- https://learn.microsoft.com/en-us/windows-hardware/drivers/install/inf-manufacturer-section
- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/infverif
