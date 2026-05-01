# PSBT Role Pipeline Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/psbt`.
> Canonical source: BIP174 sections "Roles" and "Roles in detail"
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/psbt/SKILL.md

## Concept

A PSBT moves through a fixed pipeline of roles: Creator -> Updater ->
Signer -> Combiner -> Finalizer -> Extractor. Each role has well-defined
allowed mutations. Real-world flows often interleave Updater and Signer
across multiple devices (e.g., 2-of-3 multisig with three hardware
wallets). This article walks a full 2-of-3 multisig flow byte-by-byte
with `bitcoin-cli` and `hwi`.

## Walkthrough / mechanics

The roles map to allowed PSBT mutations as follows:

| Role | Reads | Writes | Forbidden |
|------|-------|--------|-----------|
| Creator | n/a | global tx, empty input/output records | partial sigs |
| Updater | utxos, descriptors | non_witness_utxo, witness_utxo, redeem/witness scripts, BIP32 derivations, taproot fields | tx skeleton change |
| Signer | everything | partial_sig (legacy/segwit), tap_key_sig/tap_script_sig (taproot) | other inputs' fields |
| Combiner | multiple PSBTs | union of partial sigs | re-sign |
| Finalizer | partial sigs, scripts | final_scriptSig, final_scriptWitness, removes superseded fields | new sigs |
| Extractor | finalized PSBT | raw tx bytes | nothing else |

Each role MUST be idempotent except Creator. A Combiner running twice
on the same set of inputs produces the same output. A Finalizer
running on an already-finalized PSBT is a no-op.

## Worked example

Setup: 2-of-3 P2WSH multisig with descriptor

```
wsh(sortedmulti(2,
  [d34db33f/48h/0h/0h/2h]xpub.../0/*,
  [b15ec0de/48h/0h/0h/2h]xpub.../0/*,
  [c0ffee00/48h/0h/0h/2h]xpub.../0/*))#chk
```

Address index 0 has been funded with 1.0 BTC at outpoint
`abc..def:0`. We send 0.5 BTC to `bc1q...recv` with 1 sat/vB fee.

**Creator** (any node with the descriptor):

```bash
bitcoin-cli walletcreatefundedpsbt \
  '[{"txid":"abc..def","vout":0}]' \
  '[{"bc1q...recv":0.5}]' \
  0 '{"changeAddress":"bc1q...chg","feeRate":0.00001}'
# returns base64 PSBT psbt0
```

At this stage the PSBT has the unsigned tx + empty per-input records.

**Updater** (same node, since it owns the descriptor with all xpubs):

`walletprocesspsbt` is auto-Updater + Signer if any privkey is present.
For watch-only flow:

```bash
bitcoin-cli updatepsbt $psbt0
# adds witness_utxo, witness_script, bip32_derivation for each input/output
```

**Signer 1** (Coldcard with `[d34db33f/...]`):

```bash
hwi --device-type coldcard signtx $psbt0
# returns psbt1 with PSBT_IN_PARTIAL_SIG (key=pubkey1, value=ecdsa_sig1)
```

**Signer 2** (Trezor with `[b15ec0de/...]`):

```bash
hwi --device-type trezor signtx $psbt0          # uses psbt0, NOT psbt1!
# returns psbt2 with PSBT_IN_PARTIAL_SIG (key=pubkey2, value=ecdsa_sig2)
```

Each signer reads the SAME initial Updater output. They don't combine
- that's the next role.

**Combiner** (any party with both partial PSBTs):

```bash
bitcoin-cli combinepsbt '["'$psbt1'","'$psbt2'"]'
# returns psbt_combined with both PSBT_IN_PARTIAL_SIG entries
```

The combiner deduplicates: if both PSBTs have the same partial sig,
only one copy remains; conflicting values cause an error.

**Finalizer:**

```bash
bitcoin-cli finalizepsbt $psbt_combined
# returns { "psbt": "<finalized>", "hex": "<rawtx>", "complete": true }
```

The finalizer sees 2 partial sigs + 3-key witness script + threshold
of 2, so it builds:

```
final_scriptWitness = [
  <empty>,        # CHECKMULTISIG bug push
  <sig_from_pk_in_sortedmulti_pos_x>,
  <sig_from_pk_in_sortedmulti_pos_y>,
  <witness_script>
]
```

Sigs are placed in the order their pubkeys appear in `sortedmulti`,
NOT in signing order.

**Extractor:** automatic in `finalizepsbt` once `complete: true`.

## Common bugs / pitfalls

1. **Signer chaining instead of branching.** Sending PSBT to S1 then
   the OUTPUT of S1 to S2 works (S2 just adds its sig), but is fragile
   - if S1 also runs the Finalizer, S2 sees no inputs left to sign.
   Always branch from the same Updater state, then Combine.
2. **Updater modifying the unsigned tx.** Adding/removing an input
   after a Signer has signed invalidates all prior partial sigs.
   PSBT v2 was introduced partly to make this safer, but Signers
   still must re-sign after tx mutation.
3. **Skipping Updater for Taproot.** A Signer needs `PSBT_IN_TAP_INTERNAL_KEY`
   AND `PSBT_IN_TAP_BIP32_DERIVATION` to know which key path to use.
   Without these, hardware wallets refuse.
4. **Combining incompatible PSBT versions.** A v0 PSBT cannot be
   merged with a v2 PSBT - field semantics differ (v2 has per-input
   amount fields; v0 stores tx in global).
5. **Finalizer eagerly clearing fields.** Per BIP174, after finalization
   the Finalizer SHOULD remove superseded fields (partial sigs, redeem
   scripts) to shrink the PSBT. Some libs leave them; downstream tools
   may treat the PSBT as "unfinalized" if they look for partial sigs.
6. **Extractor accepting non-`complete: true` PSBT.** If a Finalizer
   ran before all signatures arrived, `complete: false` and Extractor
   must refuse - extracting partial state produces an invalid tx.

## References

- BIP174: https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki
- BIP370: https://github.com/bitcoin/bips/blob/master/bip-0370.mediawiki
- Bitcoin Core PSBT how-to: https://github.com/bitcoin/bitcoin/blob/master/doc/psbt.md
- HWI: https://github.com/bitcoin-core/HWI
