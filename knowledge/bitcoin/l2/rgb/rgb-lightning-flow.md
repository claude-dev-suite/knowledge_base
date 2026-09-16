# RGB Lightning Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/rgb`.
> Canonical source: https://github.com/RGB-Tools/rgb-lightning-node and Bitlight Labs docs
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/rgb/SKILL.md

## Concept

`rgb-lightning-node` (RLN) is an LDK fork that carries RGB asset state through Lightning
channels. Each commitment transaction includes an extra anchor output committing to the
post-update RGB state via Tapret. Asset transfers route via standard HTLCs whose payment
amounts are interpreted as asset units rather than (or in addition to) sats.

As of September 2026 RLN has **no tagged release**: development happens on `master`,
whose head commit is 26 August 2026. It vendors rust-lightning 0.2.x and declares
`rgb-lib` `=0.3.0-beta.7` (17 July 2026), though a `[patch.crates-io]` entry redirects
that dependency to `rgb-lib` git `master`, so the build floats rather than pinning.
`rgb-lib` 0.3.0-beta.7 in turn pins `rgb-ops`, `rgb-invoicing`, `rgb-psbt-utils` and
`rgb-schemas` at `=0.11.1-rc.11` (15 July 2026). RLN therefore rides the
`rgb-protocol` **v0.11.1 line**, not RGB-WG's v0.12 consensus release -- the line
RGB-WG advised against for production in its 18 July 2025 security notice.
USDT-on-RGB rides this same v0.11.1 line and is still pre-launch: no public
mainnet launch announcement had appeared as of mid-September 2026.

## Walkthrough / mechanics

### Channel funding with assets

```
Funding tx (multisig 2-of-2 between Alice, Bob):
  IN:  Alice's RGB UTXO (1000 USDT-RGB)
       Bob's BTC UTXO (channel reserve sats)
  OUT: 2-of-2 multisig output  (channel anchor)
        with Tapret commitment to initial RGB state:
        { Alice_balance: 1000 USDT, Bob_balance: 0 USDT }
```

The funding tx itself is an RGB transition. State data goes off-chain; on chain we see
a Taproot output that carries the Tapret tweak.

### Commitment txs

Each channel update produces a new asymmetric commitment tx (one per party). Compared to
vanilla LDK:

- Output 1: to_local (claimable by holder after timelock).
- Output 2: to_remote (immediate to other party).
- **Output 3 (RGB anchor)**: zero-sat dust output with Tapret tweak committing to the
  new RGB state assignment.
- HTLCs: standard, but the HTLC value carries asset units in the *RGB transition state*
  (not in the on-chain sat amount, which stays minimal).

### HTLC for asset transfer

```
Alice -> Bob HTLC for 50 USDT:
  channel_state.alice_usdt -= 50
  channel_state.bob_usdt   += 50  (pending HTLC)
  Anchor commitment encodes the new state.
  payment_hash = H, preimage = R as in vanilla LN.
```

When R is revealed:
- Both parties update commitment to a new state with the HTLC settled.
- Asset balance moves Alice -> Bob.

If the HTLC times out:
- Both parties update to a state that returns the asset to Alice.

### Multi-hop with RGB

Currently more constrained than TAP. RLN supports:
- Same-asset routing through nodes that all hold the same RGB schema.
- Atomic swap-style cross-asset: less mature than TAP's RFQ.

Swap routing over private channels was fixed in April 2026, and a
`decode_swapstring` endpoint was added in May 2026.

The expected production path involves Bitlight Labs and other
infrastructure-providers running RGB-aware liquidity hubs.

### On-chain close

Cooperative close:
- Final state is committed to a closing tx.
- Tapret tweak in the closing tx output(s) reflects final asset assignments.
- Both parties get on-chain UTXOs carrying their post-channel asset balance.

Unilateral close:
- The closing party broadcasts their commitment tx (with the RGB anchor output).
- Funds are revocable / claimable per LDK rules; assets follow the to_local / to_remote
  output's seal.

## Worked example

Alice and Bob open a channel; Alice loads 1000 USDT-RGB; Alice pays Bob 60 USDT in two
HTLCs:

```
Day 0  Funding tx:
       IN   Alice USDT UTXO (1000 USDT) + Alice 100k sat funding
       OUT  Channel multisig, value 100k sats, Tapret = H(state_v0)
            state_v0 = { alice: 1000 USDT, bob: 0 USDT }

Day 1  HTLC #1: Alice -> Bob 40 USDT
       Pending state: { alice: 960, bob: 0, htlc: 40@H1 }
       Bob settles via R1.  New state: { alice: 960, bob: 40 }.
       Both sides exchange revocation keys for prior state.

Day 2  HTLC #2: Alice -> Bob 20 USDT
       Pending: { alice: 940, bob: 40, htlc: 20@H2 }
       Bob settles. State: { alice: 940, bob: 60 }.

Day 30 Cooperative close.
       closing_tx outputs:
         OUT 1: 95k sat to Alice (sat reserve - fees)
         OUT 2: 0-sat-dust Tapret commit to alice_seal { 940 USDT }
         OUT 3: 0-sat-dust Tapret commit to bob_seal   { 60  USDT }
       Each party now has an on-chain RGB UTXO carrying their share.
```

## Trade-offs and security

- **Proof bundle size**: every channel update advances the RGB state graph. Closing
  parties must hand each other (and any future receiver) the full transition proof
  chain since channel open. Bundles can grow in long-lived channels.
- **State desync risk**: if RGB state and LDK state get out of sync (e.g., due to a
  buggy persist), a unilateral close may attempt to publish a state-prev commitment
  which the counterparty can punish. Critical to atomically persist both layers.
- **Watchtower coverage**: LDK watchtowers must be RGB-aware to detect equivocation in
  the RGB state. Adoption is still early.
- **Liquidity fragmentation**: each asset/schema combo is a separate liquidity surface.
  Cross-asset routing inside RGB is harder than in TAP today.
- **Wallet dependency on Bitlight infra**: RLN production deployments largely rely on
  Bitlight Labs's testing nodes. More implementations needed for decentralisation.
- **No release tags**: integrators pin a git commit, not a version. As of September
  2026 the repo has never cut a release.
- **Compared to TAP/LND**: RLN is more flexible (full AluVM contracts) but less
  battle-tested; TAP is locked to fungible balance tracking but more mature.

## Recent RLN work (2026)

Verified from `master` commit history (no tagged releases exist):

| Date | Change |
|------|--------|
| Mar 2026 | zero-amount RGB payment amount in `sendpayment` |
| Mar 2026 | `push_asset_amount` when opening RGB channels |
| Mar 2026 | payment preimages exposed in payment APIs |
| Mar 2026 | support for inflatable fungible assets (IFA) |
| Apr 2026 | swap routing over private channels; custom signet support |
| Aug 2026 | BOLT11 `description` / `description_hash` on invoice APIs |
| Aug 2026 | optional `bitcoind` via `lightning-transaction-sync` backend |

No BOLT12 / offers support has landed as of September 2026.

## References

- rgb-lightning-node - https://github.com/RGB-Tools/rgb-lightning-node
- rgb-lib - https://github.com/RGB-Tools/rgb-lib
- Bitlight Labs blog - https://bitlightlabs.com/blog
- RGB v0.12 consensus release (10 July 2025) - https://rgb.tech/blog/release-v0-12-consensus/
- RGB-WG notice on the v0.11.1 fork (18 July 2025) - https://rgb.tech/blog/on-rgb-fork-by-bitfinex/
- LDK - https://github.com/lightningdevkit/rust-lightning
