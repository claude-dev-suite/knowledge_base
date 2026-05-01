# PayJoin v2 Asynchronous Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/payjoin`.
> Canonical source: https://github.com/bitcoin/bips/pull/1483 (BIP77 draft)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/payjoin/SKILL.md

## Concept

PayJoin v2 (sometimes written "BIP77" / "async PayJoin") removes the BIP78
requirement that **both sender and receiver be online simultaneously**. v1
required the receiver to host an HTTPS endpoint reachable at payment time;
this excluded mobile wallets, hardware wallets, and cold-storage receivers.

v2 introduces a **store-and-forward relay** ("directory") that buffers
encrypted PSBTs between parties. The relay is untrusted: end-to-end encryption
uses **HPKE (RFC 9180)** with the receiver's static key.

## Walkthrough / mechanics

**Roles**: `sender`, `receiver`, `directory` (relay), `OHTTP gateway`
(metadata-stripping reverse proxy, RFC 9458).

1. **Receiver provisions a session**. Receiver generates a fresh secp256k1
   keypair `(pk_r, sk_r)`, registers `pk_r` with a directory URL, and emits a
   payjoin URI:
   ```
   bitcoin:bc1q...?amount=0.01&pj=https://relay.example/PJ/<pk_r>&pjos=0
   ```
   The receiver may then go offline.

2. **Sender encrypts original PSBT**. Sender constructs `original_psbt` like
   v1, then HPKE-seals it to `pk_r`. The ciphertext is HTTP-POSTed via OHTTP
   to the directory at `/<pk_r>`.

3. **Directory stores payload**. Directory holds the ciphertext keyed by
   `pk_r`. It cannot decrypt (does not have `sk_r`). OHTTP hides the
   sender's IP from the directory.

4. **Receiver polls / long-polls**. When online, receiver fetches and
   decrypts `original_psbt`. It validates as in BIP78, picks a UTXO, builds
   `payjoin_psbt`, encrypts under the **sender's ephemeral key** (carried in
   the original payload), and POSTs back to a sender-owned directory slot.

5. **Sender finalises**. Sender (next time it polls) decrypts the
   `payjoin_psbt`, validates, signs, broadcasts.

**Directory rotation**: pk_r is a **single-use subdirectory key**, so a
single receiver `xpub` can derive a unique HPKE key per invoice. This
prevents the directory from linking multiple invoices to one receiver.

## Worked example

Mobile wallet receiver:

| Time | Sender | Receiver |
|------|--------|----------|
| t0   | reads QR, builds original PSBT | offline |
| t1   | seals + POSTs to relay         | offline |
| t2   | offline                        | wakes, polls, finds payload |
| t3   | offline                        | builds payjoin PSBT, POSTs back |
| t4   | wakes, polls, finds reply, signs, broadcasts | offline |

Total wall-clock latency: bounded only by polling intervals. Both sides may
be on cellular networks behind NAT.

## Common pitfalls

- **Directory operator as adversary**: relay sees ciphertext sizes and
  timing. Use OHTTP to break IP linkage.
- **Sender-receiver clock skew**: HPKE has no timestamps; directories MUST
  TTL-expire entries (typical 24 h) to avoid replay across days.
- **Ephemeral-key reuse**: sender MUST generate a fresh HPKE keypair per
  payment, otherwise the directory can correlate multiple payments to the
  same sender wallet.
- **Fall-back tx**: still required. v2 keeps the BIP78 property that
  `original_psbt` is a valid standalone payment, broadcastable if the
  v2 round-trip never completes.
- **Versioning**: clients MUST advertise `v=2` and gracefully fall back to
  v1 (synchronous endpoint) if the receiver URL responds 404 to v2 routes.

## References

- BIP77 draft (PayJoin v2).
- RFC 9180 (HPKE).
- RFC 9458 (Oblivious HTTP).
- payjoin.org reference Rust implementation.
