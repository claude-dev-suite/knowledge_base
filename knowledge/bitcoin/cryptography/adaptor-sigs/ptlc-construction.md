# PTLC Construction - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/adaptor-sigs`.
> Canonical source: bolt-PTLC drafts + Bastien Teinturier "Trampoline Onion
>                   and PTLCs" + Lightning research notes
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/adaptor-sigs/SKILL.md

## Concept

Hash Time-Locked Contracts (HTLCs) lock funds on revealing a SHA256 preimage
matching a pre-shared hash; every hop in a Lightning route shares the same
hash, which leaks routing structure to colluding nodes. Point Time-Locked
Contracts (PTLCs) replace the SHA256 preimage with a discrete-log scalar `t`
satisfying `t G = T` for a per-hop **adaptor point** `T`. Each hop along a
multi-hop payment uses a *different* adaptor point derived from the same
final secret via point addition, so on-chain or wormhole observers cannot
correlate hops by the public locking value. This article shows the math of
the per-hop derivation, the on-chain commitment-tx-locked output, and the
HTLC-style timeout / success spend paths.

## Walkthrough / mechanics

### Per-hop adaptor derivation

Final destination payee chooses random `t_final`, `T_final = t_final G`. The
sender constructs a route Alice -> Carol -> Bob -> Dave (the payee). For each
hop, the sender picks a per-hop blinding scalar `delta_h`:

```
For hop h at relay node R_h on the route:
  delta_h    <- {1..n-1}    (per-hop blinding, sampled by sender)
  T_h        = T_{h-1} + delta_h * G        // h=0 starts at T_final
                                            // T_h is what R_h sees
  t_h        = t_{h-1} + delta_h            // R_h does NOT see this; only
                                            // each pair of adjacent hops
                                            // exchanges (delta_h) via onion
```

Inductive invariant: `t_h G = T_h` at every hop. The payee at the end of the
route holds `t_final = t_0`. Each upstream hop, when it observes the next-hop
node settle the inbound HTLC, learns `t_{h+1} = t_h + delta_{h+1}` because
delta_{h+1} was placed in the onion's payload for hop h. From `t_{h+1}` it
computes `t_h = t_{h+1} - delta_{h+1}` and uses it to settle its own pre-sig.

### Onion payload contents per hop

Standard HTLC onion carries: amount, CLTV, channel hint. PTLC onion adds:

```
PTLC payload for hop h:
  - amount_to_forward
  - outgoing_cltv_value
  - outgoing_channel_id  (or blinded route)
  - delta_h+1                        // 32-byte scalar; xor'd with hop's
                                     // shared secret in Sphinx
  - T_h+1 commitment                 // optional, allows hop to verify
                                     // continuity
```

`delta_{h+1}` is encrypted under the hop-shared Sphinx secret, so only this
hop ever sees it.

### Output script (in commitment tx)

A PTLC output looks structurally like an HTLC output but the success branch
uses a Schnorr-Taproot adaptor sig instead of a SHA256 preimage check.

```
TapLeaf: success branch
  <local_pubkey> OP_CHECKSIG
  // Witness: Schnorr sig over the success-spend tx,
  //          where the sig was a pre-sig adapted with t_h
  //          (i.e., sig is bound to revealing t_h)

TapLeaf: timeout branch
  <CSV_timeout> OP_CHECKSEQUENCEVERIFY OP_DROP
  <remote_pubkey> OP_CHECKSIG
  // Refund path after timeout, no t needed
```

Internally, the success path is implemented as: when the upstream hop signs
the commitment tx update that creates this PTLC output, they pre-sign the
**spending** tx for the success branch as a Schnorr adaptor sig with
adaptor point `T_h`. The commitment-tx update itself includes that pre-sig
exchange.

### Settlement flow (downstream->upstream)

```
Time T0:  Sender starts payment. Builds onion. Each hop's pre-sig over the
          inbound HTLC's success-spend is created with adaptor point T_h.
          Pre-sigs flow forward; each hop verifies and forwards.

Time T1:  Payee receives payment. Reveals t_final to the last hop by
          publishing the success-spend on chain (or by sending the completed
          sig to the upstream peer in the cooperative case).

Time T2:  Last hop computes t_{n-1} = t_final - delta_n (delta_n was in its
          onion). Adapts its inbound pre-sig and either claims on chain or
          settles cooperatively.

Time T3:  Each upstream hop sees the next hop settle, recovers its delta_h,
          subtracts to get t_{h-1}, claims its inbound HTLC, etc.
```

The discrete-log-secret cascade is identical in shape to the SHA256 cascade
in HTLCs, but each hop sees a different `T_h` rather than the same `H`.

### Privacy advantage

```
HTLC route observation:
  Hop 1 sees H = H_1
  Hop 2 sees H = H_1   (same value)
  Hop 3 sees H = H_1
  -> Colluding hops 1 and 3 know they're on the same payment.

PTLC route observation:
  Hop 1 sees T_1
  Hop 2 sees T_2 = T_1 + delta_2 G   (looks unrelated)
  Hop 3 sees T_3 = T_2 + delta_3 G   (looks unrelated)
  -> Without delta values, T_i are uniformly random points.
```

Even when a hop sees the upstream's pre-sig and downstream's settled sig, it
cannot derive `delta_h` from on-chain data alone; the onion payload is the
only source.

### Wormhole resistance

In HTLC routes, a malicious node can "steal" the preimage by observing the
downstream settlement and skipping fee payment to upstream peers. PTLCs
defeat this because each hop's `T_h` is different — knowing `t_h` for hop h
doesn't help finalize hop h-2's pre-sig (which uses `T_{h-2}`).

## Worked example

3-hop payment Alice -> Carol -> Bob -> Dave. Toy `n = 23`.

```
Dave picks t_final = t_0 = 7, T_0 = 7G.

Sender chooses:
  delta_1 = 4   (hop Carol -> Bob exit)
  delta_2 = 11  (hop Alice -> Carol exit; this would be the
                 last delta added going outbound from sender)

T_1 = T_0 + delta_1 G = 7G + 4G = 11G
T_2 = T_1 + delta_2 G = 11G + 11G = 22G mod 23 = 22G

Pre-sigs (each over the success-spend of the corresponding outbound HTLC):
  Alice -> Carol HTLC:  Alice pre-signs with adaptor T_2 = 22G
  Carol -> Bob HTLC:    Carol pre-signs with adaptor T_1 = 11G
  Bob -> Dave HTLC:     Bob pre-signs with adaptor T_0 = 7G

Onion payloads:
  Alice -> Carol: includes delta_2 = 11 (encrypted to Carol)
  Carol -> Bob:   includes delta_1 = 4  (encrypted to Bob)
  Bob -> Dave:    payload for Dave (no delta needed; T_0 corresponds to
                  Dave's known t_final)

Settlement:
  Dave settles t_0 = 7. Bob observes (or receives cooperatively).
  Bob computes Carol's secret: t_1 = t_0 + delta_1 = 7 + 4 = 11
  Bob settles his inbound HTLC by adapting Carol's pre-sig with t_1 = 11.
  Carol observes. Carol computes Alice's secret:
    t_2 = t_1 + delta_2 = 11 + 11 = 22
  Carol settles inbound HTLC by adapting Alice's pre-sig with t_2 = 22.

Verify per-hop equation: t_2 G = 22 G = T_2.   OK
                         t_1 G = 11 G = T_1.   OK
                         t_0 G = 7 G  = T_0.   OK
```

A passive observer of the chain sees three settlement spends with three
unrelated adaptor points and three unrelated final sigs — no correlation.

## Common pitfalls

- **Reusing `T` between unrelated payments.** Every payment needs a fresh
  `T_final`. If two payments share `T_final`, observing settlement of one
  reveals the secret for the other.
- **Forgetting cancel/timeout PTLC variant.** Lightning HTLCs have separate
  "offered" and "received" structures with asymmetric timeouts. PTLCs need
  the same; the success and timeout branches must be co-designed so neither
  party can grief the other.
- **Mixing HTLC and PTLC hops in the same route.** Currently impossible
  without a "translator" hop that knows both `H` and `T`. Bolt drafts cover
  this but it weakens privacy at the boundary.
- **Adaptor sig MuSig2 round-trip.** PTLCs in Lightning typically use
  MuSig2-aggregated channel keys. The pre-sig must be a MuSig2 partial
  pre-sig with adaptor `T`, not a single-key Schnorr pre-sig. Mixing the
  two breaks the channel.
- **CLTV window mis-sizing.** Each hop's outbound CLTV must give enough room
  for upstream-side settlement after a contested close. Too tight a window
  forces an upstream party to refund while their pre-sig is still adaptable.

## References

- Bastien Teinturier, "Trampoline onion and PTLCs":
  https://github.com/lightning/bolts/issues/865
- Mike Hearn / Lightning research, "PTLCs" (notes):
  https://lists.linuxfoundation.org/pipermail/lightning-dev/2018-April/001245.html
- Riard & Naumenko, "PTLCs and BOLTs":
  https://github.com/lightning/bolts/pulls?q=ptlc
- Aumayr et al., "Generalized Channels": https://eprint.iacr.org/2020/476
