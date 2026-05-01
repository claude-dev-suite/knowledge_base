# REST Endpoints Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/rest-api`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/doc/REST-interface.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/rest-api/SKILL.md

## Concept

The Bitcoin Core REST API is a small, opinionated, no-auth read-only HTTP surface that returns chain and tx data in JSON, raw binary, or hex. It exists to remove the per-request JSON-RPC overhead for high-volume read services that already control the network path (block explorers, indexers, internal monitoring). It is not a public API: there is no rate limit, no auth, no TLS. The right deployment posture is REST bound to localhost behind a reverse proxy that adds caching, rate limiting, and TLS.

## Walkthrough / mechanics

Enable with `rest=1` in `bitcoin.conf`. The endpoints share the same TCP port as JSON-RPC (default 8332). Path syntax is `/rest/<resource>/<id>.<format>` where format is `json`, `bin`, or `hex`. JSON is for human and JS clients; `bin` is for high-throughput binary readers (a tx as 200 bytes instead of 800 bytes of escaped JSON); `hex` is the same as `bin` but ASCII-encoded for tools that hate binary.

Endpoint families:

- **Tx**: `/rest/tx/<txid>.<fmt>`. Returns tx by id. Requires `txindex=1` for old txs.
- **Block**: `/rest/block/<hash>.<fmt>`, `/rest/block/notxdetails/<hash>.json`. Verbose JSON includes per-tx data; `notxdetails` returns tx ids only (much smaller).
- **Headers**: `/rest/headers/<count>/<hash>.<fmt>`. Up to 2000 headers starting at `<hash>`. Useful for SPV-style header sync.
- **Blockhash by height**: `/rest/blockhashbyheight/<height>.<fmt>`. The poor person's batch.
- **Chain info**: `/rest/chaininfo.json`. Equivalent to `getblockchaininfo`.
- **Mempool**: `/rest/mempool/info.json`, `/rest/mempool/contents.json`.
- **UTXOs**: `/rest/getutxos[/checkmempool]/<txid>-<vout>/.../.<fmt>`. Up to 15 outpoints per call. Returns hit bitmap plus matching UTXO data.

Server behavior: REST returns HTTP 503 during IBD. It returns 404 for unknown ids and 400 for malformed paths. There is no body schema for errors; the response body is a plain text message.

## Worked example

Enable and bind to localhost:

```ini
# bitcoin.conf
rest=1
rpcbind=127.0.0.1:8332
rpcallowip=127.0.0.1
```

Walk a contiguous range of headers (header-sync use case):

```bash
# Start at block 800000
$ HASH=$(curl -s 'http://127.0.0.1:8332/rest/blockhashbyheight/800000.json' | jq -r .blockhash)
$ curl -s "http://127.0.0.1:8332/rest/headers/2000/$HASH.json" | jq 'length'
2000

# Each entry includes hash, prev hash, merkleroot, time, bits, nonce, etc.
$ curl -s "http://127.0.0.1:8332/rest/headers/2000/$HASH.json" | jq '.[0]'
{
  "hash": "00000000...e7c",
  "confirmations": 40128,
  "height": 800000,
  "version": 545259520,
  "merkleroot": "...",
  "time": 1690000000,
  "previousblockhash": "...",
  "nextblockhash": "..."
}
```

Get a block as compact binary (~1 MB on the wire) and parse:

```bash
$ curl -s "http://127.0.0.1:8332/rest/block/$HASH.bin" -o /tmp/block.bin
$ ls -l /tmp/block.bin
-rw-r--r-- 1 user user 1234567 ...
$ xxd /tmp/block.bin | head -1
00000000: 0000... <serialized block bytes>
```

Get a tx (requires txindex for old):

```bash
$ TX=4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b
$ curl -s "http://127.0.0.1:8332/rest/tx/$TX.json" | jq '.vin[0], .vout[0]'
{ "txid": "...", "vout": 0, "scriptSig": {...}, "txinwitness": [...] }
{ "value": 0.05, "n": 0, "scriptPubKey": {...} }
```

Batch-check up to 15 UTXOs in one request:

```bash
$ curl -s "http://127.0.0.1:8332/rest/getutxos/checkmempool/${TX}-0/${TX}-1.json" | jq
{
  "chainHeight": 840127,
  "chaintipHash": "00000000...3ba",
  "bitmap": "11",
  "utxos": [
    { "height": 700123, "value": 0.05, "scriptPubKey": {...} },
    { "height": 700123, "value": 0.07, "scriptPubKey": {...} }
  ]
}
```

The `bitmap` is a string of `0`/`1` characters indicating which requested outpoints exist; matched ones appear in `utxos` in order.

Recommended reverse-proxy snippet (nginx) for public exposure:

```nginx
location /rest/ {
    proxy_pass http://127.0.0.1:8332;
    proxy_set_header Host $host;
    limit_req zone=bitcoin_rest burst=50 nodelay;
    proxy_cache rest_cache;
    proxy_cache_valid 200 1h;       # block data is immutable
    proxy_cache_valid 404 30s;
    add_header Cache-Control "public, max-age=3600";
}
```

## Common pitfalls

- Querying an old tx without `txindex=1`. Returns 404. The error body is `Transaction not found.` plus a hint to enable txindex.
- Calling REST during IBD. Returns 503. Always `GET /rest/chaininfo.json` first to check `initialblockdownload: false`.
- Mistaking `.bin` for `.hex`. `.bin` is raw binary (Content-Type `application/octet-stream`); `.hex` is ASCII hex. Wrong format choice produces garbled output in clients that expect text.
- Parsing the block JSON with all tx data when you only need txids. Use `/rest/block/notxdetails/<hash>.json`; the response is one or two orders of magnitude smaller.
- Trying to broadcast a tx via REST. There is no write endpoint. Use `sendrawtransaction` over RPC.
- Public exposure of REST without rate limiting. A single client downloading every block via `/rest/block/<hash>.bin` can saturate your link.
- Hitting the `getutxos` 15-outpoint limit. Above 15 you get 400. Page client-side.

## References

- `doc/REST-interface.md` in bitcoin/bitcoin.
- `src/rest.cpp` for the server implementation.
- nginx `proxy_cache` and `limit_req_zone` documentation.
