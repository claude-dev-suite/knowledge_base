# Trezor Shamir Backup (SLIP-39) Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/trezor`.
> Canonical source: https://github.com/satoshilabs/slips/blob/master/slip-0039.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/trezor/SKILL.md

## Concept

SLIP-39 (Shamir's Secret Sharing for Mnemonic Codes) splits the master
secret into N shares of which any M reconstruct it. Trezor calls this
Shamir Backup. Available on Model T, Safe 3, Safe 5 (NOT Trezor One).

Two structures:

- **Single-group** - simple M-of-N (e.g. 2-of-3, 3-of-5).
- **Group/super-group** - hierarchical: M_g of N groups, each with its
  own M-of-N shares. Allows policies like "2-of-3 family OR 3-of-5
  trustees".

Each share is 20 or 33 SLIP-39 words from a 1024-word wordlist
(distinct from BIP-39's 2048-word list to avoid mixing).

## Walkthrough

Generate Shamir backup on Model T / Safe 3 / Safe 5:

1. Wipe device or pick "Create new wallet".
2. Choose "Advanced backup" -> "Shamir backup".
3. Pick threshold: e.g. 3-of-5 single-group.
4. Device displays each share, one at a time:
   - Acknowledge each word.
   - Confirm two random words for verification per share.
5. Eject between shares so you can move to next paper/metal backup.

Internal flow (firmware):

```
master_secret = TRNG(256 bits)
identifier    = TRNG(15 bits)        # binds shares together
iteration_exp = 1                    # PBKDF2 iterations exponent
encrypt(master_secret, passphrase, identifier, iteration_exp)
  => encrypted_secret
split encrypted_secret into N shares with threshold M (Shamir GF256)
each share: identifier || iteration_exp || group_index || M ||
            share_index || share_value || checksum
encode -> 20 or 33 SLIP-39 words
```

Recovery via Trezor Suite or device:

```bash
# trezorctl recovery (interactive)
trezorctl recovery-device --type shamir
# Device prompts for share count, then asks each share's words
# Threshold reached -> wallet restored
```

## Worked example

3-of-5 Shamir setup with passphrase:

```text
Share 1 (20 words):  academic acid acrobat romp depart...
Share 2 (20 words):  academic acid acrobat romp drink...
Share 3 (20 words):  academic acid acrobat romp dwarf...
Share 4 (20 words):  academic acid acrobat romp easel...
Share 5 (20 words):  academic acid acrobat romp easy...
```

The first three words `academic acid acrobat` are common across all
shares because they encode (identifier, iteration_exp). The 4th word
encodes the share-set parameters. Words 5+ are unique per share.

To recover, enter any 3 of the 5 + the optional passphrase. Order of
shares doesn't matter; device reconstructs deterministically.

Combine with hidden-wallet passphrase for plausible deniability:

```
seed = SLIP-39 master_secret
seed_with_passphrase = PBKDF2(seed || passphrase)
```

## Common pitfalls

- **SLIP-39 wordlist != BIP-39**: never mix. A SLIP-39 share entered
  as BIP-39 yields garbage seed.
- **Threshold below ceiling**: if you set 2-of-3 you need EXACTLY 2;
  having all 3 doesn't add safety. If one share is lost, 2 still
  recover.
- **Group structure** with single trusted group reduces to plain M-of-N;
  the complexity only pays off with multiple custody parties.
- **No interop with non-Trezor**: Coldcard, BitBox, Ledger don't
  natively read SLIP-39 (Coldcard added partial import recently).
  Plan recovery with another Trezor or `shamir-mnemonic` CLI.
- **Identifier collision** if you naively re-roll - the firmware
  generates a fresh identifier each time, so old shares can't be
  mixed with new ones.

## References

- SLIP-39: https://github.com/satoshilabs/slips/blob/master/slip-0039.md
- Reference impl: https://github.com/trezor/python-shamir-mnemonic
- Trezor Suite docs: https://trezor.io/learn/a/what-is-shamir-backup
