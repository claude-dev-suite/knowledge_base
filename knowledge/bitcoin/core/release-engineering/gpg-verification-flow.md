# GPG Verification Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/release-engineering`.
> Canonical source: https://bitcoincore.org/en/download/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/release-engineering/SKILL.md

## Concept

Verifying a Bitcoin Core download is a two-step proof. First, you confirm that a `SHA256SUMS` file genuinely came from a maintainer or a builder you trust, by checking a GPG signature on it. Second, you confirm that the binary you have hashes to a value listed in that file. Skipping either step is a no-op: an attacker who modifies your binary will also modify the hash file unless GPG protects it; an attacker who forges a signature without rebuilding the binary will be caught by the hash mismatch. The non-obvious part is which key to trust. Bitcoin Core has many builders, the keyring evolves, and naive `gpg --recv-keys` against an arbitrary keyserver is itself a trust problem.

## Walkthrough / mechanics

A Bitcoin Core release publishes:

- The binary tarballs (`bitcoin-X.Y.Z-<host>.tar.gz`).
- `SHA256SUMS`: a text file listing one `<sha256>  <filename>` line per artifact.
- `SHA256SUMS.asc`: a detached GPG signature over `SHA256SUMS`. May be a single signature or a clearsigned bundle of several.

For Guix-built releases there is also a per-builder set of files in `bitcoin-core/guix.sigs`. These are contributed signatures from independent builders; if you trust five different builders' signatures all attesting the same hash, you have stronger guarantees than trusting one.

Trust roots: Bitcoin Core does not centrally publish a single key. The recommended approach is to bootstrap from `contrib/builder-keys/keys.txt` in the source tree, which lists fingerprints of trusted builders (the file itself is committed to the repository whose tags you have already verified via `git verify-tag`). Once you trust the source tree, you trust the keys it lists.

WoT (web of trust) helps: if you already have a signed PGP path to one of the builders, you can rely on existing trust signatures. Most users do not, so the practical trust path is "I trust the bitcoincore.org domain plus the `bitcoin/bitcoin` repository tags".

## Worked example

Pick up keys for a recent release:

```bash
# Fetch the keyring from the bitcoin/bitcoin source tree (verify against your local clone)
$ git clone https://github.com/bitcoin/bitcoin.git
$ cd bitcoin && git checkout v28.0 && git verify-tag v28.0
gpg: Good signature from "Wladimir J. van der Laan ..."
$ cat contrib/builder-keys/keys.txt
0xE944AE667CF960B1004BC32FCA662BE18B877A60 achow101
0x71A3B16735405025D447E8F274810B012346C9A6 laanwj
... (many more)
```

Import the keys you need (you can import them all, or just the maintainer's):

```bash
$ gpg --keyserver hkps://keys.openpgp.org \
      --recv-keys 0x71A3B16735405025D447E8F274810B012346C9A6
gpg: key ...: "Wladimir J. van der Laan <laanwj@gmail.com>" imported
```

Download and verify:

```bash
$ wget https://bitcoincore.org/bin/bitcoin-core-28.0/SHA256SUMS
$ wget https://bitcoincore.org/bin/bitcoin-core-28.0/SHA256SUMS.asc
$ wget https://bitcoincore.org/bin/bitcoin-core-28.0/bitcoin-28.0-x86_64-linux-gnu.tar.gz

$ gpg --verify SHA256SUMS.asc SHA256SUMS
gpg: Signature made Wed Sep 27 14:00:00 2026 UTC
gpg: Good signature from "Wladimir J. van der Laan ..."
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 71A3 B167 3540 5025 D447  E8F2 7481 0B01 2346 C9A6

$ sha256sum -c --ignore-missing SHA256SUMS
bitcoin-28.0-x86_64-linux-gnu.tar.gz: OK
```

The "key is not certified" warning is normal unless you have manually `--lsign-key`'d the key. After confirming the fingerprint matches `keys.txt`, you can lsign it for a clean run next time:

```bash
$ gpg --lsign-key 0x71A3B16735405025D447E8F274810B012346C9A6
$ gpg --verify SHA256SUMS.asc SHA256SUMS
gpg: Good signature from "Wladimir J. van der Laan ..."
```

For a multi-builder verification (Guix attestations):

```bash
$ git clone https://github.com/bitcoin-core/guix.sigs.git
$ cd guix.sigs/28.0
$ for d in */; do
    builder=${d%/}
    echo "== $builder =="
    gpg --verify $d/all.SHA256SUMS.asc $d/all.SHA256SUMS 2>&1 | grep -E 'Good|BAD'
  done
== achow101 ==
gpg: Good signature from "Andrew Chow <andrew@achow101.com>"
== fanquake ==
gpg: Good signature from "Michael Ford ..."
== laanwj ==
gpg: Good signature from "Wladimir J. van der Laan ..."
== ... ==
```

Cross-check that every builder's `all.SHA256SUMS` lists the same hash for `bitcoin-28.0-x86_64-linux-gnu.tar.gz`:

```bash
$ for d in */; do
    grep 'x86_64-linux-gnu.tar.gz$' $d/all.SHA256SUMS
  done | sort -u
0e3b1f7b...  bitcoin-28.0-x86_64-linux-gnu.tar.gz
```

A single line in the output means every builder agrees. Multiple lines means at least one diverged; treat as suspicious.

## Common pitfalls

- Running `gpg --recv-keys` against `keys.openpgp.org` for a fingerprint copied from a forum post. The keyserver returned what was requested; that does not mean it is the right key. Always cross-reference `contrib/builder-keys/keys.txt` from a verified source tag.
- Verifying only the SHA256SUMS signature and skipping `sha256sum -c`. The signature might be valid on a SHA256SUMS that does not actually list your binary.
- Verifying only the hash and skipping the GPG signature. An attacker who replaces both the binary and the SHA256SUMS line passes hash check but not signature check.
- Treating the GPG WARNING about uncertified key as a failure. It is a UX wart of GnuPG. Confirm fingerprint, lsign the key, move on.
- Using an old `keys.txt` from a tag too far in the past. Builders rotate keys; a key that was valid for v22 may have been retired by v28. Check the keyring at the version you are verifying.
- Thinking code-signing certificates (Authenticode on Windows, codesign on macOS) replace GPG. They protect against OS warnings, not against tampered tarballs at download time. Always GPG-verify before unpacking.

## References

- `contrib/builder-keys/keys.txt` in bitcoin/bitcoin.
- `bitcoin-core/guix.sigs` repository.
- bitcoincore.org release page.
- GnuPG manual on keyserver lookup and `--lsign-key`.
