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
recipe for diagnosing which hop failed. It then covers
`option_attribution_data` (feature 36/37), which layers per-hop hold
times and an HMAC chain on top of that legacy scheme, and the
`fulfillment_payload` success-side return field.

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

## Attribution data (`option_attribution_data`, feature 36/37)

The unwrap loop above tells the sender *which* hop returned a failure only
when the failure is readable. If any hop corrupts the blob, every HMAC
check fails and the sender learns nothing beyond "the route broke
somewhere". `option_attribution_data` (BOLT 9 bits 36/37, merged into
BOLT 4 on 2025-11-17 by bolts PR #1044) closes that gap with a second,
separately-authenticated return field carried as TLV type 1 on **both**
`update_fail_htlc` and `update_fulfill_htlc`:

```
1. type: 1 (attribution_data)
2. data:
   * [20*u32          : htlc_hold_times]    # units of 100 ms
   * [210*sha256[..4] : truncated_hmacs]    # 32B HMACs cut to 4B
```

### Per-hop transformation

Each intermediate hop, before wrapping the return packet with `ammag`:

```
1. shift htlc_hold_times right by 4 bytes (one u32 slot)
2. shift-and-prune truncated_hmacs: at each step back, one HMAC per hop
   becomes unreachable (it would imply position 21) and is discarded
3. write own hold time into the freed slot at the front
4. compute own 20 truncated HMACs, one per position it might occupy,
   into the freed slots at the front
5. XOR the whole attribution_data block with a stream from a new key
   type `ammagext` (derived from the same ss_i)
```

`hmac_x_y` is the HMAC added by the node `x` hops upstream of the erring
node, assuming it is `y` hops away from it. Its input is, in order:

- the return packet *before* the pseudo-random stream is applied,
- the first `y+1` entries of `htlc_hold_times`,
- the `y` downstream HMACs at the corresponding positions.

Hence the 20x20 grid collapses to 20+19+...+1 = 210 stored HMACs, and a
hop cannot alter anything downstream of itself without invalidating its
own HMAC. Truncation to 4 bytes is a deliberate size/security trade: a
guessing node is caught in expectation and penalised in future
pathfinding.

The erring node itself initializes the block: its hold time first, the
rest zeroed, and all 20 of its HMACs computed (only `hmac_0_0` is
strictly needed, but a uniform layout keeps origin-side verification
cheap).

### Origin-side verification

The origin derives `ammag`, `ammagext` and `um` for every hop and, at
each hop, additionally:

```
verify truncated_hmacs[position(i)] under um_i
  -> valid   : record htlc_hold_times[i], continue
  -> invalid : stop; penalize this hop for future path selection
```

A valid chain up to hop `k` and a break at `k+1` is what makes a failure
**attributable**. Blame still lands on a *pair* of nodes, not one node —
the origin cannot tell whether the sender or the receiver of that link
mangled the data — which is the same granularity Lightning gives
elsewhere. When not every hop advertises 36/37, attribution simply runs
out at the first unsupporting hop.

Recorded hold times are the latency side-channel this buys: a hop that
reports 0 (permitted for nodes without accurate timing) forces the origin
to spread any latency penalty over several hops, which is the incentive
to report honestly. This is the measurement that hold-time reputation
schemes for channel jamming build on.

### Blinded paths opt out

Hops that received `path_key` in `update_add_htlc` do **not** contribute
attribution data. They fail with `invalid_onion_blinding` via
`update_fail_malformed_htlc`, and BOLT 4 notes they therefore cannot
provide timing information — per-hop hold times inside a blinded path
would help the sender locate the recipient. Attribution stops at the
introduction node.

### 32 KiB caps (bolts PR #1349, merged 2026-08-26)

To keep room for `attribution_data` in the message, return fields are
now bounded:

| Field | Limit | Oversized behaviour |
|-------|-------|---------------------|
| return packet (`reason`) | 32768 bytes | intermediate node MUST truncate to the first 32768 bytes before transforming |
| `fulfillment_payload`    | 32768 bytes | receiver MUST send `error` and fail the channel |

The asymmetry is historical: earlier versions permitted larger return
packets, so rejecting them would break existing peers, whereas
`fulfillment_payload` was introduced with the limit already in place and
an oversized one is a protocol violation.

## Success-side return data (`fulfillment_payload`)

bolts PR #1344 (merged 2026-07-27) added the success counterpart of
`reason`: TLV type 3 on `update_fulfill_htlc`, originated by the **final**
node.

```
plaintext = fulfillment_payload_tlvs         # type 1 = padding;
                                             # no other types defined yet
            padded to >= 256 bytes and a multiple of 256,
            excluding the 16-byte tag
key       = key type `fulfillment` from ss_final
payload   = ChaCha20-Poly1305(key, nonce = all zeroes, plaintext) || tag
```

Relay mirrors the failure path: each intermediate hop obfuscates the
payload with its own `ammag` key exactly as it wraps a return packet; the
final node does not, because its ciphertext is already the innermost
layer. Intermediate `attribution_data` HMACs additionally cover the
`fulfillment_payload` *as received from downstream* (before that node's
own `ammag` is applied); the final node's HMACs do not cover it.

Unlike `attribution_data`, this is not gated on `path_key`: a blinded
final node MAY originate a payload and blinded hops obfuscate and relay
it, because the origin shares a secret with every hop — using the blinded
pubkey for blinded ones. A recipient that pads its blinded path with
dummy hops must originate the payload as though it were the last hop,
pre-applying the concealed hops' obfuscation, so the origin's fixed peel
count still decodes without revealing the recipient's position.

Two integrity checks with deliberately different extents:

- the Poly1305 tag is end-to-end and detects tampering by any hop,
  blinded or not;
- the attribution HMACs give per-hop blame, but only across hops that
  contribute them.

The origin MUST ignore a payload whose tag is invalid, or whose decrypted
TLV stream has malformed lengths, duplicate or unordered types, or
unknown even types, and MUST ignore `padding` records in a valid stream.

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
- **`ammag` vs `ammagext`**: the return packet is obfuscated with
  `ammag`, the `attribution_data` block with `ammagext`. Reusing
  `ammag` for both yields HMACs that never verify at the origin, so
  every failure looks unattributable.
- **Forgetting to instantiate attribution data**: an intermediate hop
  that receives no `attribution_data` from downstream (the downstream
  node does not support 36/37) MUST instantiate an all-zeroes block
  before updating it, not omit the field.
- **Rejecting instead of truncating**: a return packet larger than
  32768 bytes from a pre-cap peer must be truncated to its first
  32768 bytes, not dropped. Only an oversized `fulfillment_payload`
  is a channel-failing protocol violation.
- **Replay of wrapped failure**: a malicious upstream hop could
  re-send the same wrapped failure for a future HTLC; HMAC tied to
  `um` (which is per-`ss`, per-onion) prevents this — but only if
  `ss` is non-replayed (which is what onion replay protection
  enforces).

## References

- BOLT 4 - Returning Errors: https://github.com/lightning/bolts/blob/master/04-onion-routing.md#returning-errors
- BOLT 4 - Failure Messages: https://github.com/lightning/bolts/blob/master/04-onion-routing.md#failure-messages
- BOLT 4 - Successful Payments (`fulfillment_payload`): https://github.com/lightning/bolts/blob/master/04-onion-routing.md#successful-payments
- BOLT 2 - `update_fulfill_htlc` / `update_fail_htlc` TLVs: https://github.com/lightning/bolts/blob/master/02-peer-protocol.md#removing-an-htlc-update_fulfill_htlc-update_fail_htlc-and-update_fail_malformed_htlc
- bolts PR #1044 (attribution data, merged 2025-11-17): https://github.com/lightning/bolts/pull/1044
- bolts PR #1344 (`fulfillment_payload`, merged 2026-07-27): https://github.com/lightning/bolts/pull/1344
- bolts PR #1349 (32 KiB caps, merged 2026-08-26): https://github.com/lightning/bolts/pull/1349
- LDK error decoding: `lightning/src/ln/onion_utils.rs` (`process_onion_failure`)
- CLN: `common/onion.c`, `common/sphinx.c`
- LND failure handler: `routing/result_interpretation.go`
