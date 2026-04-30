# Dual-Funded Channels (v2) - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/channels`.
> Canonical source: BOLT 2 PR #851 (interactive-tx) and #1052 (open_channel2)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/channels/SKILL.md

## Concept

The original BOLT 2 funding flow assumes one party funds the channel
unilaterally. v2 dual-funding (`option_dual_fund`, bits 28/29) lets both
peers contribute inputs and outputs to the funding transaction in a single
interactive negotiation. The mechanism is *interactive transaction
construction* — a generic primitive also reused for splicing and the
liquidity-ads spec. Understanding the dual-fund flow is therefore the key
to understanding splicing too.

## Walkthrough / mechanics

Message sequence (BOLT 2 section "Channel Establishment v2"):

```
Initiator                         Responder
   |  open_channel2                  |
   |---------------------------------|
   |  accept_channel2                |
   |<--------------------------------|
   |  tx_add_input                   |
   |---------------------------------|
   |  tx_add_input  (or tx_complete) |
   |<--------------------------------|
   |  tx_add_output                  |
   |---------------------------------|
   |  tx_add_output (or tx_complete) |
   |<--------------------------------|
   |  ...                            |
   |  tx_complete                    |
   |---------------------------------|
   |  tx_complete                    |
   |<--------------------------------|
   |  commitment_signed              |
   |---------------------------------|
   |  commitment_signed              |
   |<--------------------------------|
   |  tx_signatures                  |
   |---------------------------------|
   |  tx_signatures                  |
   |<--------------------------------|
   broadcast funding tx
```

`open_channel2` adds: `funding_feerate_perkw`, `locktime`, `require_confirmed_inputs`,
and removes `push_msat` (replaced by RBF/contribution mechanics).

The interactive-tx FSM enforces these invariants:
- Each side adds at most `max_inputs` (default 252) and `max_outputs` (252).
- Inputs MUST be Segwit (so wtxid is stable while signing).
- Serial IDs: initiator uses even, responder uses odd. Prevents collisions.
- Both sides must alternate. A `tx_complete` only counts after the *peer*
  also sends `tx_complete` — two consecutive completes finalize.
- Input weight + change tracking: each side computes contribution and
  must hit fee target proportionally.

After `tx_complete`/`tx_complete`, both sides know the full funding tx and
its txid (deterministic). Commitment sigs follow, then `tx_signatures`
exchanges the witness data for each side's contributed inputs.

RBF: either side can send `tx_init_rbf` after broadcast and the negotiation
restarts with new inputs/outputs but the same channel state pre-active.

## Worked example

Peer A (LSP, contributes 0.5 BTC liquidity), Peer B (mobile wallet, 0.1 BTC):

```
B -> A: open_channel2
  funding_feerate_perkw: 2500
  funding_satoshis: 10_000_000  (B's contribution)
  ...
A -> B: accept_channel2
  funding_satoshis: 50_000_000  (A's contribution)
  require_confirmed_inputs: true
B -> A: tx_add_input
  serial_id: 0
  prevtx: ...      (B's UTXO 0.105 BTC)
A -> B: tx_add_input
  serial_id: 1     (A's UTXO 0.55 BTC)
B -> A: tx_complete
A -> B: tx_add_output
  serial_id: 3     (funding output, 60_000_000 sats, MuSig2 P2TR or 2-of-2)
B -> A: tx_add_output
  serial_id: 0     (B change ~0.005 BTC)
A -> B: tx_add_output
  serial_id: 5     (A change ~0.05 BTC)
B -> A: tx_complete
A -> B: tx_complete
... commitment_signed exchange ...
B -> A: tx_signatures (B's witness)
A -> B: tx_signatures (A's witness)
```

CLN command (initiator side) using `fundchannel` v2 path:
```
lightning-cli fundchannel id=03abc... amount=10000000 \
    feerate=2500perkw mindepth=3 utxos='["txid:vout"]'
```

When peer is dual-fund-capable, CLN auto-uses v2; an LSP plugin
(`fundchannel_hook`) can override `accept_channel2` to add liquidity.

## Common bugs / pitfalls

- **Non-segwit inputs**: BOLT-PR explicitly disallows. Bitcoin Core wallet
  legacy UTXOs cause `tx_abort` from a strict peer.
- **`require_confirmed_inputs=true` race**: if either peer's UTXO is
  unconfirmed, the responder sends `tx_abort`. Some wallets retry instantly
  and create a tight loop until the input confirms.
- **Fee underestimation**: each side pays for their contributed weight
  pro-rata; failing to account for the funding output weight (one side
  owes it) causes underpayment and tx rejection by the mempool.
- **RBF replay**: `tx_init_rbf` must keep at least one common input or
  the new tx is not technically a replacement. Some mempool policies
  reject it.
- **Serial-id reuse**: reusing a serial id within the same negotiation
  is an immediate `must-fail-channel` error. RBF round restarts the id
  space.
- **Commitment sig before tx_complete pair**: protocol error; the FSM
  must wait for two consecutive `tx_complete` before signing.

## References

- BOLT 2 - Peer Protocol (channel establishment v2): https://github.com/lightning/bolts/blob/master/02-peer-protocol.md
- Interactive-tx PR: https://github.com/lightning/bolts/pull/851
- Eclair dual-fund implementation notes: `eclair-core/src/main/scala/fr/acinq/eclair/channel/fund/InteractiveTxBuilder.scala`
- CLN dual-fund: `openingd/dualopend.c`
