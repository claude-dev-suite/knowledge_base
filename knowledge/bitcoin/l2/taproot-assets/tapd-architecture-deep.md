# tapd Architecture - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/taproot-assets`.
> Canonical source: https://docs.lightning.engineering/the-lightning-network/taproot-assets
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/taproot-assets/SKILL.md

## Concept

`tapd` is Lightning Labs' Taproot Assets daemon: a Go service that runs alongside `lnd`
and provides asset minting, transfer, and Lightning-channel-asset functionality. tapd
implements client-side validation backed by **MS-SMT** (Merkle-Sum Sparse Merkle Tree)
state commitments anchored into Bitcoin Taproot outputs. The on-chain footprint per
transfer is a single Taproot output; the asset state lives in tapd's local DB plus
proof exchanges between sender and receiver.

## Walkthrough / mechanics

### Process layout

```
+-----------+   gRPC   +--------+   gRPC   +-------+
|  CLI/UI   +--------->+  tapd  +--------->+  lnd  |
+-----------+          +---+----+          +---+---+
                           |                   |
                           |  embedded DB       |  bitcoind / btcd
                           |  (SQLite/Postgres) |  ZMQ + RPC
                           v                   v
                    Universe sync          Bitcoin network
```

- tapd embeds an MS-SMT state DB (SQLite by default, Postgres for production).
- tapd talks to `lnd` for funding txs, channel asset commitments, and Lightning routing.
- tapd talks to **Universe** servers for asset metadata and proof distribution.

### MS-SMT commitments

Each Taproot output that anchors an asset embeds a commitment to:

```
TaprootKey = TweakedKey(InternalKey, TaprootScriptCommitment(MS-SMT root))
```

where MS-SMT is a 256-deep sparse merkle tree mapping `asset_id || output_key` to
`(amount, asset_metadata_hash)`. Sparse means only populated leaves cost storage; sum
means each internal node carries the cumulative amount of the subtree, enabling
inflation-resistance proofs.

### Genesis / minting

Minting events are batched in a "minting tx":

```
Asset {
  genesis_outpoint: <txid:vout of minting tx>,
  asset_id:         hash(genesis_outpoint || metadata),
  asset_type:       Normal | Collectible,
  group_key:        optional (links related issuances),
  amount:           u64
}
```

The minting tx commits to the new asset(s) via MS-SMT in its Taproot output. After
mining, the issuer can transfer or distribute.

### Transfers

Each transfer is a Bitcoin tx that:

1. Spends the input asset's anchor output.
2. Creates new anchor outputs (one per recipient + change).
3. Each new anchor's MS-SMT commits to the post-transfer balance state.
4. Sender provides the recipient with **proof files**: the chain of anchor outputs
   leading from genesis to the latest state, plus the asset's MS-SMT inclusion proof at
   each step.

Receiver verifies:
- Each anchor output's Taproot key matches its claimed MS-SMT commitment.
- Each transfer preserves asset conservation (sum-MMS).
- Genesis matches a known minting (universe-server lookup).

### Universe servers

Public indexers store asset proofs and metadata. Wallets sync universes to:
- Look up an asset by id or human-readable name.
- Fetch genesis / proof history when receiving a previously-unknown asset.
- Cross-reference asset existence and supply.

Universes are *informational*, not authoritative: lying about state is detectable by
on-chain anchor verification. Federated universes (multiple servers cross-replicating)
are the production deployment model.

## Worked example

Tether mints USDC-test, sends 100k to Alice, who sends 5k to Bob:

```
Step 1: Mint
  tapcli assets mint --type normal \
      --name USDC-test --supply 100000 --meta "stablecoin"
  Result:
    minting_tx = abc...:0
    asset_id   = hash(abc...:0 || metadata)
    Anchor output Taproot key tweaked with MS-SMT root [USDC-test/100000].

Step 2: Send 100k from Tether to Alice
  Tether tapd creates tx:
    IN  abc...:0  (anchor)
    OUT alice_anchor:0 -> commits to [USDC-test/100000 -> alice_key]
    OUT change:1  (just bitcoin dust)
  Sends Alice the proof file (chain abc...:0 -> tx2:0 + MS-SMT proofs).

Step 3: Alice sends 5k to Bob
  Alice tapd creates tx:
    IN  tx2:0
    OUT bob_anchor:0   -> commits to [USDC-test/5000 -> bob_key]
    OUT alice_anchor:1 -> commits to [USDC-test/95000 -> alice_key]
  Sends Bob proof file (chain abc...:0 -> tx2:0 -> tx3:0).

Step 4: Bob verifies on receipt
  - Replays anchor outputs from Bitcoin.
  - Verifies MS-SMT commitments at each anchor.
  - Verifies asset_id equals hash(abc...:0 || metadata).
  - Optionally queries Universe for genesis to confirm Tether is the issuer.
```

## Trade-offs and security

- **Proof loss = funds loss**: a wallet that loses its proof files cannot prove
  ownership; the on-chain anchor reveals nothing about who holds what. Backup is
  critical (BIP-39 seed alone is not sufficient -- you also need proof history).
- **Universe trust**: the universe lookup is a name-resolution layer. A malicious
  universe could hide the true genesis, but can't fake one (anchor verification still
  works against actual chain state).
- **MS-SMT vs RGB AluVM**: tapd's commitments are pure-balance trees; RGB allows
  programmable state machines via AluVM. tapd is simpler but less flexible.
- **Channel-asset complexity**: Lightning channels with assets require lnd + tapd version
  compatibility. Mismatch causes channel-update failures.
- **Privacy**: anchor outputs leak the *number* of asset transfers but not asset id,
  amount, or recipient. Coin-control patterns (output-merging) are needed for full
  unlinkability.
- **Tether / regulated issuer dependence**: real-world stablecoin TAP issuance depends on
  the issuer's universe metadata service being online and honest about supply.

## References

- Lightning Labs Taproot Assets docs - https://docs.lightning.engineering/the-lightning-network/taproot-assets
- BIP-PROTO-001..007 (TAP protocol BIPs) - https://github.com/lightninglabs/taproot-assets/tree/main/docs/bip-proto
- tapd repo - https://github.com/lightninglabs/taproot-assets
- "Taro" original announcement, Lightning Labs (2022)
