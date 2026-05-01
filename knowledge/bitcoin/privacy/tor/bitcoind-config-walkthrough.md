# Bitcoind Tor Configuration Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/tor`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/doc/tor.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/tor/SKILL.md

## Concept

`bitcoind` can run as a Tor onion service so that **outbound** connections
hide the operator's IP and **inbound** connections reach the node via a
v3 onion address (`*.onion`). The wiring is a SOCKS5 proxy plus a control
port for ephemeral onion creation (`ADD_ONION`).

## Walkthrough / mechanics

### Required Tor pieces

In `/etc/tor/torrc`:

```
SocksPort 9050
ControlPort 9051
CookieAuthentication 1
CookieAuthFileGroupReadable 1
DataDirectoryGroupReadable 1
```

Add the bitcoin user to the `debian-tor` (or equivalent) group so it can
read `/run/tor/control.authcookie`:

```
sudo usermod -a -G debian-tor bitcoin
```

### Bitcoind config

Recommended `bitcoin.conf` (privacy-maxed):

```
proxy=127.0.0.1:9050
listen=1
listenonion=1
torcontrol=127.0.0.1:9051
# Disable clearnet listening — onion-only inbound:
bind=127.0.0.1
# Avoid leaking via DNS:
dnsseed=0
dns=0
# Use only onion peers for outbound to maximise privacy:
onlynet=onion
```

### What each setting does

- `proxy=127.0.0.1:9050`: route **all** outbound through Tor SOCKS5.
  Without `onlynet=onion` clearnet IPs are still resolved via Tor's
  SOCKS5h (`USE_HOSTNAME` form), avoiding local DNS.
- `listenonion=1`: bitcoind asks Tor controller for an ephemeral v3 onion
  via `ADD_ONION NEW:ED25519-V3 Port=8333,127.0.0.1:8333`. The hidden
  service key is stored in `~/.bitcoin/onion_v3_private_key` and reused
  across restarts.
- `torcontrol=...`: address of the Tor control port; bitcoind reads the
  cookie file for auth.
- `dnsseed=0`, `dns=0`: avoids any UDP DNS leak. Bootstrap then must rely
  on hardcoded `seed.*` DNS seeds via SOCKS5h or fixed `addnode=` peers.

### Ephemeral vs persistent onion

Default is **ephemeral**: removed when bitcoind exits. With
`onion_v3_private_key` saved, the same `*.onion` is reused each launch.
Operators that want their address advertised in DNS or printed on hardware
typically prefer the persistent variant.

## Worked example

Boot log on a Linux system:

```
$ tail -n 5 ~/.bitcoin/debug.log
... torcontrol thread start
... tor: Got service ID abcdefgh23456ijk7lmn..., advertising service abcdefgh...onion:8333
... AddLocal(abcdefgh...onion:8333,4)
... net thread start
... addcon thread start
```

Verify outbound is via Tor:

```
$ bitcoin-cli getnetworkinfo | jq '.networks[] | {name,reachable,proxy}'
{"name":"ipv4","reachable":false,"proxy":""}
{"name":"ipv6","reachable":false,"proxy":""}
{"name":"onion","reachable":true,"proxy":"127.0.0.1:9050"}
{"name":"i2p","reachable":false,"proxy":""}
```

Verify inbound onion:

```
$ bitcoin-cli getnetworkinfo | jq '.localaddresses'
[ {"address":"abcdefgh...onion","port":8333,"score":4} ]
```

## Common pitfalls

- **Cookie permission errors**: bitcoind logs `tor: Authentication
  cookie /run/tor/control.authcookie could not be opened`. Fix: add bitcoin
  user to debian-tor group, restart both services.
- **Clearnet leak via UPnP**: `upnp=0` and `natpmp=0` should be set
  explicitly. UPnP can punch a clearnet hole even with onion-only intent.
- **Mixing networks**: with `onlynet=onion` you only connect to onion
  peers; if Tor is degraded your node will sit at 0 connections silently.
  Production setups use `onlynet=onion` plus `onlynet=i2p` or `onlynet=cjdns`
  for resilience.
- **Bandwidth amplification**: a public onion peer is queried by SPV
  wallets; expect 50-100 GB/month upstream once `services=NETWORK,WITNESS`
  is advertised. Use `maxuploadtarget=` to cap.
- **DNS leak via `tor` subprocess**: never set `dns=1` together with
  `proxy=...` SOCKS4; SOCKS4 cannot tunnel hostnames. SOCKS5 with
  `proxyrandomize=1` is the standard mode.

## References

- bitcoin/doc/tor.md.
- Tor manual `torrc(5)`, `ControlPort` and `ADD_ONION` semantics.
- BIP155 — addrv2 (transports onion v3 in gossip).
