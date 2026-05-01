# BIP324 v2 Handshake Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/p2p`.
> Canonical source: BIP324
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/p2p/SKILL.md

## Concept

BIP324 replaces the unencrypted v1 P2P protocol with an authenticated
encrypted transport using x-only secp256k1 keys, ChaCha20Poly1305
AEAD, and BIP-340-style tagged hashes for key derivation. The skill
sketches the handshake; this article walks the full byte sequence,
the elligator-squared encoding for ephemeral pubkeys (so they look
random), and the garbage-byte indistinguishability trick that allows
v2 nodes to be opportunistically interoperable with v1 peers.

## Walkthrough / mechanics

**Phase 1: Public-key exchange.**

```
A -> B:  EllSwiftPubkey_A (64 bytes) || garbage_A (0..4095 bytes)
B -> A:  EllSwiftPubkey_B (64 bytes) || garbage_B (0..4095 bytes)
```

`EllSwiftPubkey` is a 64-byte serialization of an x-only secp256k1
pubkey via the elligator-squared encoding so it is computationally
indistinguishable from random bytes - critical for traffic analysis
resistance.

Garbage is random padding; an observer can't tell where the pubkey
ends and garbage begins until decryption succeeds.

**Phase 2: Shared secret derivation.**

```
ecdh_secret = ECDH(A.priv, B.ell_pub)        # both sides compute
                                              # via x-coord product
session_id, initiator_keys, responder_keys =
    HKDF-SHA256(ecdh_secret, salt="bitcoin_v2_p2p_protocol", info=...)
    # outputs:
    # session_id: 32 bytes (used as connection identifier)
    # initiator_aead_key, initiator_garbage_terminator: 32 + 16 bytes
    # responder_aead_key, responder_garbage_terminator: 32 + 16 bytes
    # initiator/responder length keys: 32 bytes each (for length encryption)
```

**Phase 3: Garbage authentication.**

After exchanging keys+garbage, each side sends a 16-byte "garbage
terminator" (HMAC of the garbage under the session key). The other
side scans incoming bytes for this terminator to delimit garbage
end and decryption start.

```
A -> B:  garbage_terminator_A (16) || encrypted_packet_1 || ...
B -> A:  garbage_terminator_B (16) || encrypted_packet_1 || ...
```

**Phase 4: Encrypted packets.**

Each packet:

```
length_field      = 3 bytes encrypted with the length key (XOR-style stream)
ciphertext        = AEAD(plaintext, AAD=length_field, key=session, nonce=counter)
                    -- ChaCha20Poly1305 with 16-byte tag
```

Plaintext format inside the AEAD:

```
1 byte    header: bit 0 = "ignore packet", bits 1..7 reserved
N bytes   message contents
```

Messages have a 1-byte type "shortid" for common commands (VERSION=1,
VERACK=2, ...) or a string-encoded type for uncommon ones.

## Worked example

**Outbound v2 handshake (pseudo-code):**

```python
import os
from cryptography.hazmat.primitives.kdf.hkdf import HKDF
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.ciphers.aead import ChaCha20Poly1305

def v2_handshake(sock, our_priv, peer_addr):
    # 1. send ellswift pubkey + random garbage
    our_ellswift = ellswift_encode(our_priv * G)        # 64 bytes
    garbage_len = os.urandom(2)[0] & 0x0F               # 0..15 bytes
    garbage = os.urandom(garbage_len)
    sock.sendall(our_ellswift + garbage)

    # 2. receive peer's pubkey + garbage
    peer_ellswift = sock.recv(64)
    peer_garbage = b""
    # 3. derive shared key
    peer_pub = ellswift_decode(peer_ellswift)
    ecdh = ecdh_xonly(our_priv, peer_pub)
    keys = hkdf_expand(ecdh, "bitcoin_v2_p2p_protocol")

    # 4. compute and send our garbage terminator
    our_term = hmac(keys["initiator_term_key"], garbage)
    sock.sendall(our_term)

    # 5. read until peer terminator found
    buf = b""
    while True:
        buf += sock.recv(1)
        idx = buf.find(hmac(keys["responder_term_key"], peer_garbage_so_far))
        if idx >= 0:
            peer_garbage = buf[:idx]
            buf = buf[idx + 16:]
            break

    # 6. now in encrypted packet mode
    cipher_init = ChaCha20Poly1305(keys["initiator_aead_key"])
    cipher_resp = ChaCha20Poly1305(keys["responder_aead_key"])
    return cipher_init, cipher_resp
```

**Backwards-compatible v1 fallback:**

If the first 4 bytes received match the v1 magic (`f9beb4d9` mainnet),
the receiver SHOULD assume v1 and process accordingly. v2 ellswift
keys land on these bytes with probability `2^-32`, so collisions are
rare; if a collision occurs, the v2 connection fails handshake and
both sides retry plaintext.

A node can advertise v2 support via `NODE_P2P_V2` (service flag
`0x800`). Outbound connections to peers without this flag should
either fall back to v1 immediately or attempt v2 with retry.

## Common bugs / pitfalls

1. **Assuming the garbage is empty.** Some implementations send
   exactly 0 garbage bytes; others send 4 KiB. Length is unbounded
   per spec; receivers must scan continuously.
2. **Mixing initiator/responder keys.** Initiator uses
   `initiator_aead_key` for SENDING, `responder_aead_key` for
   RECEIVING. Symmetric mixup yields garbage decryption.
3. **Length-field reuse.** The 3-byte length field is XOR-encrypted
   with a separate stream from the AEAD; counter MUST be incremented
   per-packet across the whole connection. Reusing the counter leaks
   plaintext lengths.
4. **AAD omission.** The encrypted length field is the AAD for the
   AEAD; forgetting to feed it to the AEAD decryption causes
   authentication failure on every packet.
5. **Garbage terminator brute force.** An attacker who knows the
   first few bytes of garbage cannot derive the terminator (it's a
   HMAC under the session key). However, the terminator is fixed for
   the connection; reuse is safe.
6. **Forgetting nonce reset.** Every new connection has its own
   counter starting at 0. Reusing counters across reconnects with the
   same key (e.g., after a brief network hiccup) violates AEAD
   security.
7. **No identity authentication.** BIP324 confirms a peer can do
   ECDH but not WHO they are. MITM attacks are possible if the peer
   list is compromised. Use Tor onion services or known peers for
   identity binding.

## References

- BIP324: https://github.com/bitcoin/bips/blob/master/bip-0324.mediawiki
- BIP324 reference (Bitcoin Core): https://github.com/bitcoin/bitcoin/blob/master/src/net_processing.cpp
- ellswift squared encoding: https://github.com/sipa/secp256k1/pull/1129
- BIP340 Schnorr (tagged hash): https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki
