# BOLT11 Decode and Encode - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/bolt11-decoder`.
> Canonical source: https://github.com/bitcoinjs/bolt11
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/bolt11-decoder/SKILL.md

## Concept

A BOLT11 invoice is a bech32-encoded string (`lnbc100u1p...`) whose
human-readable part encodes amount + network and whose data part
contains a 7-byte timestamp followed by TLV-style "tagged fields"
(payment hash, description, expiry, route hints, payee node id,
features, payment secret) and a Schnorr-over-ECDSA recovery
signature. Decoding requires bech32, varint length parsing per tag,
and signature recovery to derive the payee node key (unless the `n`
tag is present, which contains it explicitly).

The `bolt11` JavaScript package is the most commonly used decoder/encoder
in the JS ecosystem; equivalents exist in Python (`bolt11`),
Rust (`lightning-invoice` from LDK), and Go (`zpay32` from LND or
`bolt11` from various LSP repos).

## API walkthrough

```js
const bolt11 = require("bolt11");

const decoded = bolt11.decode("lnbc100u1p3xnhl2pp5j7u5l3...");
console.log({
    network: decoded.network.bech32,        // "bc" | "tb" | "bcrt" | "sb"
    millisatoshis: decoded.millisatoshis,   // string, e.g. "10000000"
    timestamp: decoded.timestamp,           // unix sec
    timeExpireDate: decoded.timeExpireDate,
    payeeNodeKey: decoded.payeeNodeKey,
    paymentRequest: decoded.paymentRequest,
});

for (const tag of decoded.tags) {
    console.log(tag.tagName, tag.data);
    // payment_hash, description, expire_time, payment_secret,
    // routing_info, min_final_cltv_expiry, feature_bits, ...
}
```

## Worked example: decode, mutate, re-encode

A common LSP pattern: receive an invoice from a wallet, append a route
hint to a private channel, and return the modified invoice. Re-encoding
requires re-signing with the payee node key, so this works only if your
node is the payee.

```js
const bolt11 = require("bolt11");

function addRouteHint(originalReq, privKeyHex, hint) {
    const decoded = bolt11.decode(originalReq);
    const payeeTag = { tagName: "payee_node_key",
                       data: decoded.payeeNodeKey };
    // Ensure the n tag is present so decoders don't need recovery
    if (!decoded.tags.find(t => t.tagName === "payee_node_key"))
        decoded.tags.push(payeeTag);

    decoded.tags.push({
        tagName: "routing_info",
        data: [{
            pubkey: hint.pubkey,
            short_channel_id: hint.scid,
            fee_base_msat: 1_000,
            fee_proportional_millionths: 1,
            cltv_expiry_delta: 40,
        }],
    });

    const reencoded = bolt11.encode(decoded);
    const signed = bolt11.sign(reencoded, privKeyHex);
    return signed.paymentRequest;
}
```

## Python and Rust quick refs

```python
# pip install bolt11
from bolt11 import decode
inv = decode("lnbc100u1p...")
print(inv.amount_msat, inv.payment_hash, inv.payee)
```

```rust
// lightning-invoice = "0.32"
use lightning_invoice::Bolt11Invoice;
use std::str::FromStr;

let inv = Bolt11Invoice::from_str("lnbc100u1p...")?;
println!("amount_msat: {:?}", inv.amount_milli_satoshis());
println!("payee:       {}",   inv.payee_pub_key().expect("present"));
println!("payment_hash:{}",   inv.payment_hash());
```

## Common pitfalls

- Network mismatch: `lnbc...` (mainnet), `lntb...` (testnet),
  `lnbcrt...` (regtest), `lnsb...` (signet). A decoder that ignores
  the HRP will decode wrong amounts (mainnet `bc` vs `bcrt` differ).
- `min_final_cltv_expiry`: if absent, BOLT11 specifies a default of 18.
  Some libraries return `null`; treat that as 18, not zero.
- The 0-amount invoice (`lnbc1p...`) is valid -- it lets payer choose
  the amount. Decoders return `null` / `None`; handle this branch.
- Padding in bech32: spec allows up to 4 zero bits of trailing padding;
  some old encoders emit non-canonical padding. Strict decoders reject;
  lenient ones accept.
- Re-encoding requires re-signing: you cannot mutate fields of a
  decoded invoice and ship it without the payee's private key.

## References

- Spec: https://github.com/lightning/bolts/blob/master/11-payment-encoding.md
- bolt11 (JS): https://github.com/bitcoinjs/bolt11
- bolt11 (Python): https://github.com/fiatjaf/bolt11
- lightning-invoice (Rust): https://docs.rs/lightning-invoice
- Companion: [../../lightning/bolts/SKILL.md](../../lightning/bolts/SKILL.md), [../../lightning/bolt12/SKILL.md](../../lightning/bolt12/SKILL.md)
