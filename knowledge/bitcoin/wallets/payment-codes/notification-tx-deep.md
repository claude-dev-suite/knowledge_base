# BIP47 Notification Transaction - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/payment-codes`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0047.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/payment-codes/SKILL.md

## Concept

The notification transaction is the one moment a BIP47 sender publicly tells the receiver "I am about to start paying you." It is needed because the receiver does not know the sender's master pubkey ahead of time and therefore cannot derive shared addresses without it. The notification carries the sender's payment code, blinded so only the targeted receiver can decrypt it, in a single OP_RETURN output. After this one tx, every subsequent payment is a standard P2WPKH/P2PKH output with no on-chain marker. The notification's privacy weakness is well known: anyone who learns the receiver's payment code can scan the chain for paying-to-notification-address outputs and learn the receiver's transaction graph counterparties.

## Walkthrough / mechanics

Receiver's notification address derivation:

```
P_R         = receiver_payment_code.pubkey       (33-byte compressed)
notif_priv  = derive(receiver_master, m/47'/0'/0')   (notification subaccount)
notif_pub   = notif_priv * G
notif_addr  = base58check(0x00 || hash160(notif_pub))
```

The notification address is a Base58 P2PKH address (legacy on purpose: it must be publicly indexable and was specified before SegWit was widespread). Receivers monitor it via Electrum scripthash subscription, BIP158 filter scan, or Bitcoin Core's `importaddress` + `rescanblockchain`.

Notification transaction shape:

```
inputs:
  [0] sender designated input (must be P2PKH or signed in a way that
       reveals sender's pubkey on-chain)
outputs:
  [0] dust to receiver's notification address      (>= 546 sats)
  [1] OP_RETURN <80 bytes blinded payment code>
  [2] sender change                                (optional)
```

Blinding the payment code:

```
a   = sender_designated_input_priv          (scalar from input [0])
P_N = receiver_notification_pubkey
S   = a * P_N                                (ECDH point)
s   = SHA512_HMAC(key="...",  msg=outpoint || S.x)
mask= s.bytes[0..64]
blinded = sender_payment_code XOR mask[0..80]
```

The OP_RETURN payload is the 80-byte blinded code (without the 1-byte type tag and checksum that wrap the Base58 form).

Receiver decryption:

```
A   = recover_pub(input[0])                # from sender signature
S'  = b * A    where b = notif_priv
must equal S above
mask' = SHA512_HMAC(...) using outpoint || S'.x
sender_payment_code = blinded XOR mask'[0..80]
```

The receiver verifies the result is a well-formed payment code (version byte = 0x01, valid pubkey on curve).

## Worked example

Real-style Bitcoin mainnet structure (testnet UUIDs swapped for readability):

```
notification tx:
  txid: 8c2af3d7...
  vin:
    [0] outpoint=  9f3c...:1
        scriptSig= <72-byte sig> <33-byte sender pubkey>
        prev value= 12000 sat (sender input chosen for the notification)
  vout:
    [0] 600 sat   -> 1J3eWzgi6vQekVrNJynHWJ2KNykMrWdpxF   (receiver notification addr)
    [1]   0 sat   -> OP_RETURN 0x010002b85034fb08a8bfef
                              d22848238257b252721454bb...
                              (80 bytes blinded)
    [2] 11000 sat -> bc1qsenderchange...                  (sender change)
  fee: 400 sat
```

Receiver's scanner sees output [0] arrive at the notification address. It loads the spending input [0]'s `pubkey` (recovered from `scriptSig`), computes `S' = b * A`, derives the mask, XORs the OP_RETURN bytes, and sees:

```
01 00 02 b85034fb08a8bfef... <chain_code 32B> <reserved 13B>
^  ^  ^^ ^^------ x of A_master --------^
|  |  +-- sign byte (0x02 even)
|  +-- features
+-- version
```

Now the receiver knows the sender's master pubkey and from this point forward derives addresses `B_0, B_1, ...` as in the per-payment derivation walkthrough. No additional notification is ever needed for that sender.

## Common pitfalls

- Picking a SegWit-only input as the designated input. BIP47 v1 mandates a P2PKH-style spend whose `scriptSig` reveals the pubkey explicitly. SegWit witnesses are also acceptable (the witness exposes the pubkey), but the wallet MUST extract from witness rather than `scriptSig`.
- Spending the notification UTXO before the receiver scans it. Many wallets immediately consolidate small dust; if they sweep the receiver's notification UTXO they break the receiver's history-detection. Receivers MUST treat notification UTXOs as never-spend (they are not part of the spendable wallet) and exclude them from coin selection.
- Re-broadcasting the same notification on every payment. The notification is one-time per sender->receiver pair; subsequent payments are plain P2WPKH outputs. Re-notifying wastes fees and burns privacy.
- Reorgs. If a notification gets reorg'd out, derived addresses are still cryptographically valid, but a receiver that already burned through gap-limit indices may miss subsequent payments until rescanning past the reorg.
- Not handling the privacy leak. The notification address is a permanent on-chain doxx: anyone with the receiver's payment code can ask "who has paid this person?" via chain analysis. Heavy users front the notification with Tor or use ephemeral notification addresses (BIP47 v3 proposals).
- Sending less than the dust threshold (546 sat for P2PKH) to the notification address; nodes will reject the tx as dust.

## References

- BIP47: https://github.com/bitcoin/bips/blob/master/bip-0047.mediawiki
- BIP47 v3 draft (stealth notifications): https://gist.github.com/SamouraiDev/6aad669604c5930864bd
- Sparrow Wallet notification flow: https://sparrowwallet.com/docs/paynyms.html
- Stack Wallet PayNym source: https://github.com/cypherstack/stack_wallet
- Privacy critique: "BIP47 transaction graph leak", Adam Gibson 2019
