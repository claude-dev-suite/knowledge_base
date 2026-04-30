# Recoverable ECDSA Signatures - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/ecdsa`.
> Canonical source: SEC1 sect. 4.1.6, BIP137, libsecp256k1 recovery module.
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/ecdsa/SKILL.md

## Concept

A standard ECDSA signature `(r, s)` is verifiable only if the verifier
already holds the public key `Q`. A recoverable signature additionally
encodes a small recovery hint that lets anyone reconstruct `Q` from the
signature and the message hash alone. This is what enables Ethereum's
`ecrecover`, Bitcoin's BIP137 message signing, Lightning gossip, and any
protocol where attaching a 33-byte pubkey alongside the sig is wasteful.
Recoverable signatures are NOT used in raw transaction witnesses - DER
plus a known scriptPubKey already determines `Q`.

## Walkthrough / mechanics

### Math behind recovery

ECDSA verification computes `R = s^{-1}*h*G + s^{-1}*r*Q`. Solving for `Q`:

```
Q = r^{-1} * (s*R - h*G)
```

To run this, the verifier needs:
- `r, s, h` - all available.
- `R` - given only `r = R.x mod n`, the verifier needs the full point.

There can be up to **four** candidate points `R` matching a given `r`:

1. `R` with x-coord = `r`, even y. (recid bit 0 = 0)
2. `R` with x-coord = `r`, odd y.  (recid bit 0 = 1)
3. `R` with x-coord = `r + n`, even y. (recid bit 1 = 1, plus parity)
4. `R` with x-coord = `r + n`, odd y.

Cases 3-4 only exist if `r + n < p`. Since `n < p` and `p - n ~ 2^128`, the
probability of a valid `r + n` is ~2^{-128} - vanishingly small. In practice
recid is one bit (parity of `R.y`); the second bit is reserved.

### recid encoding (canonical Bitcoin/BIP137)

A 65-byte recoverable signature: `<header> r s` where:

```
header = 27 + recid                  (uncompressed pubkey recovery)
header = 27 + recid + 4              (compressed pubkey recovery)
recid in [0, 3]:
    bit 0 = parity of R.y (0 even, 1 odd)
    bit 1 = whether x = r+n (1) or x = r (0)    [almost always 0]
```

So header bytes:
- 27 (uncomp, recid=0), 28 (uncomp, recid=1), 29 (uncomp, recid=2), 30 (uncomp, recid=3)
- 31 (comp, recid=0), 32, 33, 34.

Ethereum uses `27 + recid` directly (always-uncompressed convention) and
calls this `v`.

### Recovery algorithm

```
def recover_pubkey(r, s, h, recid):
    n = curve order
    p = field prime
    x = r if recid & 2 == 0 else r + n
    if x >= p: return None
    parity = recid & 1
    R = lift_x_with_parity(x, parity)            # may be None
    if R is None or R == infinity: return None
    rinv = modinv(r, n)
    e_neg_G = (-h % n) * G                       # use precomputed -G or negate after
    Q = rinv * (s * R + e_neg_G)
    return Q if Q != infinity else None
```

Verify the recovered `Q` is on curve and not infinity. The caller usually
already knows the expected address/pubkey hash and rejects if it does not
match.

## Worked example

Take a known signing tuple:

```
d = 0x1                              # tiny key for illustration
P = G                                # public key

m = 32 bytes of zeros
h = bits2int(m) = 0

k = RFC6979 derives some 256-bit nonce, say k_test.
R = k_test * G    -> let R have R.x = r, even y.
s = k_test^{-1} * (h + r * d) mod n
  = k_test^{-1} * r mod n            (h = 0, d = 1)

recid = 0 (even y, x < n).

To recover P from (r, s, h=0, recid=0):
R = lift_x(r, even)
rinv = r^{-1} mod n
Q = rinv * (s*R - 0*G)
  = rinv * s * R
  = rinv * s * k_test * G
  = rinv * (k_test^{-1} * r) * k_test * G
  = rinv * r * G
  = G                       -> matches P. 
```

Now flip recid to 1 (claim odd y):

```
R' = lift_x(r, odd) = -R
Q' = rinv * (s * R' - 0)
   = -Q
   = -G
```

This recovers a *different* public key. Both are mathematically valid
recoveries; the application chooses by comparing recovered `Q` to the
expected address/scripthash.

## Common pitfalls

- **Treating recid as 1 bit**: many naive implementations only check parity.
  This breaks for the rare `r + n < p` case. libsecp256k1 handles all four
  recids; rolled-from-scratch code often does not.
- **Confusing Ethereum `v` with Bitcoin `header`**: Ethereum: `v = 27 + recid`
  always uncompressed. Bitcoin BIP137: `v = 27 + recid + (4 if compressed)`.
  Mismatching causes "wrong address" recovery silently.
- **Allowing high-s recoveries**: a signature `(r, s)` and `(r, n-s)` with
  flipped recid both verify, recovering the same `Q`. To prevent third-party
  malleability of the encoded blob, normalize `s <= n/2` AND fix recid
  parity before transmission.
- **Recovering with `r = 0` or `s = 0`**: degenerate. Always reject these
  before calling recovery.
- **Trusting recovered key without context**: recovery always succeeds (you
  get *some* point). The caller MUST compare to a known address or
  pubkey-hash. Skipping this turns the recoverable signature into an
  attacker-controlled-public-key oracle.
- **Caching wrong recid for low-s normalization**: when normalizing `s` to
  `n - s`, the recid bit-0 flips (because `R` is replaced by `-R`). Mis-
  flipping breaks recovery silently.

## References

- SEC1 v2 sect. 4.1.6 (Public Key Recovery): https://www.secg.org/sec1-v2.pdf
- BIP137 (Bitcoin Signed Message): https://github.com/bitcoin/bips/blob/master/bip-0137.mediawiki
- libsecp256k1 `src/modules/recovery/main_impl.h`.
- "Notes on ecrecover" (Vitalik Buterin, ethereum-foundation blog 2014).
- BOLT 7 (Lightning gossip), uses recoverable sigs for `node_announcement`.
