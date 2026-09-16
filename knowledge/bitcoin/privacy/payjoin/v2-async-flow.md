# PayJoin v2 Asynchronous Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/payjoin`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0077.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/payjoin/SKILL.md

## Concept

PayJoin v2 is specified as **BIP77 "Async Payjoin"** (Dan Gould, Yuval Kogman).
The BIP number is assigned and the document is merged in the BIPs repo; its
status is **Draft** and its spec version is **0.2.0** as of September 2026.

BIP77 removes the BIP78 requirement that **both sender and receiver be online
simultaneously**. v1 required the receiver to host an HTTPS endpoint reachable
at payment time; this excluded mobile wallets, hardware wallets, and
cold-storage receivers.

v2 introduces a **store-and-forward relay** ("directory") that buffers
encrypted PSBTs between parties. The relay is untrusted: end-to-end encryption
uses **HPKE (RFC 9180)** with the receiver's static key.

## Walkthrough / mechanics

**Roles**: `sender`, `receiver`, `directory` (relay), `OHTTP gateway`
(metadata-stripping reverse proxy, RFC 9458).

1. **Receiver provisions a session**. Receiver generates a fresh secp256k1
   keypair `(pk_r, sk_r)`, registers `pk_r` with a directory URL, and emits a
   payjoin URI. The `pj=` URL is a **mailbox endpoint** whose path is a
   *Short ID* — SHA-256 of the 33-byte compressed `pk_r`, truncated to 8
   bytes and encoded with the bech32 character set. BIP77 adds no new BIP21
   parameters: the session parameters live in the **fragment** of that URL
   (`EX` expiry, `OH` OHTTP key config, `RK` receiver key), `-`-separated in
   lexicographical order, with `#` percent-encoded as `%23`:
   ```
   bitcoin:tb1q...?amount=0.00666666&pjos=0&pj=HTTPS://PAYJO.IN/TXJCGKTKXLUUZ%23EX1...-OH1...-RK1...
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
- **Versioning**: there is no `v=2`. BIP77 declares BIP78's `v` parameter
  redundant — HPKE binds ciphertexts to an application-specific `info`
  string, which already supplies the domain separation — and says it should
  be omitted. Nor is there a 404 version probe: a backwards-compatible
  receiver distinguishes versions from the request body itself, treating a
  UTF-8 plaintext payload as BIP78 (see next bullet).
- **Replying to a BIP78 sender**: a v2 receiver may also service plain BIP78
  senders, who post a cleartext base64 PSBT to the mailbox. BIP77 made those
  reply rules normative in June 2026: the response MUST NOT be HPKE-encrypted,
  MUST be a `PUT` whose body is the base64 PSBT encoded as ASCII, and MUST
  target the **receiver's own** mailbox — a BIP78 sender supplies no reply key
  from which a sender-side mailbox could be derived. The directory returns
  BIP78's `unavailable` error if no response arrives within 30 seconds.

## References

- BIP77 "Async Payjoin" — `bip-0077.md` in the BIPs repo (Draft, spec
  version 0.2.0 as of September 2026).
- RFC 9180 (HPKE); BIP77 uses `DHKEM(Secp256k1, HKDF-SHA256)` with
  ElligatorSwift-encoded encapsulated keys.
- RFC 9458 (Oblivious HTTP).
- `payjoin/rust-payjoin` — reference implementation; first stable release
  `payjoin-1.0.0` on 12 August 2026, covering both BIP78 and BIP77. UniFFI
  bindings `payjoin-python-0.2.0`, `payjoin-dart-0.2.2`,
  `payjoin-csharp-0.1.0`, `payjoin-javascript-0.2.0` (JS and C# published
  27 August 2026). `payjoin-cli` was still at `1.0.0-rc.2` as of
  September 2026.
- `payjoin/ohttp-relay` — the OHTTP relay implementation named by the BIP.
