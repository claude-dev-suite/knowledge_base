# Boltz Protocol Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/submarine-swaps`.
> Canonical source: https://docs.boltz.exchange/api
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/submarine-swaps/SKILL.md

## Concept

Boltz is a non-custodial submarine swap exchange supporting BTC <->
Lightning, Liquid, and other chains. Like Lightning Labs Loop, it
uses HTLCs across two layers; unlike Loop, it has multiple swap
directions (BTC mainchain, Liquid, even reverse swaps with delayed
preimage release for privacy).

This article covers the **submarine swap** (Lightning -> BTC on chain)
flow used by Boltz.

## Walkthrough / mechanics

### Roles

- **User**: holds Lightning balance, wants on-chain BTC.
- **Boltz**: provides on-chain liquidity, charges fee.

### Swap types

| Type | LN side | On-chain side | Use case |
|------|---------|---------------|----------|
| Submarine | LN payment | BTC receive | Loop out |
| Reverse Submarine | LN receive | BTC send | Loop in (drain on-chain) |
| Cross-chain | LN BTC | Liquid LBTC | Bridge to Liquid |

### Submarine swap flow

1. User requests swap from Boltz API:
   ```
   POST /createswap
     {type: "submarine", pair_id: "BTC/BTC", invoice: "lnbc...",
      refund_public_key: ..., preimage_hash: H}
   ```
2. Boltz responds with:
   - On-chain HTLC script (BTC P2WSH or P2TR).
   - On-chain address `A_htlc`.
   - Timeout block height.
   - Boltz's pubkey for the success path.
3. User funds `A_htlc` with the swap amount.
4. Boltz, on detecting funded HTLC at desired confs, pays the user's
   Lightning invoice. The invoice's payment_hash equals the on-chain
   HTLC's hash.
5. The Lightning HTLC succeeds, revealing preimage `R`.
6. Boltz uses `R` to spend the on-chain HTLC to its own wallet.

### HTLC script (Boltz Taproot variant)

```
P2TR (
  internal_key = Boltz_pk + User_pk MuSig2 (cooperative path),
  scripts:
    leaf 1 (claim): SHA256 H EQUALVERIFY <Boltz_pk> CHECKSIG
    leaf 2 (refund): <timeout> CHECKLOCKTIMEVERIFY DROP <User_refund_pk> CHECKSIG
)
```

Cooperative path: if both parties cooperate (Boltz didn't fail), they
spend via key-path which is a single Schnorr sig — looks like ordinary
spend on chain.

Non-cooperative paths revealed only on failure.

### Refund

If Boltz fails to pay LN invoice within timeout:

1. User waits for `timeout` block height.
2. User signs and broadcasts a refund tx via leaf 2:
   `<sig from User_refund_pk> + <leaf 2 script> + <control block>`.
3. Tx returns the swap amount to user (less mining fee).

### Reverse submarine (Lightning <- BTC on chain)

Direction reversed:

1. User generates preimage `R`, hashes to `H`.
2. Requests reverse swap; provides `H` and on-chain refund key.
3. Boltz creates LN invoice for `H` and on-chain HTLC funded by Boltz.
4. User pays Boltz's LN invoice.
5. Boltz reveals nothing yet; user finds `R` (only Boltz knows it!) by...
   Actually, in reverse swaps, **user** generates `R`, so user already has
   it.
6. User reveals `R` on chain to claim Boltz's funded BTC HTLC.
7. Boltz uses `R` (from on-chain reveal) to claim user's LN payment.

## Worked example

Bob has 50,000 LN sat, wants BTC.

```
1. POST /createswap {invoice: <lnbc invoice for 50000 sat>, type: "submarine"}
   -> {address: "bc1q...A_htlc", timeout: 800500, ...}

2. Bob funds bc1q... with 50,500 sat (covers 0.5 % fee).

3. Boltz waits 1 conf, pays Bob's LN invoice. Receives preimage R.

4. Boltz broadcasts claim tx:
   Input: HTLC UTXO
   Witness: <Boltz_sig> + R + <claim_script> + <control_block>
   Output: 50000 sat - mining_fee -> Boltz wallet.
```

If Boltz fails to pay (offline, route failure):

```
After timeout block 800500:
Bob broadcasts refund:
   Input: HTLC UTXO
   Witness: <Bob_sig> + <refund_script_with_CLTV> + <control_block>
   Output: 50500 sat - mining_fee -> Bob's wallet.
```

## Common pitfalls

- **Timeout selection**: must be long enough for confs + Lightning
  routing time + Boltz response time. Too short -> false refunds; too
  long -> users wait for refund needlessly.
- **Cooperative close key path**: if Boltz disappears mid-swap, user
  must use leaf 2 script path. Boltz UI normally guides through this.
- **Lightning invoice expiry**: must exceed swap timeout; otherwise
  invoice expires before Boltz can pay.
- **Fee accuracy**: at quote time, Boltz reserves a fee buffer;
  actual on-chain fee may exceed if mempool spikes. Boltz may delay
  payout.
- **MEV / preimage reveal**: in submarine direction, preimage is on
  chain at claim time; a third party can read it but has nothing to
  do (LN already settled). Reverse swap leaks preimage similarly.

## References

- Boltz API documentation (docs.boltz.exchange).
- Submarine Swaps protocol — Bosworth 2018.
- BOLT-03 (HTLC scripts).
