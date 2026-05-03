# Windows Drivers - Driver Signing - EV Code-Signing Certificates

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/dashboard/get-a-code-signing-certificate
> **Skill**: dev-suite skill `windows/driver-signing` — see SKILL.md for the always-loaded quick reference.

## What this covers

EV (Extended Validation) vs OV (Organization Validation) certificates, why a hardware token is mandatory, the current vendor list with rough pricing, the renewal process, and how to use a token from CI without it sitting on a developer's desk. Fetch when SKILL.md's "use an EV cert" line is not enough — usually because you're choosing a vendor or trying to automate signing.

## Deep dive

### Cert classes

| Class | Validation | Storage | Driver signing accepted? |
|-------|-----------|---------|--------------------------|
| DV (Domain Validation) | Email/DNS only | Software | No — never |
| OV (Organization Validation) | Phone + business records | Software | **No** for kernel drivers since 2016; OK for Authenticode app signing |
| **EV (Extended Validation)** | Phone + legal entity verification + manual review | **Hardware token (FIPS 140-2 Level 2+)** | **Yes** — required for Hardware Dev Center submission |
| Microsoft-issued (post-attestation) | N/A — issued by MS after attestation | In your driver catalog | **Yes** — the only thing the kernel actually trusts |

### Why hardware token

Kernel drivers can take over the system. Microsoft requires EV because:
1. Identity is verified to a higher legal standard
2. The private key cannot be exfiltrated by malware on the dev box
3. Each signing operation requires the token + PIN

Tokens in common use:
- SafeNet eToken 5110 CC
- YubiKey 5 FIPS series
- Vendor-provided HSM USB sticks (DigiCert KeyLocker, Sectigo eSigner)

### Current vendors (2026)

Microsoft maintains a list of CAs whose EV roots are pre-trusted. Active ones:

| Vendor | EV cert (3 yr typical) | HSM/cloud signing option |
|--------|------------------------|--------------------------|
| DigiCert | $400-700/yr | KeyLocker (cloud HSM) |
| Sectigo (formerly Comodo) | $300-500/yr | eSigner (cloud) |
| GlobalSign | $400-600/yr | Atlas (cloud HSM) |
| SSL.com | $300-500/yr | eSigner |
| Entrust | $500-800/yr | Cert Hub |
| Certum | $200-400/yr | SimplySign (cloud) |

Cloud-HSM options solve the "token on a desk" problem — signing is an API call, the key never leaves a FIPS-certified hosted HSM. Required for unattended CI signing in any reasonable team setup.

### Provisioning flow

1. Choose a CA. Place an order.
2. CA verifies legal entity: incorporation docs, D-U-N-S number, phone callback to a publicly listed number for your company.
3. CA ships the hardware token (or onboards you onto the cloud HSM).
4. You install the token's middleware (SafeNet Authentication Client, YubiKey Manager, vendor's CSP).
5. Generate a CSR on the token (key never leaves the device).
6. CA signs the CSR; the cert is loaded onto the token.
7. Microsoft Partner Center registration: sign a test file with the cert and upload — establishes the cert ↔ company link in the dashboard.

End-to-end: 1–4 weeks, mostly waiting for legal verification.

### Using a token with signtool

```cmd
:: List certs on the token
certutil -store -user My

:: Sign by SHA-1 thumbprint (most reliable when multiple certs are present)
signtool sign /v /sha1 ABCDEF0123456789ABCDEF0123456789ABCDEF01 ^
    /tr http://timestamp.digicert.com /td sha256 /fd sha256 ^
    MyDriver.sys

:: Or sign via the CSP / CNG provider directly
signtool sign /v /n "Your Company" /csp "DigiCert Hardware Authentication Provider" ^
    /kc "[{{<token-pin>}}]=<container>" /tr http://timestamp.digicert.com /td sha256 ^
    /fd sha256 MyDriver.sys
```

### Cloud-HSM signing example (DigiCert KeyLocker)

```cmd
:: Use the smctl CLI shipped by DigiCert
smctl windows certsync                    :: install certs into Windows store
signtool sign /v /sha1 <cert-thumbprint> ^
    /tr http://timestamp.digicert.com /td sha256 /fd sha256 ^
    /dlib smkp.dll /dmdf %SM_CONFIG%      ^
    MyDriver.sys
```

The `/dlib smkp.dll` flag tells signtool to delegate the actual signing to the vendor's CNG provider, which proxies to the cloud HSM via authenticated API.

### Renewal

EV certs are issued for 1–3 years. Renewal process:
1. Re-verify legal entity (CA may waive if recent)
2. Generate a new CSR — the new key replaces the old on the token (or coexists)
3. Re-register with Partner Center using the new cert
4. Drivers signed with the **timestamped** old cert continue to be valid (kernel accepts any signature where the timestamp is within the cert's validity period at signing time)

### EV cert and Partner Center linkage

The first time you submit a driver, Partner Center checks that the EV cert's subject matches your registered company. After that, **all submissions must use the same legal entity name** — even cosmetic changes ("Inc." vs "Inc") block submissions until you update the company record.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Bought OV cert thinking it works for drivers | Partner Center rejects on submission | Order EV — ~2 week lead time |
| Token on developer's laptop, signing breaks when they're out | Single point of failure | Move to cloud HSM, share access via roles |
| Forgot to timestamp | Sig invalid the day the cert expires | Always pass `/tr` and `/td sha256` |
| Token PIN locked after retries | Token hardware-locked, need to re-enrol | Use the vendor portal to unlock — keep PIN in a vault |
| Hard-coded thumbprint in build script | Renewal breaks every build | Read thumbprint from a versioned config file |
| Two different cert names on different submissions | Partner Center treats them as separate companies | Maintain one canonical company name |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/install/cross-certificates-for-kernel-mode-code-signing
- https://learn.microsoft.com/en-us/windows-hardware/drivers/dashboard/cross-cert-for-kernel-mode-drivers
- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/signtool
