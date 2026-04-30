# Routing Cost Function - Deep Walkthrough

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/routing`.
> Canonical source: BOLT 7 `channel_update`, BOLT 11 routing hints, implementation pathfinding code
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/routing/SKILL.md

## Concept

Lightning pathfinding is shortest-path on a weighted multigraph where
the weight is a *cost function* combining base fee, proportional fee,
CLTV-delta risk, and a probability-of-success heuristic. Each
implementation has its own cost function, and the choice of weights
materially affects payment success rate. Understanding the function
lets you reason about why two implementations pick different routes
for the same payment.

## Walkthrough / mechanics

The fee per channel hop, advertised in `channel_update`:

```
fee = base_fee_msat + (amount_msat * fee_proportional_millionths) / 1_000_000
```

Naive cost (LND pre-2020):

```
cost(channel) = fee
```

This produced cheap-but-failure-prone routes (going through
illiquid hops). Modern cost functions add multiple terms.

LND apriori-probability cost (`routing/router.go`):

```
cost = fee
     + risk_penalty
     + apriori_prob_penalty

risk_penalty = amount_msat * cltv_delta * RISK_FACTOR
             // RISK_FACTOR ~ 9.5e-9 default

apriori_prob_penalty = amount_msat * (1 - prob) / prob
prob = clamp(0.6 - amount/capacity * something, 0, 1)
```

After a payment attempt fails or succeeds, mission control updates the
per-channel `prob` estimate. The cost function reuses these
posterior probabilities on subsequent pathfinding.

CLN's pathfinder uses a different formulation centered on `riskfactor`
(default 10):

```
cost = fee + riskfactor * (amount/1e6) * cltv_delta_blocks
```

CLN does not maintain explicit success-probability state in core (CLN
24.x uses `xpay` plugin's renepay model for this).

LDK uses a probabilistic scorer with explicit liquidity bounds:

```
struct ChannelLiquidity {
    min_liquidity_msat: u64,  // posterior lower bound
    max_liquidity_msat: u64,  // posterior upper bound
}
prob(amt) = (max_liquidity_msat - amt) / (max_liquidity_msat - min_liquidity_msat)
penalty = -log(prob) * AMOUNT_PENALTY_MULTIPLIER + base_penalty
```

After a successful or failed attempt, the bounds tighten:
- Failure at hop H carrying amount A → `max_liquidity = min(max, A)`.
- Success → `min_liquidity = max(min, A)`.

LDK's `ProbabilisticScorer` is the most explicit Bayesian model among
the major implementations.

## Worked example

Routing 100k sats from Alice to Bob with two candidate paths:

```
Path 1: Alice -> Hub -> Bob
  Hop 1: base=1000, prop=100 (0.01%), cltv=40, capacity=1M, recent_failures=2
  Hop 2: base=0,    prop=1   (0.0001%), cltv=40, capacity=10M, recent_failures=0

  fee     = (1000 + 100_000 * 100/1e6) + (0 + 100_000 * 1/1e6)
          = 1010 + 100 = 1110 msat
  cltv    = 80 blocks
  prob    = 0.5 (hop 1 has recent failures) * 0.95 (hop 2) = 0.475

Path 2: Alice -> Slow -> Bob
  Hop 1: base=2000, prop=500, cltv=80, capacity=200k, recent_failures=0
  Hop 2: base=0,    prop=1,   cltv=40, capacity=10M, recent_failures=0

  fee     = (2000 + 100_000 * 500/1e6) + 100 = 2050 + 100 = 2150 msat
  cltv    = 120 blocks
  prob    = 0.4 (low capacity for amount) * 0.95 = 0.38
```

LND cost with default `RISK_FACTOR=9.5e-9`:
```
Path 1 risk = 100_000_000 * 80 * 9.5e-9 = 76 msat
Path 1 prob_pen = 100_000_000 * (1 - 0.475)/0.475 = 110_526_316 msat
Path 1 total = 1110 + 76 + 110_526_316 = 110_527_502

Path 2 risk = 100_000_000 * 120 * 9.5e-9 = 114
Path 2 prob_pen = 100_000_000 * (1 - 0.38)/0.38 = 163_157_894
Path 2 total = 2150 + 114 + 163_157_894 = 163_160_158
```

LND picks Path 1 because its probability penalty is lower despite
higher recent failures (the failures are old and decayed in mission
control).

CLN with `riskfactor=10`:
```
Path 1 = 1110 + 10 * 0.1 * 80 = 1110 + 80 = 1190 msat
Path 2 = 2150 + 10 * 0.1 * 120 = 2150 + 120 = 2270 msat
```

CLN also picks Path 1, by raw fee. CLN's model is simpler but more
sensitive to base fees.

`lncli queryroutes` lets you inspect the chosen path:
```
lncli queryroutes 03dest_pubkey... 100000 \
    --fee_limit 5000 --cltv_limit 1500
```

`lightning-cli getroute`:
```
lightning-cli getroute 03dest_pubkey 100000sat 10
```

## Common bugs / pitfalls

- **Setting `base_fee_msat=0` thinking you'll attract traffic**: many
  pathfinders have minimum-fee thresholds that filter out 0-fee
  channels as suspicious (potential probing trap).
- **Capacity-as-public-information fallacy**: announced capacity
  bounds *total* funds, not the *direction-specific* liquidity. A 1M
  channel might have 1M on one side and 0 the other; pathfinder
  guesses 50/50 by default.
- **Stale `channel_update`s**: a 2-week-old update still influences
  pathfinding. BOLT 7 says updates older than 14 days should be
  treated as "stale"; some impls cache them past that.
- **Mission-control reset on restart**: LND clears mission control on
  restart by default. After a restart, success rates drop until enough
  attempts re-populate it.
- **CLTV-delta sweet spot**: too low and you're refused upstream; too
  high and you're penalized by risk factor + the receiver's
  `min_final_cltv_expiry_delta` may not be reached. Standard 40-144
  blocks works.
- **Disabled flag**: `channel_update.channel_flags` bit 1 means
  disabled. Pathfinders that ignore the bit route through dead
  channels.
- **LDK score persistence**: `ProbabilisticScorer` must be persisted
  across restarts via `Persister` trait. Forgetting to wire this loses
  routing history.

## References

- BOLT 7 channel_update: https://github.com/lightning/bolts/blob/master/07-routing-gossip.md#the-channel_update-message
- LND mission control: `routing/missioncontrol.go`
- LDK ProbabilisticScorer: `lightning/src/routing/scoring.rs`
- CLN getroute: `lightning/common/route.c`
- "Pathfinding in the Lightning Network" (Pickhardt, 2021): https://arxiv.org/abs/2107.05322
