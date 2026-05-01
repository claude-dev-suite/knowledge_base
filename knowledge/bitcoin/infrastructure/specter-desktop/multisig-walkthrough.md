# Specter Desktop Multisig Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/specter-desktop`.
> Canonical source: https://docs.specter.solutions/desktop/multisig/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/specter-desktop/SKILL.md

## Concept

Specter Desktop coordinates a 2-of-3 multisig wallet across hardware
devices from different vendors using BIP380 output descriptors. The
coordinator never sees private keys; each device produces a signature for
a Partially Signed Bitcoin Transaction (PSBT) which Specter combines and
broadcasts via Bitcoin Core. This article walks the full vendor-mixed
setup (Coldcard + Trezor + BitBox02) on mainnet, including descriptor
inspection, address verification on every device, and PSBT round-trips.

## Walkthrough / mechanics

Prerequisites:

- Bitcoin Core synced; cookie-auth or `~/.specter/config.json` configured.
- HWI 3.x installed and on `$PATH`; udev rules on Linux:
  ```bash
  curl -sL https://raw.githubusercontent.com/bitcoin-core/HWI/master/hwilib/udev/51-coinkite.rules \
    | sudo tee /etc/udev/rules.d/51-coinkite.rules
  sudo udevadm trigger
  sudo udevadm control --reload-rules
  ```
- Specter installed (`pip install specter-desktop` or installer).

Step 1 - Add devices. In Specter UI: Add new device -> select vendor.
For each device:

```
Coldcard: export ckcc.json from Advanced -> MicroSD -> Export Wallet ->
          Generic JSON (covers BIP84 path)
Trezor:   plug in, unlock, click "Enable"
BitBox02: plug in, click "Pair", confirm fingerprint on device
```

Specter records each device's master fingerprint and queries xpubs at
standard derivations: `m/48'/0'/0'/2'` for native segwit multisig
(P2WSH), `m/48'/0'/0'/1'` for P2SH-P2WSH.

Step 2 - Create wallet. Add new wallet -> Multisignature ->
- Devices: select all three.
- Type: Native Segwit (P2WSH).
- Threshold: 2 of 3.
- Specter assembles the descriptor:

```
wsh(sortedmulti(2,
  [d34db33f/48'/0'/0'/2']xpub6E...Coldcard/0/*,
  [a1b2c3d4/48'/0'/0'/2']xpub6F...Trezor/0/*,
  [99887766/48'/0'/0'/2']xpub6G...BitBox/0/*
))#abc12345
```

Specter imports this descriptor into Bitcoin Core via
`importdescriptors`, which begins scanning for the wallet's UTXOs.

Step 3 - Verify a receive address on every device. For each device click
"Display address" - Specter sends the descriptor + index to the device,
and the device shows the address on its screen for manual comparison
with the screen-displayed address in Specter. **All three must match**;
disagreement means a compromise or bad descriptor.

Step 4 - Spend flow:

```
Specter -> Send -> fill recipient + fee
       -> Create unsigned PSBT
       -> Sign with device 1 (Coldcard via SD card / USB)
       -> Sign with device 2 (Trezor)
       -> Combine -> Finalize -> Broadcast (via Bitcoin Core)
```

The third device is unused unless one of the first two refuses.

## Worked example

Inspect the descriptor and address derivation manually with bitcoin-cli:

```bash
DESC='wsh(sortedmulti(2,[d34db33f/48h/0h/0h/2h]xpub6E.../0/*,[a1b2c3d4/48h/0h/0h/2h]xpub6F.../0/*,[99887766/48h/0h/0h/2h]xpub6G.../0/*))'

# Compute address at index 0
bitcoin-cli deriveaddresses "$DESC" "[0,0]"
# [ "bc1q...44chars..." ]
```

Cross-check by exporting Specter's PSBT and re-loading it into Sparrow
for an independent verification:

```bash
# After creating the unsigned PSBT in Specter UI, click Download -> psbt.json
# Then in Sparrow: File -> Open Transaction -> psbt.json
# Sparrow shows the same inputs, outputs, and fee
```

Sample PSBT signing round trip with Coldcard via microSD:

```
1. Specter exports unsigned.psbt to disk.
2. Copy to microSD, insert into Coldcard.
3. Coldcard: Ready To Sign -> review -> sign -> writes unsigned-signed.psbt.
4. Copy back to laptop, drop into Specter -> "Add Signature".
5. Repeat for Trezor (USB direct).
6. Specter shows "Final" -> Broadcast.
```

| Stage | Tool | Time | Risk if skipped |
|-------|------|------|-----------------|
| Verify each device address | All three HW screens | 2 min | Compromised descriptor goes unnoticed |
| Save descriptor + xpubs | Backup vault | 1 min | Lost wallet on device failure |
| Test small recovery on signet | All devices | 30 min | Real-money first attempt fails |
| PSBT version pin (v0 vs v2) | Specter settings | n/a | Some firmware rejects v2 |

## Common pitfalls

- Sorting matters - `sortedmulti` orders xpubs lexicographically; never
  edit the descriptor by hand to reorder, the resulting addresses change.
- Forgetting to back up the descriptor - losing it makes the wallet
  unrecoverable even with all three seeds, because address derivation
  needs the full xpub list and the threshold.
- Mixing derivation paths between vendors - some firmware versions of
  Coldcard default to `m/45'`, Trezor to `m/48'/0'/0'/2'`. Always pick a
  single path (`48'/0'/0'/2'` for native multisig P2WSH) and confirm.
- HWI permission errors on Linux - missing udev rules cause "Permission
  denied" when Specter tries to enumerate devices.
- PSBT version mismatch - older Coldcard firmware rejects PSBT v2; in
  Specter set "PSBT version" to v0 in advanced settings.
- Bitcoin Core wallet not loaded - if Core was restarted but the
  descriptor wallet was not auto-loaded, Specter shows "no balance";
  `bitcoin-cli loadwallet specter_<name>`.

## References

- Specter docs: https://docs.specter.solutions/
- Multisig guide: https://docs.specter.solutions/desktop/multisig/
- HWI repo: https://github.com/bitcoin-core/HWI
- BIP380 descriptors: https://github.com/bitcoin/bips/blob/master/bip-0380.mediawiki
