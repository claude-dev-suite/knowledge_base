# ZMQ Notification Types - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/zmq`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/doc/zmq.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/zmq/SKILL.md

## Concept

Bitcoin Core publishes five ZMQ topics, each with a different tradeoff between latency, payload size, and information density. `rawblock` and `rawtx` give you complete bytes, useful but heavy; `hashblock` and `hashtx` give you a 32-byte identifier, useful but you must round-trip an RPC to learn anything; `sequence` is the only topic with state-transition semantics (mempool add, mempool remove, block connect, block disconnect) and a monotonic counter that lets a client detect drops. Picking the right topic per use case avoids both wasted bandwidth and missed events. Mixing them with a per-topic `sequence` reconciliation is the production-grade pattern.

## Walkthrough / mechanics

Each topic is independently subscribable via `zmqpub<topic>=tcp://...`. Per-message format is a three-part ZMQ frame:

1. Topic name as ASCII bytes.
2. Payload (varies by topic).
3. 4-byte little-endian message counter scoped to (topic, publisher); resets on bitcoind restart.

Frame 3 lets a subscriber detect drops at the transport layer. ZMQ TCP itself does not lose messages on a healthy connection, but a slow consumer plus default unbounded HWM (high water mark) buffers cause out-of-memory pressure on the publisher side; setting a finite HWM means dropped frames you must detect.

Payload by topic:

| Topic | Payload bytes | Triggered by |
|---|---|---|
| `rawblock` | full serialized block (varies, often >1 MB) | every new tip, including reorg replays |
| `rawtx` | full serialized tx | mempool entry + every block inclusion |
| `hashblock` | 32 raw bytes (block hash little-endian on the wire) | every new tip |
| `hashtx` | 32 raw bytes (txid little-endian) | mempool entry + every block inclusion |
| `sequence` | 32 bytes hash + 1 byte status + (8 bytes mempool seq if A/R) | mempool churn + block connect/disconnect |

`sequence` status byte values are ASCII characters: `'A'` (mempool add), `'R'` (mempool remove, includes both eviction and confirmation), `'C'` (block connect), `'D'` (block disconnect, reorg). For `'A'` and `'R'` the trailing 8 bytes are a per-mempool monotonic counter the client can use to detect missed mempool transitions independently of the per-topic frame counter.

Notable: `rawtx` fires twice per confirmed transaction (once on mempool entry, once on block inclusion). `rawblock` does NOT fire on reorg-removed blocks, only on the new tip after the reorg, so reorg detection requires `sequence`.

## Worked example

Configure all five topics:

```ini
# bitcoin.conf
zmqpubrawblock=tcp://127.0.0.1:28332
zmqpubrawtx=tcp://127.0.0.1:28333
zmqpubhashblock=tcp://127.0.0.1:28334
zmqpubhashtx=tcp://127.0.0.1:28335
zmqpubsequence=tcp://127.0.0.1:28336
```

Verify Core registered them:

```bash
$ bitcoin-cli getzmqnotifications
[
  {"type":"pubrawblock","address":"tcp://127.0.0.1:28332","hwm":1000},
  {"type":"pubrawtx","address":"tcp://127.0.0.1:28333","hwm":1000},
  {"type":"pubhashblock","address":"tcp://127.0.0.1:28334","hwm":1000},
  {"type":"pubhashtx","address":"tcp://127.0.0.1:28335","hwm":1000},
  {"type":"pubsequence","address":"tcp://127.0.0.1:28336","hwm":1000}
]
```

Python listener that reconciles `sequence` with mempool state:

```python
import zmq, struct

ctx = zmq.Context()
sock = ctx.socket(zmq.SUB)
sock.set(zmq.RCVHWM, 1000)
sock.connect("tcp://127.0.0.1:28336")
sock.setsockopt(zmq.SUBSCRIBE, b"sequence")

last_mempool_seq = None
while True:
    topic, body, frame_counter = sock.recv_multipart()
    h = body[:32][::-1].hex()  # hash, big-endian for human display
    status = chr(body[32])
    seq = struct.unpack("<Q", body[33:41])[0] if status in "AR" else None
    fc  = int.from_bytes(frame_counter, "little")
    print(f"frame={fc} status={status} hash={h} mempool_seq={seq}")

    if status in "AR":
        if last_mempool_seq is not None and seq != last_mempool_seq + 1:
            print(f"  GAP: missed {seq - last_mempool_seq - 1} mempool events")
        last_mempool_seq = seq
```

Switching to `rawblock` to extract all coinbase outputs from new tips:

```python
sock = ctx.socket(zmq.SUB)
sock.connect("tcp://127.0.0.1:28332")
sock.setsockopt(zmq.SUBSCRIBE, b"rawblock")
while True:
    topic, body, fc = sock.recv_multipart()
    # body is the full serialized block; parse with bitcoinlib or similar
```

## Common pitfalls

- Subscribing without setting a topic filter (`SUBSCRIBE = b""`). You will receive every topic the publisher emits, and a slow consumer is fed unbounded data.
- Treating per-topic order as cross-topic order. A `hashblock` for height N may arrive before or after a `rawtx` for a tx in that block's mempool ancestry. Use `sequence` for ordering.
- Ignoring HWM. Default is 1000 messages buffered per subscriber. A few seconds of slow consumption during a block storm and bitcoind starts dropping. Either tune HWM or use `ipc://` for local low-latency consumers.
- Reading the hash bytes as displayed-form txid without reversing endianness. ZMQ payload is little-endian internal form; UI / RPC always shows big-endian.
- Assuming `'R'` means "mempool eviction". It also fires when the tx was confirmed in a block. Distinguish by also subscribing `'C'` and matching txids.
- Believing reconnection is enough. ZMQ reconnects on TCP loss but does NOT replay missed messages. Catch up by querying RPC after reconnect (compare `getbestblockhash` to last `'C'` you saw).

## References

- `doc/zmq.md` in bitcoin/bitcoin.
- `src/zmq/zmqnotificationinterface.cpp` for the publisher implementation.
- `getzmqnotifications` RPC.
