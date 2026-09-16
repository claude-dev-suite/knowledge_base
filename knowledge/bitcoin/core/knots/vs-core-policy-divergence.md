# Knots vs Core Policy Divergence - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/knots`.
> Canonical source: https://bitcoinknots.org/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/knots/SKILL.md

## Concept

Bitcoin Knots is Luke Dashjr's downstream of Bitcoin Core. It rebases Knots-specific patches on top of each Core release and ships its own binaries with a different policy stance. Knots is more aggressive about rejecting non-standard transactions, particularly those carrying inscription/Ordinals data, large `OP_RETURN` payloads, or other patterns Luke considers spam.

Through `v29.3.knots20260210` (2026-02-10) this divergence was *policy only*: a Knots node and a Core node validated the same blockchain, and differed at the mempool, relay and miner-template layers. Operators chose Knots to enforce a stricter mempool at the relay level; this affected what they helped propagate but not what blocks they accepted. That policy delta still holds.

**It no longer describes the whole relationship.** Two consensus changes shipped in 2026 — BIP-110 (RDTS) in `v29.3.knots20260508` and a BLAKE2b proof-of-work hardfork in `v29.4.1.knots20260508` — and since the chain split of 2026-08-08 the current Knots release line follows a different chain from Bitcoin Core. See [Consensus divergence since 2026](#consensus-divergence-since-2026) below.

## Walkthrough / mechanics

Bitcoin Core's `IsStandard()` and `AreInputsStandard()` functions in `src/policy/policy.cpp` define which transactions a node will accept into its mempool and forward to peers. Consensus-valid transactions that fail policy can still be mined; they just do not propagate via this node. Knots replaces or augments these functions to:

- Reject scripts containing inscription patterns (envelope opcodes used by Ordinals).
- Cap `OP_RETURN` size more aggressively: `MAX_OP_RETURN_RELAY` is 83 bytes of `scriptPubKey` (an 80-byte payload plus 3 bytes of opcode/push overhead) at the `v29.4.1.knots20260508` tag, and only one nulldata output per transaction is relayed (`multi-op-return` reject in `src/policy/policy.cpp`); a tx carrying nothing but datacarrier outputs is rejected as `bare-datacarrier` unless `-permitbaredatacarrier=1`. Bitcoin Core 30.0 (October 2025) raised its own `-datacarriersize` default from 83 to 100,000 and began relaying multiple nulldata outputs per tx, so this gap widened rather than closed — see the "Relay policy since 30.0" section of the `bitcoin/core/operations` skill for the Core side. Knots' 83 is itself a raise from 42, made in `v29.2.knots20251110` (2025-11-10) and described in those release notes as temporary, to be "reverted back to 42 in a future version".
- Optionally reject transactions whose witness data exceeds heuristic thresholds.
- Provide additional CLI flags such as `-rejectparasites`, `-acceptnonstdtxn=0` defaults, and tighter mempool eviction policy. These are no longer opt-in: `src/policy/policy.h` at the `v29.4.1.knots20260508` tag has `DEFAULT_REJECT_PARASITES{true}`, `DEFAULT_REJECT_TOKENS{true}`, `DEFAULT_PERMIT_BAREMULTISIG{false}` and `DEFAULT_PERMITBAREDATACARRIER{false}`.

While consensus was unchanged (pre-2026 releases), blocks containing inscription transactions were accepted by both Core and Knots. The policy divergence has these observable effects:

1. A user broadcasting an inscription tx to a Knots node sees it dropped at relay; the same tx broadcast to a Core node propagates normally.
2. A miner running Knots will not include inscriptions in its blocks unless an inscription tx reaches the miner via direct submission. Miners using Stratum V2 or out-of-band tx submission can still include them.
3. Mempool views via `getrawmempool` differ between Knots and Core nodes by exactly the policy delta.
4. Block templates from `getblocktemplate` differ; Knots templates exclude policy-rejected txs.

The same divergence applied historically to bare multisig, dust, and other patterns; Knots has long carried a broader and stricter `IsStandard()`.

## Worked example

The `getrawmempool` and `getblocktemplate` comparisons below isolate the policy delta only against an RDTS-free Knots build (up to `v29.3.knots20260210`, or `v29.3.knots20260507`). A `v29.4.x` node is not building on the same chain tip — see [Consensus divergence since 2026](#consensus-divergence-since-2026).

Identifying a node as Knots vs Core via RPC:

```bash
$ bitcoin-cli getnetworkinfo | jq '.subversion, .version'
"/Satoshi:31.1.0/"          # Core v31.1 (2026-07-08)
310100

# Or, on a Knots node:
$ bitcoin-cli getnetworkinfo | jq '.subversion'
"/Satoshi:29.4.1/Knots:20260508/"
```

The user agent string distinguishes them. Knots builds have long appended a separate `Knots:<date>/` segment after the base-version field — `FormatSubVersion()` in `src/clientversion.cpp` emits it as far back as `v21.2.knots20210629`, and a 2024-era build reports `"/Satoshi:27.1.0/Knots:20240801/"` (18 such nodes were still reachable in the 2026-09-15 Bitnodes snapshot). Match on the `Knots:` segment, not on the base version: that snapshot carries Knots nodes on ten different base versions, and the parenthesised field before the segment is an operator-supplied comment, not the Knots date.

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
datacarriersize=83              # OP_RETURN cap in scriptPubKey bytes (Knots default since v29.2.knots20251110)
acceptnonstdtxn=0
rejectparasites=1               # Knots-only: drops inscription patterns

# Same as Core for everything else
dbcache=4096
txindex=1
```

Most of that snippet is now redundant: at `v29.4.1.knots20260508` these are the shipped defaults (`DEFAULT_PERMIT_BAREMULTISIG{false}`, `DEFAULT_REJECT_PARASITES{true}`, `MAX_OP_RETURN_RELAY = 83`), so setting them explicitly documents intent rather than changing behavior. Later releases added more defaults-on filters worth knowing about:

```ini
# All default-on at v29.4.1.knots20260508 - set these only to *disable* them
subdustfeepenalty=0             # added default-on in v29.3.knots20260508 (knots#272)
rejecttokens=0                  # flipped to default-on in v29.4.1; also detects Counterparty
datacarrierfullcount=0          # applies the datacarriersize limit to all datacarrier methods
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

## Consensus divergence since 2026

Everything above is the policy layer and is still accurate as a description of it. Separately, Knots acquired consensus changes during 2026, and the two implementations now validate different chains.

**BIP-110, "Reduced Data Temporary Softfork" (RDTS).** Shipped in `v29.3.knots20260508` (2026-05-09) as a modified BIP9 deployment (knots#238). It limits data fields at the consensus level for a one-year window: new output scriptPubKeys over 34 bytes are invalid unless they start with `OP_RETURN` (then 83 bytes), pushdata payloads and script-argument witness items over 256 bytes are invalid, spending undefined witness/Tapleaf versions is invalid, Taproot annexes are invalid, and Tapscripts containing `OP_SUCCESS*` or executing `OP_IF`/`OP_NOTIF` are invalid. UTXOs created before activation are grandfathered and remain spendable. Deployment parameters: bit 4, starttime 1764547200 (~2025-12-01), threshold 1109/2016 (55%), `max_activation_height` 965664 (~2026-09-01), `active_duration` 52416 blocks. Enforcement in `v29.3.knots20260508` was opt-in — `consensusrules=rdts` in `bitcoin.conf` or a GUI confirmation — and a non-RDTS build of the same version shipped as `v29.3.knots20260507`. `v29.4.1` removed that consent requirement (knots#362).

**The 2026-08-08 split.** BIP-110's mandatory-signaling window covers blocks 961632-963647, during which blocks not signaling bit 4 are rejected as invalid. AntPool mined the first non-signaling block at height 961,632; the main network accepted it and BIP-110 nodes rejected it, following instead a block mined via Ocean. That minority chain produced two blocks in roughly eight hours while the main chain advanced 48, then stalled — it had inherited the main chain's difficulty with a fraction of its hashrate. Only 2.53% of blocks had signaled bit 4 over the preceding two weeks, against the 55% threshold. BIP-110 was marked **Closed** in the BIPs repo on 2026-08-09, "following a chain split with stalled mining".

**The BLAKE2b hardfork.** `v29.4.1.knots20260508` (2026-09-02) mitigates the stall with an explicitly backward-incompatible change. Its release notes list: a BLAKE2b proof-of-work algorithm (knots#359, knots#385), a temporary 800 kWU block weight limit (~300 kB serialized), RDTS activation moved to the PoW-change flag day (knots#358), unified opt-in sighash for all transaction types (knots#357), and fixes for CVE-2013-2292, CVE-2020-14199 and CVE-2017-12842. Per bitcoinknots.org's BLAKE2b page, the first BLAKE2b block is 961,640 (2026-08-30) and the data limits run to 2027-09-01; SHA256d miners cannot add blocks to that chain. The release also adds a `NODE_BLAKE2B` service bit, preferred over `NODE_REDUCED_DATA` for peer selection (knots#368), and exposes new header fields (`header_version`, `nonce2`, `nonce3`, `extranonce`, `xor_key`, `mm_rhs` and others) via `getblockheader` / `getblock`.

Practical consequence for anything in the "Walkthrough" section above: comparing `getrawmempool` or `getblocktemplate` between a `v29.4.x` Knots node and a Core node no longer isolates the policy delta, because the two are not building on the same chain tip. To reproduce the policy comparison, use an RDTS-free build: anything up to and including `v29.3.knots20260210`, or the non-RDTS `v29.3.knots20260507`.

## Common pitfalls

- Assuming Knots and Core are still the same network. Through `v29.3.knots20260210` they were: same network, different relay filters, and a Knots-mined block was accepted by every Core node and vice versa. Since the 2026-08-08 split and the BLAKE2b hardfork in `v29.4.1.knots20260508` that is no longer true of current releases — check the build string before assuming either way.
- Believing an RDTS-free Knots node "blocks" a tx from being mined. It only refuses to relay it. The tx can still reach miners via private mempool, sidechannels, or directly via `submitblock`.
- Configuring a Lightning node behind Knots and being surprised when commitment broadcasts work but anchor-output CPFP gets stuck. Anchor txs may be classified as non-standard by some Knots configurations; check before deploying.
- Mixing `bitcoin.conf` files between Core and Knots and being silently confused: a flag that Knots adds (e.g., `rejectparasites=1`) makes Core fail to start. Keep separate conf files when switching binaries.
- Assuming Knots's relay filter affects you when most of the network is Core. Your tx may be rejected by your local Knots node but propagate fine via any Core peer that hears about it. To enforce non-relay, every peer in the path must run Knots; in practice this is rare.
- Using the node share of Knots as a measure of consensus support. It is not. Node counts are not hashrate, and since August 2026 the trackers bucket by user agent, so Knots nodes on the BLAKE2b chain and Core nodes on the SHA256d chain land in the same total despite validating different chains. As of 2026-09-15 Coin Dance reports Knots at 4,399 of 25,864 public nodes (~17.0%) and Core at 21,431 (~82.8%); a Bitnodes snapshot the same day counts 4,420 Knots agents of 26,478 reachable nodes (~16.7%). The BIP-110 chain split showed how little this predicts: 2.53% of blocks signaled.
- Upgrading a Knots deployment without reading which consensus rules the new build carries. Builds are identified by their date suffix, not their base version: everything up to and including `v29.3.knots20260210`, plus the non-RDTS `v29.3.knots20260507`, follows Core's chain; `v29.3.knots20260508` enforces RDTS if consented to; `v29.4.1.knots20260508` follows the BLAKE2b chain unconditionally. The `v29.4.1` notes warn a node that was "old, pruned, and followed invalid blocks" may need a full resync.

## References

- bitcoinknots.org for current release notes.
- `src/policy/policy.cpp` in bitcoin/bitcoin (the upstream of what Knots patches).
- `src/policy/policy.h` at `bitcoinknots/bitcoin` tag `v29.4.1.knots20260508` for the current Knots policy defaults.
- `doc/release-notes.md` at `bitcoin/bitcoin` tag `v30.0` for the 100,000-byte `-datacarriersize` default and multiple-OP_RETURN relay (#32406).
- `src/clientversion.cpp` at any `bitcoinknots/bitcoin` tag for the subversion format `FormatSubVersion()` builds.
- Knots release notes per version for the exact policy delta - `doc/release-notes.md` at the `v29.3.knots20260508` and `v29.4.1.knots20260508` tags cover the 2026 consensus changes.
- BIP 110 (`bip-0110.mediawiki` in bitcoin/bips) for the RDTS rules, deployment parameters and the Closed-status changelog entry.
- https://bitcoin-blake2b.org/ (redirect target of bitcoinknots.org/learn/2026-blake2b) for the BLAKE2b flag-day height and data-limit expiry.
- BIP 152 (compact blocks) for relay-layer behavior unaffected by either choice.
