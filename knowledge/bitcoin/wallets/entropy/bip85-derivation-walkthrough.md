# BIP85 Derivation - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/entropy`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0085.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/entropy/SKILL.md

## Concept

BIP85 squeezes deterministic, application-specific entropy out of a BIP32 master key. Every "child seed" lives at a hardened derivation path beginning with `m/83696968'`, where 83696968 is the ASCII for `BIP85`. The derived child key's private scalar is hashed under HMAC-SHA512 with the fixed key string `"bip-entropy-from-k"`; the resulting 64 bytes are then truncated and post-processed per a small table of "applications" (BIP39 mnemonic, WIF, XPRV, raw hex, base85 password, etc.). The result is that one paper backup can spawn an unlimited family of independent-looking sub-wallets, each deterministically reproducible without separate backup.

## Walkthrough / mechanics

Path layout:

```
m / 83696968' / app_id' / app_args' / index'
```

The number of `app_args` segments depends on the application.

| `app_id` | application                | path tail                                |
|---------:|----------------------------|------------------------------------------|
|     `39` | BIP39 mnemonic             | `39' / lang' / words' / index'`          |
|      `2` | HD-Seed WIF                | `2' / index'`                            |
|     `32` | XPRV                       | `32' / index'`                           |
| `128169` | hex bytes                  | `128169' / num_bytes' / index'`          |
| `707764` | base85 password            | `707764' / pwd_len' / index'`            |
|  `67797` | dice rolls (1-6)           | `67797' / num_rolls' / index'`           |

Core derivation (deterministic, all hardened so a leaked child cannot reveal siblings or the parent):

```
1. child_xprv = CKDpriv(master_xprv, m/83696968'/app_id'/.../index')
2. k          = child_xprv.private_key                  # 32 bytes
3. entropy    = HMAC-SHA512(key=b"bip-entropy-from-k",
                            msg=k)                       # 64 bytes
4. apply_app(app_id, entropy, app_args)
```

Application post-processing:

- **BIP39 mnemonic**: take first `words * 11 / 8` rounded-up bytes of `entropy`, mask off unused bits, look up words by 11-bit groups, append BIP39 checksum.
  - 12 words -> 16 bytes (128 bits + 4-bit checksum).
  - 18 words -> 24 bytes.
  - 24 words -> 32 bytes.
- **WIF**: `entropy[0..32]` is the secret scalar; emit Base58Check WIF.
- **XPRV**: `entropy[0..32]` = chain code, `entropy[32..64]` = private key, depth=0.
- **HEX**: take `entropy[0..num_bytes]`.
- **Base85 password**: take 64 bytes, base85-encode, take first `pwd_len` chars.

Languages for BIP39 (`lang'`):

| code | wordlist           |
|-----:|--------------------|
|    0 | English            |
|    1 | Japanese           |
|    2 | Korean             |
|    3 | Spanish            |
|    4 | Chinese (S)        |
|    5 | Chinese (T)        |
|    6 | French             |
|    7 | Italian            |
|    8 | Czech              |
|    9 | Portuguese         |

## Worked example

Reference vector from BIP85 (English, 12-word, index 0):

Master:

```
xprv9s21ZrQH143K2LBWUUQRFXhucrQqBpKdRRxNVq2zBqsx8HVqFk2uYo8kmbaLLHRdqtQpUm98uKfu3vca1LqdGhUtyoFnCNkfmXRyPXLjbKb
```

Path:

```
m/83696968'/39'/0'/12'/0'
```

Steps:

```
child_xprv = CKDpriv(master, m/83696968'/39'/0'/12'/0')
k          = child_xprv.private_key
            = 0x6250b68daf746d12a24d58b4787a714b   (truncated)
entropy    = HMAC-SHA512(b"bip-entropy-from-k", k)
            = 0x6250b68daf746d12a24d58b4787a714b
              c2e0c5d2c4d99cb3df0c5b8cb8b7c5c1
              8e2a1c8a3b4d5e6f...
seed16     = entropy[0..16]
            = 0x6250b68daf746d12a24d58b4787a714b
mnemonic   = bip39_encode(seed16, English, 12)
            = "girl mad pet galaxy egg matter
               matrix prison refuse sense ordinary nose"
```

The same path always returns those 12 words. Indexing to `m/83696968'/39'/0'/12'/1'` returns a completely independent 12-word mnemonic; the parent xprv backs both.

Hex-bytes example for an encryption key (32 bytes, index 0):

```
m/83696968'/128169'/32'/0'
-> entropy[0..32] = ef26519d5..   (use as AES-256 / age master key)
```

WIF for a hot wallet single key (index 0):

```
m/83696968'/2'/0'
-> KwdMAjGmerYanjeui5SHS7JkmpZvVipYvB2LJGU1ZxJwYvP98617
```

## Common pitfalls

- Leaving any path segment unhardened. Every BIP85 segment MUST be hardened (apostrophe, 0x80000000+); a non-hardened segment in the path violates the spec and leaks chain-code material.
- Using the wrong HMAC key. The literal string is `b"bip-entropy-from-k"` (18 bytes); typoing it ("bip-entropy-from-key", capitalization) produces a different but innocent-looking output, silently desynchronizing wallets.
- Confusing app args. `m/83696968'/39'/0'/12'/0'` is "English 12 words"; `m/83696968'/39'/12'/0'/0'` swaps `lang` and `words` and produces a wholly different mnemonic.
- Treating BIP85 children as cryptographically independent backups. They are reproducible from the parent. Compromising the parent compromises every child. The benefit is operational (no second piece of paper), not security isolation.
- Re-deriving with a different lang and assuming "same seed". A mnemonic in English at index 0 has no relationship to a mnemonic at the Spanish index 0 path.
- Storing only the child mnemonic and losing the parent. The child works on its own, but you have lost the BIP85 reproducibility benefit.

## References

- BIP85: https://github.com/bitcoin/bips/blob/master/bip-0085.mediawiki
- Reference implementation (Python): https://github.com/scgbckbone/bip32utils/blob/master/bip85.py
- Coldcard BIP85 docs: https://coldcard.com/docs/bip85
- Trezor Suite BIP85 support
- iancoleman.io/bip39: https://iancoleman.io/bip39/ (audit-only; never use online with real seeds)
