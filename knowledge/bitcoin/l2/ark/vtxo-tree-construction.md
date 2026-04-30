# VTXO Tree Construction - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/ark`.
> Canonical source: https://docs.arklabs.to/ and https://arkdev.info/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/ark/SKILL.md

## Concept

Ark batches many user balances into a single on-chain "pool tx" per round. The pool tx
spends ASP-funded coins into a binary tree of pre-signed transactions; each leaf of the
tree is a VTXO (Virtual Transaction Output) belonging to a specific user. Users can
transact against their VTXOs entirely off-chain inside the round, and at any point can
unilaterally walk down the pre-signed branch to extract their VTXO on-chain. The tree
shape is what gives Ark its log(N) unilateral exit and zero per-user setup cost.

## Walkthrough / mechanics

### Pool tx and tree shape

```
                    [Pool tx]
                        |
                  [Tree root output]
                  /              \
              [Inner]           [Inner]
              /    \             /    \
          [VTXO] [VTXO]      [VTXO] [VTXO]
          Alice  Bob          Carol  Dave
```

Each non-leaf output is spent by exactly one pre-signed tx that splits it into two children.
All branch-spending txs are pre-signed by the ASP and the relevant subset of round
participants using MuSig2/FROST cosigning during the round. After the round, only the
pool tx is broadcast; everything below it remains off-chain unless someone exits.

### Round timeline

1. **Registration phase** (~5s in ARKADE): users submit "intents" -- a payment they want
   to make and the inputs they bring (existing VTXOs being respent or fresh on-chain BTC
   for boarding).
2. **Tree construction**: ASP determines the tree of outputs satisfying all intents, then
   coordinates a multi-party cosigning ceremony for every internal branch tx. ARKADE uses
   Taproot key-path with MuSig2 nonce aggregation.
3. **Pool tx broadcast**: after every branch is signed, ASP signs and broadcasts the pool
   tx. Users receive their branch path + pre-signed witnesses.
4. **Forfeit / refresh**: respending a VTXO requires the prior owner to sign a "forfeit
   tx" that lets the ASP claim that VTXO if the user tries to exit later. This prevents
   double-spending across rounds.

### CSV expiry

Every branch output carries a relative timelock (commonly 4 weeks in ARKADE). Before the
timelock expires, only the cooperative cosigning path can spend. After expiry, the ASP
alone can sweep the branch -- which is why users must either refresh into a new round
(rotating their VTXO into a new tree) or exit on-chain before expiry.

## Worked example

Round R1 has 4 users, each with 0.01 BTC VTXO. Tree depth = 2.

```
Pool tx fee: 1 vbyte * 4 = 4 sats/vbyte * ~150 vbytes ~= 600 sats total
Per-user cost share: 150 sats amortized
ASP capital lock: 0.04 BTC (sum of leaves) until tree refresh

User Bob wants to exit unilaterally on day 20:
  - Needs to broadcast: branch_tx_root -> branch_tx_inner_left -> bob_leaf_tx
  - 3 txs, each ~120 vbytes, at exit-time fee rate (Bob attaches CPFP if needed)
  - Total exit cost: ~360 vbytes + CPFP. Roughly 5x the cooperative cost,
    but safe even if ASP refuses.

User Alice wants to send 0.005 BTC to Carol off-chain:
  - In round R2, Alice's VTXO is spent (forfeited) and split into:
      * 0.005 BTC VTXO leaf for Alice (change)
      * 0.005 BTC VTXO leaf for Carol
  - Carol now has a VTXO under R2's tree; original R1 tree path is no longer
    Alice's claim.
```

## Trade-offs and security

- **ASP collusion**: a malicious ASP that successfully gets users to forfeit then refuses
  to publish the new round can steal up-to-recently-active balances. Forfeit txs are
  designed so that the ASP cannot claim a VTXO without first publishing the new tree --
  but only if clients verify before signing forfeits. Wallet correctness is critical.
- **Liveness**: ASP downtime extends "no-payment" windows. Users can always exit
  on-chain, but exit cost is non-trivial.
- **Tree depth**: ARKADE balances trees to depth O(log N). At N = 10,000 leaves the worst
  exit is 14 txs; at N = 1,000,000 it is 20.
- **Privacy**: ASP sees the full graph of intents. Better than on-chain because the chain
  observer just sees the pool tx, but worse than Lightning's onion routing.
- **Capital efficiency**: ASP must lock liquidity equal to total VTXO sum until the
  CSV-expiry sweep. This is recouped via fees and round refreshes.

## References

- Burak, "Ark" original post (bitcoin-dev, 2023)
- ARKADE docs - https://docs.arklabs.to/
- arkdev.info - https://arkdev.info/
- Second's `bark` reference impl - https://codeberg.org/ark-network/bark
- "Ark on Bitcoin" presentation, BTC++ 2024
