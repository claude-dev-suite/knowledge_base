# BIP32 Normal vs Hardened Derivation - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/bip32`.
> Canonical source: BIP32 (https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki).
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/bip32/SKILL.md

## Concept

BIP32 defines two child-key derivation modes that differ only in what input
seeds the HMAC-SHA512 step. Normal (unhardened) derivation lets a holder of
the parent xpub derive child xpubs without secret material - the foundation
of watch-only wallets. Hardened derivation hashes the parent's secret key
directly, breaking the public-derivation path. This article focuses on the
*algebraic* relationship between the two modes, the security properties
each provides, and the precise threat model where they differ.

## Walkthrough / mechanics

### The HMAC inputs side by side

```
Normal child (i < 2^31):
    data = serP(K_par) || ser32(i)
        K_par = compressed pubkey of d_par   (33 bytes)
        ser32 = big-endian uint32 of i

Hardened child (i >= 2^31):
    data = 0x00 || ser256(d_par) || ser32(i)
        ser256 = big-endian 32-byte secret
```

In both cases:

```
I = HMAC-SHA512(key=c_par, msg=data)   (64 bytes)
I_L = I[0:32]   I_R = I[32:64]
d_child = (I_L + d_par) mod n
c_child = I_R
```

### The key linearity that enables xpub derivation

For normal derivation, the public key satisfies:

```
K_child = (I_L + d_par) * G
       = I_L * G + d_par * G
       = I_L * G + K_par
```

Anyone holding `K_par` and `c_par` (i.e., the xpub) can compute
`I_L = HMAC(c_par, serP(K_par) || ser32(i))[0:32]` and then
`K_child = I_L * G + K_par`. No secret material involved. This is the basis
of HD watch-only wallets, BIP44/49/84/86 receive address generation in
view-only mode, and PSBT workflows where an air-gapped signer holds keys
but a hot wallet generates addresses.

For hardened derivation, the input includes `ser256(d_par)`. Without `d_par`,
HMAC is a PRF and `I_L` is unpredictable - cannot be computed from xpub
alone.

### The xpub-leak attack (defeated by hardened derivation)

Threat model: attacker has obtained:
1. The parent xpub `(K_par, c_par)`.
2. Some unhardened child's secret key `d_child`.

For unhardened child `i`:

```
d_child = (I_L + d_par) mod n
I_L     = HMAC(c_par, serP(K_par) || ser32(i))      (publicly computable)
=> d_par = (d_child - I_L) mod n
```

The attacker recovers the parent's secret key. From `d_par` and `c_par`,
they derive every sibling and descendant. **Result**: leaking even one
unhardened child's privkey + the parent's xpub equals leaking the whole
parent subtree.

This is why BIP44/49/84/86 mandate hardening at the account level
(`m/purpose'/coin'/account'`). The xpub at the account level can be safely
exported because:
- An attacker who steals the xpub plus any descendant unhardened privkey
  recovers only the *account* secret, not the master.
- Hardened derivation between master and account ensures the master
  remains protected.

### Why both modes coexist

Hardened-only derivation would block xpub-based address generation
entirely. Normal-only derivation would expose the full master key to xpub
leak. The compromise: harden at boundaries (purpose, coin, account), use
unhardened below (change branch, address index) so a single xpub generates
unlimited addresses without separate communication.

### Index range encoding

BIP32 sets index bit 31 as the hard/soft flag:

```
i in [0, 2^31)        -> normal (a.k.a. unhardened)
i in [2^31, 2^32)     -> hardened
```

Path notation: `0` means index 0 unhardened; `0'` or `0h` means index
`0 + 2^31`. So `m/44'/0'/0'/0/5` translates to indices
`(2^31 + 44, 2^31 + 0, 2^31 + 0, 0, 5)`.

## Worked example

Set up a tiny case with `d_par = 1`, `c_par = 32 bytes of 0x42`.

### Normal child, i = 0

```
K_par = 1 * G = G          (compressed: 02 || G.x)
data  = 02 || G.x || 00 00 00 00      (37 bytes)
I     = HMAC-SHA512(key = 0x42 * 32, msg = data)
I_L   = first 32 bytes of I
I_R   = last 32 bytes of I

d_child = (I_L + 1) mod n
K_child = K_par + I_L * G
        = G + I_L * G
        = (1 + I_L) * G   ✓
c_child = I_R
```

### Hardened child, i = 2^31 (written 0')

```
data  = 00 || 00...01 || 80 00 00 00     (1 + 32 + 4 = 37 bytes)
I     = HMAC-SHA512(0x42 * 32, data)
d_child = (I_L + 1) mod n
c_child = I_R

K_par's bytes are NOT in `data`. The xpub holder cannot compute I_L.
```

### Demonstrating the xpub-leak attack with concrete numbers

Suppose attacker has:
- `K_par = G`, `c_par = 0x42*32`.
- `d_child` for unhardened child 0 (somehow leaked - perhaps via a
  compromised hot wallet that signed a transaction with `d_child`).

Attacker computes:
```
data = 02 || G.x || 00 00 00 00
I    = HMAC-SHA512(0x42*32, data)   (publicly computable - they know c_par)
I_L  = first 32 bytes of I

d_par = (d_child - I_L) mod n
```

With one HMAC and one mod-subtract, attacker has the parent's secret key.
This is why every modern wallet hardens between master and account.

## Common pitfalls

- **Exporting an xpub above the account level**: e.g., exporting
  `m/84'/0'` xpub. If any granddaughter privkey leaks, the account secret
  is recoverable. Always export at hardened-leaf level.
- **Using non-hardened "purpose"**: writing `m/84/0'/0'` instead of
  `m/84'/0'/0'` - works mechanically but defeats BIP44 separation.
- **Multi-account wallets sharing the master xpub** with view-only peers:
  same root attack surface across all accounts. Use per-account xpubs.
- **Cross-coin reuse with same seed**: BIP44 isolates by coin index but
  if you also derive `m/44'/0'/0'/0/5` and `m/44'/2'/0'/0/5` (BTC and
  Litecoin) from the same seed, leaking either coin's xpub does not leak
  the other - because each coin's derivation hardens at coin level.
- **Confusing `2^31` boundary in code**: indices are uint32 with the
  high bit as the hard flag. Off-by-one (`>=` vs `>`) at `2^31 - 1` vs
  `2^31` is a common bug.
- **Treating `m` as derivable**: `m` is the master and has no parent. Some
  libraries error with "no parent" if `m/` is requested as a child path.
- **Forgetting the chain code in the leak attack**: the attack requires
  *both* the xpub (which includes `c_par`) and a child secret. Leaking the
  child secret alone is contained.

## References

- BIP32: https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
- "Derivation of HD wallet keys" (Pieter Wuille, 2013-09 bitcoin-dev).
- Greg Maxwell, "BIP32 ParentXPub + child priv = parent priv" exposition (2014).
- BIP44 / BIP49 / BIP84 / BIP86 specifications.
