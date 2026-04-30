# Mercury Layer Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/statechains`.
> Canonical source: https://mercurylayer.com/ and https://github.com/commerceblock/mercurylayer
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/statechains/SKILL.md

## Concept

Mercury Layer is CommerceBlock's production statechain implementation. It enables off-chain
transfer of UTXO ownership via a 2-of-2 multisig held between the user and a Statechain
Operator (SO). Each transfer is a key-share rotation rather than an on-chain spend, so
ownership can move between parties for the cost of a server round-trip rather than a Bitcoin
fee. Mercury Layer is currently the only widely deployed statechain protocol after the
2018 Somsen design.

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

1. Receiver B publishes a fresh key kB and a fresh nonce commitment.
2. Sender A computes a "transfer message" containing the keys/proofs needed for B to
   continue cosigning with the SO.
3. A reveals their secret share to B (encrypted to B's key) and signs a statement
   delegating ownership to B.
4. B contacts the SO. The SO verifies A's delegation, deletes the prior cosigning state,
   and binds itself to cosign only with B going forward (key-update protocol on the SO
   side).
5. The SO publishes a public proof-of-transfer log entry so that any future transfer
   chain can be audited.

The on-chain UTXO never moves. Only the off-chain control of the 2-of-2 has rotated from
{A, SO} to {B, SO}.

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
Day 31: Bob receives, runs `mercury-cli verify`,
        sees full transfer chain {genesis -> Alice -> Bob}.
Day 60: Bob withdraws on-chain to his own address.
        SO co-signs the cooperative withdrawal tx (1 on-chain tx, normal fee).
```

If at Day 60 the SO is offline, Bob can broadcast his backup tx after its `nLockTime`
expires.

## Trade-offs and security

- **1-of-N trust on the SO**. A malicious SO that retains an old signing share can
  collude with a prior owner to double-spend. Mercury mitigates this with the public
  transfer log: equivocation is publicly detectable, which destroys the SO's
  reputation/business but does not automatically reverse the theft.
- **No script flexibility**: statechain UTXOs are P2TR/P2WSH cosigners; you cannot
  attach Lightning HTLCs or arbitrary contracts to a Mercury position without exiting.
- **Unilateral exit is bounded** by the decreasing timelock; long-held positions must
  periodically refresh the backup tx with the SO or risk the timelock expiring on a
  prior owner's branch.
- **Privacy**: better than on-chain transfer (no chain tx per move) but the SO sees the
  full transfer graph in plaintext; not comparable to Lightning onion routing.

Compared to Spark (FROST k-of-n SOs) and Ark (round-based VTXO trees), Mercury is the
simplest model and the only one historically pre-dating taproot. Spark and Ark both
trace their lineage to Somsen's statechain design.

## References

- Ruben Somsen, "Statechains" (2018) - https://medium.com/@RubenSomsen/statechains-non-custodial-off-chain-bitcoin-transfer-1ae4845a4a39
- Mercury Layer - https://mercurylayer.com/
- mercurylayer GitHub - https://github.com/commerceblock/mercurylayer
- CommerceBlock blog posts on the 2024 redesign (post-MuSig2)
