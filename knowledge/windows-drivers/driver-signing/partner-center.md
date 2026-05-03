# Windows Drivers - Driver Signing - Hardware Dev Center / Partner Center

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/dashboard/
> **Skill**: dev-suite skill `windows/driver-signing` — see SKILL.md for the always-loaded quick reference.

## What this covers

The Microsoft Partner Center (formerly "Hardware Dev Center") account model: company onboarding, EV cert registration, role-based access, the dashboards you'll use day-to-day, and how this fits with Microsoft Hardware Compatibility Program. Fetch when SKILL.md's "submit via Partner Center" line is opaque — typically because you're setting up an account for the first time.

## Deep dive

### Account hierarchy

```
Partner Center Tenant (one per Azure AD tenant)
  Hardware Program enrollment (one per Tenant)
    Companies / Publishers
      Submissions (drivers, hardware products, Windows Apps)
      Shipping labels (post-submission distribution to Windows Update)
      Test pools (HLK/HCK)
```

A Partner Center *Tenant* is a single Azure AD tenant. Inside it, you enrol once for the Hardware program, and may have multiple "publishers" (companies) under one umbrella account.

### Onboarding flow

1. **Create or use an Azure AD tenant** — `*.onmicrosoft.com` works; you can map a custom domain later.
2. **Sign up** at https://partner.microsoft.com → Membership → Hardware program.
3. Enter your legal company name (must match your D-U-N-S record exactly).
4. Pay the one-time $99 enrolment fee (some regions: free with EV cert verification).
5. **EV cert verification**: download the verification token, sign it with your EV cert using `signtool sign /v /a /n "Your Co" verification-blob.bin`, upload the signed result. This binds the EV cert to your account.
6. After 1–4 weeks, the account is activated.

### What the EV cert binding actually does

Every driver submission's CAB must be signed by the same EV cert (or one issued to the **same legal entity** with matching subject CN). If the cert changes (renewal, vendor change), you must re-verify in Account → Identifiers before the next submission.

Multiple EV certs can be attached — useful if you're transitioning vendors. They're alternatives, not chained.

### Roles and permissions

| Role | Can do |
|------|--------|
| **Owner** | Everything, including managing roles and billing |
| **Manager** | Create submissions, manage products, approve shipping labels |
| **Developer** | Create submissions; cannot approve shipping or manage account |
| **Reporter** | View dashboards, submission history |
| **Business contributor** | Marketing/PR side; no driver permissions |

For CI signing pipelines, create a **Service Principal** in Azure AD, grant it Developer role in Partner Center → Users → Add Azure AD app, and use the SP's client_id/secret to call the Hardware Submission API.

### Dashboards you'll actually use

#### Hardware → Drivers
- **Submissions list**: every driver package you've sent. Filter by status, OS, product.
- **New submission**: upload CAB + optional HLK package, choose targets, submit.
- **Submission detail**: download the signed package, view validation logs, see Microsoft signing timestamp.

#### Hardware → Shipping labels
- After a submission completes, a **shipping label** controls who/where it goes:
  - **Manual download only** (default) — only you have the signed package
  - **Targeted to PnP IDs** — drivers go to specific HWID match via Windows Update
  - **Automatic update** — push to existing devices on Windows Update
- Approval flow: create label → reviewer signs off → goes live in Windows Update channel within ~2 hours.

#### Hardware → Products
- A **product** groups submissions for a single physical device or driver across versions
- Used by the Microsoft Compatible Products list (the "Designed for Windows" badge)

#### Account settings
- **Identifiers**: EV certs, Microsoft Publisher ID (PID), tax info
- **Users**: invite Azure AD users, assign roles
- **Tax & payment**: only matters for paid submissions / Marketplace

### Submission-related metadata the form expects

For each submission you'll be asked:

- **Submission name**: free-text, must be unique within the product
- **Driver category**: Hardware, Software, etc. (for organisation)
- **Target operating systems**: choose from the OS family list
- **Submission method**: Attestation only / HLK / Both
- **Marketing name**: optional, shown to OEMs
- **Hardware IDs (HWIDs)**: PnP IDs your driver claims via INF — extracted automatically
- **Shipping options**: discussed above

### API model (programmatic submissions)

Endpoint: `https://manage.devcenter.microsoft.com/v2.0/my/`

Auth: Azure AD client_credentials flow with scope `https://manage.devcenter.microsoft.com/.default`

Key endpoints:

```
POST /hardware/products                   ; create a product
POST /hardware/products/{pid}/submissions ; create a submission
GET  /hardware/products/{pid}/submissions/{sid}  ; poll status
GET  /hardware/products/{pid}/submissions/{sid}/signedPackageUrl  ; download
POST /hardware/products/{pid}/shippingLabels ; publish to Windows Update
```

Complete reference: https://learn.microsoft.com/en-us/windows-hardware/drivers/dashboard/dashboard-api

### Microsoft Compatibility Program tiers

| Tier | What you submit | What you get |
|------|-----------------|--------------|
| Attestation | EV-signed CAB, no HLK | Signed package, no logo, manual distribution only |
| Compatible | EV-signed CAB + minimal HLK pass | Signed package, listed in Compatible Products |
| Certified | EV-signed CAB + full HLK + product testing | "Designed for Windows" logo, Windows Update push eligible |

Most ISVs stop at Attestation. OEMs and IHVs targeting branded / Windows Update distribution need Compatible or Certified.

### Useful settings to enable on first login

- **Email notifications** for submission status changes (Account → Notifications)
- **Two-factor auth** on all owner accounts (Azure AD level)
- **Audit log retention** — retain for at least 1 year for compliance
- **Submission-level reviewer alerts** — useful for catching failed submissions before customers complain

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Creating multiple Tenants instead of multiple Companies | Each Tenant needs separate EV verification, separate billing | Use one Tenant + multiple Publishers |
| EV cert with subject CN slightly different from registered company | Submission rejected | Re-issue cert with exact matching legal name |
| Submitting from a personal MSA instead of work account | Hardware program requires Azure AD work account | Use Azure AD account |
| Forgot to renew EV cert verification after cert renewal | Submissions silently fail | Re-verify in Identifiers within 30 days of renewal |
| Roles too permissive — every dev is Owner | Audit risk, accidental approvals | Use Developer for engineers, Manager for leads, Owner sparingly |
| Shipping label set to "Automatic update" by mistake | All existing devices auto-update, possibly breaking some | Always default to "Manual download" until QA signs off |
| Service Principal expires unnoticed | CI pipeline breaks silently | Calendar reminder for SP secret rotation |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/dashboard/register-for-the-hardware-program
- https://learn.microsoft.com/en-us/windows-hardware/drivers/dashboard/manage-hardware-submissions
- https://learn.microsoft.com/en-us/windows-hardware/drivers/dashboard/dashboard-api
- https://learn.microsoft.com/en-us/windows-hardware/drivers/dashboard/manage-shipping-labels-for-hardware-submissions
