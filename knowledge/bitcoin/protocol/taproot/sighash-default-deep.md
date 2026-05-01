# SIGHASH_DEFAULT and BIP341 Sighash Internals - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/taproot`.
> Canonical source: BIP341 section "Common signature message"
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/taproot/SKILL.md

## Concept

`SIGHASH_DEFAULT` (0x00) is exclusive to Taproot and equivalent to
SIGHASH_ALL semantically, but compresses the signature from 65 to 64
bytes by omitting the trailing sighash byte. The skill quick-ref
covers what's committed; this article walks the digest construction
field-by-field, explains the script-path extension (ext_flag), and
shows how SINGLE/NONE/ANYONECANPAY interact with the digest.

## Walkthrough / mechanics

The BIP341 sighash digest is built from these concatenated pieces
hashed with `TaggedHash("TapSighash", ...)`:

```
0x00                                  # epoch (constant)
hash_type                             # 1 byte: 0x00 (DEFAULT) or 0x01..0x83
nVersion                              # 4 bytes LE
nLockTime                             # 4 bytes LE

# When (hash_type & 0x80) == 0 (NOT ANYONECANPAY):
sha256(serialized_prevouts)           # 32 bytes; sequence of all 36-byte outpoints
sha256(serialized_amounts)            # 32 bytes; all spent amounts (8 bytes each)
sha256(serialized_scriptpubkeys)      # 32 bytes; concat of (varint len + spk)
sha256(serialized_sequences)          # 32 bytes; all 4-byte sequences

# When (hash_type & 0x03) != SINGLE && != NONE:
sha256(serialized_outputs)            # all outputs

spend_type                            # 1 byte: ext_flag<<1 | annex_present
                                      #   ext_flag=0 keypath, =1 scriptpath

# If ANYONECANPAY:
outpoint                              # 36 bytes
amount                                # 8 bytes LE
scriptpubkey                          # varint + bytes
nSequence                             # 4 bytes LE
# Else:
input_index                           # 4 bytes LE

# If annex present:
sha256(annex)

# If SINGLE:
sha256(this_output)

# If script-path (ext_flag=1):
tapleaf_hash                          # 32 bytes
key_version                           # 1 byte (always 0x00 for BIP342)
codeseparator_position                # 4 bytes LE (default 0xffffffff)
```

`ext_flag = 0` for key-path; `ext_flag = 1` for script-path. Same
function, two callers.

## Worked example

Single-input, single-output P2TR key-path spend with SIGHASH_DEFAULT.
Spent prevout: 100_000 sats at scriptPubKey `OP_1 PUSH32 <Q_x>`.

```python
from hashlib import sha256

def tagged(tag, msg):
    h = sha256(tag.encode()).digest()
    return sha256(h + h + msg).digest()

def ser_compactsize(n):
    if n < 0xfd:    return bytes([n])
    if n < 0x10000: return b'\xfd' + n.to_bytes(2, 'little')
    if n < 0x100000000: return b'\xfe' + n.to_bytes(4, 'little')
    return b'\xff' + n.to_bytes(8, 'little')

# inputs to digest
nVersion   = (2).to_bytes(4, 'little')
nLockTime  = (0).to_bytes(4, 'little')
hash_type  = b'\x00'
epoch      = b'\x00'

# single input/output for clarity
prevout_buf  = bytes.fromhex("aa"*32 + "00000000")          # txid + vout=0
amount_buf   = (100_000).to_bytes(8, 'little')
spk_buf      = ser_compactsize(34) + bytes.fromhex("5120" + "bb"*32)
seq_buf      = (0xffffffff).to_bytes(4, 'little')
output_buf   = (99_500).to_bytes(8, 'little') + ser_compactsize(34) + bytes.fromhex("5120" + "cc"*32)

sha_prevouts = sha256(prevout_buf).digest()
sha_amounts  = sha256(amount_buf).digest()
sha_spks     = sha256(spk_buf).digest()
sha_seqs     = sha256(seq_buf).digest()
sha_outputs  = sha256(output_buf).digest()

spend_type   = bytes([0])           # ext_flag=0, no annex
input_index  = (0).to_bytes(4, 'little')

m  = epoch + hash_type + nVersion + nLockTime
m += sha_prevouts + sha_amounts + sha_spks + sha_seqs
m += sha_outputs
m += spend_type
m += input_index

digest = tagged("TapSighash", m)
# Sign with Schnorr: sig = schnorr_sign(seckey, digest)  # 64 bytes
# witness = [sig]
```

For a script-path spend on the same tx with leaf hash `L`:

```python
spend_type   = bytes([2])           # ext_flag=1 -> 1<<1, no annex
m += sha_prevouts + sha_amounts + sha_spks + sha_seqs
m += sha_outputs
m += spend_type
m += input_index
m += L                              # tapleaf_hash
m += b'\x00'                        # key_version
m += (0xffffffff).to_bytes(4, 'little')   # CODESEPARATOR pos
digest = tagged("TapSighash", m)
```

## Common bugs / pitfalls

1. **Hashing ALL prevouts even with ANYONECANPAY.** ANYONECANPAY drops
   the four "sha_*" sums for prevouts/amounts/spks/sequences and
   replaces them with this input's data only. Forgetting to switch the
   schema produces an invalid sig.
2. **Missing per-input amounts.** A signer that doesn't have prevout
   amounts cannot compute `sha_amounts`. PSBT v0 must include
   `PSBT_IN_WITNESS_UTXO` for every taproot input - not just the one
   being signed.
3. **Wrong epoch byte.** Epoch is a literal `0x00` constant, not a
   counter. Future BIPs may bump it; today every Taproot sighash
   starts with 0x00.
4. **CompactSize encoding of scriptPubKey.** Each spk in `sha_spks` is
   serialized as `varint(len) || bytes`. A 34-byte P2TR spk is
   `0x22 || 0x51 0x20 || <32>`. Forgetting the varint costs you the
   leading 0x22.
5. **Mixing 0x00 and 0x01.** SIGHASH_DEFAULT and SIGHASH_ALL produce
   IDENTICAL digests but DIFFERENT witnesses (64 vs 65 bytes). Verifiers
   accept both for ALL semantics, but if you sign with 0x01 you MUST
   append the 0x01 byte to the witness sig. BIP322 simple form
   recommends DEFAULT for Taproot.
6. **Annex byte handling.** An annex starts with 0x50 in the witness;
   if present, set `spend_type |= 1` and append `sha256(annex)` to
   the digest. Every annex you don't expect MUST still be committed.

## References

- BIP341: https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
- BIP342: https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki
- BIP341 reference Python in repo: https://github.com/bitcoin/bips/blob/master/bip-0341/wallet-test-vectors.json
- rust-bitcoin `sighash::TapSighashType`: https://docs.rs/bitcoin/latest/bitcoin/sighash/
