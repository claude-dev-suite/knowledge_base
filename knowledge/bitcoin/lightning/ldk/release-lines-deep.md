# LDK Release Lines and Version Floors - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/ldk`.
> Canonical source: https://github.com/lightningdevkit/rust-lightning/blob/main/CHANGELOG.md
> Check the newest release tag's copy too - as of 2026-09-15 `main` tops out at 0.2.5.
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/ldk/SKILL.md

## Concept

LDK is a set of crates, not a daemon, so it has no "run the latest
release" story. A downstream app pins `lightning = "0.2"` in its
`Cargo.toml` and inherits whatever patch release Cargo resolves. That
makes the *version floor* - the oldest patch release that is not known
to be exploitable - the number that actually matters, and it makes the
release channel unusual in three ways:

1. **There are no GitHub Releases.** As of September 2026 the
   `lightningdevkit/rust-lightning` releases API returns an empty list;
   releases are published as git tags plus crates.io publishes.
2. **There are no GitHub Security Advisories.** The repo's
   security-advisories list is empty as of September 2026. The
   `## Security` section of `CHANGELOG.md` is the authoritative
   advisory channel. Nothing will appear in `cargo audit` from the
   project's own GHSA entries, because it does not file any.
3. **Two lines are maintained in parallel.** Security fixes land on the
   current line *and* are backported to the previous one on the same
   day, so an app that cannot take the API churn of a major bump still
   has a patched option.

The practical consequence: subscribe to the CHANGELOG, not to release
notifications, and re-check the floor on every dependency bump.

## Walkthrough / mechanics

### Release lines (as of September 2026)

Dates are crates.io publication dates for the `lightning` crate.

| Version | Published | Line | Note |
|---------|-----------|------|------|
| 0.2.6 | 2026-09-09 | 0.2 | Current stable. "The More You Dig" |
| 0.3.0-rc1 | 2026-08-31 | 0.3 | Pre-release |
| 0.2.5 | 2026-08-05 | 0.2 | "The MegaScan Project" |
| 0.1.12 | 2026-08-05 | 0.1 | Backport of the 0.2.5 security fixes |
| 0.3.0-beta1 | 2026-07-24 | 0.3 | Pre-release |
| 0.2.4 | 2026-06-30 | 0.2 | |
| 0.1.11 | 2026-06-25 | 0.1 | |
| 0.2.3 | 2026-06-19 | 0.2 | "Through the Loupe" |
| 0.1.10 | 2026-06-18 | 0.1 | "Loupe de Loupe" |
| 0.2.0 | 2025-12-03 | 0.2 | Line opened. "Natively Asynchronous Splicing" |

The 0.3 line is still in pre-release as of 2026-09-15: `0.3.0-beta1`
and `0.3.0-rc1` are on crates.io, but no final `0.3.0` has been
published. Note that the `CHANGELOG.md` shipped inside the
`0.3.0-beta1` tag predates the 0.2.3 entry - the beta branched early,
so for the 0.2-line history read `main`, not the pre-release tag.

The satellite crates are versioned independently and do not track the
core crate: at `ldk-node` v0.7.0 the pin set is `lightning = "0.2.0"`,
`lightning-types = "0.3.0"`, `lightning-invoice = "0.34.0"`, with the
rest of the `lightning-*` crates at `"0.2.0"`. Bump them as a set.

### ldk-node tracks LDK loosely

`ldk-node` is the opinionated wrapper, on its own cadence: v0.7.0
(2025-12-03) is the newest tagged release as of September 2026, and it
pins the `lightning` crates at `0.2.0`. `ldk-node` `main` currently
pins the `lightning-*` crates to a **git revision** rather than a
crates.io version, so building from `main` does not give you a
release-audited LDK.

Consequence for the version floor: an app on `ldk-node` v0.7.0 resolves
`lightning` from the `0.2` line and will pick up `0.2.6` on a
`cargo update`, but only if the lockfile is actually refreshed. A
committed `Cargo.lock` that has not been updated since December 2025
pins `0.2.0` and carries every vulnerability listed below.

### Reading the CHANGELOG's Security section

Each `## Security` block opens with a one-paragraph impact summary
naming the *class* of vulnerability, then itemises fixes with PR
numbers. The summary paragraph is what determines urgency; "funds
theft" and "denial of service" are held to different bars.

**Check the release tag as well as `main`.** A patch release's entry is
written on its release branch and lands on `main` only afterwards, so
for a period the newest advisory exists *only* in the tag. As of
2026-09-15 this is the live case: `CHANGELOG.md` on `main` still tops
out at `# 0.2.5 - Aug 4, 2026`, and the 0.2.6 `## Security` section
covering #4905 and #4982 is visible only at the `v0.2.6` tag. Reading
`main` alone would tell you the floor is 0.2.5.

## Worked example

Tracing the 0.2 line's security history through 2026:

**0.2.3 / 0.1.10 (2026-06-18/19)** - "several underestimates of the
anchor reserves required to ensure we can reliably close channels,
several denial-of-service vulnerabilities and a sanitization issue".
The notable items:

- `possiblyrandom` did not actually generate random data unless
  explicitly configured, leaving LDK open to HashDoS by default (#4719).
- `anchor_channel_reserves` ignored zero-fee-commitment channels (#4592)
  and overestimated wallet UTXO value by ignoring `TxIn` spend cost
  (#4670) - a counterparty could open many channels and leave the node
  unable to force-close properly.
- Several remotely-reachable panics: `recover_payee_pub_key` on an
  invoice with an explicit pubkey (#4717), overflowing invoice feerates
  (#4716), pre-1970 `LSPSDateTime` from a counterparty message (#4715),
  unicode at the start of an `OMNameResolver` TXT record (#4718).

Credited to Project Loupe, which is where both release names come from.

**0.2.5 / 0.1.12 (2026-08-04, published 08-05)** - the serious one:
"an on-chain funds-theft vulnerability for nodes forwarding HTLCs which
accept channels from untrusted nodes", plus DoS.

- `ChannelMonitor` confused two HTLCs with equal `payment_hash` and
  amount while processing an un-revoked counterparty's claim, leading
  to incorrect HTLC resolution (#4854). This is the theft vector.
- Panic on an onion message whose invalid reply path lists the local
  node as introduction point (#4850).
- Panic sending messages too large for the protocol framing, including
  an oversized failure onion and an oversized splice transaction
  (#4845, #4852).
- The 50-peers-with-unfunded-channels limit was not enforced against an
  `open_channel` flood (#4851).
- Panic when a reorg changes whether an on-chain HTLC claim should be
  batched (#4849).

Reported by Project Loupe and Kyle W. Santiago.

**0.2.6 (2026-09-09)** - splice fee inflation plus a DoS:

- A malicious splice peer could make us over-allocate fee when
  contributing to a splice, with the over-allocation going to *their*
  output (#4905). Small loss, but a real one, and 0.2-line only since
  0.1 has no splicing.
- A bogus payment HTLC rejected immediately after a second HTLC with
  the same `payment_hash` was successfully forwarded could leave
  `ChannelManager` in a state that fails deserialization - i.e. the
  node does not restart (#4982). Reported by Erick Cestari.

So the floors as of September 2026: **0.2.6** on the 0.2 line, **0.1.12**
on the 0.1 line. Neither 0.2.6 fix was backported to 0.1: #4905 has no
0.1 counterpart, and upstream says nothing either way about whether
#4982 reaches 0.1. Anything older than 0.1.12 / 0.2.5 is exposed to the
HTLC-resolution theft bug.

## Common pitfalls

- **Treating `cargo audit` as coverage.** LDK files no GHSA entries, so
  a clean audit run says nothing about LDK. Diff the CHANGELOG.
- **Stale `Cargo.lock`.** A caret requirement of `"0.2"` does not
  upgrade anything on its own. `cargo update -p lightning` is the step
  that moves the floor.
- **Bumping `lightning` alone.** The `lightning-*` satellite crates are
  separately versioned and must move together; mismatched versions
  fail to compile at best and silently skip a fix at worst.
- **Pinning `ldk-node` `main`.** It resolves the LDK crates from a git
  rev, which is not a release-audited tree.
- **Assuming 0.1 is unmaintained.** It is still receiving same-day
  security backports as of August 2026. It does not receive features -
  splicing, for instance, is 0.2-only, which is why #4905 has no 0.1
  counterpart.
- **Reading the changelog from a pre-release tag.** Beta and rc tags
  carry a changelog frozen at branch point - `0.3.0-beta1`'s tops out
  before the 0.2.3 entry. For line history, read `main`.
- **Reading only `main` for the newest floor.** The inverse trap: a
  freshly published patch release's `## Security` section appears on
  its release tag before it is merged to `main`. Check both. As of
  2026-09-15, 0.2.6's advisory is tag-only.

## References

- rust-lightning `CHANGELOG.md` on `main` and at the newest release
  tag (security sections) - as of 2026-09-15 the 0.2.6 entry is only
  at the tag.
- crates.io `lightning` version index (publication dates).
- `ldk-node` `Cargo.toml` at v0.7.0 and on `main` (dependency pins).
- PR numbers above are `lightningdevkit/rust-lightning` PRs.
