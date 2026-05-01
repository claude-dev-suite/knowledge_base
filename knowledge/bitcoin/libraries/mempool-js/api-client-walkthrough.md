# mempool.js API Client - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/mempool-js`.
> Canonical source: https://github.com/mempool/mempool.js
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/mempool-js/SKILL.md

## Concept

`@mempool/mempool.js` is the official TypeScript client for the
mempool.space REST + WebSocket API. The HTTP surface is a thin wrapper
over the upstream REST endpoints (e.g. `/api/v1/fees/recommended`,
`/api/address/:addr/utxo`); the library's value is type-safe wrappers,
network parameterisation (`hostname`, `network`, `liquid`), and a
ready-made WebSocket subscription helper.

For wallets it's primarily used to fetch UTXOs, fee suggestions, and
broadcast txs without running your own indexer. For explorers it adds
real-time block + mempool subscription. The same client works against
self-hosted instances by changing `hostname`.

## API walkthrough

```ts
import mempoolJS from "@mempool/mempool.js";

const init = mempoolJS({
    hostname: "mempool.space",
    network: "",                  // "" mainnet, "testnet", "signet"
});

const { addresses, fees, transactions, blocks, mempool } = init.bitcoin;

// Recommended fees
const r = await fees.getFeesRecommended();
console.log(r);
// { fastestFee: 5, halfHourFee: 4, hourFee: 3,
//   economyFee: 2, minimumFee: 1 }

// UTXOs for an address
const utxos = await addresses.getAddressTxsUtxo({
    address: "bc1qexample...",
});

// Tx detail + status
const tx = await transactions.getTx({ txid: "abc..." });
const status = await transactions.getTxStatus({ txid: "abc..." });
// { confirmed, block_height, block_hash, block_time }
```

## Worked example: select UTXOs and broadcast a tx

```ts
import mempoolJS from "@mempool/mempool.js";

interface Utxo { txid: string; vout: number; value: number; }

async function pickUtxos(addr: string, target: number): Promise<Utxo[]> {
    const { bitcoin: { addresses } } =
        mempoolJS({ hostname: "mempool.space" });
    const utxos = await addresses.getAddressTxsUtxo({ address: addr });
    const sorted = [...utxos].sort((a, b) => b.value - a.value);
    const picked: Utxo[] = [];
    let acc = 0;
    for (const u of sorted) {
        picked.push(u);
        acc += u.value;
        if (acc >= target) return picked;
    }
    throw new Error(`insufficient funds: ${acc} < ${target}`);
}

async function broadcast(rawHex: string): Promise<string> {
    const { bitcoin: { transactions } } =
        mempoolJS({ hostname: "mempool.space" });
    const txid = await transactions.postTx({ txhex: rawHex });
    return txid;
}
```

## WebSocket subscription

```ts
import mempoolJS from "@mempool/mempool.js";

const { bitcoin: { websocket } } =
    mempoolJS({ hostname: "mempool.space" });
const ws = websocket.initServer({ options: ["blocks", "mempool-blocks"] });

ws.on("message", (raw: Buffer) => {
    const data = JSON.parse(raw.toString());
    if (data.block) console.log("block", data.block.id, data.block.height);
    if (data["mempool-blocks"]) console.log("projection", data["mempool-blocks"]);
});
```

The Node WebSocket helper is `initServer`; for browsers use
`initClient`. Either way you must opt in to channels with `options`
or via post-connect `send({ action: "want", data: [...] })`.

## Common pitfalls

- Hitting the public mempool.space too hard -> rate limiting (429).
  Self-host if you need >10 req/s sustained.
- The fee recommendation reflects current mempool state; cache for at
  most ~30 s in user-facing wallets.
- Broadcast errors come back with descriptive message bodies but HTTP
  400 -- always log the response body, not just the status code.
- `value` on UTXO objects is in sats (number); be careful with values
  >= 2^53. Use `BigInt` if your project handles large balances.
- The library re-exports REST shapes; test against
  https://mempool.space/docs/api/rest before treating fields as stable.

## References

- Repo: https://github.com/mempool/mempool.js
- API docs: https://mempool.space/docs/api
- WebSocket reference: https://mempool.space/docs/api/websocket
- Companion: [../../infrastructure/mempool-space/SKILL.md](../../infrastructure/mempool-space/SKILL.md), [bitcoinjs-lib/SKILL.md](../bitcoinjs-lib/SKILL.md)
