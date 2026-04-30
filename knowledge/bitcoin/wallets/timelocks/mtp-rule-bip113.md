# MTP Rule (BIP113) - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/timelocks`.
> Canonical source: BIP113 https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/timelocks/SKILL.md

## Concept

When a script-level timelock compares against unix time (rather than block
height), the question is: which "current time" do we compare against?
Naive answer: the timestamp of the block that's currently being validated.
The correct answer, since BIP113 (active August 2016): the **median time
past** (MTP) — the median of the *previous 11 blocks'* timestamps.

This rule blocks a small but real class of miner manipulation. Block
timestamps are loosely constrained (a block can be up to 2 hours into the
future and must exceed MTP), so an individual miner could backdate or
forward-date their block by up to a couple of hours. BIP113 forces locks
to use a value that is much harder to manipulate because it requires
collusion across many blocks.

## Walkthrough / mechanics

### Where MTP applies

| Mechanism | Pre-BIP113 | Post-BIP113 |
|-----------|------------|-------------|
| `tx.nLockTime` (time mode) | block timestamp | **MTP** of last 11 |
| Script `OP_CHECKLOCKTIMEVERIFY` (time mode) | block timestamp | **MTP** |
| `nSequence` relative time (BIP68 type 2) | parent block timestamp | **MTP at parent** |
| `OP_CHECKSEQUENCEVERIFY` (time mode) | parent block timestamp | **MTP at parent** |

Block-height-based locks are unaffected — heights are integers and not
manipulable by individual miners.

### Computing MTP

```
def mtp(tip):
    times = sorted(tip.timestamp for tip in last_11_blocks(tip))
    return times[5]   # the median
```

Edge case: at chain heights below 11, the median is over however many
blocks exist. Most validation paths sidestep this by enforcing BIP113 only
after activation height.

### Why "past" matters

MTP is computed over the *previous* 11 blocks, not including the current
block. This is intentional: at validation time, the current block is being
proposed, and its timestamp is one of the inputs the validator must
verify. Using MTP-of-past blocks gives a stable target — by the time the
validator sees the block, the previous 11 are immutable.

### Behaviour during reorgs

A reorg can change MTP because the "last 11 blocks" set changes. In
practice this is rarely material: median calculations are stable against
single-block changes (a single different timestamp at depth 11 cannot move
the median by more than one position).

## Worked example

A CLTV script that should activate on January 1, 2026 (unix
1 767 225 600):

```
<1767225600> OP_CHECKLOCKTIMEVERIFY OP_DROP <pk> OP_CHECKSIG
```

Spending tx must set `nLockTime = 1767225600` (or higher). At
validation time, the node computes `MTP` of the previous 11 blocks. If
`MTP >= 1767225600`, the lock is satisfied.

Demonstration with bitcoin-cli:

```bash
# Get tip and the last 11 timestamps
tip_height=$(bitcoin-cli getblockcount)
for i in $(seq 0 10); do
  h=$((tip_height - i))
  bitcoin-cli getblockheader $(bitcoin-cli getblockhash $h) | jq .mediantime
done | sort -n | head -6 | tail -1
# That's the MTP at tip
```

`mediantime` in `getblockheader` output is exactly this value:

```json
{
  "hash": "00000000...",
  "time": 1735689600,
  "mediantime": 1735687800,    <-- MTP of last 11 blocks
  ...
}
```

Note: `mediantime` lags `time` by ~1-2 hours typically. A wallet
building a CLTV-time spending tx must use a target value such that
`tx.nLockTime <= MTP_at_intended_block_height`, which means accounting
for the lag.

### CSV time-based example

Vault with relative time-lock of 24 hours:

```
DELTA = 0x00400000 | (86400 / 512) = 0x004000A8   (~24 hours in 512s units)
```

The script `<0x004000A8> OP_CHECKSEQUENCEVERIFY` checks whether the input
spends a UTXO at least 24 hours old, where "age" = `MTP_at_spending_block
- MTP_at_funding_block`.

If the funding tx confirmed at MTP_F = 1735687800 and the spending tx is
in a block whose MTP is 1735776000, the elapsed time is 88 200s ≈ 24.5
hours → satisfies the lock.

## Implications

- **Wallet UI lag**: a wallet showing "spendable in 24 hours" should use
  MTP-derived projections, which are typically 1-2 hours behind wall
  clock. A naive countdown looks complete to the user but the lock is
  not yet satisfied.
- **Cross-implementation parity**: every full-node implementation
  computes MTP identically. There's no implementation drift to worry
  about.
- **Lightning HTLCs** universally use block-height locks rather than
  time, partly to avoid the MTP cognitive overhead. BOLT-2 mandates
  height-based timeouts.

## Common pitfalls

- **Using `block.time` as the MTP proxy**: works "in practice" because
  miners can't drift far, but breaks for vaults at the boundary. Always
  use `mediantime` from `getblockheader`.
- **Forgetting MTP applies to BIP68 time-relative locks too**: a vault
  using `nSequence = 0x00400048` (~10 hours) is comparing MTP-at-spend to
  MTP-at-fund, not block timestamps.
- **Setting `nLockTime` to "now"** when building a tx for inclusion in
  the next block: nodes reject because MTP < `nLockTime`. Use
  `MTP_now - safety_buffer` (e.g., `mediantime - 600`).
- **Cross-network wallet code**: regtest blocks have arbitrary
  timestamps; MTP can be wildly different from wall clock. Don't assume
  MTP ≈ now in tests.
- **Misinterpreting "median time past"**: the *past* refers to the
  blocks, not the median calculation. It is the median of timestamps
  from past blocks — not "the past median".

## References

- BIP113: https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki
- BIP68 timestamp encoding: https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki
- See also: [cltv-vs-csv-deep.md](cltv-vs-csv-deep.md), [../vaults/single-sig-csv-walkthrough.md](../vaults/single-sig-csv-walkthrough.md)
