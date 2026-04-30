# Single Random Draw (SRD) Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/coin-selection`.
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/coin-selection/SKILL.md

## Concept

Single Random Draw is the simplest coin-selection algorithm that still
produces sensible privacy properties. The wallet shuffles its UTXOs and
draws sequentially until the running sum reaches `target + fee + min_change`.
That's it.

The algorithm has been the **default fallback** in Bitcoin Core since v23
when BnB cannot find an exact match. It is also Wasabi's primary algorithm
because the random ordering breaks input-pattern fingerprints used by chain
analysts.

## Walkthrough / mechanics

```
SRD(target, utxos, fee_rate):
    shuffle(utxos)
    selected = []
    sum_eff = 0
    for u in utxos:
        selected.append(u)
        sum_eff += effective_value(u, fee_rate)
        # required = target + change_cost (we will create change)
        if sum_eff >= target + min_change_threshold:
            return selected
    return None  # not enough funds
```

Two subtleties:

1. **Effective value** is computed at the prevailing fee rate, same as BnB.
   Adding a UTXO costs ~68 vB for P2WPKH; that fee must be subtracted to
   get the contribution toward the target.
2. **Stop condition** uses a *minimum change* threshold (Bitcoin Core uses
   `change_target` typically a uniform random number between 50 000 and
   150 000 sats, randomized per draw). This prevents trivially-distinguishable
   change amounts. Bitcoin Core calls this `GenerateChangeTarget`.

The algorithm always creates change (unlike BnB). It does not optimize for
fee minimization or input count; it optimizes for entropy.

## Worked example

Same UTXO set as the BnB walkthrough:

```
A 10_000 sats
B  4_000
C  3_500
D  1_200
```

Target 6 000 sats, fee rate 10 sat/vB.

`change_target = 80 000` sats randomly chosen → effectively requires sum_eff
>= target + 80 000 = 86 000 because Core wants the change to land near a
randomized non-trivial value. *Wait* — that exceeds available funds. SRD
would return None.

In practice, Core's `change_target` is bounded by the minimum it can produce
given balance. The actual stop condition is:

```
required = target + cost_of_change_output  (typically 31 vB * fee_rate)
if sum_eff - target >= max(min_change, dust_threshold):
    stop
```

Re-running with the realistic `min_change = 1000` sats:

```
shuffle -> [C, A, B, D]   (random permutation)
take C: sum_eff = 3_500 - 680 = 2_820   < 6_000+1_000 ; continue
take A: sum_eff = 2_820 + (10_000-680) = 12_140   >= 7_000 ; STOP
```

Result: inputs `{A, C}`, change ≈ 12 140 - 6 000 = 6 140 sats. Different
shuffles produce different results — that's the point. A second run might
yield `{B, A}` or `{D, B, C, A}`.

```python
# BDK example
from bdk_python import Wallet, SignOptions, AddressIndex
selected = wallet.coin_selection_strategy("largest_first" | "single_random_draw")
# In raw Python:
import random
def srd(utxos, target, fee_rate):
    pool = list(utxos)
    random.shuffle(pool)
    chosen, s = [], 0
    for u in pool:
        chosen.append(u)
        s += u.value - u.input_vsize * fee_rate
        if s >= target + 1000:  # min_change
            return chosen
    raise InsufficientFunds()
```

## Privacy comparison vs deterministic algorithms

| Heuristic chain analysts use | BnB | Largest-first | SRD |
|------------------------------|-----|---------------|-----|
| Round number outputs reveal the change | strong | strong | weaker |
| Input ordering hints at wallet | yes (sorted) | yes | no |
| Common-input-ownership | applies | applies | applies |
| Single-input txs as fingerprint | rare | common | rare |

SRD doesn't fix the common-input-ownership heuristic — that's structural —
but it removes the "this wallet always picks the largest UTXO" signal.

## Common pitfalls

- Implementing SRD without `effective_value` math → fee under-count when
  spending small UTXOs; tx stuck.
- Using a non-cryptographic shuffle (e.g., `Math.random()`) — gives an
  attacker who can guess the shuffle some advantage. Use `secrets.SystemRandom`
  or libsodium-grade RNG.
- Skipping the dust check: if SRD's draw produces change that's below the
  dust limit (e.g., 540 sats P2WPKH), the change output is unspendable.
  Detect and either drop change into fee or pull another UTXO.
- For frequent small spends, SRD tends to produce more inputs than BnB,
  raising long-term fee cost. A sensible wallet runs BnB first, falls
  back to SRD only when BnB fails.
- Wasabi uses an SRD-variant that *also* avoids spending UTXOs from
  different anonymity sets together; vanilla SRD does not have this
  privacy guard.

## References

- Bitcoin Core change-target rationale (PR 17331): https://github.com/bitcoin/bitcoin/pull/17331
- Wasabi WabiSabi protocol coin-control: https://github.com/zkSNACKs/WabiSabi
- See also: [bnb-walkthrough.md](bnb-walkthrough.md), [waste-metric-deep.md](waste-metric-deep.md)
