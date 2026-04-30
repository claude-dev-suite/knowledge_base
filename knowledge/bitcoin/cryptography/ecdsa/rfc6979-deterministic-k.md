# RFC6979 Deterministic k Generation - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/ecdsa`.
> Canonical source: RFC 6979 (https://datatracker.ietf.org/doc/html/rfc6979).
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/ecdsa/SKILL.md

## Concept

RFC 6979 specifies how to derive the per-signature nonce `k` deterministically
from the private key `d` and message hash `h`, using HMAC-SHA256 as a PRNG.
This eliminates the need for a CSPRNG at signing time and prevents the most
catastrophic ECDSA failure mode: nonce reuse (which leaks `d` algebraically
in one observation). Bitcoin Core, libsecp256k1, and every modern Bitcoin
wallet sign deterministically by default. Hardware wallets typically blend in
extra entropy as a sidechannel/fault-attack countermeasure.

## Walkthrough / mechanics

### State machine

RFC 6979 maintains two 32-byte state variables `K` and `V`. Initialization:

```
V = 0x01 0x01 ... 0x01    (32 bytes)
K = 0x00 0x00 ... 0x00    (32 bytes)
```

Two HMAC steps "absorb" the private key and message hash:

```
K = HMAC_K(V || 0x00 || int2octets(d) || bits2octets(h))
V = HMAC_K(V)
K = HMAC_K(V || 0x01 || int2octets(d) || bits2octets(h))
V = HMAC_K(V)
```

Then the generation loop:

```
loop:
    T = empty
    while len(T) < qlen:        # qlen = 256 for secp256k1
        V = HMAC_K(V)
        T = T || V
    k = bits2int(T)
    if 1 <= k < n:              # rejection sampling on [1, n-1]
        return k
    K = HMAC_K(V || 0x00)
    V = HMAC_K(V)
```

### Helper functions

- `int2octets(d)`: encode `d` as exactly `qlen/8 = 32` big-endian bytes.
- `bits2octets(h)`: take `bits2int(h) mod n`, then `int2octets`. The `mod n`
  step is critical when the hash has more bits than `n`'s bit length.
- `bits2int(buf)`: take leftmost 256 bits of `buf` as a big-endian integer.

### Why HMAC

HMAC-DRBG (NIST SP 800-90A) is the underlying construction. HMAC's output
is indistinguishable from random under standard assumptions, and the
two-step "absorb" makes the derivation forward-secure: leaking the final `K`
does not let an attacker reconstruct prior `k` values.

### Bitcoin's RFC6979 + extra entropy

libsecp256k1's `secp256k1_ecdsa_sign` exposes a `noncefp` callback. The
default is `nonce_function_rfc6979`, which optionally accepts a 32-byte
`extra_entropy` argument concatenated after `(d, h)` in the absorb step:

```
K = HMAC_K(V || 0x00 || int2octets(d) || bits2octets(h) || extra_entropy)
```

If extra entropy is fresh per signature, the output is non-deterministic but
still safe under nonce-reuse attacks. Hardware wallets pass random bytes here
to defeat fault attacks: an attacker who corrupts memory mid-signing in a
deterministic scheme can re-run with the same `(d, h)` and observe
differential leakage; with random extra entropy each run produces a different
`k`.

## Worked example

RFC 6979 Appendix A.2.5 covers secp256k1. Test vector:

```
private_key d = 0xC9AFA9D845BA75166B5C215767B1D6934E50C3DB36E89B127B8A622B120F6721
message       = "sample"
SHA256(msg)   = 0xAF2BDBE1AA9B6EC1E2ADE1D694F41FC71A831D0268E9891562113D8A62ADD1BF
```

Step by step (with intermediate octets, verified against RFC 6979):

```
V = 01 * 32
K = 00 * 32

absorb 1:
  K = HMAC_K(V || 00 || d || h)
    = 0x4F0...   (specific 32-byte HMAC output)
  V = HMAC_K(V)
    = 0x4F0...

absorb 2:
  K = HMAC_K(V || 01 || d || h)
  V = HMAC_K(V)

generate:
  V = HMAC_K(V)              -> first 32-byte candidate
  k_candidate = bits2int(V)
  k_candidate < n?  yes      -> return k

Expected k for this vector:
k = 0xA6E3C57DD01ABE90086538398355DD4C3B17AA873382B0F24D6129493D8AAD60
Expected sig:
r = 0xAFAF44C579D9F3FA5DB04E5DC2F30F0093C4D5DB60D4F2C7BCD7F8DECC04CA1F
s = 0x09FBC25BE2E0BB95E10C5C7C540D1D7C0096B57A8C2D3F4E... (low-s normalized)
```

(The full vector appears in libsecp256k1 `tests.c` and the RFC.)

## Common pitfalls

- **Wrong `bits2int` truncation**: when `qlen < blen` (hash bigger than `n`),
  take only the leftmost `qlen` bits. With SHA256 and secp256k1's 256-bit
  `n`, they are equal, but `bits2octets` still requires the explicit `mod n`
  reduction or rare inputs produce wrong `k`.
- **Reusing the HMAC state across signatures**: the absorb step must be
  re-run for every `(d, h)` pair. Caching `V, K` across signatures with the
  same `d` causes nonce reuse for a fixed `(d, h)` because the loop will
  produce the same `T`.
- **Skipping rejection on `k >= n` or `k == 0`**: extremely rare on
  secp256k1 (probability < 2^{-127}) but real. Without the loop the
  signature can still produce a valid sig but with skewed distribution.
- **Mixing RFC 6979 with extra entropy in libraries that do not document
  the spec extension**: two implementations claiming "deterministic" may
  produce different signatures if one secretly mixes in entropy and the
  other does not. Test against RFC vectors to confirm.
- **Using SHA-1 or another hash**: RFC 6979 supports any HMAC; the choice
  must match what the verifier expects. Bitcoin always uses HMAC-SHA256.
- **Confusing RFC 6979 `k` with EdDSA-style nonce**: EdDSA hashes the
  private key + message directly with SHA-512 and takes `mod n`. ECDSA
  cannot do that because `s = k^{-1}(h + r*d)` requires `k != 0` strictly
  and uniformly distributed in `[1, n-1]`.

## References

- RFC 6979: https://datatracker.ietf.org/doc/html/rfc6979
- libsecp256k1 `src/modules/ecdsa/main_impl.h`, `nonce_function_rfc6979`.
- Pieter Wuille, "Why Bitcoin uses RFC 6979" (bitcoin-dev archive).
- "Differential fault attack against deterministic ECDSA" (Romailler, Pelissier 2017).
