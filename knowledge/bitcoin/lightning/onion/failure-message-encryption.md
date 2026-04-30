# Onion Failure Message Encryption - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/onion`.
> Canonical source: BOLT 4 "Returning Errors"
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/onion/SKILL.md

## Concept

When an HTLC fails along a route, the failing hop sends a
`update_fail_htlc` carrying an *encrypted* failure message. The encryption
chain is the inverse of the onion: each upstream hop wraps the failure
in another layer using the shared secret it derived during forward
processing. Only the original sender can decrypt the full failure;
intermediate hops can wrap but not read. This article walks through
the wrapping algorithm, the failure-code format, and a forensics
recipe for diagnosing which hop failed.

## Walkthrough / mechanics

When hop i decides to fail:

```
failure_msg = bigsize_failure_code || data
hmac        = HMAC-SHA256(um_i, failure_msg || padding)
plaintext   = hmac (32B) || failure_len (2B) || failure_msg || pad_len (2B) || pad
encrypted   = ChaCha20-stream(ammag_i, plaintext)
```

Here `um_i` and `ammag_i` are the failure-onion sub-keys derived from
the same `ss_i` that the hop derived during forward routing.

The packet has a fixed 256-byte data field plus 32-byte HMAC = 292
bytes total (or 256+32 + length headers as per spec). Plaintext is
padded to 256 bytes with random data so length doesn't leak.

The failed hop sends `update_fail_htlc.reason = encrypted` upstream.

Each upstream hop, upon receiving `update_fail_htlc`, wraps:

```
re-encrypt(reason, ammag_{my_hop})
```

i.e. they ChaCha20-XOR the entire blob with their own ammag stream.
They do NOT decrypt or re-MAC; they simply add another layer.

Final hop (sender) unwraps by iterating hops in order:

```
For each hop i from 1 to k:
    reason = ChaCha20-XOR(reason, ammag_i)
    candidate_hmac = HMAC-SHA256(um_i, reason[32:])
    if candidate_hmac == reason[0:32]:
        # Hop i failed
        parse failure_msg from reason
        break
```

This iterative HMAC check identifies which hop sent the failure
(provided the sender knows the route).

## Failure codes (BOLT 4 "Failure Messages")

Selected codes (16-bit big-endian):

| Code   | Name                                        | Hop type   |
|--------|---------------------------------------------|------------|
| 0x2002 | `temporary_node_failure`                    | any        |
| 0x4007 | `temporary_channel_failure`                 | forwarder  |
| 0x4015 | `incorrect_or_unknown_payment_details`      | terminal   |
| 0x4011 | `final_incorrect_cltv_expiry`               | terminal   |
| 0x100A | `incorrect_payment_amount` (deprecated)     | terminal   |
| 0x4014 | `final_expiry_too_soon`                     | terminal   |
| 0x400F | `amount_below_minimum`                      | forwarder  |
| 0x4010 | `fee_insufficient`                          | forwarder  |
| 0x4013 | `unknown_next_peer`                         | forwarder  |
| 0x400D | `incorrect_cltv_expiry`                     | forwarder  |
| 0x100C | `expiry_too_far`                            | forwarder  |
| 0x4007 | `MPP_TIMEOUT` (modern)                      | terminal   |

High-bit conventions: bit 15 = `BADONION`, bit 14 = `PERM`, bit 13 =
`NODE`, bit 12 = `UPDATE`. Codes with `UPDATE` carry a fresh
`channel_update` so sender refreshes routing data.

## Worked example

Route Alice -> Bob -> Carol -> Dave; payment fails at Carol due to
insufficient balance:

```
Carol -> Bob:
  update_fail_htlc(reason = E_C(0x4007 || channel_update_C-D || pad))
  where E_C means encrypt with ammag_C and prepend HMAC(um_C, msg)

Bob -> Alice:
  update_fail_htlc(reason = E_B(prev_reason))   // ChaCha-stream wrap

Alice unwraps:
  step 1: r1 = ChaCha-XOR(reason, ammag_B)
          hmac_check(um_B, r1[32:]) -- fails (Bob did not generate)
          continue
  step 2: r2 = ChaCha-XOR(r1, ammag_C)
          hmac_check(um_C, r2[32:]) -- passes!
          Carol failed with 0x4007 (temporary_channel_failure)
          Parse channel_update_C-D from r2 payload
          Update mission control: penalize C->D
          Use new channel_update for next attempt
```

LND surfaces this via `lncli trackpayment`:
```
lncli trackpayment <payment_hash> --json | jq '.htlcs[].failure'
{
  "code": "TEMPORARY_CHANNEL_FAILURE",
  "channel_update": { ... fresh update ... },
  "failure_source_index": 2   // 0-indexed, so hop 2 (Carol)
}
```

CLN:
```
lightning-cli waitsendpay <payment_hash> | jq '.failuremsg'
```

LDK exposes failure via `Event::PaymentPathFailed` with
`PathFailure::OnPath { network_update }` — the channel_update is
automatically applied to the scorer.

## Common bugs / pitfalls

- **Wrapping order**: each upstream hop wraps in the order they processed
  the forward packet. Reversing wraps causes sender HMAC checks to
  always fail (UNREADABLE_FAILURE).
- **`um` vs `ammag` mix-up**: HMAC uses `um`, encryption uses
  `ammag`. Swapping them produces noise and unreadable failures.
- **Padding leak**: plaintext is 256 bytes; non-uniform padding leaks
  failure length. Spec mandates uniform padding to total 256B.
- **Forwarder failure of own choice**: a forwarder failing the HTLC
  *for itself* (e.g. balance insufficient) wraps with its OWN um/ammag
  keys. A forwarder relaying a failure from downstream wraps with the
  same keys — sender's iteration finds the right hop either way.
- **`channel_update` injection**: hops can lie in the included
  `channel_update`. Sender MUST verify signature against the gossip
  table before applying. Older LND ignored the signature, allowing
  griefing.
- **MPP failure correlation**: receiver failing one MPP part for
  `mpp_timeout` means *all* parts should be retried as a unit. Failing
  to do so causes lost funds split across stuck HTLCs.
- **Replay of wrapped failure**: a malicious upstream hop could
  re-send the same wrapped failure for a future HTLC; HMAC tied to
  `um` (which is per-`ss`, per-onion) prevents this — but only if
  `ss` is non-replayed (which is what onion replay protection
  enforces).

## References

- BOLT 4 - Returning Errors: https://github.com/lightning/bolts/blob/master/04-onion-routing.md#returning-errors
- BOLT 4 - Failure Messages: https://github.com/lightning/bolts/blob/master/04-onion-routing.md#failure-messages
- LDK error decoding: `lightning/src/ln/onion_utils.rs` (`process_onion_failure`)
- CLN: `common/onion.c`, `common/sphinx.c`
- LND failure handler: `routing/result_interpretation.go`
