# Arkade Mainnet Features - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/ark`.
> Canonical source: https://arklabs.to/ and https://docs.arklabs.to/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/ark/SKILL.md

## Concept

ARKADE is Ark Labs' production Ark deployment. As of 2025-2026 it is in **public mainnet
beta**: real BTC, real users, but with conservative round caps and continuous protocol
iteration. ARKADE is the first Ark protocol implementation to reach mainnet; the
reference `bark` (by Second) is still on signet/testnet. ARKADE differentiates from the
spec with three key features: 5-second rounds, Taproot/MuSig2-only signing, and Arkade
Assets for native asset issuance.

## Walkthrough / mechanics

### 5-second rounds

ARKADE compresses the registration/cosign/broadcast cycle to ~5 seconds by:

- Pre-publishing a tentative tree shape so cosigners can pre-compute MuSig2 nonces.
- Using a "fast path" where small rounds (< 256 leaves) skip multi-party FROST and use
  pure MuSig2 (n-of-n) for branch signatures, then fall back to ASP-only signing if any
  party drops out (with the dropout's intent removed).
- Streaming commitments over websocket rather than batched HTTP.

### V-PACK (Stateless VTXO Verification)

V-PACK is ARKADE's extension allowing a wallet to verify a VTXO and its lineage **without
holding round state**. A V-PACK is a self-contained proof:

```
V-PACK {
  vtxo_id:        bytes32,
  round_pool_txid: bytes32,
  branch_path:     [tx, witness, output_index, ...],   // root -> leaf
  taproot_tweak:   bytes32,
  csv_height:      u32
}
```

Wallets verify by replaying the branch txs and checking commitments against on-chain
data. Eliminates the need for full round-history sync, which is the main UX problem
in vanilla Ark.

### Arkade Assets

Native asset issuance on top of Ark:

- **Issuance**: a special intent that mints N units of a new asset against the issuer's
  VTXO. Asset metadata (name, ticker, supply schedule) is committed in a Taproot leaf.
- **Transfer**: regular VTXO intents can carry asset+amount tuples; the cosigning ceremony
  enforces conservation per asset.
- **Redemption**: holders can unilaterally exit a VTXO carrying assets; the asset state
  is preserved off-chain via the V-PACK proof.

This is conceptually similar to Taproot Assets (sparse-merkle commitment) but coupled to
the round structure rather than per-output proofs.

### Taproot-only design

ARKADE rejects pre-Taproot scripts. Every output is P2TR with key-path = MuSig2 aggregate
{ASP, user-set}, script-path = unilateral-exit branch. This means:

- Smaller witness sizes (Schnorr 64-byte sigs).
- Privacy: cooperative spends look identical regardless of round complexity.
- Compatibility issues with older wallets / nodes (requires Bitcoin Core 24+ for full
  Taproot consensus).

## Worked example

Issuing and transferring an Arkade Asset:

```
Step 1: Alice issues "ARKDOG" with 1M supply
  Intent: { issue: { ticker: "ARKDOG", supply: 1000000, decimals: 8 },
            collateral_input: alice_vtxo_50k_sat }
  Result: Alice receives a "ARKDOG-VTXO-1M" leaf in the next pool tx.

Step 2: Alice sends 100 ARKDOG to Bob
  Intent: { inputs: [alice_arkdog_1M_vtxo],
            outputs: [
              { recipient: bob_ark_addr,   asset: "ARKDOG", amount: 100 },
              { recipient: alice_ark_addr, asset: "ARKDOG", amount: 999900 }
            ] }
  Round R+1 commits a new tree; Bob now has a 100-ARKDOG VTXO.

Step 3: Bob exits unilaterally on day 25
  bark-cli unilateral-exit --vtxo bob_arkdog_100
  Wallet broadcasts the chain of pre-signed branch txs from V-PACK proof.
  After confirmations, Bob holds an on-chain Taproot UTXO carrying the
  ARKDOG asset commitment; he can re-board it later or use a TAP-aware
  bridge to move the asset elsewhere.
```

## Trade-offs and security

- **Mainnet beta**: caps on max round size, max VTXO age, and ASP collateral are tighter
  than the spec. ARKADE has not yet served million-leaf rounds in production.
- **MuSig2 fast-path**: a single uncooperative cosigner forces a round restart. ARKADE
  publishes a "drop list" so users can avoid known-bad cosigners.
- **Arkade Assets vs TAP / RGB**: an asset minted on ARKADE is *not* portable to LN or
  on-chain Taproot Assets without a bridge. Cross-protocol asset interop is still
  experimental.
- **Centralization**: in beta, Ark Labs is the sole ASP. Multi-ASP federation is on the
  roadmap but introduces inter-ASP atomic-swap problems.
- **Regulatory exposure**: native asset issuance with conservation rules (ARKDOG above)
  resembles a Liquid-like compliance surface; jurisdictions may treat issuance as a
  regulated act.

Compared to Spark, ARKADE optimises for round throughput and on-chain footprint; Spark
optimises for instant continuous transfer at the cost of an SO collective. The two
designs are complementary: Spark for "always-on" flows, Ark for batched settlement and
issuance.

## References

- ARKADE docs - https://docs.arklabs.to/
- Ark Labs blog, "Mainnet Beta" announcement (2025)
- Burak, "Ark Protocol" (bitcoin-dev archives)
- Second's bark - https://codeberg.org/ark-network/bark
