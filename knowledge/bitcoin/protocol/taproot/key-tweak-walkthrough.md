# Taproot Key Tweak Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/taproot`.
> Canonical source: BIP341 (sections "Constructing and spending Taproot outputs", "Test vectors")
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/taproot/SKILL.md

## Concept

The taproot tweak is the cryptographic glue that binds an internal
public key `P` to a script tree commitment `R`, producing the on-chain
output key `Q = P + t*G` where `t = TaggedHash("TapTweak", P_x || R)`.
The skill quick-ref covers tweak math; this article focuses on the
parity-bookkeeping that trips up implementers, the relationship between
internal-key parity and tweaked-key parity, and how a key-path signer
actually derives the secret.

## Walkthrough / mechanics

BIP340 Schnorr keys are encoded as **x-only** (32 bytes). Every
x-coordinate corresponds to two points on the curve, distinguished by
y-parity. BIP340 fixes the convention "use the even-y point". When
you add the tweak scalar to an x-only key, the resulting point may
have either parity, so we record the parity explicitly:

```
1. Let P_internal be a 32-byte x-only key.
2. P = lift_x(P_internal)             # always even-y point.
3. t = int(TaggedHash("TapTweak", P_internal || R)) mod n
4. Q = P + t*G
5. Q_x = Q.x_bytes()                  # goes into scriptPubKey
6. parity_Q = Q.y is odd ? 1 : 0      # goes into control block byte 0
```

Signing key-path requires the tweaked **secret** `q` such that
`q*G = Q`. Two parity adjustments compose:

```
# given internal secret p with P = p*G
# (p may correspond to odd-y P; we require even-y for x-only)
if not P.has_even_y():
    p = n - p

t = TaggedHash("TapTweak", P_x || R)
q = (p + t) mod n

# now q*G = Q. But Schnorr signing requires the *signing* key to
# correspond to even-y. If Q has odd y:
if not Q.has_even_y():
    q = n - q
```

The control-block parity bit (`byte0 & 1`) is for **verifiers** to
recover the y-coordinate of `Q` during script-path validation. It is
NOT used during key-path signing - the verifier uses `lift_x(Q_x)`
which assumes even y.

## Worked example

Using BIP341 test vector 0 (single key, no scripts):

```
internal_pubkey  = d6889cb081036e0faefa3a35157ad71086b123b2b144b649798b494c300a961d
merkle_root      = ""                         # empty, no taptree

# tweak = TaggedHash("TapTweak", internal_pubkey || "")
tweak            = b86e7be8f24bef6600ad6325c1bbf1d05a0e8a2e4f7c93c69d40efbf3b29c2c8
                   (mod n; for this vector the unreduced hash is below n)

# Q_x in scriptPubKey
output_key       = 53a1f6e454df1aa2776a2814a721372d6258050de330b3c6d10ee8f4e0dda343
parity           = 1                          # odd y

scriptPubKey     = 51 20 53a1f6e454df...da343
```

Python with `python-bitcoinlib`'s key utilities:

```python
from bitcointx.core.key import XOnlyPubKey, tap_tweak_pubkey

internal = XOnlyPubKey(bytes.fromhex(
    "d6889cb081036e0faefa3a35157ad71086b123b2b144b649798b494c300a961d"))
output, parity = tap_tweak_pubkey(internal, b"")
assert output.hex() == "53a1f6e454df1aa2776a2814a721372d6258050de330b3c6d10ee8f4e0dda343"
assert parity == 1
```

For a tree with one leaf:

```python
from bitcointx.core.script import CScript, OP_CHECKSIG, TapLeafScript

leaf_script = CScript([bytes.fromhex("ab" * 32), OP_CHECKSIG])
leaf_hash   = TapLeafScript(0xc0, leaf_script).GetHash()    # TapLeafHash
merkle_root = leaf_hash                                     # single leaf -> root = leaf
output, parity = tap_tweak_pubkey(internal, merkle_root)
```

## Common bugs / pitfalls

1. **Forgetting the parity flip on `p`.** If your internal key was
   generated normally and happens to land on an odd-y point, you must
   negate `p` BEFORE computing the tweak - otherwise `q*G != Q`.
2. **Double-flipping.** Some libraries return the secret already
   negated; calling `tap_tweak_seckey` again on a derived key flips
   it back. Always test against BIP341 vectors.
3. **Using non-tagged SHA256.** TapTweak/TapLeaf/TapBranch all use the
   tagged-hash domain (`SHA256(SHA256(tag) || SHA256(tag) || msg)`).
   A plain SHA256 will compute a syntactically valid but consensus-incompatible value.
4. **Empty merkle root encoding.** When there is no taptree,
   `R = bytes()` (zero-length), NOT 32 zero bytes. Many libraries
   default to a fixed-size zero buffer.
5. **rawtr vs tr in descriptors.** `rawtr(K)` skips the tweak entirely
   - `Q = K`. Use only when `K` is already a committed key (e.g.,
   silent payments). Mixing rawtr with a script tree is meaningless.
6. **Reusing `P` across outputs without different `R`.** Two outputs
   with same `P` and same `R = ""` produce identical `Q` and on-chain
   reuse. Add a per-output salt into the tree, or derive distinct `P`s.

## References

- BIP340: https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
- BIP341: https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
- BIP341 test vectors: https://github.com/bitcoin/bips/blob/master/bip-0341/wallet-test-vectors.json
- python-bitcointx tap utilities: https://github.com/Simplexum/python-bitcointx
- rust-bitcoin `taproot` module: https://docs.rs/bitcoin/latest/bitcoin/taproot/
