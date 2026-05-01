# PSBT Finalization Bugs - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/psbt`.
> Canonical source: BIP174 (Roles), BIP371 (Taproot finalization)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/psbt/SKILL.md

## Concept

Finalization converts partial signatures into the final
`scriptSig`/`scriptWitness`. It is the most error-prone PSBT step
because the Finalizer must understand the spending path, ordering
conventions, witness-script parsing, and Taproot leaf selection.
This article catalogs the recurring bugs across HWI, Bitcoin Core,
Sparrow, and bitcoinjs-lib, with the byte-level cause of each.

## Walkthrough / mechanics

A correct Finalizer pseudocode for legacy/segwit:

```
for each input i:
  read input fields
  classify_spk(witness_utxo or non_witness_utxo):
    P2PKH         -> need 1 sig
    P2WPKH        -> need 1 sig, build [<sig>, <pk>] witness
    P2SH-P2WPKH   -> as P2WPKH, plus scriptSig = PUSH(redeem_script)
    P2SH(multi)   -> need k partial_sigs ordered by ms_pubkeys
                     scriptSig = OP_0 <sig1> ... <sigk> PUSH(redeem_script)
    P2WSH(multi)  -> witness = [empty, sig1, ..., sigk, witness_script]
    P2TR keypath  -> witness = [<tap_key_sig>]
    P2TR scriptpath-> witness = [<args>, leaf_script, control_block]
  set final_scriptSig / final_scriptWitness
  remove partial_sig, redeem_script, witness_script (now superseded)
```

The "extra empty push" before P2SH/P2WSH multisig sigs is the historic
`OP_CHECKMULTISIG` off-by-one. It stays even in modern wallets.

## Worked example

**Bug case 1: sig order from sortedmulti.**

PSBT has partial sigs from pubkeys `B`, `A` in `sortedmulti(2, A, B, C)`
witness script. Sortedmulti reorders pubkeys at descriptor expansion
time to `A < B < C`. The witness MUST list sigs in `A`-then-`B` order
(matching pubkey sorted order), not signing order.

```
witness_script = OP_2 <A> <B> <C> OP_3 OP_CHECKMULTISIG
witness        = [<empty>, <sig_A>, <sig_B>, witness_script]   # CORRECT
witness        = [<empty>, <sig_B>, <sig_A>, witness_script]   # WRONG -- script fails
```

A common bug: Finalizer iterates partial_sigs in the order they appear
in the PSBT field map (often insertion order = signing order).

**Bug case 2: Taproot key-path with leftover script-path sig.**

A multi-path Taproot wallet produces a key-path sig AND a
script-path sig. PSBT finalization SHOULD pick whichever is "preferred"
per BIP371; commonly key-path because witness is 64 bytes vs ~200.

```
PSBT_IN_TAP_KEY_SIG       = <64-byte sig>
PSBT_IN_TAP_SCRIPT_SIG    = <pkX || leafhash> -> <64-byte sig>
```

If Finalizer naively writes both into the witness:

```
witness = [<tap_key_sig>, <tap_script_sig>]      # WRONG -- script-path
                                                 # interpretation expects
                                                 # control_block last
```

Correct: pick exactly one mode and discard the other branch's fields.

**Bug case 3: missing `non_witness_utxo` post-pinning-attack hardening.**

After 2020 reports, hardware wallets require BOTH `witness_utxo` AND
`non_witness_utxo` for SegWit inputs to defend against spoofed
fee-bumping signatures. A Finalizer that strips `non_witness_utxo`
prematurely breaks downstream signing rounds.

```
# Bad finalizer (before sigs collected):
del psbt.inputs[i].non_witness_utxo

# Result: when the second signer receives this PSBT, HW refuses.
```

**Bug case 4: P2SH-P2WSH wrapping confusion.**

```
witness_utxo.scriptPubKey   = OP_HASH160 PUSH20 <h160(redeemScript)> OP_EQUAL
redeem_script               = OP_0 PUSH32 <sha256(witness_script)>
witness_script              = OP_2 <A> <B> <C> OP_3 OP_CHECKMULTISIG
```

Finalizer must produce:

```
final_scriptSig    = PUSH22 <redeem_script>
final_scriptWitness = [<empty>, <sig_A>, <sig_B>, witness_script]
```

Common error: omit the scriptSig, treating it like native P2WSH.
Bitcoin Core produces "non-mandatory-script-verify-flag (Witness
program hash mismatch)" on broadcast.

**Bug case 5: SIGHASH byte mismatch in tx_signing.**

Signer used SIGHASH_ALL (0x01); Finalizer's witness check expects
SIGHASH_DEFAULT (0x00) for Taproot. ECDSA signers append 0x01;
Schnorr signers MUST omit the suffix for SIGHASH_DEFAULT, append for
others. Finalizer must detect the witness sig length (64 vs 65) and
trust the included byte.

## Common bugs / pitfalls

1. **Finalizing before all signers contributed.** PSBT remains usable
   for further signing only if Finalizer left partial sigs alone. Once
   `final_scriptWitness` is written, downstream signers can't add more.
2. **Wrong empty-push for `OP_CHECKMULTISIG`.** Must be a zero-length
   push (`OP_0` or `OP_PUSHBYTES_0`, encoded as 0x00), not an empty
   element with no opcode prefix.
3. **Witness for P2WSH(multi) with k > 2 sigs collected.** Only the
   first `k` (in pubkey-sorted order) should appear. Excess sigs cause
   "invalid stack operation".
4. **Taproot finalizer choosing a leaf without sigs for it.** With
   multiple `TAP_LEAF_SCRIPT` entries, the Finalizer must choose a
   leaf for which all its signatures are present. Picking a leaf
   whose `pk` did NOT contribute a `TAP_SCRIPT_SIG` produces an
   incomplete witness.
5. **Stripping `tap_internal_key` post-finalization.** Some downstream
   tools (label tracking, sweep) need the internal key to identify
   the wallet; strip only `tap_*_sig` and `tap_leaf_script`.
6. **CSV/CLTV scripts forgotten in witness.** A miniscript with
   `older(n)` requires the script to be revealed at the END of the
   witness stack (P2WSH case). Finalizers that don't run miniscript
   satisfaction can omit the witness_script suffix.

## References

- BIP174: https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki
- BIP371: https://github.com/bitcoin/bips/blob/master/bip-0371.mediawiki
- Bitcoin Core PSBT how-to: https://github.com/bitcoin/bitcoin/blob/master/doc/psbt.md
- HWI fee-bumping defense: https://github.com/bitcoin-core/HWI/issues
- bitcoinjs-lib finalization: https://github.com/bitcoinjs/bitcoinjs-lib/blob/master/ts_src/psbt.ts
