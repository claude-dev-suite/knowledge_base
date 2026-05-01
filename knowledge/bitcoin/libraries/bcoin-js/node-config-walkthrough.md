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

- Soft fork lag: bcoin sometimes lacks support for a new soft fork
  (BIP-numbers) for weeks/months after Core ships. Verify via
  `getnetworkinfo` softfork status before depending on a feature.
- Mempool policy differs from Core defaults (min relay fee, RBF
  signalling). A tx broadcast through bcoin may not relay through Core
  peers and vice-versa.
- LevelDB corruption on hard shutdown: always `await node.close()`
  before exit. Worker threads need clean teardown too.
- `apiKey` is mandatory for HTTP; without it the server exists but
  rejects every call. In dev you can still set a static key.
- BIP37 SPV: `bip37: true` opens you up to filter-based deanonymisation
  by your peers.

## References

- Repo: https://github.com/bcoin-org/bcoin
- API: https://bcoin.io/api-docs/
- Configuration guide: https://bcoin.io/guides/config.html
- Companion: [bitcoinjs-lib/SKILL.md](../bitcoinjs-lib/SKILL.md), [../../core/operations/SKILL.md](../../core/operations/SKILL.md)
