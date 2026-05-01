# Bitcoin <-> Monero Atomic Swap Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/atomic-swaps`.
> Canonical source: https://github.com/comit-network/xmr-btc-swap (research) and Joel Gugger paper 2020
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/atomic-swaps/SKILL.md

## Concept

Bitcoin (secp256k1) and Monero (ed25519) live on different curves with
different transaction models. Yet a trustless, on-chain atomic swap is
possible using **two-curve adaptor signatures** plus a **DLEQ proof**
(Discrete Log Equality across curves). The result: BTC moves on Bitcoin,
XMR on Monero, no third party, no escrow.

## Walkthrough / mechanics

### Cryptographic primitives

- secp256k1 generator `G`, ed25519 generator `H`.
- Two private keys derived from one shared secret `s`: `s*G` (Bitcoin
  pubkey) and `s*H` (Monero pubkey).
- **DLEQ** proof: a non-interactive zero-knowledge proof that the same
  scalar `s` was used on both curves. \~256 bytes.

### Roles

- **Alice** has BTC, wants XMR.
- **Bob** has XMR, wants BTC.

### Protocol

1. **Setup**. Both parties generate two-curve keypairs. Bob picks a
   random scalar `r_B`; computes `R_B = r_B * G`, `R_B' = r_B * H`, and a
   DLEQ proof. Sends both points + proof to Alice. Alice verifies. Alice
   sends Bob her two-curve pair similarly.
2. **Lock BTC**. Alice funds a 2-of-2 BTC multisig (or MuSig2) output of
   `(A + R_B)`. The tx becomes the **lock tx**. Refund script:
   `OP_CSV t1 -> Alice` after timelock.
3. **Lock XMR**. Bob funds a 2-of-2 Monero output to
   `(B + R_A)` (XMR multisig is constructed via aggregated edDSA key).
   He waits for sufficient XMR confirmations.
4. **Swap pre-image broadcast (BTC)**. Alice signs a BTC redeem tx
   spending the lock to Bob, but provides only her partial signature with
   adaptor point `T = R_B`. Bob, knowing `r_B`, completes the signature
   and broadcasts. Resulting witness reveals `r_B`.
5. **Bob receives BTC**. The BTC redeem confirms. Alice now reads the
   completed Schnorr sig, extracts `r_B`, applies it across curves
   (`r_B * H` is the corresponding ed25519 scalar) and builds a Monero
   spend tx using `(s_A + r_B)` as the spend key — the aggregated
   Monero secret she had with Bob's `r_B'` half.
6. **Alice receives XMR**. She broadcasts the Monero tx and walks away
   with XMR.

### Refund paths

- BTC refund: timelocked back to Alice if Bob never broadcasts.
- XMR refund: timelocked Monero "punish" tx — but Monero has no native
  CLTV. Implementations use a back-and-forth pre-signed sequence with
  on-chain anchors.

## Worked example

Alice has 1 BTC. Bob has 100 XMR. Rate 100 XMR per BTC.

| Step | Chain | Cost | Visible info |
|------|-------|------|--------------|
| Lock BTC | Bitcoin | ~150 vB | a P2WSH or P2TR multisig output |
| Lock XMR | Monero | ~1.5 KB | confidential output, no amount visible |
| BTC redeem | Bitcoin | ~110 vB | completed Schnorr sig (reveals `r_B`) |
| XMR redeem | Monero | ~1.5 KB | confidential, ring-signature-mixed |

A blockchain observer sees only ordinary multisig spend on BTC and an
ordinary spend on XMR. No HTLC, no obvious link.

## Common pitfalls

- **Different chain finalities**: Monero ~2 min, Bitcoin ~10 min. Set
  refund timelocks so that a 2-block Bitcoin reorg cannot make both
  parties able to claim.
- **Punishment via "dishonest abort"**: if Alice broadcasts the BTC
  redeem and then Bob refuses to publish the Monero redeem, Bob can
  retain BTC. The protocol must include a Bitcoin "punishment tx" that
  Bob signs in advance, leveraging the leaked `r_B` to slash him.
- **DLEQ proof correctness**: a buggy DLEQ implementation lets a
  malicious party use different `s` values across curves and steal funds
  on the cheaper side. Use the audited implementation in xmr-btc-swap.
- **Monero multisig fragility**: ed25519 aggregation is more interactive
  than secp256k1; lost messages can leave funds locked. Most production
  implementations use 2-step retry with persistent state.
- **Mempool sniping**: the BTC redeem tx has a small RBF window. Run
  with anchor outputs or CPFP-able fee design so you can bump if
  congestion hits.

## References

- Joel Gugger. "BTC<->XMR atomic swap" 2020.
- comit-network/xmr-btc-swap repo.
- "Discrete Log Equality across curves" — Maxwell, Poelstra 2017.
