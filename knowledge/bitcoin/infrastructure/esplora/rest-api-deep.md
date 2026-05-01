# Esplora REST API - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/esplora`.
> Canonical source: https://github.com/Blockstream/esplora/blob/master/API.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/esplora/SKILL.md

## Concept

Esplora exposes a stable, well-documented HTTP REST API that has become a de
facto standard for non-Electrum block exploration. Wallets like LDK Node,
Mutiny, BDK, and Sparrow's HTTP backend all speak Esplora. This article walks
the most-used endpoints, their JSON shapes, pagination semantics, and shows a
worked transaction-broadcast + monitoring flow against the canonical
`blockstream.info/api` instance and a self-hosted equivalent.

## Walkthrough / mechanics

Base URL conventions:

- Mainnet: `https://blockstream.info/api/...`
- Testnet: `https://blockstream.info/testnet/api/...`
- Self-hosted: `http://node.example.com:3000/...` (no `/api` prefix when
  hitting the electrs-fork directly).

Endpoint families (read):

```
GET /block-height/<height>     -> hash (text/plain)
GET /block/<hash>              -> block header + summary
GET /block/<hash>/txs/<index>  -> 25 txs per page
GET /tx/<txid>                 -> tx with vin/vout, status
GET /tx/<txid>/hex             -> raw hex
GET /tx/<txid>/merkle-proof    -> merkle path JSON
GET /address/<addr>            -> chain + mempool stats
GET /address/<addr>/txs        -> last 25 confirmed + all mempool
GET /address/<addr>/txs/chain/<last_seen_txid>  -> paginate older
GET /address/<addr>/utxo       -> UTXO array
GET /scripthash/<hash>/...     -> same shape, by Electrum scripthash
GET /mempool                   -> count, fee histogram, total vsize
GET /fee-estimates             -> {target_blocks: sat/vB}
```

Endpoint families (write):

```
POST /tx                       -> body: raw hex; returns txid
```

Pagination uses last-seen txid, not offsets. To page deeper into an
address's history, take the last entry's `txid` and append it to
`/address/<addr>/txs/chain/<last_seen_txid>`.

Confirmation status object inside `tx`:

```json
{
  "confirmed": true,
  "block_height": 868234,
  "block_hash": "0000...",
  "block_time": 1714326123
}
```

Unconfirmed txs return `{"confirmed": false}`. Use this rather than parsing
mempool endpoints for per-tx lookup.

## Worked example

End-to-end flow: build, broadcast, monitor a tx.

Get a fee estimate, broadcast a pre-signed tx, then poll until confirmed:

```bash
BASE=https://blockstream.info/api

# 1. Fee estimate (sat/vB per block target)
curl -s $BASE/fee-estimates | jq '."6", ."144"'
# 12.34
# 4.567

# 2. Broadcast
RAW=$(cat signed-tx.hex)
TXID=$(curl -s -X POST -d "$RAW" $BASE/tx)
echo "Broadcast: $TXID"

# 3. Watch status
while true; do
  STATUS=$(curl -s $BASE/tx/$TXID/status)
  echo "$STATUS"
  CONFIRMED=$(echo "$STATUS" | jq -r .confirmed)
  [ "$CONFIRMED" = "true" ] && break
  sleep 30
done
```

UTXO listing for a wallet's first receive address:

```bash
ADDR=bc1qar0srrr7xfkvy5l643lydnw9re59gtzzwf5mdq
curl -s $BASE/address/$ADDR/utxo | jq '.[0]'
# {
#   "txid": "...",
#   "vout": 0,
#   "value": 50000,
#   "status": { "confirmed": true, "block_height": 700000, ... }
# }
```

| Endpoint | Use case | Avg payload |
|----------|----------|-------------|
| /fee-estimates | RBF/fee bumping | <1 KB |
| /address/{a}/utxo | wallet refresh | 1-50 KB |
| /address/{a}/txs | history page | 5-30 KB |
| /tx/{id} | inspector view | 1-5 KB |
| /mempool | dashboard | <1 KB |
| POST /tx | broadcast | small |

## Common pitfalls

- Treating the public `blockstream.info/api` as guaranteed - rate-limited,
  ~600 req/min per IP; self-host or use a paid tier for heavy use.
- Paginating with offsets - the API has no `offset` parameter; you must use
  the `chain/<last_seen_txid>` form.
- Broadcast errors swallowed - `POST /tx` returns 400 with a plain-text
  reason ("min relay fee not met"); read the body, not just the status code.
- `/fee-estimates` returns sat/vB as a float, not sat/kvB - some libraries
  multiply incorrectly.
- Confused with mempool.space endpoints - they overlap but `/api/v1/...`
  paths are mempool.space-specific (not Esplora).
- Self-hosted instance behind nginx - forgetting to forward
  `Content-Type: text/plain` for the broadcast endpoint causes 415s.

## References

- Esplora API spec: https://github.com/Blockstream/esplora/blob/master/API.md
- esplora-electrs source: https://github.com/Blockstream/electrs
- BDK Esplora client: https://github.com/bitcoindevkit/bdk/tree/master/crates/esplora
- LDK Node Esplora chain source: https://docs.rs/ldk-node
