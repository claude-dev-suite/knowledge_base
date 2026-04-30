# SIGHASH Deep Walkthrough - Legacy, BIP143, BIP341

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/transactions`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki, https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/transactions/SKILL.md

## Concept

`SIGHASH` decides which parts of a transaction a signature commits to. The mechanism has three generations: legacy (pre-segwit), BIP143 (segwit v0), and BIP341 (taproot). Each fixed real or perceived bugs in the previous, and they differ in *exactly* which fields are double-SHA256-ed and in what order. This article gives the byte-level commitment for each, so you can implement or audit a signing routine.

## Walkthrough / mechanics

### Legacy (pre-BIP143)

```
hash_to_sign = SHA256d( serialize(tx_modified) || hashtype_le32 )

tx_modified construction:
  copy tx
  for each input i:
    set vin[i].scriptSig = ""        (empty)
  set vin[input_index].scriptSig = scriptCode
  apply hashtype rules:
    NONE         -> clear all vout, set sequence=0 except for input_index
    SINGLE       -> truncate vout to [0..input_index]; pad with (-1, "") for indices < input_index; set sequence=0 except for input_index
    ANYONECANPAY -> drop all inputs except input_index
```

Bugs / quirks:
- `SIGHASH_SINGLE` with `input_index >= len(vout)` returns hash `0x000...001` (the famous "SINGLE bug").
- Quadratic hashing: a tx with N inputs produces O(N^2) hashing work - a spam vector.

### BIP143 (segwit v0)

Solves quadratic hashing via cached hashes:

```
hashPrevouts  = SHA256d( concat( txid || vout for each input ) )      (cached)
hashSequence  = SHA256d( concat( sequence_le32 for each input ) )     (cached)
hashOutputs   = SHA256d( concat( serialize(out) for each vout ) )     (cached)

preimage =
    nVersion (4)
    hashPrevouts (32)            zero if ANYONECANPAY
    hashSequence (32)            zero if ANYONECANPAY|NONE|SINGLE
    outpoint (36)                txid || vout of this input
    scriptCode (varint+bytes)    the scriptCode for THIS input
    amount (8)                   value of the prevout being spent
    nSequence (4)
    hashOutputs (32)             zero if SINGLE/NONE; SINGLE -> SHA256d(out[i])
    nLocktime (4)
    hashtype (4)

sighash = SHA256d(preimage)
```

Key change: `amount` is now committed to. Hardware wallets can verify they are signing the value the user expects. The cached hashes make signing N inputs O(N) rather than O(N^2).

### BIP341 (taproot)

Single SHA256 (no double), tagged hash, commits to **all spent prevouts and scriptPubKeys**:

```
sighash = TaggedHash("TapSighash", 0x00 || sigmsg)

sigmsg =
    hash_type (1)               0x00 = DEFAULT, else low 6 bits = ALL/NONE/SINGLE, top bit = ANYONECANPAY
    nVersion (4)
    nLocktime (4)
    sha_prevouts (32)           if !ANYONECANPAY
    sha_amounts (32)            if !ANYONECANPAY
    sha_scriptpubkeys (32)      if !ANYONECANPAY
    sha_sequences (32)          if !ANYONECANPAY
    sha_outputs (32)            if hash_type & 3 == ALL or DEFAULT
    spend_type (1)              bit 0 = annex present; bit 1 = ext_flag (script-path)
    if ANYONECANPAY:
        outpoint (36)
        amount (8)
        scriptPubKey (varint+bytes)
        nSequence (4)
    else:
        input_index (4)
    if annex:
        sha_annex (32)          SHA256(annex with length prefix)
    if hash_type & 3 == SINGLE:
        sha_single_output (32)  SHA256(out[input_index])
    # script-path additions (ext_flag = 1):
        tapleaf_hash (32)
        key_version (1)         always 0x00 today
        codeseparator_position (4)  default 0xffffffff
```

Differences vs BIP143:
- Single, not double, SHA256.
- All four `sha_*` (prevouts, amounts, scriptpubkeys, sequences) are committed - includes scriptPubKeys of every input.
- DEFAULT == ALL but with no trailing 0x01 byte: signature is 64 bytes instead of 65.
- Annex (witness item starting with 0x50) is signed.

## Worked example

Sign input 0 of a 2-input p2wpkh tx with SIGHASH_ALL (BIP143). Suppose:
- input 0 prevout: `aaaa...:0`, amount = 100_000 sat, sequence = 0xfffffffd
- input 1 prevout: `bbbb...:1`, amount = 50_000 sat, sequence = 0xfffffffd
- output 0: 140_000 sat to bc1q...

```
scriptCode = 0x1976a914<pkhash>88ac     (p2pkh template for the input)
hashPrevouts = SHA256d( aaaa..00000000 || bbbb..01000000 )
hashSequence = SHA256d( fdffffff fdffffff )
hashOutputs  = SHA256d( 60ea000000000000 || 16001432... )
preimage = 02000000 || hashPrevouts || hashSequence ||
           aaaa..00000000 || 1976a914<pkhash>88ac ||
           a086010000000000 || fdffffff || hashOutputs ||
           00000000 || 01000000
sighash = SHA256d(preimage)
```

A common implementation (rust-bitcoin):

```rust
let mut cache = SighashCache::new(&tx);
let sighash = cache.p2wpkh_signature_hash(
    0, &script_code, Amount::from_sat(100_000), EcdsaSighashType::All
)?;
```

## Common bugs / anti-patterns

- **scriptCode for P2WPKH must be the `p2pkh` template**, not the p2wpkh scriptPubKey. Using the latter produces a valid-looking but useless signature.
- Using `SHA256d` for taproot - it must be single-SHA256 with the tagged-hash prefix.
- Forgetting to zero `hashPrevouts`/`hashSequence` for `ANYONECANPAY` and `NONE` flags.
- Reusing the same sighash across two different inputs spending the same prevout type: `outpoint` and `amount` change between them.
- Signing without committing `amount`: still seen in custom signers, breaks security after BIP143.
- Forgetting BIP341 covers **all** input scriptPubKeys: a hardware wallet displaying only the current input's address is now lying.
- Annex bit set in `spend_type` but no annex actually present, or vice versa: invalid signature.

## References

- BIP143: https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki
- BIP341 sighash: https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki#signature-hash
- rust-bitcoin SighashCache docs: https://docs.rs/bitcoin/latest/bitcoin/sighash/struct.SighashCache.html
