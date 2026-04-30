# Keystone Pro 3 Multi-Coin Trade-offs - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/keystone`.
> Canonical source: https://docs.keyst.one/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/keystone/SKILL.md

## Concept

Keystone Pro 3 (formerly Cobo Vault) supports Bitcoin **plus** Ethereum,
Cosmos, Solana, Tron, Polkadot, and many more chains. This convenience
comes with trade-offs vs Bitcoin-only devices like Coldcard or Passport:
the firmware has a larger trusted code base, parsing logic for multiple
chains and signature schemes, and the attack surface is correspondingly
larger.

For a Bitcoin-focused user, the question is: do the Keystone-specific
features (4" touchscreen, fingerprint, removable SE module, QR-only
airgap, BIP-39 + Shamir / SLIP-39) outweigh the multi-coin overhead?

## Walkthrough

### Keystone signing model

Keystone is QR-only; USB is for charging and firmware updates only.
For Bitcoin, the flow is:

```
Coordinator (Sparrow / Specter / Keystone Companion)        Keystone
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

### Removable SE

Keystone's distinguishing feature: the SE is on a swappable module. You
can:

- Remove the SE for cold storage between sessions (just the SE has the
  encrypted seed; the main board has no key material).
- Move the SE to a backup Keystone device.
- Treat the SE as the **single secret object** to physically secure.

```
Main device + SE inserted = full functionality
Main device alone         = no signing capability (SE is the root of trust)
SE alone                  = useless without main board
```

### Multi-coin overhead

| Aspect | Coldcard Mk4 | Keystone Pro 3 |
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
4. Settings -> Lock the device to "Bitcoin only" mode (if available in
   your firmware) to disable altcoin parsers and reduce surface.
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
  parsers are present. A bug in any chain's transaction parser is a
  potential vector. Keystone has a "Bitcoin-only" firmware variant
  in some releases - prefer it if available.
- **Removable SE as a feature**: separating the SE physically is a
  reasonable cold-storage practice but most users keep them assembled,
  losing the benefit. Only useful if you actually swap.
- **Fingerprint as second factor**: convenient but biometrics are not
  high-security against a coercive attacker (forced fingerprint).
  PIN remains primary.
- **Shamir on Keystone vs Trezor**: SLIP-39 implementations should
  interop, but cross-vendor recovery is rare and not always tested.
  Keep your recovery process within the same vendor unless you've
  verified.
- **OTA update channel**: Keystone updates via SD card image, signed by
  vendor key. Always verify GPG signature on the firmware blob before
  flashing.

## References

- Keystone docs: https://docs.keyst.one/
- Keystone firmware repos: https://github.com/KeystoneHQ
- Keystone Companion app: https://keyst.one/companion-app
