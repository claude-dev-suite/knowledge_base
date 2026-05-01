# Adaptor-Signature Atomic Swap - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/atomic-swaps`.
> Canonical source: https://github.com/ElementsProject/scriptless-scripts/blob/master/md/atomic-swap.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/atomic-swaps/SKILL.md

## Concept

A scriptless atomic swap uses **adaptor signatures** so that the on-chain
transactions look like ordinary single-sig (Schnorr) spends — no hashlock,
no timelock script, just a P2TR key-path spend. The atomicity comes from
the cryptographic relation between two adaptor signatures rather than
matching preimages in HTLCs.

This is the swap design used in production by Lloyd Fournier's research,
the LEAF protocol, and the experimental Bitcoin <-> Liquid LBTC swaps.

## Walkthrough / mechanics

### Adaptor signature primer (Schnorr)

For Schnorr `sig = (R, s)` where `s = r + e * x` and `e = H(R || P || m)`:

- An **adaptor signature** is `(R + T, s')` where `T = t * G` is the
  "adaptor point" and `s' = r + e*x`. So `s' + t = s` for the **complete**
  signature `(R + T, s' + t)`.
- Anyone with `s'` and the complete sig `s' + t` can extract
  `t = (s' + t) - s'`.

### Cross-chain swap protocol

Alice has BTC, Bob has LTC. Both want to atomically swap.

1. **Setup**. Alice picks a secret scalar `t`, computes `T = t * G`.
   She constructs PSBT_A spending her UTXO to a 1-of-1 P2TR
   `Q = A + B` (MuSig2 aggregated key). Bob constructs PSBT_B on LTC
   spending his UTXO to LTC-MuSig2 aggregated key.
2. **Adaptor exchange**. Alice signs PSBT_B (the LTC tx) but with adaptor
   point `T`: she sends `s'_A`. The signature is "missing" `t` to be
   complete. Bob signs PSBT_A (the BTC tx) likewise: `s'_B` with adaptor
   `T`.
3. **First broadcast**. Alice (already knows `t`) completes her signature
   on PSBT_B: `s_A_complete = s'_A + t`. Broadcasts on **LTC** chain. The
   tx appears as a normal taproot key-path spend.
4. **Extract & broadcast**. Bob observes the LTC tx, reads `s_A_complete`
   from the witness, computes `t = s_A_complete - s'_A`. He completes
   his signature on PSBT_A: `s_B_complete = s'_B + t`. Broadcasts on BTC.
5. Both swaps complete. To outsiders, both transactions are
   indistinguishable from any other taproot key-path spend.

### Refund path

If Alice does not broadcast on LTC by time `T_refund`, both parties
publish a pre-agreed **timelocked** refund tx (a separate PSBT with
`OP_CSV`). This still requires script in the refund branch but lives in
the Tapscript leaf, not the key-path that is normally broadcast.

## Worked example

Pre-swap setup (numbers symbolic):

```
Alice keypair: x_A, A = x_A * G
Bob   keypair: x_B, B = x_B * G
Adaptor: t random; T = t*G
MuSig2 aggregate: Q = A + B (with rogue-key fix terms in real spec)
```

Resulting on-chain footprint:

| Chain | Tx | Witness | Looks like |
|-------|----|---------| -----------|
| LTC   | sends to Q' | 64-byte Schnorr sig | normal P2TR spend |
| BTC   | sends to Q  | 64-byte Schnorr sig | normal P2TR spend |

Block-explorer chain analysis sees no swap; both txs look unrelated.

## Common pitfalls

- **MuSig2 nonce reuse**: nonces in MuSig2 are extremely sensitive.
  Re-using a nonce across a successful and a failed swap leaks the
  signing key. Use deterministic nonces per BIP327 and erase on success.
- **Refund-path trust**: the timelocked refund **must be exchanged before
  funding**. If the funder broadcasts before receiving the counter-signed
  refund, they can be griefed by an unresponsive counterparty.
- **Cross-chain timelock asymmetry**: BTC blocks ~10 min, LTC ~2.5 min.
  Set Alice's refund timelock significantly shorter than Bob's so a
  reorg cannot cause both refunds to be valid simultaneously.
- **Adaptor point binding**: `T` must be cryptographically bound to the
  swap; a malicious party who reuses `T` across two simultaneous swaps
  with the same victim can extract `t` from the first and steal the
  second. Use a deterministic `T = H(swap_id) * G`.
- **Quantum resistance**: adaptor sigs rely on discrete log; a future
  quantum attacker who sees the in-flight adaptor sig before
  completion can extract `t`. Same caveat as Schnorr in general.

## References

- Andrew Poelstra. "Scriptless Scripts" 2017.
- BIP327 — MuSig2 for BIP340.
- "Cross-chain atomic swaps via adaptor signatures" — Fournier 2019.
