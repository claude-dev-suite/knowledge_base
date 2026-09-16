# bcoin Node Configuration - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/bcoin-js`.
> Canonical source: https://github.com/bcoin-org/bcoin
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/bcoin-js/SKILL.md

## Concept

`bcoin` is a JavaScript implementation of a full Bitcoin node. It can
run as a full node, an SPV node, or in pruned mode -- all from Node.js
with no native dependencies. Subsystems (chain, mempool, wallet, http,
ws) are wired together by a `FullNode` or `SPVNode` instance and
exposed via a REST + JSON-RPC HTTP server compatible with most
Bitcoin Core RPC calls.

For developers, the value is embedding a node directly in a JS app
(Electron desktop wallet, integration test harness) rather than
shelling out to `bitcoind`. Adoption is niche; for production
treasury / exchange use prefer Bitcoin Core, but for developer tooling
and JS-native apps bcoin is convenient.

> Status as of September 2026: the project is dormant. Last tagged
> release v2.2.0 (November 2021), last commit on `master` August 2023,
> last push of any kind to the repository February 2024, 114 open
> issues and 87 open pull requests. The npm package is abandoned too:
> `npm install bcoin` resolves to the deprecated `1.0.2` (July 2018)
> and no 2.x release was ever published to npm. The only supported
> install is `git clone
> https://github.com/bcoin-org/bcoin && cd bcoin && npm rebuild`.
> Everything below still describes the code on `master`, but read it
> as a reference implementation, not as a node to put in front of
> funds -- see the Taproot pitfall below.

## API walkthrough

```js
const bcoin = require("bcoin").set("main");

const node = new bcoin.FullNode({
    network: "main",
    db: "leveldb",            // or "memory"
    memory: false,
    prefix: "/var/lib/bcoin",
    workers: true,            // worker threads for verification
    listen: true,
    bip37: false,             // disable BIP37 SPV peers
    indexAddress: true,
    indexTX: true,
    httpHost: "127.0.0.1",
    httpPort: 8332,
    apiKey: process.env.BCOIN_API_KEY,
});

await node.open();
await node.connect();
node.startSync();

node.chain.on("connect", (entry, block) => {
    console.log("block", entry.height, entry.hash.toString("hex"));
});

node.mempool.on("tx", (tx) => {
    console.log("mempool tx", tx.hash().toString("hex"));
});
```

## Worked example: query height + UTXOs via the embedded HTTP API

bcoin auto-starts an HTTP server when constructed; clients in the same
process or remote processes can hit it.

```js
const { NodeClient } = require("bclient");
const Network = require("bcoin/lib/protocol/network");

const network = Network.get("main");
const client = new NodeClient({
    network: network.type,
    port: network.rpcPort,
    apiKey: process.env.BCOIN_API_KEY,
});

const info = await client.getInfo();
console.log("height:", info.chain.height);

const tx = await client.getTX("abc123...");
const utxos = await client.getCoinsByAddress("bc1q...");
console.log(utxos);
```

For SPV (BIP37) nodes -- which only download block headers + bloom-
filtered txs -- swap `FullNode` for `SPVNode` and skip the
`indexAddress`/`indexTX` flags. Note that BIP37 has known privacy
weaknesses; prefer BIP157/158 (compact filters) via Neutrino in
production.

## Common pitfalls

- Soft fork lag, now permanent: `master` never gained Taproot. There
  is no `OP_CHECKSIGADD` opcode, no taproot deployment in
  `lib/protocol/networks.js`, and `Script.verifyProgram` returns
  without error for any witness version above 0 unless
  `VERIFY_DISCOURAGE_UPGRADABLE_WITNESS_PROGRAM` is set -- so in
  blocks a v1 (P2TR) output is anyone-can-spend as far as bcoin is
  concerned (source checked September 2026). The taproot work sits
  on an unmerged `taproot` branch last touched September 2022.
  Address code does handle bech32m (BIP350), so bcoin parses P2TR
  addresses whose spends it cannot validate. Never run bcoin as the
  validating backend for mainnet.
- Mempool policy has drifted behind Core's in two concrete ways (bcoin
  `master` and Core v31.1 both read September 2026). RBF: bcoin's
  `replace-by-fee` option defaults to `false` (`docs/configuration.md`)
  and mainnet sets `requireStandard = true`, so `Mempool.insertTX`
  throws a nonstandard `replace-by-fee` error for any tx where
  `tx.isRBF()` holds -- while Core 29.0 (April 2025) removed
  `-mempoolfullrbf` altogether and made full RBF the unconditional
  standard behaviour (PR #30592). bcoin's `isRBF()` additionally returns
  `false` for every version-2 transaction whatever its input sequences,
  which is not what Core's `SignalsOptInRBF` does. Fee floor:
  `policy.MIN_RELAY` and every `network.minRelay` are 1000 sat/kB, ten
  times Core's default `-minrelaytxfee`, which dropped to 100 sat/kvB
  (along with `-incrementalrelayfee`) in Core 29.1 (September 2025) and
  30.0 (October 2025) via PR #33106. So a tx broadcast through bcoin may
  not relay through Core peers and vice-versa.
- LevelDB corruption on hard shutdown: always `await node.close()`
  before exit. Worker threads need clean teardown too.
- `apiKey` is mandatory for HTTP; without it the server exists but
  rejects every call. In dev you can still set a static key.
- BIP37 SPV: `bip37: true` opens you up to filter-based deanonymisation
  by your peers.

## References

- Repo: https://github.com/bcoin-org/bcoin
- API: https://bcoin.io/api-docs/ -- bcoin.io is a GitHub Pages site
  and was serving a mismatched `*.github.com` certificate plus a
  redirect loop when checked in September 2026; use the in-repo docs.
- Configuration: https://github.com/bcoin-org/bcoin/blob/master/docs/configuration.md
- Getting started: https://github.com/bcoin-org/bcoin/blob/master/docs/getting-started.md
- Companion: [bitcoinjs-lib/SKILL.md](../bitcoinjs-lib/SKILL.md), [../../core/operations/SKILL.md](../../core/operations/SKILL.md)
