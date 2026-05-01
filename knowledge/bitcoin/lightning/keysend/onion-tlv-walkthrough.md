# Keysend Onion TLV Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/keysend`.
> Canonical source: https://github.com/lightning/bolts/blob/master/04-onion-routing.md (TLV onion)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/keysend/SKILL.md

## Concept

Keysend lets a Lightning sender pay a recipient **without an invoice**.
The sender picks a random preimage, hashes it to a payment_hash, and
embeds the preimage in the onion's final-hop TLV payload. The recipient
reads the preimage, verifies it hashes to the HTLC's payment_hash, and
settles. No invoice, no payment_secret, no prior interaction.

Keysend is the foundation for podcast streaming sats, Sphinx chat, and
zaps-style spontaneous payments.

## Walkthrough / mechanics

### TLV format

Lightning onion final-hop payload uses TLV (BOLT-04). Standard TLVs:

| Type | Name | Required for keysend? |
|------|------|------------------------|
| 2 | amt_to_forward | Yes |
| 4 | outgoing_cltv_value | Yes |
| 5482373484 | keysend_preimage | Yes (keysend specific) |
| 7629169 | custom record (Sphinx chat msg) | Optional |
| 33 | payment_metadata | Optional |

The `keysend_preimage` TLV (type 5482373484, hex `0x14668DAC`) carries
the 32-byte preimage.

### Sender flow

1. Sender picks `preimage = random 32 bytes`.
2. Computes `payment_hash = SHA256(preimage)`.
3. Constructs onion final-hop payload with TLV 5482373484 = preimage.
4. Routes HTLC normally to recipient (sender knows recipient's pubkey).
5. Recipient settles using preimage.

### Recipient verification

Recipient on receiving HTLC:

1. Decrypt onion final hop.
2. Find TLV type 5482373484. If absent -> fail HTLC ("invalid_onion_payload").
3. Compute SHA256(preimage); compare to HTLC's payment_hash. If mismatch
  -> fail.
4. Extract amt_to_forward; if amt < htlc_amt minus tolerable fee, fail.
5. Settle HTLC with preimage. Credit recipient's wallet.

### Why a high TLV type number?

5482373484 = 0x14668DAC chosen to avoid collision with reserved BOLT
types (low numbers <= 256). Lightning convention reserves high custom
TLVs for application/extension use.

### CLTV and amount

Sender includes amt_to_forward and outgoing_cltv_value as in normal
payments. Recipient's node must accept.

### Receiver opt-in

Receiver must enable `keysend` in their node config:
- LND: `accept-keysend=true`
- CLN: `keysend` plugin (default off in some versions)
- LDK: `accept_unverified_keysend=true`

If disabled, HTLC fails with "unknown_payment_hash".

### Custom records (Sphinx chat, podcast metadata)

Senders attach extra TLVs alongside the preimage:

```
TLV 5482373484: preimage
TLV 34349334  : "msg" -> "Hello Bob, here's 1000 sat"  (Sphinx message)
TLV 7629169   : podcast metadata JSON
```

Receiver applications parse these and display.

## Worked example

Alice sends 5000 sat to Bob (pubkey 03abc...) via keysend.

```
1. Alice picks preimage = 0xa3b1...
2. payment_hash = SHA256(preimage) = 0x91f2...
3. Builds onion: route Alice -> Hop1 -> Hop2 -> Bob
   Final hop payload TLVs:
     type=2 (amt_to_forward): 5000 sat = 5,000,000 msat
     type=4 (outgoing_cltv_value): 800100
     type=5482373484 (keysend_preimage): 0xa3b1...
4. Sends HTLC with payment_hash=0x91f2..., amount=5000+routing_fee.
5. Bob receives HTLC, decrypts final payload, finds keysend_preimage.
6. Verifies SHA256(0xa3b1...) == 0x91f2...
7. Settles HTLC, wallet credited 5000 sat.
```

## Common pitfalls

- **Receiver disabled**: keysend must be enabled; default-off on many
  setups for safety.
- **No invoice means no metadata**: receiver doesn't know who Alice is
  unless custom records are attached. Use Sphinx chat TLVs for chat
  context; otherwise the payment is anonymous.
- **No payment_secret**: keysend payments lack the basic_mpp
  payment_secret. Replay-protection: recipient must store seen
  payment_hashes (preimage hashing prevents replay since SHA256 is
  one-way; same preimage couldn't be reused without sender knowing).
  In practice, fresh preimages are random and collision-free.
- **MPP keysend** is supported by some implementations but
  non-interoperable. AMP-style keysend variants exist in LND but not
  elsewhere.
- **Onion size**: large custom records may exceed 1300-byte onion
  payload; receiver fails the HTLC.

## References

- LND keysend documentation.
- BOLT-04 Onion Routing § "TLV format".
- Sphinx chat protocol (Stakwork docs).
