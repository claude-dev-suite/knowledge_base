# Electrum Wallet Server Configuration Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/electrum-wallet`.
> Canonical source: https://electrum.readthedocs.io/en/latest/tor.html
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/electrum-wallet/SKILL.md

## Concept

Electrum's default server discovery connects to public ElectrumX servers
chosen automatically; this leaks every address in your wallet to a
third-party operator who can correlate IP, address set, and timing into a
fingerprint. The privacy fix is to either run your own Electrum server
(electrs / Fulcrum) and pin Electrum to it, or proxy through Tor so the
public servers see only an exit-node IP. This article walks both setups,
the relevant CLI flags and `config` JSON keys, the difference between
"auto-connect" and "single-server" modes, and how to verify Electrum is
actually using the server you intended.

## Walkthrough / mechanics

Electrum's per-user config lives at:

- Linux: `~/.electrum/config`
- macOS: `~/Library/Application Support/Electrum/config`
- Windows: `%APPDATA%\Electrum\config`

It is a JSON file. Relevant keys:

```json
{
  "auto_connect": false,
  "oneserver": true,
  "server": "node.example.com:50002:s",
  "proxy": "socks5:127.0.0.1:9050:::",
  "use_tor": true,
  "skipmerklecheck": false,
  "show_addresses_tab": true
}
```

The server string format is `host:port:protocol` where protocol is `s`
for SSL or `t` for plaintext TCP.

Method 1 - Pin to your own electrs / Fulcrum:

```
GUI: Tools -> Network
  Server: node.example.com
  Use SSL: yes
  Auto-connect: OFF
  Use as a single server: ON
  -> click "Apply"
```

Or via CLI:

```bash
electrum setconfig server "node.example.com:50002:s"
electrum setconfig oneserver true
electrum setconfig auto_connect false
```

Method 2 - Tor onion-routing for public servers:

```bash
sudo apt install tor
sudo systemctl enable --now tor
```

Then in Electrum:

```
Tools -> Network
  Proxy: SOCKS5
  Host: 127.0.0.1
  Port: 9050
  Use Tor proxy at port 9050: yes
  Auto-connect: ON (Electrum picks Tor-friendly servers)
```

Or:

```bash
electrum setconfig proxy "socks5:127.0.0.1:9050:::"
electrum setconfig use_tor true
```

Method 3 - Pin to your own server *and* go through Tor (server is on
LAN, you want anonymity from your ISP that you are running Electrum):

```bash
electrum setconfig server "<onion>.onion:50002:s"
electrum setconfig oneserver true
electrum setconfig proxy "socks5:127.0.0.1:9050:::"
```

To expose your own electrs over Tor add to `/etc/tor/torrc`:

```
HiddenServiceDir /var/lib/tor/electrs/
HiddenServicePort 50001 127.0.0.1:50001
```

After `systemctl reload tor`, the file `/var/lib/tor/electrs/hostname`
contains the v3 onion address. Note Tor onion services do not need TLS
- the protocol is already authenticated and encrypted.

Verify which server Electrum is actually using:

```bash
electrum getservers | jq 'to_entries | map(select(.value.connected==true))'
electrum getinfo | jq '.server'
# "node.example.com"
```

## Worked example

Switch from public servers to a self-hosted electrs over Tor in five
minutes:

```bash
# 1. Confirm electrs is reachable locally
echo '{"jsonrpc":"2.0","method":"server.version","params":["",""],"id":0}' \
  | nc -q 1 127.0.0.1 50001

# 2. Add Tor hidden service
sudo tee -a /etc/tor/torrc <<EOF
HiddenServiceDir /var/lib/tor/electrs/
HiddenServiceVersion 3
HiddenServicePort 50001 127.0.0.1:50001
EOF
sudo systemctl reload tor
ONION=$(sudo cat /var/lib/tor/electrs/hostname)
echo "$ONION"

# 3. Configure Electrum (laptop) - assumes Tor running on laptop too
electrum setconfig server "${ONION}:50001:t"
electrum setconfig oneserver true
electrum setconfig auto_connect false
electrum setconfig proxy "socks5:127.0.0.1:9050:::"

# 4. Restart Electrum, verify connection
electrum daemon stop
electrum daemon start
electrum getinfo | jq '.server,.connected'
# "<onion>.onion"
# true
```

| Mode | Privacy from server | Privacy from ISP | Latency |
|------|---------------------|------------------|---------|
| Public + clearnet | none | none | low |
| Public + Tor | partial | yes | high |
| Own server + clearnet | full | none | low |
| Own server + Tor | full | yes | medium |

## Common pitfalls

- `auto_connect: true` overrides your pinned server when the connection
  fails - Electrum silently switches to a public one. Set `oneserver:
  true` to refuse fallback.
- TLS cert mismatch when pointing at electrs - electrs has no built-in
  TLS; either front it with nginx/stunnel and use port 50002 with SSL,
  or use the plaintext port 50001 over Tor (which encrypts at the
  transport layer).
- Onion service syntax - Electrum expects `host.onion:port:protocol`
  exactly, not `tcp://host.onion`.
- Daemon vs GUI conflict - if `electrum daemon` is running with old
  config, GUI changes won't apply until daemon restart.
- Skipping merkle check (`skipmerklecheck: true`) hides invalid responses
  from a malicious server - leave it false.
- Public servers being unreachable - the default list churns; if all
  fail, `electrum --offline` lets you sign offline while the network is
  down.

## References

- Electrum docs: https://electrum.readthedocs.io/
- Tor configuration: https://electrum.readthedocs.io/en/latest/tor.html
- Server pinning: https://electrum.readthedocs.io/en/latest/protocol.html
- electrs: https://github.com/romanz/electrs
