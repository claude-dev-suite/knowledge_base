# BIP47 Notification Transaction Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/bip47-paynyms`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0047.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/bip47-paynyms/SKILL.md

## Concept

A BIP47 notification transaction (NTX) is the **one-time on-chain handshake**
that lets Alice tell Bob "I am about to send you payments derived from your
payment code". Bob will scan his **notification address** (a fixed P2PKH
derived from his payment code) to discover Alice's payment code. After this
single tx, Alice and Bob can exchange unlimited payments without any further
on-chain registration.

## Walkthrough / mechanics

### Notation

- `Alice payment code`: 80 bytes = 1-byte version + 1-byte features +
  33-byte chaincode-prefixed pubkey + 32-byte chaincode + 13 reserved bytes.
- `Bob payment code` is structured the same way.
- Bob's notification address `N_B` = P2PKH(`HASH160(B_0)`) where `B_0` is the
  pubkey at index 0 derived from Bob's payment code.

### NTX structure

| Output | Script | Value |
|--------|--------|-------|
| 0 | P2PKH to Bob's notification address `N_B` | dust-respecting amount (~546 sat) |
| 1 | OP_RETURN <80-byte blinded payment code of Alice> | 0 sat |
| 2..n | Alice's change | remainder |

The blinded payload is computed:

1. Pick a "designated input" `I` that Alice will spend. Typically the input
   with the smallest outpoint.
2. Let `a` be the private key of `I`, and `B` Bob's notification pubkey
   (`B_0`).
3. Compute shared secret `S = SHA512(a * B)` -> 64 bytes.
4. Blinding key = `S` interpreted as `(IV || keystream)`.
5. XOR Alice's 80-byte plain payment code with the keystream -> 80-byte
   blinded payload.
6. Place blinded payload in `OP_RETURN`.

Bob, scanning, sees a transaction to `N_B`, picks the **first** input he can
identify (i.e. takes its `prev_out` pubkey), computes
`S = SHA512(b * A_input_pub)` and reverses the XOR to recover Alice's
payment code.

### Why a designated input

- Guarantees a stable pubkey Bob can recover. Bob does not have Alice's
  private key, so he cannot recompute `S` from arbitrary witness data; he
  needs a canonical pubkey.
- Designated input rule: smallest outpoint by lexical order of
  `txid || vout` BE.

### Cost and privacy implications

- ~150 vB tx, 1 dust output, 1 OP_RETURN, 1 change. ~1500 sat at
  10 sat/vB plus dust.
- The NTX **publicly links Alice's spending UTXO to Bob's notification
  address** forever. Chain analysis can build "everyone who notified Bob"
  -> partial sender graph.
- The 80-byte payload is encrypted but stable; if Alice's `a` ever leaks,
  the payment code can be unblinded.

## Worked example

Alice payment code (abbreviated): `PM8TJTLJbPRGxSbc8EJi42Wrr6QbNSaSSV...`.
Bob notification address: `1JDdmqFLhpzcUwPeinhJbUPw4Co3aWLyzW`.

```
NTX outputs:
 vout 0: OP_DUP OP_HASH160 <Bob N_B hash160> OP_EQUALVERIFY OP_CHECKSIG  546 sat
 vout 1: OP_RETURN  <80-byte blinded Alice payment code>                  0 sat
 vout 2: change to Alice                                                  X sat
```

Bob's wallet scans every block for outputs paying `N_B`. On match, it
reads vout 1 OP_RETURN, fetches the prev-output of vout-1's first input,
extracts the pubkey, computes `S = SHA512(b * pub)`, XORs the OP_RETURN
payload, and stores Alice's payment code if version + features check out.

## Common pitfalls

- **Reusing the notification UTXO** for a regular spend: never spend the
  designated input again before the NTX confirms; reorg can re-derive a
  different "first input" and corrupt detection.
- **OP_RETURN size**: must be exactly 80 bytes; some node policies cap at
  83. Build with care to avoid mempool-policy rejection.
- **Notification dust outputs** create a permanent UTXO on Bob's side;
  a popular receiver accumulates thousands of unspendable dust outputs
  unless they consolidate (and consolidation links them).
- **Cross-chain replay**: don't reuse the same payment code on Bitcoin and
  testnet/Litecoin; the notification could reveal cross-network identity.
- **OP_RETURN is unencrypted to anyone with `b`**: i.e. Bob's notification
  privkey leaking outs every sender that ever notified him.

## References

- BIP47 — Reusable Payment Codes for HD wallets.
- Samourai bitcoinj-extensions BIP47 implementation.
- "PayNyms in the wild" — Samourai docs.
