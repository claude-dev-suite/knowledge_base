# PSBT Taproot Fields (BIP371) - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/psbt`.
> Canonical source: BIP371
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/psbt/SKILL.md

## Concept

BIP371 extends PSBT with seven new field types so signers can produce
Taproot key-path and script-path signatures and finalizers can build
control blocks. The skill covers field type IDs; this article walks
the byte layout of a real Taproot PSBT input, including how
`PSBT_IN_TAP_LEAF_SCRIPT` is encoded, how multiple script-path sigs
combine, and the rare `PSBT_IN_TAP_MERKLE_ROOT` case.

## Walkthrough / mechanics

The seven input fields:

| Type | Name | Key | Value |
|------|------|-----|-------|
| 0x13 | TAP_KEY_SIG | (none) | 64 or 65 bytes Schnorr |
| 0x14 | TAP_SCRIPT_SIG | xonly_pk (32) + leaf_hash (32) | 64 or 65 bytes Schnorr |
| 0x15 | TAP_LEAF_SCRIPT | control_block (33+32m) | script + 1-byte leaf_version |
| 0x16 | TAP_BIP32_DERIVATION | xonly_pk (32) | varint(num_leaf_hashes) + leaf_hashes + master_fp + path |
| 0x17 | TAP_INTERNAL_KEY | (none) | 32 bytes xonly P |
| 0x18 | TAP_MERKLE_ROOT | (none) | 32 bytes (or 0 bytes for keypath-only) |

Output fields:
| Type | Name | Key | Value |
|------|------|-----|-------|
| 0x05 | OUT_TAP_INTERNAL_KEY | (none) | 32 bytes |
| 0x06 | OUT_TAP_TREE | (none) | encoded tree (depth + leaf_ver + script per leaf) |
| 0x07 | OUT_TAP_BIP32_DERIVATION | xonly_pk | leaf_hashes + fp + path |

The `TAP_LEAF_SCRIPT` value layout is:

```
[ script_bytes... ]                  # variable
[ 1 byte leaf_version ]              # 0xc0 today
```

The script length is implicit from the value's varint length minus 1.
Be careful: the leaf_version is a SUFFIX of the value, while the
control block is the KEY.

`TAP_TREE` for outputs uses a depth-first encoding:

```
For each leaf (in tree-traversal order):
  [ varint depth ]
  [ 1 byte leaf_version ]
  [ varint script_len ]
  [ script_bytes... ]
```

Reading this list lets a tool reconstruct the merkle tree structure
without ambiguity.

## Worked example

Spending a P2TR output with two leaves, using leaf 1 (depth 1).

```
Internal key P_x = aabb...ee   (32 bytes)
Leaf 1 script  L1 = <pkA> CHECKSIG       # 34 bytes
Leaf 2 script  L2 = <144> CSV DROP <pkB> CHECKSIG
TapLeafHash(0xc0, L1) = h1
TapLeafHash(0xc0, L2) = h2
merkle_root           = TapBranchHash(h1, h2)
Q_x                   = lift_x(P) + tweak*G   (32 bytes)
parity_Q              = 0 or 1
```

Control block for spending L1:

```
control_block = bytes([0xc0 | parity_Q]) || P_x || h2     # 65 bytes
```

PSBT input record (selected keys):

```
PSBT_IN_WITNESS_UTXO              -> 8-byte amount + spk(OP_1 PUSH32 Q_x)
PSBT_IN_TAP_INTERNAL_KEY          -> P_x (32)
PSBT_IN_TAP_MERKLE_ROOT           -> merkle_root (32)
PSBT_IN_TAP_LEAF_SCRIPT           -> key=control_block, value=L1 || 0xc0
PSBT_IN_TAP_BIP32_DERIVATION      -> key=pkA, value=varint(1) || h1 || fp || path
PSBT_IN_TAP_BIP32_DERIVATION      -> key=internal_x, value=varint(0) || fp || path
```

Note `value=varint(0)` for the internal key (it doesn't appear in any
leaf). This signals to a signer "this is the internal key, sign with
key-path if requested".

After signing, signer A adds:

```
PSBT_IN_TAP_SCRIPT_SIG            -> key=pkA || h1, value=64-byte schnorr sig
```

Finalizer assembles the witness:

```
final_scriptWitness = [
  <sig_for_L1_from_pkA>,        # stack input to script
  L1,                           # the leaf script
  control_block                  # 65 bytes
]
```

For key-path spend, the witness is just `[<tap_key_sig>]` and only
`PSBT_IN_TAP_KEY_SIG` is consumed by the finalizer.

## Common bugs / pitfalls

1. **Forgetting the leaf-version suffix.** `TAP_LEAF_SCRIPT` value is
   `script || version_byte`. Many implementations originally placed
   the version FIRST; BIP371 fixed this in a late revision. Use
   tested libraries.
2. **`TAP_BIP32_DERIVATION` empty leaf list.** When the derivation
   describes the internal key, encode `varint(0)` followed directly
   by fingerprint+path. Some signers reject if the leaf list is
   present but empty-padded.
3. **Multiple `TAP_LEAF_SCRIPT` for the same leaf.** Allowed. A signer
   that wants to support multiple alternate paths can list each leaf
   it has authority over. The Finalizer picks one at finalization time.
4. **Mixing key-path and script-path sigs.** A finalized witness can
   contain ONE OR THE OTHER, never both. If both `TAP_KEY_SIG` and
   `TAP_SCRIPT_SIG` are present, the Finalizer must choose - typically
   key-path is preferred because it is shorter.
5. **`TAP_MERKLE_ROOT` of zero length vs absent.** A keypath-only
   output should OMIT the field. A 32-byte zero `TAP_MERKLE_ROOT` is a
   different thing (root = `0x00*32` is a valid leaf hash for an empty
   script in some implementations).
6. **`OUT_TAP_TREE` vs no-tree outputs.** If the output has no tree,
   omit `OUT_TAP_TREE`; do not encode an empty list. Wallets reading
   the output must treat absence as "key-path only".

## References

- BIP371: https://github.com/bitcoin/bips/blob/master/bip-0371.mediawiki
- BIP370: https://github.com/bitcoin/bips/blob/master/bip-0370.mediawiki
- BIP341: https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
- bitcoinjs-lib taproot PSBT helpers: https://github.com/bitcoinjs/bitcoinjs-lib
