# Sparrow PSBT Flows Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/sparrow`.
> Canonical source: https://sparrowwallet.com/docs/transactions.html
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/sparrow/SKILL.md

## Concept

Sparrow is the most PSBT-native of the popular desktop wallets: every
transaction goes through the BIP174 PSBT lifecycle (create, sign, combine,
finalize, broadcast), with a UI that exposes each stage explicitly. This
article walks the four canonical PSBT flows in Sparrow: a single-sig spend
to a hardware wallet, an air-gapped flow via animated QR (BCUR), a
multisig 2-of-3 flow with two distinct devices, and a "watch-only sign on
offline laptop" flow using file transfer. Throughout we use real menu
paths and CLI helpers.

## Walkthrough / mechanics

Flow A - Single-sig with USB-connected Coldcard:

```
File -> New Wallet -> "Single Signature"
Script Type: Native Segwit (P2WPKH)
Keystore: Connected Hardware Wallet -> Discover -> select Coldcard
Sparrow imports xpub at m/84'/0'/0'
```

Send tab:
1. Enter recipient + amount + fee rate.
2. "Create Transaction" -> Sparrow builds an unsigned PSBT.
3. Transaction tab shows hex, inputs, outputs, witness placeholders.
4. Click "Finalize Transaction for Signing" -> "Sign".
5. Coldcard prompts; user reviews on device screen, approves.
6. Sparrow receives the signed PSBT, finalizes, "Broadcast Transaction".

Flow B - Air-gapped via animated QR (BCUR):

```
File -> New Wallet -> Single Signature
Keystore: Airgapped Hardware Wallet -> "Show on Camera"
Coldcard / SeedSigner / Passport -> Export xpub via QR
Sparrow camera scans QR, imports xpub
```

Send: Sparrow displays the unsigned PSBT as an animated QR; the device
camera ingests, signs, and displays a signed PSBT QR back; Sparrow's
camera scans, broadcasts.

Flow C - Multisig 2-of-3:

```
File -> New Wallet -> "Multi Signature"
Threshold: 2 of 3
Keystores: add three (Coldcard, Trezor, BitBox02)
Script Type: Native Segwit (P2WSH)
```

Sparrow assembles the descriptor:

```
wsh(sortedmulti(2,
  [d34db33f/48'/0'/0'/2']xpub6E.../0/*,
  [a1b2c3d4/48'/0'/0'/2']xpub6F.../0/*,
  [99887766/48'/0'/0'/2']xpub6G.../0/*))#hash
```

Sign cycle:
1. Sparrow creates PSBT.
2. Right-click signature row -> "Sign" with device 1.
3. After first signature, second signature row populates the same way.
4. Once threshold reached, Sparrow auto-finalizes.
5. Broadcast.

Flow D - Watch-only on online laptop, sign on offline laptop:

```
ONLINE laptop:
  Wallet: Single Signature, Keystore: "Software Wallet" (xpub-only)
  Or import descriptor JSON exported from offline machine.
  Build PSBT -> File -> Save Transaction -> unsigned.psbt

(transfer USB stick)

OFFLINE laptop:
  Same wallet template but with seed-loaded software keystore.
  File -> Open Transaction -> unsigned.psbt
  Click Sign -> save signed.psbt

(transfer back)

ONLINE laptop:
  File -> Open Transaction -> signed.psbt
  Finalize + Broadcast.
```

## Worked example

Inspect a saved PSBT using Bitcoin Core's `decodepsbt`:

```bash
PSBT=$(base64 -w0 unsigned.psbt)
bitcoin-cli decodepsbt "$PSBT" | jq '{
  inputs: [.inputs[] | {witness_utxo, bip32_derivs: .bip32_derivs}],
  outputs: [.outputs[] | {amount, address: .script.address}]
}'
```

Sample output for a P2WSH multisig input:

```json
{
  "inputs": [
    {
      "witness_utxo": { "amount": 0.05, "scriptPubKey": { "address": "bc1q..." } },
      "bip32_derivs": [
        { "master_fingerprint": "d34db33f", "path": "m/48h/0h/0h/2h/0/3" },
        { "master_fingerprint": "a1b2c3d4", "path": "m/48h/0h/0h/2h/0/3" },
        { "master_fingerprint": "99887766", "path": "m/48h/0h/0h/2h/0/3" }
      ]
    }
  ],
  "outputs": [
    { "amount": 0.04985, "address": "bc1q...recipient..." },
    { "amount": 0.00010, "address": "bc1q...change..." }
  ]
}
```

Run a small CI-style sanity check that two PSBTs combine cleanly:

```bash
bitcoin-cli combinepsbt '["base64-sig-1","base64-sig-2"]'
bitcoin-cli finalizepsbt '<combined>'
# returns { "psbt": "...", "complete": true, "hex": "raw-tx" }
```

| Flow | Online security | Speed | Best for |
|------|-----------------|-------|----------|
| A: USB single-sig | Medium | Fast | Daily small spends |
| B: Animated QR | Highest | Slow | Cold storage withdrawals |
| C: Multisig USB | High | Medium | Treasury, vault |
| D: File-transfer airgap | High | Medium | Power user with two laptops |

## Common pitfalls

- Saving as PSBT v0 vs v2 - some firmware (older Coldcard, certain
  ColdCard MK3 builds) reject v2; Settings -> Wallet -> "PSBT version 0"
  if signing fails.
- Wrong derivation in multisig - after rebuilding a wallet, Sparrow
  defaults to native segwit P2WSH at `m/48'/0'/0'/2'`; do not mix with a
  device exported at `m/48'/0'/0'/1'` (P2SH-P2WSH).
- Animated QR misalignment - device camera not centered or low light
  -> incomplete PSBT scan; rescan from the start.
- Confusing "Sign" button greyed out - this indicates Sparrow has no
  matching keystore for the input (wrong descriptor or missing xpub).
- Not verifying change address - in airgap flow always check that the
  device flagged the change output as belonging to the wallet, otherwise
  malicious software could redirect change.
- Broadcast errors after finalization - Sparrow shows the bitcoind
  rejection reason; common cause is fee bumping past the wallet's max
  fee setting (Settings -> Send tab).

## References

- Sparrow docs: https://sparrowwallet.com/docs/
- PSBT walkthrough: https://sparrowwallet.com/docs/transactions.html
- BIP174 PSBT v0: https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki
- BIP370 PSBT v2: https://github.com/bitcoin/bips/blob/master/bip-0370.mediawiki
