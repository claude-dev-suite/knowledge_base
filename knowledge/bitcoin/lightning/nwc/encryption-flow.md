# NWC Encryption Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/nwc`.
> Canonical source: NIP-04 + NIP-44
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/nwc/SKILL.md

## Concept

NWC inherits Nostr's E2E encryption. NIP-04 (legacy) uses AES-256-CBC with
ECDH-derived symmetric keys; NIP-44 v2 (current) uses ChaCha20 with
HMAC-SHA-256 + HKDF derivation. Both protect the request/response payload
from relays. The wallet and app each have a secp256k1 keypair, and the
shared secret is the X coordinate of the EC-DH point. NIP-44 also adds
length-padding to defeat traffic analysis.

## Walkthrough / mechanics

### NIP-04 (deprecated but still common)

```
sk_a, pk_a   = client keypair
sk_w, pk_w   = wallet keypair (shared via NWC URI)

shared      = sk_a * pk_w   (or sk_w * pk_a, ECDH)
key         = X coord of shared (32 bytes)
iv          = random 16 bytes
ciphertext  = AES-256-CBC(key, iv).encrypt(plaintext_json)
content     = base64(ciphertext) + "?iv=" + base64(iv)
```

Decrypt: parse `content`, base64-decode parts, run AES-CBC decrypt with
`key` and `iv`.

NIP-04 weaknesses: malleable padding (CBC), no AEAD (no integrity over
ciphertext), length leaks plaintext size.

### NIP-44 v2

```
shared       = sk_a * pk_w   (X coord, 32 bytes)
conv_key     = HKDF-extract(salt='nip44-v2', ikm=shared)
nonce        = random 32 bytes
chacha_key   = HKDF-expand(conv_key, info=nonce, len=32)
chacha_nonce = HKDF-expand(conv_key, info=nonce||0x01, len=12)
hmac_key     = HKDF-expand(conv_key, info=nonce||0x02, len=32)
plaintext_padded = pad_to_pow2_minus_2(plaintext)   // length-hiding
ciphertext   = ChaCha20(chacha_key, chacha_nonce).encrypt(plaintext_padded)
mac          = HMAC-SHA-256(hmac_key, nonce || ciphertext)
content      = base64( 0x02 || nonce || ciphertext || mac )
```

Decrypt: verify MAC, ChaCha20 decrypt, strip padding.

NIP-44 strengths: AEAD via HMAC, length-padding (random reveals only
log2 of size), versioned (`0x02` byte).

## Worked example

NIP-44 over NWC pay_invoice:

```
secret_app   = 0x111...111  (32 bytes from URI)
pk_wallet    = 0x02fab1...  (33 bytes from URI)
plaintext    = {"method":"pay_invoice","params":{"invoice":"lnbc..."}}
              total length = 168 bytes

# pad to next pow2 - 2 = 254 bytes
plaintext_padded = plaintext || 0x00 * 86

# ECDH
shared = ecdh(secret_app, pk_wallet) = 32 bytes
conv_key = HKDF-Extract("nip44-v2", shared)

# random nonce
nonce = 0xabcdef...

# derive subkeys
chacha_key   = HKDF-Expand(conv_key, nonce,        32)
chacha_nonce = HKDF-Expand(conv_key, nonce||0x01,  12)
hmac_key     = HKDF-Expand(conv_key, nonce||0x02,  32)

# encrypt
ct  = ChaCha20(chacha_key, chacha_nonce).enc(plaintext_padded)
mac = HMAC-SHA-256(hmac_key, nonce || ct)

content = base64(0x02 || nonce || ct || mac)

# Publish kind=23194 with content + tags=[["p", pk_wallet]]
```

The wallet, after subscribing and receiving the event, recomputes
`shared = sk_wallet * (secret_app * G)`, then runs the same HKDF + ChaCha20
+ HMAC verification path to recover plaintext.

## Common bugs / pitfalls

- Mixing NIP-04 and NIP-44 events: a wallet that supports both MUST
  detect via the leading byte (NIP-04 has `?iv=` separator; NIP-44 starts
  with `0x02`). Misclassification yields garbled JSON.
- Reusing nonces across messages with the same shared secret: with NIP-04
  IV reuse breaks AES-CBC; with NIP-44 it breaks ChaCha20. Always
  generate fresh nonces.
- Forgetting the X-only flag: secp256k1 ECDH produces a point; only the
  X coordinate is used. Some libraries return the full point and the
  developer hashes the wrong representation.
- Padding mistakes in NIP-44: padding length encoded as 2-byte big-endian
  prefix; off-by-one yields invalid plaintext.
- HMAC includes nonce: forgetting to include nonce in the HMAC input is
  a common implementation bug; signatures still verify within a session
  but cross-session forgeries become possible.
- Constant-time comparison: HMAC verification must use constant-time
  compare. `==` on raw bytes leaks timing.

## References

- NIP-04: https://github.com/nostr-protocol/nips/blob/master/04.md
- NIP-44 v2: https://github.com/nostr-protocol/nips/blob/master/44.md
- NWC NIP-47: https://github.com/nostr-protocol/nips/blob/master/47.md
- noble-secp256k1 + noble-hashes reference implementations
