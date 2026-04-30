# Sphinx Onion Construction - Deep Walkthrough

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/onion`.
> Canonical source: BOLT 4 "Packet Construction", original Sphinx paper (Danezis & Goldberg 2009)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/onion/SKILL.md

## Concept

Lightning's onion routing is a Sphinx-style construction that gives
each hop a fixed-size 1366-byte packet to peel. Each hop sees only the
previous and next hop, and the packet length is constant regardless of
route depth, preventing length-based deanonymization. The construction
is specifically Sphinx (not Tor's circuit-switched onion) because it's
single-use and replay-protected.

## Walkthrough / mechanics

The full packet (BOLT 4 "Onion Packet"):

```
+--------+----------+--------------+--------+
| 1 byte | 33 bytes | 1300 bytes   | 32 byte|
| ver    | E (eph)  | hops_data    | HMAC   |
+--------+----------+--------------+--------+
```

`hops_data` is encrypted with a stream cipher; HMAC authenticates the
whole packet.

### Sender-side construction

Sender chooses route `[N1, N2, ..., Nk]` with public keys `[P1, P2, ...,
Pk]`. Generates ephemeral key:

```
e1 = random_scalar()
E1 = e1 * G
```

For each hop i, derive shared secret via ECDH and ratchet:

```
ss_i = SHA256(e_i * P_i)              // shared secret with hop i
b_i  = SHA256(E_i || ss_i)            // blinding factor
e_{i+1} = e_i * b_i                    // next ephemeral private
E_{i+1} = E_i * b_i                    // next ephemeral public
```

From `ss_i` derive five sub-keys (BOLT 4 "Key Generation"):

```
rho_i = HMAC-SHA256(ss_i, "rho")    // stream cipher
mu_i  = HMAC-SHA256(ss_i, "mu")     // HMAC key
um_i  = HMAC-SHA256(ss_i, "um")     // failure-onion HMAC key
pad_i = HMAC-SHA256(ss_i, "pad")    // filler generation
ammag_i = HMAC-SHA256(ss_i, "ammag") // failure stream cipher
```

Build hops_data inside-out:

1. Start with the final hop's payload `pl_k` and HMAC = 0.
2. For i from k down to 1:
   - Pack `pl_i || hmac_{i+1}` to length, prepend, encrypt with stream
     cipher keyed by `rho_i`.
   - Compute `hmac_i = HMAC-SHA256(mu_i, hops_data || associated_data)`.
3. After all hops: prepend filler bytes (deterministic from
   `pad_i` keys) so packet is always 1300 bytes regardless of how
   many hops are real.

### Per-hop processing

Hop i receives `(E_i, hops_data_i, hmac_i)`:

```
ss_i = SHA256(privkey_i * E_i)         // ECDH
verify hmac_i over hops_data_i with mu_i
decrypt first portion of hops_data with rho_i (one block)
parse payload, find next_hop (or terminal flag)
shift hops_data left by payload size (filler from rho_i fills tail)
recompute next E: E_{i+1} = E_i * SHA256(E_i || ss_i)
forward update_add_htlc with new (E_{i+1}, hops_data_{i+1}, hmac_{i+1})
```

Filler generation is deterministic so that after shifting, the
packet's tail can be reconstructed from the `pad_i` of the *removed*
hop. This is what keeps the packet exactly 1300 bytes at every hop.

### Replay protection

Each hop maintains a *replay table* of seen `ss_i` values (typically
the SHA-256 hash of `ss_i`). Reusing the same shared secret means
either replay attack or sender error; either way, fail.

## Worked example

A 3-hop route with payloads:

```
Hop 1 (B): payload = { amt_to_forward: 100_500, outgoing_cltv: 750_080,
                       short_channel_id: B->C }
Hop 2 (C): payload = { amt_to_forward: 100_000, outgoing_cltv: 750_040,
                       short_channel_id: C->D }
Hop 3 (D): payload = { amt_to_forward: 100_000, outgoing_cltv: 750_000,
                       payment_data: { secret, total }, ... }   // terminal
```

Onion construction (pseudo-code from `lightning-onion` ref impl):

```python
e = random_priv()
ss_list = []
for P in [P_B, P_C, P_D]:
    ss = ECDH(e, P)
    ss_list.append(ss)
    e = e * SHA256(P_E || ss)  # ratchet, where P_E = e_prev * G

# Build hops_data inside-out
data = b'\x00' * 1300
filler = generate_filler(rho_keys, lengths)
for i in range(2, -1, -1):
    payload = serialize(payloads[i])
    encoded = bigsize(len(payload)) + payload + hmac_buffer
    data = (encoded + data[:1300 - len(encoded)])
    if i == 0:
        data[1300 - filler_len:] = filler
    data = stream_xor(data, rho_keys[i])
    hmac = HMAC(mu_keys[i], data + assoc_data)
    hmac_buffer = hmac

packet = (0x00 || E_0 || data || hmac)
```

LND `routing/route` invokes `lightning-onion` to build packets.
LDK `lightning/src/ln/onion_utils.rs` is a complete Rust impl. CLN's
`common/sphinx.c` is the canonical C implementation cited by the
spec.

Decoding a real onion packet head:
```
00                                            # version 0
03 a1 b2 ... (33 bytes)                       # E_0
xx xx ... (1300 bytes)                        # encrypted hops_data
yy yy ... (32 bytes)                          # HMAC over above
```

## Common bugs / pitfalls

- **Blinding factor formula**: `b = SHA256(E || ss)` — using
  `SHA256(P || ss)` or different ordering is a common bug that breaks
  interop only at hop 2+.
- **Filler generation**: filler size = sum of removed hop lengths;
  off-by-one in the loop produces wrong packet bytes that pass HMAC at
  hop 1 but fail at hop 2.
- **Replay table memory**: storing every shared secret hash forever
  is unbounded. Spec recommends time-bounded retention (e.g. 1 hour
  past CLTV-expiry-corresponding-block).
- **Variable-length payload framing**: legacy fixed 65-byte payload
  vs TLV (`var_onion_optin`) — implementations must handle both. A
  recent payload is TLV-prefixed with bigsize length; legacy was raw.
- **Shared-secret reuse across MPP**: if two parts of an MPP share
  the same hop, the same `ss` for that hop gets used twice. Replay
  protection MUST be MPP-aware (use payment-id, not just `ss`).
- **HMAC key derivation domain separation**: using `ss` directly as a
  key (not via `mu = HMAC(ss, "mu")`) leaks key material across
  layers. The five-key derivation is mandatory.
- **Ephemeral key reuse**: `e1` reuse across two payments leaks the
  receiver to a colluding pair of hops. Always fresh `e1`.

## References

- BOLT 4 - Onion Routing Protocol: https://github.com/lightning/bolts/blob/master/04-onion-routing.md
- "Sphinx: A Compact and Provably Secure Mix Format" (Danezis, Goldberg 2009): https://cypherpunks.ca/~iang/pubs/Sphinx_Oakland09.pdf
- LDK onion utils: `lightning/src/ln/onion_utils.rs`
- CLN sphinx: `common/sphinx.c`
- `lightning-onion` Go library: https://github.com/lightningnetwork/lightning-onion
