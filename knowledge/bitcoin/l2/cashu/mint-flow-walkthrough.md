# Cashu Mint Flow Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/cashu`.
> Canonical source: https://github.com/cashubtc/nuts/blob/main/04.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/cashu/SKILL.md

## Concept

The Cashu **mint flow** (NUT-04) converts Lightning sats into blinded
e-cash notes. It's the entry point into the protocol: a user pays a
BOLT11 invoice into the mint, and in return receives Chaumian-blinded
proofs spendable for goods, swaps, or eventual melt back to Lightning.

The flow is two-step (quote -> mint) so that the mint can detect a paid
invoice independently of the user's request and cap exposure if the
user disconnects.

## Walkthrough / mechanics

### Endpoints (NUT-04)

```
POST /v1/mint/quote/bolt11
POST /v1/mint/bolt11
GET  /v1/mint/quote/bolt11/{quote_id}
```

### Step 1: Quote request

Client sends:

```json
{ "amount": 1000, "unit": "sat" }
```

Mint replies:

```json
{
  "quote": "abc-uuid",
  "request": "lnbc10n1...",   // BOLT11 invoice
  "paid": false,
  "expiry": 1714000000
}
```

### Step 2: Pay invoice

Client pays the BOLT11 over Lightning via any wallet. Mint's CLN/LND
node receives the payment; mint flips `paid: true` once invoice is
settled (with sufficient confidence — typically 1 confirmation for
on-chain or HTLC settlement for LN).

### Step 3: Mint request

Client constructs blinded outputs `B'_i` summing to `1000` sats using
power-of-two denominations:

```
1000 = 512 + 256 + 128 + 64 + 32 + 8
```

Sends:

```json
{
  "quote": "abc-uuid",
  "outputs": [
    {"amount": 512, "id": "00abcd...", "B_": "02f4..."},
    {"amount": 256, "id": "00abcd...", "B_": "0257..."},
    ...
  ]
}
```

Mint verifies:

1. `quote_id` exists and is paid.
2. Sum of `outputs.amount` matches quote amount.
3. Each `id` matches an active keyset.
4. No `B_` repeats (replay protection).

If valid, mint computes `C'_i = sk_{amount_i} * B'_i` and returns:

```json
{
  "signatures": [
    {"amount": 512, "id": "00abcd...", "C_": "03ef..."},
    ...
  ]
}
```

### Step 4: Unblind

Client unblinds: `C_i = C'_i - r_i * A_{amount_i}` where `A_{amount_i} = sk_{amount_i} * G` (mint's public amount key, fetched from `/v1/keysets`).

Now client has proofs `(secret_i, C_i, amount_i, id)` for each note.

### NUT-12 DLEQ proof

Optional: mint can also return a DLEQ proof that
`log_G(A) == log_{B_}(C_)`. This lets the client verify off-line that
the mint did not use a different secret key for one note (which would
let mint mark some notes inflated).

### NUT-13 deterministic secrets

Instead of random `r` and `secret`, derive both from a BIP32-style
keypath: `secret_i = HMAC(seed, "secret/i")`, `r_i = HMAC(seed, "blind/i")`.
This lets the wallet **restore notes from seed** by re-deriving and
asking mint via NUT-09 which secrets are still unspent.

## Worked example

Alice mints 1000 sats:

```
POST /v1/mint/quote/bolt11   {"amount": 1000, "unit": "sat"}
-> {"quote": "abc", "request": "lnbc10n1...", "expiry": 1714000300}

[Alice pays Lightning invoice, takes 0.4s]

POST /v1/mint/bolt11   {"quote": "abc", "outputs": [...8 blinded outputs...]}
-> {"signatures": [...8 blind-signed C_'s...]}

[Alice unblinds locally]

Alice now holds 8 proofs: [512, 256, 128, 64, 32, 8] sums to 1000.
```

Total round-trip: ~1-2s on a healthy mint with active LN channel.

## Common pitfalls

- **Quote expiry**: typical 10 min. If user pays invoice past expiry,
  mint must still credit (BOLT11 settled) but UX must reissue mint
  request before expiry.
- **Output amount sum mismatch**: mint rejects with `400 amount mismatch`.
  Client must total exactly.
- **Keyset rotation mid-mint**: if mint rotates keys between quote and
  mint, the mint may reject outputs targeting the old keyset. Client
  must refetch `/v1/keysets` and use current id.
- **Network failure between mint receiving invoice payment and mint
  returning signatures**: client retries `/v1/mint/bolt11` with same
  outputs. Mint's idempotency keyed on `quote_id`.
- **Double-mint attempts**: mint MUST only sign once per quote; later
  attempts return same signatures or error.
- **Privacy**: mint links the IP/HTTP request to BOLT11 invoice paid
  unless client uses Tor + onion address.

## References

- cashubtc/nuts NUT-04, NUT-09, NUT-12, NUT-13.
- BDHKE construction (NUT-00).
- nutshell (Cashu reference Python implementation).
