# Mercury Layer Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/statechains`.
> Canonical source: https://mercurylayer.com/ - the only URL the project itself serves.
> (The repo that site links to, `commerceblock/mercurylayer`, 404s as of September 2026;
> a copy of the tree survives at https://github.com/mercury-layer/mercurylayer, a fork
> with no push since January 2025.)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/statechains/SKILL.md

## Concept

Mercury Layer is the statechain implementation CommerceBlock built, and redesigned around
blinded MuSig2 in January 2024. It enables off-chain transfer of UTXO ownership via a
2-of-2 multisig held between the user and a Statechain Operator (SO). Each transfer is a
key-share rotation rather than an on-chain spend, so ownership can move between parties for
the cost of a server round-trip rather than a Bitcoin fee. It is the reference statechain
implementation after the 2018 Somsen design, but whether it is *running* is unconfirmed as
of September 2026. The `commerceblock/mercurylayer` repo linked from mercurylayer.com 404s.
The project does publish operator endpoints — mercurylayer.com's "Status" button points at
`https://api.mercurylayer.com`, and docs.mercurylayer.com documents a signet key server at
`http://test.mercurylayer.com:8500` (`statechain_entity = "https://test.mercurylayer.com"`)
— but both hosts, while they still resolve in DNS (192.248.157.233 and 45.76.136.11),
answer nothing on any port tried (443, 80, 8500), while a control fetch of mutinynet.com,
the electrum server those same docs configure, returns 200. mercurywallet.com (the earlier
Mercury client's domain) no longer resolves at all (DNS SERVFAIL). All checked 16 September
2026. Read what follows as the protocol as specified, not as a service you can assume is
reachable.

The defining change from the original Mercury design is **blinding**. Since the January
2024 redesign the SO co-signs using a blinded two-party MuSig2 variant: key and nonce
aggregation happen only on the client, and the client sends the SO a blinded challenge
`c = e + b` in place of the real sighash. The SO therefore never learns the aggregate
public key `P`, the message it is signing, or the final signature — the project's own
site puts it as "The Mercury layer server is blind - it is not aware of bitcoin, and
does not perform any verifcation [sic] of transactions". Everything below follows from
that.

## Walkthrough / mechanics

### UTXO setup

```
[User A keypair (kA)]    [SO keypair (kSO)]
            \                /
             \              /
            2-of-2 MuSig output (P = kA + kSO)
                     |
            On-chain Bitcoin UTXO
```

The deposit tx funds an output controlled by the aggregate key. A pre-signed, time-locked
"backup" transaction is generated up front and given to A so that even if the SO disappears,
A can sweep the output after the timelock.

### Transfer A -> B

1. Receiver B generates an owner key `O2` and an authentication key `A2`; `O2 || A2`
   is B's "mercury address".
2. Sender A sends B an encrypted `TransferMsg` holding the full chain of signed backup
   transactions, a statechain signature over B's public key, and a *blinded* transfer
   value `t1 = o1 + x1` (where `x1` is a random the SO gave A). B never learns A's raw
   key share `o1`. The message may be relayed via the SO, which cannot read it.
3. B validates the chain client-side (below), then sends the SO `t2 = t1 - o2`,
   authenticated with `A2`. The SO updates its own share to
   `s2 = s1 + t2 - x1 = s1 + o1 - o2` and deletes `s1`, leaving the aggregate key `P`
   unchanged. The SO is *not* verifying a delegation — it is blind, and simply applies
   the offset it is handed.
4. The SO publishes its new public key share `S2` and the number of signatures it has
   produced for that `statechain_id`. There is no public proof-of-transfer log: the SO
   does not know the TXIDs, the aggregate key, or the signatures involved.

The on-chain UTXO never moves. Only the off-chain control of the 2-of-2 has rotated from
{A, SO} to {B, SO}.

### What the receiver verifies

Because the SO attests to nothing about the transfer, correctness rests on client-side
validation by the receiver. B checks that:

- The latest backup tx pays to `O2` and its input (`Tx0`) is still unspent.
- Each of the `K` backup txs carries a valid signature, was built exactly to spec, and
  has an `nLockTime` below its predecessor's, with the latest one not yet expired.
- The SO's reported signature count `N` for that `statechain_id` equals `K`. A count
  higher than the handed-over chain means the SO signed something B cannot account for.
- `t1.G = O1 + X1`, and the statechain signature verifies against `O1` with
  `P = S1 + O1` — which rules out a key-cancellation attack.

If any check fails, B refuses the coin. There is no after-the-fact audit trail to fall
back on.

### Backup tx update

Each new owner gets a fresh backup tx with a *shorter* timelock than the previous owner's,
so the latest owner can always sweep on-chain first. The decreasing-timelock chain is the
unilateral-exit safety net.

## Worked example

Alice deposits 0.1 BTC into Mercury, then sells the position to Bob over Discord:

```
Day 0:  Alice -> 2-of-2(Alice, SO) UTXO funded.   Backup nLockTime = 144000 (1 year).
Day 30: Alice -> Bob transfer.                     Backup nLockTime = 138240 (~960 days
                                                   left effective; Mercury uses block-
                                                   height-decreasing timelocks).
Day 31: Bob validates client-side: the backup-tx chain
        Alice handed him {genesis -> Alice -> Bob}, plus the
        SO's signature count, which must equal that chain's
        length. The SO supplies the count and nothing else.
Day 60: Bob withdraws on-chain to his own address.
        SO co-signs the cooperative withdrawal tx (1 on-chain tx, normal fee).
        The SO cannot tell this request from any other co-signing
        request.
```

If at Day 60 the SO is offline, Bob can broadcast his backup tx after its `nLockTime`
expires.

## Trade-offs and security

- **1-of-N trust on the SO**. A malicious SO that retains an old signing share can
  collude with a prior owner to double-spend. Mercury's defence is not a public transfer
  log — a blinded SO has nothing to publish about transfers — but the signature *count*:
  the SO reports how many times it has signed for a statechain, and a receiver refuses
  the coin when that count exceeds the number of backup txs the sender handed over. So
  equivocation is caught by the next receiver before it accepts, rather than by third
  parties auditing a log; it is missed entirely if a receiver skips the check, and it
  still does not automatically reverse a theft.
- **No script flexibility**: statechain UTXOs are P2TR/P2WSH cosigners; you cannot
  attach Lightning HTLCs or arbitrary contracts to a Mercury position without exiting.
- **Unilateral exit is bounded** by the decreasing timelock; long-held positions must
  periodically refresh the backup tx with the SO or risk the timelock expiring on a
  prior owner's branch.
- **Privacy**: since the January 2024 blinded redesign the SO sees no transfer graph at
  all — it learns neither the TXID, nor the address/aggregate public key, nor any
  signature it helped produce, and cannot distinguish a cooperative withdrawal from an
  onward transfer. What it does still learn is the per-`statechain_id` signature count
  and the timing/network metadata of the requests it serves. The transfer history
  remains fully visible to the *receiver*, who needs it in order to validate.

Compared to Spark (FROST k-of-n SOs) and Ark (round-based VTXO trees), Mercury is the
simplest model and the only one historically pre-dating taproot. Spark and Ark both
trace their lineage to Somsen's statechain design.

## Rebindable statechains (BIP448)

The decreasing-timelock backup chain above is the part a covenant soft fork would
remove. BIP448 "Taproot-native (Re)bindable Transactions" (Draft; BIP number assigned
2026-03-11; Gregory Sanders, Antoine Poinsot, Steven Roose) bundles `OP_TEMPLATEHASH`
(BIP446, redefining `OP_SUCCESS206`), `OP_CHECKSIGFROMSTACK` (BIP348, `OP_SUCCESS204`)
and `OP_INTERNALKEY` (BIP349, `OP_SUCCESS203`), and its Motivation section names
statechains as a beneficiary: the rebindable signatures make LN-Symmetry ("Eltoo")
possible, whose simplicity "can substantially improve statechains". Activation is "left
to be determined at a later date", and the opcodes are not active on mainnet as of
September 2026.

A prototype exists on test networks. The `rebindable_statechains` branch of
`stutxo/mercurylayer` — a fork of the Mercury tree, last pushed September 2026 — funds
each statechain to a Taproot output whose internal key is the aggregate `P` and whose
single leaf is:

```
OP_TEMPLATEHASH OP_INTERNALKEY OP_CHECKSIGFROMSTACK
```

Each logical state is an update tx `U(n)` plus a settlement tx `S(n)`, built against a
Bitcoin Inquisition revision pinned by that repository. The chain of decrementing
backup txs is replaced by Eltoo-style state replacement, which trades the bounded-exit
problem for a watching requirement: the accompanying Mutinynet browser wallet at
bip448.cash warns that if a previous owner publishes an old state, "you or a delegated
watcher have 144 blocks to broadcast the newer saved state", and that the prototype
"has no automatic watcher". It describes itself as "a Mutinynet research prototype ...
not a production wallet".

## References

- Ruben Somsen, "Statechains" (2018) - https://medium.com/@RubenSomsen/statechains-non-custodial-off-chain-bitcoin-transfer-1ae4845a4a39
- Mercury Layer - https://mercurylayer.com/
- mercurylayer GitHub - https://github.com/mercury-layer/mercurylayer - a fork with no
  push since January 2025, and the only copy of the tree that still resolves; the
  `commerceblock/mercurylayer` path linked from mercurylayer.com 404s as of
  September 2026
- Mercury protocol spec - https://github.com/mercury-layer/mercurylayer/blob/dev/docs/protocol.md
- Blinded two-party MuSig2 - https://github.com/mercury-layer/mercurylayer/blob/dev/docs/blind_musig.md
- "Mercury Layer: A Massive Improvement On Statechains", Bitcoin Magazine, 8 January 2024
- CommerceBlock blog posts on the 2024 redesign (post-MuSig2)
- BIP448, "Taproot-native (Re)bindable Transactions" -
  https://github.com/bitcoin/bips/blob/master/bip-0448.md - Draft, BIP number assigned
  2026-03-11; read September 2026
- BIP448 rebindable-statechain prototype -
  https://github.com/stutxo/mercurylayer/blob/rebindable_statechains/docs/bip448_rebindable_statechains.md
  - fork of the Mercury tree, last pushed September 2026
- Mutinynet browser-wallet demo for that prototype - https://bip448.cash/
