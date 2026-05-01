# BIP47 Per-Payment Derivation - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/payment-codes`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0047.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/payment-codes/SKILL.md

## Concept

A BIP47 payment code is a single static identifier the receiver publishes once. From it, every sender derives a fresh, unlinkable address per payment using ECDH between the sender's private key and the receiver's payment-code pubkey, mixed with a nonce. This means a tip jar can publish one QR / handle and still have every donor's payment land at a different on-chain address while the receiver scans only one derivation chain. The trick lives in how the shared secret is mixed into a BIP32 child path: each payment uses index `n = 0, 1, 2, ...` and both sides arrive at the same address independently.

## Walkthrough / mechanics

Payment-code structure (80 bytes, version 1):

```
[ 0]      version       = 0x01
[ 1]      features      = 0x00
[ 2]      sign          = 0x02 / 0x03 (compressed pubkey prefix)
[ 3..34]  x-coordinate  = 32 bytes (P_R = receiver's notification pubkey)
[35..66]  chain code    = 32 bytes
[67..79]  reserved      = 0x00 * 13
```

Plus 1-byte type tag (0x47) and 4-byte Base58Check checksum, encoded as a string with prefix `P` (for "payment code"); typical length ~116 chars in Base58.

Per-payment derivation (sender):

1. Sender derives its own BIP47 account at `m/47'/0'/<idx>'`. Sender index 0 = first BIP47 account.
2. Compute notification key: receiver's master pubkey `P_R = pc.deserialize().pubkey`.
3. For payment `n`:
   - Sender private child: `a_n = derive_priv(sender_account, "0/n")` -> scalar.
   - Shared secret: `s_n = SHA256(ECDH(a_n, P_R))`.
   - Reject if `s_n >= n_curve`. Otherwise tweak: `B_n = P_R + s_n*G`.
   - Address = P2WPKH (or P2PKH per receiver feature flag) of `B_n`.

Receiver computes the same `B_n` from the other direction:

1. Receive notification tx, decrypt sender's payment code from OP_RETURN blob.
2. Sender pubkey `A = pc_sender.pubkey`.
3. For each `n = 0, 1, 2, ...`:
   - `s_n = SHA256(ECDH(b, A_n))` where `b` is receiver's master priv and `A_n = derive_pub(sender_account, "0/n")`.
   - Receiver private key for this address: `b + s_n` (mod n_curve).
   - Watch-address = P2WPKH of `B + s_n*G`.

Receiver scans up to a gap limit (typically 10) past the highest seen.

## Worked example

Receiver payment code (truncated for brevity):

```
PM8TJTLJbPRGxSbc8EJi42Wrr6QbNSaSSVJ5Y3E4pbCYiTHUskHg13935Ubb7q8tx9GVbh2UuRnBc3WSyJHhUrw8KhprKnn9eDznYGieTzFcwQRya4GA
```

Sender's BIP44 wallet seed (test fixture, BIP39):

```
"praise you muffin lion enable neck grocery crash acute change host vault"
```

Sender computes BIP47 account 0:

```
m/47'/0'/0'  -> sender_xprv
```

For the first payment (`n = 0`):

```
a_0 = derive_priv(sender_xprv, "0/0")        # 32-byte scalar
P_R = decode(receiver_pc).pubkey             # 33-byte compressed
ecdh = (a_0 * P_R).x                         # x-coordinate only
s_0  = SHA256(ecdh)                          # 32 bytes
B_0  = P_R + s_0 * G                         # new pubkey
addr_0 = bech32_encode(hash160(B_0))         # bc1q...
```

For payment `n = 1` repeat with `a_1 = derive_priv(sender_xprv, "0/1")`. Both sides arrive at the same `addr_1` without further communication.

OP_RETURN blob in the notification tx (sender -> receiver, Bitcoin testnet example):

```
6a4c50010002b85034fb08a8bfefd22848238257b252721454bbbf80...
^^   ^^ ^^^^---- 80-byte blinded payment code -----------
|    |  |
|    |  +-- BIP47 marker ('A' = 0x41 in some impls; here payload tag)
|    +-- pushdata length (0x50 = 80)
+-- OP_RETURN
```

The blinding XORs the payment code with `HMAC-SHA512("notification key", receiver_pubkey * sender_priv)` so only the receiver can read it.

## Common pitfalls

- Skipping the `s_n >= n_curve` check. The probability is ~`2^-128` but BIP47 mandates the check; on hit, increment `n` and try again.
- Reusing the notification address for a second sender. Each sender's notification UTXO must be a fresh address; reusing it links senders publicly.
- Deriving from `m/47'/0'/0'` for both BIP44 funds and BIP47 funds. Keep BIP47 strictly under the `47'` purpose to avoid path collisions.
- Forgetting the gap limit on the receiver side. If a sender skips ahead (e.g., wallet crash) and resumes at `n = 30`, a receiver that only scans 0-9 will not see the payment.
- Computing ECDH on the full uncompressed point and hashing the whole thing. BIP47 hashes only the x-coordinate (32 bytes) of the shared point, matching SECG's compact ECDH.
- Confusing notification-tx privacy with on-chain privacy. The on-chain payments are unlinkable, but the notification tx itself reveals "this sender intends to pay this receiver".

## References

- BIP47: https://github.com/bitcoin/bips/blob/master/bip-0047.mediawiki
- libpaynym reference (Samourai legacy): https://github.com/Samourai-Wallet/PayNym-Resolver
- Sparrow Wallet PayNym source: https://github.com/sparrowwallet/sparrow
- BIP352 (Silent Payments comparison): https://github.com/bitcoin/bips/blob/master/bip-0352.mediawiki
- "PayNym v2 spec" (Sparrow): https://paynym.is/
