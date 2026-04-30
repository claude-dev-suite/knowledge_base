# PTLC vs HTLC - Detailed Comparison

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/htlcs`.
> Canonical source: BOLT 3 (HTLC scripts), BIP-340/341, "PTLC over Lightning" papers
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/htlcs/SKILL.md

## Concept

HTLCs use a hash-preimage pair `(H, r)`: the commitment locks coins
to anyone who reveals `r` such that `SHA256(r) = H`. PTLCs (Point Time-Locked
Contracts) replace this with a Schnorr point-secret pair `(T, t)` where
`t * G = T`. Functionally equivalent — both gate spends on revealing a
secret — but PTLCs swap the SHA256 lock for a *point* that can be
combined with adaptor signatures, enabling stronger privacy and new
protocols (PTLC-based atomic swaps, blinded amounts).

## Walkthrough / mechanics

Cryptographic primitive comparison:

| Property              | HTLC                          | PTLC                              |
|-----------------------|-------------------------------|-----------------------------------|
| Lock                  | `H = SHA256(r)`               | `T = t * G`                       |
| Reveal                | `r`                           | adaptor signature → `t`           |
| Witness on-chain      | preimage in script execution  | Schnorr signature                 |
| Per-hop secret        | same `r` everywhere           | different `t_i` per hop           |
| Script complexity     | `HASH160(r) OP_EQUALVERIFY ...` | key-spend with adaptor          |
| Witness size          | ~140 bytes (HTLC-success)     | ~64 bytes (Schnorr)               |

The privacy win: in a multi-hop HTLC payment, every channel records the
same `H` on chain if force-closed. A blockchain analyst correlating
two force-closes can prove they were on the same payment route. With
PTLCs, each hop derives `T_i = T - sum(b_j for j<i)*G` for blinding
factors b_j, so on-chain points differ.

Adaptor signature mechanics (BIP-340 Schnorr base):

```
Standard Schnorr sig:    s = k + H(R || P || m) * x       where R = k*G
Adaptor pre-sig:         s' = k + H(R + T || P || m) * x   (note R+T)
Recipient verifies:      s'*G == R + T + H(...)*P
After learning t:        s = s' + t  (now a valid sig)
And:                     t = s - s'
```

Anyone seeing both `s'` (the pre-sig) and `s` (final sig) can extract
`t = s - s'`. This means revealing the final signature on chain reveals
the secret to the previous hop, propagating the secret backward exactly
as a preimage does in HTLCs.

In Lightning, each hop holds `s'_i` as the "pre-signature" for that
hop's HTLC analogue. Forwarding works:

```
Receiver: has t (secret), publishes s' + t = s in cooperative claim or
                          script-path on force-close.
Hop 3:    sees s_3, computes t = s_3 - s'_3, publishes s'_2 + t = s_2.
Hop 2:    same.
Hop 1:    same; sender sees s_1, knows payment completed.
```

## Worked example

Hypothetical PTLC route (no production deployment as of writing):

```
Sender derives:         t = receiver's secret point's discrete log
                        T_sender = t * G
                        b_2 = random blinding for hop 2
                        b_3 = random blinding for hop 3
                        T_hop1 = T_sender (last hop)
                        T_hop2 = T_sender - b_3 * G  (intermediate)
                        T_hop3 = T_sender - b_2*G - b_3*G  (first)

Sender -> hop 1: PTLC commitment locked at T_hop3
                 Onion contains: b_2 (so hop 1 can derive next T)
Hop 1 -> hop 2:  PTLC at T_hop2 (= T_hop3 + b_2*G)
                 Onion: b_3
Hop 2 -> Receiver: PTLC at T_hop1 (= T_hop2 + b_3*G = T_sender)

Receiver knows t, signs adaptor with t to claim.
Hop 2 sees signature, extracts t, claims via t + b_3 from hop 1.
Hop 1 sees, extracts t + b_3, claims via (t + b_3) + b_2 from sender.
```

The on-chain footprint at each force-close shows DIFFERENT points
(`T_hop1`, `T_hop2`, `T_hop3`), unlinkable without knowing the
blindings.

PTLC + MuSig2 commitment tx: cooperative spend uses a single Schnorr
signature, no script revealed. Force-close uses Tapscript leaf
containing the adaptor verification.

## Common bugs / pitfalls

- **Adaptor signature reuse**: signing the same message twice with
  different adaptor points reveals the secret key. Each hop's pre-sig
  must use a unique nonce — and Schnorr nonce reuse is catastrophic
  (recovers signer key).
- **Blinding factor leakage**: if hop 2 learns `b_3`, it can correlate
  hops 1 and 2. Onion encryption must keep blindings hop-local.
- **MPP atomicity**: HTLC MPP works because all parts share `H`. PTLC
  MPP needs a Schnorr-aggregation protocol (research stage). Sending
  an MPP with vanilla PTLCs makes parts fail individually.
- **No standard yet**: there is no merged BOLT for PTLCs at time of
  writing. Production claims are misleading; testnet-only for LDK and
  research forks.
- **Revocation**: HTLC revocation key path works on hash. PTLC
  revocation needs a separate revocation point (`R_revoke`) per
  commitment — doubles the per-commitment key derivation work.
- **Privacy assumes Taproot channels**: PTLCs are only meaningful if
  the commitment uses key-path spends most of the time. Mixing PTLCs
  on legacy channels reveals the script and undoes privacy.
- **Older nodes can't forward**: a network with mixed PTLC/HTLC nodes
  must convert at boundaries (e.g. LSP-style nodes). The conversion
  hop is a privacy weak point.

## References

- BOLT 3 HTLC scripts: https://github.com/lightning/bolts/blob/master/03-transactions.md
- "Flipping the scriptless script" (Poelstra, 2017): https://github.com/apoelstra/scriptless-scripts/blob/master/md/atomic-swap.md
- "PTLCs in Lightning" (AJ Towns): https://gnusha.org/url/https://lists.linuxfoundation.org/pipermail/lightning-dev/2022-January/003428.html
- BIP-340 Schnorr: https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
- LDK adaptor sig WIP: `bitcoin-rs/secp256k1` zkp module
