# Windows Drivers - Driver Signing - signtool Reference

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/signtool
> **Skill**: dev-suite skill `windows/driver-signing` — see SKILL.md for the always-loaded quick reference.

## What this covers

`signtool.exe` full reference for `sign`, `verify`, `timestamp`, and dual-sign workflows. SKILL.md covers the basic sign + verify; this file covers cert selection, RFC 3161 vs Authenticode timestamps, dual-signing for legacy targets, sign with `/dlib`, and the full set of verify policies.

## Deep dive

### signtool sign — flag reference

```cmd
signtool sign [options] <file>...
```

| Flag | Meaning |
|------|---------|
| `/v` | Verbose |
| `/debug` | Print every cert chain considered |
| `/a` | Auto-select best cert from current user's My store |
| `/n <subject>` | Select cert by subject CN |
| `/sha1 <thumb>` | Select cert by SHA-1 thumbprint (most precise) |
| `/i <issuer>` | Filter by issuer |
| `/s <store>` | Cert store (default `My`) |
| `/sm` | Use machine store instead of user store |
| `/f <pfx>` | Sign with a PFX file |
| `/p <pwd>` | PFX password |
| `/csp <name>` | Use a specific Cryptographic Service Provider |
| `/kc <container>` | Key container name (with /csp) |
| `/dlib <dll>` | Delegate to a custom signing DLL (cloud HSM) |
| `/dmdf <file>` | Metadata file passed to /dlib |
| `/fd sha256` | File digest algorithm — **always sha256** for kernel |
| `/td sha256` | Timestamp digest algorithm — **always sha256** |
| `/tr <url>` | RFC 3161 timestamp URL — **prefer this over /t** |
| `/t <url>` | Authenticode timestamp URL (legacy) |
| `/as` | Append signature (for dual-signing) |
| `/ph` | Add page hashes (HVCI requirement for SYS) |
| `/nph` | Suppress page hashes |
| `/ac <cer>` | Add an additional intermediate certificate |
| `/q` | Quiet |

### Recommended canonical sign command

```cmd
signtool sign /v /sha1 ABCDEF0123456789ABCDEF0123456789ABCDEF01 ^
    /tr http://timestamp.digicert.com /td sha256 ^
    /fd sha256 /ph ^
    MyDriver.sys
```

- `/sha1` for deterministic cert selection (no surprises if you have multiple certs)
- `/tr` (RFC 3161) over `/t` (Authenticode) — RFC 3161 is the modern standard
- `/td sha256` and `/fd sha256` — never SHA-1 for new drivers
- `/ph` ensures the catalog signing later embeds page hashes for HVCI

### Timestamp servers

| Provider | RFC 3161 URL |
|----------|--------------|
| DigiCert | `http://timestamp.digicert.com` |
| Sectigo | `http://timestamp.sectigo.com` |
| GlobalSign | `http://timestamp.globalsign.com/tsa/r6advanced1` |
| Entrust | `http://timestamp.entrust.net/TSS/RFC3161sha2TS` |
| SSL.com | `http://ts.ssl.com` |

Timestamping makes the signature outlive the cert: kernel verifies that the cert was valid at the **timestamp** moment, not at load moment. Without it, every signed driver becomes invalid the day the cert expires.

### Dual signing (SHA-1 + SHA-256)

Only needed for drivers that must load on Windows 7 / Server 2008 R2:

```cmd
:: First signature (SHA-1, with cross-sign chain for Win7)
signtool sign /v /sha1 <SHA1_thumbprint> ^
    /ac C:\certs\MSCV-VSClass3.cer ^
    /tr http://timestamp.digicert.com /td sha1 /fd sha1 ^
    MyDriver.sys

:: Append second signature (SHA-256, for Win10+)
signtool sign /v /as /sha1 <SHA256_thumbprint> ^
    /tr http://timestamp.digicert.com /td sha256 /fd sha256 /ph ^
    MyDriver.sys
```

The `/as` flag appends to the existing PE signature directory. Verify both:

```cmd
signtool verify /v /pa /all MyDriver.sys
```

Most new projects target Win10+ only and skip this entirely.

### Cloud-HSM signing via /dlib

```cmd
:: DigiCert KeyLocker example
signtool sign /v /sha1 <thumbprint> ^
    /dlib smkp.dll /dmdf C:\smkp.dmdf ^
    /tr http://timestamp.digicert.com /td sha256 /fd sha256 /ph ^
    MyDriver.sys

:: smkp.dmdf points to vendor config:
:: smkp_url=https://clientauth.one.digicert.com/api/v1/...
:: smkp_apikey=xxxxxxxx
```

`/dlib` loads the vendor's signing DLL into signtool; the DLL forwards the hash-to-be-signed to the cloud HSM and returns the signature. The local machine never sees the private key.

### signtool verify — policy modes

```cmd
:: Authenticode default policy (any trusted CA)
signtool verify /v /pa MyDriver.sys

:: All signatures, not just the primary
signtool verify /v /pa /all MyDriver.sys

:: Kernel-mode driver policy — the strictest, simulates kernel load
signtool verify /v /kp /a MyDriver.sys

:: Microsoft Hardware Compatibility policy (post-MS-signing)
signtool verify /v /ms /a MyDriver.sys

:: Verify a catalog
signtool verify /v /pa MyDriver.cat

:: Verify a CAB (signed package)
signtool verify /v /pa MyDriverPackage.cab
```

`/kp` is the canary — if it fails, the driver will not load on production Windows even if `/pa` passes. Always run `/kp` before shipping.

### Verifying a timestamp

```cmd
:: Output includes a "Timestamp Verified" line if the timestamp is good
signtool verify /v /pa /tw MyDriver.sys
```

If the timestamp is missing, the line is absent — re-sign immediately.

### signtool timestamp (re-timestamp existing signature)

If you signed without `/tr` originally:

```cmd
signtool timestamp /v /tr http://timestamp.digicert.com /td sha256 MyDriver.sys
```

This adds an RFC 3161 timestamp without re-signing. Useful when you discover a missing timestamp on an already-shipped binary and want to re-publish without rebuilding.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Used `/t` instead of `/tr` | Authenticode timestamp uses SHA-1 — kernel rejects | Use `/tr <RFC3161-url>` |
| Forgot `/fd sha256` | Defaults to SHA-1, kernel rejects | Always specify `/fd sha256` |
| Forgot `/ph` on a SYS | No page hashes — HVCI fails to load | Always `/ph` for kernel binaries |
| Multiple certs in store, no `/sha1` | Picks wrong cert randomly | Always select by thumbprint |
| Sign DLL fails with "no certificates were found" | Cert is in machine store but `/sm` missing | Add `/sm` |
| Signed but `signtool verify /v /kp` fails | Cert not from Microsoft chain | EV cert is fine for SYS dev signing; Microsoft chain only after attestation |
| Used Authenticode timestamp URL with `/tr` | Server doesn't speak RFC 3161 — error | Use the RFC 3161 URLs in the table above |
| Re-signed without removing old signature on dual-sign mistake | Two SHA-256 sigs, neither for SHA-1 — Win7 still broken | Remove existing sigs (`signtool remove /s file.sys`) and start over |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/install/dual-signing-driver-files-for-multiple-versions-of-windows
- https://learn.microsoft.com/en-us/dotnet/framework/tools/signtool-exe
- https://learn.microsoft.com/en-us/windows-hardware/drivers/install/timestamping-and-signature-verification
