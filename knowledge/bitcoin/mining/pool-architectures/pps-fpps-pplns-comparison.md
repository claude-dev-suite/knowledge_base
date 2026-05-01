# PPS vs FPPS vs PPLNS Comparison - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/mining/pool-architectures`.
> Canonical source: Pool published terms (Foundry, AntPool, Braiins, Ocean, ViaBTC)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/mining/pool-architectures/SKILL.md

## Concept

A reward scheme is a contract between miner and pool: the miner contributes
hashrate (measured in shares), the pool measures contribution and pays
out. The schemes differ in *who absorbs variance*, *whether transaction
fees are passed through*, and *whether pool-hopping is profitable*.

Three dominant schemes today:

- **PPS** (Pay-Per-Share) - fixed payout per share; pool absorbs variance.
- **FPPS** (Full PPS) - PPS plus pro-rata transaction fees.
- **PPLNS** (Pay-Per-Last-N-Shares) - paid only when the pool finds a block,
  proportional to your share count in the last `N`.

## Walkthrough / mechanics

### PPS math

Per-share payout:

```
share_value = subsidy * 2^32 / network_difficulty / 2^32 * D_share / D_share
            = subsidy * D_share / network_difficulty / (2^32) * 2^32
            = subsidy * D_share / D_network
```

So a share at difficulty `D_share` on a network of difficulty `D_network`
is worth `subsidy * D_share / D_network` BTC. The pool pays this amount
*per share*, regardless of whether the pool actually found a block.

If `D_share = 65_536`, `D_network = 95e12`, `subsidy = 3.125 BTC`:

```
share_value = 3.125 * 65536 / 95e12 = 2.155e-9 BTC = 2.155 sats
```

A miner submitting 2560 shares/hour earns `2560 * 2.155 = 5517 sats/hr`.

The pool must hold enough cash reserve to pay out even during long
unlucky stretches. Industry rule of thumb: 2-4 weeks of expected revenue.

### FPPS math

FPPS layers transaction fees on top of subsidy. The pool measures the
average per-block fee revenue over a rolling window (typically 24h or 144
blocks) and pro-rates:

```
share_value_FPPS = (subsidy + avg_fee_per_block) * D_share / D_network
```

If average fees are 0.06 BTC/block:

```
share_value_FPPS = (3.125 + 0.06) * 65536 / 95e12 = 2.196e-9 BTC = 2.196 sats
```

The miner gets `2.196 / 2.155 - 1 = 1.9%` more in this example. During
high-fee periods (ordinals/runes spikes) the bonus can be 30%+.

FPPS pools must accurately track recent average fees; "lag" in updating
the rate window is a common point of dispute between pools and miners.

### PPLNS math

The pool only pays when a block is found. The block reward (subsidy + fees
minus pool fee) is split among all shares in the last `N` shares window:

```
your_payout = block_reward * (your_shares_in_N / N)
```

`N` is typically chosen so the window covers ~1-2 expected block intervals
worth of shares. A common formula: `N = X * D_network / D_share` where
`X = 2` to `10`. Larger `X` smooths variance at the cost of "slow ramp"
when joining.

For a miner at 200 TH/s on a pool with 100 EH/s total:
- Pool finds ~144 * 100/800 = 18 blocks/day expected.
- Miner's share fraction: 200e12 / 100e18 = 2e-6.
- Per-block payout: `(3.125 + 0.06) * 0.95 * 2e-6 = 6.05e-6 BTC = 605 sats`.
- Daily expected: `18 * 605 = 10_890 sats`.

Variance: the standard deviation per day for 18 Poisson blocks with the
miner getting 605 sats per block is `sqrt(18) * 605 ~ 2566 sats`, i.e.
~24% of mean. PPS by contrast has variance ~ `1/sqrt(N_shares)` which
for 60_000 shares/day is ~0.4%.

### Pool-hopping resistance

A "pool hopper" exploits Proportional schemes by:
1. Joining a pool just after a block is found (when expected payout per
   share is highest, since round is fresh).
2. Leaving when the round drags on (expected payout per share drops if
   round is short and expected payout per share *of a not-yet-found
   block* drops as time goes by - actually grows - depends on
   formula).

PPLNS is hopping-resistant because the window is anchored to the *last*
N shares regardless of round boundaries; a miner who hops out forfeits
their shares from the window once they age out. To benefit from PPLNS
you must mine consistently.

PPS / FPPS are hopping-immune because each share has a fixed value
independent of when blocks are found.

### Pool fee economics

Typical 2025 fees:

| Scheme | Fee | Why |
|--------|-----|-----|
| PPS    | 4-5% | Pool bears all variance; needs reserve premium |
| FPPS   | 2-3% | Pool bears variance + fee accounting overhead |
| PPLNS  | 1-2% | Miner bears variance |
| PPS+   | 2-4% | Hybrid |
| Solo   | 0-2% | No reward sharing; just block discovery service |

Pool revenue per TH/s served = fee_rate * block_subsidy_density. Operators
must size fees to cover bandwidth, validators, payouts engine, and a
risk buffer for variance under PPS.

## Worked example

A 200 TH/s miner runs simulations on three pools for one month (30 days):

- PPS @ 4% fee, 95e12 network, 3.125 BTC subsidy.
  - Shares/day: ~60_000 at vardiff D=65536.
  - Gross daily: `60000 * 2.155 = 129_300 sats`.
  - After fee: `129_300 * 0.96 = 124_128 sats/day = 3.72M sats/mo`.
  - Std dev / mo: `~ 0.4% * 3.72M = ~15_000 sats`. Very smooth.

- FPPS @ 2% fee, fees averaged 0.06 BTC.
  - Gross daily: `60000 * 2.196 = 131_760 sats`.
  - After fee: `131_760 * 0.98 = 129_125 sats/day = 3.87M sats/mo`.

- PPLNS @ 1% fee, N covers 2 expected blocks.
  - Mean daily: `(3.125 + 0.06) * 0.99 * 2e-6 * 144 = 9.07e-4 BTC = 90_700 sats`.
  - Std dev / mo: `~24% * 30 = sqrt(30) * 90_700 * 0.24 = 119_000 sats`.
  - Miner could end month +2 sigma (+238k sats above mean) or -2 sigma.
  - Long-run mean ~2.72M sats/mo - lower than FPPS in this scenario
    because PPLNS pool's network share is small.

Conclusion: FPPS gives smoothest income with reasonable mean; PPLNS
rewards loyal miners at the cost of variance; pure PPS underpays unless
fees are abnormally low.

## Common pitfalls

- **Comparing PPS-fee to PPLNS-fee headline rates** - apples to oranges;
  PPS rates already include variance insurance, PPLNS rates do not.
- **Ignoring fee revenue** - during ordinal/runes spikes, fees can be
  50%+ of block reward; non-FPPS schemes leave that upside on the table.
- **Misusing PPLNS at low hashrate** - if your share fraction in N is
  too small, individual block luck dominates and you may go weeks
  with zero payouts.
- **Switching pools right before a difficulty drop** - your historical
  share log doesn't transfer; you start fresh on the new pool's PPLNS
  window.
- **Trusting unaudited share counts** - PPS pools without share-log
  transparency can shave shares off your contribution. Look for pools
  publishing per-worker share streams.

## References

- Foundry USA pool terms (PPS+) <https://foundrydigital.com/>
- AntPool reward schemes <https://help.antpool.com/>
- Braiins Pool / SV2 PPLNS docs <https://braiins.com/pool>
- Ocean non-custodial PPS <https://ocean.xyz/>
- "Analysis of Bitcoin Pooled Mining Reward Systems" - Rosenfeld (2011)
