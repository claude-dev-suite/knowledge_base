# LUD-04 LNURL-auth - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/lnurl`.
> Canonical source: LUD-04 (https://github.com/lnurl/luds/blob/luds/04.md)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/lnurl/SKILL.md

## Concept

LNURL-auth is a stateless, password-less login mechanism: the server issues
a random 32-byte challenge `k1`, the wallet signs it with a key derived
deterministically from (master_seed, server_domain), and the server records
the resulting pubkey as the user's identity. Same wallet + same domain ->
same pubkey on every visit, with no shared secret stored anywhere.

## Walkthrough / mechanics

1. Server generates `k1` (32 random bytes hex), constructs URL:

```
https://service.example.com/login?tag=login&k1=<hex32>&action=login
```

2. URL is bech32-encoded as LNURL, displayed as QR or button.

3. Wallet decodes LNURL, sees `tag=login`, derives the linking key:

```
hashing_key  = hmac_sha256(key="Symmetric Key seed", msg=master_seed_bytes)
domain_path  = first 4 bytes of hmac_sha256(hashing_key, domain_name) interpreted as 4 BIP32 indexes
linking_priv = BIP32_derive(master_xprv, m/138'/<idx0>/<idx1>/<idx2>/<idx3>)
linking_pub  = linking_priv * G
```

4. Wallet signs `k1` with `linking_priv` (ECDSA, low-s, DER encoding):

```
sig = ecdsa_sign(linking_priv, k1)
```

5. Wallet GETs the callback:

```
GET https://service.example.com/login
        ?tag=login
        &k1=<hex32>
        &sig=<DER-hex>
        &key=<linking_pub-hex>
```

6. Server verifies `ecdsa_verify(linking_pub, k1, sig)` and that `k1` is one
   it issued and has not expired. On success, the server treats `linking_pub`
   as the account identifier.

The path `m/138'/.../...` is BIP32 hardened so server-domain knowledge cannot
help an attacker derive other-domain keys.

## Worked example

```
Server issues:
  k1 = "e3a0b3a7d51a4c87b2f97d5d2a1c8a0b9c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3"
  url = https://login.example.com/lnurl?tag=login&k1=<above>&action=register

Wallet derives:
  domain = "login.example.com"
  hashing_key = hmac_sha256("Symmetric Key seed", master_seed)
  derivation_idx = first_16_bytes(hmac_sha256(hashing_key, domain))
  linking_priv = bip32_derive(master, m/138'/i0/i1/i2/i3)

Wallet signs:
  sig = ecdsa_sign(linking_priv, hex_decode(k1))
       = "30440220...0220..."

Wallet calls:
  GET https://login.example.com/lnurl?
      tag=login&k1=e3a0...&sig=304402...&key=02af09...

Server replies:
  { "status": "OK" }
```

Subsequent logins on the same domain produce the SAME `key`, so the server
recognizes the returning user without storing anything wallet-side.

## Common bugs / pitfalls

- Implementing per-key derivation without BIP32 hardening. A non-hardened
  path leaks the master xpub, so a hostile server can compute keys for any
  other domain. ALWAYS use hardened m/138'/...
- Server reusing `k1` across multiple sessions. `k1` MUST be single-use; on
  successful verification, mark it consumed.
- Action mismatch: spec defines `action ∈ {register, login, link, auth}`.
  Some wallets refuse unknown actions; default to `auth`.
- ECDSA malleability: server should require low-s normalized signatures or
  it will accept two different signature encodings for the same message.
- Phishing: a malicious site can present any LNURL with `tag=login`. The
  wallet should display the domain prominently. Wallets that auto-sign
  without UI confirmation are dangerous.
- DNS rebinding: server domain check happens at LNURL-decode time but the
  callback is followed afterwards; wallet must pin the domain.

## References

- LUD-04: https://github.com/lnurl/luds/blob/luds/04.md
- LNURL spec index: https://github.com/lnurl/luds
- Alby Browser Extension LNURL-auth implementation
- BIP-32 hardened derivation
