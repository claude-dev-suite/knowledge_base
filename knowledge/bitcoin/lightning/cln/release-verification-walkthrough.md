# CLN Release Verification Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/cln`.
> Canonical source: https://github.com/ElementsProject/lightning/releases
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/cln/SKILL.md

## Concept

Core Lightning ships **signed binaries, sometimes before the source**.
That inverts the usual "build from source, verify yourself" posture and
makes signature verification the only check available during an
embargo. An operator whose only upgrade path is a source build cannot
patch at all until the embargo lifts.

The `v26.06.7` security release (2026-08-28) is the worked case for
every part of this: an embargoed source drop, a signed multi-signer
manifest, a Docker supply-chain incident, and a reproducible build that
does not reproduce unless you pass a flag the release tooling cannot
pass for you. Everything below is drawn from that release's notes.

Three artefacts, three different trust levels:

| Artefact | Signed manifest | Reproducible |
|----------|-----------------|--------------|
| amd64 tarballs | Yes (`SHA256SUMS-v26.06.7`) | Yes, with `COPTFLAGS=-O3` |
| arm64 tarballs | Yes (`SHA256SUMS-v26.06.7-arm64`) | No - tooling not in tree |
| `linux/arm/v7` Docker | No | No |

## Walkthrough / mechanics

### 1. Verify checksums, then signatures

```sh
sha256sum -c SHA256SUMS-v26.06.7 --ignore-missing
gpg --verify SHA256SUMS-v26.06.7.asc SHA256SUMS-v26.06.7
```

`SHA256SUMS-v26.06.7` covers the amd64 tarballs.
`SHA256SUMS-v26.06.7-arm64` is a separate manifest with its own
signature file for the arm64 tarballs. Verifying one says nothing about
the other.

Signing keys for v26.06.7, per the release notes:

| Signer | Fingerprint |
|--------|-------------|
| nGoline | `4E4A 142F 8BD3 C38A 56B3 62ED 578C AC08 4725 45C5` |
| Christian Decker | `B731 AAC5 21B0 1385 9313 F674 A26D 6D9F E088 ED58` |
| Peter Neuroth | `653B 19F3 3DF7 EFF3 E9D1 C94C C3F2 1EE3 87FF 4CD2` |
| daywalker90 | `8A07 9421 A871 D0B1 0835 1193 7AB4 802E D5A6 39F3` |

Only Decker's and Neuroth's keys are in `contrib/keys/` at this tag;
the other two were added on master afterwards, and the tag's tree
cannot change retroactively. Fetch those from a keyserver
(`gpg --recv-keys <fingerprint>`).

`gpg --verify` will report a fingerprint that does not match the table.
That is expected: signers use signing subkeys and the reported
fingerprint is the subkey of the listed primary. It resolves once the
primary is imported. Do not treat the mismatch as a failure.

### 2. Docker: pin by digest, not by tag

A tag is a mutable pointer, and on this release it pointed at the wrong
thing. Between **28 August 2026 16:04 UTC and 1 September 2026**, the
`v26.06.7`, `latest`, `v26.06.7-vls` and `latest-vls` tags served
images that reported `v26.06.7` on startup but did **not** contain the
fixes - CI had published them automatically from a placeholder tag.
They were later replaced. Users pinned to `v26.06.6` or earlier were
never affected.

```sh
docker image inspect --format '{{index .RepoDigests 0}}' \
  elementsproject/lightningd:v26.06.7
```

Expected digests for v26.06.7:

| Tag | Digest |
|-----|--------|
| `v26.06.7`, `latest` | `sha256:0421a5f0d1b2e1ad639edfa17d777816040e3850d91bae7f2d32186d9c1e6da4` |
| `v26.06.7-vls`, `latest-vls` | `sha256:6a5e05c13a65613f8c0fe3830c60248a6724e7206c1c23dd26ac2e98a3e72c1f` |

If the digest does not match, `docker pull` again. The images are
assembled from the signed release tarballs rather than compiled in the
image, so the container binaries are the ones the manifests cover - for
`linux/amd64` and `linux/arm64`. `linux/arm/v7` has no release tarball;
those binaries are cross-compiled separately and covered by nothing.
The images carry no provenance or SBOM attestations.

Two more v26.06.7 image changes worth catching before a restart:

- CLN now installs to `/usr/bin` and `/usr/libexec/c-lightning`;
  earlier images used `/usr/local`. Symlinks from the old paths are
  included, so hardcoded paths still work.
- VLS users: `v26.06.7-vls` carries VLS `v0.14.0`, unchanged from
  `v26.06.6-vls`, but `VLS_CLN_VERSION` must be bumped to `v26.06.7`
  or `remote_hsmd_socket` refuses to start.

### 3. Reproducing the build

The trap: `configure` defaults to `-Og`, and the release binaries were
built at `-O3`. Checking out the tag and running the normal
reproducible build produces binaries that do not match the manifest.
`tools/build-release.sh` forwards only `FORCE_MTIME`, `FORCE_VERSION`
and `MAKEPAR` into the build containers, so it cannot pass `COPTFLAGS`
through - the containers have to be invoked directly.

```sh
git clone https://github.com/ElementsProject/lightning && cd lightning
git checkout v26.06.7
git submodule update --init --recursive

mkdir -p release
for d in jammy noble resolute; do
  docker run --rm -v "$(pwd)":/repo \
    -e FORCE_MTIME=2026-08-26 -e FORCE_VERSION=v26.06.7 -e MAKEPAR=8 \
    -e COPTFLAGS=-O3 cl-repro-$d
done
```

Confirm the flag reached the compiler rather than inferring it from
tarball size. The build log must contain:

```
Setting COPTFLAGS... -O3 -ffunction-sections
```

or, on a finished binary:

```sh
strings -a usr/bin/lightningd | grep 'GNU C11'
# expect: GNU C11 ... -g -O3 -std=gnu11 ... -ffunction-sections
```

Known caveats, both predating this release: the Fedora builder image is
rebuilt `--no-cache` from `fedora:40` with a live `dnf update` and a
`wget` of Bitcoin Core, so two builds days apart can pick up different
toolchains - if every Ubuntu target matches and only Fedora differs,
that is why. And the arm64 build script, arm64 signing script and arm64
support in `contrib/reprobuild/` are not in the v26.06.7 tree, so the
arm64 tarballs are verifiable by signature but not independently
reproducible until that work is upstreamed.

## Worked example

Timeline of the v26.06.7 embargo, as an upgrade decision:

| When (2026) | Event |
|-------------|-------|
| 08-28 16:08 UTC | Release published: signed binaries, **no source** |
| 08-28 16:04 - 09-01 | Docker tags serve placeholder-built images |
| 09-11 11:42 UTC | Embargo ends; `v26.06.7` tag repointed, source zip attached |

The stated rationale for the 14-day hold was to reduce the chance of
attackers reverse-engineering the fixes before the network updated. The
release notes also attribute the rising volume and pace of
vulnerability reports to increasingly capable AI models being pointed
at open-source code - i.e. expect this pattern again.

An operator on 2026-08-29 therefore had exactly one option: take the
signed tarball.

```sh
sudo tar -xvf <release>.tar.xz -C /usr/local --strip-components=2
# then restart lightningd
```

No database migration steps beyond the automatic ones applied at
startup.

## Common pitfalls

- **Source-only upgrade paths.** A deployment that can only build from
  git cannot patch during an embargo. Keep a binary-install path
  ready and tested *before* you need it.
- **Trusting a tag across a security release.** `latest` was wrong for
  four days on v26.06.7. Pin by digest and re-check after a pull.
- **Verifying one manifest and assuming both.** amd64 and arm64 have
  separate `SHA256SUMS` files and separate signatures.
- **Reading a subkey fingerprint as a bad signature.** Import the
  primary key and re-run.
- **Reproducing without `COPTFLAGS=-O3`.** You will get a clean build
  that does not match the manifest, and conclude the release is
  tampered with.
- **Expecting arm64 to reproduce.** It cannot, from this tag.
- **Skipping `VLS_CLN_VERSION`.** The signer refuses to start on a
  version mismatch, which looks like a broken upgrade rather than a
  config gap.

## References

- CLN `v26.06.7` release notes (2026-08-28, updated 2026-09-11 when
  the embargo lifted) - source of every fingerprint, digest and
  timestamp above.
- CLN `tools/build-release.sh`, `contrib/cl-repro.sh`,
  `contrib/reprobuild/Dockerfile.<dist>`.
- CalVer release line: `v26.06` (2026-06-02) through `v26.06.7`
  (2026-08-28); `v26.04` (2026-04-20); `v25.12` (2025-12-04).
