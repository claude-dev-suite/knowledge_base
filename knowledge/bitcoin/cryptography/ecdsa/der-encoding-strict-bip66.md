# Strict DER Encoding (BIP66) - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/ecdsa`.
> Canonical source: BIP66 (https://github.com/bitcoin/bips/blob/master/bip-0066.mediawiki).
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/ecdsa/SKILL.md

## Concept

ECDSA signatures `(r, s)` are wire-encoded as ASN.1 DER inside Bitcoin
transactions. ASN.1 has multiple equivalent encodings for the same logical
value (different leading zeros, indeterminate-length forms). Pre-BIP66 nodes
accepted any encoding that OpenSSL parsed; this enabled third-party
malleability (a relayer rewrote the DER without invalidating the signature).
BIP66 (soft-forked July 2015) fixed exactly one canonical DER form. The
modern Tapscript spec (BIP342) further restricts to 64-byte raw Schnorr,
but BIP66 still governs every legacy/SegWit-v0 input.

## Walkthrough / mechanics

### The canonical layout

```
30 <total_len>
   02 <r_len> <r_bytes>
   02 <s_len> <s_bytes>
<sighash_byte>          (1 byte, NOT part of DER, appended in tx witness)
```

Concretely:
- `0x30` = SEQUENCE tag.
- `total_len` = single byte, `1 <= total_len <= 70`. Encoded with the short
  form only (no `0x81` long form).
- Each integer is `0x02 <len> <bytes>`.
- `<r_bytes>` and `<s_bytes>` are big-endian, **with one leading 0x00 byte
  if and only if** the high bit of the first content byte is set. This
  prevents misreading a non-negative integer as negative in two's-complement.
- No trailing bytes after `s`.
- `r_len`, `s_len` in `[1, 33]`.

### BIP66 strictness rules (from the BIP)

A signature is invalid (consensus, since soft-fork) if any of the following:

1. Total length of signature (excluding sighash byte) is not in `[9, 73]`.
2. Byte 0 != `0x30`.
3. Byte 1 != `total_len - 2` (length field correct).
4. Byte 2 != `0x02` (r tag).
5. `r_len` is zero.
6. `r_len` makes the buffer overrun.
7. `r_len > 1` AND `r[0] == 0x00` AND `r[1] & 0x80 == 0` (leading zero
   not necessary).
8. `r[0] & 0x80 != 0` (negative encoding - high bit set, no leading zero).
9. Same checks for `s`.
10. Trailing bytes after `s` (other than the sighash byte handled separately).

### Length tables

| field | min | max | typical |
|-------|-----|-----|---------|
| total_len | 8 | 71 | 70 |
| r_len | 1 | 33 | 32 |
| s_len | 1 | 33 | 32 |

The `33` bound covers the case where `r` or `s` has the high bit set and
needs a leading `0x00`. Total length `total_len = 4 + r_len + s_len`. With
typical `r_len = s_len = 32` and one or no leading zero each, total ranges
68 to 70 bytes; plus the `0x30 LL` header = 70-72 bytes; plus 1 sighash byte
= 71-73 in the witness.

### libsecp256k1 strict serializer

```c
secp256k1_ecdsa_signature_serialize_der(
    ctx, output, &output_len, &sig);
// output_len in [70, 72] always. Output is BIP66-canonical.
```

The library also exposes `_parse_der_lax` for recovering historic, malformed
signatures (e.g., from tests, archived data) - never used in consensus paths.

## Worked example

A canonical 71-byte sig with sighash byte:

```
30 44                                  ; SEQUENCE, 0x44 = 68 bytes
   02 20                               ; r INTEGER, 32 bytes
   2B 90 ... 5C 7E                     ; r (high bit clear, no leading 0)
   02 20                               ; s INTEGER, 32 bytes
   1A 33 ... 7F 91                     ; s (high bit clear)
01                                     ; SIGHASH_ALL

Total in witness: 71 bytes (70 DER + 1 sighash).
```

A 72-byte sig where one integer needs the leading-zero pad:

```
30 45                                  ; SEQUENCE, 0x45 = 69 bytes
   02 21                               ; r INTEGER, 33 bytes (high bit set)
   00 EF AB ... 12                     ; leading 0x00 required
   02 20                               ; s INTEGER, 32 bytes
   55 12 ... AB                        ;
01                                     ; SIGHASH_ALL
```

### Failing case (rejected by BIP66)

```
30 46                                  ; SEQUENCE, 70 bytes
   02 21                               ; r 33 bytes
   00 02 ...                           ; leading 0x00 followed by 0x02 -- high bit clear
                                        ; UNNECESSARY leading zero -> RULE 7 fail
   02 20                               ;
   ...
01
```

The leading `0x00` is unnecessary because the next byte's high bit is clear
(`0x02 < 0x80`). Pre-BIP66 nodes accepted this; post-BIP66 they reject.

## Common pitfalls

- **Not stripping the sighash byte before parsing**: the DER parser must be
  fed exactly `len - 1` bytes of the witness. Off-by-one bugs are the most
  common BIP66 mistake.
- **Allowing `r = 0` or `s = 0`**: even if DER-valid, these are not valid
  ECDSA signatures. `r = 0` makes verification reject; `s = 0` is also
  invalid. Reject before signing.
- **Using OpenSSL's `i2d_ECDSA_SIG`**: produces BIP66-canonical output for
  normal cases, but historically had edge-case bugs around negative `r`. Use
  libsecp256k1.
- **Confusing strict-DER (BIP66) with low-s (BIP146)**: BIP66 governs the
  byte layout; BIP146 additionally requires `s <= n/2`. Both are needed for
  a "standard" tx. Tapscript signatures (Schnorr) bypass DER entirely - 64
  bytes flat plus optional sighash.
- **Re-encoding after parsing**: round-trip parsing -> serializing canonical
  may differ from the original even for valid pre-BIP66 sigs. Never relay a
  re-encoded signature thinking it is "the same" - txid changes.
- **Allowing `total_len > 73`**: would mean `r` or `s` is more than 33
  bytes - impossible for valid ECDSA values.

## References

- BIP66: https://github.com/bitcoin/bips/blob/master/bip-0066.mediawiki
- ITU-T X.690 (DER spec): https://www.itu.int/rec/T-REC-X.690
- libsecp256k1 `src/ecdsa_impl.h` `secp256k1_ecdsa_sig_serialize`.
- Bitcoin Core `src/script/interpreter.cpp` `IsValidSignatureEncoding`.
