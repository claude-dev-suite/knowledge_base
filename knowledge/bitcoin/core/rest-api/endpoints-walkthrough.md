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
- **Block part**: `/rest/blockpart/<hash>.<bin|hex>?offset=<o>&size=<n>`. A byte range inside a serialized block, without transferring the block. Added in 31.0 (April 2026); 404 if the block or the byte range doesn't exist.
- **Headers**: `/rest/headers/<hash>.<fmt>?count=<count>`. Up to 2000 headers starting at `<hash>`, walking upward. Useful for SPV-style header sync. `count` defaults to **5** when omitted, so a caller that forgets it gets a short list rather than an error. The older path form `/rest/headers/<count>/<hash>.<fmt>` has been deprecated (but not removed) since 24.0 and still works as of 31.1 (July 2026).
- **Block filters**: `/rest/blockfilterheaders/<type>/<hash>.<fmt>?count=<count>` (same 2000 cap, same default of 5, same deprecated `<count>`-in-path form) and `/rest/blockfilter/<type>/<hash>.<fmt>` for a single block's BIP158 filter. `basic` is the only filter type Core implements; both endpoints answer 400 `Index is not enabled for filtertype basic` unless `blockfilterindex=1`.
- **Blockhash by height**: `/rest/blockhashbyheight/<height>.<fmt>`. The poor person's batch.
- **Spent tx outputs**: `/rest/spenttxouts/<hash>.<fmt>`. One list of spent prevouts per transaction in the block, read from undo data. Added in 30.0 (October 2025). Through 31.1 (July 2026) and `v32.0rc1` each prevout carries only `value` and `scriptPubKey`, so it is *not* the same shape as a `getblock` verbosity 3 prevout, which also carries `generated` and `height`. Master adds those two fields (landed 14 September 2026, after the `v32.0rc1` tag; unreleased as of 15 September 2026). 404 if the block doesn't exist or its undo data is not available.
- **Chain info**: `/rest/chaininfo.json`. Equivalent to `getblockchaininfo`.
- **Deployment info**: `/rest/deploymentinfo.json`, `/rest/deploymentinfo/<blockhash>.json`. Equivalent to `getdeploymentinfo`, at the tip or at a given block. JSON only.
- **Mempool**: `/rest/mempool/info.json`, `/rest/mempool/contents.json?verbose=<true|false>&mempool_sequence=<false|true>`.
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
$ curl -s "http://127.0.0.1:8332/rest/headers/$HASH.json?count=2000" | jq 'length'
2000

# Drop the query parameter and you get 5, not 2000, and not an error
$ curl -s "http://127.0.0.1:8332/rest/headers/$HASH.json" | jq 'length'
5

# Each entry includes hash, prev hash, merkleroot, time, bits, nonce, etc.
$ curl -s "http://127.0.0.1:8332/rest/headers/$HASH.json?count=2000" | jq '.[0]'
{
  "hash": "00000000...e7c",
  "confirmations": 167259,
  "height": 800000,
  "version": 545259520,
  "merkleroot": "...",
  "time": 1690000000,
  "previousblockhash": "...",
  "nextblockhash": "..."
}
```

Height-derived numbers in the sample outputs are a snapshot at chain tip 967258
(16 September 2026); only their shape is stable.

Get a block as compact binary (~1 MB on the wire) and parse:

```bash
$ curl -s "http://127.0.0.1:8332/rest/block/$HASH.bin" -o /tmp/block.bin
$ ls -l /tmp/block.bin
-rw-r--r-- 1 user user 1234567 ...
$ xxd /tmp/block.bin | head -1
00000000: 0000... <serialized block bytes>
```

Or pull just the bytes you need, instead of the whole megabyte (31.0 and later):

```bash
# The 80-byte block header only
$ curl -s "http://127.0.0.1:8332/rest/blockpart/$HASH.hex?offset=0&size=80"
<160 hex chars>
```

List every prevout the block spent, without `txindex` and without refetching
each parent transaction (30.0 and later):

```bash
$ curl -s "http://127.0.0.1:8332/rest/spenttxouts/$HASH.json" | jq '.[1][0]'
{ "value": 0.05, "scriptPubKey": {...} }
```

The outer array has one entry per transaction in the block, in block order, so
index 0 is the coinbase (an empty list) and the first real spend is at index 1.
On master (unreleased as of 15 September 2026) the same object also carries
`generated` and `height`; through 31.1 (July 2026) and `v32.0rc1` it does not.

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
  "chainHeight": 967258,
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

Note for the next major: through 31.1 (July 2026) Core sends no `Cache-Control`
of its own, so the proxy is the only cache policy. Master and `v32.0rc1`
(tagged 14 September 2026; 32.0 not yet released as of 15 September 2026) add
default response headers - `public, immutable, max-age=86400` for the
immutable-by-construction responses (`/block` and `/block/notxdetails` in bin
and hex, `/blockpart`, `/blockfilter`, `/spenttxouts`,
`/deploymentinfo/<blockhash>.json`) and `no-store` for everything else,
including all JSON block responses, `/tx`, `/headers`, `/chaininfo`,
`/mempool`, `/getutxos` and every error. Those responses carry no `ETag` or
`Last-Modified`, so an overriding proxy rule is the only way to cache them.

## Hardening and connection limits

REST is not a separate server. It is handled by the same HTTP server, on the
same port, as JSON-RPC, so the HTTP-server settings apply to REST traffic and
the two compete for one pool of connections:

- `-rpcmaxconnections=<n>` caps simultaneously connected HTTP clients, RPC and
  REST together, at **16** by default. It is new in 32.0, which rewrote the
  HTTP server from scratch to replace libevent (#35182); through 31.1
  (July 2026) there is no connection cap at all.
- `-rpcworkqueue=<n>` (default 64) and `-rpcthreads=<n>` (default 16) bound
  queued and in-flight requests. Both are present in 31.1 too.
- `-rpcallowip` gates REST as well. From 32.0 a client connecting from an
  address the option does not allow is disconnected immediately.

The concrete reason to set that cap deliberately is PR #36123, merged
5 September 2026. Between the HTTP-server rewrite and that fix, master kept
reading pipelined data from a connection whose previous request was still being
processed, so a client could grow server memory without bound - remote OOM. The
fix stops selecting the read event for a busy connection whose receive buffer is
non-empty, and lets TCP backpressure hold the sender off. A reviewer measured
sixteen unauthenticated REST connections - exactly the default cap - moving RSS
by 3.2 GB over 90 seconds on the pre-fix branch, against 3 MB after it. #36123
is an ancestor of the `v32.0rc1` tag (14 September 2026), so the regression
never shipped in a release, and it does not apply to the libevent server used
through 31.1. The cap bounds per-connection buffering; it does not remove it.

## Common pitfalls

- Querying an old tx without `txindex=1`. Returns 404. The error body is `Transaction not found.` plus a hint to enable txindex.
- Calling REST during IBD. Returns 503. Always `GET /rest/chaininfo.json` first to check `initialblockdownload: false`.
- Mistaking `.bin` for `.hex`. `.bin` is raw binary (Content-Type `application/octet-stream`); `.hex` is ASCII hex. Wrong format choice produces garbled output in clients that expect text.
- Parsing the block JSON with all tx data when you only need txids. Use `/rest/block/notxdetails/<hash>.json`; the response is one or two orders of magnitude smaller.
- Trying to broadcast a tx via REST. There is no write endpoint. Use `sendrawtransaction` over RPC.
- Public exposure of REST without rate limiting. A single client downloading every block via `/rest/block/<hash>.bin` can saturate your link.
- Hitting the `getutxos` 15-outpoint limit. Above 15 you get 400. Page client-side.
- Letting a REST crawler eat your RPC connection slots. From 32.0 `-rpcmaxconnections` (default 16) counts RPC and REST clients together, so an unthrottled REST consumer can starve the RPC callers on the same node.
- Omitting `?count=` on `/rest/headers/<hash>.<fmt>`. You get 5 headers and a 200, not an error - the quietest bug on this surface. `count` outside 1..2000 is a 400 (`Header count is invalid or out of acceptable range (1-2000)`), not a clamp.
- Still writing the old `/rest/headers/<count>/<hash>` path form. Deprecated (but not removed) since 24.0, and still working as of 31.1 (July 2026); new code should not add more callers.
- Calling `/rest/blockfilter*` on a node without `blockfilterindex=1`. 400 `Index is not enabled for filtertype basic`, which reads like a bad request rather than a missing index.
- Refetching parent transactions to learn what a block spent. Use `/rest/spenttxouts/<hash>.json` (30.0 and later); it needs undo data, not `txindex`. Its outer array is offset by the coinbase placeholder at index 0, so transaction *n* of the block is at index *n*.

## References

- `doc/REST-interface.md` in bitcoin/bitcoin.
- `src/rest.cpp` for the server implementation.
- nginx `proxy_cache` and `limit_req_zone` documentation.
