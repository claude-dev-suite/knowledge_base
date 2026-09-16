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
- `SHA256SUMS.asc`: a detached GPG signature over `SHA256SUMS`. May be a single signature or, as with recent releases, a concatenated bundle of one detached signature per builder.

For Guix-built releases there is also a per-builder set of files in `bitcoin-core/guix.sigs`. These are contributed signatures from independent builders; if you trust five different builders' signatures all attesting the same hash, you have stronger guarantees than trusting one. Each builder publishes two attestations per version, because the Guix build has two stages: `noncodesigned.SHA256SUMS` covers the binaries compiled from source, and `all.SHA256SUMS` covers those same binaries after the Windows/macOS detached code signatures (from `bitcoin-core/bitcoin-detached-sigs`) have been attached. `all.SHA256SUMS` is the one that covers everything uploaded to the website, so it is what release downloads should be checked against.

Trust roots: Bitcoin Core does not centrally publish a single key. As of Bitcoin Core v22.0 the builder keys live in the **guix.sigs** repository, one key file per signer at `builder-keys/<signer>.gpg` (39 signers as of September 2026). The old `contrib/builder-keys/keys.txt` in `bitcoin/bitcoin` shipped through v24.0 and was removed in November 2022 (commit `e6864fa1`); it 404s on `master` and on every tag from v25.0 onward (checked September 2026). You (the end user) decide which of those keys to trust and how many matching signatures to demand; no threshold is enforced on your behalf at verification time. The release process itself does set thresholds - `doc/release-process.md` gates the upload on 6 or more matching Guix builds (as of September 2026) - but those bind the release managers, not your check.

Core also ships `contrib/verify-binaries/verify.py`, which is the supported one-command version of everything below: it downloads `SHA256SUMS` and `SHA256SUMS.asc` from bitcoincore.org/bitcoin.org, accepts the checksum file only on a plurality of trusted signatures (`--min-good-sigs`, default 3), then downloads the release files and checks their hashes. `--import-keys` makes it fetch signer keys it does not recognise.

WoT (web of trust) helps: if you already have a signed PGP path to one of the builders, you can rely on existing trust signatures. Most users do not, so the practical trust path is "I trust the bitcoincore.org domain plus the `bitcoin/bitcoin` repository tags".

## Worked example

Pick up builder keys from guix.sigs (Bitcoin Core 31.1, the current release as of September 2026):

```bash
# The keyring is the builder-keys/ directory of guix.sigs, one file per signer
$ git clone https://github.com/bitcoin-core/guix.sigs.git
$ ls guix.sigs/builder-keys/
0xb10c.gpg  achow101.gpg  cfields.gpg  fanquake.gpg  glozow.gpg
hebasto.gpg  laanwj.gpg   ... (39 files)
```

Import the ones you decided to trust - the file itself is the key, no keyserver round-trip needed:

```bash
$ gpg --import guix.sigs/builder-keys/achow101.gpg
gpg: key 17565732E08E5E41: public key "Ava Chow <me@achow101.com>" imported
```

Download and verify:

```bash
$ wget https://bitcoincore.org/bin/bitcoin-core-31.1/SHA256SUMS
$ wget https://bitcoincore.org/bin/bitcoin-core-31.1/SHA256SUMS.asc
$ wget https://bitcoincore.org/bin/bitcoin-core-31.1/bitcoin-31.1-x86_64-linux-gnu.tar.gz

$ gpg --verify SHA256SUMS.asc SHA256SUMS
gpg: Signature made Tue Jul  7 00:09:57 2026 GMT
gpg:                using RSA key 152812300785C96444D3334D17565732E08E5E41
gpg:                issuer "me@achow101.com"
gpg: Good signature from "Ava Chow <me@achow101.com>" [unknown]
gpg: WARNING: The key's User ID is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 1528 1230 0785 C964 44D3  334D 1756 5732 E08E 5E41
gpg: Signature made Tue Jul  7 08:53:43 2026 GMT
gpg:                using RSA key CFB16E21C950F67FA95E558F2EEB9F5CC09526C1
gpg:                issuer "fanquake@gmail.com"
gpg: Can't check signature: No public key
... (one block per signer in the bundle)

$ sha256sum -c --ignore-missing SHA256SUMS
bitcoin-31.1-x86_64-linux-gnu.tar.gz: OK
```

`SHA256SUMS.asc` is a bundle of detached signatures, so `gpg --verify` walks it signer by signer. "No public key" for signers you did not import is expected and is not a failure; what matters is that at least one - preferably several - of the keys you chose to trust produced a `Good signature`.

The "key is not certified" warning is likewise normal unless you have manually `--lsign-key`'d the key. After confirming the fingerprint matches the one in `builder-keys/`, you can lsign it for a clean run next time:

```bash
$ gpg --lsign-key 152812300785C96444D3334D17565732E08E5E41
$ gpg --verify SHA256SUMS.asc SHA256SUMS
gpg: Good signature from "Ava Chow <me@achow101.com>"
```

Or skip all of it and let Core's own helper do the plurality check for you:

```bash
$ ./contrib/verify-binaries/verify.py --import-keys pub 31.1-x86_64-linux
```

For a multi-builder verification (Guix attestations):

```bash
$ git clone https://github.com/bitcoin-core/guix.sigs.git
$ cd guix.sigs/31.1
$ ls
0xb10c  Emzy  Sjors  achow101  benthecarman  fanquake  guggero  hebasto
m3dwards  marcofleon  pinheadmz  sedited  sipsorcery  svanstaa  theStack
willcl-ark
$ for d in */; do
    builder=${d%/}
    echo "== $builder =="
    gpg --verify $d/all.SHA256SUMS.asc $d/all.SHA256SUMS 2>&1 | grep -E 'Good|BAD'
  done
== achow101 ==
gpg: Good signature from "Ava Chow <me@achow101.com>"
== fanquake ==
gpg: Good signature from "Michael Ford ..."
== ... ==
```

Use `all.SHA256SUMS`, not `noncodesigned.SHA256SUMS`: the latter only covers the stage-1 outputs, so it will not list the signed Windows `.exe` or the signed macOS bundles that the website actually serves.

Cross-check that every builder's `all.SHA256SUMS` lists the same hash for `bitcoin-31.1-x86_64-linux-gnu.tar.gz` (checked September 2026):

```bash
$ for d in */; do
    grep 'x86_64-linux-gnu.tar.gz$' $d/all.SHA256SUMS
  done | sort -u
b80d9c3e04da78fb6f0569685673418cf686fadba9042d926d13fb87ff503f9e  bitcoin-31.1-x86_64-linux-gnu.tar.gz
```

A single line in the output means every builder agrees. Multiple lines means at least one diverged; treat as suspicious.

## Common pitfalls

- Running `gpg --recv-keys` against `keys.openpgp.org` for a fingerprint copied from a forum post. The keyserver returned what was requested; that does not mean it is the right key. Always cross-reference `builder-keys/<signer>.gpg` in `bitcoin-core/guix.sigs`.
- Looking for `contrib/builder-keys/keys.txt` in `bitcoin/bitcoin`. It was removed after v24.0 and had already stopped being the trust root at v22.0; tutorials that tell you to `cat` it are unfollowable against any release you would actually run.
- Checking a download against `noncodesigned.SHA256SUMS`. It only covers stage-1 outputs. Release binaries go with `all.SHA256SUMS`.
- Verifying only the SHA256SUMS signature and skipping `sha256sum -c`. The signature might be valid on a SHA256SUMS that does not actually list your binary.
- Verifying only the hash and skipping the GPG signature. An attacker who replaces both the binary and the SHA256SUMS line passes hash check but not signature check.
- Treating the GPG WARNING about uncertified key as a failure. It is a UX wart of GnuPG. Confirm fingerprint, lsign the key, move on.
- Using a stale copy of the keyring. Builders rotate keys and the signer set changes between releases - achow101's current builder key is `1528 1230 0785 C964 44D3  334D 1756 5732 E08E 5E41` (checked September 2026), not the `E944AE66...` key that older guides quote. Pull `guix.sigs` fresh and check which signers actually attested the version you are verifying.
- Verifying a release from an End-of-Life line. As of September 2026 only 29.x, 30.x and 31.x are maintained (see <https://bitcoincore.org/en/lifecycle/>); a valid signature on a 28.x tarball proves authenticity, not that the binary is still getting security fixes.
- Thinking code-signing certificates (Authenticode on Windows, codesign on macOS) replace GPG. They protect against OS warnings, not against tampered tarballs at download time. Always GPG-verify before unpacking.

## References

- `builder-keys/<signer>.gpg` in [bitcoin-core/guix.sigs](https://github.com/bitcoin-core/guix.sigs/tree/main/builder-keys).
- `bitcoin-core/guix.sigs` README - two-stage attestation layout.
- `contrib/verify-binaries/README.md` in bitcoin/bitcoin - `verify.py` usage.
- bitcoincore.org release page and <https://bitcoincore.org/en/lifecycle/>.
- GnuPG manual on keyserver lookup and `--lsign-key`.
