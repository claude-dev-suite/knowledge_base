# Consensus Cleanup (BIP54) - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/proposals`.
> Canonical source: BIP54
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/proposals/SKILL.md

## Concept

BIP54 "Consensus Cleanup" (Antoine Poinsot, Matt Corallo; assigned
2025-04-11; **Status: Complete** as of version 1.0.0, 2026-05-22) is
the most advanced soft-fork proposal in the Bitcoin pipeline and the
only one in this directory that adds no new capability. It bundles
fixes for four long-standing defects that all date to the 2009
codebase. The BIP's own rationale for bundling: "Bundling these fixes
together amortizes the fixed cost of deploying a Bitcoin soft fork."

Every rule it adds is a **tightening**. The BIP states there is "no
block that is valid under the rules proposed in this BIP but not under
the existing Bitcoin consensus rules", so non-upgraded nodes are not
forked off.

The skill lists the four defects; this article walks each rule as
specified, the Core versions where the mitigations have already
landed as policy or template rules, and the activation situation as
of September 2026.

## Walkthrough / mechanics

All four rules apply to blocks after activation.

**1. Timewarp (and the Murch-Zawy variant).**

The difficulty-adjustment off-by-one lets a majority-hashrate attacker
arbitrarily lower difficulty. The BIP quantifies the worst case: an
attacker "can bring down the difficulty to its minimum within 38 days
of starting the attack".

```
Given a block at height N:

  if N % 2016 == 0     ->  T_N  >=  T_(N-1) - 7200
  if N % 2016 == 2015  ->  T_N  >=  T_(N-2015)
```

The first clause is the time-warp fix proper, with a two-hour grace
period. The BIP notes an earlier proposal used a ten-minute grace
period, "and this approach has been adopted for testnet4" (BIP94);
7200 seconds was chosen "out of an abundance of caution". The second
clause requires a whole difficulty period to have non-negative
duration, which is the Murch-Zawy mitigation.

**2. Worst-case block validation time.**

```
For each input in a (non-coinbase) transaction, count CHECKSIG and
CHECKMULTISIG in:
  - the input scriptSig
  - the previous output's scriptPubKey
  - the P2SH redeemScript
using BIP16 accounting:
  CHECKSIG / CHECKSIGVERIFY                      -> 1
  CHECKMULTISIG(VERIFY) preceded by OP_1..OP_16  -> 1..16
  all other CHECKMULTISIG(VERIFY)                -> 20
(counted whether or not they are evaluated)

If the sum over all inputs is strictly > 2500, the tx is invalid.
```

The BIP says 2500 "was chosen as the tightest value that did not make
any non-pathological standard transaction" invalid. Note the scope:
**legacy** sigops only, and the coinbase is exempt.

**3. Merkle-tree weakness.**

```
A transaction whose witness-stripped serialized size is exactly
64 bytes is invalid.
```

A 64-byte non-witness transaction is indistinguishable from an inner
node of the merkle tree, which lets an attacker misrepresent the
contents of a valid block to anyone verifying a merkle proof. The BIP
observes that "64-byte transactions can only contain a scriptPubKey
that lets anyone spend the funds, or one that burns them", and that
they have been non-standard since 2019 and unused since 2016, so the
rule costs nothing real. Alternatives that were considered and
rejected: committing to merkle tree depth in the header version field,
or invalidating inner nodes that decode as transactions. This fix has
its own standalone spec, **BIP53 "Disallow 64-byte transactions"**
(Chris Stewart, Draft).

**4. Duplicate transactions / BIP30.**

```
The coinbase transaction's nLockTime must equal (block height - 1),
and its nSequence must not be 0xffffffff.
```

Since BIP34 activation, explicit BIP30 validation is unnecessary only
until block height **1,983,702** - the earliest future block that
could contain a duplicate coinbase. Resuming BIP30 checks then would
cost every node a database lookup per block. Forcing height into the
coinbase locktime makes coinbase transactions unique forever, so BIP30
can be skipped permanently. A side benefit the BIP notes: applications
can recover a block's height from the coinbase without parsing BIP34
scriptSig encodings. The cost: `nLockTime` is no longer available for
extranonce rolling by ASIC controllers.

## Worked example

Three of the four mitigations already affect software you can run
today, as policy or block-template rules rather than consensus. This
is what "forward compatibility" means in the BIP's own section of
that name.

**64-byte transactions - Bitcoin Core 0.16.1 and later**

Already neither relayed nor included in block templates. Nothing to
do.

**Legacy sigops - Bitcoin Core 30.0**

The 30.0 release notes, under Policy:

```
The maximum number of potentially executed legacy signature operations
in a single standard transaction is now limited to 2500. Signature
operations in all previous output scripts, in all input scripts, as
well as all P2SH redeem scripts (if there are any) are counted toward
the limit. The new limit is assumed to not affect any known typically
formed standard transactions. The change was done to prepare for a
possible BIP54 deployment in the future. (#32521)
```

So a node running 30.0 will already refuse to relay a transaction that
BIP54 would make invalid - without enforcing any new consensus rule.

**Timestamps - Bitcoin Core 29.0 and later, extended in #35949**

29.0 and later will not generate a block template violating the new
timestamp restrictions. Core PR #35949 "miner: Enforce Murch-Zawy rule
(BIP54)", merged 2026-09-07, closed the remaining gap: for the last
block of each 2,016-block period the minimum timestamp must be at
least that of the period's first block, and `getblocktemplate` now
adjusts both `mintime` and the proposed `curtime` when the node's
clock is behind that minimum. This applies on **all** networks and
changes no consensus validation.

The BIP's standing advice to miners: use the `curtime` or `mintime`
field from `getblocktemplate` rather than your own clock. That was
never optional - a timestamp below `mintime` already produced an
invalid block.

**Coinbase construction - mining pools**

The BIP notes that coinbase transactions are usually crafted by pool
software, for which "there does not exist an open source reference
broadly in use today", and asks pools to make their coinbase
transactions forward-compatible. Setting `nLockTime = height - 1` is
what "BIP54-compatible coinbase" means in reporting on pool readiness.

## Activation status (as of September 2026)

| Item | State |
|------|-------|
| BIP54 status | Complete (v1.0.0, 2026-05-22) |
| Activation parameters | **None chosen.** No BIP9/BIP8 deployment exists |
| Bitcoin Core | PR #35793, consensus rules without mainnet activation - open, unmerged |
| Bitcoin Inquisition | Implementation of the BIP54 rules for signet (Optech, 2026-02-13) |
| Signet | Slow-block demonstration run (Optech, 2026-05-01) |
| Testnet5 | Draft BIP would activate BIP54 from block 1 (Optech, 2026-06-12) |

**Pool readiness.** mainnet-observer tracks which pools have shipped
forward-compatible coinbase construction. Its "First BIP-54 Coinbase
Transactions by Pool" table, fetched 2026-09-15, reads:

| Pool | First BIP54 coinbase | Date | Blocks |
|------|---------------------|------|--------|
| WhitePool | 937,404 | 2026-02-19 | 28 |
| Unknown | 940,484 | 2026-03-13 | 2 |
| MARA Pool | 940,548 | 2026-03-13 | 1,278 |
| ViaBTC | 949,094 | 2026-05-12 | 1,527 |
| Solo CK | 951,408 | 2026-05-28 | 4 |
| Foundry USA | 952,880 | 2026-06-08 | 3,605 |
| CKPool | 956,021 | 2026-06-30 | 2 |

The companion "BIP-54 Coinbase Locktime set" series put the share of
coinbase transactions with a BIP54 locktime between 33% and 49% per
day over 2026-09-01..15. That is **not** signaling; it is
forward-compatible coinbase construction and cannot lock in a soft
fork.

The remaining resistance is operational rather than ideological.
F2Pool uses the coinbase `nLockTime` field to store job metadata so a
disconnected miner can resubmit shares on reconnect; MARA used it the
same way and switched to the coinbase `nVersion` after Poinsot's
bitcoinminingdev post. Poinsot reported on 2026-02-23 that F2Pool
does not intend to follow "until BIP 54 is further along the way
toward activation" - which he called "a nice catch-22 in itself".
ViaBTC's upgrade is partial: only some of its blocks are compatible,
apparently because it runs several pool-software versions.

## Common bugs / pitfalls

1. **Reading "Status: Complete" as "activated".** Under BIP-3,
   Complete means the authors concluded all planned work and
   recommend adoption, with a reference implementation and test
   vectors in hand. `Deployed` is the status that implies activation
   criteria were met on the network. BIP54 is Complete and not
   deployed.
2. **Confusing forward-compatible coinbases with BIP9 signaling.**
   Only version-bit signaling in a real deployment can lock in a soft
   fork, and no such deployment exists for BIP54.
3. **Treating the 2500-sigop cap as consensus today.** In Bitcoin Core
   30.0 it is a *standardness* rule. A miner can still include a
   transaction exceeding it and the block is valid.
4. **Assuming the sigop cap covers segwit or tapscript.** It counts
   legacy `CHECKSIG`/`CHECKMULTISIG` in scriptSig, prevout
   scriptPubKey and P2SH redeemScript, using BIP16 accounting. Witness
   sigops have their own pre-existing limits.
5. **Assuming the 64-byte rule is about total size.** It is about the
   **witness-stripped** serialized size being exactly 64 bytes. A
   segwit transaction with a large witness can still trip it.
6. **Expecting `nLockTime` to stay free in coinbases.** The BIP takes
   it for the height encoding, which removes it as an extranonce
   rolling field for ASIC controllers. Pool software that rolls
   coinbase locktime needs to change.
7. **Citing a merged Core PR as activation.** #35949 is merged and
   changes only block-template creation. #35793, which implements the
   consensus rules without mainnet activation, is still open.

## References

- BIP54: https://github.com/bitcoin/bips/blob/master/bip-0054.md
- BIP53 (64-byte transactions, standalone): https://github.com/bitcoin/bips/blob/master/bip-0053.mediawiki
- Optech topic: https://bitcoinops.org/en/topics/consensus-cleanup-soft-fork/
- Core PR #35793 (implementation, no mainnet activation): https://github.com/bitcoin/bitcoin/pull/35793
- Core PR #35949 (miner: Enforce Murch-Zawy rule): https://github.com/bitcoin/bitcoin/pull/35949
- Core 30.0 release notes (2500 legacy sigops policy cap): https://bitcoincore.org/en/releases/30.0/
- Core 29.0 release notes (template timestamp rules): https://bitcoincore.org/en/releases/29.0
- BIP94 testnet4 timewarp fix: https://github.com/bitcoin/bips/blob/master/bip-0094.mediawiki
- BIP-3 status semantics: https://github.com/bitcoin/bips/blob/master/bip-0003.md
- bip54.org: https://bip54.org/
- mainnet-observer, first BIP-54 coinbase by pool: https://mainnet.observer/charts/mining-pools-mining-bip54-coinbase/
- mainnet-observer, BIP-54 coinbase locktime set: https://mainnet.observer/charts/transactions-coinbase-locktime-bip54/
- BNOC, "Forward-compatible coinbase locktimes for BIP-54": https://bnoc.xyz/t/forward-compatible-coinbase-locktimes-for-bip-54/75
