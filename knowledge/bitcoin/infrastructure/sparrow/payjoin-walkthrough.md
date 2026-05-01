# Sparrow PayJoin (BIP78) Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/sparrow`.
> Canonical source: https://sparrowwallet.com/docs/payjoin.html
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/sparrow/SKILL.md

## Concept

PayJoin (BIP78) is a collaborative coinjoin between sender and receiver
that breaks the common-input-ownership heuristic chain analysts rely on.
Both parties contribute inputs to the transaction; from the outside it
looks like an ordinary 2-input/2-output payment, but the receiver actually
got *more* than the displayed amount and the sender's UTXO topology has
been mixed. Sparrow implements the BIP78 sender-side flow against a
PayJoin endpoint URL (typically advertised via BIP21 `pj=...` parameter).
This article walks the receiver setup using BTCPay Server, then a sender
flow from Sparrow.

## Walkthrough / mechanics

Receiver side - BTCPay Server already implements the BIP78 endpoint when
"PayJoin" is enabled per store:

```
BTCPay UI -> Store Settings -> General -> "Payjoin"
Check: "Enable PayJoin"
Save
```

Now any invoice's BIP21 includes a `pj=` parameter:

```
bitcoin:bc1q...?amount=0.005&pj=https://btcpay.example.com/BTC/pj
```

Sparrow sender flow:

1. Receive the BIP21 URI (QR or pasted).
2. Send tab -> paste URI; Sparrow detects `pj=` and shows a yellow
   "PayJoin available" badge.
3. Build the transaction normally.
4. Click "Send" -> Sparrow signs locally, then POSTs the signed PSBT to
   the receiver's `pj=` URL.
5. Receiver returns a *modified* PSBT with their own input added and
   their output bumped.
6. Sparrow verifies the modification matches BIP78 rules:
   - Same recipient amount or higher (never lower).
   - Sender's inputs and change preserved.
   - Sender's signature still valid on their inputs.
7. Sparrow re-signs only its inputs (the new ones added by the receiver
   require receiver's signatures, already attached).
8. Finalize + broadcast.

Network path:

```
Sparrow ------(POST PSBT_v0)----> https://btcpay.example.com/BTC/pj
        <-----(PSBT_v0+receiver inputs)-----
        ------(broadcast finalized tx via Bitcoin Core / Electrum)----> network
```

If the receiver endpoint is offline or returns 404, Sparrow falls back
to a normal payment automatically (default behavior; configurable).

## Worked example

End-to-end on signet for safe practice:

```bash
# 1. Receiver: spin up BTCPay-signet
docker run -d --name btcpay-signet \
  -p 8443:443 -e BTCPAY_NETWORK=signet btcpayserver/btcpayserver:latest

# 2. Configure Store, enable PayJoin, get an invoice
INV=$(curl -s -X POST https://btcpay.local/api/v1/stores/<id>/invoices \
  -H "Authorization: token $TOKEN" \
  -d '{"amount":"0.005","currency":"BTC"}')
echo "$INV" | jq -r .checkoutLink
# https://btcpay.local/i/abc... -> page shows BIP21 with pj=...
```

Sender side from Sparrow with two UTXOs of 0.003 and 0.004 BTC:

```
Sparrow Send tab:
  Recipient: bc1q...receiver
  Amount: 0.005
  PayJoin URL (auto-detected): https://btcpay.local/BTC/pj
  Fee: 5 sat/vB
Click "Create Transaction"
Click "Sign"
Click "Send"
```

Inspect the broadcast tx in the explorer:

```
inputs:
  vin[0]: 0.004 BTC (sender)
  vin[1]: 0.003 BTC (sender)
  vin[2]: 0.012 BTC (receiver, added by PayJoin)
outputs:
  vout[0]: 0.017 BTC -> receiver address  (= 0.005 + 0.012)
  vout[1]: 0.001985 BTC -> sender change
```

The displayed payment was 0.005 BTC but the receiver vin contributed
0.012, so the visible "payment amount" looks like 0.017 - which it is
not, from the user's perspective. This is the privacy-breaking
ambiguity that makes PayJoin effective.

| Wallet | Sender support | Receiver support |
|--------|----------------|------------------|
| Sparrow | yes | manual via BTCPay |
| BlueWallet | yes | no |
| Wasabi 2 | yes | no |
| BTCPay Server | indirect | yes |
| Joinmarket | yes | yes (Pay-to-Endpoint) |
| LND on-chain | no (use external) | no |

## Common pitfalls

- HTTPS required - BIP78 mandates HTTPS for the `pj=` endpoint; HTTP-only
  endpoints are rejected by Sparrow.
- Mismatched receiver amount - if the receiver returns a PSBT where their
  output decreased below the original amount, Sparrow refuses the
  modification (anti-skimming guard).
- Endpoint timeout - default 30s; under Tor with a slow circuit, the
  request may exceed it and Sparrow falls back to non-PayJoin. Increase
  Settings -> Server -> Timeout for Tor users.
- Single-input wallets - PayJoin shines when sender has multiple UTXOs;
  single-UTXO senders gain less privacy because one of the inputs is
  unambiguously theirs.
- Receiver not running BIP78 - many self-custodial wallets advertise
  static addresses without `pj=`; the privacy benefit is one-sided.
- Re-broadcasting the original PSBT - if the user is impatient and falls
  back to non-PayJoin, the receiver's original endpoint may still attempt
  to push a PayJoin variant; ensure only one tx hits the mempool per
  payment.

## References

- BIP78 PayJoin: https://github.com/bitcoin/bips/blob/master/bip-0078.mediawiki
- Sparrow docs: https://sparrowwallet.com/docs/payjoin.html
- BTCPay PayJoin: https://docs.btcpayserver.org/Payjoin/
- Reference impl: https://github.com/payjoin/rust-payjoin
