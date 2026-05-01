# AMP Shamir Derivation - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/amp-mpp`.
> Canonical source: https://github.com/lightning/bolts/pull/658 (AMP draft)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/amp-mpp/SKILL.md

## Concept

AMP (Atomic Multi-Path) is an alternative to basic MPP that uses
**Shamir Secret Sharing** to make each part of a multi-path payment use
its own unique payment hash, while still requiring all parts to arrive
before the recipient can reconstruct the secret. This achieves:

- Stateless invoices (one invoice can pay multiple amounts).
- No correlation between parts (each part has unique payment_hash).
- Non-reusable payment proof per AMP payment.

AMP is implemented in LND but never standardized in BOLTs (basic_mpp
became the de-facto standard).

## Walkthrough / mechanics

### Shamir secret sharing primer

Given secret `s` and threshold `(K, N)`:

- Build polynomial `f(x) = s + a_1*x + a_2*x^2 + ... + a_{K-1}*x^{K-1}`
  with random coefficients.
- Generate N shares: `(i, f(i))` for `i = 1..N`.
- Any K shares reconstruct `s` via Lagrange interpolation.
- Fewer than K shares reveal nothing.

### AMP construction

Sender splits payment into N parts (where N == K, all required):

1. Choose **AMP root secret** `r` (32 bytes).
2. Compute "child preimages" using Shamir:
   - For each part `i = 1..N`, compute `child_preimage_i = SHA256(r || i)`
     (simplified — real spec uses Shamir over a finite field).
3. Each part's payment_hash = `SHA256(child_preimage_i)`.
4. Sender includes both `child_preimage_i` and `r` (XORed/derived) in
   the part's onion final-hop payload.

### Recipient flow

Recipient holds incoming HTLCs and tries to reconstruct `r`:

1. From each part's payload, extract `child_preimage_i` (or share).
2. Once N parts arrived, perform Shamir reconstruction to derive `r`.
3. Verify: each `child_preimage_i` SHA256 matches incoming part's
   payment_hash.
4. Reveal `child_preimage_i` to settle each HTLC individually.

### Per-part settlement

Unlike basic_mpp where all settle simultaneously with one preimage,
AMP settles each part with **its own unique preimage** derived from
the root secret. This is important: the recipient must possess all
shares before they have any preimage to reveal.

### Stateless invoices

Because each part has unique payment_hash, a recipient can advertise a
"static" AMP invoice (LND term: "spontaneous AMP") without prior
invoice creation. Sender pays N parts; recipient reconstructs root
and credits the account. This enables stream-payment use cases
(e.g. tipping a podcast).

## Worked example

Alice pays Bob via AMP:

```
Alice picks root r = 0xa1b2c3...
Decides 3 parts, threshold K=3.
Generates Shamir shares:
  share_1 = (1, f(1))
  share_2 = (2, f(2))
  share_3 = (3, f(3))

For each part i:
  child_preimage_i = derive(r, share_i)
  payment_hash_i = SHA256(child_preimage_i)

Sends 3 separate HTLCs:
  Part 1: amount=33k, payment_hash_1, payload: share_1
  Part 2: amount=33k, payment_hash_2, payload: share_2
  Part 3: amount=34k, payment_hash_3, payload: share_3

Bob receives:
  Part 1 arrives, holds.
  Part 2 arrives, holds.
  Part 3 arrives, has all 3 shares.
  Reconstructs r via Lagrange interpolation.
  For each i: derives child_preimage_i, checks SHA256 against payment_hash_i.
  Settles each part with its own preimage.
```

## Common pitfalls

- **Not standardized**: only LND implements; CLN and LDK don't. AMP
  invoices have a custom feature bit (LND-specific) and are not
  interoperable with non-LND wallets.
- **Stateless invoice replay**: same invoice can be paid multiple
  times; recipient must check duplicate root. UX for "this is a
  one-time invoice" requires extra signaling.
- **Part loss = full failure**: AMP requires all N parts. If even 1
  fails after timeout, all are forfeited. basic_mpp has same property
  but failure recovery is simpler.
- **Onion size**: payload per part includes Shamir share (~100 bytes);
  Lightning onion has 1300-byte limit; AMP supports max ~N=10 parts.
- **Non-correlation tradeoff**: each part has unique payment_hash, so
  routing nodes can't link parts. But timing analysis still correlates
  if parts arrive simultaneously at same intermediate.

## References

- LND AMP documentation (lightning.engineering).
- AMP draft proposal (Bastien Teinturier, 2020).
- Shamir. "How to Share a Secret" 1979.
- BOLT-04 onion format (constrains payload size).
