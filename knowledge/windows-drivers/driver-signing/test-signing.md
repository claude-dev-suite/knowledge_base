# Windows Drivers - Driver Signing - Test Signing

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/install/installing-an-unsigned-driver-during-development-and-test
> **Skill**: dev-suite skill `windows/driver-signing` — see SKILL.md for the always-loaded quick reference.

## What this covers

The full test-signing pipeline: enabling test mode, generating a self-signed cert with `MakeCert` or modern PowerShell, putting the cert into the right stores on the target, signing both the SYS and CAT, the watermark behaviour, and the most common reasons a test-signed driver still won't load. Fetch when SKILL.md's brief snippet doesn't explain why your test-signed driver is rejected.

## Deep dive

### Enable test signing on the target

```cmd
:: Admin cmd
bcdedit /set testsigning on
bcdedit /set nointegritychecks off       :: keep integrity check ON; only relax cert chain
shutdown /r /t 0
```

After reboot, the bottom-right of the desktop shows `Test Mode  Windows 10  Build XXXX`. The watermark cannot be removed without disabling test signing.

If Secure Boot is on in firmware, `testsigning on` is **silently rejected** at the next boot — you must disable Secure Boot in BIOS first.

### Generate a self-signed test cert

#### Modern: PowerShell

```powershell
$cert = New-SelfSignedCertificate `
    -Type CodeSigningCert `
    -Subject "CN=WDKTestCert" `
    -KeyExportPolicy Exportable `
    -CertStoreLocation Cert:\CurrentUser\My `
    -KeyUsage DigitalSignature `
    -KeyAlgorithm RSA -KeyLength 2048 `
    -NotAfter (Get-Date).AddYears(5) `
    -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.3")  # Code Signing EKU

# Export public cert and PFX with private key
Export-Certificate -Cert $cert -FilePath C:\test\WDKTest.cer
Export-PfxCertificate -Cert $cert -FilePath C:\test\WDKTest.pfx `
    -Password (ConvertTo-SecureString -String "test123" -Force -AsPlainText)
```

#### Legacy: MakeCert (still in WDK)

```cmd
makecert -r -pe -ss PrivateCertStore -n "CN=WDKTestCert" -eku 1.3.6.1.5.5.7.3.3 ^
    C:\test\WDKTest.cer
pvk2pfx -pvk WDKTest.pvk -spc WDKTest.cer -pfx WDKTest.pfx -po test123
```

### Install the cert in the right stores

The target machine must trust your test cert. The kernel checks **Trusted Root** and **Trusted Publishers**:

```cmd
:: Run on the TARGET as admin
certmgr.exe /add C:\test\WDKTest.cer /s /r localMachine root
certmgr.exe /add C:\test\WDKTest.cer /s /r localMachine trustedpublisher

:: Verify
certutil -store -enterprise root | findstr WDKTest
certutil -store -enterprise trustedpublisher | findstr WDKTest
```

PowerShell equivalent:

```powershell
Import-Certificate -FilePath C:\test\WDKTest.cer -CertStoreLocation Cert:\LocalMachine\Root
Import-Certificate -FilePath C:\test\WDKTest.cer -CertStoreLocation Cert:\LocalMachine\TrustedPublisher
```

If you skip this step, the driver SYS verifies as signed but PnP refuses to install it (treats publisher as untrusted, prompts in Action Center).

### Sign the SYS and the CAT

```cmd
:: Sign the SYS
signtool sign /v /f C:\test\WDKTest.pfx /p test123 ^
    /tr http://timestamp.digicert.com /td sha256 /fd sha256 ^
    C:\drvbuild\MyDriver.sys

:: Generate the catalog from the INF (do this on the build box)
inf2cat /driver:C:\drvbuild /os:10_X64

:: Sign the catalog with the same cert
signtool sign /v /f C:\test\WDKTest.pfx /p test123 ^
    /tr http://timestamp.digicert.com /td sha256 /fd sha256 ^
    C:\drvbuild\MyDriver.cat

:: Verify (kernel policy)
signtool verify /v /kp /a C:\drvbuild\MyDriver.sys
```

### Install on target

```cmd
:: Stage the driver into the DriverStore
pnputil /add-driver C:\drvbuild\MyDriver.inf /install

:: Or, for a non-PnP service driver
sc create MyDriver type= kernel binPath= C:\Windows\System32\drivers\MyDriver.sys start= demand
sc start MyDriver
```

### Troubleshooting load failures

```cmd
:: 1. Verify test mode is on
bcdedit /enum {current} | findstr testsigning

:: 2. Verify integrity checks are not disabled by some other policy
fltmc instances

:: 3. Look at SetupAPI logs (PnP install errors)
notepad %WINDIR%\INF\setupapi.dev.log

:: 4. Look at CodeIntegrity events
wevtutil qe Microsoft-Windows-CodeIntegrity/Operational /c:50 /rd:true /f:text
```

`CodeIntegrity` event ID 3033 = "Code integrity determined that a process attempted to load a driver that did not meet the Microsoft signing level" — usually means cert not in Trusted Publishers.

### When test-sign isn't enough: full DSE bypass

Some scenarios (boot-start drivers on a system with a particular policy) ignore test signing. Last resort:

```cmd
bcdedit /set nointegritychecks on
:: Reboot — accepts entirely unsigned drivers, very dangerous
```

Never on a production machine. Some Windows updates re-enable integrity checks silently.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Test cert only in CurrentUser store | Kernel checks LocalMachine | Always install with `certmgr /s /r localMachine` |
| Forgot to add to Trusted Publishers | Driver signs OK but PnP install fails | Add to both Root and Trusted Publishers |
| Signed SYS with SHA1 | `signtool verify /v /kp` fails on modern Windows | Use `/fd sha256` |
| Signed SYS but not CAT | INF install fails ("hash for file not found") | Always sign both, in this order: SYS → inf2cat → CAT |
| Modified SYS after generating CAT | Catalog hash mismatch | Re-run inf2cat then re-sign CAT |
| Test-sign on a Secure Boot system | Silent failure — testsigning ignored | Disable Secure Boot in firmware |
| Test cert expired | Existing installs continue but new ones fail | Generate a new cert with longer expiry, re-sign, reinstall |
| Forgot to disable test mode before shipping | Watermark visible to end users | `bcdedit /set testsigning off` and reboot before imaging |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/install/the-testsigning-boot-configuration-option
- https://learn.microsoft.com/en-us/windows-hardware/drivers/install/installing-test-certificates
- https://learn.microsoft.com/en-us/windows-hardware/drivers/install/signing-a-driver
