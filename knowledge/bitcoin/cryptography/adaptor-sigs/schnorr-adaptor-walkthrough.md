# Schnorr Adaptor Signature Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/adaptor-sigs`.
> Canonical source: Aumayr et al. "Generalized Channels" / Poelstra blog series
>                   "Scriptless Scripts" + secp256k1-zkp adaptor module
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/adaptor-sigs/SKILL.md

## Concept

A Schnorr adaptor signature is a tweak of the BIP340 signing equation that
defers the "completion" of the signature on a chosen secret scalar `t`. The
signer publishes a **pre-signature** `(R, s')` that fails normal BIP340
verification but passes a modified verifier that checks against `R + T` where
`T = t * G`. Any holder of `t` can convert the pre-sig into a valid sig with a
single scalar add. Conversely, observing the completed sig reveals `t = s - s'`.
This article gives the precise group-element and scalar arithmetic for the
three primitives — `PreSign`, `PreVerify`, `Adapt`, `Extract` — and works a
hex example to a libsecp256k1-zkp test vector.

## Walkthrough / mechanics

### Setup

```
Signer: secret d, pubkey P = d * G                (BIP340 even-y normalized)
Adaptor: t, T = t * G                              (chosen by some party,
                                                   not necessarily the signer)
Message: m (32 bytes)
```

### PreSign(d, m, T)

```
1. Sample nonce k <- {1..n-1} (deterministic per BIP340 aux-rand recommended)
2. R0 = k * G
3. R  = R0 + T                                     // adaptor offset
4. e  = TaggedHash("BIP0340/challenge",
                   xbytes(R) || xbytes(P) || m) mod n
5. d' = d if has_even_y(P) else n - d
6. s' = (k + e * d') mod n
7. return pre_sig = (R0, s')                       // both 32 bytes; R0
                                                   // *not* R is published
```

Note: published `R0` is the nonce *without* the adaptor offset. Some library
APIs publish `R = R0 + T` instead and ship `s'` only — it depends on the
encoding choice (see "encoding variants" below).

### PreVerify(P, m, T, pre_sig=(R0, s'))

```
1. R = R0 + T
2. e = TaggedHash("BIP0340/challenge", xbytes(R) || xbytes(P) || m) mod n
3. P_eff = P if has_even_y(P) else lift_x(P) with even y forced
4. check: s' * G == R0 + e * P_eff
5. (Note: equivalently, s' * G == R - T + e * P_eff)
```

If the equality holds, the pre-sig is well-formed. The verifier learns
nothing about `t`.

### Adapt(pre_sig=(R0, s'), t)

```
s = (s' + t) mod n
return final_sig = (xbytes(R0 + T), s)             // BIP340-formatted
```

The R component published in `final_sig` is the x-coordinate of `R0 + T` —
which we already know is `R` from PreSign. So the final sig is exactly
`(R, s)` where `s = k + t + e * d`.

### Extract(pre_sig=(R0, s'), final_sig=(R, s))

```
t = (s - s') mod n
verify: t * G == T          // sanity check
return t
```

Extracting `t` requires knowing both `s'` (from the pre-sig) and `s` (from
the published completed sig). Anyone who saw the pre-sig and observes the
completed sig on chain can run `Extract`.

### Why the verifier sees a normal Schnorr sig

After `Adapt`:
```
s * G = (s' + t) * G
      = s' * G + t * G
      = (R0 + e * P_eff) + T          // using PreVerify equation
      = R + e * P_eff
```

This is exactly the BIP340 verification equation `s * G == R + e * P` (with
parity normalization). The verifier has no idea anything special happened.

### Encoding variants

```
Variant A (BIP-340-style, secp256k1-zkp default):
  pre_sig = R0 (32 bytes) || s' (32 bytes)
  T sent separately

Variant B (Aumayr / "preadaptor" form):
  pre_sig = (R = R0 + T) (32 bytes) || s' (32 bytes) || T (33 bytes)
  Only T need be tracked separately; R encodes the offset

Variant C (DLC encoding, dlcspecs):
  s' (32 bytes) only, with R derived from oracle nonce + outcome point
```

Cross-implementation interop requires nailing down which variant your library
uses. secp256k1-zkp's `secp256k1_schnorrsig_pre_sign` returns Variant A.

## Worked example

Synthetic vector inspired by `secp256k1-zkp` adaptor sig tests. We pick small
example scalars (real ones are 256-bit) for arithmetic clarity:

```
n          = 23                  (toy group order; secp256k1's n is huge)
G is the generator.

Signer:
  d        = 5
  P        = 5G       (assume even-y for the example)

Adaptor:
  t        = 11
  T        = 11G

Message:
  m        = 0x...   (used inside hash; treat as fixed input)

PreSign:
  k        = 7
  R0       = 7G
  R        = R0 + T = 7G + 11G = 18G
  e        = H(xbytes(18G) || xbytes(5G) || m) mod 23     -> say e = 9
  s'       = (k + e * d) mod n
           = (7 + 9 * 5) mod 23
           = (7 + 45) mod 23
           = 52 mod 23 = 6
  pre_sig  = (R0=7G, s'=6)

PreVerify:
  Check 6 G ?= R0 + e P_eff
              = 7G + 9 * 5G = 7G + 45G = 52G mod 23 = 6G   OK

Adapt(pre_sig, t=11):
  s = (s' + t) mod n = (6 + 11) mod 23 = 17
  final_sig = (xbytes(R = 18G), s = 17)

BIP340 Verify:
  s G ?= R + e P
  17 G ?= 18 G + 9 * 5 G
       = 18 G + 45 G = 63 G mod 23 = 17 G       OK

Extract(pre_sig, final_sig):
  t = (s - s') mod n = (17 - 6) mod 23 = 11      matches our chosen t

Sanity: 11 G == T?  yes.
```

The same arithmetic, run on secp256k1's actual `n`, produces 32-byte
big-endian hex strings; the libsecp256k1-zkp tests assert byte equality.

## Common pitfalls

- **Forgetting parity normalization on P.** BIP340 mandates even-y; if `P`
  has odd-y, the signer flips `d` to `n - d` for both PreSign and PreVerify.
  Skipping this flip produces a sig that fails verification.
- **Forgetting parity on R.** BIP340 `R` must have even-y too. If `R0 + T`
  has odd-y, the signer must flip `(k + t)` together — handled by flipping
  `s'` and the adaptor secret coordinately. Each library has its own
  convention; misalignment is a frequent interop bug.
- **Reusing nonces.** Same as plain Schnorr: same `k` for two different
  messages leaks `d`. Same `k` with two different `T` values leaks
  `t1 - t2 = s1' - s2'` if the verifier sees both pre-sigs.
- **PreVerify shortcut error.** A pre-sig `(R0, s')` that "looks" valid but
  whose `R0 + T` lifts to a non-curve point is a malformed tweak — reject
  before relying on it.
- **Encoding mismatch (R0 vs R).** Variant A's `R0` is the bare nonce;
  Variant B's `R` includes the adaptor. Mixing them produces a 50% chance of
  apparent verification (when `T = O`) and otherwise fails silently.

## References

- Poelstra, "Scriptless Scripts" (2017):
  https://download.wpsoftware.net/bitcoin/wizardry/mw-slides/2017-mit-bitcoin-expo/slides.pdf
- Aumayr et al., "Generalized Channels from Limited Blockchain Scripts and
  Adaptor Signatures": https://eprint.iacr.org/2020/476
- secp256k1-zkp adaptor sig module:
  https://github.com/BlockstreamResearch/secp256k1-zkp/tree/master/src/modules/ecdsa_adaptor
- Erwig et al., "Two-Party Adaptor Signatures": https://eprint.iacr.org/2021/426
