# UFW + fail2ban: Production Hardening Guide

**Author perspective:** Senior Linux sysadmin, 10+ years of production Ubuntu/Debian systems.  
**Target OS:** Ubuntu 22.04 LTS / 24.04 LTS (Debian compatible where noted)  
**Last reviewed:** 2026-04  

---

## Table of Contents

1. [Overview and Philosophy](#overview-and-philosophy)
2. [UFW — Uncomplicated Firewall](#ufw--uncomplicated-firewall)
   - [Installation and Initial State](#installation-and-initial-state)
   - [Default Policies](#default-policies)
   - [Basic Rule Syntax](#basic-rule-syntax)
   - [Application Profiles](#application-profiles)
   - [Rate Limiting](#rate-limiting)
   - [IPv6 Support](#ipv6-support)
   - [Rule Management](#rule-management)
   - [Logging](#logging)
   - [The Docker UFW Bypass Problem](#the-docker-ufw-bypass-problem)
3. [fail2ban — Intrusion Prevention](#fail2ban--intrusion-prevention)
   - [Architecture](#architecture)
   - [Installation](#installation)
   - [jail.local — The Only File You Should Edit](#jaillocal--the-only-file-you-should-edit)
   - [SSH Jail](#ssh-jail)
   - [Nginx Jails](#nginx-jails)
   - [Custom Filter Regex](#custom-filter-regex)
   - [Recidive Jail](#recidive-jail)
   - [Cloudflare Action](#cloudflare-action)
   - [fail2ban-client Reference](#fail2ban-client-reference)
   - [Whitelist and ignoreip](#whitelist-and-ignoreip)
   - [Email Notifications](#email-notifications)
4. [Putting It All Together](#putting-it-all-together)
5. [Operational Checklist](#operational-checklist)

---

## Overview and Philosophy

UFW and fail2ban are complementary tools. UFW is your static ruleset — it defines what traffic is structurally allowed or denied based on port, protocol, and source. fail2ban is your dynamic layer — it watches logs and temporarily bans IPs that exhibit abusive behaviour.

The correct mental model:

```
Internet → UFW (static gatekeeping) → Service → fail2ban (behavioural banning)
```

Neither tool alone is sufficient. UFW without fail2ban leaves brute-force attacks free to hammer allowed ports. fail2ban without UFW means every port is theoretically reachable.

**Opinions you should hold:**
- Default-deny everything inbound. Always.
- Never open ports you don't need. Obvious, but people forget after "quick tests."
- fail2ban should be running before you open any public-facing service port.
- The Docker/UFW bypass is a real problem that bites everyone who isn't paying attention. Read that section.
- `jail.conf` is upstream-owned. `jail.local` is yours. Never touch `jail.conf`.

---

## UFW — Uncomplicated Firewall

UFW is a frontend for iptables (and ip6tables). It does not replace iptables — it generates iptables rules. This distinction matters enormously when Docker enters the picture.

### Installation and Initial State

```bash
sudo apt update && sudo apt install ufw
```

Check current status:

```bash
sudo ufw status verbose
```

On a fresh install, UFW is **disabled**. Do not enable it until you have at minimum an SSH allow rule in place, or you will lock yourself out.

```bash
# Verify SSH rule is in place BEFORE enabling
sudo ufw allow ssh

# Then enable
sudo ufw enable
```

---

### Default Policies

Set your default policies first, before adding any rules. This is the foundation.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed
```

What these do:
- `deny incoming` — drops all inbound connections unless explicitly allowed. This is the only sane default.
- `allow outgoing` — permits all outbound connections. For most servers this is fine; lock it down further only if you have a compliance requirement.
- `deny routed` — disables IP forwarding/routing through this host. If this machine is not a router, it should not route. Docker complicates this (see below).

These are stored in `/etc/default/ufw`:

```ini
# /etc/default/ufw
DEFAULT_INPUT_POLICY="DROP"
DEFAULT_OUTPUT_POLICY="ACCEPT"
DEFAULT_FORWARD_POLICY="DROP"
```

After `ufw enable`, these policies are reflected in `/etc/ufw/before.rules`, `after.rules`, and the `COMMIT` statements therein. You can inspect the raw iptables output with:

```bash
sudo iptables -L -n -v --line-numbers
sudo ip6tables -L -n -v --line-numbers
```

---

### Basic Rule Syntax

#### Allow/Deny by Port

```bash
# Allow a single port (TCP + UDP)
sudo ufw allow 80

# Allow a single port, specific protocol
sudo ufw allow 80/tcp
sudo ufw allow 53/udp

# Allow a port range
sudo ufw allow 6000:6007/tcp

# Deny a port
sudo ufw deny 23/tcp
```

#### Allow/Deny by Source IP

```bash
# Allow all traffic from a trusted IP
sudo ufw allow from 192.168.1.50

# Allow from a subnet
sudo ufw allow from 10.0.0.0/8

# Allow from IP to specific port
sudo ufw allow from 203.0.113.15 to any port 5432

# Allow from subnet to specific port and protocol
sudo ufw allow from 10.10.0.0/16 to any port 443 proto tcp
```

#### Allow/Deny by Destination

```bash
# Allow to a specific local IP (multi-homed server)
sudo ufw allow to 192.168.1.100 port 80 proto tcp

# Deny outbound to a specific destination
sudo ufw deny out to 198.51.100.0/24
```

#### Full Source-to-Destination Rules

```bash
# Allow from 10.0.0.5 to 10.0.0.10 on port 3306 (MySQL)
sudo ufw allow from 10.0.0.5 to 10.0.0.10 port 3306 proto tcp
```

#### Service Names

UFW resolves service names from `/etc/services`:

```bash
sudo ufw allow ssh      # resolves to port 22/tcp
sudo ufw allow http     # resolves to port 80/tcp
sudo ufw allow https    # resolves to port 443/tcp
sudo ufw allow smtp     # resolves to port 25/tcp
```

#### Non-Standard SSH Port

If you've moved SSH to a non-standard port (you should):

```bash
sudo ufw allow 2222/tcp comment 'SSH non-standard port'
```

The `comment` flag is underused. Use it. Six months from now you will thank yourself.

---

### Application Profiles

UFW ships with application profiles stored in `/etc/ufw/applications.d/`. These are INI files that define named rule sets for common services.

List available profiles:

```bash
sudo ufw app list
```

Example output:
```
Available applications:
  Apache
  Apache Full
  Apache Secure
  Nginx Full
  Nginx HTTP
  Nginx HTTPS
  OpenSSH
  Postfix
  Postfix SMTPS
  Postfix Submission
```

Inspect a profile:

```bash
sudo ufw app info 'Nginx Full'
```

Output:
```
Profile: Nginx Full
Title: Web Server (Nginx, HTTP + HTTPS)
Description: Small, but very capable, web server and mail proxy server.

Ports:
  80,443/tcp
```

Apply a profile:

```bash
sudo ufw allow 'Nginx Full'
sudo ufw allow 'OpenSSH'
```

Remove a profile rule:

```bash
sudo ufw delete allow 'Nginx Full'
```

Create a custom application profile. Drop a file in `/etc/ufw/applications.d/`:

```ini
# /etc/ufw/applications.d/myapp
[MyApp]
title=My Application
description=Custom app running on ports 8080 and 8443
ports=8080,8443/tcp
```

Then:

```bash
sudo ufw app update MyApp
sudo ufw allow MyApp
```

---

### Rate Limiting

UFW has a built-in rate-limiting rule type using the `limit` keyword. It uses iptables `recent` module under the hood: it blocks IPs that initiate more than 6 connections within 30 seconds.

```bash
# Rate-limit SSH (the canonical use case)
sudo ufw limit ssh

# Rate-limit a custom port
sudo ufw limit 2222/tcp
```

This is a blunt instrument — it only triggers on connection rate, not failed authentication. It is a useful first line of defence but is **not a substitute for fail2ban**. Use both.

What it actually does in iptables:

```
-A ufw-user-input -p tcp --dport 22 -m state --state NEW -m recent --set
-A ufw-user-input -p tcp --dport 22 -m state --state NEW -m recent --update --seconds 30 --hitcount 6 -j ufw-user-limit
-A ufw-user-input -p tcp --dport 22 -m state --state NEW -j ufw-user-limit-accept
```

---

### IPv6 Support

UFW handles IPv6 natively. You must ensure it is enabled in the configuration file.

Edit `/etc/default/ufw`:

```ini
# /etc/default/ufw
IPV6=yes
```

After changing this, reload UFW:

```bash
sudo ufw disable && sudo ufw enable
```

With `IPV6=yes`, every rule you create applies to both IPv4 and IPv6 automatically. UFW writes mirrored rules to `ip6tables`.

Verify:

```bash
sudo ip6tables -L ufw6-user-input -n -v
```

Dual-stack rules work transparently:

```bash
# This applies to both 0.0.0.0/0 and ::/0
sudo ufw allow 443/tcp

# Allow from IPv6 subnet
sudo ufw allow from 2001:db8::/32 to any port 22 proto tcp

# Deny a specific IPv6 address
sudo ufw deny from 2001:db8::bad:actor
```

If you need an IPv6-only rule, you currently cannot express that with plain `ufw` syntax — you would add it directly to `/etc/ufw/before6.rules`. That's a rare edge case; in most environments the dual-stack approach is correct.

The files UFW manages for IPv6:
- `/etc/ufw/before6.rules`
- `/etc/ufw/after6.rules`
- `/etc/ufw/user6.rules` (auto-generated, do not edit directly)

---

### Rule Management

#### Viewing Rules

```bash
# Simple list
sudo ufw status

# Verbose (shows default policies and all rules)
sudo ufw status verbose

# Numbered list — essential for deletion
sudo ufw status numbered
```

Numbered output example:
```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere
[ 2] 80/tcp                     ALLOW IN    Anywhere
[ 3] 443/tcp                    ALLOW IN    Anywhere
[ 4] 5432/tcp                   ALLOW IN    10.0.0.0/8
[ 5] 22/tcp (v6)                ALLOW IN    Anywhere (v6)
[ 6] 80/tcp (v6)                ALLOW IN    Anywhere (v6)
[ 7] 443/tcp (v6)               ALLOW IN    Anywhere (v6)
```

#### Deleting Rules

By number (preferred — unambiguous):

```bash
sudo ufw delete 4
```

By rule specification:

```bash
sudo ufw delete allow 80/tcp
sudo ufw delete allow from 10.0.0.0/8 to any port 5432
```

By application profile:

```bash
sudo ufw delete allow 'Nginx Full'
```

#### Inserting Rules at a Specific Position

Rules are evaluated in order. Insert before rule 2:

```bash
sudo ufw insert 2 deny from 198.51.100.0/24
```

#### Resetting UFW

Nuclear option — wipes all rules and disables UFW:

```bash
sudo ufw reset
```

This backs up existing rules to `/etc/ufw/*.rules.YYYYMMDD_HHMMSS` before wiping. Never do this on a production system without a console or out-of-band access standing by.

---

### Logging

UFW logging writes to `/var/log/ufw.log` (via rsyslog's `kern.*` facility). It integrates with journald on systemd systems.

```bash
# Enable logging (default level: low)
sudo ufw logging on

# Disable logging
sudo ufw logging off

# Set level
sudo ufw logging low
sudo ufw logging medium
sudo ufw logging high
sudo ufw logging full
```

**Logging levels explained:**

| Level  | What is logged                                                                 |
|--------|--------------------------------------------------------------------------------|
| off    | Nothing                                                                        |
| low    | Blocked packets not matching a rule, plus packets matching log rules          |
| medium | low + allowed packets, invalid packets, new connections                        |
| high   | medium + all packets (regardless of rate limiting)                             |
| full   | high, without rate limiting — will generate enormous log volume on busy hosts |

**Production recommendation:** Use `medium` on servers where you're actively troubleshooting or building your ruleset. Drop to `low` once you've validated everything. `high` and `full` are for forensic investigation only — they will fill your disk on any reasonably trafficked server.

Log entries look like:

```
Apr  6 14:32:01 hostname kernel: [UFW BLOCK] IN=eth0 OUT= MAC=... SRC=198.51.100.42 DST=203.0.113.5 LEN=44 TOS=0x00 PREC=0x00 TTL=244 ID=54321 PROTO=TCP SPT=54321 DPT=3306 WINDOW=1024 RES=0x00 SYN URGP=0
```

To watch UFW logs live:

```bash
sudo journalctl -f -k | grep UFW
# or
sudo tail -f /var/log/ufw.log
```

---

### The Docker UFW Bypass Problem

This is the most important section if you run Docker on a UFW-protected host. Most people who run Docker + UFW have a **false sense of security**. Their UFW rules are being silently bypassed.

#### The Problem Explained

Docker manipulates iptables directly. When you start a container with a published port (`-p 8080:80`), Docker adds rules to the `DOCKER` chain and the `DOCKER-FORWARD` chain that punch holes in the firewall **before UFW's rules are even consulted**.

The packet flow on a Docker host looks like this:

```
Incoming packet
    → PREROUTING (Docker adds DNAT rules here)
    → FORWARD chain
        → DOCKER-USER chain  ← your intervention point
        → DOCKER chain       ← Docker's rules (bypasses UFW's INPUT chain)
        → ACCEPT
```

UFW rules sit in the `INPUT` chain. Docker-published ports bypass INPUT entirely — they go through FORWARD via Docker's NAT rules. This means:

```bash
# You set this rule thinking port 8080 is blocked from the internet:
sudo ufw deny 8080

# But anyone on the internet can still reach your container on port 8080.
# UFW's deny rule is irrelevant for Docker-forwarded ports.
```

This is not a bug in UFW. It is expected iptables behaviour. Docker is operating correctly for its design goals. The two tools simply have different mental models about who owns iptables.

#### Solution 1: DOCKER-USER Chain (Recommended)

Docker explicitly provides the `DOCKER-USER` chain for this purpose. Rules you add here are evaluated before Docker's own rules. This is the least invasive solution — Docker keeps managing its own chains, and you add restrictions in DOCKER-USER.

The `DOCKER-USER` chain persists across Docker restarts but **does not persist across reboots** unless you save it. Use `iptables-persistent` to save:

```bash
sudo apt install iptables-persistent
```

Add rules to DOCKER-USER:

```bash
# Block all external access to Docker-published ports by default
# Allow only from your office IP range (203.0.113.0/24)
sudo iptables -I DOCKER-USER -i eth0 ! -s 203.0.113.0/24 -j DROP

# Allow from internal network
sudo iptables -I DOCKER-USER -i eth0 -s 10.0.0.0/8 -j RETURN

# Allow established connections (important — must come before DROP)
sudo iptables -I DOCKER-USER -m conntrack --ctstate RELATED,ESTABLISHED -j RETURN
```

**Order matters.** Rules in DOCKER-USER are evaluated top-down. Use `-I` (insert at top) carefully, building your chain in reverse order of evaluation, or use explicit positions:

```bash
# Correct order using positions:
# Position 1: allow RELATED,ESTABLISHED
sudo iptables -I DOCKER-USER 1 -m conntrack --ctstate RELATED,ESTABLISHED -j RETURN

# Position 2: allow from trusted subnet  
sudo iptables -I DOCKER-USER 2 -i eth0 -s 10.0.0.0/8 -j RETURN

# Position 3: drop everything else coming from external interface
sudo iptables -I DOCKER-USER 3 -i eth0 -j DROP
```

Save rules to persist across reboots:

```bash
sudo netfilter-persistent save
```

This writes to `/etc/iptables/rules.v4` and `/etc/iptables/rules.v6`.

**Limitation:** You are now managing two parallel rule systems (UFW + raw iptables for DOCKER-USER). Keeping them mentally consistent requires discipline.

#### Solution 2: `--iptables=false` + Manual Rules

Configure Docker to not touch iptables at all:

Edit `/etc/docker/daemon.json`:

```json
{
  "iptables": false
}
```

Restart Docker:

```bash
sudo systemctl restart docker
```

With iptables management disabled, Docker containers are still accessible on their published ports **from localhost**, but external routing does not happen automatically. You must manually set up NAT and forwarding rules.

```bash
# Enable IP forwarding in the kernel
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.d/99-docker-forward.conf
sudo sysctl -p /etc/sysctl.d/99-docker-forward.conf

# Set up masquerading for Docker's default bridge (172.17.0.0/16)
sudo iptables -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE

# Allow forwarding to/from Docker bridge
sudo iptables -A FORWARD -i docker0 -j ACCEPT
sudo iptables -A FORWARD -o docker0 -j ACCEPT

# Now UFW controls everything — add rules as needed
sudo ufw allow from any to any port 8080 proto tcp
```

**Downside:** This is high-maintenance. Every time Docker or a container needs outbound internet access, you must trace the iptables flows yourself. Docker Compose networking becomes fragile. Internal container-to-container networking may break depending on your bridge configuration. Avoid this approach unless you are a network engineer who enjoys debugging NAT.

#### Solution 3: ufw-docker Project

The [ufw-docker](https://github.com/chaifeng/ufw-docker) project is a shell script and iptables ruleset that patches the UFW/Docker integration cleanly. It:

1. Blocks Docker traffic in FORWARD by default (via UFW's forward chain)
2. Provides `ufw-docker` commands to expose specific containers

Installation:

```bash
sudo wget -O /usr/local/bin/ufw-docker \
  https://github.com/chaifeng/ufw-docker/raw/master/ufw-docker
sudo chmod +x /usr/local/bin/ufw-docker
sudo ufw-docker install
sudo systemctl restart ufw
```

Usage:

```bash
# Allow external access to a container's port
sudo ufw-docker allow mycontainer 80/tcp

# Allow from a specific source
sudo ufw-docker allow mycontainer 80/tcp 203.0.113.0/24

# Remove an allow rule
sudo ufw-docker delete allow mycontainer 80/tcp

# List all ufw-docker rules
sudo ufw-docker status
```

#### Which Solution Is Best and Why

**Use Solution 1 (DOCKER-USER chain) in most cases.**

Here is the reasoning:

- Solution 2 (`--iptables=false`) breaks Docker's own networking assumptions and creates more work than it saves. It is a footgun for anyone who comes after you and tries to troubleshoot why Docker networking behaves oddly.

- Solution 3 (ufw-docker) is elegant but introduces an external dependency on a third-party shell script that wraps iptables. If it breaks after an OS upgrade or a Docker update, you may have an invisible firewall hole. It also adds cognitive overhead for anyone who doesn't know about ufw-docker.

- Solution 1 is explicit, uses Docker's officially supported extension point (DOCKER-USER), works with every Docker version, and does not require third-party tools. The raw iptables commands look intimidating but there are only 3–4 rules to manage. Combined with `iptables-persistent` for boot persistence, it is robust and auditable.

**The DOCKER-USER approach in practice for a typical web server:**

```bash
# Save this as /usr/local/sbin/apply-docker-user-rules.sh
#!/bin/bash
# Applied at boot via iptables-persistent, or manually after Docker restart

# Flush DOCKER-USER first (idempotent)
iptables -F DOCKER-USER

# 1. Always allow RELATED,ESTABLISHED (do not break existing connections)
iptables -A DOCKER-USER -m conntrack --ctstate RELATED,ESTABLISHED -j RETURN

# 2. Allow from internal private networks (trusted)
iptables -A DOCKER-USER -s 10.0.0.0/8 -j RETURN
iptables -A DOCKER-USER -s 172.16.0.0/12 -j RETURN
iptables -A DOCKER-USER -s 192.168.0.0/16 -j RETURN

# 3. Allow from your office/management IP
iptables -A DOCKER-USER -s 203.0.113.0/24 -j RETURN

# 4. Drop everything else from external interfaces
iptables -A DOCKER-USER -i eth0 -j DROP

# 5. Return for everything else (loopback, etc.)
iptables -A DOCKER-USER -j RETURN
```

---

## fail2ban — Intrusion Prevention

### Architecture

fail2ban has three core concepts:

**Filters** — Regular expressions that match log lines indicating a failed/abusive action. Stored in `/etc/fail2ban/filter.d/*.conf`. A filter defines *what to look for*.

**Actions** — Scripts that are executed when a ban threshold is crossed. Stored in `/etc/fail2ban/action.d/*.conf`. An action defines *what to do* (add iptables rule, call API, send email, etc.).

**Jails** — The glue. A jail binds a filter to an action, specifies which log file to monitor, and sets the thresholds (maxretry, findtime, bantime). Configured in `jail.local`.

Data flow:

```
Log file → fail2ban (applies filter regex) → match found → increment counter
                                                           → counter >= maxretry within findtime
                                                           → execute action (ban IP)
                                                           → after bantime → execute unban action
```

fail2ban maintains its state in `/var/lib/fail2ban/fail2ban.sqlite3`. Bans survive service restarts (but not if the database is deleted).

---

### Installation

```bash
sudo apt update && sudo apt install fail2ban

# Start and enable
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Verify it's running:

```bash
sudo systemctl status fail2ban
sudo fail2ban-client status
```

---

### jail.local — The Only File You Should Edit

`/etc/fail2ban/jail.conf` is owned by the package. Every `apt upgrade` of fail2ban may overwrite it. **Never edit jail.conf.**

Always create `/etc/fail2ban/jail.local`. Settings in `jail.local` override `jail.conf`. You only need to specify what you're changing.

```bash
sudo touch /etc/fail2ban/jail.local
```

The `[DEFAULT]` section in `jail.local` applies to all jails unless overridden:

```ini
# /etc/fail2ban/jail.local

[DEFAULT]
# Whitelist — never ban these IPs
ignoreip = 127.0.0.1/8 ::1 10.0.0.0/8 192.168.0.0/16 203.0.113.50

# Default ban duration: 1 hour
bantime  = 3600

# Window in which maxretry must be reached
findtime  = 600

# Number of failures before ban
maxretry = 5

# Ban action — ufw is preferred if UFW is your firewall
banaction = ufw

# Backend for log watching (auto is fine; systemd for journald-only logs)
backend = auto

# Enable DNS lookups in logs (warn = log a warning if reverse DNS is slow)
usedns = warn

# Email settings (if using mail actions)
destemail = admin@example.com
sender    = fail2ban@example.com
mta       = sendmail

# Default action: ban only (no email)
# Other options: action_mw (ban + email), action_mwl (ban + email + log excerpt)
action = %(action_)s
```

**On `banaction = ufw`:** When using UFW as your firewall, set `banaction = ufw` in `[DEFAULT]`. This tells fail2ban to use `ufw.conf` action instead of `iptables-multiport`. The UFW action adds a UFW rule to block the IP, which is cleaner and survives UFW reloads properly. The action file is at `/etc/fail2ban/action.d/ufw.conf`.

---

### SSH Jail

The most important jail. Every internet-facing server should have this.

```ini
# /etc/fail2ban/jail.local (append to file)

[sshd]
enabled   = true
port      = ssh
# If SSH is on a non-standard port:
# port    = 2222
filter    = sshd
logpath   = %(sshd_log)s
backend   = %(sshd_backend)s
maxretry  = 3
findtime  = 300
bantime   = 86400
```

**Option breakdown:**
- `enabled = true` — Jails are disabled by default in `jail.conf`. You must explicitly enable each one in `jail.local`.
- `port = ssh` — The port to block. Resolves to 22. Use a number if SSH is on a non-standard port.
- `filter = sshd` — Uses `/etc/fail2ban/filter.d/sshd.conf`. This ships with fail2ban and handles OpenSSH log formats.
- `logpath = %(sshd_log)s` — Macro that resolves to the correct log path based on OS. On Ubuntu 22.04+ with systemd, this typically resolves to the journald backend.
- `backend = %(sshd_backend)s` — On systemd systems, this is `systemd`, which reads from journald without needing a log file path.
- `maxretry = 3` — 3 failures and you're banned. Be aggressive. Legitimate users do not fail SSH auth 3 times.
- `findtime = 300` — 5-minute window. 3 failures in 5 minutes = ban.
- `bantime = 86400` — 24-hour ban. The default 600 seconds (10 minutes) is too lenient. Attackers retry after the ban lifts.

For systemd-based SSH logging (Ubuntu 22.04+), verify the backend resolves correctly:

```bash
sudo fail2ban-client get sshd logpath
sudo fail2ban-client get sshd backend
```

If SSH logs are going to journald and not `/var/log/auth.log`, ensure `backend = systemd` is used:

```ini
[sshd]
enabled  = true
port     = ssh
filter   = sshd
backend  = systemd
maxretry = 3
findtime = 300
bantime  = 86400
```

---

### Nginx Jails

#### HTTP Authentication Failures

Blocks IPs that repeatedly fail HTTP basic auth:

```ini
[nginx-http-auth]
enabled  = true
port     = http,https
filter   = nginx-http-auth
logpath  = /var/log/nginx/error.log
maxretry = 5
findtime = 300
bantime  = 3600
```

The filter `/etc/fail2ban/filter.d/nginx-http-auth.conf` looks for lines like:
```
2024/01/15 10:23:45 [error] 12345#0: *1 no user/password was provided for basic authentication, client: 198.51.100.42
```

#### Bad Bots / Scanner Detection

Create a custom filter for common scanner patterns. Create `/etc/fail2ban/filter.d/nginx-bad-bots.conf`:

```ini
# /etc/fail2ban/filter.d/nginx-bad-bots.conf
[Definition]
failregex = ^<HOST> .* "(GET|POST|HEAD) .*(\.php|\.asp|\.aspx|wp-login|xmlrpc|phpmyadmin|admin|eval\(|base64_decode) .*" (404|403|400|500) .*$
            ^<HOST> .* "-(.*)" \d+ .*$

ignoreregex =
```

Add the jail:

```ini
[nginx-bad-bots]
enabled  = true
port     = http,https
filter   = nginx-bad-bots
logpath  = /var/log/nginx/access.log
maxretry = 2
findtime = 60
bantime  = 86400
```

`maxretry = 2` — scanners make dozens of requests per second. Two hits of the pattern within 60 seconds is enough evidence.

#### Request Rate Limiting (4xx Flood)

Blocks IPs generating excessive 4xx errors (scrapers, enumerators):

Create `/etc/fail2ban/filter.d/nginx-req-limit.conf`:

```ini
# /etc/fail2ban/filter.d/nginx-req-limit.conf
[Definition]
failregex = ^<HOST> .* "(GET|POST|HEAD|OPTIONS) .+" (400|403|404|405|408|429) \d+ ".*" ".*"$

ignoreregex = ^<HOST> .* "(GET|POST) /favicon\.ico .*$
              ^<HOST> .* "(GET|POST) /robots\.txt .*$
```

Jail:

```ini
[nginx-req-limit]
enabled  = true
port     = http,https
filter   = nginx-req-limit
logpath  = /var/log/nginx/access.log
maxretry = 20
findtime = 60
bantime  = 7200
```

---

### Custom Filter Regex

The `failregex` directive is the heart of fail2ban filtering.

**Key syntax rules:**
- `<HOST>` is a special placeholder that matches IPv4 and IPv6 addresses. It captures the IP to ban.
- The regex is applied to each log line independently (single-line by default).
- Python regex syntax is used.
- The `ignoreregex` directive filters out false positives.

Example: custom application that logs like:
```
2024-01-15 10:23:45 [WARN] Failed login for user 'admin' from IP 198.51.100.42
```

Filter file `/etc/fail2ban/filter.d/myapp.conf`:

```ini
[Definition]
failregex = ^%(__prefix_line)s\[WARN\] Failed login for user '.*' from IP <HOST>$

ignoreregex =
```

The `%(__prefix_line)s` macro matches common log prefixes (timestamp + hostname + service name combinations). Check `/etc/fail2ban/filter.d/common.conf` for this and other useful macros.

#### Testing Filters

Always test your filter before deploying:

```bash
# Test against a log file
sudo fail2ban-regex /var/log/nginx/access.log /etc/fail2ban/filter.d/nginx-bad-bots.conf

# Test against a systemd journal
sudo fail2ban-regex 'systemd-journal' /etc/fail2ban/filter.d/sshd.conf

# Test with verbose output (shows each line matched/unmatched)
sudo fail2ban-regex --print-all-missed /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf

# Test a single log line
sudo fail2ban-regex "Jan 15 10:23:45 hostname sshd[12345]: Failed password for invalid user admin from 198.51.100.42 port 54321 ssh2" /etc/fail2ban/filter.d/sshd.conf
```

Output includes statistics: how many lines matched, how many were missed, and performance metrics. Aim for zero false negatives on known-bad lines and zero false positives on known-good lines.

---

### Recidive Jail

The recidive jail is fail2ban's "three strikes, longer ban" mechanism. It monitors fail2ban's own log for IP addresses that have been banned multiple times and applies a much longer ban.

This is the correct response to attackers who wait out your ban time and return.

```ini
[recidive]
enabled   = true
filter    = recidive
logpath   = /var/log/fail2ban.log
action    = %(action_mwl)s
banaction = ufw
maxretry  = 3
findtime  = 86400
bantime   = 604800
```

**What this does:**
- Watches `/var/log/fail2ban.log` for entries showing an IP was banned
- If an IP gets banned 3 times within 24 hours (findtime = 86400)
- Bans that IP for 7 days (bantime = 604800)
- Also sends an email with a log excerpt (`action_mwl`)

The recidive filter (`/etc/fail2ban/filter.d/recidive.conf`) ships with fail2ban:

```ini
[Definition]
failregex = ^%(__prefix_line)s(?:NOTICE|INFO|WARNING) \s*\[(?!recidive)[^\]]+\] Ban <HOST>$
ignoreregex =
```

If you store fail2ban logs in journald only (no `/var/log/fail2ban.log`), you need to configure fail2ban to write to a file. In `/etc/fail2ban/fail2ban.local`:

```ini
[Definition]
logtarget = /var/log/fail2ban.log
```

---

### Cloudflare Action

If your server sits behind Cloudflare, banning at the iptables level is pointless — the real client IPs are hidden behind Cloudflare's proxies. You need to ban at the Cloudflare level via their API.

fail2ban ships with a Cloudflare action at `/etc/fail2ban/action.d/cloudflare.conf`. However, the modern approach uses the API v4 action. Check if `cloudflare-apiv4.conf` exists; if not, create it:

```ini
# /etc/fail2ban/action.d/cloudflare-apiv4.conf
[Definition]
actionstart =
actionstop  =
actioncheck =

actionban   = curl -s -X POST "https://api.cloudflare.com/client/v4/user/firewall/access_rules/rules" \
              -H "X-Auth-Email: <cfuser>" \
              -H "X-Auth-Key: <cftoken>" \
              -H "Content-Type: application/json" \
              --data '{"mode":"block","configuration":{"target":"ip","value":"<ip>"},"notes":"fail2ban <name>"}'

actionunban = curl -s -X DELETE "https://api.cloudflare.com/client/v4/user/firewall/access_rules/rules/$(curl -s -X GET \
              "https://api.cloudflare.com/client/v4/user/firewall/access_rules/rules?mode=block&configuration_target=ip&configuration_value=<ip>&page=1&per_page=1" \
              -H "X-Auth-Email: <cfuser>" \
              -H "X-Auth-Key: <cftoken>" \
              -H "Content-Type: application/json" | python3 -c "import sys,json;print(json.load(sys.stdin)['result'][0]['id'])")" \
              -H "X-Auth-Email: <cfuser>" \
              -H "X-Auth-Key: <cftoken>" \
              -H "Content-Type: application/json"

[Init]
cfuser  = your@email.com
cftoken = your_cloudflare_global_api_key
name    = fail2ban
```

Use it in a jail:

```ini
[nginx-http-auth]
enabled   = true
port      = http,https
filter    = nginx-http-auth
logpath   = /var/log/nginx/error.log
maxretry  = 5
findtime  = 300
bantime   = 3600
action    = cloudflare-apiv4[cfuser="your@email.com", cftoken="your_api_key"]
```

**Important notes on Cloudflare action:**
- Use your Global API Key, not a scoped API token, unless you scope the token to Firewall Rules write permission.
- Cloudflare account-level IP access rules (the endpoint above) apply across all zones. If you want zone-specific blocking, use the zone firewall rules endpoint instead.
- Store the API key outside the config file. Use an environment variable or reference a secrets file:

```ini
action = cloudflare-apiv4[cfuser="%(cfuser)s", cftoken="%(cftoken)s"]
```

In `jail.local` `[DEFAULT]`:
```ini
cfuser  = your@email.com
cftoken = <%= read from vault or env %>
```

For production, retrieve the key from a secrets manager at service start time and inject it into the environment.

---

### fail2ban-client Reference

`fail2ban-client` is the primary operational interface. The fail2ban service must be running.

#### Status and Inspection

```bash
# Overall status: list all active jails
sudo fail2ban-client status

# Status of a specific jail
sudo fail2ban-client status sshd

# Output includes:
# - Filter: file, currently failed count, total failed count
# - Actions: ban, currently banned IPs, total banned count, banned IP list
```

Example `status sshd` output:
```
Status for the jail: sshd
|- Filter
|  |- Currently failed:	2
|  |- Total failed:	47
|  `- Journal matches:	_SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned:	1
   |- Total banned:	12
   `- Banned IP list:	198.51.100.42
```

#### Banning and Unbanning

```bash
# Manually ban an IP in a specific jail
sudo fail2ban-client set sshd banip 198.51.100.42

# Manually unban an IP from a specific jail
sudo fail2ban-client set sshd unbanip 198.51.100.42

# Unban from all jails at once (useful when you've locked yourself out)
sudo fail2ban-client unban 198.51.100.42
```

#### Reloading Configuration

```bash
# Reload all jails (re-reads jail.local without restart)
sudo fail2ban-client reload

# Reload a specific jail only
sudo fail2ban-client reload sshd

# Restart the entire service (loses in-memory state, re-reads database)
sudo systemctl restart fail2ban
```

#### Other Useful Commands

```bash
# Get a configuration value for a jail
sudo fail2ban-client get sshd maxretry
sudo fail2ban-client get sshd bantime
sudo fail2ban-client get sshd logpath

# Set a configuration value at runtime (does not persist to jail.local)
sudo fail2ban-client set sshd maxretry 3

# Add an IP to ignoreip at runtime
sudo fail2ban-client set sshd addignoreip 10.0.0.5

# Remove an IP from ignoreip at runtime
sudo fail2ban-client set sshd delignoreip 10.0.0.5

# Ping the fail2ban server (check if responding)
sudo fail2ban-client ping

# Show fail2ban server version
sudo fail2ban-client version
```

---

### Whitelist and ignoreip

`ignoreip` prevents fail2ban from ever banning listed addresses. This is critical — you do not want to ban your own management IPs.

In `jail.local` `[DEFAULT]` section:

```ini
ignoreip = 127.0.0.1/8 ::1 10.0.0.0/8 192.168.0.0/16 172.16.0.0/12 203.0.113.50
```

Multiple addresses/networks are space-separated.

**What to whitelist:**
- Loopback (`127.0.0.1/8` and `::1`)
- Internal RFC1918 networks you manage
- Your office static IP(s)
- Your VPN exit IP(s)
- Monitoring system IPs (Nagios, Zabbix, etc. will trip SSH auth failures if their keys expire)

**What not to whitelist:**
- Cloudflare IP ranges (if you want to catch abuse at the origin)
- "Most of the internet" — defeats the purpose
- Shared hosting provider IPs — too broad

You can also configure `ignoreip` per-jail to override the default:

```ini
[sshd]
enabled   = true
ignoreip  = 127.0.0.1/8 ::1 10.0.0.0/8 203.0.113.0/24
# This overrides [DEFAULT] ignoreip for sshd only
```

At runtime, manage ignoreip without restart:

```bash
sudo fail2ban-client set sshd addignoreip 203.0.113.100
sudo fail2ban-client get sshd ignoreip
```

---

### Email Notifications

fail2ban can send emails on ban events. This requires a working MTA on the server.

#### Install a Lightweight MTA

For a server that only needs to send outbound email:

```bash
sudo apt install postfix mailutils
# Choose "Satellite system" or "Internet with smarthost" during setup
```

Or use `msmtp` as a lightweight alternative:

```bash
sudo apt install msmtp msmtp-mta
```

#### Configure Email Actions

fail2ban ships with several action variants that include email:

| Action variable | What it does                              |
|----------------|-------------------------------------------|
| `%(action_)s`  | Ban only, no email                        |
| `%(action_m)s` | Ban + email notification                  |
| `%(action_mw)s`| Ban + email with whois lookup             |
| `%(action_mwl)s`| Ban + email + whois + log lines          |

In `jail.local` `[DEFAULT]`:

```ini
destemail = admin@example.com
sender    = fail2ban@yourhostname.example.com
mta       = sendmail

# Send email on every ban with whois and log excerpt
action = %(action_mwl)s
```

Or configure per-jail:

```ini
[sshd]
enabled   = true
port      = ssh
filter    = sshd
backend   = systemd
maxretry  = 3
findtime  = 300
bantime   = 86400
action    = %(banaction)s[name=%(__name__)s, port="%(port)s", protocol="%(protocol)s"]
            %(mta)s-whois-lines[name=%(__name__)s, dest="%(destemail)s", logpath="%(logpath)s", chain="%(chain)s"]
```

#### Custom Email Action

If you want to control the email format precisely, create `/etc/fail2ban/action.d/custom-mail.conf`:

```ini
# /etc/fail2ban/action.d/custom-mail.conf
[Definition]
actionstart = printf "%%b" "Subject: [fail2ban] <name> jail started on <fq-hostname>
From: <sendername> <<sender>>
To: <dest>

The jail <name> has been started." | /usr/sbin/sendmail -f <sender> <dest>

actionstop = printf "%%b" "Subject: [fail2ban] <name> jail stopped on <fq-hostname>
From: <sendername> <<sender>>
To: <dest>

The jail <name> has been stopped." | /usr/sbin/sendmail -f <sender> <dest>

actionban = printf "%%b" "Subject: [fail2ban] Banned <ip> in jail <name> on <fq-hostname>
From: <sendername> <<sender>>
To: <dest>

Banned <ip> after <failures> failures.

Whois: $(whois <ip>)

Log lines:
$(grep '<ip>' <logpath> | tail -20)" | /usr/sbin/sendmail -f <sender> <dest>

actionunban =

[Init]
sendername = Fail2Ban
```

---

## Putting It All Together

Here is a complete, production-ready `jail.local` for a web server running SSH, Nginx, and protected by UFW:

```ini
# /etc/fail2ban/jail.local
# Production configuration — $(hostname) — last updated $(date +%Y-%m)

[DEFAULT]
ignoreip  = 127.0.0.1/8 ::1 10.0.0.0/8 192.168.0.0/16 172.16.0.0/12
bantime   = 86400
findtime  = 600
maxretry  = 5
banaction = ufw
backend   = auto
usedns    = warn
destemail = admin@example.com
sender    = fail2ban@example.com
mta       = sendmail
action    = %(action_mw)s


# ─── SSH ──────────────────────────────────────────────────────────────────────

[sshd]
enabled  = true
port     = ssh
filter   = sshd
backend  = systemd
maxretry = 3
findtime = 300
bantime  = 86400


# ─── Nginx ────────────────────────────────────────────────────────────────────

[nginx-http-auth]
enabled  = true
port     = http,https
filter   = nginx-http-auth
logpath  = /var/log/nginx/error.log
maxretry = 5
findtime = 300
bantime  = 3600

[nginx-bad-bots]
enabled  = true
port     = http,https
filter   = nginx-bad-bots
logpath  = /var/log/nginx/access.log
maxretry = 2
findtime = 60
bantime  = 86400

[nginx-req-limit]
enabled  = true
port     = http,https
filter   = nginx-req-limit
logpath  = /var/log/nginx/access.log
maxretry = 20
findtime = 60
bantime  = 7200


# ─── Recidive ─────────────────────────────────────────────────────────────────

[recidive]
enabled   = true
filter    = recidive
logpath   = /var/log/fail2ban.log
banaction = ufw
maxretry  = 3
findtime  = 86400
bantime   = 604800
action    = %(action_mwl)s
```

And the corresponding UFW setup:

```bash
#!/bin/bash
# Initial UFW setup for a web server
# Run once as root

set -euo pipefail

# Set default policies
ufw default deny incoming
ufw default allow outgoing
ufw default deny routed

# SSH (adjust port if needed)
ufw limit 22/tcp comment 'SSH with rate limiting'

# Web
ufw allow 80/tcp comment 'HTTP'
ufw allow 443/tcp comment 'HTTPS'

# Internal monitoring (adjust subnet)
ufw allow from 10.0.0.0/8 to any port 9100 proto tcp comment 'Prometheus node exporter'

# Enable (confirm when prompted, or use --force in scripts)
ufw --force enable

ufw status verbose
```

---

## Operational Checklist

Use this checklist when provisioning a new server or auditing an existing one.

### UFW Checklist

- [ ] UFW is installed and enabled (`ufw status` shows "Status: active")
- [ ] Default policies: `deny incoming`, `allow outgoing`, `deny routed`
- [ ] `IPV6=yes` in `/etc/default/ufw`
- [ ] SSH is allowed before UFW was enabled (you are not locked out)
- [ ] SSH uses `ufw limit` not just `ufw allow` for rate limiting
- [ ] Logging is set to `medium` or higher
- [ ] All rules have comments (`comment 'purpose'`)
- [ ] `ufw status numbered` output has been reviewed and no unnecessary rules exist
- [ ] If Docker is running: DOCKER-USER chain is configured and saved with `netfilter-persistent`
- [ ] `/etc/ufw/applications.d/` custom profiles are in version control

### fail2ban Checklist

- [ ] fail2ban is installed and running (`systemctl status fail2ban`)
- [ ] `jail.local` exists and `jail.conf` has not been edited
- [ ] `[DEFAULT]` ignoreip includes all management IPs
- [ ] `[sshd]` jail is enabled
- [ ] `[recidive]` jail is enabled
- [ ] All filters have been tested with `fail2ban-regex` against real log samples
- [ ] `banaction = ufw` is set in `[DEFAULT]` (not `iptables-multiport`)
- [ ] `/var/log/fail2ban.log` exists and is being written to (needed by recidive)
- [ ] `fail2ban-client status` shows all expected jails as active
- [ ] A test ban has been performed and verified: `fail2ban-client set sshd banip <test-ip>` then `ufw status numbered` shows the ban rule
- [ ] Unban was verified: `fail2ban-client set sshd unbanip <test-ip>` removed the UFW rule
- [ ] Email notifications have been tested (if configured)
- [ ] fail2ban database backed up: `/var/lib/fail2ban/fail2ban.sqlite3`

### Post-Change Verification

After any firewall change:

```bash
# Check UFW status
sudo ufw status verbose

# Check fail2ban is still running and jails are active
sudo fail2ban-client status

# Check iptables directly (the ground truth)
sudo iptables -L -n -v --line-numbers
sudo ip6tables -L -n -v --line-numbers

# Test connectivity from an external host (not the server itself)
# nc -zv <server-ip> 22
# nc -zv <server-ip> 80
# nc -zv <server-ip> 3306  # should fail
```

**The cardinal rule of firewall changes:** always have an out-of-band access method (console, IPMI, cloud provider console) before making significant changes. `ufw reset` and `systemctl stop fail2ban` are your emergency exits — know where they are.
