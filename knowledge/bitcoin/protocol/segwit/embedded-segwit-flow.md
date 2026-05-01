# Embedded SegWit Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/segwit`.
> Canonical source: BIP141 (Witness program), BIP16 (P2SH), BIP49 (P2SH-P2WPKH derivation)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/segwit/SKILL.md

## Concept

Embedded (a.k.a. nested or wrapped) SegWit is a transition mechanism:
the witness program is wrapped inside a P2SH `redeemScript`, so the
output looks like a legacy `3...` address to senders that don't know
about Bech32. Activation in 2017 needed embedded SegWit so that
exchanges and merchants without Bech32 support could still spend to
SegWit-aware wallets and reap the witness discount on the spend side.

Two variants:
- **P2SH-P2WPKH** (BIP49 derivation `m/49'/coin'/account'`)
- **P2SH-P2WSH** (P2WSH wrapped in P2SH; rarely used in modern wallets)

Both pay 23 vB more per input than native SegWit; almost all new wallets
use BIP84 (`bc1q...`) or BIP86 (`bc1p...`) instead.

## Walkthrough / mechanics

For P2SH-P2WPKH with compressed pubkey `pk`:

```
witnessProgram = OP_0 PUSH20 <hash160(pk)>            # 22 bytes
redeemScript   = witnessProgram                        # exactly 22 bytes
scriptPubKey   = OP_HASH160 PUSH20 <hash160(redeemScript)> OP_EQUAL
address        = base58check(0x05 || hash160(redeemScript))   # mainnet "3..."
```

When spending:

```
scriptSig      = PUSH22 <redeemScript>                # ONE push, the 22-byte witnessProgram
witness        = [<sig>, <pk>]                        # native P2WPKH witness
```

Validation order:
1. Run `scriptSig` -> stack has the 22-byte push.
2. P2SH rule (BIP16): pop top, hash160, compare to `scriptPubKey`. Match.
3. The popped element is exactly a witness program -> trigger SegWit
   v0 P2WPKH validation using witness `[<sig>, <pk>]` against
   `OP_DUP OP_HASH160 <hash160(pk)> OP_EQUALVERIFY OP_CHECKSIG`.

For P2SH-P2WSH, the inner `witnessProgram = OP_0 PUSH32 <sha256(witnessScript)>`
(34 bytes) and the witness stack ends with the full `witnessScript`.

## Worked example

Single-sig P2SH-P2WPKH using BIP49 derivation:

```python
from hashlib import sha256, new as new_hash
import base58

def hash160(b):
    h = sha256(b).digest()
    r = new_hash('ripemd160')
    r.update(h)
    return r.digest()

pk_hex = "03a1b2c3...d4e5f6"            # compressed 33 bytes
pk = bytes.fromhex(pk_hex)

witness_program = b'\x00\x14' + hash160(pk)        # OP_0 PUSH20 <h160>
redeem_script   = witness_program                   # 22 bytes
script_hash     = hash160(redeem_script)            # 20 bytes
addr            = base58.b58encode_check(b'\x05' + script_hash).decode()
# -> "3..."
```

When signing input that spends this address (PSBT):

```
PSBT_IN_NON_WITNESS_UTXO   = full prev tx           # required for HW
PSBT_IN_WITNESS_UTXO       = prev txout
PSBT_IN_REDEEM_SCRIPT      = OP_0 PUSH20 <h160(pk)> # 22 bytes
PSBT_IN_BIP32_DERIVATION   = (pk, fingerprint, m/49h/0h/0h/0/N)
```

The signer produces a 71-73 byte ECDSA sig with BIP143 sighash
(committing to value), placed in `PSBT_IN_PARTIAL_SIG`. Finalizer
emits:

```
final_scriptSig         = OP_PUSH22 <redeemScript>
final_scriptWitness     = [ <sig>, <pk> ]
```

vsize cost vs native:
- Native P2WPKH input: ~68 vB.
- P2SH-P2WPKH input: ~91 vB (extra 23 vB for the 22-byte scriptSig push counted at 4x weight).

## Common bugs / pitfalls

1. **Forgetting the `OP_PUSH22` byte** in scriptSig. The redeemScript
   must be wrapped in a single push opcode; concatenating raw bytes
   produces an invalid scriptSig.
2. **Using uncompressed pubkeys.** BIP143 forbids uncompressed pubkeys
   in P2SH-P2WPKH (consensus-active since SegWit). The output looks
   valid but no signature will satisfy it on chain.
3. **Wrong derivation purpose.** P2SH-P2WPKH MUST use BIP49 (`m/49'/...`).
   Mixing with BIP44 paths breaks coordinator compatibility (Sparrow,
   Trezor Suite, Specter all expect BIP49 for `3...` SegWit addresses).
4. **HW wallet sees malformed PSBT.** Without `PSBT_IN_REDEEM_SCRIPT`,
   the signer cannot reconstruct the witness program; signing fails.
5. **Treating P2SH-P2WSH like P2WSH.** The inner witnessScript still
   needs to be in `PSBT_IN_WITNESS_SCRIPT`, AND the wrapper redeemScript
   in `PSBT_IN_REDEEM_SCRIPT`.
6. **No NODE_WITNESS peer downstream.** Pre-0.16 nodes strip witness
   data when relaying; the wrapped form ensures legacy nodes at least
   relay the tx (they see it as anyone-can-spend P2SH).

## References

- BIP141: https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki
- BIP16: https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki
- BIP49: https://github.com/bitcoin/bips/blob/master/bip-0049.mediawiki
- BIP143: https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki
- bitcoinjs-lib `payments.p2sh({ redeem: payments.p2wpkh(...) })`
