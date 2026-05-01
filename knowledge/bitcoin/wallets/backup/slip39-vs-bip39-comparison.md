# SLIP-39 vs BIP39 - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/backup`.
> Canonical source: https://github.com/satoshilabs/slips/blob/master/slip-0039.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/backup/SKILL.md

## Concept

BIP39 turns 128-256 bits of entropy plus a checksum into a single mnemonic; recovering the wallet requires every word, in order. SLIP-39 (Trezor's "Shamir Backup") splits a master secret into M-of-N shares using Shamir's Secret Sharing over GF(256), each share encoded as a different mnemonic from a 1024-word wordlist that is intentionally disjoint from BIP39's. SLIP-39 also supports a two-level "group" scheme (e.g., need 2 of 3 family shares AND 2 of 5 board shares) so threshold structures can mirror real-world authority. Critically, SLIP-39 derives the BIP32 master seed slightly differently than BIP39, so the same entropy produces different addresses; SLIP-39 and BIP39 mnemonics are NOT interchangeable even when carrying identical bytes.

## Walkthrough / mechanics

BIP39 pipeline:

```
entropy (128/160/192/224/256 bits)
  + 4-32 checksum bits = SHA256(entropy)[..n]
  -> split into 11-bit groups
  -> map each group to wordlist[0..2047]
  -> join with spaces
mnemonic + "mnemonic" + passphrase --PBKDF2-HMAC-SHA512--> 64-byte seed
seed -> HMAC-SHA512("Bitcoin seed", seed) -> BIP32 master
```

SLIP-39 pipeline:

```
master_secret (128/256 bits)
  + identifier (15 bits, random)
  + iteration_exponent (4 bits)
  -> encrypt with Feistel of PBKDF2(passphrase) -> encrypted_master_secret
  -> Shamir-split over GF(256) into N shares (T-of-N threshold)
  -> each share = (group_idx, member_idx, threshold, value)
  -> each share serialized to 20/33 mnemonic words (10-bit groups)
mnemonic_share words come from SLIP-39 wordlist (1024 words, disjoint from BIP39)
recovered_master_secret -> BIP32 master seed (no PBKDF2 stretch)
```

Wordlist differences:

- BIP39: 2048 words, 11 bits each, words have unique 4-letter prefixes within the same language list.
- SLIP-39: 1024 words, 10 bits each, words are 4-8 letters, designed for low confusability when spoken.

Consequence: a BIP39 mnemonic word like "abandon" is NOT in SLIP-39, and a SLIP-39 word like "academic" is NOT in BIP39. Any wallet "auto-detect" must distinguish format by trying both wordlists.

Group structure (SLIP-39 only):

```
group_threshold = T_g            (e.g., 2 groups required)
groups = [
  (member_threshold = 2, members = 3),   # group 0: family, 2-of-3
  (member_threshold = 3, members = 5),   # group 1: board,  3-of-5
]
recovery: collect at least T_g groups, each with its member_threshold
```

## Worked example

BIP39, 12 words at 128 bits:

```
entropy = 0x12345678901234567890123456789012
mnemonic = "begin friend black ridge sword champion sand hello chuckle stove
            galaxy snow"
```

SLIP-39, single-group 2-of-3 of the same 128-bit master_secret:

```
master_secret = 0x12345678901234567890123456789012
identifier    = 0x1234     (15 bits)
iteration_exp = 0
threshold     = 2
total_shares  = 3
passphrase    = ""

share[0] = "academic acid acrobat romp activity device daughter device
            criminal coastal disaster decision election crystal exchange
            charity ceiling charity diet acrobat"

share[1] = "academic acid beard romp away adapt easel eyebrow fancy
            element frost forecast genuine corner ecology drift family
            diet ceiling anatomy"

share[2] = "academic acid ceramic romp burning artwork fishing fawn
            galaxy entrance evidence frequent exact craft eraser
            deny ceramic ceiling cradle activity"
```

Any 2 of those 3 reconstruct `master_secret`. None of the 20 words appears in BIP39's list. Each share starts with `academic acid` because the first two words encode `(identifier, iteration_exponent, threshold)` shared across the group; only the third word onward is per-share.

Recovering as a Bitcoin wallet:

```python
shares = [share[0], share[2]]
ms = slip39.combine(shares, passphrase=b"")
seed = ms                       # 16 or 32 bytes
master = bip32.from_seed(seed)  # NOTE: no PBKDF2 stretch, unlike BIP39
addr0 = master.derive("m/84'/0'/0'/0/0").address  # bc1q...
```

The same `master_secret` typed as a BIP39 mnemonic ("begin friend ...") would feed into PBKDF2-HMAC-SHA512 first and produce a completely different `addr0`. The two mnemonic systems produce different wallets even when carrying identical entropy.

## Common pitfalls

- Trying to type SLIP-39 words into a BIP39 box (or vice versa). Some wallets accept "wordlist auto-detect" but most do not; recovery silently produces an empty wallet because the first word ("academic") is not in BIP39 -> partial guess succeeds with garbage.
- Confusing share threshold T with total shares N. With T=2, ANY 2 of the N shares recover. With T=N, all shares are required (and the scheme degrades to a multi-piece backup with no redundancy).
- Sharing only T shares (no margin). If T=2 and you make exactly 2 shares, losing one is unrecoverable. The whole point is N > T.
- Forgetting the SLIP-39 passphrase. SLIP-39's passphrase, like BIP39's, is NOT in the shares. Lose it and the shares decrypt to a different master_secret.
- Mixing groups. Hard to reconstruct: the 2-of-3 family group and 3-of-5 board group must each independently reach quorum; you cannot substitute extra family shares for missing board shares.
- Believing SLIP-39 makes you safer against a single thief. With a 2-of-3, a thief who finds two shares can reconstruct. Geographic separation matters more than the cryptography.
- Forgetting that SLIP-39 derivation skips PBKDF2. Wallets that import SLIP-39 must NOT run BIP39's PBKDF2 step on the recovered seed; addresses will all be wrong.

## References

- SLIP-39: https://github.com/satoshilabs/slips/blob/master/slip-0039.md
- BIP39: https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki
- python-shamir-mnemonic: https://github.com/trezor/python-shamir-mnemonic
- Trezor Suite Shamir docs: https://trezor.io/learn/a/what-is-shamir-backup
- Keystone partial SLIP-39 doc
