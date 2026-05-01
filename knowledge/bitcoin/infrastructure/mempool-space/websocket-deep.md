# mempool.space WebSocket - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/mempool-space`.
> Canonical source: https://mempool.space/docs/api/websocket
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/mempool-space/SKILL.md

## Concept

The mempool.space WebSocket at `wss://mempool.space/api/v1/ws` is the
fastest way to get push notifications for new blocks, mempool events, fee
updates, and address-level activity. Unlike the REST API, where clients
poll, the WS pushes only deltas - a single block notification is a few KB
versus repeatedly fetching `/api/blocks`. This article covers the
subscription protocol, the message shapes, and a reconnect-with-resume
pattern that survives the typical 5-minute idle-timeout that load balancers
impose.

## Walkthrough / mechanics

Connect:

```js
const ws = new WebSocket("wss://mempool.space/api/v1/ws");
```

Subscribe by sending a `want` action with a list of streams:

```js
ws.send(JSON.stringify({
  action: "want",
  data: ["blocks", "mempool-blocks", "stats", "live-2h-chart"]
}));
```

Stream identifiers:

| Stream | Push payload |
|--------|--------------|
| blocks | `{block: {...}, mempoolInfo: {...}}` per new tip |
| mempool-blocks | `{mempool-blocks: [...]}` projected blocks update |
| stats | `{mempoolInfo: {size, bytes, ...}}` |
| live-2h-chart | rolling fee-rate vs time |
| loadingIndicators | sync-status data |

Address subscription (replaces previous one for that connection):

```js
ws.send(JSON.stringify({
  "track-address": "bc1qar0srrr7xfkvy5l643lydnw9re59gtzzwf5mdq"
}));
```

Multi-address tracking:

```js
ws.send(JSON.stringify({
  "track-addresses": ["bc1q...", "bc1p...", "3..."]
}));
```

Per-tx tracking (until confirmed or replaced):

```js
ws.send(JSON.stringify({ "track-tx": "<txid>" }));
```

The server responds with messages keyed by topic name:

```json
{ "block": { "id": "0000...", "height": 868235, "timestamp": 1714327000,
            "tx_count": 3214, "size": 1431234, "extras": {...} } }

{ "mempool-blocks": [
    { "blockSize": 1543210, "blockVSize": 998200,
      "nTx": 3000, "feeRange": [3, 6, 14, 28, 250], "medianFee": 14 },
    ...
] }

{ "address-transactions": [
    { "txid": "...", "fee": 1200, "vout": [...], ... }
] }
```

Heartbeat: send any message every <60s to keep the connection alive
through proxies. A common pattern is to re-send the original `want` list
every 30 seconds as a no-op refresh.

## Worked example

Browser/Node minimal client with reconnect:

```js
function connect() {
  const ws = new WebSocket("wss://mempool.space/api/v1/ws");
  let pingTimer;

  ws.onopen = () => {
    ws.send(JSON.stringify({
      action: "want",
      data: ["blocks", "mempool-blocks", "stats"]
    }));
    pingTimer = setInterval(() => {
      ws.send(JSON.stringify({ action: "ping" }));
    }, 30000);
  };

  ws.onmessage = (ev) => {
    const msg = JSON.parse(ev.data);
    if (msg.block) console.log("new block", msg.block.height);
    if (msg["mempool-blocks"])
      console.log("next block median fee",
                  msg["mempool-blocks"][0].medianFee);
    if (msg.mempoolInfo)
      console.log("mempool size", msg.mempoolInfo.size);
  };

  ws.onclose = () => {
    clearInterval(pingTimer);
    setTimeout(connect, 2000);
  };
  ws.onerror = () => ws.close();
}
connect();
```

Same pattern in Python with `websockets`:

```python
import asyncio, json, websockets

async def run():
    async for ws in websockets.connect(
        "wss://mempool.space/api/v1/ws", ping_interval=30
    ):
        try:
            await ws.send(json.dumps(
                {"action": "want", "data": ["blocks", "stats"]}
            ))
            async for raw in ws:
                msg = json.loads(raw)
                if "block" in msg:
                    print("new block", msg["block"]["height"])
        except websockets.ConnectionClosed:
            continue

asyncio.run(run())
```

| Reconnect signal | Action |
|------------------|--------|
| `close` event | wait 2s, reconnect, re-subscribe |
| Error event | force-close, then reconnect |
| 60s of silence | send `{"action":"ping"}` |
| `track-address` reply missing for 10s | re-issue `track-address` |

## Common pitfalls

- Forgetting to re-subscribe after reconnect - the server has no session
  state across socket lifetimes, all `want` and `track-*` messages must be
  re-sent on every new connection.
- One address per `track-address` - a second `track-address` replaces the
  first; use `track-addresses` (plural) for multi-address watching.
- Treating `mempool-blocks` updates as block-confirm signals - they fire on
  any mempool change, including 1-tx differences; gate UI updates by the
  `block` event.
- Idle timeouts in cloud load balancers (AWS ALB defaults to 60s; Cloudflare
  ~100s) - send ping payloads or a no-op `want` every 30s.
- Self-hosted CORS - the bundled nginx upgrades `wss://` correctly but
  reverse proxies often miss the `Upgrade` and `Connection` headers; verify
  with `curl -I -H "Connection: Upgrade" -H "Upgrade: websocket" ...`.

## References

- WebSocket docs: https://mempool.space/docs/api/websocket
- mempool-js: https://github.com/mempool/mempool.js
- Source: https://github.com/mempool/mempool/blob/master/backend/src/api/websocket-handler.ts
