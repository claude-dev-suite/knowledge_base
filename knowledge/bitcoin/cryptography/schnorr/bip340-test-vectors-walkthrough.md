# BIP340 Test Vectors Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/schnorr`.
> Canonical source: BIP340 test-vectors.csv (https://github.com/bitcoin/bips/blob/master/bip-0340/test-vectors.csv).
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/schnorr/SKILL.md

## Concept

The BIP340 specification ships a CSV of 19 test vectors covering signing,
verification, and a wide range of edge cases. Every implementation must pass
all of them - they catch bugs that ordinary integration tests miss
(parity flips, lift_x corner cases, overflow `s = n`, malformed pubkeys).
This article walks through the categories of vectors, what each is testing,
and what your implementation does wrong if it fails.

## Walkthrough / mechanics

### Vector schema

```
index, secret_key, public_key, aux_rand, message, signature,
verification_result, comment
```

- `secret_key` empty for verification-only vectors.
- `verification_result` is `TRUE` or `FALSE`. `FALSE` rows are
  malformed-input cases that a verifier MUST reject.

### Categories

| Index | Category | What it catches |
|-------|----------|-----------------|
| 0 | Signing, simple even-y, even-y nonce | Baseline path |
| 1 | Signing, even-y secret, even-y nonce, large message | Full-length mod 2^256 |
| 2 | Signing, large secret close to n | Wrapping in `(k + e*d) mod n` |
| 3 | Signing, message all zeros | No empty-message bias |
| 4 | Sign+verify, d at high range, hash near zero | Edge-case interaction |
| 5 | Verify, public key not on curve | lift_x must reject |
| 6 | Verify, has_even_y(R) is false | Parity rule enforcement |
| 7 | Verify, sig[0:32] is not on curve | r is invalid x |
| 8 | Verify, sig[0:32] = 0 (R = O) | Zero-point rejection |
| 9 | Verify, sig[32:64] = 0 (s = 0) | Trivial signature |
| 10 | Verify, sig[0:32] = field prime p | r out of range |
| 11 | Verify, sig[32:64] = group order n | s out of range |
| 12 | Verify, public key = field prime p | Pubkey x must be < p |
| 13 | Verify, message length != 32 | Length check |
| 14 | Verify, valid sig with all-even-y | Sanity TRUE case |
| 15-18 | Various aux_rand values | Deterministic nonce is independent of aux |

### Anatomy of a signing vector

Vector 0 (slightly abridged for legibility):

```
secret    = 0x0000...0003
public    = 0xF9308A019258C31049344F85F89D5229B531C845836F99B08601F113BCE036F9
aux_rand  = 0x0000...0000
message   = 0x0000...0000
signature = 0xE907831F80848D1069A5371B402410364BDF1C5F8307B0084C55F1CE2DCA8215
            25F66A4A85EA8B71E482A74F382D2CE5EBEEE8FDB2172F477DF4900D310536C0
result    = TRUE
```

To recompute:

```
d0 = 3
P = 3 * G   -> some point with given x; note y is even (our public encodes
              x-only, even-y is implicit). If 3*G had odd y, BIP340 would
              negate d0; here we are lucky.
d = d0 (no flip)

t  = d XOR int(tagged_hash("BIP0340/aux", aux_rand))
   = 3 XOR (tagged_hash of zeros)
   = some 256-bit value

rand = tagged_hash("BIP0340/nonce", t || P.x || message)
k0   = int(rand) mod n
k0 != 0? yes.

R = k0 * G
If R.y is odd, k = n - k0; else k = k0.

e = tagged_hash("BIP0340/challenge", R.x || P.x || message) mod n

s = (k + e * d) mod n

sig = R.x.to_bytes(32) || s.to_bytes(32)
```

If your `sig` differs from the vector by even one byte, one of these is
wrong: tagged-hash domain string, parity flip rule, modular reduction. Bisect
by printing intermediate `t`, `rand`, `k0`, `R.x`, `R.y`, `e`, `s`.

## Worked example

Verify-FALSE vector 5 (pubkey not on curve):

```
public    = 0xEEFDEA4CDB677750A420FEE807EACF21EB9898AE79B9768766E4FAA04A2D4A34
message   = 0x0000...0000
signature = some-valid-looking 64 bytes
expected  = FALSE
```

Compute `lift_x(0xEEFDEA4C...)`:

```
y2 = (x^3 + 7) mod p
   = some non-quadratic-residue value
y  = pow(y2, (p+1)//4, p)
check (y * y) % p == y2  -> FALSE
return None
```

A correct verifier returns FALSE here without even computing the challenge.
A subtly broken implementation that skips the residue check might compute a
phantom `y`, derive a phantom `R'`, and return TRUE for a forged signature.
This is exactly the vulnerability vector 5 catches.

### Recommended runner

Pseudocode for a CSV-driven test:

```python
import csv, sys
from your_lib import schnorr_sign, schnorr_verify

passed = failed = 0
for row in csv.DictReader(open("test-vectors.csv")):
    idx = row["index"]; ok = row["verification_result"] == "TRUE"
    pk = bytes.fromhex(row["public_key"])
    msg = bytes.fromhex(row["message"])
    sig = bytes.fromhex(row["signature"])

    if row["secret_key"]:
        sk  = bytes.fromhex(row["secret_key"])
        aux = bytes.fromhex(row["aux_rand"])
        recomputed = schnorr_sign(sk, msg, aux)
        assert recomputed == sig, f"vec {idx} sign mismatch"

    result = schnorr_verify(pk, msg, sig)
    if result != ok:
        print(f"FAIL vec {idx}: got {result}, want {ok}, comment={row['comment']}")
        failed += 1
    else:
        passed += 1
sys.exit(0 if failed == 0 else 1)
```

## Common pitfalls

- **Hex parsing**: trim whitespace; CSV quoting can leave embedded spaces.
- **Empty `secret_key` field**: skip the sign step but still run verify.
- **Message length 0 or != 32 in vector 13**: this MUST fail. Some libs
  silently pad - that hides the bug.
- **Ignoring `aux_rand` in vectors 15-18**: these test that determinism
  holds across different aux values when `(d, m)` are fixed.
- **Using a constant-time-only signer that does not return early on
  invalid input**: fine for production but during testing wrap with a
  variant that surfaces the rejection reason.
- **Tagged-hash typos**: `BIP0340/challenge`, `BIP0340/nonce`, `BIP0340/aux`
  - exact ASCII bytes including the slash. A typo (`BIP340/...` without the
  zero) silently produces a different challenge and breaks all vectors at
  once. If all vectors fail, suspect a tag.

## References

- BIP340: https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
- Test vector CSV: https://github.com/bitcoin/bips/blob/master/bip-0340/test-vectors.csv
- Reference implementation: https://github.com/bitcoin/bips/blob/master/bip-0340/reference.py
- libsecp256k1 `src/modules/schnorrsig/tests_impl.h` for additional vectors.
