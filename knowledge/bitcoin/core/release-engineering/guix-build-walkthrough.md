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
3. Check out the tag you want to verify (e.g., `v28.0`).
4. Run `./contrib/guix/guix-build`. Guix downloads the package definitions, computes a derivation graph, builds the entire toolchain (gcc, binutils, make, the cross-compilers for macOS and Windows), and finally compiles bitcoind/bitcoin-cli/bitcoin-qt for each target.
5. Output goes to `guix-build-<version>/output/<host>/`. Compute sha256 of each tarball.
6. Run `./contrib/guix/guix-attest` with your GPG key to sign the output hashes.
7. Open a PR against `bitcoin-core/guix.sigs` adding `<version>/<your-codename>/all.SHA256SUMS` and the `.asc`.

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

Build Bitcoin Core 28.0:

```bash
$ git clone https://github.com/bitcoin/bitcoin.git
$ cd bitcoin
$ git checkout v28.0
$ git verify-tag v28.0
gpg: Good signature from "Bitcoin Core ..."

# Limit which targets to build (full set is x86_64-linux-gnu, aarch64-linux-gnu,
# arm-linux-gnueabihf, riscv64-linux-gnu, x86_64-w64-mingw32, x86_64-apple-darwin, arm64-apple-darwin)
$ HOSTS="x86_64-linux-gnu x86_64-w64-mingw32" \
  ./contrib/guix/guix-build
... (4-8 hours later) ...
INFO: Build successful
```

Inspect outputs:

```bash
$ ls guix-build-28.0/output/x86_64-linux-gnu/
SHA256SUMS.part
bitcoin-28.0-x86_64-linux-gnu-debug.tar.gz
bitcoin-28.0-x86_64-linux-gnu.tar.gz
bitcoin-28.0.tar.gz

$ sha256sum guix-build-28.0/output/x86_64-linux-gnu/bitcoin-28.0-x86_64-linux-gnu.tar.gz
0e3b1f7b... bitcoin-28.0-x86_64-linux-gnu.tar.gz
```

Compare with another builder via guix.sigs:

```bash
$ git clone https://github.com/bitcoin-core/guix.sigs.git
$ cat guix.sigs/28.0/laanwj/all.SHA256SUMS | grep linux-gnu
0e3b1f7b...  bitcoin-28.0-x86_64-linux-gnu.tar.gz
```

If your hash matches `laanwj`'s and `fanquake`'s and others', you have a multi-attestation reproducible build. If it differs, something in your environment is non-deterministic; investigate before publishing.

Sign and contribute attestation:

```bash
$ ./contrib/guix/guix-attest
INFO: Wrote SHA256SUMS to guix-build-28.0/output/x86_64-linux-gnu/SHA256SUMS
INFO: Signed with GPG key <fingerprint>

$ cd guix.sigs
$ mkdir -p 28.0/<your-codename>
$ cp -r ../bitcoin/guix-build-28.0/output/all.SHA256SUMS \
        ../bitcoin/guix-build-28.0/output/all.SHA256SUMS.asc \
        28.0/<your-codename>/
$ git add 28.0/<your-codename>/
$ git commit -s -m "guix: add 28.0 attestations for <your-codename>"
$ git push origin guix-attest-28.0
# Open PR
```

## Common pitfalls

- Running `guix-build` without enough disk. The derivation graph can require 50+ GB during expansion. Free space first or set `--max-jobs=1` to slow down concurrent unpacking.
- A non-tagged source tree: Guix builds whatever is checked out. `git status` should be clean and on a signed tag.
- Mismatched Guix version across builders. The `manifest.scm` in `contrib/guix/` pins the channel; everyone must use that exact channel commit. Skipping `guix time-machine -C ...` produces nondeterministic results.
- Using a build VM with the wrong locale or timezone. Some upstream tools embed these. Bitcoin's Guix scripts already neutralize this, but custom tweaks (e.g., `LANG=C.UTF-8` outside Guix) can leak.
- Attesting binaries you did not build yourself. The whole point is independent verification; signing someone else's tarball defeats it.
- Forgetting to run `git verify-tag` before building. Building from an unsigned source defeats the chain. Always verify the upstream tag first.
- Believing reproducibility implies absence of bugs. It only proves no extra bytes were inserted between source and binary. The source itself can still have vulnerabilities.

## References

- `contrib/guix/README.md` in bitcoin/bitcoin.
- `bitcoin-core/guix.sigs` repository.
- GNU Guix manual.
- bitcoincore.org release announcements with builder attestations.
