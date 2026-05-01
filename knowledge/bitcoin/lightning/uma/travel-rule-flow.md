# UMA Travel Rule Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/uma`.
> Canonical source: https://docs.uma.me/uma-protocol
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/uma/SKILL.md

## Concept

The **Universal Money Address** (UMA) protocol extends Lightning
Address with structured **Travel Rule compliance**: when a sender's
country/regulator requires KYC information to be transmitted alongside
the payment (FATF Travel Rule, FinCEN >$3000, EU TFR), UMA negotiates
the disclosure during the payment handshake. Senders' VASPs (Virtual
Asset Service Providers) communicate over a signed protocol with
receivers' VASPs before the Lightning HTLC is sent.

UMA is built on top of LNURL-pay (LUD-06) and Lightning Address with a
new "lnurlpay request" extension carrying compliance fields.

## Walkthrough / mechanics

### Roles

- **Sender VASP**: holds sender's account, runs UMA-aware wallet.
- **Receiver VASP**: holds receiver's account, exposes UMA endpoint.
- **Sender / receiver**: human users.
- **Compliance metadata**: KYC fields (name, address, account ID).

### UMA address

Standard Lightning Address format with `$` separator:

```
$alice@vasp.io        # equivalent to alice@vasp.io but UMA-aware
```

Resolves to `https://vasp.io/.well-known/lnurlp/alice` (LUD-16) but the
returned LNURL-pay metadata includes UMA flags.

### Three-step handshake

**Step 1: LNURLP request (sender VASP -> receiver VASP)**

```
GET https://vasp.io/.well-known/lnurlp/alice?
  signature=<sender_vasp_sig>
  &nonce=<random>
  &timestamp=<unix>
  &senderUserId=hash(<sender_kyc>)
  &senderUserKyc={"firstName":"Bob",...}
  &travelRuleInfo=<encrypted blob>
  &payerData={...}
  &compliance.utxoCallback=<webhook URL>
```

Receiver VASP verifies sender VASP's signature against its registered
public key (PKI on-chain or off-chain).

**Step 2: LNURLP response (receiver VASP -> sender VASP)**

```json
{
  "callback": "https://vasp.io/uma/payreq/alice",
  "minSendable": 1000,
  "maxSendable": 1000000000,
  "metadata": "[[\"text/plain\",\"$alice@vasp.io\"]]",
  "compliance": {
    "kycStatus": "VERIFIED",
    "isSubjectToTravelRule": true,
    "receiverIdentifier": "$alice@vasp.io",
    "signature": "<receiver_vasp_sig>",
    "signatureNonce": "<random>",
    "signatureTimestamp": 1714000000
  },
  "currencies": [
    {"code":"USD","multiplier":52000.0,"convertible":{"min":1,"max":100000}},
    {"code":"SAT","multiplier":1.0,"convertible":{"min":1,"max":100000000}}
  ]
}
```

`compliance` block confirms receiver VASP's KYC posture.

**Step 3: payreq (sender VASP -> receiver VASP)**

```
POST https://vasp.io/uma/payreq/alice
  amount=<msat>
  payerData={...}
  compliance.utxoCallback=<webhook>
  travelRuleInfo=<encrypted shared payload>
  ...
```

Receiver VASP returns BOLT11 invoice + final compliance status. Sender
VASP pays invoice over Lightning.

### Travel Rule encryption

Sender VASP encrypts the KYC payload to receiver VASP's published
public key (commonly Curve25519 ECIES) so intermediate parties cannot
read PII. Encrypted payload travels HTTPS, not over LN.

### VASP PKI

UMA requires a registry of VASP public keys. Implementations use:

- DNS TXT records (`_uma.vasp.io TXT "key=..."`).
- Bitcoin OP_RETURN-anchored DIDs (planned).
- Centralized directories (early phase).

### Compliance status

Receiver VASP can refuse based on sender KYC: e.g. only accepts payments
from VASPs with verified accounts of >18-year-old users. UMA returns
`compliance.kycStatus = REJECTED` if so.

## Worked example

Bob ($bob@usaVasp) sends $50 USDT-equivalent to Alice ($alice@euVasp).

```
1. Sender VASP signs LNURLP request with Bob's KYC payload (encrypted to
   euVasp pubkey).
2. euVasp verifies VASP signature, decrypts payload, runs sanctions check.
3. euVasp returns compliance block + currencies (offers EUR conversion).
4. Sender VASP requests payreq with chosen amount in USD.
5. euVasp returns BOLT11 invoice for euros at quoted rate.
6. Sender VASP pays invoice over Lightning.
7. Both VASPs log encrypted Travel Rule transcript for compliance audit.
```

## Common pitfalls

- **VASP key compromise**: encrypted KYC payload's confidentiality
  depends on receiver VASP key secrecy. Use HSM-backed keys.
- **PKI bootstrap**: small VASPs may not be discoverable; UMA falls back
  to "self-attested" KYC, with disclosed reduced trust.
- **Currency multiplier drift**: between LNURLP response and payment,
  exchange rate may change; UMA includes timestamp + max age window.
- **Privacy vs. compliance trade-off**: sharing real-name KYC across
  VASPs is by design; users opting for privacy must use non-UMA paths.
- **Latency**: 3-step handshake adds 200-500 ms vs raw LNURL-pay; UX
  must mask the delay.

## References

- UMA documentation (docs.uma.me).
- LUD-06 LNURL-pay specification.
- FATF Recommendation 16 (Travel Rule).
- LUD-16 Lightning Address.
