# Windows Drivers - Driver Signing - Policy Overview

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/install/driver-signing
> **Skill**: dev-suite skill `windows/driver-signing` — see SKILL.md for the always-loaded quick reference.

## What this covers

The Windows kernel driver signing policy across Windows versions: pre-1607 cross-signing, 1607+ attestation requirement, HVCI compatibility constraints, and what loads where (kernel vs UMDF reflector vs WSL2 kernel modules). Fetch when you need to decide whether an existing cross-signed driver still ships on a given Windows version.

## Deep dive

### Policy timeline

| Windows version | Policy |
|-----------------|--------|
| Win7 / 8 / 8.1 | Authenticode signature with chain to a Microsoft-approved CA. SHA-1 acceptable. Cross-signing certs (`.cer` from CA) used. |
| Win10 1507 | Same as Win8.1 — cross-signing still works |
| **Win10 1607** (Aug 2016) | **Inflection point**. New kernel drivers must be signed by Microsoft via the Hardware Dev Center (attestation or WHQL). Existing cross-signed drivers continue to load if signed before July 29, 2015. |
| Win10 1703+ | Cross-sign chain only honoured for drivers with notarised pre-1607 timestamp |
| Win11 22H2+ | HVCI (Memory Integrity) on by default for new SKUs — additional constraints apply |
| Win11 24H2+ | Driver block list enforced (Microsoft Vulnerable Driver Blocklist) |

### What "must be signed by Microsoft" means

The signature chain must terminate at one of these Microsoft roots:
- "Microsoft Code Verification Root"
- "Microsoft Windows Production PCA 2011"
- "Microsoft Hardware Dev Center 2014" (attestation chain)

You don't get there by buying a cert — you get there by **submitting** your driver package (signed with your EV cert) to Partner Center, which then re-signs the catalog with a Microsoft-issued cert. The output is what you ship.

### HVCI (Hypervisor-protected Code Integrity)

HVCI is on by default on:
- Modern OEM Win11 systems
- Windows Server 2019+ in shielded VM mode
- Any system where "Memory Integrity" is on in Windows Security

Drivers running under HVCI must:
1. Have NX (no-execute) data sections — no executable pool
2. Not use self-modifying code
3. Not allocate executable memory at runtime via `MmAllocatePagesForMdl(...)` and execute it
4. Not patch the IAT or call APIs that bypass CFG
5. Be properly Hardware Dev Center signed

If a driver fails HVCI checks, it is silently rejected at load time on HVCI systems even though it loads fine on non-HVCI systems. Test with HVCI explicitly enabled in a VM.

### What loads where

| Component | Signing requirement |
|-----------|--------------------|
| KMDF / WDM `.sys` | Hardware Dev Center signed (attestation or WHQL) |
| UMDF 2 reflector binaries (`WUDFRd.sys` etc.) | Microsoft signed (system-supplied) |
| UMDF 2 driver `.dll` (yours) | Same as KMDF — kernel reflector validates |
| Windows service binaries | Authenticode (any trusted CA) — not the same policy as drivers |
| WSL2 kernel modules | WSL2 uses an upstream Linux kernel; Linux module signing applies, **not** Windows |
| Filter drivers / minifilters | KMDF rules — must be Hardware Dev Center signed |
| Boot-start drivers | Same as kernel + must pass Secure Boot validation; some early boot drivers also need Microsoft Boot Loader signature |
| Hyper-V root partition drivers | Same as kernel |
| Hyper-V child / paravirtual drivers | Sometimes shipped via Windows Update with separate signing |

### Test signing exception

```cmd
bcdedit /set testsigning on
shutdown /r /t 0
```

When test signing is on, the kernel accepts any chain — including self-signed. The desktop shows a watermark in the bottom right. This is for development only; never instruct end users to enable test signing.

### Disable Driver Signature Enforcement (DSE)

Pressing F8 → "Disable Driver Signature Enforcement" at boot time loads the driver once for that boot. Useful for one-off recovery. Not a long-term option — the next boot re-enables enforcement.

### Boot configuration variants

```cmd
bcdedit /set nointegritychecks off       :: enforce integrity (default)
bcdedit /set nointegritychecks on        :: disable enforcement (dev/test only)
bcdedit /set testsigning on              :: accept any test-signed cert
bcdedit /set loadoptions DDISABLE_INTEGRITY_CHECKS :: legacy form
```

Secure Boot (when enabled in firmware) ignores `nointegritychecks` — the only way to load unsigned kernel code is to disable Secure Boot in BIOS.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Targeting Win10 RTM but using a Hardware Dev Center submission | Ms-issued sig only honoured 1607+ | Either drop pre-1607 support or dual-sign with cross-cert + attestation |
| Driver loads on dev box but not on customer's Win11 | HVCI rejected at load — silent | Test in a VM with Memory Integrity ON |
| Self-modifying tracelog code | Fails HVCI | Refactor to use ETW instead |
| Shipping with cross-sign cert in 2026 | Cert chain not trusted on new Windows | Use attestation signing |
| Test-sign in production by accident | Watermark on desktop, security risk | `bcdedit /set testsigning off` and reboot before shipping |
| Driver listed in Vulnerable Driver Blocklist | Microsoft permanently blocks it | Get a new code-signing cert and re-attest with security fixes |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/install/kernel-mode-code-signing-policy--windows-vista-and-later-
- https://learn.microsoft.com/en-us/windows-hardware/drivers/bringup/device-guard-and-credential-guard
- https://learn.microsoft.com/en-us/windows/security/threat-protection/microsoft-recommended-driver-block-rules
