# Windows Drivers - Driver Signing - Attestation Signing Step-by-Step

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/dashboard/attestation-signing-a-kernel-driver-for-public-release
> **Skill**: dev-suite skill `windows/driver-signing` — see SKILL.md for the always-loaded quick reference.

## What this covers

Complete Partner Center attestation submission flow: package layout, the new vs upgrade decision, OS family selection, status interpretation, and the typical pitfalls that bounce a submission. Fetch when SKILL.md's high-level "submit to Partner Center" line needs to be turned into a checklist.

## Deep dive

### Pre-flight checklist

Before you upload anything, confirm:

```cmd
:: 1. INF passes universal compliance
infverif /v /w MyDriver.inf

:: 2. Catalog signed by your EV cert (and timestamp present)
signtool verify /v /pa MyDriver.cat
signtool verify /v /kp /a MyDriver.sys

:: 3. .cab is signed end-to-end with EV cert
signtool verify /v /pa MyDriverPackage.cab

:: 4. INF DriverVer date is in the past and version is monotonic vs prior submissions
::    DriverVer = 04/18/2026,1.2.3.4

:: 5. No co-installers in package (deprecated)
::    No DPInst*.exe, no CoInstaller= entries
```

### Package layout

```
MyDriverPackage\
  MyDriver.inf
  MyDriver.sys           ; signed with EV
  MyDriver.cat           ; from inf2cat, signed with EV
  amd64\                 ; arch-specific binaries (one folder per arch)
    extra-binary.dll
  arm64\
    extra-binary.dll
  disk1                  ; cabinet sequencing — auto-generated
```

Compress to a single CAB at the root:

```cmd
:: sources.ddf
.OPTION EXPLICIT
.Set CabinetNameTemplate=MyDriverPackage.cab
.Set DiskDirectoryTemplate=.
.Set CompressionType=MSZIP
.Set Cabinet=on
.Set Compress=on
MyDriverPackage\MyDriver.inf
MyDriverPackage\MyDriver.sys
MyDriverPackage\MyDriver.cat
MyDriverPackage\amd64\extra-binary.dll
MyDriverPackage\arm64\extra-binary.dll

:: Build
makecab /F sources.ddf

:: Sign the CAB itself
signtool sign /v /a /n "Your Company" ^
    /tr http://timestamp.digicert.com /td sha256 /fd sha256 ^
    MyDriverPackage.cab
```

### Submission via Partner Center UI

1. Sign in at https://partner.microsoft.com/dashboard/hardware
2. **Drivers → New hardware → Driver submission**
3. **Submission name**: `<product>-<version>` (e.g. `MyDriver-1.2.3.4`)
4. Upload `MyDriverPackage.cab`
5. The validator runs server-side — wait 1–5 minutes
6. **Target Windows versions**: choose by OS family. For attestation only, you can pick:
   - Windows 10 Client x64 / arm64
   - Windows 11 Client x64 / arm64
   - Windows Server 2019/2022/2025
   - **Cannot** target older OS families (those need WHQL/HLK)
7. **Distribution**: leave as "Don't publish to Windows Update" for attestation-only (which is the only option without HLK results)
8. **Submission type**: select "Automatic" — server picks attestation since no HLK package is present
9. Submit
10. Wait for status `Completed` (typically <1 day for attestation)
11. Download the **signed package** — it's a new CAB whose catalog is now signed by Microsoft

### Submission via the Microsoft Hardware API (CI-friendly)

```bash
# 1. Get an Azure AD token for partner-center scope
TOKEN=$(curl -X POST https://login.microsoftonline.com/$TENANT/oauth2/v2.0/token \
  -d "client_id=$CLIENT_ID&client_secret=$CLIENT_SECRET&scope=https://manage.devcenter.microsoft.com/.default&grant_type=client_credentials" \
  | jq -r .access_token)

# 2. Create a product
curl -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  https://manage.devcenter.microsoft.com/v2.0/my/hardware/products \
  -d '{"productName":"MyDriver","testHarness":"attestation",...}'

# 3. Create a submission, upload the CAB to the SAS URL it returns
# 4. Poll for "completed" status, then GET the signed package URL
```

Full reference: https://learn.microsoft.com/en-us/windows-hardware/drivers/dashboard/dashboard-api

### Status flow

```
Initialised
  -> Scanning  (validators run)
    -> InProgress  (catalog being signed)
       -> Completed  (download signed package)
       -> Failed     (read errors, fix, resubmit)
```

Average time for attestation: 30 minutes – 4 hours. SLA is 1 business day.

### Failure messages and fixes

| Failure | Fix |
|---------|-----|
| `INF parsing failed` | Run `infverif /v /w` locally — fix all errors before resubmit |
| `Catalog signature missing` | Run `signtool verify /v /pa MyDriver.cat` — re-sign |
| `DriverVer date is in the future` | Update INF, re-cat, re-sign, re-cab |
| `Architecture mismatch` | INF claims amd64 but binary is arm64 — rebuild |
| `Co-installer not allowed` | Remove `CoInstaller=` line, refactor logic into the driver |
| `EV cert not registered to this company` | Re-verify in Partner Center → Account → Identifiers |

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Uploading the unsigned CAB | Validator rejects in seconds | Always EV-sign the CAB before upload |
| Submission name conflicting with prior version | Partner Center rejects duplicate | Bump the build number |
| Forgot to bump `DriverVer` between submissions | New package can't replace old via PnP | Always increment version monotonically |
| Targeting Win10 RS5 (1809) for attestation | Some old OS families need HLK | Stick to currently-supported clients (Win10 21H2+, Win11) |
| Modified the SYS but not the CAT before submitting | Catalog hash mismatch | Always: change → inf2cat → sign cat → cab → sign cab |
| Submitting from an account without "Driver submission" role | Permission denied | Add the role in Partner Center → Users |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/dashboard/preparing-hardware-submission-package
- https://learn.microsoft.com/en-us/windows-hardware/drivers/dashboard/dashboard-api
- https://learn.microsoft.com/en-us/windows-hardware/drivers/dashboard/manage-hardware-submissions
