# HWI Multisig Flow Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/hwi`.
> Canonical source: https://github.com/bitcoin-core/HWI
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/hwi/SKILL.md

## Concept

HWI handles multisig signing across vendors via the Wallet Policies
spec (BIP-388, Ledger's wallet-policy concept generalized). The
coordinator builds a descriptor, makes sure each device that needs
policy registration has confirmed it - Ledger (per-call HMAC) and
BitBox02 (persistent on-device script config) - then the spend round
iterates through devices, each adding its partial signature to the PSBT.

> **Which HWI is this?** Everything below targets HWI 3.2.0
> (10 February 2026), the current release. In released HWI there is no
> registration subcommand: the Ledger driver reconstructs the wallet
> policy from the PSBT's `bip32_derivation` fields and registers it
> inline on every `signtx` / `displayaddress` call, so the user
> re-confirms the policy on the device each run and no HMAC is stored
> host-side. Explicit BIP-388 plumbing (`hwi registerdescriptor`, plus
> `--registration` on `signtx` and `displayaddress`) is merged on master
> via PRs [#841](https://github.com/bitcoin-core/HWI/pull/841) and
> [#792](https://github.com/bitcoin-core/HWI/pull/792) (August 2026) but
> unreleased as of September 2026. Note also that HWI entered wind-down
> in August 2026 - see the lifecycle note in `cli-walkthrough.md`.

The PSBT v2 format makes multi-vendor signing easier because:

- Inputs carry `non_witness_utxo` or `witness_utxo` plus
  `bip32_derivation` per signer pubkey.
- Each device knows which inputs belong to it via fingerprint match.
- Devices ignore inputs not matching their fingerprint (don't error).

## Walkthrough

### Build the multisig descriptor

```bash
# Get xpub from each device (record fingerprint + path)
hwi -f a1b2c3d4 getxpub "m/48h/0h/0h/2h"   # Trezor
# {"xpub": "xpub6E..."}

hwi -f e5f6a7b8 getxpub "m/48h/0h/0h/2h"   # Ledger
# {"xpub": "xpub6F..."}

hwi -f c9d0e1f2 getxpub "m/48h/0h/0h/2h"   # Coldcard
# {"xpub": "xpub6G..."}

# Construct 2-of-3 wsh descriptor
DESCRIPTOR='wsh(sortedmulti(2,
[a1b2c3d4/48h/0h/0h/2h]xpub6E.../<0;1>/*,
[e5f6a7b8/48h/0h/0h/2h]xpub6F.../<0;1>/*,
[c9d0e1f2/48h/0h/0h/2h]xpub6G.../<0;1>/*))#chk'
```

### Register on Ledger (policy confirmation)

On HWI 3.2.0 there is nothing to run here. The Ledger driver builds a
`MultisigWallet` policy from the PSBT (or from the `--desc` passed to
`displayaddress`) and calls `register_wallet` itself; the device shows
the policy and the user approves it. The HMAC it returns is used for
that one call and then discarded, so the prompt reappears on every
signing run. Budget for it in any scripted flow - it cannot be
pre-approved.

On master (unreleased as of September 2026) the registration becomes
explicit and reusable:

```bash
hwi -f e5f6a7b8 registerdescriptor "MyMultisig" "$DESCRIPTOR"
# {"registration": "..."}   # save this blob
```

BitBox02 is not HMAC-bound. Its HWI driver (`hwilib/devices/bitbox02.py`
at 3.2.0) calls `btc_is_script_config_registered` and, only if the
multisig script config is unknown, `btc_register_script_config` - you
name the wallet on the device once and the registration persists there,
so later signings do not re-prompt for it.

For other vendors (Trezor, Coldcard, Jade) you typically also import
the descriptor via vendor-specific commands or device-side file:

```bash
# Coldcard via SD card: write multisig wallet definition to SD
cat > /media/sd/multisig.txt <<EOF
Name: MyMultisig
Policy: 2 of 3
Format: P2WSH
Derivation: m/48'/0'/0'/2'
A1B2C3D4: xpub6E...
E5F6A7B8: xpub6F...
C9D0E1F2: xpub6G...
EOF
# Coldcard menu: Settings -> Multisig Wallets -> Import from SD
```

### Verify address on each device

`displayaddress --desc` on HWI 3.2.0 takes a *concrete* descriptor -
each key needs a fixed receive/index suffix, not a `<0;1>/*` multipath
range. Derive the single-address descriptor first:

```bash
# Same descriptor as above but with /0/0 instead of /<0;1>/*
ADDR0_DESC='wsh(sortedmulti(2,
[a1b2c3d4/48h/0h/0h/2h]xpub6E.../0/0,
[e5f6a7b8/48h/0h/0h/2h]xpub6F.../0/0,
[c9d0e1f2/48h/0h/0h/2h]xpub6G.../0/0))'

for fp in a1b2c3d4 e5f6a7b8 c9d0e1f2; do
    echo "Verifying on $fp:"
    hwi -f $fp displayaddress --desc "$ADDR0_DESC"
    # Device shows full bech32 address; user confirms it matches
    # what coordinator generated.
done
```

On master, a saved registration replaces the hand-derived descriptor:
`hwi -f $fp displayaddress --registration "$REG" --index 0`.

### Spend round

```bash
# Create unsigned PSBT (e.g., from Bitcoin Core watch-only wallet
# tracking the descriptor)
PSBT=$(bitcoin-cli walletcreatefundedpsbt ...)

# Sign on Trezor (fingerprint a1b2c3d4)
PSBT=$(hwi -f a1b2c3d4 signtx "$PSBT" | jq -r .psbt)

# Sign on Coldcard (fingerprint c9d0e1f2)
PSBT=$(hwi -f c9d0e1f2 signtx "$PSBT" | jq -r .psbt)

# (Skip Ledger; we have 2 of 3 already)

# Finalize
bitcoin-cli finalizepsbt "$PSBT"
# Broadcast
bitcoin-cli sendrawtransaction "$RAW_TX"
```

### Signing on Ledger

```bash
# HWI 3.2.0: identical to any other device. The policy prompt appears
# on the Ledger screen first; approve it, then approve the spend.
hwi -f e5f6a7b8 signtx "$PSBT"

# On master (unreleased as of September 2026), a saved registration
# skips the re-confirmation:
hwi -f e5f6a7b8 signtx --registration "$REG" "$PSBT"
```

## Worked example

End-to-end script for 2-of-3 multisig sign:

```bash
#!/bin/bash
DESCRIPTOR=$(cat my-multisig.desc)
PSBT_IN=$(cat unsigned.psbt | base64 -w0)

# First signer: Trezor
PSBT=$(hwi -f a1b2c3d4 signtx "$PSBT_IN" | jq -r .psbt)
echo "After Trezor:"
bitcoin-cli decodepsbt "$PSBT" | jq '.inputs[0].partial_signatures | keys'

# Second signer: Coldcard
PSBT=$(hwi -f c9d0e1f2 signtx "$PSBT" | jq -r .psbt)
echo "After Coldcard:"
bitcoin-cli decodepsbt "$PSBT" | jq '.inputs[0].partial_signatures | keys'

# Finalize
RESULT=$(bitcoin-cli finalizepsbt "$PSBT")
TX=$(echo "$RESULT" | jq -r .hex)
bitcoin-cli sendrawtransaction "$TX"
```

## Common pitfalls

- **Sortedmulti vs multi**: `sortedmulti` re-orders pubkeys per BIP-67;
  `multi` keeps original order. Mismatched descriptor on devices
  produces different addresses. Always use `sortedmulti` for fresh
  setups; older wallets may use `multi`.
- **Path inconsistency**: BIP-48 path with hardened indicator differs
  across tools (`/48h/0h/0h/2h` vs `/48'/0'/0'/2'`). HWI normalizes,
  but coordinator output may not. Pin the format.
- **Ledger policy prompt looks like a hang**: on HWI 3.2.0 every
  multisig `signtx` / `displayaddress` first asks the Ledger to approve
  the wallet policy, and only then the spend. Unattended scripts stall
  there; a declined or timed-out prompt comes back as `0x6985`
  (cancelled by user), not as a policy error.
- **Descriptor must match byte-for-byte**: the policy HWI derives from
  the PSBT has to reproduce the one the device already knows - same key
  order, same origin fingerprints, same `sortedmulti` vs `multi`.
  Because the driver registers whatever it derived, a mismatch shows up
  as a fresh registration prompt for an unfamiliar policy rather than a
  loud failure - read the device screen, do not click through it.
- **Coldcard SD wallet not imported**: device sees the PSBT but says
  "not multisig wallet". Import via SD before signing.
- **Cross-vendor PSBT version mismatch**: PSBT v0 vs v2; HWI converts
  internally but some old firmware emits only v0. Standardize on v0
  for compat or check vendor support matrices.
- **Witness UTXO required**: native segwit multisig requires
  `witness_utxo` field in the PSBT input; legacy P2SH-multisig requires
  `non_witness_utxo`. Coordinator must populate per-input.

## References

- HWI repo: https://github.com/bitcoin-core/HWI
- HWI wind-down announcement: https://github.com/bitcoin-core/HWI/issues/850
- BIP-388 (Wallet Policies): https://github.com/bitcoin/bips/blob/master/bip-0388.mediawiki
- BIP-48 (Multi-Sig HD): https://github.com/bitcoin/bips/blob/master/bip-0048.mediawiki
- Sparrow multisig docs: https://www.sparrowwallet.com/docs/multisig.html
