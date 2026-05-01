# Atomic Swap with Adaptor Signatures - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/adaptor-sigs`.
> Canonical source: Poelstra "Scriptless Scripts" + Bitcoin-Litecoin atomic
>                   swap reference (Decred / Lloyd Fournier writeups)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/adaptor-sigs/SKILL.md

## Concept

Two parties want to swap coins on different chains atomically: either both
swaps complete or neither does. The classical solution is a Hash-Time-Locked
Contract (HTLC) using a SHA256 preimage. Adaptor signatures replace the
preimage with a discrete-log secret — both transactions become indistinguishable
from cooperative cosigns on chain, and the swap leaks no metadata. This article
walks Alice (BTC) and Bob (LTC) through the four-message protocol, including
the refund branch, with the exact group operations at each step.

## Walkthrough / mechanics

### Setup and key exchange

```
Alice:  d_A, P_A = d_A G    (BTC side)
Bob:    d_B, P_B = d_B G    (LTC side)

Funding outputs (off-chain agreed):
  BTC: 2-of-2 multisig with keys (P_A_btc, P_B_btc) - aggregated via MuSig2
       agg key Q_btc = MuSig2.KeyAgg([P_A_btc, P_B_btc])
  LTC: 2-of-2 multisig with keys (P_A_ltc, P_B_ltc) -> Q_ltc

Each party also constructs a CSV-locked refund tx pre-signed and stored
locally for the disaster path.
```

### Step 1 - Alice picks the secret

```
Alice samples t <- {1..n-1}
Alice computes T = t * G

t is the "shared secret" of the swap. Alice keeps t secret. T is broadcast
to Bob.
```

### Step 2 - Build pre-signatures

Both parties build two transactions:

```
Tx_BTC: spends 2-of-2 BTC funding to Bob's P_B_btc payout address
Tx_LTC: spends 2-of-2 LTC funding to Alice's P_A_ltc payout address
```

Each party uses MuSig2 with adaptor variant on each tx:

```
Alice computes:
  pre_sig_BTC = MuSig2.PreSign(d_A_btc, sighash_btc, T)
  // Alice's partial sig over Tx_BTC, offset by T
  Sends pre_sig_BTC partial to Bob.

Bob computes:
  pre_sig_LTC = MuSig2.PreSign(d_B_ltc, sighash_ltc, T)
  // Bob's partial sig over Tx_LTC, offset by T
  Sends pre_sig_LTC partial to Alice.

Each side also exchanges its non-adaptor partial sig for its OWN payout side:
  Alice -> Bob: full partial sig for Tx_LTC (her payout)
  Bob -> Alice: full partial sig for Tx_BTC (his payout)
```

After this exchange, both parties have:

- **A pre-signed Tx_LTC** that Alice can complete by adding `t` (only Alice
  knows it).
- **A pre-signed Tx_BTC** that Bob can complete by adding `t` once it leaks.

### Step 3 - PreVerify

Each party verifies the other's pre-sig before depositing funds:

```
Alice: PreVerify(pre_sig_LTC, T, Bob's pubkey, sighash_ltc) - must pass
Bob:   PreVerify(pre_sig_BTC, T, Alice's pubkey, sighash_btc) - must pass
```

If either fails, abort. No funds at risk yet.

### Step 4 - Fund the multisigs

Both parties broadcast the funding txs to their respective chains. Wait for
sufficient confirmations on both. If only one funds, the other refunds at
timeout.

### Step 5 - Alice claims LTC, revealing t

```
Alice runs Adapt(pre_sig_LTC, t):
  s_LTC = pre_sig_LTC.s' + t mod n
  Tx_LTC.witness = aggregate_sig(R_LTC, s_LTC)

Alice broadcasts Tx_LTC.
```

The LTC chain confirms Tx_LTC. Bob is watching the chain for it.

### Step 6 - Bob extracts t and claims BTC

```
Bob observes Tx_LTC on chain. He knows pre_sig_LTC's s' value.
Bob runs Extract(pre_sig_LTC, observed_sig):
  t = s_LTC - pre_sig_LTC.s' mod n

Bob runs Adapt(pre_sig_BTC, t):
  s_BTC = pre_sig_BTC.s' + t mod n
  Tx_BTC.witness = aggregate_sig(R_BTC, s_BTC)

Bob broadcasts Tx_BTC.
```

Both swaps complete. The on-chain footprint is two ordinary cooperative
2-of-2 spends.

### Refund path

A timelock-paranoid setup ensures **Alice's refund timelock is shorter than
Bob's**. The asymmetry guarantees Alice refunds first if Bob disappears, and
Bob refunds first if Alice disappears, without enabling griefing.

```
BTC funding timelock: T_BTC = now + 48 h
LTC funding timelock: T_LTC = now + 24 h    (shorter)
```

If after 24 h Alice has not claimed LTC (e.g., Bob's pre_sig was a denial-
of-service ploy), the LTC refund unlocks. Then at 48 h the BTC refund unlocks.
Bob cannot grief Alice by waiting because his only path to BTC is via the
adaptor sig, which requires `t` reveal — reveal happens only when Alice
claims LTC.

### Why this is atomic

```
Property: t leaks <=> Tx_LTC confirms.

Forward direction: Alice's claim publishes (R, s) where s = s' + t.
  Anyone can compute t = s - s'. Bob can.

Reverse direction: t cannot leak any other way. Alice keeps t secret;
  Bob's only way to get t is to observe a sig that uses it. The pre_sig_BTC
  *uses* the same T; if Alice publishes it (impossible without t), it would
  reveal t too. So Alice's claim is the only leak path.

Therefore: if Alice claims, Bob can claim. If Alice doesn't claim, neither
gets paid (and refund kicks in).
```

## Worked example

Toy group with `n = 23`, single-key pretend (no MuSig2, just plain Schnorr
for clarity).

```
Alice on BTC: d_A = 7,  P_A = 7G  (pretend even-y)
Bob on LTC:   d_B = 13, P_B = 13G

Alice picks t = 5, T = 5G.

Bob pre-signs Tx_LTC (paying Alice from LTC funding):
  k_B = 9
  R0_B = 9G
  R_B  = R0_B + T = 9G + 5G = 14G
  e_B  = H(xbytes(14G) || xbytes(13G) || sighash_ltc) mod 23, say e_B = 8
  s'_B = (k_B + e_B * d_B) mod 23
       = (9 + 8 * 13) mod 23 = 113 mod 23 = 113 - 4*23 = 21

pre_sig_LTC = (R0=9G, s'=21)

Alice PreVerify:
  21 G ?= 9G + 8 * 13G = 9G + 104G = 113G mod 23 = 21G    OK

Alice Adapt with t=5:
  s_LTC = (21 + 5) mod 23 = 3
  publishes final_sig = (R = 14G, s = 3)

BIP340 verify on Tx_LTC:
  3 G ?= 14G + 8 * 13G = 14G + 104G = 118G mod 23 = 3G    OK

Bob Extract:
  t = (3 - 21) mod 23 = -18 mod 23 = 5    OK

Bob Adapt pre_sig_BTC with t=5 -> publishes Tx_BTC. Done.
```

## Common pitfalls

- **Refund timelocks symmetric or wrong-direction.** Must be `T_LTC < T_BTC`
  (Alice's *receive* leg refund earlier than Bob's *send* leg). Reversing
  this allows Alice to refund LTC while still claiming BTC.
- **Insufficient confirmations before claiming.** A reorg on the LTC chain
  invalidates Alice's claim, but `t` is already public — Bob walks with both
  sides if BTC is claimed before LTC is final.
- **Reusing T across swaps.** If `T = t G` is the same in two unrelated
  swaps, leaking `t` from one breaks the other. Always sample fresh `t` per
  swap.
- **Forgetting Bob has to *watch* the LTC chain.** If Bob is offline, he
  misses the `t` leak window. Mitigation: anyone who sees Tx_LTC can compute
  `t` and finalize Tx_BTC for Bob (with appropriate authorization) — the
  swap is **third-party-completable**.
- **Pre-sig as proof of intent.** A pre-sig is *not* an authorization to
  spend — it only becomes spendable on chain when adapted with `t`. Don't
  treat receipt of a pre-sig as a guarantee.

## References

- Lloyd Fournier, "One-time verifiable random functions and adaptor sigs":
  https://github.com/LLFourn/one-time-VES/blob/master/main.pdf
- Decred atomic swap (predates adaptor): https://github.com/decred/atomicswap
- Aumayr et al. "Generalized Channels": https://eprint.iacr.org/2020/476
- Bitcoin-Litecoin adaptor swap demo (rust-bitcoin):
  https://github.com/comit-network/xmr-btc-swap (Monero variant, similar shape)
