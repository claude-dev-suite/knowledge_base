# Batch and Streaming RPCs - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/rpc`.
> Canonical source: https://www.jsonrpc.org/specification#batch
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/rpc/SKILL.md

## Concept

A naive integration calls one HTTP request per RPC. Each request pays TLS/keepalive cost, JSON parsing cost, and a global lock acquisition inside `bitcoind`. The JSON-RPC 2.0 batch syntax lets a client submit an array of calls in a single body and receive a same-size array of responses. Bitcoin Core processes the batch sequentially under the same connection, which is the cheapest way to ask for thousands of small things (header chains, block hashes by height, mempool entries). For genuinely streaming workloads (push notifications) ZMQ is the answer; for pull workloads, batching plus HTTP keep-alive is the answer.

## Walkthrough / mechanics

A batch request is a JSON array of standard request objects. Each object should carry a unique `id` so the response array can be reordered by the client. Bitcoin Core does not guarantee response order matches request order across all versions, so always reconcile by `id`.

There is no Bitcoin-specific batch limit, but the HTTP request size cap (`-rpcworkqueue` queue depth, `-rpcthreads` workers, body size limit set in `httpserver.cpp`) constrains practical batch sizes. The default body limit is 32 MB, which comfortably holds tens of thousands of small calls. For very large batches split into chunks of 500 to 5000 to avoid blocking other clients on the single connection.

`bitcoind` will execute each call in the batch under the appropriate locks (`cs_main` for chain reads, the wallet lock for wallet RPCs). A batch that mixes wallet and node calls works, but every wallet entry takes the wallet lock once. If you can avoid mixing, throughput is higher.

## Worked example

Fetch hashes for 1000 contiguous heights with a single HTTP request:

```bash
START=800000
COUNT=1000
python3 - <<'PY' > /tmp/batch.json
import json
start, count = 800000, 1000
print(json.dumps([
  {"jsonrpc":"2.0","id":i,"method":"getblockhash","params":[start+i]}
  for i in range(count)
]))
PY

curl -s -u "$(cat ~/.bitcoin/.cookie)" \
  -H 'Content-Type: application/json' \
  --data @/tmp/batch.json \
  http://127.0.0.1:8332/ | jq 'length, .[0], .[999]'
```

Response shape:

```json
[
  {"result":"0000000000000000000...e7c","error":null,"id":0},
  {"result":"00000000000000000017...09b","error":null,"id":1},
  ...
]
```

Client side, always index by `id`:

```python
import json, requests
auth = open("/home/bitcoin/.bitcoin/.cookie").read().strip().split(":",1)
req = [{"jsonrpc":"2.0","id":h,"method":"getblockhash","params":[h]}
       for h in range(800000, 801000)]
r = requests.post("http://127.0.0.1:8332/",
                  auth=tuple(auth), json=req, timeout=60)
by_id = {x["id"]: x for x in r.json()}
hash_for = lambda h: by_id[h]["result"]
```

For a "streaming" walk through millions of blocks, do not batch all of them. Use a sliding window of, say, 1000 hashes per batch, then issue a follow-up batch of `getblock` calls (verbosity 1) keyed by the hash array. Two HTTP requests per 1000 blocks scales to the entire chain in minutes against a fast node.

A push-style stream is different: subscribe to ZMQ `hashblock` for new tips and only RPC for the body of the new block.

## Common pitfalls

- Sending a batch but reading responses positionally without checking `id`. A single error inside the batch shifts nothing in JSON-RPC 2.0, but a future Core release may legitimately reorder responses for parallel dispatch.
- Putting one bad call in a 10000-entry batch and getting back 9999 results plus one `error` object. The HTTP status is still 200. You must scan every entry for `error`.
- Mixing `notification` requests (no `id` field) into a batch and expecting responses for them. Per spec, notifications produce no entry in the response array.
- Hitting `-rpcworkqueue` saturation: the default is 16 with `rpcthreads=4`. A single huge batch consumes one worker for its full duration; concurrent batches plus a long IBD-era reindex can spike queue depth. Raise `rpcworkqueue=64` and `rpcthreads=8` for high-fanout services.
- Treating the Core RPC as a streaming WebSocket. It is not. For push semantics use ZMQ `rawblock`/`rawtx`/`sequence` topics.

## References

- JSON-RPC 2.0 batch spec: https://www.jsonrpc.org/specification#batch
- `src/httprpc.cpp`, `src/rpc/server.cpp` in bitcoin/bitcoin.
- `doc/JSON-RPC-interface.md` in bitcoin/bitcoin.
- `getzmqnotifications` for push alternatives.
