# BIP352 Silent Payments Derivation Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/silent-payments`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0352.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/silent-payments/SKILL.md

## Concept

Silent Payments (BIP352) let a recipient publish **one static address** off
chain (`sp1q...`) that the sender uses to derive a **fresh, unlinkable P2TR
output** for every payment. No interactivity, no notification transactions,
no on-chain reuse. Two ECDH-style scalars are used: `b_scan` to scan, and
`b_spend` to spend.

The address is `sp1q || ser_P(B_scan) || ser_P(B_spend)` bech32m-encoded.

## Walkthrough / mechanics

Notation: `G` is the secp256k1 generator. `a` is a sender private key,
`A = a * G`. `B_scan = b_scan * G`, `B_spend = b_spend * G`.

### Sender derives output

1. Pick the inputs to spend. Compute `a_sum = sum of input private keys`
   for **eligible inputs** (P2WPKH, P2TR with even key, P2SH-P2WPKH; **never**
   P2PKH from coinbase, **never** unbroadcast keys). Define `A_sum = a_sum * G`.
2. Compute `input_hash = hash_BIP0352("Inputs" || smallest_outpoint_lex || A_sum)`.
3. Compute shared secret `ecdh = (input_hash * a_sum) * B_scan`.
4. For payment `k = 0, 1, 2, ...` to this recipient in this transaction:
   `t_k = hash_BIP0352("SharedSecret" || ser_P(ecdh) || ser_uint32(k))`.
   Output point `P_k = B_spend + t_k * G`. Use `x(P_k)` as the P2TR output
   (BIP341 with no script path).

### Receiver scans

For each block:
1. Compute the same `A_sum` and `input_hash` from the transaction's inputs.
2. Compute `ecdh' = b_scan * (input_hash * A_sum)` (note: receiver does this
   without holding `a_sum`; it has `b_scan` and reads `A_sum` from chain).
3. For `k = 0, 1, ...`: derive `P_k = B_spend + t_k * G`. Scan the
   transaction outputs for an x-only key matching `x(P_k)`.
4. On match, the spending key is `b_spend + t_k mod n`. Save the UTXO.

The receiver's `b_scan` may live on a hot wallet; `b_spend` can stay cold.

## Worked example

Address (testnet, abbreviated):
```
tsp1qqgst...   B_scan = 02f4..., B_spend = 03ae...
```

Sender input: a P2WPKH UTXO with private key `a = 0x1a...`, outpoint
`<txid>:0`. There is one recipient, `k=0`.

```
A_sum         = a*G                          = 0297ab...
input_hash    = TaggedHash("BIP0352/Inputs", outpoint || A_sum) = 0x4f3c...
ecdh          = (input_hash*a) * B_scan      = 03c1d2...
t_0           = TaggedHash("BIP0352/SharedSecret", ecdh || 0x00000000) = 0x9b7e...
P_0           = B_spend + t_0 * G            = 02e6f1...
P2TR output   = x(P_0)                       = e6f1c5...
```

The transaction has one output `OP_1 OP_PUSHBYTES_32 e6f1c5...`. To anyone
without `b_scan` it looks like an unrelated taproot UTXO.

## Common pitfalls

- **Mixing eligible / ineligible inputs**: only certain script types feed
  `a_sum` (BIP352 §"Inputs for shared secret derivation"). Including a
  P2PKH input flips behaviour and breaks the receiver's scan.
- **Unspent change**: the sender's change output is **not** silent; it is a
  normal output. Reusing the change address is a privacy leak.
- **Even/odd y-coordinate**: when summing private keys, negate any whose
  corresponding public key has an odd y (taproot inputs are x-only).
- **Coinbase-spending with maturity 0**: BIP352 forbids coinbase outputs as
  inputs; chain reorg could rewrite the outpoint and invalidate scans.
- **Multiple recipients in one tx** require distinct `k` indexes; senders
  MUST not reuse `k` even across separate recipients with the same SP
  address (that would only happen with payment "labels"; see BIP352 §labels).

## References

- BIP352 — Silent Payments specification.
- secp256k1 reference implementation `silentpayments` module.
- "Silent Payments" — Ruben Somsen 2022 explainer.
