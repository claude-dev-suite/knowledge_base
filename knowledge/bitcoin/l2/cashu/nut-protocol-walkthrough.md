# Cashu NUT Protocol Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/cashu`.
> Canonical source: https://github.com/cashubtc/nuts
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/cashu/SKILL.md

## Concept

Cashu is a single-mint Chaumian e-cash protocol on top of Bitcoin and
Lightning. It is specified as a series of **NUTs** (Notation, Usage,
Terminology — basically Cashu's BIPs). Each NUT is an independent
extension; the core mint speaks NUT-00 to NUT-06, and additional NUTs
add features like P2PK locks (NUT-10/-11), DLEQ proofs (NUT-12), and
multi-mint operations.

## Walkthrough / mechanics

### Token structure

NUT-00 serialises a token as `cashu` + a single `base64_urlsafe`
character naming the version + the payload. As of September 2026 two
versions are defined:

- **V4** (`cashuB`) — CBOR, single-character keys, hex fields carried as
  raw bytes. Single-mint only. The encouraged form.
- **V3** (`cashuA`) — `base64_urlsafe` JSON, can hold proofs from
  several mints. NUT-00 marks it deprecated in favour of V4; wallets
  still decode it.

A `cashu:` URI prefix makes either form clickable on the web. For binary
transports (NFC) NUT-00 defines
`utf8("craw") || utf8("B") || cbor(token_v4_object)`.

The V4 payload, shown in its equivalent JSON form:

```json
{
  "m": "https://mint.example",
  "u": "sat",
  "t": [{
    "i": h'00ffd48b8f5ecf80',
    "p": [
      {"a": 64, "s": "407915bc..", "c": h'02bc9097..'},
      {"a": 32, "s": "fe151093..", "c": h'029e8e50..'}
    ]
  }]
}
```

`m` is the mint URL (trailing slashes stripped), `u` the unit, optional
`d` a memo; inside `t`, `i` is the keyset id and `p` the proofs sharing
it, with the per-proof `id` field omitted. `h''` values are CBOR byte
strings, hex only for readability here. The `i` above is NUT-00's own
example value and is still a **v1** keyset id (version byte `00`), not
the short form of a v2 id.

The deprecated V3 payload keeps the long-form JSON:

```json
{
  "token": [{
    "mint": "https://mint.example",
    "proofs": [
      {"id": "009a1f293253e41e", "amount": 64, "secret": "0x..", "C": "0x.."},
      {"id": "009a1f293253e41e", "amount": 32, "secret": "0x..", "C": "0x.."}
    ]
  }],
  "unit": "sat"
}
```

Each `proof` represents a single power-of-two-denominated note:
- `id` / `i`: keyset id, derived by the mint from its public keys and
  metadata (NUT-02). Keyset id **v2** — the current version as of
  September 2026 — is 33 bytes (66 hex chars) with version byte `01`;
  the deprecated **v1** form is 8 bytes (16 hex chars) with version
  byte `00`. A V4 token MAY instead carry a *short* keyset id, the
  first 8 bytes of the 33-byte id; wallets MUST resolve it to the full
  id and MUST fail parsing if it is ambiguous. Mint endpoints always
  use the full id.
- `amount` / `a`: 1, 2, 4, ..., 2^n sats.
- `secret` / `s`: UTF-8 secret message `x` (NUT-00 recommends a 64-char
  hex string over 32 random bytes to avoid fingerprinting).
- `C` / `c`: blind-signed `C = sk_a * H_to_curve(secret)` from the mint
  where `sk_a` is the per-amount secret key.

### Blind signatures (NUT-00)

Cashu uses BDHKE (Blind Diffie-Hellman Key Exchange):

1. User picks `secret x`, computes `Y = H_to_curve(x)`, blinds with
   random `r`: `B' = Y + r*G`.
2. Sends `B'` to mint. Mint signs: `C' = sk_a * B'`.
3. User unblinds: `C = C' - r*A` where `A = sk_a*G`.
4. To redeem, user reveals `(x, C)`. Mint computes `Y = H_to_curve(x)`,
   checks `C == sk_a * Y`. Marks `x` as spent.

### NUT index (as of September 2026)

**Mandatory** — every wallet and mint MUST implement NUT-00 to NUT-06:

| NUT | Purpose |
|-----|---------|
| NUT-00 | Cryptography and models (BDHKE, proofs, token encoding) |
| NUT-01 | Mint public keys (`GET /v1/keys`) |
| NUT-02 | Keysets and fees (`input_fee_ppk`; keyset id v1/v2) |
| NUT-03 | Swap operation (split/combine notes) |
| NUT-04 | Mint tokens (generic quote flow, method-agnostic) |
| NUT-05 | Melt tokens (generic quote flow, method-agnostic) |
| NUT-06 | Mint info (`GET /v1/info`, `nuts` capability map) |

**Optional** — each advertised under its number in the NUT-06 `nuts`
map. On the cashubtc/nuts tree as of 2026-09-15 the optional range runs
NUT-07 to NUT-30:

| NUT | Purpose |
|-----|---------|
| NUT-07 | Token state check (proof spent / pending) |
| NUT-08 | Overpaid Lightning fee return |
| NUT-09 | Signature restore (recover proofs from seed) |
| NUT-10 | Spending conditions (well-known `secret` format) |
| NUT-11 | Pay-to-Public-Key (P2PK) locks |
| NUT-12 | Offline DLEQ proofs (DLEQ-blinded sig verification) |
| NUT-13 | Deterministic secrets (BIP32-style derivation) |
| NUT-14 | Hashed timelock contracts (HTLC notes) |
| NUT-15 | Partial multi-path payments (MPP) |
| NUT-16 | Animated QR codes |
| NUT-17 | WebSocket subscriptions |
| NUT-18 | Payment requests (`creqA`, CBOR + base64url) |
| NUT-19 | Cached responses (safe request replay) |
| NUT-20 | Signature on mint quote |
| NUT-21 | Clear authentication |
| NUT-22 | Blind authentication |
| NUT-23 | Payment method: BOLT11 |
| NUT-24 | HTTP 402 Payment Required (`X-Cashu` header) |
| NUT-25 | Payment method: BOLT12 |
| NUT-26 | Payment request bech32m encoding (`creqb1...`) |
| NUT-27 | Nostr mint backup |
| NUT-28 | Pay-to-Blinded-Key (P2BK) |
| NUT-29 | Batched minting |
| NUT-30 | Payment method: on-chain |

Note the split that arrived with the method NUTs: NUT-04 and NUT-05
carry the generic mint/melt quote machinery, while the concrete
settlement rails moved out into NUT-23 (BOLT11), NUT-25 (BOLT12) and
NUT-30 (on-chain). A mint advertises the `method`/`unit` pairs it will
quote in the `4` and `5` entries of its `nuts` map.

### Payment requests (NUT-18 / NUT-26)

NUT-18 serialises a payment request as
`"creq" + "A" + base64_urlsafe(CBOR(PaymentRequest))`. NUT-26 adds an
alternative TLV + Bech32m encoding with HRP `creqb` and version
separator `1`; the spec puts it 30–60% smaller than the NUT-18 form and
recommends emitting it uppercase so QR encoders can use alphanumeric
mode. Decoders MUST accept both: a value starting `creqA` is NUT-18,
valid Bech32m with HRP `creqb` is NUT-26.

A NUT-26 request can ride inside a BIP-321 Bitcoin URI as a `creq`
query parameter, so one QR code can offer several rails at once and the
wallet picks whichever it supports:

```
bitcoin:?lightning=lnbc...&creq=CREQB1...
```

NUT-24 reuses the same two encodings in an `X-Cashu` header on an HTTP
402 response.

### Swap (NUT-03)

To split a 64-sat note into 32+16+8+8:

1. Client crafts 4 new blinded outputs `[B'_32, B'_16, B'_8, B'_8]`.
2. Sends `swap` request: input proofs + outputs.
3. Mint verifies inputs (signature + not-spent), marks them spent,
   blind-signs outputs.
4. Returns 4 fresh proofs to client.

Privacy property: the new notes are unlinkable to the input notes due to
the blinding factor.

### Mint (NUT-04)

1. Client requests `mint quote` for `v` sat. Mint returns BOLT11 invoice.
2. Client pays invoice via Lightning.
3. Client requests `mint` with desired blinded outputs summing to `v`.
4. Mint verifies invoice paid, blind-signs outputs.

### Melt (NUT-05)

Reverse of mint: spend proofs to pay an external BOLT11.

1. Client requests `melt quote` providing target BOLT11.
2. Mint quotes fee + total amount required.
3. Client submits proofs covering quote total + (optional) blinded
   change outputs.
4. Mint pays BOLT11 over Lightning, settles change blind-sigs (NUT-08
   excess fee return).

## Worked example

Alice mints 100 sat:

```
1. mint_quote(100) -> {quote_id: "abc", invoice: "lnbc..."}
2. Alice pays Lightning invoice
3. mint(quote_id="abc", outputs=[B'_64, B'_32, B'_4])
   -> [C_64, C_32, C_4] blind-signed
4. Alice has proofs for 64+32+4 = 100 sat
```

Send 50 sat to Bob:

```
1. swap(inputs=[proof_64], outputs=[B'_50? no]; need binary]: [B'_32, B'_16, B'_2, B'_8, B'_2])
2. Take 50 sat (32+16+2) of new proofs
3. Send token JSON to Bob
4. Bob runs NUT-07 check or swaps immediately to claim
```

## Common pitfalls

- **Mint downtime = note unredeemable**: e-cash is mint-bound. Cross-mint
  exchange via Lightning melt+mint cycle.
- **Power-of-two denominations** mean change isn't always free; for small
  amounts you may need swap rounds. Some implementations use mixed
  denominations (NUT-13 deterministic only for 2^n).
- **Replay across mints**: each mint has its own keyset; proofs are
  bound to keyset id. Cross-mint replay impossible.
- **Privacy from mint**: mint sees blinded outputs (cannot link to spent
  inputs by the blinding property), but mint sees IP / TLS metadata. Use
  Tor.
- **Token lifetime**: long-lived tokens accumulate metadata risk;
  consider regular `swap` cycles to refresh.

## References

- cashubtc/nuts repository (NUT-00 to NUT-30 as of September 2026).
- "Cashu: A Chaumian Mint over Lightning" — Calle 2022 whitepaper.
- BDHKE: Wagner 2003 reformulation.
