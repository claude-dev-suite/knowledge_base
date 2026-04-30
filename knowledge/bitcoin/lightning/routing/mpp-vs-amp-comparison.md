# MPP vs AMP - Detailed Comparison

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/routing`.
> Canonical source: BOLT 4 (`payment_secret`, `total_msat`), AMP spec proposal
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/routing/SKILL.md

## Concept

Both MPP (Multi-Path Payment) and AMP (Atomic Multi-Path) split a
payment across multiple routes to overcome single-channel liquidity
limits. They differ fundamentally in the cryptographic atomicity
mechanism: MPP uses one shared `payment_hash` across all parts (parts
are atomic via the receiver waiting for full amount before fulfilling);
AMP uses different per-part hashes derived from a shared seed
(parts are atomic via cryptographic key reconstruction). The
distinction determines what the receiver can do with partial payments
and how privacy works.

## Walkthrough / mechanics

### MPP (BOLT 4 `basic_mpp`, bits 16/17)

Sender splits payment into N parts, each with the same
`payment_hash = H` and the same `payment_secret = S` (BOLT 11 `s`
tag). Each part's onion-final-hop TLV includes:

```
TLV 8 (payment_data):
  payment_secret: S          (32 bytes)
  total_msat:     T          (full payment amount)
```

Part's `update_add_htlc.amount_msat` is its share `t_i`, where
`sum(t_i) = T`. The receiver:

1. On first incoming HTLC, records expected total `T`.
2. Waits up to ~60s (mppTimeout) for `sum(received) >= T`.
3. When reached: fulfill ALL parts with the same preimage `r` such
   that `SHA256(r) = H`.

Atomicity: receiver MUST NOT fulfill any part until all parts arrive,
otherwise stranded parts are lost. If timeout fires without total,
fail all parts.

### AMP (proposal, LND-only deployment)

Sender derives per-part keys:

```
root_share, root_secret = random(32), random(32)
For each part i:
  child_index_i = i
  share_i = HMAC-SHA256(root_share, "amp" || i)
  secret_i = HMAC-SHA256(root_secret, "amp" || i)
  payment_hash_i = SHA256(secret_i)
  amp_payload TLV:
    root_share: root_share
    set_id:     unique_id
    child_index: i
```

Receiver, on receiving N parts with the same `set_id`:
1. Checks `total_msat` matches sum.
2. Each part has its own `payment_hash_i` and the receiver derives
   `secret_i` from `root_share` + `child_index_i`.
3. Reveals each `secret_i` to claim each part — N different preimages.

Atomicity now: the *root_share* alone is useless. Without all
`child_index`-revealed shares (combined via SSS-like reconstruction in
the AMP TLV), the sender cannot prove receipt for each part. Sender
verifies completeness off-chain.

### Key differences

| Aspect              | MPP                              | AMP                             |
|---------------------|----------------------------------|---------------------------------|
| `payment_hash`      | one for all parts                | unique per part                 |
| `payment_secret`    | shared `s` from BOLT11 invoice   | derived per part                |
| Spec status         | BOLT-merged                      | proposal, LND only              |
| Spontaneous?        | no (needs invoice)               | yes (sender-derived secrets)    |
| Privacy from analyst| same hash leaks correlation      | hashes look independent         |
| Repeated reuse      | invoice single-use               | `set_id` makes payments unique  |
| Receiver wallet     | works with any BOLT 11 invoice   | requires AMP-aware receiver     |

## Worked example

### MPP

Sender pays 500k sats, splits 200/150/150:

```
Part 1: amount=200_000_000 msat, hash=H, secret=S, total=500_000_000
        route: A -> B -> Receiver
Part 2: amount=150_000_000 msat, hash=H, secret=S, total=500_000_000
        route: A -> C -> Receiver
Part 3: amount=150_000_000 msat, hash=H, secret=S, total=500_000_000
        route: A -> D -> E -> Receiver

Receiver waits, verifies sum = 500M, fulfills all with preimage R.
```

LND command:
```
lncli payinvoice --max_parts 16 --max_shard_size_msat 200000000 \
    lnbc500u1...
```

CLN automatically MPPs via `xpay`:
```
lightning-cli xpay bolt11=lnbc500u1...
```

### AMP

LND keysend AMP:
```
lncli sendpayment --dest 03receiver_pubkey \
    --amt 500000 \
    --keysend \
    --amp
```

The receiver must run LND with AMP enabled (`--amp.enabled=true`) and
generate an `addinvoice --amp` invoice (or accept spontaneous). Each
part has a unique `payment_hash`.

A blockchain analyst seeing two force-closes on different channels
during MPP can correlate them by `H`. With AMP, the on-chain hashes
are different and unlinkable without the root_share.

## Common bugs / pitfalls

- **`payment_secret` mismatch in MPP**: every part must carry the same
  secret matching the invoice's `s` tag. A typo on one part fails
  that part with `incorrect_or_unknown_payment_details`.
- **Total-msat drift**: sender includes total = invoice amount; some
  pathfinders accidentally set total = part amount, causing receiver
  to think the full payment arrived after one part.
- **Timeout too short**: 60s is the default; congested networks need
  longer. Underestimating drops in-flight parts.
- **AMP receiver leak**: the receiver storing `set_id` long-term still
  allows sender-receiver correlation for that user. Use ephemeral
  set_ids.
- **No AMP without LND on both sides**: cross-impl AMP fails because
  CLN/LDK/Eclair don't decode AMP TLVs. Verify peer.
- **Splitting too aggressively**: 16+ parts has diminishing returns;
  each part adds CLTV-delta penalty and risk of a part stuck.
- **MPP through nodes that don't support `basic_mpp`**: their
  `channel_update` doesn't disable them, but they MAY reject the
  partial-amount HTLC. Pathfinder must filter by feature bit (BOLT 11
  routing hints carry features).

## References

- BOLT 4 `payment_data` TLV: https://github.com/lightning/bolts/blob/master/04-onion-routing.md#payload-format
- BOLT 9 bits 14/15, 16/17: https://github.com/lightning/bolts/blob/master/09-features.md
- AMP spec draft (Olaoluwa Osuntokun): https://lists.linuxfoundation.org/pipermail/lightning-dev/2018-February/000993.html
- LND AMP docs: https://docs.lightning.engineering/lightning-network-tools/lnd/amp
- CLN MPP design: `plugins/pay.c`
