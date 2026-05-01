# Stacks Proof-of-Transfer (PoX) Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/stacks`.
> Canonical source: https://github.com/stacksgov/sips/blob/main/sips/sip-007/sip-007-stacking-consensus.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/stacks/SKILL.md

## Concept

Proof-of-Transfer (PoX) is the consensus mechanism of the Stacks
blockchain. Instead of burning energy (PoW) or capital (PoS), Stacks
miners **transfer BTC** to a set of recipient Bitcoin addresses chosen by
**Stackers** (STX holders who lock STX). Whoever transfers the most BTC
in a round earns the right to mine the next Stacks block; recipient
Stackers get the BTC as yield.

The result: Stacks consensus is anchored to Bitcoin, every Stacks block is
finalised after \~150 BTC blocks, and Stackers earn native BTC for
locking STX.

## Walkthrough / mechanics

### Reward cycle

A **reward cycle** is 2100 Bitcoin blocks (~2 weeks). Each cycle has two
phases:

1. **Prepare phase** (last 100 blocks of previous cycle): Stackers
   announce their lock and the BTC reward address.
2. **Reward phase** (next 2000 blocks): miners send BTC to a sample of
   reward addresses; miner-elections occur every Bitcoin block.

### Reward set

At the start of each cycle, Stacks consensus computes the **reward set**:
2000 BTC payout slots filled by Stackers proportional to STX locked.
Minimum lock to qualify: `floor(total_locked / 2000)` (the "stacking
threshold"). Stackers locking less must **delegate** to a pool that
aggregates above the threshold.

### Sortition (per Bitcoin block)

For each Bitcoin block in the reward phase:

1. Each Stacks miner submits a Bitcoin **commit transaction** with two
   outputs paying two reward addresses (sampled deterministically from
   the reward set per block height). Output amounts equal the miner's
   bid.
2. The commit also encodes a VRF proof + the previous Stacks block hash
   in OP_RETURN.
3. The "winner" of the sortition is selected probabilistically weighted
   by BTC committed: `P(miner wins) = miner_BTC / sum(all_BTC)`.

Winning miner produces the next Stacks block. Loser miners' BTC is also
spent to the reward addresses (so all BTC committed becomes Stacker
yield).

### Stacking threshold and delegation

Threshold scales with locked supply. To delegate:

1. STX holder calls `delegate-stx` Clarity contract specifying delegate
   pubkey + max amount.
2. Pool operator calls `delegate-stack-stx` to lock individual delegations
   into pool's reward address.
3. Operator calls `stack-aggregation-commit` to register the aggregated
   pool entry in the next reward set.

### Forks and finality

Stacks has its own fork-choice — the longest "chain weight" branch wins —
but all branches share the Bitcoin block as the **sortition oracle**. A
Stacks block is **Bitcoin-finalised** once 150 Bitcoin blocks have
confirmed its anchoring sortition (Nakamoto-style probabilistic finality).

## Worked example

Cycle 80, 100M STX locked, 2000 reward slots:

| Quantity | Value |
|----------|-------|
| Lock threshold | 50 000 STX |
| Stackers (>= threshold) | 1500 (each gets >=1 slot) |
| Delegate pools | 5 pools aggregating ~500 slots |
| Total slots | 2000 |
| Cycle BTC payout | sum of all miner commits over 2000 blocks |

Per-block miner commit example: 0.05 BTC paid 50/50 to two reward
addresses. Across 2000 blocks ~100 BTC distributed among Stackers.

## Common pitfalls

- **Underbid attacks**: miner commits zero BTC (or minimum) to participate
  but still gets sortition odds proportional to commit. Counter: floor
  + sortition weight algorithm penalises sub-threshold commits.
- **Reorg sensitivity**: a Bitcoin reorg of >2 blocks can flip Stacks
  block elections. Stacks Nakamoto release moved to a 150-block
  finalisation rule, so wallets see "anchored" status for older blocks.
- **Reward address signedness**: Stackers must pre-commit a Bitcoin
  scriptPubKey in `pox-info`. If they pass a pubkey hash for a wallet
  they don't control, BTC is lost. Always verify on hardware.
- **Threshold drift**: between cycles the threshold can rise (more STX
  locked) and a stacker auto-loses qualification. Pools must
  re-aggregate every cycle.
- **PoX-2 / Nakamoto migration**: contract addresses changed; legacy
  `pox-2` delegations had to be re-issued for `pox-3` and again for
  Nakamoto. Always read SIP-022/SIP-024 before scripting.

## References

- SIP-007 (PoX), SIP-022 (PoX-3), SIP-021 (Nakamoto release).
- stacks-blockchain repo `src/chainstate/stacks/db/sortition.rs`.
- "Proof of Transfer" — Ali, Freeland, Blankstein 2020 paper.
