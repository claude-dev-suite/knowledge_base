# Trampoline Onion Structure Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/trampoline`.
> Canonical source: https://github.com/lightning/bolts/blob/master/proposals/trampoline.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/trampoline/SKILL.md

## Concept

Trampoline routing solves a UX problem: mobile Lightning wallets cannot
keep an up-to-date view of the entire 75 000-channel public graph. With
trampoline, the wallet only knows a small set of well-connected
**trampoline nodes** (e.g. ACINQ's, c=phoenix's). The wallet onion-routes
to a trampoline; the trampoline computes the full payment route and
forwards. The recipient still receives a normal HTLC.

The structure is an **onion within an onion**: the outer onion (BOLT-04)
hops between channels, while the inner trampoline onion specifies a
sequence of trampoline nodes the payment must traverse.

## Walkthrough / mechanics

### Two-layer onion

```
Outer onion (per-hop, as in BOLT-04):
  Hop 1 -> Hop 2 -> ... -> Trampoline T1
    payload at T1: trampoline_onion (encrypted to T1 + next trampoline keys)

Trampoline onion:
  T1 -> T2 -> ... -> Tn
    Tn payload includes payment_hash, final_amount, MPP info, etc.
```

A trampoline node, on receiving an HTLC with a trampoline payload:

1. Decrypts trampoline onion outer layer; learns next trampoline
   node + amount + CLTV.
2. Builds a **new outer onion** to reach the next trampoline (using its
   local routing graph).
3. Adds the trampoline-onion payload (with this hop peeled off) to the
   new outer onion's last hop.
4. Forwards the resulting HTLC.

### Payload format

Trampoline onion payload (TLV):

| TLV type | Content |
|----------|---------|
| 4 | next_node_id (33 bytes) — next trampoline pubkey |
| 14 | total_msat (vUint) — for MPP |
| 33 | payment_metadata |
| (others) | invoice features, route hints, etc. |

### Recipient discovery

The final trampoline (Tn) needs to be one that knows the recipient's
node. Either:

- Recipient's node IS Tn (most common — receiver runs trampoline
  software).
- Tn finds recipient via gossip and constructs a final outer onion to
  recipient.

If recipient is private, they include a route hint in their invoice
indicating which trampoline can reach them.

### Fees and CLTV

Each trampoline charges:

- A **trampoline fee** (per-hop, advertised in their `node_announcement`).
- Plus the underlying outer-onion fees they pay to channel peers.

Sender includes total trampoline fees + buffer. Trampoline keeps the
difference between the sender's prepaid trampoline fee and the actual
outer routing fee they paid.

CLTV is similarly cumulative: each trampoline subtracts its CLTV expiry
delta + the underlying CLTV deltas.

### Phoenix wallet pattern

The most-deployed trampoline implementation is Phoenix (ACINQ). Phoenix
wallet only connects to ACINQ's node and trampolines to ACINQ's routing
graph. The wallet's BOLT11 invoices include ACINQ as a route hint.

## Worked example

Alice (mobile wallet) pays Bob via trampoline.

```
Alice -> ACINQ_T1 (trampoline)
ACINQ_T1 -> ACINQ_T2 -> ... outer routing to T2
T2 -> Bob (or directly delivers if T2 knows Bob)

Outer onion that Alice constructs:
  Hop 1 (Alice's only channel peer = ACINQ): payload = trampoline_onion

Trampoline onion that Alice constructs:
  Hop 1 (T1=ACINQ_T1): {next=T2, amount=10000, cltv=200}
  Hop 2 (T2): {amount_to_forward=10000, payment_hash=H, total_msat=10000, ...}
```

T1 receives HTLC. Decrypts trampoline_onion -> "next is T2, send 10 000
msat". T1 builds outer onion via its graph: T1 -> NodeA -> NodeB -> T2.
Forwards.

T2 decrypts -> "I am final, payment_hash=H, total=10 000". T2 either is
Bob or constructs an outer onion to Bob. Bob settles HTLC; preimage
propagates back.

## Common pitfalls

- **Trampoline DoS**: trampolines pay fees out of pocket if they fail
  to recover from sender. Trampoline pre-collects funds via locked
  HTLC; DoS-resistant designs require sender to pre-prepay fee buffer.
- **Trampoline fee griefing**: dishonest trampoline could overspend on
  routing and absorb the loss; mitigated by capping outer routing fee
  proportionally.
- **MPP across trampolines**: complex; must coordinate `total_msat`
  across paths and ensure final trampoline reassembles correctly.
- **Privacy**: trampolines see source (sender's first hop = trampoline)
  and destination (final trampoline knows recipient). Less private than
  full source-routed onion.
- **CLTV inflation**: each trampoline adds CLTV expiry; payments with
  many trampolines may exceed receiver's allowed CLTV.

## References

- BOLT proposal "Trampoline Onion Routing".
- ACINQ Phoenix architecture docs.
- BOLT-04 Onion Routing (base format).
