# Runes vs BRC-20 - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/metaprotocols/runes`.
> Canonical source: https://docs.ordinals.com/runes/specification.html
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/metaprotocols/runes/SKILL.md

## Concept

BRC-20 (May 2023) and Runes (April 2024) solve the same problem -
fungible tokens on Bitcoin - with very different architectures.
BRC-20 piggybacks on the inscription envelope and JSON parsing,
relying on off-chain indexers for state. Runes is a native UTXO-
bound protocol whose validation rules are spec-defined and parsed
from compact binary OP_RETURN payloads. The choice between them
shapes fee profile, indexer trust model, programmability, and
exchange support.

## Walkthrough / mechanics

| Dimension | BRC-20 | Runes |
|-----------|--------|-------|
| State carrier | Inscription as bearer token | UTXO with implicit balance |
| Validation | Off-chain indexer convention | Protocol-defined cenotaph rules |
| Encoding | JSON in inscription body | LEB128 varints in OP_RETURN |
| Avg op size | ~150-300 vB | ~30-100 vB |
| Multi-op tx | One JSON, one inscription | Multiple edicts per tx |
| Mint race | Inscribe + first-confirm wins | Mint tx + cap reached, FCFS |
| Decimals | `dec` field, default 18 | `divisibility` 0..38 |
| Front-run protection | None (visible in mempool) | 6-block commit-reveal for names |
| Indexer divergence risk | High (multiple impls, edge cases) | Low (deterministic from spec) |
| Wallet UX | Two-step transfer awkward | Single-tx transfer |
| Privacy | Address-bound, indexer-public | UTXO-bound, indexer-public |
| Activation | None (just inscriptions) | Block 840_000 only |

## Worked example  (encoded data, real txs, configs)

Identical "send 100 tokens" intent on both protocols.

BRC-20 step 1 (inscribe transfer, 130 vB):

```
inscription body: { "p": "brc-20", "op": "transfer",
                    "tick": "ordi", "amt": "100" }
fee at 50 sat/vB: 6_500 sats for inscribe
```

BRC-20 step 2 (send the inscription, 110 vB):

```
input: the transfer inscription UTXO (546 sats)
output: P2TR to recipient (546 sats)
fee at 50 sat/vB: 5_500 sats
```

Total cost: 12_000 sats + 1_092 sats locked dust.

Runes equivalent (one tx, ~150 vB including OP_RETURN):

```
input:  UTXO holding 1_000 of rune 840000:1
output0: OP_RETURN runestone
         edict: (840000, 1, 100, 1)
         pointer: 2
output1: P2TR to recipient (546 sats)
output2: P2TR to self for change (546 sats)
fee at 50 sat/vB: 7_500 sats
```

Total cost: 7_500 sats + 1_092 sats dust. Runes is ~38% cheaper at
peak congestion, with a single confirmation instead of two.

For a deploy / etch comparison, BRC-20 deploy is roughly 100 vB
(small JSON). Runes etching with `terms` is roughly 200 vB. BRC-20
wins on raw etch size but loses on every transfer thereafter.

## Trade-offs / pitfalls

Pick BRC-20 if you need:

- Compatibility with the broadest existing wallet ecosystem (UniSat,
  OKX Web3, Magic Eden, ME marketplace flow). Indexer adoption is
  more mature.
- Trading volume. BRC-20 still has more secondary-market liquidity
  in late 2025 than Runes for established tickers.
- Tickers under 4 chars that are already taken on Runes (or vice
  versa). The two namespaces are independent.

Pick Runes if you need:

- Lower per-transfer fees, especially during congestion.
- Atomic multi-rune transfers (multiple edicts per tx).
- Cleaner protocol semantics, less indexer-disagreement risk.
- Anti-front-run for the etching of named tokens.
- Ability to run a wallet without trusting any external indexer
  (all state derivable from chain bytes alone).

Hard pitfalls common to both:

- **Privacy**: both protocols force address re-use and create on-
  chain ledgers indexable by anyone. CoinJoin destroys token
  identity on Runes UTXOs and BRC-20 inscriptions alike.
- **Exchange support**: many CEXes accept neither for deposit; some
  accept BRC-20 but not Runes or vice versa. Always check the
  destination's deposit-asset list before moving balance.
- **Atomical / Spark / other variants**: the question is not just
  "BRC-20 vs Runes" - other token standards exist. Cross-standard
  arbitrage is a real source of value loss when users mistake one
  for the other.
- **Cenotaph hazard (Runes)**: a malformed runestone burns balances.
  A buggy JSON parse on BRC-20 only fails the op silently. The
  Runes failure mode is less forgiving.
- **Fee spikes around halvings or new etchings**: both ecosystems
  produce mempool storms that affect non-token users too.

## References

- Runes spec: https://docs.ordinals.com/runes/specification.html
- BRC-20 indexer: https://github.com/bestinslot-xyz/OPI
- Comparative analysis: https://research.figmentcapital.io/p/runes-and-brc-20
