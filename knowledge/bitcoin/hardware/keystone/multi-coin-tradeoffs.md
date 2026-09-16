# Keystone 3 Pro Multi-Coin Trade-offs - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/keystone`.
> Canonical source: https://guide.keyst.one/ (the Keystone 3 Pro
> doc set). The older `docs.keyst.one` host redirects to
> support.keyst.one, which is now the legacy Keystone Pro / Cobo Vault
> documentation, as of September 2026.
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/keystone/SKILL.md

## Concept

Keystone 3 Pro (formerly Cobo Vault; the product name is "Keystone 3
Pro", not "Keystone Pro 3") supports Bitcoin **plus** Ethereum, Cosmos,
Solana, Tron, Polkadot, and many more chains. This convenience
comes with trade-offs vs Bitcoin-only devices like Coldcard or Passport:
the firmware has a larger trusted code base, parsing logic for multiple
chains and signature schemes, and the attack surface is correspondingly
larger.

For a Bitcoin-focused user, the question is: do the Keystone-specific
features (4" touchscreen, fingerprint, three secure element chips,
QR-only airgap, BIP-39 + Shamir / SLIP-39) outweigh the multi-coin
overhead? Note that the multi-coin overhead is optional: Keystone ships
a Bitcoin-only firmware edition (see below).

## Walkthrough

### Keystone signing model

Keystone is QR-only; USB is for charging and firmware updates only.
For Bitcoin, the flow is:

```
Coordinator (Sparrow / Specter / Keystone Nexus)            Keystone
1. Build PSBT, display animated UR crypto-psbt
                                                            2. Tap "Scan QR"
                                                            3. Camera reads
                                                            4. Touchscreen shows tx details
                                                            5. Fingerprint to confirm
                                                            6. Sign internally
                                                            7. Display animated QR
8. Coordinator scans signed PSBT, broadcasts
```

For altcoins, the analogous flow uses chain-specific encodings (e.g.,
`eth-sign-request` UR for Ethereum, raw transaction CBOR for Cosmos).

### Three secure elements

Keystone's distinguishing hardware feature: the 3 Pro carries **three**
secure element chips rather than one, with seed material and fingerprint
data held in separate chips, so one compromised chip does not hand an
attacker the key. The chips are not a removable module - the device is
PCI-grade anti-tamper ("PCI Level Self-Destruct" in Keystone's own 3 Pro
feature list) and detects disassembly, wiping all information including
the private keys.

```
3 SE chips, split roles   = seed in one, fingerprint data in another
Firmware update paths     = microSD image or WebUSB (USB is charge +
                            update only; no USB signing path)
Disassembly detected      = self-destruct wipes keys, device bricked
                            (funds recoverable from the seed phrase)
```

There is no user-swappable SE module on the 3 Pro: you cannot pull the
secure element out for separate cold storage, and you cannot move it to
a spare device. Opening the case is destructive by design.

### Multi-coin overhead

As of September 2026 Keystone publishes three firmware editions for the
same 3 Pro hardware, cross-upgradable at or above the installed
version:

| Edition | Latest | Scope |
|---------|--------|-------|
| Multi-coin | 3.0.8 (Sep 2026) | BTC + 5,500+ assets |
| Bitcoin-only | 3.0.4-BTC (Aug 2026) | Bitcoin-related functions only |
| Cypherpunk | 3.0.4-CYPHERPUNK (Aug 2026) | BTC, ZEC, XMR |

Devices ship on multi-coin, so a Bitcoin-only user flashes the BTC
build after unboxing.

| Aspect | Coldcard Mk4 | Keystone 3 Pro |
|--------|--------------|----------------|
| Firmware code size | ~600 KB | ~3-5 MB |
| Supported chains | BTC only | 30+ |
| Open-source | Partial | Partial (firmware open, hardware mixed) |
| Attack surface | Minimal | Larger (parser bugs per chain) |
| UI complexity | Buttons + small screen | Full touch UI |

## Worked example

Bitcoin-only user setting up Keystone:

1. Initialize device, generate 24-word BIP-39 (or 12, or import).
2. Optionally add Shamir Backup (SLIP-39) - Keystone supports.
3. Enable fingerprint for confirmations.
4. Flash the Bitcoin-only firmware edition (3.0.4-BTC as of August
   2026) from https://keyst.one/firmware to drop the altcoin parsers
   and reduce surface. The BTC build keeps testnet, dice entropy,
   cross-vendor multisig and Shamir Backup.
5. Pair with Sparrow:
   ```
   Sparrow -> Add Wallet -> From Hardware -> Keystone
   Keystone -> Show QR -> animated UR crypto-account
   Sparrow scans, imports xpub
   ```
6. Send / receive via QR PSBT flow.

For multisig, Keystone supports up to 15 cosigners and registers the
descriptor like other devices.

## Common pitfalls

- **Multi-coin firmware bloat**: even if you never use altcoins, the
  parsers are present on the stock build. A bug in any chain's
  transaction parser is a potential vector. The Bitcoin-only edition is
  a maintained, separately versioned release line (not an occasional
  variant), so prefer it.
- **Self-destruct lifespan is undocumented for the 3 Pro**: the
  often-quoted line that the lifespan of the self-destruct components
  "may not exceed 2 years" comes from the legacy support page
  "Self-Destruct Mechanism (Pro-only)" (support.keyst.one, last updated
  April 2022), whose body documents the previous-generation Keystone
  Pro. No Keystone 3 Pro source - shop feature list, btc-only page,
  guide.keyst.one or the firmware README - states a component lifespan,
  as of September 2026. Treat anti-tamper as a deterrent of unknown
  durability: do not carry the 2-year figure over to the 3 Pro, and do
  not assume a permanent guarantee either.
- **Fingerprint as second factor**: convenient but biometrics are not
  high-security against a coercive attacker (forced fingerprint).
  PIN remains primary.
- **Shamir on Keystone vs Trezor**: SLIP-39 implementations should
  interop, but cross-vendor recovery is rare and not always tested.
  Keep your recovery process within the same vendor unless you've
  verified.
- **Update channel**: firmware arrives either as a `keystone3.bin`
  image on microSD or over WebUSB from https://keyst.one/firmware, and
  the device checks the vendor signature either way. The user-facing
  check is a **SHA-256 checksum**, not a GPG signature, and the release
  notes publish two labelled hashes that are not interchangeable. The
  "Firmware SHA-256 Checksum" covers the downloaded `keystone3.bin` -
  hash that file yourself before flashing. The "Source Code SHA-256
  Checksum" is the one to compare against the device, under Device
  Settings -> About -> Device info -> Firmware Version -> Verify Source
  Code -> Show Checksum, because the device reports the checksum of
  `mh1903.bin`, the uncompressed unsigned source build that
  `keystone3.bin` is derived from. Comparing one label against the
  other always mismatches.

## References

- Keystone 3 Pro docs: https://guide.keyst.one/
- Firmware checksum verification: https://guide.keyst.one/docs/verify-checksum
- Legacy Keystone Pro / Cobo Vault docs: https://support.keyst.one/
- Keystone firmware downloads: https://keyst.one/firmware
- Keystone 3 firmware source: https://github.com/KeystoneHQ/keystone3-firmware
- Keystone Nexus companion app: https://keyst.one/nexus
