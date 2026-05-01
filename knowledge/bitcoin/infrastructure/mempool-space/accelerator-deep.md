# mempool.space Accelerator - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/mempool-space`.
> Canonical source: https://mempool.space/docs/accelerator
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/mempool-space/SKILL.md

## Concept

The mempool.space Accelerator is a paid service that bumps a stuck or
under-priced transaction into the next 1-3 blocks. Unlike RBF or CPFP -
which require the user to control inputs/outputs and rebroadcast - the
accelerator works on any transaction by leveraging mempool.space's mining
pool partnerships: the operator pays a participating pool to include the
target tx out-of-mempool, regardless of whether competitor txs would
otherwise outbid it. Payment is in Lightning sats and pricing depends on
target depth and tx vsize.

## Walkthrough / mechanics

Accelerator API endpoints:

```
GET  /api/v1/services/accelerator/estimate?txInput=<txid>
POST /api/v1/services/accelerator/accelerate
GET  /api/v1/services/accelerator/history
```

Estimate first to see prices for each target depth:

```bash
TXID=4a5b6c...
curl -s https://mempool.space/api/v1/services/accelerator/estimate?txInput=$TXID
```

Sample response shape:

```json
{
  "txSummary": { "txid": "...", "effectiveVsize": 246,
                 "effectiveFee": 1230, "ancestorCount": 0 },
  "nextBlockFee": 18,
  "targetFeeRate": 22,
  "userBalance": 0,
  "options": [
    { "fee":  4500, "targetFeeRate": 22, "targetFeeDelta": 4 },
    { "fee":  9200, "targetFeeRate": 28, "targetFeeDelta": 10 },
    { "fee": 15800, "targetFeeRate": 35, "targetFeeDelta": 17 }
  ],
  "available": true,
  "hasAccess": true
}
```

`fee` is the price the user pays in sats; `targetFeeRate` is the resulting
effective fee rate of the package after acceleration. If `available: false`
the partnered pools are at capacity or the tx is ineligible (already
confirmed, double-spend conflict, etc.).

Place an order. The flow is two-step: POST returns a Lightning invoice,
client pays, server completes the acceleration:

```bash
curl -s -X POST https://mempool.space/api/v1/services/accelerator/accelerate \
  -H "Content-Type: application/json" \
  -d '{"txInput":"'"$TXID"'","userBid":9200}'
```

Response:

```json
{
  "lightningInvoice": "lnbc92u1p...",
  "expires": 1714329000,
  "orderId": "ord_abc123"
}
```

Pay the invoice from any LN wallet. Once preimage is observed, the
operator forwards the tx to a participating pool (Foundry, F2Pool, Antpool
in past disclosed partnerships). Targeting "next block" works only when a
partnered pool wins; mempool.space's UI shows expected probability based
on 24h pool share.

Eligibility constraints:

- Tx must already be in the public mempool.
- No conflicting RBF replacement currently being relayed.
- Total package vsize <= ~25 vKB.
- Not part of an already-accelerated package.

## Worked example

Stuck commerce payment - a user paid a BTCPay invoice at 4 sat/vB during a
fee spike; the merchant wants confirmation within 30 minutes:

```bash
TXID=$(btcpay-cli get-invoice INV-001 | jq -r .txid)
mempool=https://mempool.space/api/v1/services

# 1. estimate
EST=$(curl -s "$mempool/accelerator/estimate?txInput=$TXID")
echo "$EST" | jq '.options'

# pick the option for next-block (highest targetFeeRate)
BID=$(echo "$EST" | jq '.options[-1].fee')

# 2. order
RESP=$(curl -s -X POST $mempool/accelerator/accelerate \
       -H 'Content-Type: application/json' \
       -d "{\"txInput\":\"$TXID\",\"userBid\":$BID}")
INVOICE=$(echo "$RESP" | jq -r .lightningInvoice)

# 3. pay via LND
lncli payinvoice --force "$INVOICE"

# 4. monitor
while true; do
  STATUS=$(curl -s https://mempool.space/api/tx/$TXID/status)
  CONF=$(echo "$STATUS" | jq -r .confirmed)
  [ "$CONF" = "true" ] && echo "Confirmed at $(echo $STATUS | jq .block_height)" && break
  sleep 30
done
```

| Target depth | Typical cost (300 vB tx, 20 sat/vB market) | When to use |
|--------------|--------------------------------------------|-------------|
| Next block | $5-50 | Time-critical commerce, exchange deadline |
| 1-2 blocks | $2-15 | Within ~30 min |
| 1-3 blocks | $1-8 | Within ~45 min |
| Cheapest available | $0.50-3 | Just want it unstuck |

## Common pitfalls

- Treating Accelerator as RBF - it is *not* a replacement; the original
  txid stays the same, no signature changes, no input churn. This is its
  main advantage when the wallet is non-RBF (older Coldcard signing path,
  for example).
- Paying for low-fee txs with no descendants - if your tx has no ancestor/
  descendant burden, often a regular CPFP child from a wallet you control
  is cheaper than Accelerator.
- Partnered-pool dependency - the service can only deliver to "next block"
  with the probability that a partnered pool mines it. With ~50% combined
  share that is ~50% per attempt. Re-bid for higher targets if needed.
- Lightning invoice expiry - default expiry is short (5-15 min); if the
  payment fails or is delayed, place a new order rather than reusing the
  invoice.
- Tx already confirmed between estimate and order - the API returns an
  error with reason; handle gracefully and stop polling.

## References

- Accelerator product page: https://mempool.space/accelerator
- Accelerator API: https://mempool.space/docs/accelerator
- Mempool Enterprise: https://mempool.space/enterprise
