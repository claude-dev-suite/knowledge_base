# Bitcoin Soft Fork History Timeline - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/bips`.
> Canonical source: bitcoin/bips repository, Bitcoin Core release notes
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/bips/SKILL.md

## Concept

Bitcoin's consensus has evolved exclusively through SOFT forks
(restrictions on previously-valid behavior, so old nodes still accept
new blocks as valid). The skill lists individual BIPs by topic; this
article orders them chronologically with activation mechanism,
controversy level, and what each enables. Useful for context when
reasoning about why a feature exists, what came before, and what
patterns repeat.

## Walkthrough / mechanics

| Year | BIP | Name | Activation | Notes |
|------|-----|------|-----------|-------|
| 2010 | n/a | OP_VER and friends disabled | Hard fork (early genesis era) | Pulled buggy/dangerous opcodes (`OP_CAT`, `OP_LSHIFT`, etc.). |
| 2012 | 16 | Pay-to-Script-Hash | IsSuperMajority(supermajority of last 1000 blocks) | First major opcode-style addition; enabled multisig addresses. |
| 2012 | 30 | Duplicate txid prevention | Buried via BIP90 | Response to a real chain split caused by duplicate coinbase txids. |
| 2013 | 34 | Block height in coinbase | IsSuperMajority | Set the precedent for soft-fork upgrade semantics. |
| 2014 | 65 | OP_CHECKLOCKTIMEVERIFY | IsSuperMajority | Absolute timelocks; enables HTLC patterns. |
| 2015 | 66 | Strict DER signatures | IsSuperMajority | Closed signature malleability vector. |
| 2016 | 68/112/113 | Relative locktime suite | versionbits (BIP9) | nSequence-based timelocks; foundation for Lightning. |
| 2017 | 141/143/144 | SegWit | versionbits + UASF threat | Witness segregation, BIP143 sighash, weight discount. Activated August 2017 after 17-month standoff. |
| 2021 | 340/341/342 | Taproot | Speedy Trial (BIP9 variant) | Schnorr signatures, taptweak, Tapscript. Activated November 2021. |
| 2026 | 110 | Reduced Data Temporary Softfork | Modified BIP9 + BIP8-style mandatory signaling | Failed. Signaling seldom exceeded ~2.5% against a 55% threshold; enforcing nodes split onto a chain that stalled after two blocks (August 2026). BIP status now Closed. |

**Activation mechanisms over time:**

- **IsSuperMajority** (early): require N of last 1000 block versions
  to signal the new feature. Used for BIP30/34/65/66/16. Replaced
  because it provided no clear "lock-in" semantics.
- **BIP9 versionbits**: 24 bits in block version reserved for
  signaling. 95% threshold per 2016-block period, with timeout. Used
  for BIP68/112/113 and SegWit (with friction).
- **BIP148 / BIP91 (UASF)**: user-activated soft fork - economic nodes
  reject blocks NOT signaling the new rule after a date. Used to
  pressure miners into SegWit signaling in mid-2017.
- **BIP8** (proposed, never used on mainnet as such): versionbits with
  mandatory `lockinontimeout=true`, which would activate even without
  miner signaling at the deadline. BIP110 (2026) borrowed its
  mandatory-signaling window while otherwise remaining a modified BIP9
  deployment.
- **Speedy Trial** (BIP9 variant for Taproot): 90% signaling threshold,
  90-day window. If activated, lock-in. If not, no UASF fallback -
  proposers go back to drawing board. Worked smoothly for Taproot.

## Worked example

**SegWit activation timeline:**

```
2015 Dec : BIP141 first published.
2016 Apr : Bitcoin Core 0.13.0 ships SegWit code (deployment via BIP9, threshold 95%).
2016 Nov : BIP9 deployment begins.
2017 Mar : Slow signaling -> ~20-30%. Stalled.
2017 May : BIP148 UASF threat: nodes will reject non-SegWit blocks from Aug 1.
2017 Jun : NYA (New York Agreement) miners commit to SegWit2x. BIP91 deploys, 80% threshold over 336 blocks.
2017 Jul 21 : BIP91 locks in. Miners begin orphaning non-SegWit-signaling blocks.
2017 Aug 1 : BIP148 activation date (mostly moot, BIP91 already enforcing).
2017 Aug 8 : BIP141 95% threshold reached.
2017 Aug 24 : SegWit lock-in -> BIP141 active, weight discount live.
2017 Nov : Planned 2x hardfork cancelled.
```

**Taproot activation timeline:**

```
2018 Jan : Pieter Wuille floats Schnorr/MAST.
2020 Jan : BIP340/341/342 published.
2021 Apr : Bitcoin Core 0.21.1 ships Speedy Trial parameters.
2021 May 1 - Aug 11 : Signaling window. 90% threshold.
2021 Jun 12 : Block 687,284 lock-in (well before deadline).
2021 Nov 14 : Block 709,632 activation. P2TR live.
```

**Comparison:**

```
SegWit    : 17 months from BIP to activation, intense political debate, UASF used.
Taproot   : ~3.5 years from initial concept, 4 months from spec to activation,
            no controversy, "Speedy Trial" worked.
```

**BIP110 activation timeline (2026, failed):**

```
2025 Dec 03 : BIP110 "Reduced Data Temporary Softfork" assigned (author Dathon Ohm).
2026 Jun 25 : BIP110 advanced to Complete status (BIP Changelog v1.0.0; the
              repo commit landed 2026-06-24).
2026 Aug 08 : Block 961,632 (19:35 UTC) opens the mandatory-signaling window
              (961,632-963,647). Enforcing nodes reject blocks not signaling bit 4.
2026 Aug 08 : The block the network actually mined at 961,632 does not signal
              bit 4. Enforcing nodes split onto their own chain.
2026 Aug    : That minority chain produces two blocks and stalls.
2026 Aug 10 : BIPs repo sets BIP110 status to Closed (PR #2245). Luke Dashjr, who
              championed the proposal, is removed as a BIP Editor (PR #2248,
              motioned by Mark "Murch" Erhardt the previous day).
```

BIP110's declared parameters: signaling bit 4, threshold 1109/2016
(55%), `timeout` disabled in favor of a BIP8-style
`max_activation_height` of 965,664 (~1 September 2026), and an
`active_duration` of 52,416 blocks (~1 year) after which the rules
would have expired. Miner signaling seldom exceeded ~2.5% (CoinDesk,
August 2026). Note that bip110.org still presents the deployment as a
successful activation - that is the minority chain's own view, not the
state of the chain the economic majority follows.

The editor removal is a governance data point, not a consensus one: as
of September 2026 the BIP Editors are Bryan Bishop, Jon Atack, Mark
"Murch" Erhardt, Olaoluwa Osuntokun and Ruben Somsen.

The community lesson: activation mechanism design matters. BIP9 alone
without an explicit "what if it stalls" plan invites stalemate, and
BIP110 demonstrated the opposite failure - mandatory signaling against
a threshold the economic majority never intended to meet simply ejects
the enforcing nodes onto a dead chain. Future soft forks (CTV/BIP119,
still Draft; OP_CAT/BIP347, Complete; Consensus Cleanup/BIP54,
Complete) still face the unresolved question of which mechanism
replaces "Speedy Trial"; as of September 2026 no replacement has
consensus. Note that OP_VAULT (BIP345) is no longer on that list: the
BIPs repo marks it **Closed** with `Proposed-Replacement: 443`
(`OP_CHECKCONTRACTVERIFY`, Draft) as of September 2026.

## Common bugs / pitfalls

1. **Confusing soft-fork "active" with "in-effect for all nodes".** A
   soft fork is enforced by economic majority; old nodes happily
   accept post-soft-fork blocks because they look valid (just more
   restricted). This is the WHOLE point of soft forks vs hard forks.
2. **Treating IsSuperMajority deployments as permanently anchored.**
   BIP90 retroactively buried activation heights into consensus -
   modern nodes don't do supermajority counting; they use the
   hardcoded heights.
3. **Reasoning about active rules at low heights.** Pre-BIP65 (block
   388,381) blocks need not satisfy CLTV; node verification logic
   skips these rules below activation. Don't apply post-fork rules
   when validating historical blocks.
4. **Missing BIP30 corner case.** The BIP30 rule has a known
   exception at heights 91722 and 91842 (duplicate coinbases pre-BIP34
   that are unspendable). BIP90 grandfathers these.
5. **Quoting "BIP" when meaning "consensus rule".** Not all BIPs are
   consensus changes. BIP21 (URI scheme) is an interface BIP. BIP125
   (RBF) is policy-only, not consensus. Always check status field.
6. **Using the pre-2026 BIP status vocabulary.** BIP3 (Deployed since
   January 2026) replaced BIP2's Draft/Proposed/Final/Active/Replaced/
   Withdrawn set with exactly four statuses: Draft, Complete, Deployed,
   Closed. As of September 2026 the README status column holds no
   "Active" or "Final" entries at all (78 Deployed, 57 Closed, 54 Draft,
   22 Complete). Closed also does not mean "was once deployed" - BIP110
   is Closed and never activated on the chain the economic majority
   follows.

## References

- Bitcoin BIPs repo: https://github.com/bitcoin/bips
- Bitcoin Core release notes: https://github.com/bitcoin/bitcoin/tree/master/doc/release-notes
- SegWit history: https://en.bitcoin.it/wiki/SegWit
- Taproot activation: https://taproot.watch/
- BIP90: https://github.com/bitcoin/bips/blob/master/bip-0090.mediawiki
- BIP110: https://github.com/bitcoin/bips/blob/master/bip-0110.mediawiki
- BIP3 (current BIP process): https://github.com/bitcoin/bips/blob/master/bip-0003.md
