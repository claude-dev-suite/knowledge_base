# BRC-20 Deploy / Mint / Transfer - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/metaprotocols/brc-20`.
> Canonical source: https://layer1.gitbook.io/layer1-foundation/protocols/brc-20/indexing
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/metaprotocols/brc-20/SKILL.md

## Concept

BRC-20 is a JSON convention layered onto inscriptions. There is no
on-chain validation; an indexer scans every inscription, parses the
body as JSON, classifies operations as deploy / mint / transfer, and
maintains an off-chain ledger of balances per address per ticker.
Two indexers must implement the same edge-case rules to agree on
balances - a fragile property that has produced multiple disputes
and "consensus" forks in the BRC-20 indexer space.

## Walkthrough / mechanics

A BRC-20 op is an inscription whose body is JSON with `p == "brc-20"`
and one of three `op` values:

- `deploy`: registers a ticker (4 chars, case-insensitive) with max
  supply and per-mint limit. First valid deploy wins; later deploys
  of the same ticker are invalid.
- `mint`: claims `amt` of ticker. Each mint must be inscribed once.
  Indexer caps total minted at `max` (FCFS).
- `transfer`: a two-step move. Step 1 inscribes a transfer JSON in
  the holder's wallet, locking that amount into a "transferable
  balance". Step 2 sends the transfer inscription to the recipient;
  receipt moves balance.

Required inscription metadata: `content_type` should be
`text/plain;charset=utf-8` or `application/json`. Most indexers
accept either; some reject payloads inside other content types.

Address attribution: the inscription's "owner" is whoever holds the
UTXO carrying it. ord computes this by following the inscribed sat
through chain history.

## Worked example  (encoded data, real txs, configs)

Deploy `gold` with 21M cap and 1000 per mint:

```json
{ "p": "brc-20", "op": "deploy", "tick": "gold",
  "max": "21000000", "lim": "1000" }
```

Mint 1000 gold:

```json
{ "p": "brc-20", "op": "mint", "tick": "gold", "amt": "1000" }
```

Transfer step 1 (inscribed in holder's wallet):

```json
{ "p": "brc-20", "op": "transfer", "tick": "gold", "amt": "100" }
```

After step 1, the indexer view of the holder is:

```
gold:
  available:    900       (was 1000, 100 locked)
  transferable: 100       (carried by the new inscription)
```

Step 2: send the transfer inscription UTXO to recipient. After the
send confirms:

```
sender:    gold available 900, transferable 0
recipient: gold available 100
```

A real chain example (mainnet block 779_832, ticker `ordi`):

```
deploy txid:   b61b0172d95e266c18aea0c624db987e971a5d6d4ebc2aaed85da4642d635735
mint   txid:   ...                  amt=1000      thousands of mints in same window
transfer txid: ...                  amt=  10
send     txid: ...                  inscription input -> recipient output
```

## Trade-offs / pitfalls

- **Self-transfer pitfall**: sending a transfer inscription to your
  own address still consumes the transferable balance and re-credits
  available. Many users do this by accident expecting it to "cancel"
  the lock; the original spec says yes but some indexers historically
  diverged.
- **Mint race in same block**: when the cap is approached, multiple
  mints may land in the same block. Indexers order by inscription
  number (sequence in chain), not txid; so block-internal ordering
  matters and a mint that "won" by tx position in mempool may fail
  on chain.
- **Decimal limit**: `dec` field defaults to 18; `amt` uses string
  arithmetic to avoid float drift. Indexers must use big-decimal
  math, not double precision.
- **Tick collisions**: ticker is case-insensitive but visually
  case-sensitive. `Ordi` and `ordi` are the same ticker; first
  deploy wins. Some marketplaces show different cases as separate
  entries, confusing users.
- **Self-issued inscriptions**: an attacker can inscribe a transfer
  for ticker they do not own; indexer must reject because available
  balance is 0.
- **Self-mint after cap**: any mint after `max` is reached is
  invalid. The unlucky last mint that overflows the cap is rejected
  in full; partial mint is not permitted by the canonical OPI rules.
- **Indexer divergence**: OPI, Best in Slot, OKX, and UniSat have
  occasionally produced different balances on edge cases. Production
  systems should pin to a specific indexer and version.

## References

- BRC-20 spec (informal): https://layer1.gitbook.io/layer1-foundation/protocols/brc-20
- OPI reference indexer: https://github.com/bestinslot-xyz/OPI
- ord inscriptions: https://docs.ordinals.com/inscriptions.html
