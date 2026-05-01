# Tor Configuration Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/operations`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/doc/tor.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/operations/SKILL.md

## Concept

There are three orthogonal questions when wiring `bitcoind` to Tor: how do we make outbound connections, do we accept inbound connections, and how do we publish our address. The answers map to the flags `-proxy=`, `-listen=` plus `-listenonion=`, and `-externalip=` plus the Tor control port. Confusing the three is the most common Tor-related Bitcoin issue. The recommended setup is a hidden service for inbound, SOCKS5 via Tor for outbound, and `onlynet=onion` to refuse clearnet peers entirely.

## Walkthrough / mechanics

Outbound: `-proxy=127.0.0.1:9050` routes every outbound peer connection through Tor's SOCKS5 port. By itself this leaks nothing about your IP to other Bitcoin peers, but DNS resolution can leak via the ASN of your DNS server. Pair with `-dns=0` and `-dnsseed=0` and use `-seednode=<onion>` for bootstrap if you want strict isolation.

Inbound: `-listenonion=1` (default 1) tells `bitcoind` to ask the local Tor daemon to provision a v3 onion service via the Tor control port (default 9051). The control port must accept either cookie auth or password auth. Tor writes the service descriptor out as a private key file under `<datadir>/onion_v3_private_key`. Each restart reuses the same `.onion` so peers' addr databases remain valid.

Address advertisement: `-externalip=...` is normally not needed; Core auto-advertises the onion address it received from Tor. For dual-stack nodes (clearnet + onion) Core advertises both addresses to peers, depending on the network they connected from.

`onlynet=onion` is the privacy-maximalist choice: outbound only over Tor, refuse to peer with clearnet addresses even if you know them. Pair with `proxy=127.0.0.1:9050`.

The `bind=` flag controls inbound. With Tor hidden services, the hidden service forwards to a local TCP port; that port should be `-bind=127.0.0.1:8334` (or a different port from your clearnet bind) so an external scan cannot find an open Bitcoin port on a public interface.

## Worked example

`/etc/tor/torrc`:

```
ControlPort 9051
CookieAuthentication 1
CookieAuthFileGroupReadable 1
DataDirectoryGroupReadable 1
```

Add the `bitcoin` user to the `debian-tor` group so it can read the auth cookie:

```bash
$ sudo usermod -aG debian-tor bitcoin
$ sudo systemctl restart tor
```

`/etc/bitcoin/bitcoin.conf` for an onion-only node:

```ini
# Outbound through Tor
proxy=127.0.0.1:9050

# Refuse clearnet peers
onlynet=onion

# Inbound via Tor hidden service
listen=1
listenonion=1
bind=127.0.0.1:8334=onion
torcontrol=127.0.0.1:9051

# Quiet down DNS leak surface
dns=0
dnsseed=0

# Seed via known onions on first start
seednode=2bqghnldu6mcug4pikzprwhtjjnsyederctvci6klcwzepnjd46ikjyd.onion
```

Start, then verify:

```bash
$ bitcoind -daemon
$ bitcoin-cli getnetworkinfo | jq '.networks, .localaddresses'
[
  {"name":"ipv4","limited":true,"reachable":false,"proxy":"127.0.0.1:9050","proxy_randomize_credentials":true},
  {"name":"ipv6","limited":true,"reachable":false,"proxy":"127.0.0.1:9050","proxy_randomize_credentials":true},
  {"name":"onion","limited":false,"reachable":true,"proxy":"127.0.0.1:9050"},
  {"name":"i2p","limited":true,"reachable":false},
  {"name":"cjdns","limited":true,"reachable":false}
]
[
  {"address":"abcdefghij...xyz.onion","port":8333,"score":4}
]

$ bitcoin-cli getpeerinfo | jq '.[] | {addr, network}' | head
{"addr":"abc...xyz.onion:8333","network":"onion"}
{"addr":"def...uvw.onion:8333","network":"onion"}
```

For a dual-stack node that accepts both clearnet and onion inbound, drop `onlynet=onion` and add a clearnet bind on a different port:

```ini
listen=1
listenonion=1
bind=0.0.0.0:8333
bind=127.0.0.1:8334=onion
proxy=127.0.0.1:9050
```

## Common pitfalls

- `bitcoind` cannot read the Tor cookie: Tor logs `... Authentication required.` The fix is the group membership step above; without it the hidden service is never created and you appear as a normal `proxy=`-only node.
- Forgetting `=onion` on the bind: `bind=127.0.0.1:8334` with no tag is treated as a regular clearnet bind, exposing 8334 if you ever drop `bind=127.0.0.1`. Always tag the onion bind explicitly.
- Using `proxy=` alone for "Tor mode" while leaving `listen=1` and a public clearnet bind. Outbound is anonymous; inbound is not. Other peers connecting to your clearnet IP can correlate it with your onion address from `addr` advertisements.
- Pinning `externalip=<your.onion>` manually. Not needed; Core does it. A typo here breaks address gossip.
- `onlynet=onion` plus `proxy=` not pointing at a working Tor: the node has zero peers, IBD never starts. Always check `bitcoin-cli getpeerinfo | jq length` after enabling.
- Different `onion_v3_private_key` after a server move: the `.onion` address changes, peers' `addr` cache is stale. Copy the file from the old datadir to keep stable addressing.

## References

- `doc/tor.md` in bitcoin/bitcoin.
- `src/torcontrol.cpp` for the control protocol implementation.
- Tor manual on `ControlPort` and `CookieAuthentication`.
