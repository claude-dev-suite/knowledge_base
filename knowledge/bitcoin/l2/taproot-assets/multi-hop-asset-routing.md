# Multi-Hop Asset Routing on Taproot Assets - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/taproot-assets`.
> Canonical source: https://lightning.engineering/posts/2024-05-01-taproot-assets-v0.4
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/taproot-assets/SKILL.md

## Concept

Taproot Assets v0.4+ ships **multi-hop asset routing** over Lightning. An invoice priced
in an asset (e.g., 100 USDT-on-TAP) can be paid by a sender holding a different asset
(or just sats), with edge nodes performing asset/sat conversions. This unlocks
"Bitcoin-routed stablecoin payments" where end users hold assets but the network's
liquidity backbone remains Bitcoin / sats. Lightning Labs calls this approach
"Decentralized FX": no operator or pool, just routing-node fees.

## Walkthrough / mechanics

### Channel structure with assets

A TAP-aware channel between nodes A and B can hold:

```
Channel commitment outputs:
   to_local    (A's balance: X sats + Y_usdt USDT)
   to_remote   (B's balance: X' sats + Y'_usdt USDT)
   to_outputs HTLC anchors

Each per-asset balance is committed via MS-SMT in the channel anchor's
Taproot output script.
```

When A pushes 5 USDT to B, the commitment update changes both the sat balance (to pay
the LN HTLC fee in sats) and the asset balance (in USDT-on-TAP).

### Routing through a non-asset-aware hop

The killer property: an edge node holds USDT-on-TAP, but a middle hop is a vanilla LN
node holding only sats. The first/last hop convert; middle hop never sees the asset.

```
Sender (USDT only)        Edge node 1            Hop X (sats only)        Edge node 2          Receiver
       |                       |                        |                       |                  |
       |--- HTLC: 100 USDT --->|                        |                       |                  |
       |                       |--- HTLC: 100k sat ---> |--- HTLC: 100k sat --->|                  |
       |                       |  (converted at edge1   |  (vanilla LN routing) |--- HTLC:        --|
       |                       |   at FX rate)          |                       |       100 USDT ->|
       |                       |                        |                       |                  |
       |  preimage settles back through:    edge2 -> X -> edge1 -> sender                          |
```

### Conversion at edges

Edge nodes hold inventory in both sats and assets and quote a conversion rate on demand.
The TAP RFQ ("request for quote") protocol negotiates the FX rate before the HTLC is
sent:

```
Sender -> Edge1: "I want to pay 100 USDT. What rate?"
Edge1 -> Sender: "100 USDT for 102,500 sat (24 hour expiry)."
Sender:           accepts.  Pays 100 USDT to Edge1, who routes 102,500 sat onward.
```

The rate quote is signed by edge1 and pinned to a payment_hash; if the HTLC fails the
quote can be refreshed.

### v0.6 / v0.7 features

- **v0.6 (Jun 2025)**: stable multi-hop routing for fungible assets. RFQ stable.
- **v0.7 (Dec 2025)**: AddressV2 (reusable static asset addresses with grouped assets);
  zero-amount-friendly invoices for refunds and probing.

## Worked example

Alice (US, holds USDT-on-TAP) pays Carlos (Argentina, wants USDT-on-TAP) 50 USDT through
a Lightning network where the middle hop is a Strike-style sat-only node:

```
Setup:
  Alice's wallet has USDT-on-TAP channel to Voltage (edge1).
  Voltage has sats channels to LN graph.
  Lemon (edge2 in Argentina) has sats channels to LN graph + USDT-on-TAP channel to Carlos.

Flow:
  T+0    Alice scans Carlos's invoice: lnbc... carrying TAP asset_id=USDT, amount=50.
  T+10   Alice's wallet asks Voltage for RFQ: "50 USDT outbound".
  T+50   Voltage replies: "50 USDT for 51,200 sat outbound, expires in 1h".
  T+60   Alice's wallet asks Lemon (via probe) for inbound RFQ: "I'll send 51,000 sats,
         what USDT will arrive?"
  T+90   Lemon replies: "50 USDT inbound for 51,000 sats".
  T+100  Alice fails to balance the rates -> negotiate again or pick alternative path.
  T+200  Eventually Alice sends 51,200 sat HTLC to Voltage payable to invoice's H.
  T+250  Voltage forwards via 2 hops of vanilla LN (sats only): 51,180 sat -> 51,150 sat.
  T+400  Lemon receives 51,150 sat HTLC. Generates 50 USDT HTLC to Carlos for same H.
  T+450  Carlos's wallet settles, reveals preimage R.
  T+500  Preimage propagates back; Voltage debits Alice's USDT balance by 50, gains
         the 50 - some_fee USDT economic value implicitly via the sat differential.

Net: Alice -50 USDT.  Carlos +50 USDT.  Routing nodes earn sats fees.  Edges arbitrage
the FX spread.
```

## Trade-offs and security

- **Edge-node centralisation**: practical asset routing depends on a small number of
  high-liquidity edge nodes (Voltage, Lemon, Olympus). Censorship resistance is weaker
  than vanilla LN.
- **Rate slippage**: quotes are bilateral; sender must compose rates across edges.
  Failed payments waste probing traffic.
- **Inventory imbalance**: edge nodes need both sats and asset liquidity. Persistent
  one-way flow drains the asset side; rebalancing is via on-chain TAP transfers
  (expensive) or peer trades.
- **Privacy**: middle hops see only sat HTLCs, but edges see asset/amount. The asset_id
  itself is metadata-revealing.
- **MEV at edges**: an edge could front-run by adjusting rates against high-volume
  senders. Reputation and quote-pinning (via signed RFQ) mitigate.
- **Compared to RGB on Lightning**: TAP's RFQ + sat-routing is more mature; RGB's
  rgb-lightning-node achieves similar but is in earlier production beta.

## References

- Lightning Labs v0.4 announcement - https://lightning.engineering/posts/2024-05-01-taproot-assets-v0.4
- v0.6 / v0.7 release notes - https://github.com/lightninglabs/taproot-assets/releases
- TAP RFQ spec - https://github.com/lightninglabs/taproot-assets/blob/main/docs/RFQ.md
- BOLT11 / BOLT12 - https://github.com/lightning/bolts
