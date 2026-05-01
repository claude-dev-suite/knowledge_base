# PayJoin URI Extension - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/uri-schemes`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0078.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/uri-schemes/SKILL.md

## Concept

PayJoin (BIP78) is a collaborative transaction where the receiver contributes one or more inputs alongside the sender's inputs. The result is a transaction that breaks the common-input-ownership heuristic used by chain analysis. The BIP21 URI carries a single new parameter, `pj=`, pointing at the receiver's PayJoin endpoint. The sender's wallet detects the parameter, opens an HTTP(S) session to the endpoint, posts an unsigned PSBT (the "original" or "fallback" tx), and the receiver responds with a modified PSBT that adds inputs and shifts amounts. If the handshake fails, the sender broadcasts the fallback transaction unchanged so funds always move. PayJoin v2 adds an asynchronous relay mode (`pjos=`, `pjes=`) so the sender and receiver no longer need to be online at the same time.

## Walkthrough / mechanics

URI grammar additions:

```
pjparam   = "pj="   1*qchar          # endpoint URL (HTTPS or .onion)
pjosparam = "pjos=" ("0" / "1")      # output substitution allowed
pjesparam = "pjes=" 1*qchar          # ephemeral relay session key (v2)
```

Sender-side flow (BIP78 v1):

1. Parse the BIP21 URI; if `pj=` is present, store the endpoint.
2. Build a "fallback" tx paying the receiver's address for `amount` sats with the sender's normal coin selection.
3. Sign all sender inputs to produce a finalized PSBT (the "original PSBT").
4. POST the original PSBT (base64) to `pj=` with query parameters `v=1`, `additionalfeeoutputindex=`, `maxadditionalfeecontribution=`, `disableoutputsubstitution=`.
5. Receiver responds with a modified PSBT (the "proposal"); receiver-added inputs are unsigned by sender.
6. Sender validates: same outputs to receiver address, total fee within bounds, no extra outputs to unknown scripts, sender-input set unchanged.
7. Sign the new sender-input witnesses (their previous signatures are invalidated because the txid changed).
8. Broadcast the finalized PayJoin tx.
9. On any error before step 8, broadcast the fallback tx.

Receiver-side flow:

1. Listen on `pj=` endpoint.
2. Validate the original PSBT: signed, pays correct address, fee rate sane.
3. Choose receiver inputs whose UTXO values are similar to the sender's (defeats heuristic).
4. Build proposal PSBT, lower fee output proportionally, return to sender.

PayJoin v2 (async / serverless):

- `pjes=` carries a fresh ephemeral X25519 key.
- A relay (often a public OHTTP gateway) holds the encrypted PSBTs.
- Sender drops the original; receiver pulls when next online; receiver drops proposal; sender pulls.

## Worked example

A merchant URI for a 0.005 BTC sale at a coffee shop:

```
bitcoin:bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh?amount=0.005&label=Coffee&pj=https://payjoin.example.com/api/v1/pj
```

After URL parsing, the sender wallet POSTs an "original PSBT" describing a non-PayJoin tx:

```
inputs:  [a1b2...:0  (0.01 BTC, sender)]
outputs: [bc1qxy2... 0.005, bc1q-change... 0.00499 BTC]
fee:     0.00001 BTC (1 vB sat-rate per spec)
```

Receiver returns a proposal:

```
inputs:  [a1b2...:0 (sender), 9f3c...:1 (receiver, 0.004 BTC)]
outputs: [bc1qxy2... 0.009, bc1q-change... 0.00499 BTC]
fee:     0.00001 BTC unchanged
```

The merchant address now appears to receive 0.009 BTC; chain analysis cannot tell which input was the sender's and which was the receiver's. The sender re-signs its input (the txid changed because a new input was added) and broadcasts.

PayJoin v2 URI for an offline-capable async exchange:

```
bitcoin:bc1qxy2...?amount=0.005&pj=https://relay.payjoin.org/&pjos=0&pjes=AKxR9w8...base64-x25519-pubkey...
```

`pjos=0` forbids output substitution (receiver may NOT change the destination address); `pjes=` is the receiver's ephemeral pubkey.

## Common pitfalls

- Failing to broadcast the fallback tx on PayJoin error. The whole point is that funds always move; a wallet that aborts silently leaves the user wondering why payment never landed.
- Trusting the receiver's proposal blindly. The sender must verify the receiver address output amount did not shrink and no unexpected outputs were added.
- Sending more fee than necessary. BIP78 caps `maxadditionalfeecontribution` precisely to stop the receiver from draining sender funds via fee inflation.
- Ignoring `pjos=0`. If the URI sets it, the wallet MUST reject any proposal that changes the receiver output script.
- Posting to a non-HTTPS / non-Tor endpoint. PayJoin v1 mandates HTTPS or `.onion`; cleartext HTTP leaks the original PSBT to passive observers.
- Reusing input ordering. Both wallets should shuffle inputs/outputs to a deterministic random order before signing to avoid leaking who contributed which.

## References

- BIP78 (PayJoin): https://github.com/bitcoin/bips/blob/master/bip-0078.mediawiki
- payjoin.org spec hub: https://payjoin.org
- BIP77 (PayJoin v2 async relay): https://github.com/bitcoin/bips/blob/master/bip-0077.mediawiki
- `payjoin` Rust crate: https://github.com/payjoin/rust-payjoin
- Wasabi 2 PayJoin docs: https://docs.wasabiwallet.io/why-wasabi/PayJoin.html
