# Hashrate Math Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/mining/pow`.
> Canonical source: Bitcoin Core `src/pow.cpp`, `src/rpc/blockchain.cpp::GetNetworkHashPS`
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/mining/pow/SKILL.md

## Concept

Hashrate is the number of SHA256d block-header hashes a miner (or the network)
performs per second. It cannot be measured directly - only inferred from the
rate at which valid blocks (or pool shares) are produced relative to the
target.

The fundamental identity: a miner with hashrate `H` working against a target
`T` finds a valid hash with probability `T / 2^256` per attempt, so the
expected time per solution is

```
E[t] = 2^256 / (H * T)   = (2^256 / T) / H   = D * 2^32 / H
```

where `D = max_target / T = T_max / T` and `T_max ~= 2^224` (the genesis
target's effective max). The factor `2^32` falls out of the historical
normalisation that "difficulty 1" requires `2^32` expected hashes.

## Walkthrough / mechanics

### Difficulty to expected hashes

```
expected_hashes_per_block = difficulty * 2^32
```

So at difficulty `D`, the network performs on average `D * 2^32` hashes per
block. With a target block time of 600 seconds:

```
network_hashrate = D * 2^32 / 600   [hashes / second]
```

### Share difficulty

Pools issue shares at a much lower difficulty `D_share` so miners hit them
frequently. A share found at difficulty `D_share` is also a block iff its
hash also satisfies `hash <= network_target`. The probability a share is
also a block is `D_share / D_network`.

For variable-difficulty (vardiff) pools, `D_share` is tuned so each miner
submits roughly 1 share every 5-30 seconds regardless of hashrate.

### Pool hashrate from share rate

If a miner submits `n` shares of difficulty `D_share` over time `t`, their
hashrate estimate is

```
H = n * D_share * 2^32 / t
```

Pool dashboards typically use a 5-minute or 1-hour rolling window for this
estimate. Variance is high at short windows; standard deviation of the
estimate is `H / sqrt(n)`.

### Time to find a block (variance)

Block discovery is a Poisson process. With expected time `mu = D * 2^32 / H`,
the actual time follows an exponential distribution:

```
P(t) = (1/mu) * exp(-t/mu)
```

So `P(no_block_in_2*mu) = exp(-2) ~= 13.5%`. The probability of going
3 hours without a block at network scale is non-trivial - this is "luck".

## Worked example

A miner runs an Antminer S21 rated at 200 TH/s = `2 * 10^14` H/s. Network
difficulty is `D = 95_000_000_000_000` (95 T).

Expected solo block time:

```
mu = 95e12 * 2^32 / 2e14
   = 95e12 * 4.295e9 / 2e14
   = 4.080e23 / 2e14
   = 2.040e9 seconds
   ~= 64.7 years
```

So a single S21 expects to find one block in ~65 years - hence pooling.

Pool share rate at `D_share = 65_536`:

```
shares_per_second = H / (D_share * 2^32)
                  = 2e14 / (65536 * 4.295e9)
                  = 2e14 / 2.815e14
                  = 0.71 shares/sec
```

So the miner submits about one share every 1.4 seconds. Over an hour, they
submit ~2560 shares.

Daily expected payout from a 1% pool of total 800 EH/s, with 144 blocks/day
times 3.125 BTC subsidy:

```
miner_share = H_miner / H_network = 2e14 / 8e20 = 2.5e-7
daily_btc   = 144 * 3.125 * 2.5e-7 = 0.0001125 BTC = 11250 sats
```

Plus fees, minus pool fee.

## Common pitfalls

- **Confusing share difficulty with network difficulty** - 1 share at
  `D_share=65536` is NOT 1/65536 of a block. The share is a probabilistic
  proof of work effort, not a fractional block.
- **Using subsidy alone** - fee revenue (esp. during high-fee periods) can
  exceed subsidy. Always include `coinbasevalue`, not just `subsidy`.
- **Short measurement windows** - estimating hashrate from < 100 shares
  yields ~10% standard error; pool "1m hashrate" displays are noise.
- **Forgetting hardware errors** - rejected/stale shares mean true hashrate
  is higher than accepted hashrate. A 1% HW error rate burns 1% of revenue.
- **Conflating GH/s and TH/s** - `1 TH/s = 1000 GH/s = 10^12 H/s`. Easy to
  drop a factor of 1000 in spreadsheets.

## References

- Bitcoin Core `src/rpc/blockchain.cpp` GetNetworkHashPS
- "Bitcoin Mining Math" - Braiins blog series
- Slush Pool Vardiff analysis
- Skill: `bitcoin/mining/difficulty`
