# RPC Authentication and Whitelisting - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/rpc`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/share/rpcauth/rpcauth.py
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/rpc/SKILL.md

## Concept

Bitcoin Core ships three authentication paths for the JSON-RPC interface: an auto-generated cookie file, salted HMAC credentials via `rpcauth=`, and legacy plaintext `rpcuser`/`rpcpassword`. On top of authentication, Core supports per-user RPC method whitelisting so a compromised credential can only invoke a small set of read-only methods. The combination of a salted credential and a tight whitelist is the recommended posture for any RPC endpoint that crosses a process or network boundary.

## Walkthrough / mechanics

The cookie file lives at `<datadir>/.cookie` (or `<datadir>/<chain>/.cookie` for non-mainnet) and is rewritten with a random secret each time `bitcoind` starts. Its content is `__cookie__:<32 random bytes hex>`, used as HTTP Basic credentials. `bitcoin-cli` discovers it automatically; remote callers cannot.

`rpcauth=` works by storing `username:salt$hmac` where `hmac = HMAC_SHA256(salt, password)`. The clear password never lives on disk. The helper at `share/rpcauth/rpcauth.py` generates the line plus a fresh password printed to stdout; you copy the line into `bitcoin.conf` and the password into your client.

`rpcwhitelist=user:method,method,...` restricts what an authenticated user may invoke. A second flag, `rpcwhitelistdefault=0`, blocks all RPCs by default unless explicitly listed. Multiple `rpcwhitelist=` lines accumulate per user.

## Worked example

Generate a credential:

```bash
$ python3 share/rpcauth/rpcauth.py readonly
String to be appended to bitcoin.conf:
rpcauth=readonly:b9c8a1d6...$3ef2c9...
Your password:
4Q7v9wR-z6sX9pK_t1bN
```

`bitcoin.conf` for a read-only API endpoint plus a fully privileged operator:

```ini
rpcbind=127.0.0.1:8332
rpcallowip=127.0.0.1
rpcallowip=10.0.0.0/24

rpcauth=readonly:b9c8a1d6...$3ef2c9...
rpcauth=ops:7e1aafe2...$11ab44...

rpcwhitelistdefault=0
rpcwhitelist=readonly:getblockcount,getblockhash,getbestblockhash,getrawtransaction,getblock,getblockheader,getblockchaininfo
rpcwhitelist=ops:getblockchaininfo,getmempoolinfo,getpeerinfo,stop,sendrawtransaction
```

Test the read-only credential and verify the whitelist enforces:

```bash
$ curl -s -u readonly:4Q7v9wR-z6sX9pK_t1bN \
    --data '{"jsonrpc":"2.0","id":1,"method":"getblockcount"}' \
    http://127.0.0.1:8332/ | jq
{"result":864123,"error":null,"id":1}

$ curl -s -u readonly:4Q7v9wR-z6sX9pK_t1bN \
    --data '{"jsonrpc":"2.0","id":1,"method":"stop"}' \
    http://127.0.0.1:8332/ | jq
{"result":null,"error":{"code":-32601,"message":"Method not found"},"id":1}
```

Note the error is `-32601 Method not found`, not `-32603` or an HTTP 401. From the caller's view a whitelisted-out method is indistinguishable from a misspelled method, which is intentional.

## Common pitfalls

- Forgetting `rpcwhitelistdefault=0`: with the default of `1`, any user without an explicit whitelist line gets *every* RPC. The whitelist line for a different user does not implicitly restrict you.
- Putting `rpcuser=`/`rpcpassword=` in `bitcoin.conf` while also using `rpcauth=` for the same username: `rpcuser` wins and the salted credential is silently ignored.
- Sharing one cookie between multiple processes by `cat`-piping it into HTTP headers: the cookie rotates on each `bitcoind` restart, which breaks long-lived sessions. Use `rpcauth=` for daemons.
- `rpcbind=0.0.0.0:8332` without a matching `rpcallowip=`: bitcoind refuses to start (good), but pairing it with `rpcallowip=0.0.0.0/0` exposes RPC to the public internet with whatever auth strength you have.
- Believing `rpcwhitelist` blocks the dangerous parts of an allowed method. `getblock` is read-only; `sendrawtransaction` is not. The whitelist is a method filter, not an argument filter.
- Whitelisting `createwallet` on a node that also sets `-walletnotify`. On non-Windows builds the `%w` wallet-name placeholder was substituted using `std::regex_replace`, so a wallet name containing `$'` broke the shell escaping of the name and let the caller get shell metacharacters into the command, which is handed to `system()`. Introduced in Bitcoin Core 24.0 (#25803) and fixed on master by #36048 (merged 2026-09-02); the 31.x branch still carries the vulnerable substitution as of September 2026. It is not reachable over P2P or by an unauthenticated peer - an RPC credential allowed to create wallets is the entire prerequisite, which is exactly what a whitelist is supposed to bound.

## Hardening the transport (as of September 2026)

Authentication is only half the posture. Three changes landed after the 31.1 release (July 2026) and are expected in 32.0 (32.0rc1 tagged 14 September 2026):

- `-rpcmaxconnections=<n>`, default 16, caps simultaneously connected HTTP clients (#35730, merged 2026-08-24). No released version has it, so on 31.1 and earlier the only connection ceiling is `rpcallowip` plus whatever a reverse proxy enforces.
- The libevent HTTP server was replaced with an in-tree implementation on master in June 2026 (merged 2026-06-22), after the 31.x branch was cut - so 31.1 (July 2026) still ships the libevent server and the replacement first appears in 32.0. #36123 (merged 2026-09-05) fixed unbounded read buffering in it: the server kept reading a client's next request while still handling the current one, with no size limit, so a client could block its own queue with a long call such as `waitforblock` and then flood the socket. The fix stops reading from the socket while a request is in flight and lets kernel TCP backpressure apply. The PR notes the vector is limited to authenticated clients - unauthenticated REST requests do not block for long enough to stop the server draining its receive buffer. A reviewer measured sixteen unauthenticated REST connections moving node RSS by 3 MB over 90 s afterwards, against 3.2 GB before.
- #36169 (merged 2026-09-06) stops the RPC listener from setting `SO_REUSEADDR` on Windows. A reuse-enabled listener does not reserve the port exclusively there, so another local process could bind the same port, accept a connection and read the HTTP Basic `Authorization` header - the cookie credential - crossing a local-user boundary. Windows now uses `SO_EXCLUSIVEADDRUSE`.

## References

- `share/rpcauth/rpcauth.py` in the bitcoin/bitcoin repo.
- `doc/JSON-RPC-interface.md` in bitcoin/bitcoin.
- `src/httprpc.cpp` for the request authentication path.
- `src/rpc/server.cpp` for whitelist enforcement.
