# The Testnet Difficulty Exception and Testnet5 - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/testing/testnet`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0094.mediawiki and https://github.com/bitcoin/bips/blob/master/bip-0095.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/testing/testnet/SKILL.md

## Concept

Every mined Bitcoin test network so far has carried one deliberate
deviation from mainnet: the 20-minute exception. If a block's
timestamp is more than 20 minutes past its parent's, the block may be
mined at the minimum difficulty (`nBits = 0x1d00ffff`, difficulty 1)
regardless of the network's real difficulty. It exists so that a CPU
miner can still move the chain, acquire coins, and mine non-standard
transactions that no other miner would relay.

Every failure mode of testnet3 and testnet4 traces back to that one
rule.

On testnet3 the exception fed the difficulty retarget. Retargeting
took its base value from the *last* block of the previous period; if
that block used the exception, the next period's difficulty was
pinned between 1 and 4, and an attacker could author entire subsidy
epochs in weeks. Those are the "block storms" that emptied testnet3's
faucets.

BIP94 (testnet4, Bitcoin Core 28.0, October 2024) fixed the storm
without removing the exception: the retarget base moved to the
*first* block of the previous period, and a time-warp rule was added
so the first block of a period must have `nTime >= parent nTime -
600`. Storms stopped. The exception itself did not.

What happened instead is saturation. Miners run with the clock pushed
forward and take the tip with difficulty-1 blocks, so the chain
advances by propagation race rather than by work, and exception
blocks re-org each other continuously. BIP95's own motivation section
describes the result as "constant re-orgs of small numbers of blocks
due to multiple difficulty-exception blocks competing for the tip".

BIP95 (testnet5) concludes that no exception to the PoW rules can
survive a motivated attacker, and removes it.

## Walkthrough / mechanics

The three testnet4 rules, as specified in BIP94:

1. For any block except the first of a difficulty period: if
   `nTime > parent.nTime + 20 minutes`, the block MUST use
   `nBits = 0x1d00ffff`. The first block of each period MUST use the
   real difficulty.
2. Retarget base difficulty comes from the first block of the
   previous period, not the last. The 1/4 .. 4x clamp is unchanged.
3. For a block at height ≡ 0 (mod 2016):
   `nTime >= parent.nTime - 600`, on top of the usual
   median-time-past constraint.

What testnet5 changes (BIP95, draft, `Requires: 54`, `Replaces: 94`):

| Property | testnet4 (BIP94) | testnet5 (BIP95 draft) |
|----------|------------------|------------------------|
| 20-minute exception | kept, with first-block carve-out | removed entirely |
| PoW limit (`nBits`) | `0x1d00ffff` (difficulty 1) | `0x1a0fffff` (difficulty ~1,000,000) |
| BIP54 consensus cleanup | not enforced | enforced from block 1 |
| Message start | `0x1c163f28` | `0x46495645` (ASCII `FIVE`) |
| Default p2p port | 48333 | 18335 |
| Genesis | mined, 1714777860 | not mined yet (Sept 2026) |

The raised minimum difficulty is only possible *because* the
exception is gone. BIP94 explicitly considered and rejected a higher
minimum on the grounds that it would lock CPU miners out of the
exception; with no exception to use, BIP95 records that the argument
no longer applies, and picks ~1,000,000 to damp the burst of easy
blocks right after launch while still letting "a handful of at-home
miners or a single ASIC" keep the chain moving.

BIP54 is enforced from block 1 for a reason signet cannot serve:
signet already runs BIP54 through Bitcoin Inquisition, but signet
block production is authorised by signature, so miners cannot use it
to test that their own software enforces the cleanup rules.

## Worked example

Measure how much of testnet4's tip is real work. This uses the public
mempool.space testnet4 API; the numbers below were taken on
15 September 2026.

```python
import requests

API = "https://mempool.space/testnet4/api"

tip = int(requests.get(f"{API}/blocks/tip/height", timeout=15).text)
recent = requests.get(f"{API}/v1/blocks/{tip}", timeout=15).json()

mindiff = [b for b in recent if b["difficulty"] == 1]
print(f"tip {tip}: {len(mindiff)}/{len(recent)} recent blocks at difficulty 1")

# The real difficulty of the current period is set by its first block.
period_start = tip - (tip % 2016)
h = requests.get(f"{API}/block-height/{period_start}", timeout=15).text
blk = requests.get(f"{API}/block/{h}", timeout=15).json()
print(f"period {period_start} real difficulty: {blk['difficulty']:.0f}")
```

Output on 15 September 2026:

```
tip 152566: 13/15 recent blocks at difficulty 1
period 151200 real difficulty: 1325634660
```

Thirteen of the last fifteen blocks were mined at difficulty 1
against a real network difficulty of about 1.33 billion. Anthony
Towns posted the same measurement over a whole retarget period to the
bitcoin-dev list in June 2026: 192 real-difficulty blocks against
1,824 min-difficulty blocks between heights 135,072 and 137,087,
which is roughly 90% of blocks confirmed without meaningful proof of
work.

The same check against a running node, without any third-party API:

```bash
bitcoin-cli -testnet4 getblockchaininfo | jq '.chain, .blocks, .difficulty'

TIP=$(bitcoin-cli -testnet4 getblockcount)
for h in $(seq $((TIP-14)) $TIP); do
  bitcoin-cli -testnet4 getblockheader \
    "$(bitcoin-cli -testnet4 getblockhash $h)" \
    | jq -r '"\(.height) \(.bits) \(.difficulty)"'
done
```

Blocks printing `bits` of `1d00ffff` and `difficulty` of `1` are
exception blocks. They are valid: do not "fix" your validator.

Tracking the testnet5 rollout, from primary sources only:

```bash
# BIP text and status
curl -s https://raw.githubusercontent.com/bitcoin/bips/master/bip-0095.md | head -20

# Core implementation PR (draft as of September 2026)
gh pr view 35861 --repo bitcoin/bitcoin --json state,isDraft,title,updatedAt

# The BIP 54 patchset it is stacked on
gh pr view 35793 --repo bitcoin/bitcoin --json state,isDraft,title
```

As of September 2026 bitcoin/bitcoin#35861 ("Testnet 5 (BIP95)",
fjahr, opened 1 August 2026) is still a draft, stacked on #35793
("Implement BIP 54 (Consensus Cleanup) without mainnet activation",
darosior). The PR description states it will not make the v32 feature
freeze; the plan is to settle review, mine a genesis block, start the
network outside a release, and then ship a patched v32 carrying
testnet5. There are no DNS seeds and no fixed seeds yet, and
`-testnet5` does not exist in Core master.

## Common pitfalls

- Writing tests against testnet4 that assume confirmation within a
  bounded time. The practical symptom developers report (as of
  September 2026) is faucet transactions sitting unconfirmed for
  30-60 minutes and then dropping from an almost-empty mempool.
  Use signet or regtest wherever determinism matters.
- Treating a re-org of one or two blocks on testnet4 as a network
  incident. It is the steady state.
- Hardcoding `0x1d00ffff` as "the testnet minimum difficulty".
  Testnet5 raises the PoW limit to `0x1a0fffff`.
- Assuming testnet5 will simply be a new magic number and port.
  It also enables BIP54 from block 1, so a validator that does not
  implement consensus cleanup cannot follow the chain.
- Planning a release around testnet5 availability. No genesis block
  had been mined as of September 2026.
- Assuming `-testnet` selects testnet4. It selects testnet3, whose
  config section is `[test]` and whose datadir is `testnet3`.

## References

- BIP94 (Testnet 4, Deployed): `https://github.com/bitcoin/bips/blob/master/bip-0094.mediawiki`
- BIP95 (Testnet 5, Draft): `https://github.com/bitcoin/bips/blob/master/bip-0095.md`
- BIP54 (Consensus Cleanup): `https://github.com/bitcoin/bips/blob/master/bip-0054.md`
- Core PR #35861 (Testnet 5 implementation): `https://github.com/bitcoin/bitcoin/pull/35861`
- Core PR #35793 (BIP 54 without mainnet activation): `https://github.com/bitcoin/bitcoin/pull/35793`
- Core PR #31117 (reorg testnet4 min-difficulty blocks): `https://github.com/bitcoin/bitcoin/pull/31117`
- Core issue #31975 (when to drop testnet3): `https://github.com/bitcoin/bitcoin/issues/31975`
- Block storms background: `https://blog.lopp.net/the-block-storms-of-bitcoins-testnet/`
- testnet4 explorer/API: `https://mempool.space/testnet4/`
