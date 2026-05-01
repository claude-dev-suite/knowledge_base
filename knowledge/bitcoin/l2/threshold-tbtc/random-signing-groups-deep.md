# Threshold tBTC Random Signing Groups - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/threshold-tbtc`.
> Canonical source: https://docs.threshold.network/applications/tbtc-v2
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/threshold-tbtc/SKILL.md

## Concept

tBTC v2 (Threshold Network) custodies BTC in **randomly-sampled threshold
ECDSA signing groups**. Each deposit is held by a small group (51 nodes,
51-of-100 threshold) chosen via VRF lottery from the wider Threshold
operator set. Mints and redemptions require co-signing by the assigned
group; if a group misbehaves, individual operators' staked T tokens are
slashed.

Random sampling is the security innovation: an attacker controlling
fewer than threshold operators globally is unlikely to control threshold
in any given group.

## Walkthrough / mechanics

### Operator set

- Anyone can register an operator node with a stake of T tokens (the
  Threshold native token).
- Operators run the tBTC software, participate in DKG ceremonies, and
  earn fees.

### Group selection

When a deposit comes in:

1. The Threshold contract emits a `groupSelectionRequest` with a deposit
   ID.
2. A VRF (random beacon) generates a random seed.
3. The seed is used to pseudo-randomly sample 51 operators from the
   active set, weighted by stake.
4. Selected operators run a DKG ceremony to compute a shared
   secp256k1 secret key. Public key `Q_group` is published on Ethereum.

This typically takes 5-10 minutes for the DKG to complete.

### Deposit script

User funds a Bitcoin output:

```
P2WSH script:
  IF
    <user_pk> CHECKSIGVERIFY
    <Q_group> CHECKSIG
  ELSE
    <8 weeks (CSV)> CHECKLOCKTIMEVERIFY DROP
    <user_pk> CHECKSIG
  ENDIF
```

User's CSV refund path lets them reclaim BTC if the group never mints.

### Mint flow

1. User funds the deposit script with V BTC.
2. After confs, user submits SPV proof of deposit to Threshold's
   Ethereum contract.
3. Group's relay verifies and triggers `mintTBTC(V)` to user's Ethereum
   address.
4. Group's BTC remains under their control, ready for future redemptions.

### Redemption flow

1. User burns tBTC and specifies Bitcoin destination address.
2. Threshold contract picks an active group whose UTXOs cover the
   request.
3. Group runs ECDSA threshold signing protocol over a Bitcoin tx
   spending its UTXOs to user.
4. Tx broadcast.

### Group rotation

Active groups expire after a service period (~6 months). Existing UTXOs
are migrated via on-chain "heartbeat" transactions to a new
randomly-sampled group. This rotation pattern bounds operator-set
influence over time.

### Slashing

If a group:

- Fails to produce a valid signature within timeout: members lose
  reward, mild slashing.
- Signs a fraudulent tx: each member loses their full T stake; tBTC is
  refunded from insurance fund.
- Goes offline: group is rotated and members lose reward.

## Worked example

Alice deposits 1 BTC.

```
1. Alice -> Threshold contract: requestDeposit(P_user_pubkey)
2. Random seed -> 51 operators selected: {O3, O11, O22, ...}
3. DKG: produces Q_group public key.
4. Contract emits address bc1q... derived from script {P_user, Q_group}.
5. Alice broadcasts BTC tx: 1 BTC -> bc1q...
6. After 6 confs, Alice submits SPV proof.
7. Threshold contract mints 1 tBTC to Alice's Ethereum addr.
```

Six months later, Alice redeems 0.5 tBTC:

```
1. Alice burns 0.5 tBTC on Ethereum, specifies bc1qabc...
2. Contract picks group G_42 holding sufficient UTXOs.
3. G_42 runs threshold ECDSA over the redemption tx.
4. After ~30 minutes, BTC tx broadcast.
5. After 1 BTC conf, Alice has 0.5 BTC.
```

## Common pitfalls

- **VRF security**: if VRF is biased or predictable, attacker can sample
  groups they control. Threshold uses on-chain commitments + reveals to
  bind the seed.
- **DKG aborts**: if any operator misbehaves in DKG, the protocol
  aborts and reselects. This adds latency for fresh deposits.
- **Group capacity**: large deposits require splitting across multiple
  groups; complicates UTXO accounting.
- **Slashing insurance solvency**: if many groups are slashed
  simultaneously (e.g. coordinated attack), insurance fund may run dry.
- **Operator centralisation**: a few large operators might dominate the
  active set; mitigated by per-operator stake caps.

## References

- Threshold Network tBTC v2 documentation.
- Gennaro, Goldfeder. "Fast Multiparty Threshold ECDSA" 2018.
- Random Beacon design (random.network).
- BIP143 (SegWit signing for tx witness data).
