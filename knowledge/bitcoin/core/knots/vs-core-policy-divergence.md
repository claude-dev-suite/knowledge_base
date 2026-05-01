# Knots vs Core Policy Divergence - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/knots`.
> Canonical source: https://bitcoinknots.org/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/knots/SKILL.md

## Concept

Bitcoin Knots is Luke Dashjr's downstream of Bitcoin Core. It rebases Knots-specific patches on top of each Core release and ships its own binaries with a different policy stance. Crucially, Knots does not change consensus: a Knots node and a Core node validate the same blockchain. Where they diverge is the *policy* layer (mempool, relay, miner template selection). Knots is more aggressive about rejecting non-standard transactions, particularly those carrying inscription/Ordinals data, large `OP_RETURN` payloads, or other patterns Luke considers spam. Operators choose Knots to enforce a stricter mempool at the relay level; this affects what they help propagate but not what blocks they accept.

## Walkthrough / mechanics

Bitcoin Core's `IsStandard()` and `AreInputsStandard()` functions in `src/policy/policy.cpp` define which transactions a node will accept into its mempool and forward to peers. Consensus-valid transactions that fail policy can still be mined; they just do not propagate via this node. Knots replaces or augments these functions to:

- Reject scripts containing inscription patterns (envelope opcodes used by Ordinals).
- Cap `OP_RETURN` size more aggressively (Knots preserves an 80-byte limit Core has relaxed in recent versions).
- Optionally reject transactions whose witness data exceeds heuristic thresholds.
- Provide additional CLI flags such as `-rejectparasites`, `-acceptnonstdtxn=0` defaults, and tighter mempool eviction policy.

Because consensus is unchanged, blocks containing inscription transactions are accepted by both Core and Knots. The divergence has these observable effects:

1. A user broadcasting an inscription tx to a Knots node sees it dropped at relay; the same tx broadcast to a Core node propagates normally.
2. A miner running Knots will not include inscriptions in its blocks unless an inscription tx reaches the miner via direct submission. Miners using Stratum V2 or out-of-band tx submission can still include them.
3. Mempool views via `getrawmempool` differ between Knots and Core nodes by exactly the policy delta.
4. Block templates from `getblocktemplate` differ; Knots templates exclude policy-rejected txs.

The same divergence applied historically to bare multisig, dust, and other patterns; Knots has long carried a broader and stricter `IsStandard()`.

## Worked example

Identifying a node as Knots vs Core via RPC:

```bash
$ bitcoin-cli getnetworkinfo | jq '.subversion, .version'
"/Satoshi:28.0.0/"          # Core
281000

# Or, on a Knots node:
$ bitcoin-cli getnetworkinfo | jq '.subversion'
"/Satoshi:27.0.0(knots20240801)/"
```

The user agent string distinguishes them; `subversion` includes `knots<date>` for Knots binaries.

Demonstrating mempool divergence on the same network:

```bash
# On a Core node:
$ HEX=<inscription tx hex>
$ bitcoin-cli sendrawtransaction "$HEX"
4a5e1e4baab8...

$ bitcoin-cli getmempoolinfo | jq '.size'
142853

# On a Knots node sharing peers with the Core node:
$ bitcoin-cli sendrawtransaction "$HEX"
error code: -26
error message:
non-mandatory-script-verify-flag (witness program data push exceeds policy limit)
# OR with -rejectparasites
error code: -26
error message:
inscription
```

Knots-specific `bitcoin.conf` snippet:

```ini
# Stricter mempool
permitbaremultisig=0
datacarriersize=80              # OP_RETURN cap (Knots default is 80)
acceptnonstdtxn=0
rejectparasites=1               # Knots-only: drops inscription patterns

# Same as Core for everything else
dbcache=4096
txindex=1
```

The non-Knots equivalent of `rejectparasites` does not exist in Core; setting that flag in `bitcoin.conf` on Core produces an error at startup:

```
$ bitcoind
Error: Invalid argument: -rejectparasites
```

Compare templates a miner would receive:

```bash
# Core: includes inscription txs
$ bitcoin-cli getblocktemplate '{"rules":["segwit"]}' | jq '.transactions | length'
3127

# Knots, same height, same peers: excludes them
$ bitcoin-cli getblocktemplate '{"rules":["segwit"]}' | jq '.transactions | length'
2854
```

The `coinbasevalue` differs by exactly the sum of fees on the excluded txs.

## Common pitfalls

- Treating Knots and Core as different networks. They are the same network with different relay filters. A Knots-mined block is accepted by every Core node and vice versa.
- Believing a Knots node "blocks" a tx from being mined. It only refuses to relay it. The tx can still reach miners via private mempool, sidechannels, or directly via `submitblock`.
- Configuring a Lightning node behind Knots and being surprised when commitment broadcasts work but anchor-output CPFP gets stuck. Anchor txs may be classified as non-standard by some Knots configurations; check before deploying.
- Mixing `bitcoin.conf` files between Core and Knots and being silently confused: a flag that Knots adds (e.g., `rejectparasites=1`) makes Core fail to start. Keep separate conf files when switching binaries.
- Assuming Knots's relay filter affects you when most of the network is Core. Your tx may be rejected by your local Knots node but propagate fine via any Core peer that hears about it. To enforce non-relay, every peer in the path must run Knots; in practice this is rare.
- Using the network share of Knots as a measure of consensus support. It is not. Knots and Core agree on consensus.

## References

- bitcoinknots.org for current release notes.
- `src/policy/policy.cpp` in bitcoin/bitcoin (the upstream of what Knots patches).
- Knots release notes per version for the exact policy delta.
- BIP 152 (compact blocks) for relay-layer behavior unaffected by either choice.
