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

A Cashu **token** is a JSON object:

```json
{
  "token": [{
    "mint": "https://mint.example",
    "proofs": [
      {"id": "00abcd", "amount": 64, "secret": "0x..", "C": "0x.."},
      {"id": "00abcd", "amount": 32, "secret": "0x..", "C": "0x.."}
    ]
  }]
}
```

Each `proof` represents a single power-of-two-denominated note:
- `id`: 16-hex-char keyset id (mint-derived).
- `amount`: 1, 2, 4, ..., 2^n sats.
- `secret`: 32-byte random `x`.
- `C`: blind-signed `C = sk_a * H_to_curve(secret)` from the mint where
  `sk_a` is the per-amount secret key.

### Blind signatures (NUT-00)

Cashu uses BDHKE (Blind Diffie-Hellman Key Exchange):

1. User picks `secret x`, computes `Y = H_to_curve(x)`, blinds with
   random `r`: `B' = Y + r*G`.
2. Sends `B'` to mint. Mint signs: `C' = sk_a * B'`.
3. User unblinds: `C = C' - r*A` where `A = sk_a*G`.
4. To redeem, user reveals `(x, C)`. Mint computes `Y = H_to_curve(x)`,
   checks `C == sk_a * Y`. Marks `x` as spent.

### Core NUT roadmap

| NUT | Purpose |
|-----|---------|
| NUT-00 | Notation, key derivation, proofs |
| NUT-01 | Mint info endpoint |
| NUT-02 | Keysets (rotating amount keys) |
| NUT-03 | Swap operation (split/combine notes) |
| NUT-04 | Mint via Lightning |
| NUT-05 | Melt to Lightning |
| NUT-06 | Mint info v2 |
| NUT-07 | State check (proof spent / pending) |
| NUT-08 | Lightning fee return |
| NUT-09 | Restore proofs from keypair |
| NUT-10 | Spending conditions (P2PK, HTLC) |
| NUT-11 | Pay-to-Public-Key locks |
| NUT-12 | Offline DLEQ proofs (DLEQ-blinded sig verification) |
| NUT-13 | Deterministic secrets (BIP32-style derivation) |
| NUT-14 | HTLC notes |
| NUT-15 | Multi-path payments |

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

- cashubtc/nuts repository (NUTs 00–15).
- "Cashu: A Chaumian Mint over Lightning" — Calle 2022 whitepaper.
- BDHKE: Wagner 2003 reformulation.
