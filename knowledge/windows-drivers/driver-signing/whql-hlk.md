# Windows Drivers - Driver Signing - WHQL / HLK

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/test/hlk/
> **Skill**: dev-suite skill `windows/driver-signing` — see SKILL.md for the always-loaded quick reference.

## What this covers

The Hardware Lab Kit (HLK) end-to-end: Studio + Controller setup, playlist selection, running tests against your device, generating the `.hlkx` package, and the difference between running it in your own lab vs paying a partner. Fetch when SKILL.md's one-paragraph WHQL summary is not enough.

## Deep dive

### When you need WHQL vs attestation

| Need | Track |
|------|-------|
| Driver loads on Win10/11, no UI label | Attestation only |
| "Certified for Windows" label / branded | WHQL/HLK |
| Distribution via Windows Update | WHQL/HLK |
| OEM pre-install requirement | Often WHQL/HLK (depends on OEM) |
| Device class with mandatory tests (e.g. mobile broadband, HID-FIDO2) | WHQL/HLK |
| Server logo (Windows Server 2022 etc.) | WHQL/HLK with the server playlist |

### Architecture: Controller + Studio + Clients

```
HLK Controller (one machine)
  - SQL Server Express + IIS
  - Hosts the test database, scheduler, log aggregator
  - Has the HLK Studio installed for management

HLK Studio (UI on Controller, or remote)
  - Browse machine pools
  - Select playlists, schedule runs, review logs
  - Generate .hlkx package for submission

HLK Client (one or more test machines)
  - Where your driver + device under test live
  - Runs Test Authoring and Execution Framework (TAEF) test suites
  - Reports results back to Controller
```

All three components are versioned; **the HLK version must match the OS version** of the client (HLK 22H2 for Windows 11 22H2, HLK Server 2025 for Windows Server 2025). Mixing versions silently produces invalid logs.

### Setup (Controller side)

1. Provision a machine with the matching OS (Win Server 2019+ recommended)
2. Install **HLK Controller + Studio** from the HLK ISO (`HLKSetup.exe /studiocontroller`)
3. Open inbound HTTP/HTTPS for client communication
4. Open HLK Studio → File → Create new project

### Setup (Client side)

1. Install matching OS on the device under test machine
2. From the Controller's `\\<controller>\HLKInstall\Client\` share, run `setup.exe`
3. Reboot — client appears in HLK Studio under Machine Pools as a new machine
4. Set the machine **Role** to "Client (Test)"
5. Move it from Default pool to a project-specific pool

### Running tests

1. In Studio: **Selection** tab → choose the device or driver from the pool
2. Studio auto-suggests the **playlist** based on device class (HID, Display, Net, Storage, etc.)
3. Add or remove tests with the playlist editor (XML at `<HLK>\Tests\<arch>\NTTEST\<area>\*.wtl`)
4. **Schedule** tab → submit, monitor progress
5. **Results** tab → green = pass, red = fail. Fail logs include `WTTLog.log`, `Crash.dmp`, and `EtlLogs\*.etl`
6. After all selected tests pass, **Package** tab → "Create Package"
7. Studio bundles `.hlkx` (= renamed `.zip`) with all logs, playlists, supplemental folders, and a manifest

### Playlists — how Microsoft picks the test set

Microsoft publishes per-OS-version compat playlists at:
```
https://docs.microsoft.com/en-us/windows-hardware/test/hlk/windows-hardware-lab-kit#filters-and-playlists
```

Download the `.xml` playlist for your release (e.g. `HLK Version 24H2 CompatPlaylist x64 ARM64.xml`). Studio applies it as a filter to the test selection — only tests in the playlist are required for that submission.

A playlist may also include **filters** that mark certain failures as acceptable (e.g. "this test requires a feature your device doesn't claim"). Apply filters in Studio → Selection → Manage filters.

### Lab vs partner

| Path | Cost | Time | Tradeoffs |
|------|------|------|-----------|
| **In-house lab** | Hardware + license-free HLK | Days–weeks of engineering | Full control, repeatable, expensive bench setup for some classes (mobile broadband needs cellular gear) |
| **Authorized test partner** | $$$ per submission | Days | They have the gear (anechoic chamber, USB analysers, multi-monitor rigs) — no upfront investment |

Partners published at: https://partner.microsoft.com/en-us/dashboard/hardware/AuthorizedTestlabs

### Submitting the .hlkx

1. Build the driver package (.cab, EV-signed) as for attestation
2. In Partner Center → New submission, upload **both** the package CAB **and** the `.hlkx`
3. Choose submission type "Hardware lab kit" (or "Both" if you also want attestation)
4. Microsoft validates HLK results and signs the catalog
5. After completion, optionally request "Publish to Windows Update" with the signed package

### Re-running tests after a driver fix

You don't need to re-run the entire playlist; HLK Studio supports **deltas**:

1. Open the existing project
2. Selection → only re-select the failed tests
3. Schedule → re-run
4. Package → Studio merges new + old results (validator on the server checks that retest delta is within policy)

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| HLK and OS version mismatch | Silent log corruption, submission rejected | Match HLK version to OS exactly (24H2 HLK on 24H2 client) |
| Forgot to install matching .NET / WMF on Controller | HLK Studio crashes on launch | Follow the prerequisites list precisely |
| Test pool empty / clients offline | Schedule never starts | Verify machine state in Studio = "Ready" |
| Packaging while tests still running | Incomplete .hlkx, validator rejects | Wait for all tests to reach a final state |
| HLK firewall blocked between Controller and Client | Tests appear stuck at "queued" | Open ports per the Setup doc; usually 80, 443, 1433 |
| Re-running the entire playlist for one fix | Wastes days of lab time | Use delta re-runs |
| Submitting without supplemental folder | Some tests need extra binaries — submission incomplete | Studio adds these automatically; don't manually edit the .hlkx |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/test/hlk/getstarted/install-hlk
- https://learn.microsoft.com/en-us/windows-hardware/test/hlk/user/managing-machine-pools
- https://learn.microsoft.com/en-us/windows-hardware/test/hlk/user/creating-a-package
- https://partner.microsoft.com/en-us/dashboard/hardware/AuthorizedTestlabs
