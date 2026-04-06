# WireGuard Server Setup

WireGuard is a modern, fast, and secure VPN protocol built into the Linux kernel. If you are still running OpenVPN, stop. WireGuard is simpler to configure, faster (especially on mobile), uses better cryptography (ChaCha20, Poly1305, Curve25519, BLAKE2), and the entire implementation is around 4,000 lines of code versus OpenVPN's ~70,000. Fewer lines of code means a smaller attack surface and easier auditing. This guide covers everything you need to run WireGuard in production.

---

## Table of Contents

1. [Why WireGuard Over OpenVPN](#why-wireguard-over-openvpn)
2. [Installation](#installation)
3. [Key Generation](#key-generation)
4. [IP Forwarding](#ip-forwarding)
5. [Server Configuration (wg0.conf)](#server-configuration)
6. [PostUp / PostDown — Firewall and NAT](#postup--postdown)
7. [Client Configuration](#client-configuration)
8. [Full Tunnel vs Split Tunnel](#full-tunnel-vs-split-tunnel)
9. [QR Codes for Mobile](#qr-codes-for-mobile)
10. [DNS Leak Prevention](#dns-leak-prevention)
11. [Adding and Removing Peers Without Restart](#adding-and-removing-peers-without-restart)
12. [Site-to-Site VPN](#site-to-site-vpn)
13. [Monitoring with wg show](#monitoring-with-wg-show)
14. [Troubleshooting](#troubleshooting)
15. [MTU Considerations](#mtu-considerations)
16. [Security Hardening](#security-hardening)
17. [Systemd Service Management](#systemd-service-management)

---

## Why WireGuard Over OpenVPN

This is an opinion piece as much as a setup guide. WireGuard wins on:

- **Simplicity**: A WireGuard config is 20 lines. An OpenVPN config with a CA is 200+ lines plus certificate files.
- **Performance**: WireGuard lives in the kernel. OpenVPN runs in userspace. The throughput difference is real — expect 2–4x higher throughput on the same hardware.
- **Roaming**: WireGuard reconnects silently when your IP changes (mobile handoffs, switching WiFi). OpenVPN drops and re-establishes the tunnel visibly.
- **Cryptography**: WireGuard uses a fixed, modern crypto suite. No negotiation, no downgrade attacks, no choosing weak cipher suites.
- **Auditability**: 4,000 lines in the kernel. OpenVPN is ~70,000 lines of C.
- **No PKI**: You exchange public keys, not certificates. No CA to manage, no certificate expiry surprises.

**When OpenVPN is still appropriate:**
- You need TCP transport (some corporate firewalls block UDP; OpenVPN-over-TCP port 443 tunnels through).
- You need compatibility with very old client platforms.
- You require client certificate revocation (WireGuard has no concept — you remove the peer's public key from the server config).

For everything else, WireGuard.

---

## Installation

### Ubuntu 22.04 LTS and Later

WireGuard has been in the mainline Linux kernel since 5.6. Ubuntu 22.04 ships kernel 5.15 and has the userspace tools in the default repositories.

```bash
sudo apt update
sudo apt install -y wireguard wireguard-tools
```

That installs:
- `wg` — the low-level WireGuard configuration utility
- `wg-quick` — the higher-level interface manager (reads wg0.conf, sets up routes, runs PostUp/PostDown)
- Kernel module (already compiled in on Ubuntu 22.04+)

### Ubuntu 20.04

The kernel module is available but you need the PPA for the userspace tools on older installs:

```bash
sudo add-apt-repository ppa:wireguard/wireguard
sudo apt update
sudo apt install -y wireguard
```

### Verify the Module is Loaded

```bash
lsmod | grep wireguard
# or
modinfo wireguard
```

If the module is not loaded automatically, load it:

```bash
sudo modprobe wireguard
```

### Other Distributions

```bash
# Debian
sudo apt install wireguard

# CentOS/RHEL 8+
sudo dnf install epel-release
sudo dnf install wireguard-tools

# Arch Linux
sudo pacman -S wireguard-tools

# Alpine
apk add wireguard-tools
```

---

## Key Generation

WireGuard uses Curve25519 for key exchange. Keys are 32-byte values, base64-encoded. Every endpoint (server and each client) needs its own keypair. Never share private keys. Never reuse keys across peers.

### Server Keys

```bash
# Create a secure directory for keys
sudo mkdir -p /etc/wireguard
sudo chmod 700 /etc/wireguard

# Generate server private key and derive public key in one pipeline
wg genkey | sudo tee /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key

# Lock down the private key file
sudo chmod 600 /etc/wireguard/server_private.key
```

View the keys:

```bash
sudo cat /etc/wireguard/server_private.key   # keep this secret
sudo cat /etc/wireguard/server_public.key    # distribute to clients
```

### Client Keys

Generate separately for each client. Do not generate them on the server unless you immediately transfer and delete them.

```bash
# Best practice: generate on the client machine itself
wg genkey | tee client1_private.key | wg pubkey > client1_public.key
chmod 600 client1_private.key
```

If you must generate on the server (e.g., for scripted provisioning):

```bash
wg genkey | sudo tee /etc/wireguard/clients/client1_private.key | wg pubkey | sudo tee /etc/wireguard/clients/client1_public.key
sudo chmod 600 /etc/wireguard/clients/client1_private.key
```

### Preshared Keys (Optional but Recommended)

A preshared key (PSK) adds a layer of symmetric encryption on top of the asymmetric handshake. This provides post-quantum resistance — even if Curve25519 is broken in the future, an attacker still needs the PSK to decrypt traffic. The cost is near zero. Always use PSKs in production.

Generate one PSK per peer pair:

```bash
wg genpsk | sudo tee /etc/wireguard/clients/client1_psk.key
sudo chmod 600 /etc/wireguard/clients/client1_psk.key
```

---

## IP Forwarding

WireGuard is a VPN tunnel. For the server to forward packets between the VPN interface and the internet (or other networks), the kernel must have IP forwarding enabled. Without this, clients can reach the server but not anything beyond it.

### Persistent Configuration via sysctl

```bash
sudo nano /etc/sysctl.d/99-wireguard.conf
```

Contents:

```ini
# /etc/sysctl.d/99-wireguard.conf
# Enable IPv4 forwarding — required for WireGuard to route packets
net.ipv4.ip_forward = 1

# Enable IPv6 forwarding — required if you are routing IPv6 through the tunnel
net.ipv6.conf.all.forwarding = 1

# Disable reverse path filtering on the WireGuard interface
# (optional, but helps in asymmetric routing scenarios)
net.ipv4.conf.all.rp_filter = 0
```

Apply immediately without rebooting:

```bash
sudo sysctl --system
# or just apply our specific file
sudo sysctl -p /etc/sysctl.d/99-wireguard.conf
```

Verify:

```bash
sysctl net.ipv4.ip_forward
# Expected: net.ipv4.ip_forward = 1

sysctl net.ipv6.conf.all.forwarding
# Expected: net.ipv6.conf.all.forwarding = 1
```

---

## Server Configuration

The canonical location for WireGuard configs is `/etc/wireguard/`. The interface name is derived from the filename: `wg0.conf` creates interface `wg0`. You can run multiple interfaces (wg0, wg1, wg2) for different purposes.

### Detect Your Outbound Network Interface

The PostUp NAT rule needs to know which interface connects to the internet. Find it:

```bash
ip route get 8.8.8.8 | awk '{print $5; exit}'
# Typical output: eth0  or  ens3  or  enp0s3
```

Use this value in the PostUp MASQUERADE rule below.

### Full wg0.conf (Annotated)

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
# /etc/wireguard/wg0.conf
# WireGuard server configuration
# Managed by: ops team
# Last updated: 2026-04-06

[Interface]
# The VPN subnet address for this server endpoint.
# The server itself will be 10.0.0.1 on the wg0 interface.
# Use /24 so the server knows the entire 10.0.0.0/24 subnet is local to wg0.
Address = 10.0.0.1/24

# Optional: add an IPv6 ULA range alongside IPv4
# Address = 10.0.0.1/24, fd42:dead:beef::1/64

# UDP port WireGuard listens on. 51820 is the IANA-registered port.
# Use 51820 unless you have a specific reason to change it.
# Firewalls should allow UDP 51820 inbound.
ListenPort = 51820

# The server's private key. Read from file to avoid storing inline.
# Always protect this file: chmod 600 /etc/wireguard/server_private.key
PrivateKey = <paste-server-private-key-here>

# DNS server(s) pushed to clients via wg-quick.
# Using the server's VPN IP as DNS requires running a resolver (e.g., unbound, systemd-resolved)
# on the server listening on 10.0.0.1.
# For a simple setup, use a trusted public resolver.
# For leak prevention, always set this — clients that don't have DNS configured
# will use their system default, which leaks queries outside the tunnel.
DNS = 1.1.1.1, 1.0.0.1

# MTU for the WireGuard interface.
# WireGuard adds ~60 bytes of overhead (IP + UDP + WireGuard headers).
# If the upstream MTU is 1500, set this to 1420 to avoid fragmentation.
# In cloud/container environments, the upstream MTU may already be 1450 or lower —
# check with: ip link show <outbound-interface>
# In those cases you may need 1380 or lower. See MTU Considerations section.
MTU = 1420

# PostUp: commands run AFTER the interface is brought up.
# This sets up NAT so VPN clients can reach the internet through the server.
# Replace eth0 with your actual outbound interface name.
# The -o eth0 flag means: only masquerade traffic going OUT via eth0.
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp = iptables -A FORWARD -o wg0 -j ACCEPT
# IPv6 equivalent (remove if not using IPv6 routing)
PostUp = ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostUp = ip6tables -A FORWARD -i wg0 -j ACCEPT

# PostDown: commands run AFTER the interface is brought down.
# Mirror of PostUp — clean up the rules we added.
# If you don't clean up, the iptables rules persist until reboot
# and accumulate on each wg-quick up/down cycle.
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
PostDown = iptables -D FORWARD -o wg0 -j ACCEPT
PostDown = ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
PostDown = ip6tables -D FORWARD -i wg0 -j ACCEPT

# SaveConfig = true
# Uncomment to have wg-quick write runtime changes (wg set ...) back to this file
# on interface shutdown. Useful for dynamic peer management but can cause
# unexpected overrides of manual edits. Use with caution in production.

###############################################################################
# PEERS
# Each [Peer] block represents one client (or another server in site-to-site).
# The order does not matter.
###############################################################################

[Peer]
# Client: alice-laptop
# Added: 2026-04-06
# Alice's laptop — full tunnel (all traffic through VPN)
PublicKey = <alice-laptop-public-key>

# PresharedKey adds post-quantum symmetric encryption layer.
# Generate with: wg genpsk
# This key must match the PresharedKey in Alice's client config.
PresharedKey = <alice-psk>

# AllowedIPs: the IP address(es) assigned to this peer inside the VPN.
# The server will route packets destined for 10.0.0.2 to Alice's peer.
# Use /32 for a single client — do NOT give clients a /24 unless they are
# acting as a gateway for a subnet (site-to-site pattern).
AllowedIPs = 10.0.0.2/32

# PersistentKeepalive is NOT needed on the server side for regular clients.
# It IS needed on the client side if the client is behind NAT (most are).

[Peer]
# Client: bob-phone (iOS)
# Added: 2026-04-06
PublicKey = <bob-phone-public-key>
PresharedKey = <bob-psk>
AllowedIPs = 10.0.0.3/32

[Peer]
# Client: carol-workstation (split tunnel — only routes company subnet)
# Added: 2026-04-06
PublicKey = <carol-workstation-public-key>
PresharedKey = <carol-psk>
AllowedIPs = 10.0.0.4/32

[Peer]
# Site-to-site: branch-office-router
# Routes the entire 192.168.10.0/24 (branch office LAN) through this peer.
# See the Site-to-Site section for full setup.
PublicKey = <branch-office-public-key>
PresharedKey = <branch-psk>
AllowedIPs = 10.0.0.10/32, 192.168.10.0/24
```

### Lock Down the Config File

```bash
sudo chmod 600 /etc/wireguard/wg0.conf
sudo chown root:root /etc/wireguard/wg0.conf
```

---

## PostUp / PostDown

The PostUp and PostDown lines deserve deeper explanation because they are the most common source of confusion.

### Why MASQUERADE?

When a VPN client (10.0.0.2) sends a packet to 8.8.8.8, the packet arrives at the server's wg0 interface. The server needs to forward it out through eth0 to the internet. But the source address is 10.0.0.2 — a private VPN address that the internet does not know how to route back. MASQUERADE replaces the source address with the server's public IP so that replies come back to the server, which then reverse-NATs them back to the client.

### Dynamic Interface Detection in PostUp

If your outbound interface name might change (e.g., in cloud environments where it is sometimes eth0, ens3, or enp0s3), detect it dynamically:

```ini
PostUp = iptables -t nat -A POSTROUTING -o $(ip route get 8.8.8.8 | awk '{print $5; exit}') -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o $(ip route get 8.8.8.8 | awk '{print $5; exit}') -j MASQUERADE
```

### The FORWARD Rules

```ini
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp = iptables -A FORWARD -o wg0 -j ACCEPT
```

These are needed if your server has a default DROP policy on the FORWARD chain (common on hardened systems). If your default FORWARD policy is ACCEPT, these are redundant but harmless.

### Firewall on the Host (ufw)

If you use ufw, you also need to allow the WireGuard port:

```bash
sudo ufw allow 51820/udp
sudo ufw allow OpenSSH    # don't lock yourself out
sudo ufw enable
```

And ensure IP forwarding works with ufw by editing `/etc/default/ufw`:

```
DEFAULT_FORWARD_POLICY="ACCEPT"
```

Then edit `/etc/ufw/before.rules` and add before the `*filter` section:

```
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE
COMMIT
```

---

## Client Configuration

### Linux Client

```ini
# /etc/wireguard/wg0.conf  (on the client machine)
[Interface]
# The IP address assigned to this client inside the VPN
Address = 10.0.0.2/32

# Client's own private key (generated on this machine)
PrivateKey = <client-private-key>

# DNS to use when the tunnel is up.
# Prevents DNS leaks — queries will go through the tunnel to this server.
DNS = 1.1.1.1, 1.0.0.1

# MTU tuned to avoid fragmentation. Match the server MTU or go slightly lower.
MTU = 1420

[Peer]
# The VPN server
PublicKey = <server-public-key>

# Must match the PresharedKey in the server's [Peer] block for this client
PresharedKey = <psk-for-this-client>

# The server's public endpoint: IP (or hostname) and UDP port
Endpoint = vpn.example.com:51820

# AllowedIPs controls what traffic goes through the tunnel.
# Full tunnel (all traffic):
AllowedIPs = 0.0.0.0/0, ::/0

# OR split tunnel (only VPN subnet):
# AllowedIPs = 10.0.0.0/24

# PersistentKeepalive: send a keepalive packet every 25 seconds.
# Required if the client is behind NAT (home router, mobile, etc.) and
# needs to receive incoming connections from the server or other peers.
# Without this, the NAT mapping expires after ~60–300 seconds of idle.
# 25 seconds is the recommended value from the WireGuard documentation.
PersistentKeepalive = 25
```

Start and enable:

```bash
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0
```

### macOS Client

The official WireGuard app from the Mac App Store is the easiest path. The configuration format is identical to Linux.

Alternatively, install via Homebrew:

```bash
brew install wireguard-tools
```

Then manage with `wg-quick up /path/to/wg0.conf`.

The macOS client config is identical in format:

```ini
[Interface]
Address = 10.0.0.2/32
PrivateKey = <client-private-key>
DNS = 1.1.1.1
MTU = 1420

[Peer]
PublicKey = <server-public-key>
PresharedKey = <psk>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

### Windows Client

Download the official WireGuard client from https://www.wireguard.com/install/. It installs a GUI that reads the same `.conf` format. You can import a config file directly into the GUI or generate QR codes for import.

The config format is identical. Save as `wg0.conf` and import via "Import tunnel(s) from file" in the GUI.

### iOS Client

The WireGuard app is available on the App Store. It supports QR code import (recommended — see QR Codes section) or manual config entry.

Config for iOS (identical format, typically full tunnel):

```ini
[Interface]
Address = 10.0.0.3/32
PrivateKey = <client-private-key>
DNS = 1.1.1.1
MTU = 1380

[Peer]
PublicKey = <server-public-key>
PresharedKey = <psk>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

Note the MTU of 1380 for iOS — Apple's network stack adds overhead in some scenarios. If you see performance issues on cellular, try 1280.

### Android Client

Available on Google Play and F-Droid. Same config format. Import via QR code or file.

```ini
[Interface]
Address = 10.0.0.4/32
PrivateKey = <client-private-key>
DNS = 1.1.1.1
MTU = 1380

[Peer]
PublicKey = <server-public-key>
PresharedKey = <psk>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

---

## Full Tunnel vs Split Tunnel

The `AllowedIPs` field on the client controls what traffic goes through the tunnel. This is both a routing directive and a security policy.

### Full Tunnel

```ini
# In the client's [Peer] block:
AllowedIPs = 0.0.0.0/0, ::/0
```

This routes all IPv4 and IPv6 traffic through the VPN. The client's default gateway becomes the VPN server.

**Use full tunnel when:**
- The client needs privacy (all traffic hidden from local network/ISP)
- You need to enforce a consistent security boundary (all traffic inspected/logged)
- Remote employees accessing the internet should appear to come from the corporate IP
- Traveling employees on untrusted hotel/coffee shop WiFi

**Tradeoffs:**
- All traffic (including Netflix, YouTube, etc.) goes through your server — bandwidth cost
- If the VPN drops, the client loses all internet access unless you configure a kill switch

### Split Tunnel

```ini
# In the client's [Peer] block:
# Route only traffic destined for internal resources through the VPN
AllowedIPs = 10.0.0.0/8, 192.168.1.0/24, 172.16.0.0/12
```

Only traffic to those subnets goes through the tunnel. All other traffic (internet browsing, streaming) goes directly out the client's own network connection.

**Use split tunnel when:**
- You want VPN for internal resource access only, not general privacy
- Bandwidth on the VPN server is constrained
- Employees should access the internet at full speed without going through your server
- Latency-sensitive traffic (video calls, gaming) should not be tunneled

**Split tunnel examples:**

```ini
# Access only the company's internal app subnet and the VPN peers
AllowedIPs = 10.0.0.0/24, 192.168.1.0/24

# Access a specific internal service (e.g., internal API at 10.1.2.3)
AllowedIPs = 10.0.0.0/24, 10.1.2.3/32

# Access internal DNS server only (minimal split tunnel)
AllowedIPs = 10.0.0.0/24, 10.1.0.53/32
```

### The "Exclude One IP" Trick

WireGuard's AllowedIPs matching uses longest-prefix-match (like a routing table). You can use this to exclude a specific IP from a full tunnel. For example, to send all traffic through the VPN except your monitoring endpoint:

```ini
# Full tunnel except 203.0.113.5
AllowedIPs = 0.0.0.0/1, 128.0.0.0/1, ::/0
# Then add more-specific routes to exclude if needed via script
```

The WireGuard AllowedIPs calculator (https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/) is helpful for computing these exclusion ranges.

---

## QR Codes for Mobile

Generating QR codes is the correct way to onboard mobile devices. Typing a 44-character base64 private key on a phone keyboard is not.

### Install qrencode

```bash
sudo apt install -y qrencode
```

### Generate a QR Code in the Terminal

```bash
# Display directly in the terminal (ANSI art)
qrencode -t ansiutf8 < /etc/wireguard/clients/client1.conf

# Save to a PNG file for display elsewhere
qrencode -t png -o /tmp/client1-qr.png < /etc/wireguard/clients/client1.conf

# UTF8 variant (sometimes cleaner in different terminals)
qrencode -t utf8 < /etc/wireguard/clients/client1.conf
```

The `-t ansiutf8` flag renders the QR code using Unicode block characters and ANSI colors directly in the terminal. Show this to the user and have them scan it with the WireGuard mobile app. Once scanned, immediately clear the terminal:

```bash
clear
```

### Security Notes for QR Distribution

- Generate and display QR codes only over SSH sessions — never on a shared screen
- Delete the client config from the server after distribution if possible
- The QR code encodes the private key — treat it like a secret
- If using a PNG file, delete it after the client scans it

### Scripted Client Provisioning

Here is a minimal script to generate a new client config and QR code:

```bash
#!/usr/bin/env bash
# Usage: ./add-client.sh <client-name> <client-vpn-ip>
# Example: ./add-client.sh alice 10.0.0.5

CLIENT_NAME="$1"
CLIENT_IP="$2"
SERVER_PUBKEY=$(cat /etc/wireguard/server_public.key)
SERVER_ENDPOINT="vpn.example.com:51820"
CLIENTS_DIR="/etc/wireguard/clients"

mkdir -p "$CLIENTS_DIR"
chmod 700 "$CLIENTS_DIR"

# Generate client keys
CLIENT_PRIVKEY=$(wg genkey)
CLIENT_PUBKEY=$(echo "$CLIENT_PRIVKEY" | wg pubkey)
CLIENT_PSK=$(wg genpsk)

# Write client config
cat > "$CLIENTS_DIR/${CLIENT_NAME}.conf" <<EOF
[Interface]
Address = ${CLIENT_IP}/32
PrivateKey = ${CLIENT_PRIVKEY}
DNS = 1.1.1.1, 1.0.0.1
MTU = 1420

[Peer]
PublicKey = ${SERVER_PUBKEY}
PresharedKey = ${CLIENT_PSK}
Endpoint = ${SERVER_ENDPOINT}
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
EOF

chmod 600 "$CLIENTS_DIR/${CLIENT_NAME}.conf"

# Save the public key and PSK for adding to server config
echo "$CLIENT_PUBKEY" > "$CLIENTS_DIR/${CLIENT_NAME}.pubkey"
echo "$CLIENT_PSK" > "$CLIENTS_DIR/${CLIENT_NAME}.psk"

echo "Client config generated: $CLIENTS_DIR/${CLIENT_NAME}.conf"
echo "Client public key: $CLIENT_PUBKEY"
echo ""
echo "Add this to /etc/wireguard/wg0.conf on the server:"
echo ""
echo "[Peer]"
echo "# Client: ${CLIENT_NAME}"
echo "PublicKey = ${CLIENT_PUBKEY}"
echo "PresharedKey = ${CLIENT_PSK}"
echo "AllowedIPs = ${CLIENT_IP}/32"
echo ""
echo "QR code for mobile:"
qrencode -t ansiutf8 < "$CLIENTS_DIR/${CLIENT_NAME}.conf"
```

---

## DNS Leak Prevention

DNS leaks are one of the most common VPN failures — the VPN is up, traffic is tunneled, but DNS queries still go to the system's default resolver (often the ISP or local router), revealing what sites the user visits.

### How DNS Leaks Happen

1. Client VPN config does not set `DNS =` — system uses its existing resolvers
2. Client OS has a fallback mechanism that queries multiple resolvers in parallel
3. systemd-resolved or NetworkManager keeps using its pre-VPN configuration

### Prevention: Set DNS in Client Config

Always set `DNS =` in the `[Interface]` block of every client config:

```ini
[Interface]
...
DNS = 10.0.0.1
# Use the VPN server's internal IP as DNS if you run a resolver there,
# or use a trusted resolver reachable through the tunnel
```

### Running Your Own Resolver on the Server

For maximum control, run `unbound` on the server and have clients use it:

```bash
sudo apt install -y unbound

# /etc/unbound/unbound.conf.d/wireguard.conf
cat << 'EOF' | sudo tee /etc/unbound/unbound.conf.d/wireguard.conf
server:
    interface: 10.0.0.1
    access-control: 10.0.0.0/24 allow
    access-control: 127.0.0.0/8 allow
    do-ip6: yes
    hide-identity: yes
    hide-version: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: yes
    cache-min-ttl: 900
    cache-max-ttl: 14400
EOF

sudo systemctl enable unbound
sudo systemctl start unbound
```

Then in client configs: `DNS = 10.0.0.1`

### Verifying No DNS Leaks

From the client machine, once connected:

```bash
# Check what DNS server is being used
resolvectl status   # on systemd-resolved systems
cat /etc/resolv.conf

# Quick DNS leak test
curl -s https://dnsleaktest.com/results.json
# Or visit https://dnsleaktest.com and run "Extended test"

# Manual check: query a known server and see which resolver responds
dig +short whoami.akamai.net
# If this returns your VPN server's IP, DNS is going through the tunnel
```

### macOS DNS Leak Fix

macOS sometimes ignores the WireGuard DNS setting. Force it:

```bash
# After wg-quick up, set DNS explicitly via networksetup
networksetup -setdnsservers Wi-Fi 1.1.1.1
```

### Linux systemd-resolved DNS Leak Fix

On Ubuntu with systemd-resolved, `DNS =` in the WireGuard config is usually handled correctly by wg-quick via `resolvconf` integration. Verify:

```bash
resolvectl dns wg0
# Should show the DNS servers from your config
```

If resolvconf is not installed:

```bash
sudo apt install -y resolvconf
sudo systemctl enable resolvconf
sudo systemctl start resolvconf
```

---

## Adding and Removing Peers Without Restart

One of WireGuard's major operational advantages: you can add or remove peers on a live interface without dropping connections for other peers. There is no concept of "reloading" a WireGuard interface — the kernel module maintains session state separately from the configuration.

### Add a Peer Dynamically

```bash
# Add a peer to the running wg0 interface immediately
sudo wg set wg0 peer <client-public-key> \
    preshared-key /etc/wireguard/clients/client1.psk \
    allowed-ips 10.0.0.5/32

# Verify it was added
sudo wg show wg0
```

This takes effect instantly. The new peer can connect immediately.

### Remove a Peer Dynamically

```bash
sudo wg set wg0 peer <client-public-key> remove

# Verify removal
sudo wg show wg0
```

### Sync the Config File to the Running Interface (Preferred Method)

The cleanest pattern: edit `wg0.conf` first, then sync it to the live interface without restarting:

```bash
# 1. Edit the config file (add or remove [Peer] blocks)
sudo nano /etc/wireguard/wg0.conf

# 2. Strip the wg-quick-specific directives (Address, DNS, PostUp, PostDown, MTU)
#    and feed the result to wg syncconf
sudo wg syncconf wg0 <(sudo wg-quick strip wg0)

# wg syncconf applies only the diff — adds new peers, removes deleted peers,
# updates changed peers, leaves unchanged peers alone
```

The `wg-quick strip` command outputs the interface config minus the wg-quick extensions (Address, DNS, PostUp/PostDown), leaving only what the wg kernel module understands. `wg syncconf` then applies that to the live interface.

This is the production-safe workflow. No service restart, no dropped connections.

### Verify the Sync

```bash
sudo wg show wg0
# All expected peers should appear
# All removed peers should be gone
```

### Persist the Changes

The sync only affects the running interface. The file is already updated (you edited it in step 1), so systemd will pick up the correct state on next restart.

### Using `wg addconf` for Incremental Additions

If you want to add peers from a separate file (useful for automation):

```bash
# /etc/wireguard/new-peer.conf
cat > /tmp/new-peer.conf << 'EOF'
[Peer]
PublicKey = <new-client-public-key>
PresharedKey = <new-psk>
AllowedIPs = 10.0.0.6/32
EOF

sudo wg addconf wg0 /tmp/new-peer.conf
rm /tmp/new-peer.conf
```

Then append the block to wg0.conf manually or via script to make it persistent.

---

## Site-to-Site VPN

Site-to-site connects two networks (e.g., main office 192.168.1.0/24 and branch office 192.168.10.0/24) so hosts on either side can reach each other directly without individual client configs.

### Topology

```
Main Office LAN                         Branch Office LAN
192.168.1.0/24                          192.168.10.0/24
      |                                        |
[Gateway/Server A]  ---- WireGuard ----  [Gateway/Server B]
 10.0.0.1/24 (wg0)                       10.0.0.2/24 (wg0)
 Public IP: 1.2.3.4                       Public IP: 5.6.7.8
```

### Server A Configuration (Main Office / Hub)

```ini
# /etc/wireguard/wg0.conf on Server A

[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <server-a-private-key>

# Route traffic to branch office LAN via wg0
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp = iptables -A FORWARD -o wg0 -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
PostDown = iptables -D FORWARD -o wg0 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# Server B (branch office gateway)
PublicKey = <server-b-public-key>
PresharedKey = <site-to-site-psk>
Endpoint = 5.6.7.8:51820
# This tells Server A: packets for 10.0.0.2/32 (Server B's VPN IP)
# AND packets for 192.168.10.0/24 (the branch LAN) should go to this peer
AllowedIPs = 10.0.0.2/32, 192.168.10.0/24
PersistentKeepalive = 25
```

### Server B Configuration (Branch Office)

```ini
# /etc/wireguard/wg0.conf on Server B

[Interface]
Address = 10.0.0.2/24
ListenPort = 51820
PrivateKey = <server-b-private-key>

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp = iptables -A FORWARD -o wg0 -j ACCEPT
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
PostDown = iptables -D FORWARD -o wg0 -j ACCEPT

[Peer]
# Server A (main office gateway)
PublicKey = <server-a-public-key>
PresharedKey = <site-to-site-psk>
Endpoint = 1.2.3.4:51820
# Server B sends packets for 10.0.0.1/32 and 192.168.1.0/24 to Server A
AllowedIPs = 10.0.0.1/32, 192.168.1.0/24
PersistentKeepalive = 25
```

### Routing on Each LAN Gateway

The LAN hosts need to know to route inter-site traffic through their respective gateways. On Server A:

```bash
# Server A adds a route for branch LAN (usually automatic via AllowedIPs)
ip route add 192.168.10.0/24 via 10.0.0.2 dev wg0

# For hosts on main office LAN to reach branch office,
# they need a route. Add on Server A (it's their gateway):
ip route add 192.168.10.0/24 dev wg0
```

Or set the LAN hosts' default gateway to the WireGuard gateway machine, or add static routes on each host/router.

Make routes persistent in `/etc/wireguard/wg0.conf` using PostUp:

```ini
PostUp = ip route add 192.168.10.0/24 dev wg0
PostDown = ip route del 192.168.10.0/24 dev wg0
```

### Hub-and-Spoke vs Mesh

- **Hub-and-spoke**: All branch offices connect to one central server. Simple but central server is a single point of failure. Works well for up to ~20 sites.
- **Mesh**: Every site has a peer entry for every other site. More resilient, more config. Use tools like `wg-meshconf` or `innernet` for managing mesh configs.

---

## Monitoring with wg show

`wg show` is your primary operational tool. Learn to read it.

### Full Output

```bash
sudo wg show
# or
sudo wg show wg0
```

Example output:

```
interface: wg0
  public key: AbCdEfGhIjKlMnOpQrStUvWxYz0123456789ABCDEF=
  private key: (hidden)
  listening port: 51820

peer: xYzAbCdEfGhIjKlMnOpQrStUvWxYz0123456789ABC=
  preshared key: (hidden)
  endpoint: 203.0.113.42:54321
  allowed ips: 10.0.0.2/32
  latest handshake: 32 seconds ago
  transfer: 1.24 MiB received, 892 KiB sent

peer: QrStUvWxYz0123456789ABCDEFAbCdEfGhIjKlMnOp=
  preshared key: (hidden)
  endpoint: (none)
  allowed ips: 10.0.0.3/32
  latest handshake: 3 days, 4 hours ago
  transfer: 512 B received, 1.50 KiB sent
```

### Interpreting the Output

**latest handshake**:
- < 3 minutes: peer is actively connected
- 3–15 minutes: peer connected recently, may be idle
- > 3 minutes with PersistentKeepalive set: peer may have lost connectivity — investigate
- `(none)` or very old: peer has not connected or has been offline

WireGuard initiates a new handshake every 180 seconds for active sessions, and every session key expires after ~3 minutes of inactivity (requiring a new handshake on next packet). This is by design — forward secrecy.

**endpoint: (none)**:
The peer has never connected, or the server does not know the peer's current IP. This is normal for server-side configs before a client connects. The server learns the endpoint from incoming packets.

**transfer stats**:
Cumulative since the interface came up. Useful for rough bandwidth accounting. For production monitoring, use Prometheus with the wireguard_exporter or parse `wg show --json`.

### Machine-Readable Output

```bash
sudo wg show all dump
# Tab-separated: interface, public-key, preshared-key, endpoint, allowed-ips, latest-handshake, tx-bytes, rx-bytes, persistent-keepalive

sudo wg show wg0 latest-handshakes
# One line per peer: pubkey and Unix timestamp of last handshake

sudo wg show wg0 transfer
# One line per peer: pubkey, rx-bytes, tx-bytes
```

### Prometheus Monitoring

Install `prometheus-wireguard-exporter` for production metrics:

```bash
# Via cargo
cargo install prometheus_wireguard_exporter

# Run alongside WireGuard
prometheus_wireguard_exporter --port 9586 &
```

Or use the Docker image. Scrape port 9586 from Prometheus and alert on peers with stale handshakes.

---

## Troubleshooting

### No Handshake

This is the most common problem. The client sends packets but no handshake completes. Check in order:

**1. Firewall blocking UDP 51820 on the server**

```bash
# On the server, check if port is listening
sudo ss -ulnp | grep 51820
# Expected: UNCONN  0  0  0.0.0.0:51820  0.0.0.0:*  users:(("wg",pid=...,fd=...))

# Test from client (requires netcat with UDP support)
nc -vzu vpn.example.com 51820

# Check server firewall
sudo iptables -L INPUT -n -v | grep 51820
sudo ufw status
```

Fix:
```bash
sudo ufw allow 51820/udp
# or
sudo iptables -A INPUT -p udp --dport 51820 -j ACCEPT
```

**2. Wrong endpoint in client config**

```bash
# From client, resolve the endpoint hostname
nslookup vpn.example.com
ping vpn.example.com

# Verify the IP and port are correct in the client wg0.conf
grep Endpoint /etc/wireguard/wg0.conf
```

**3. Wrong public key**

The most silent failure — wrong public key produces no error, just no handshake. Double-check:

```bash
# On client
wg show wg0
# Note: "peer: <pubkey>" — this must match the server's actual public key

# On server
cat /etc/wireguard/server_public.key
# This must match the PublicKey in the client's [Peer] block
```

**4. IP forwarding disabled**

```bash
sysctl net.ipv4.ip_forward
# Must be 1
```

**5. Enable WireGuard debug logging**

```bash
# Enable debug logging (kernel module)
echo module wireguard +p | sudo tee /sys/kernel/debug/dynamic_debug/control

# Watch the logs
sudo dmesg -w | grep wireguard
```

You will see handshake attempts, errors, and peer interactions. Disable when done:

```bash
echo module wireguard -p | sudo tee /sys/kernel/debug/dynamic_debug/control
```

**6. Clock skew**

WireGuard's handshake uses a timestamp to prevent replay attacks. If the server and client clocks differ by more than ~3 minutes, handshakes fail silently.

```bash
# Check system time
date
timedatectl

# Sync NTP
sudo systemctl enable systemd-timesyncd
sudo timedatectl set-ntp true
```

### Clients Can Connect But Cannot Reach the Internet (Full Tunnel)

The handshake succeeds but no internet access through the tunnel.

**1. IP forwarding not enabled:**
```bash
sysctl net.ipv4.ip_forward
# Fix: see IP Forwarding section above
```

**2. NAT/MASQUERADE rule missing:**
```bash
sudo iptables -t nat -L POSTROUTING -n -v
# Should show a MASQUERADE rule for your outbound interface
```

If missing, the PostUp did not run. Check:
```bash
sudo journalctl -u wg-quick@wg0 --no-pager
```

Add manually:
```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

**3. FORWARD chain dropping packets:**
```bash
sudo iptables -L FORWARD -n -v
# Check for DROP rules before your ACCEPT rules
```

### DNS Leaks

```bash
# Quick check
curl -s ifconfig.me        # should return VPN server IP, not client's real IP
dig +short myip.opendns.com @resolver1.opendns.com  # should also return VPN IP

# DNS leak test
# Visit https://dnsleaktest.com from the client browser
```

Fix: Ensure `DNS =` is set in the client config. On Linux, ensure `resolvconf` is installed and running. On macOS, set DNS manually after connecting.

### Routing Issues (Specific Destinations Not Reachable)

```bash
# Check the routing table on the client
ip route
# With full tunnel, you should see:
# 0.0.0.0/1 dev wg0
# 128.0.0.0/1 dev wg0
# (two routes covering 0.0.0.0/0 via the tunnel, avoiding conflicts with the default route)

# Check that traffic is actually going through the tunnel
traceroute 8.8.8.8
# First hop should be 10.0.0.1 (the VPN server)

# On the server, check if packets are arriving
sudo tcpdump -i wg0 -n host 8.8.8.8
```

### MTU Problems

Symptoms: Small packets work, large transfers fail or are slow. TCP connections hang after initial data. SSH works but file transfers fail.

```bash
# Test with a specific packet size (adjust until it works)
ping -M do -s 1400 vpn.example.com   # Linux: -M do forces no fragmentation
# macOS: ping -D -s 1400 vpn.example.com

# If 1400 succeeds but 1420 fails, your effective MTU is around 1420-28 = 1392
```

WireGuard overhead is 60 bytes for IPv4 (20 IP + 8 UDP + 32 WireGuard header) and 80 bytes for IPv6.

Standard ethernet MTU: 1500
WireGuard interface MTU: 1500 - 60 = 1440 (recommended default: 1420 for safety margin)

In cloud/container environments where the underlying MTU is already reduced:
- AWS/GCP typically: 1460 (they add 40 bytes of their own overhead)
- WireGuard on top: 1460 - 60 = 1400 (use 1380 for safety)
- Docker bridge: 1500 default, but containers may see 1450 or less

Set explicitly in both server and client configs:
```ini
MTU = 1420   # or lower if on cloud/container infra
```

You can also clamp TCP MSS in iptables as a belt-and-suspenders approach:
```ini
PostUp = iptables -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
PostDown = iptables -D FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

---

## MTU Considerations

MTU deserves its own section because it is the source of mysterious, intermittent connectivity failures that are hard to diagnose.

### The Math

| Layer | Size |
|-------|------|
| Ethernet frame (physical) | 1500 bytes max payload |
| IPv4 header | 20 bytes |
| UDP header | 8 bytes |
| WireGuard header | 32 bytes |
| WireGuard overhead total | ~60 bytes |
| Safe WireGuard interface MTU | 1420 bytes |

For IPv6 encapsulation, add 20 more bytes: safe MTU = 1400.

### Cloud Environments

Most cloud providers use jumbo frames internally but the external MTU varies:
- AWS EC2: external interfaces are 1500, but some instance types with ENA drivers support 9001 (jumbo). The WireGuard interface should still be 1420 to be safe with external peers.
- DigitalOcean: 1500 external
- GCP: 1460 (they subtract 40 bytes for their encapsulation)
  - On GCP: WireGuard MTU = 1460 - 60 = 1400 (use 1380 for safety)

### Docker / Kubernetes

Containers typically inherit the host MTU minus some overhead:
- Docker bridge default: 1500 (but actual effective MTU may be lower)
- Kubernetes with overlay networking (Flannel VXLAN, Calico): 1450 or 1430
- Running WireGuard inside a container on top of already-reduced MTU:
  - Host physical: 1500
  - Docker bridge: 1450
  - WireGuard inside container: 1450 - 60 = 1390 (use 1360 for safety)

Always test explicitly in container/K8s environments. Use the `ping -M do -s <size>` method described in Troubleshooting.

---

## Security Hardening

### Key Management

- Never store private keys in plaintext outside `/etc/wireguard/` with strict permissions (600)
- Consider using secret management tools (HashiCorp Vault, AWS Secrets Manager) for automated provisioning
- Rotate keys periodically — WireGuard makes this easy (update both sides' configs and syncconf)
- Always use PresharedKey for post-quantum resistance

### Peer Isolation

By default, WireGuard peers can communicate with each other if you set overlapping AllowedIPs. To prevent peer-to-peer communication (isolate clients from each other):

```ini
# In PostUp, add a FORWARD rule that drops wg0→wg0 forwarding
PostUp = iptables -A FORWARD -i wg0 -o wg0 -j DROP
PostDown = iptables -D FORWARD -i wg0 -o wg0 -j DROP
```

This allows wg0→eth0 (internet) but not wg0→wg0 (peer-to-peer).

### Limiting What the Server Exposes

Use strict AllowedIPs per peer — do not use 0.0.0.0/0 in server [Peer] blocks unless you intend to allow that peer to route anything. The server's AllowedIPs is a whitelist of source IPs accepted from that peer.

### Rate Limiting

WireGuard has built-in cookie mechanisms against DDoS. For additional protection:

```bash
# Rate limit new connections to the WireGuard port
sudo iptables -A INPUT -p udp --dport 51820 -m state --state NEW -m recent --set
sudo iptables -A INPUT -p udp --dport 51820 -m state --state NEW -m recent --update --seconds 60 --hitcount 20 -j DROP
```

### Disable WireGuard on Untrusted Interfaces

If the server has multiple network interfaces, bind WireGuard to specific ones:

```ini
# wg0.conf [Interface] — no way to bind to a specific IP directly,
# but you can restrict via iptables:
PostUp = iptables -A INPUT -p udp --dport 51820 ! -i eth0 -j DROP
PostDown = iptables -D INPUT -p udp --dport 51820 ! -i eth0 -j DROP
```

---

## Systemd Service Management

wg-quick integrates with systemd cleanly.

### Enable and Start

```bash
# Start the interface now and on every boot
sudo systemctl enable --now wg-quick@wg0

# Start only now (no auto-start on boot)
sudo systemctl start wg-quick@wg0

# Stop
sudo systemctl stop wg-quick@wg0

# Restart (teardown and bring up — disconnects all peers briefly)
sudo systemctl restart wg-quick@wg0

# For multiple interfaces:
sudo systemctl enable --now wg-quick@wg0 wg-quick@wg1
```

### Status and Logs

```bash
# Service status
sudo systemctl status wg-quick@wg0

# Full logs
sudo journalctl -u wg-quick@wg0 -f

# Logs since last boot
sudo journalctl -u wg-quick@wg0 -b

# Check for PostUp errors specifically
sudo journalctl -u wg-quick@wg0 --no-pager | grep -i error
```

### Reload Without Full Restart

As covered in the Adding/Removing Peers section:

```bash
# Edit wg0.conf, then:
sudo wg syncconf wg0 <(sudo wg-quick strip wg0)
# No service restart, no dropped connections
```

### Interface Status Commands

```bash
sudo wg show               # show all WireGuard interfaces
sudo wg show wg0           # show specific interface
sudo ip link show wg0      # show link-layer status
sudo ip addr show wg0      # show IP addressing on the interface
sudo ip route show dev wg0 # show routes via this interface
```

---

## Quick Reference

### Essential Commands

| Task | Command |
|------|---------|
| Generate server keypair | `wg genkey \| tee server.key \| wg pubkey > server.pub` |
| Generate client keypair | `wg genkey \| tee client.key \| wg pubkey > client.pub` |
| Generate preshared key | `wg genpsk > client.psk` |
| Bring interface up | `sudo wg-quick up wg0` |
| Bring interface down | `sudo wg-quick down wg0` |
| Show all peers | `sudo wg show` |
| Show specific interface | `sudo wg show wg0` |
| Sync config without restart | `sudo wg syncconf wg0 <(sudo wg-quick strip wg0)` |
| Add peer dynamically | `sudo wg set wg0 peer <pubkey> allowed-ips 10.0.0.X/32` |
| Remove peer dynamically | `sudo wg set wg0 peer <pubkey> remove` |
| Generate QR code | `qrencode -t ansiutf8 < client.conf` |
| Enable on boot | `sudo systemctl enable wg-quick@wg0` |

### IP Address Allocation Convention

```
10.0.0.1/24   — VPN server
10.0.0.2/32   — first client
10.0.0.3/32   — second client
...
10.0.0.10/32  — site-to-site peer A
10.0.0.11/32  — site-to-site peer B
```

### Default Ports

| Service | Port | Protocol |
|---------|------|----------|
| WireGuard | 51820 | UDP |
| Alternative (firewall bypass) | 443 | UDP |

WireGuard does not support TCP natively. If UDP 51820 is blocked, there is no easy workaround short of wrapping WireGuard in a UDP-to-TCP proxy (udp2raw or similar). This is where OpenVPN-over-TCP-443 has an advantage.

---

## References

- WireGuard official site: https://www.wireguard.com/
- WireGuard quickstart: https://www.wireguard.com/quickstart/
- WireGuard netns (network namespace isolation): https://www.wireguard.com/netns/
- AllowedIPs calculator: https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/
- WireGuard whitepaper: https://www.wireguard.com/papers/wireguard.pdf
- DNS leak test: https://dnsleaktest.com/
- Prometheus WireGuard exporter: https://github.com/MindFlavor/prometheus_wireguard_exporter
