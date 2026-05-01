# PTLC Submarine Swap - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/submarine-swaps`.
> Canonical source: https://github.com/lightning/bolts/blob/master/proposals/route-blinding.md (PTLC research)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/submarine-swaps/SKILL.md

## Concept

A **PTLC** (Point Time-Locked Contract) replaces the SHA256 hash in an
HTLC with a **public key** (a "point") `T`. To claim funds, the spender
must reveal `t` such that `t * G = T` (the discrete log of `T`). This
swaps the hash-preimage primitive for a discrete-log primitive, enabling:

- Unique payment hashes per part (eliminates MPP correlation).
- Adaptor-signature-based atomic swaps (cross-chain, scriptless).
- Better privacy: PTLC values are independent and look random; HTLC
  payment hashes can be probed.

PTLC submarine swaps replace the hash-preimage exchange with a
discrete-log-equality (DLEQ) exchange. The on-chain side uses Schnorr
adaptor signatures.

## Walkthrough / mechanics

### Adaptor signature primer

Standard Schnorr signature: `(R, s)` where `s = r + H(R||P||m)*x`.

**Adaptor signature**: signer commits to a **secret** `t` by giving
`(R + T, s')` where `s' = r - H(R+T||P||m)*x`. To convert into a
valid Schnorr sig, anyone can compute `s = s' + t`. So:

- Producer of adaptor sig knows everything except `t`.
- Whoever learns `t` (via on-chain spend that reveals it) can complete
  the adaptor into a valid signature.
- The point `T` is publicly bound to the swap.

### PTLC submarine swap flow

User Bob has LN balance, wants on-chain BTC. Boltz-PTLC provides
on-chain liquidity.

1. Boltz picks secret scalar `t`, computes `T = t * G`. Sends `T` to
   Bob.
2. Bob constructs Lightning PTLC paying through routes that ultimately
   require revealing `t` to claim.
3. Boltz funds an on-chain Taproot output:
   ```
   P2TR (
     internal_key = Boltz_pk + User_pk MuSig2,
     scripts:
       leaf 1 (claim): <Boltz_pk> + ADAPTOR_SIG_for_T verification
       leaf 2 (refund): timelock to user
   )
   ```
   The claim path uses an adaptor sig committing to `T`. Boltz provides
   the partial adaptor sig; the on-chain script accepts the full
   signature only when `t` is known.
4. Bob pays the LN PTLC; on success, the LN payment reveals `t` to
   Boltz (Boltz is the LN final hop).
5. Boltz now has `t`, completes the adaptor sig, broadcasts on chain
   spending the HTLC to its own wallet.
6. Bob, watching the chain, observes the completed Schnorr sig and
   extracts `t = s - s'`. (Used in cross-chain swaps to chain
   atomicity.)

### Privacy properties

- The on-chain spend looks like an ordinary Taproot key-path spend: a
  single 64-byte Schnorr signature. No HTLC script visible.
- LN payment doesn't expose hash-preimage relation; uses BIP340
  point-locked contract (research feature, not yet in mainline BOLT).
- Multiple PTLC swaps can share infrastructure without correlation
  (each has unique `T`).

### Differences vs HTLC swap

| Property | HTLC (BOLT) | PTLC |
|----------|-------------|------|
| On-chain footprint | Reveal preimage in script | Just key-path Schnorr |
| Privacy | hash visible | invisible |
| Atomicity primitive | SHA256 | Discrete log (Schnorr/secp256k1) |
| Cross-chain compat | hash matches both chains | adaptor on each chain's curve |
| BOLT support | yes (HTLC) | no, research only as of 2026 |

### Refund

Refund path identical to HTLC: a CSV-locked alternative spending
script returning funds to user after timeout.

## Worked example

Boltz-PTLC swap of 50,000 sat.

```
1. Boltz picks t = 0xa3b1c5..., T = t * G = 0297de...
2. Bob constructs LN PTLC route ending at Boltz, with point T.
3. Boltz publishes on-chain Taproot output funded with 50,000 sat.
   internal_key Q = Boltz_pk + Bob_pk MuSig2.
   tap_script (claim): partial adaptor sig (sig', T).
4. Bob pays LN PTLC. Boltz reveals t to Bob via the LN settle.
5. Boltz computes s = sig' + t, broadcasts BTC tx with witness:
     <s>  (just the schnorr signature; key-path spend, no script)
6. Tx confirms; Bob has 50,000 sat in his on-chain wallet
   (Boltz's claim came from a different on-chain UTXO; in submarine
   loop direction Bob actually receives BTC, not Boltz; corrected:
   Bob is recipient).
```

(In a Loop-out style swap, the on-chain HTLC pays Bob, and Bob's claim
uses `t`. The exact direction depends on the swap type; in any case,
the adaptor reveals `t` on chain.)

## Common pitfalls

- **PTLC not in mainline BOLT**: as of 2026, the basic Lightning
  protocol still uses HTLC. PTLC requires both endpoints + intermediate
  nodes to upgrade.
- **Adaptor verification correctness**: implementations must verify
  adaptor sig properties (R+T binding, rogue-key prevention) to avoid
  malleability.
- **MuSig2 nonce reuse**: signing across multiple swaps must use fresh
  nonces; reuse leaks the secret key.
- **Refund script-path**: still requires CSV. Setup transaction must
  include the script-path commitment.
- **Cross-chain mismatch**: BTC (secp256k1) and Liquid (secp256k1) ok.
  BTC <-> XMR (ed25519) needs DLEQ proof across curves (extra
  complexity; see Monero atomic swaps).

## References

- Andrew Poelstra. "Scriptless Scripts" 2017.
- BOLT proposals on PTLC (2022-2024).
- Lloyd Fournier work on adaptor signatures.
- Boltz Exchange research notes on PTLC.
