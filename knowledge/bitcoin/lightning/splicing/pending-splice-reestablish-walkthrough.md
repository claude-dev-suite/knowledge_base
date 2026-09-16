# Splicing: Pending Splices and Reestablish - Walkthrough

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/splicing`.
> Canonical source: https://github.com/lightning/bolts/blob/master/02-peer-protocol.md (Channel Splicing)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/splicing/SKILL.md

## Concept

Splicing left draft status when PR #1160 ("Channel Splicing (feature
62/63)") was merged into `lightning/bolts` on 2026-03-23.
`option_splice` sits at bits **62/63** in `09-features.md` with no
declared dependencies, and BOLT 2 gained a "Channel Splicing" section.

The part that surprises implementers is what happens *after* the splice
is signed. Splicing is not a stop-the-world operation: the channel is
quiescent only for the negotiation. BOLT 2 is explicit that "operation
returns to normal once the splice transaction has been signed (while
waiting for one of the splice transactions to confirm), at which point
the channel isn't quiescent anymore".

So between `tx_signatures` and `splice_locked` the channel is routing
payments while **multiple funding transactions are simultaneously
viable**. That window is where the interesting state lives.

## Walkthrough / mechanics

### Multiple commitments at once

While a splice is pending, a node keeps one commitment transaction per
candidate funding transaction and signs all of them. BOLT 2's diagram:

```
+------------+        +-----------+
| Funding Tx |---+--->| Commit Tx |
+------------+   |    +-----------+
                 |    +-----------+            +-----------+
                 +--->| Splice Tx |----------->| Commit Tx |
                 |    +-----------+            +-----------+
                 |    +---------------+        +-----------+
                 +--->| Splice RBF #1 |------->| Commit Tx |
                 |    +---------------+        +-----------+
                 |    +---------------+        +-----------+
                 +--->| Splice RBF #2 |------->| Commit Tx |
                      +---------------+        +-----------+
```

The rule for HTLCs in this window: "payments must be valid for all
splice transactions". An HTLC that fits the post-splice-in capacity but
not the pre-splice capacity cannot be added until the splice locks,
because the old funding transaction is still a live candidate.

Each RBF attempt adds another branch, and every branch needs its own
`commitment_signed` exchange. The commitment count grows with RBF
attempts, not with splice count.

### `splice_locked` and txid agreement

```
1. type: 77 (`splice_locked`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`sha256`:`splice_txid`]
```

The `splice_txid` field is what makes this safe under reorg. A node
MUST send `splice_locked` when any splice transaction reaches
acceptable depth, and the receiver MUST send a `warning` and close the
connection (or `error` and fail the channel) if the txid matches none
of its pending splice transactions.

Once both sides have sent and received `splice_locked`:

- **txids match** - stop sending `commitment_signed` for RBF attempts
  and ancestors; those may be discarded. If `announce_channel` is set,
  send `announcement_signatures` with the `short_channel_id` of *this*
  splice transaction.
- **txids differ** (different RBF candidates) - SHOULD ignore the
  message; MAY error and fail the channel.

The rationale is a fork, not a bug: peers on different forks can
disagree about which RBF attempt confirmed. Ignoring and waiting for
one fork to replace the other lets both sides converge and re-exchange
`splice_locked` for the same txid. Failing the channel on a txid
mismatch is spec-legal but strictly worse.

Under `option_zeroconf` a node SHOULD send `splice_locked` immediately
after exchanging `tx_signatures`, collapsing the window to zero.

### Reconnecting mid-splice

`channel_reestablish` carries two TLVs that matter here:

```
1. type: 1 (`next_funding`)
2. data:
    * [`sha256`:`next_funding_txid`]
    * [`byte`:`retransmit_flags`]

1. type: 5 (`my_current_funding_locked`)
2. data:
    * [`sha256`:`my_current_funding_locked_txid`]
    * [`byte`:`retransmit_flags`]
```

`next_funding` resumes an interrupted signing session. A node MUST
include it if it has sent `commitment_signed` for an interactive
transaction construction but has not received `tx_signatures`, setting
`next_funding_txid` to that transaction's txid; otherwise it MUST NOT
include the TLV. `retransmit_flags` bit 0 is `commitment_signed`, set
when the node has not received `commitment_signed` for that txid.

`my_current_funding_locked` handles the other case - the splice
confirmed while the peers were disconnected. If `option_splice` was
negotiated and a splice transaction reached acceptable depth during the
disconnection, the node MUST include this TLV with the txid of the
latest such transaction.

The txid is the disambiguator in both TLVs, for the same reason it is
in `splice_locked`: with several candidates outstanding, a bare "resume
the splice" signal is ambiguous.

## Worked example

A splice-in that is RBF-bumped once, then the peers disconnect.

1. Quiescence (`stfu`), `splice_init` / `splice_ack`, interactive-tx,
   `commit_sig` both ways, `tx_signatures` both ways. Channel resumes.
   Candidates: original funding, Splice A.
2. Fee too low. `tx_init_rbf` / `tx_ack_rbf`, another interactive-tx
   round, `commit_sig`, `tx_signatures`. Candidates: original funding,
   Splice A, Splice RBF #1. Three commitment branches signed.
3. HTLCs continue to flow, but only amounts valid against *all three*.
4. Disconnect after the node sent `commit_sig` for RBF #1 but before
   receiving `tx_signatures`.
5. On reconnect it sends `channel_reestablish` with `next_funding`,
   `next_funding_txid` = RBF #1's txid. If it also never received
   `commit_sig` for that txid, it sets bit 0 of `retransmit_flags`.
6. RBF #1 confirms to acceptable depth. Both send `splice_locked` with
   RBF #1's txid. Splice A and the original funding are discarded, and
   `commitment_signed` stops being sent for them.

Had the disconnection instead spanned the confirmation, step 5 would
carry `my_current_funding_locked` with RBF #1's txid rather than
`next_funding`.

## Common pitfalls

- **Assuming the channel is offline during a splice.** It is quiescent
  only until `tx_signatures`. Routing resumes with the splice pending.
- **Sizing HTLCs against the new capacity.** Until `splice_locked`,
  payments must be valid for every candidate funding transaction,
  including the pre-splice one. Spliced-in capacity is not usable
  immediately.
- **Failing the channel on a `splice_locked` txid mismatch.** The spec
  says SHOULD ignore; mismatch usually means a fork, which resolves
  itself. Erroring is permitted but throws the channel away.
- **Tracking one pending commitment.** Every RBF attempt adds a branch
  that must be signed and retained until the splice locks.
- **Treating `next_funding` and `my_current_funding_locked` as
  interchangeable.** The first resumes signing; the second reports a
  confirmation that happened while disconnected.
- **Expecting `option_splice` to imply quiescence support.** BOLT 9
  lists no dependency for 62/63, but a splice can only start on a
  quiescent channel, so `option_quiesce` (34/35) is needed in practice.

## References

- BOLT 2 "Channel Splicing", "Splice Completion", and the
  `channel_reestablish` TLV table, at `lightning/bolts` master
  (September 2026).
- BOLT 9 `09-features.md`: `option_splice` at 62/63.
- `lightning/bolts` PR #1160, merged 2026-03-23.
