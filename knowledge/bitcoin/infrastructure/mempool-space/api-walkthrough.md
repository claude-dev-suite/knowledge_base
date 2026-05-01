# mempool.space REST API Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/mempool-space`.
> Canonical source: https://mempool.space/docs/api/rest
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/mempool-space/SKILL.md

## Concept

mempool.space's REST API extends the Esplora-compatible base with rich
mempool projection, mining stats, fee buckets, and Lightning network data.
This article focuses on the parts that have no Esplora analogue: the
`/api/v1/fees/recommended` and `/api/v1/fees/mempool-blocks` endpoints, the
per-tx CPFP / RBF info, mining pool statistics, and Lightning channel
queries. Examples use `https://mempool.space` but apply equally to a
self-hosted instance at `https://mempool.local`.

## Walkthrough / mechanics

Endpoint groups:

```
# Fees
GET /api/v1/fees/recommended
GET /api/v1/fees/mempool-blocks
GET /api/v1/historical-price?currency=USD&timestamp=...

# Mempool
GET /api/mempool
GET /api/v1/mempool/recent
GET /api/v1/mempool/statistics?interval=24h

# Tx (Esplora-compatible plus extensions)
GET /api/tx/<txid>
GET /api/tx/<txid>/cpfp
GET /api/v1/tx/<txid>/rbf
GET /api/v1/transaction-times?txId[]=...

# Mining
GET /api/v1/mining/pools/24h
GET /api/v1/mining/pool/<slug>/blocks
GET /api/v1/mining/hashrate/3y

# Lightning
GET /api/v1/lightning/statistics/latest
GET /api/v1/lightning/channels/txids?public_key=<pubkey>
GET /api/v1/lightning/nodes/<pubkey>

# Acceleration
GET /api/v1/services/accelerator/estimate?txInput=<txid>
POST /api/v1/services/accelerator/accelerate
```

Key shapes:

`/api/v1/fees/recommended`:
```json
{ "fastestFee": 18, "halfHourFee": 14, "hourFee": 10,
  "economyFee": 6, "minimumFee": 1 }
```

`/api/v1/fees/mempool-blocks`: array of upcoming projected blocks, each
with `medianFee`, `feeRange[]`, `nTx`, `totalFees`, `blockSize`. Index 0
is the next block.

`/api/v1/tx/<txid>/rbf`: returns the full RBF replacement tree (parent +
descendants) so a wallet can detect whether a competing tx replaced this
one and at what fee delta.

Authentication: read endpoints are unauthenticated; the Accelerator
`POST /accelerate` requires a Lightning payment proof.

## Worked example

Build an RBF dashboard for a single tx:

```bash
TXID=4a5b6c...
BASE=https://mempool.space

# 1. Status
curl -s $BASE/api/tx/$TXID/status | jq
# { "confirmed": false, "block_height": null, ... }

# 2. CPFP package fee rate
curl -s $BASE/api/tx/$TXID/cpfp | jq
# {
#   "ancestors": [...],
#   "descendants": [{ "txid": "...", "weight": 760, "fee": 12000 }],
#   "effectiveFeePerVsize": 18.4
# }

# 3. RBF history
curl -s $BASE/api/v1/tx/$TXID/rbf | jq '.replacements[0]'
# {
#   "tx": { "txid": "newer-txid", "fee": 14500, ... },
#   "time": 1714326100,
#   "fullRbf": false,
#   "interval": 60
# }

# 4. Compare to current next-block target
NEXT=$(curl -s $BASE/api/v1/fees/recommended | jq -r .fastestFee)
echo "Next block expects $NEXT sat/vB"
```

Mining-pool 24h share:

```bash
curl -s $BASE/api/v1/mining/pools/24h \
  | jq '.pools[] | {name, blockCount, share}' | head -30
# { "name": "Foundry USA", "blockCount": 51, "share": 35.4 }
# { "name": "AntPool", "blockCount": 32, "share": 22.2 }
```

| Metric | Endpoint | Update cadence |
|--------|----------|----------------|
| Recommended fees | /api/v1/fees/recommended | ~30 s |
| Mempool blocks | /api/v1/fees/mempool-blocks | ~30 s |
| Tx status | /api/tx/{id}/status | per-block |
| Pool stats 24h | /api/v1/mining/pools/24h | per-block |
| LN stats | /api/v1/lightning/statistics/latest | hourly |

## Common pitfalls

- Hitting public mempool.space at high request rates - the WAF rate-limits
  by IP and bans aggressive scrapers; for production polling, self-host or
  use the paid Mempool Enterprise tier.
- Treating `/api/v1/fees/recommended` as gospel - it is a smoothed fee
  estimate; for low-priority sends compute your own from
  `/api/v1/fees/mempool-blocks` index 5-9.
- Misreading RBF responses - `replacements[0]` is the *first* replacement;
  the chain may continue (`replacements[].replacements`).
- Using mempool.space endpoints assuming Esplora compatibility - the
  Esplora subset (`/api/tx/{id}`, `/api/address/{a}`) is compatible, but
  `/api/v1/...` paths are mempool-specific.
- CORS - public API has wildcard CORS; self-hosted defaults to none.

## References

- mempool.space REST docs: https://mempool.space/docs/api/rest
- mempool.space WebSocket docs: https://mempool.space/docs/api/websocket
- Source: https://github.com/mempool/mempool
- mempool-js client: https://github.com/mempool/mempool.js
