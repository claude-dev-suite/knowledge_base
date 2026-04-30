# Xpub + Child-Privkey Leak Attack - Walkthrough

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/bip32`.
> Canonical source: BIP32 sect. "Implications", historical bitcoin-dev posts.
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/bip32/SKILL.md

## Concept

Pure exposition of *the* attack that justifies BIP32's hardened derivation
distinction. We construct a compromise scenario, run the attack on toy
numbers, then discuss real-world variants (sharded multisig with
unhardened branches, shared xpubs in coordination services, leaky
hardware wallets that expose chain codes).

## Walkthrough / mechanics

### Threat model

Adversary has obtained:

- The parent extended public key `xpub_par = (K_par, c_par)` via any of:
  - Watch-only wallet share, descriptor export, multisig coordinator,
    backup file leak, BIP174 PSBT round trip, etc.
- One descendant private key `d_child` from an *unhardened* path under
  `xpub_par`. Realistic sources:
  - An online hot wallet sending a tx (the privkey was loaded into RAM).
  - A chain analysis spike that links a known address to a key-revealed
    burn address.
  - A side-channel on a software wallet leaking one privkey.

Both items must be obtained for the same parent. Either alone is contained.

### The math (recap from BIP32)

Unhardened derivation:

```
data = serP(K_par) || ser32(i)
I    = HMAC-SHA512(c_par, data)
I_L, I_R = I[0:32], I[32:64]
d_child = (I_L + d_par) mod n
```

Both `serP(K_par)` and `c_par` are inside `xpub_par`. The attacker can
compute `I_L` directly. Then:

```
d_par = (d_child - I_L) mod n
```

One HMAC, one modular subtract.

### Recursion - propagating the recovery

After recovering `d_par`, the attacker can:

1. Derive every sibling unhardened child (forward).
2. Derive every grandchild under any unhardened path (forward).
3. If `d_par` is itself an unhardened child of *its* parent, and that
   grandparent's xpub is also exposed, recurse upward. This continues
   until a hardened boundary is hit.

In a wallet using BIP44 with hardening at purpose/coin/account, the
recovery stops at the account level: the account secret is fully
compromised, but master and other accounts remain safe.

### Defense as path discipline

BIP44 path: `m/44'/0'/account'/change/index`. The three apostrophes mark
hardened nodes. Hierarchically:

```
m  ----- (hardened) -----> m/44'  ----- (hardened) -----> m/44'/0'
                                                         |
                                                         (hardened)
                                                         v
                                                m/44'/0'/account'
                                                         |
                                                         (unhardened branches)
                                                         v
                                              m/44'/0'/account'/change/index
```

The attacker who recovers `d_child` at any leaf can only walk back as far
as `m/44'/0'/account'`. The xpub exposed in coordinators/watch-only is
typically the **account xpub** at this level - the recovery contains the
damage to that one account.

## Worked example

Concrete numbers. Let:

```
d_par      = 0x0000...000A                 (the integer 10)
c_par      = 0x0000...0042 padded to 32 bytes        (toy chain code)
K_par      = 10 * G = (some specific point on secp256k1)
serP(K_par)= 02 || K_par.x   (33 bytes, even-y compressed)
```

Compute child `i = 5`:

```
data = serP(K_par) || 0x0000_0005          (37 bytes)
I    = HMAC-SHA512(c_par, data)
I_L  = first 32 bytes (call this value alpha)
I_R  = last 32 bytes
d_child = (alpha + 10) mod n
K_child = (alpha + 10) * G
```

Now play attacker. Given `xpub_par = (K_par, c_par)` and leaked
`d_child = (alpha + 10) mod n`:

```
recompute data    = serP(K_par) || 0x0000_0005       (publicly known)
recompute I_L     = first 32 bytes of HMAC-SHA512(c_par, data)
                  = alpha                            (matches!)
recover  d_par    = (d_child - alpha) mod n
                  = ((alpha + 10) - alpha) mod n
                  = 10                               (the parent secret)
```

Done. Now derive every sibling:

```
For j = 0, 1, 2, ...:
    data_j = serP(K_par) || ser32(j)
    I_L_j  = HMAC-SHA512(c_par, data_j)[0:32]
    d_j    = (I_L_j + 10) mod n
```

Every funded sibling address can now be drained.

### Real-world variant: multisig coordinator

A 2-of-3 coordinator service holds an xpub for each cosigner. If one
cosigner uses a hardware wallet that leaks the privkey for a single
signed input (fault attack, sidechannel), and the cosigner's xpub was
shared with the coordinator - the coordinator can recover the full
account xpub if any unhardened descendants are involved. Modern
descriptor wallets export the account xpub directly (already past the
hardening boundary), so the recovered account is the maximum compromise.

### Real-world variant: chain code leak

If only `c_par` leaks (without `K_par` or any privkey), the attack does
not start. But `c_par` is part of `xpub_par`; export discipline conflates
these. Some hardware wallets historically exported xpub fragments piecemeal
- if both `c_par` and `serP(K_par)` are recoverable (they always are from
xpub Base58 decode), the attack proceeds the moment one leaf privkey leaks.

## Common pitfalls

- **"Watch-only is safe to share widely"**: watch-only is safe in
  *isolation*. Combined with even one leaked descendant privkey, it
  expands the breach. Treat xpubs as sensitive metadata.
- **Sharing the master xpub for "convenience"**: the master xpub is
  catastrophic to share. If any descendant privkey ever leaks (in any
  branch), the entire wallet is compromised.
- **Using unhardened derivation everywhere because it is "simpler"**:
  some toy wallets skip the hardening and produce all addresses from one
  unhardened tree. Any single signed-tx privkey leak compromises all
  addresses.
- **Chain-code reuse across protocols**: if the same chain code seeds
  derivation in two unrelated protocols (e.g., BIP32 + a custom hardened-
  password derivation), a leak in one might transfer.
- **Forgetting BIP85**: BIP85 sub-derivation produces independent seeds.
  The xpub of one BIP85 child does NOT compromise the parent (because
  BIP85 uses hardened derivation throughout).
- **Treating xpub Base58 as opaque**: the Base58Check decode reveals all
  fields including chain code - it is not a hash, it is a reversible
  encoding.

## References

- BIP32 sect. "Implications": https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki#implications
- "BIP32 wallet attacks" (Andrey Polyakov, 2016).
- "Hardened vs non-hardened derivation in HD wallets" (Casa engineering blog).
- BIP44: https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki
- BIP85: https://github.com/bitcoin/bips/blob/master/bip-0085.mediawiki
