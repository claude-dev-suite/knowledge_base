# HTLC Lifecycle - Deep Walkthrough

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/htlcs`.
> Canonical source: BOLT 2 sections "Adding an HTLC" through "Committing Updates"
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/htlcs/SKILL.md

## Concept

The skill summarizes HTLC states. This article walks through one HTLC's
full life on the wire, with the exact sequence of `commitment_signed` /
`revoke_and_ack` round trips that shift it between irrevocable states.
The state model has SEVEN states per HTLC, not the four most overviews
list, and the irrevocability rules are subtle enough that getting them
wrong leads to channel desync.

## Walkthrough / mechanics

BOLT 2 defines the per-HTLC state machine for the *offerer's* side:

```
LOCAL_PROPOSED       -> peer sees update_add_htlc, hasn't acked
LOCAL_SIGNED         -> we sent commitment_signed including this HTLC
LOCAL_COMMITTED      -> peer revoked previous, our HTLC irrevocably in their next commitment
COMMITTED            -> both sides have HTLC in their irrevocable state
REMOTE_REMOVED       -> peer sent fulfill/fail, not yet committed
REMOTE_SIGNED_REMOVED-> peer signed commit removing the HTLC
DEAD                 -> we revoked, fully gone
```

The minimum to *add* an HTLC and have it irrevocable on both sides:

```
A -> B: update_add_htlc          (HTLC enters A's pending-out)
A -> B: commitment_signed         (B's new commitment includes the HTLC)
B -> A: revoke_and_ack            (B revokes previous, HTLC is on B's
                                   side irrevocably)
B -> A: commitment_signed         (A's new commitment includes the HTLC)
A -> B: revoke_and_ack            (A revokes previous, HTLC fully
                                   committed both sides)
```

That's TWO full round trips, not one. The first half-trip locks the
HTLC on the receiver's side; the second locks it on the sender's. Only
after BOTH revokes does the HTLC enter `COMMITTED`.

To remove (fulfill or fail):

```
B -> A: update_fulfill_htlc(preimage) or update_fail_htlc
B -> A: commitment_signed
A -> B: revoke_and_ack
A -> B: commitment_signed
B -> A: revoke_and_ack
```

The two-round-trip rule applies symmetrically to removal. This is why
busy channels batch many adds + removes into a single `commitment_signed`
exchange.

`update_fee` follows the same FSM: it must be irrevocably committed
before the new fee applies to the next commitment.

## Worked example

Routing a 100k-sat payment through Bob (forwarding hop):

```
T0   Alice -> Bob: update_add_htlc(id=42, amt=100_500, hash=H, cltv=752000, onion)
                                  (100 sat fee for Bob)
T1   Alice -> Bob: commitment_signed
                  (Bob's commitment N+1 = current + new HTLC out from Alice)
T2   Bob -> Alice: revoke_and_ack
                  (Bob releases revocation secret for commitment N-1)
T3   Bob -> Alice: commitment_signed
                  (Alice's commitment includes Alice's HTLC out)
T4   Alice -> Bob: revoke_and_ack

# Bob now forwards downstream to Carol over a different channel.
# Carol fulfills, Bob receives preimage.

T5   Bob -> Alice: update_fulfill_htlc(id=42, preimage=R)
T6   Bob -> Alice: commitment_signed
T7   Alice -> Bob: revoke_and_ack
T8   Alice -> Bob: commitment_signed
T9   Bob -> Alice: revoke_and_ack
```

Five distinct messages each direction over ~20-50ms WAN. Real-world
forwarding nodes batch — if 8 HTLCs land within a 50ms window, Bob
sends one `commitment_signed` covering all 8 adds.

LDK's `ChannelMonitor` tracks per-HTLC state as `InboundHTLCState`
and `OutboundHTLCState` enums in `lightning/src/ln/channel.rs`. Bugs
here historically caused payment stuck for hours.

LND `chanbackup` records the latest revoked commitment per HTLC
direction. Recovery from this requires reconstructing the FSM
position.

## Common bugs / pitfalls

- **Forgetting the second round trip**: implementations that treat one
  `commitment_signed` as final lose payments on reconnection because
  the peer's view differs.
- **Assuming `update_add_htlc` is committed**: it is *proposed*, not
  committed. Failure modes between propose and commit are recoverable
  (re-send) — failures after commit require force-close.
- **Reordering removes vs adds**: BOLT 2 says `commitment_signed` covers
  the cumulative state. If you send `update_add_htlc` then
  `update_fulfill_htlc` then `commitment_signed`, the new commitment
  includes neither (add+fulfill cancel). Subtle bug if your code
  applies adds and fulfills in different orders.
- **Race on disconnect**: the first half-round-trip locks on receiver
  only. Disconnecting after `commitment_signed` but before
  `revoke_and_ack` requires `channel_reestablish` to retransmit the
  ack. Many "stuck HTLC for 20 minutes" reports trace to this.
- **Dust HTLCs in commitment but not in commitment-sigs**: HTLCs below
  trim threshold are excluded from the commitment outputs but STILL
  counted toward `max_accepted_htlcs`. Off-by-one bug if you forget.
- **Replay across reconnect**: `update_add_htlc` IDs are monotonic
  per direction. If reconnect causes you to re-issue id 42 with
  different parameters, peer must reject as duplicate. LND 0.13.x had
  a regression here — fixed in 0.13.4.
- **Fee-update on small channel**: `update_fee` raising the rate can
  push channel below reserve. Receiver must reject; some impls signed
  anyway, leading to `must-fail-channel`.

## References

- BOLT 2 - Updating Fees / Committing Updates: https://github.com/lightning/bolts/blob/master/02-peer-protocol.md#normal-operation
- LDK channel state machine: `lightning/src/ln/channel.rs` (`HTLCState`)
- CLN `channeld/full_channel.c`: HTLC FSM
- LND `htlcswitch/link.go`: forwarding state machine
