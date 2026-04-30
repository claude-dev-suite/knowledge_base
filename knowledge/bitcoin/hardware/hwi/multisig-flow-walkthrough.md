# HWI Multisig Flow Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/hwi`.
> Canonical source: https://github.com/bitcoin-core/HWI
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/hwi/SKILL.md

## Concept

HWI handles multisig signing across vendors via the Wallet Policies
spec (BIP-388, Ledger's wallet-policy concept generalized). The
coordinator builds a descriptor, registers it on each device that
needs HMAC-binding (Ledger, BitBox02), then the spend round iterates
through devices, each adding its partial signature to the PSBT.

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

### Register on Ledger (HMAC required)

```bash
# Ledger needs explicit registration to sign multisig
hwi -f e5f6a7b8 \
    --wallet-name "MyMultisig" \
    --wallet-hmac "" \
    enumerate

# Returns hmac after device confirmation:
# {"name": "MyMultisig", "hmac": "abc123...def"}

# Save the hmac; you'll need it for every subsequent sign call.
```

For other vendors (Trezor, Coldcard, BitBox02, Jade) you typically
also import the descriptor via vendor-specific commands or device-side
file:

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

```bash
for fp in a1b2c3d4 e5f6a7b8 c9d0e1f2; do
    echo "Verifying on $fp:"
    hwi -f $fp displayaddress \
        --desc "$DESCRIPTOR" \
        --addr-index 0
    # Device shows full bech32 address; user confirms it matches
    # what coordinator generated.
done
```

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

### Signing on Ledger (with HMAC)

```bash
hwi -f e5f6a7b8 \
    --wallet-name "MyMultisig" \
    --wallet-hmac "abc123...def" \
    signtx "$PSBT"
```

## Worked example

End-to-end script for 2-of-3 multisig sign:

```bash
#!/bin/bash
DESCRIPTOR=$(cat my-multisig.desc)
PSBT_IN=$(cat unsigned.psbt | base64 -w0)
LEDGER_HMAC=$(cat ledger-hmac.txt)

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
- **Missing HMAC on Ledger**: returns `0x6985` (cancelled by user) but
  it's actually the policy-registration check failing. Re-run register
  step.
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
- BIP-388 (Wallet Policies): https://github.com/bitcoin/bips/blob/master/bip-0388.mediawiki
- BIP-48 (Multi-Sig HD): https://github.com/bitcoin/bips/blob/master/bip-0048.mediawiki
- Sparrow multisig docs: https://www.sparrowwallet.com/docs/multisig.html
