# APO / SIGHASH_ANYPREVOUT (BIP118) - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/proposals`.
> Canonical source: BIP118
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/proposals/SKILL.md

## Concept

`SIGHASH_ANYPREVOUT` (APO) and `SIGHASH_ANYPREVOUTANYSCRIPT` (APOAS)
are new sighash flags for Schnorr/Taproot signatures. They tell the
verifier to OMIT the spent prevout (and optionally the spent
scriptPubKey) from the sighash digest. The result: a signature
remains valid when applied to ANY transaction spending an output with
the same key (and, for APO, the same script).

The skill mentions Eltoo as the original use case; this article walks
the digest changes vs BIP341, the security model implications of
sig-binding to less context, and why APO requires a new key version
flag in Tapscript.

## Walkthrough / mechanics

**Standard BIP341 sighash commits to:**

```
- nVersion, nLockTime, hash type
- sha_prevouts (or specific prevout if ANYONECANPAY)
- sha_amounts (or specific amount if ANYONECANPAY)
- sha_scriptpubkeys
- sha_sequences
- (depending on hash type) sha_outputs or specific output
- input index
- (script-path) tapleaf hash, key version, codesep
```

**APO-mode sighash omits:**

```
- spent prevout (the txid:vout being spent)

Still committed:
- spent script
- spent amount
- the input's sequence
```

**APOAS-mode further omits:**

```
- spent script
- (some variants) spent amount
```

**Key version requirement:**

To distinguish APO-style sigs from legacy ones, a new "key version"
byte is introduced in Tapscript. BIP118 uses a 0x01 leaf-version-byte
encoding to signal "this leaf accepts APO sigs"; the 0x00 standard
version still commits to all of BIP341.

The 33-byte pubkey form is also extended: the first byte signals the
sighash mode the key supports.

## Worked example

**Eltoo settlement:**

Lightning channels using APO can replace the BOLT3 commitment-tx +
penalty-tx mechanism with a simpler "latest state wins" replacement.

Setup:

```
funding_tx -> 2-of-2 multisig output with APO key

Both parties have signatures over a state transition tx that updates
the channel balance:

state_tx_n:
  inputs:  [funding_outpoint OR previous state_tx_outpoint]
  outputs:
    out0: balance_alice
    out1: balance_bob
    out2: state_n CSV gate (forces latest state)
```

Because the signature is APO-bound, it doesn't need to specify which
prevout it spends. State n+1's signatures can spend either:
- The original funding output, OR
- A prior state_tx's output

This gives "always-latest-state" semantics without per-state penalty
keys.

**Concrete digest comparison:**

For input 0 of a state_tx using APO with SIGHASH_ALL semantics:

```
m  = 0x00                                # epoch
m += 0x41                                # APO | ALL (hash type)
m += version_le, locktime_le             # tx fields
# ANYPREVOUT: SKIP sha_prevouts and sha_amounts
m += sha256(scriptpubkeys)               # still committed
m += sha256(sequences)
m += sha256(outputs)
m += spend_type_byte                     # ext_flag adjustments
# ANYPREVOUT: SKIP outpoint, amount
m += scriptpubkey_serialized             # this input's spk (APO commits)
m += sequence_le
m += input_index_le
# script-path additions (tapleaf, key_version, codesep)
digest = TaggedHash("TapSighash", m)
```

Signature `schnorr_sign(seckey, digest)` verifies against ANY tx
spending any prevout with the same scriptPubKey + amount at the
specified input position.

**Replay risk:**

A signed APO message is, by design, replayable across different
prevouts of the same script. The wallet must include enough commitment
(via the sequence, locktime, or output binding) to prevent unintended
replay. In Eltoo, locktime + state-counter make replay produce
identical outputs.

## Common bugs / pitfalls

1. **Confusing APO with ANYONECANPAY.** ANYONECANPAY drops other
   inputs from the commit; APO drops THIS input's prevout. Different
   axis.
2. **Using APO with unintended outputs.** A naive APO key over a
   simple `<key> CHECKSIG` script lets ANY future tx spending an
   output of this script reuse the signature. Wallets must NEVER
   present APO addresses as "normal receive addresses".
3. **State versioning in Eltoo.** Without a state counter committed
   in outputs (e.g., via locktime trick), an attacker can replay
   an old state's APO sig to a later commitment. Eltoo uses
   `nLockTime = block_height + state_n` to encode state monotonically.
4. **Activation interaction with key tagging.** APO requires a Tapscript
   leaf-version that doesn't conflict with 0xc0. Using APO sigs in a
   0xc0 leaf would either fail verification or silently accept,
   depending on activation status.
5. **APOAS unbinds the script.** A signature valid for spending P2TR
   output A could be reused for output B (different script) - useful
   for some L2 designs but extremely dangerous if misused. APOAS
   keys should be considered single-purpose.
6. **No mainnet activation.** Reasoning about Lightning Eltoo as if
   APO were live is incorrect. Always tag "requires BIP118 activation".

## References

- BIP118: https://github.com/bitcoin/bips/blob/master/bip-0118.mediawiki
- Eltoo paper: https://blockstream.com/eltoo.pdf
- Reference impl signet: https://github.com/ajtowns/bitcoin/tree/202203-anyprevout
- Anthony Towns talks on APO: youtube
- Lightning future: https://blog.bitmex.com/lightnings-eltoo/
