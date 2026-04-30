# Branch and Bound Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/coin-selection`.
> Canonical source: Murch's "An Evaluation of Coin Selection Strategies" (2016)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/coin-selection/SKILL.md

## Concept

Branch and Bound (BnB) is the only mainstream coin-selection algorithm that
**actively avoids creating change**. It searches the UTXO set for a subset
whose sum lands within a tight `[target, target + cost_of_change]` window.
When BnB succeeds, the wallet creates a transaction with no change output —
saving 31 vB on P2WPKH or 43 vB on P2TR plus the future spend cost of that
change UTXO.

It is implemented in Bitcoin Core (`SelectCoinsBnB`), BDK
(`BranchAndBoundCoinSelection`), and most modern wallets. SRD and knapsack
are fallbacks for when no exact subset exists.

## Walkthrough / mechanics

The algorithm performs depth-first search over UTXOs sorted descending by
*effective* value (value minus the cost to spend that input). At each level
it branches into "include this UTXO" or "exclude". Two cuts prune the tree:

1. **Upper bound** — if `current_sum + remaining_sum < target`, no inclusion
   pattern can reach the target; backtrack.
2. **Tight bound** — if `current_sum > target + cost_of_change`, this branch
   would force a change output anyway; backtrack (BnB only cares about
   exact-match-ish solutions).

A run is bounded to ~100 000 iterations in Bitcoin Core (`TOTAL_TRIES`); on
failure the wallet falls through to SRD with `min_change` set to a typical
value.

```
BnB(target, utxos, cost_of_change):
    sort_desc_by_effective_value(utxos)
    best = None
    best_waste = +inf
    iter = 0
    stack = [(0, [], 0)]   # (depth, included, sum)
    while stack and iter < TOTAL_TRIES:
        iter += 1
        depth, included, s = stack.pop()
        if depth == len(utxos):
            if target <= s <= target + cost_of_change:
                w = waste(included, s, target)
                if w < best_waste: best, best_waste = list(included), w
            continue
        u = utxos[depth]
        # branch: skip u
        stack.append((depth+1, included, s))
        # branch: take u (only if it doesn't blow past target+cost_of_change)
        if s + u.eff <= target + cost_of_change:
            stack.append((depth+1, included + [u], s + u.eff))
    return best
```

The "effective value" trick is the key: each UTXO's value is reduced by
`vsize_to_spend × fee_rate` so that the search target is the *post-fee*
amount, not the gross output. Without this, BnB would consistently
under-fund.

## Worked example

Suppose the wallet has these confirmed UTXOs (P2WPKH; effective input
vsize 68, fee rate 10 sat/vB → 680 sats per input):

```
utxo  value      eff = value - 680
A     10_000     9_320
B      4_000     3_320
C      3_500     2_820
D      1_200       520
```

User wants to send `target = 6_000` to a P2WPKH recipient. Output cost:
31 vB × 10 = 310 sats. `cost_of_change` ~= 31 (output) + 68 (future spend) =
99 vB worth of fee = 990 sats at 10 sat/vB.

```
sorted desc by eff: [A(9320), B(3320), C(2820), D(520)]

Try A only:    sum=9320  > 6000+990 -> backtrack
Try B+C:       sum=6140  -> inside [6000, 6990] ! candidate, waste = excess (140) + small input fee = ~140
Try B+C+D:     sum=6660  -> inside, waste ~660
Try B+D:       sum=3840  < 6000 -> backtrack
Try C+D:       sum=3340  < 6000 -> backtrack
Try B alone:   3320 < 6000 -> backtrack
```

Two valid solutions; BnB picks `B+C` (lower waste). Resulting tx has two
inputs, one output (recipient), and 140 sats of "excess" rolled into fee.
No change output. Recipient sees `6_000` exactly.

## Common pitfalls

- **Too tight a `cost_of_change` window**: if the wallet uses 0, BnB will
  almost always fail because exact-match is rare. Use a realistic estimate:
  output_vsize × fee_rate + future_input_vsize × long_term_fee_rate.
- **Effective value wrong**: forgetting to subtract per-input fee means BnB
  finds a "match" that under-pays the actual fee — tx gets stuck.
- **Long-term fee rate vs current rate**: the cost to spend change *later*
  uses an estimate of future fees. Bitcoin Core uses `m_long_term_feerate`,
  defaulted to a 1008-block estimate. Setting both rates equal biases away
  from change creation; using a much lower long-term rate (e.g., assuming
  fees fall) makes change look cheap and BnB rejects it less often.
- **Fee bumping interaction**: a tx built with BnB has no change output, so
  RBF must reduce one of the inputs' implied fee by lowering payment
  amount (rarely OK) or add a new input. Watch out — the Updater logic must
  fall back to a "change-creating" rebuild when bumping.
- **Privacy leak**: a no-change tx is a fingerprint; chain analysts know the
  recipient amount equals the output amount minus 0 (no candidate change).
  Mitigate by sometimes producing change anyway (`avoid_partial_spends` is
  unrelated; consider SRD when privacy matters).

## References

- Murch's coin-selection paper: https://murch.one/wp-content/uploads/2016/11/erhardt2016coinselection.pdf
- Bitcoin Core implementation: `src/wallet/coinselection.cpp` `SelectCoinsBnB`
- BDK implementation: https://github.com/bitcoindevkit/bdk/blob/master/crates/wallet/src/wallet/coin_selection.rs
- See also: [srd-walkthrough.md](srd-walkthrough.md), [waste-metric-deep.md](waste-metric-deep.md)
