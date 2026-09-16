# Guix Build Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/release-engineering`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/contrib/guix/README.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/release-engineering/SKILL.md

## Concept

A Guix build of Bitcoin Core is a fully reproducible cross-compilation: anyone with the same source tag, the same Guix version, and a Linux box can produce byte-identical binaries for Linux, macOS, and Windows. The point is supply-chain protection. If five independent builders all reach the same `sha256sum` for `bitcoin-X.Y.Z-x86_64-linux-gnu.tar.gz`, no single one of them could have inserted malware without colluding. Guix achieves this by defining a deterministic build environment from the bootstrap C compiler upward, with no reliance on the host system's libraries or compiler. The cost is disk and time; the payoff is an attestation chain that is the strongest a software project can offer.

## Walkthrough / mechanics

The flow:

1. Install Guix (a package manager with deterministic functional semantics). It can run alongside any Linux distro; `guix-daemon` is a separate process.
2. Clone `bitcoin/bitcoin` and `bitcoin-core/guix.sigs`.
3. Check out the tag you want to verify (e.g., `v31.1`, the current release as of September 2026). Prefer a tag on a maintained line - 29.x, 30.x and 31.x as of September 2026, per <https://bitcoincore.org/en/lifecycle/>.
4. Run `./contrib/guix/guix-build`. Guix downloads the package definitions, computes a derivation graph, builds the entire toolchain (gcc, binutils, make, the cross-compilers for macOS and Windows), and finally compiles bitcoind/bitcoin-cli/bitcoin-qt for each target.
5. Output goes to `guix-build-<version>/output/<host>/`. Compute sha256 of each tarball.
6. Run `./contrib/guix/guix-attest` with your GPG key to sign the output hashes.
7. Open a PR against `bitcoin-core/guix.sigs` adding `<version>/<your-codename>/`. Each builder contributes two attestation pairs: `noncodesigned.SHA256SUMS(.asc)` for the stage-1 outputs built from source, and `all.SHA256SUMS(.asc)` for the same binaries once the Windows/macOS detached code signatures from `bitcoin-core/bitcoin-detached-sigs` have been attached. `all.SHA256SUMS` is the one that covers every file the website serves.

Disk: ~30 GB for the build cache, growing across versions. RAM: ~8 GB recommended. Wall clock: 4-8 hours on a modern multi-core box, much longer on slow hardware.

The first run downloads and builds the world; subsequent runs reuse the cached store. Bumping a Guix version requires a full rebuild because the derivation hash changes.

## Worked example

Install Guix on Debian (other distros: see Guix manual):

```bash
$ wget https://git.savannah.gnu.org/cgit/guix.git/plain/etc/guix-install.sh
$ sudo bash guix-install.sh
$ source /etc/profile.d/guix.sh
$ guix --version
guix (GNU Guix) 1.4.0
```

Build Bitcoin Core 31.1:

```bash
$ git clone https://github.com/bitcoin/bitcoin.git
$ cd bitcoin
$ git checkout v31.1
$ git verify-tag v31.1
gpg: Good signature from "Michael Ford (bitcoin-otc) <fanquake@gmail.com>"
# v31.1 was tagged by fanquake; other tags carry a different maintainer UID

# Limit which targets to build. The default HOSTS set in contrib/guix/guix-build is
# x86_64-linux-gnu, arm-linux-gnueabihf, aarch64-linux-gnu, riscv64-linux-gnu,
# powerpc64-linux-gnu, x86_64-w64-mingw32, x86_64-apple-darwin, arm64-apple-darwin
$ HOSTS="x86_64-linux-gnu x86_64-w64-mingw32" \
  ./contrib/guix/guix-build
... (4-8 hours later) ...
INFO: Build successful
```

Inspect outputs:

```bash
$ ls guix-build-31.1/output/x86_64-linux-gnu/
SHA256SUMS.part
bitcoin-31.1-x86_64-linux-gnu-debug.tar.gz
bitcoin-31.1-x86_64-linux-gnu.tar.gz
bitcoin-31.1.tar.gz

$ sha256sum guix-build-31.1/output/x86_64-linux-gnu/bitcoin-31.1-x86_64-linux-gnu.tar.gz
b80d9c3e04da78fb6f0569685673418cf686fadba9042d926d13fb87ff503f9e  bitcoin-31.1-x86_64-linux-gnu.tar.gz
```

That hash is the one every 31.1 builder attested and the one published in the release `SHA256SUMS` (checked September 2026). Compare with another builder via guix.sigs:

```bash
$ git clone https://github.com/bitcoin-core/guix.sigs.git
$ cat guix.sigs/31.1/fanquake/all.SHA256SUMS | grep x86_64-linux-gnu.tar.gz
b80d9c3e04da78fb6f0569685673418cf686fadba9042d926d13fb87ff503f9e  bitcoin-31.1-x86_64-linux-gnu.tar.gz
```

If your hash matches `fanquake`'s and `achow101`'s and others', you have a multi-attestation reproducible build. If it differs, something in your environment is non-deterministic; investigate before publishing.

Sign and contribute attestation:

```bash
$ ./contrib/guix/guix-attest
INFO: Wrote SHA256SUMS to guix-build-31.1/output/x86_64-linux-gnu/SHA256SUMS
INFO: Signed with GPG key <fingerprint>

$ cd guix.sigs
$ mkdir -p 31.1/<your-codename>
$ cp -r ../bitcoin/guix-build-31.1/output/all.SHA256SUMS \
        ../bitcoin/guix-build-31.1/output/all.SHA256SUMS.asc \
        31.1/<your-codename>/
$ git add 31.1/<your-codename>/
$ git commit -s -m "guix: add 31.1 attestations for <your-codename>"
$ git push origin guix-attest-31.1
# Open PR
```

First-time builders also add their public key once, as `builder-keys/<your-codename>.gpg` in the same repository - that directory (39 signers as of September 2026) is the keyring downstream verifiers bootstrap from. The old `contrib/builder-keys/` directory in `bitcoin/bitcoin` was removed in November 2022 and is not the place to add it.

## Common pitfalls

- Running `guix-build` without enough disk. The derivation graph can require 50+ GB during expansion. Free space first or set `--max-jobs=1` to slow down concurrent unpacking.
- A non-tagged source tree: Guix builds whatever is checked out. `git status` should be clean and on a signed tag.
- Mismatched Guix version across builders. The `manifest.scm` in `contrib/guix/` pins the channel; everyone must use that exact channel commit. Skipping `guix time-machine -C ...` produces nondeterministic results.
- Using a build VM with the wrong locale or timezone. Some upstream tools embed these. Bitcoin's Guix scripts already neutralize this, but custom tweaks (e.g., `LANG=C.UTF-8` outside Guix) can leak.
- Attesting binaries you did not build yourself. The whole point is independent verification; signing someone else's tarball defeats it.
- Forgetting to run `git verify-tag` before building. Building from an unsigned source defeats the chain. Always verify the upstream tag first.
- Believing reproducibility implies absence of bugs. It only proves no extra bytes were inserted between source and binary. The source itself can still have vulnerabilities.

## References

- `contrib/guix/README.md` and `contrib/guix/INSTALL.md` in bitcoin/bitcoin.
- `bitcoin-core/guix.sigs` repository - attestations and `builder-keys/`.
- `doc/release-process.md` in bitcoin/bitcoin - where guix-build sits in the release.
- GNU Guix manual.
- bitcoincore.org release announcements with builder attestations.
