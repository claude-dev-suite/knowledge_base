# Stratum V1 Methods Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/mining/stratum-v1`.
> Canonical source: Slush Pool Stratum spec (de-facto), `braiins/stratum-v1` reference
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/mining/stratum-v1/SKILL.md

## Concept

Stratum V1 is a line-delimited JSON-RPC 2.0 protocol over plain TCP. Each
message is a single JSON object terminated by `\n`. The pool acts as both
server (accepting `mining.subscribe`, `mining.authorize`, `mining.submit`)
and client (pushing `mining.notify`, `mining.set_difficulty`,
`client.reconnect`).

There is no formal RFC. The reference is the Slush Pool wiki plus
implementation behaviour in cgminer / bmminer / Braiins's `stratum-v1` Rust
crate.

## Walkthrough / mechanics

### Connection lifecycle

```
TCP connect to pool:port
  -> mining.subscribe        (request, id=1)
  <- subscription response   (id=1 result)
  -> mining.authorize        (request, id=2)
  <- authorize response      (id=2 result = true)
  <- mining.set_difficulty   (notification, id=null)
  <- mining.notify           (notification, id=null)
  ... loop ...
  -> mining.submit           (request, id=N)
  <- submit response         (id=N result = true/false)
```

The pool may push `mining.set_difficulty` or `mining.notify` at any time,
even before authorization completes.

### `mining.subscribe`

Request:

```json
{"id": 1, "method": "mining.subscribe", "params": ["bmminer/2.0", null]}
```

Params: `[user_agent, session_id_to_resume_or_null, host, port]` (last two
optional, often omitted).

Response:

```json
{"id": 1, "result":
  [
    [["mining.set_difficulty","abc"],["mining.notify","abc"]],
    "08000002",
    4
  ],
 "error": null}
```

- `result[0]` - subscription details (largely ignored by miners).
- `result[1]` - `extranonce1` as hex (here 4 bytes: `08 00 00 02`).
- `result[2]` - `extranonce2_size` in bytes (here 4, so miner appends 4
  bytes of extranonce2 in the coinbase).

### `mining.authorize`

```json
{"id": 2, "method": "mining.authorize",
 "params": ["bc1q...workername", "x"]}
```

Response: `{"id": 2, "result": true, "error": null}`.

The username encodes the payout address (full pool) or wallet address
(non-custodial pools like Ocean/Public-Pool). Password is conventional
"x" - rarely validated.

### `mining.set_difficulty`

```json
{"id": null, "method": "mining.set_difficulty", "params": [4096]}
```

Sets the share target to `diff1_target / 4096`. `diff1_target` is the
"difficulty 1" target = `0x00000000FFFF0000...` (top 32 bits = 0, next 16
bits = 0xFFFF, rest = 0). New shares must hash <= this scaled target.

### `mining.notify`

```json
{"id": null, "method": "mining.notify", "params": [
  "bf",
  "4d16b6f8...0000",
  "01000000010000...ffffffff",
  "ffffffff02...00000000",
  ["a1b2...","c3d4..."],
  "20000000",
  "17034219",
  "66301c00",
  true
]}
```

Fields in order: `job_id`, `prev_hash` (LE-32 byte hash), `coinb1`,
`coinb2`, `merkle_branch[]`, `version`, `nbits`, `ntime`, `clean_jobs`.

If `clean_jobs == true`, miner must abandon all prior jobs (a new tip was
found upstream). If false, the new job can be queued alongside.

### `mining.submit`

```json
{"id": 15, "method": "mining.submit",
 "params": ["bc1q...workername",
            "bf",
            "00000001",
            "66301c0a",
            "0a1b2c3d"]}
```

Fields: `worker`, `job_id`, `extranonce2`, `ntime`, `nonce` - all hex
strings. The pool reconstructs the full block header by combining these
with the original notify payload, hashes it, and verifies.

Response on accept: `{"id": 15, "result": true, "error": null}`.

Response on reject: `{"id": 15, "result": null,
 "error": [21, "Job not found", null]}`.

Common error codes: 21 stale, 22 duplicate, 23 low-difficulty share,
24 unauthorized worker, 25 not subscribed.

### `mining.set_extranonce`

Some pools push this to rotate `extranonce1` mid-session:

```json
{"id": null, "method": "mining.set_extranonce",
 "params": ["08000003", 4]}
```

Not all miners support it; those that don't will keep the old extranonce1
and submit shares against a re-derived job.

### `client.reconnect`

```json
{"id": null, "method": "client.reconnect",
 "params": ["pool-eu.example.com", 3334, 30]}
```

Pool requests the miner reconnect to a new endpoint after `30` seconds.
Used for load balancing.

## Worked example

Full reconstruction of a header from notify + submit:

1. `coinbase = coinb1 || extranonce1 || extranonce2 || coinb2`
2. `coinbase_hash = SHA256d(coinbase)` (32 bytes, LE)
3. Walk the merkle branch:
   ```
   root = coinbase_hash
   for branch_hash in merkle_branch:
       root = SHA256d(root || branch_hash)
   ```
4. Build header = `version_LE || prev_hash || merkle_root || ntime_LE ||
   nbits_LE || nonce_LE`. Note that `prev_hash` and `merkle_root` use the
   internal byte order, while `version`, `ntime`, `nbits`, `nonce` are
   little-endian 4-byte integers.
5. Hash. If `SHA256d(header) <= share_target`, it's a valid share.
   If also `<= block_target`, the pool submits to the network.

Numerical sanity check: at share difficulty 4096, `share_target =
diff1_target / 4096 ~ 6.99e71`. Probability per nonce of finding a
share = `share_target / 2^256 ~ 6.99e71 / 1.16e77 ~ 6.03e-6`. So
~165k nonces per share - 165k hashes is microseconds for an ASIC, hence
miners rotate `extranonce2` to widen the search space.

## Common pitfalls

- **Endianness** - `prev_hash` in `notify` is in internal byte order
  (already swapped). The header field must use it as-is. `version`,
  `nbits`, `ntime`, `nonce` are little-endian when serialized.
- **Stale submissions on `clean_jobs=true`** - miners must drop in-flight
  work or risk rejection 21.
- **JSON line framing** - missing `\n` makes the pool buffer indefinitely;
  pretty-printed multi-line JSON breaks the protocol.
- **Extranonce mismatch** - if the miner uses `extranonce2_size + 1`
  bytes (off-by-one), the coinbase TXID changes from what the pool
  expects, and shares fail to validate.
- **Plaintext credentials** - Stratum V1 has no encryption, so a MITM
  on the path can swap the payout address. Pools should be reached over
  Stratum V2 or stunnel-wrapped TLS.

## References

- Slush Pool Stratum spec (archived) <https://slushpool.com/help/stratum-protocol/>
- `braiins/stratum-v1` <https://github.com/braiins/stratum-v1>
- cgminer source `stratum.c`
- Stratum V2 spec (compares to V1) <https://stratumprotocol.org/specification/>
