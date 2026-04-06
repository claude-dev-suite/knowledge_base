# SSH Hardening: Comprehensive Production Guide

**Audience:** Senior sysadmins managing Linux servers in production.  
**Scope:** OpenSSH 8.x and 9.x. Tested on Ubuntu 22.04/24.04, Debian 12, RHEL 9, Rocky Linux 9.  
**Philosophy:** Default OpenSSH configs are not hardened. Every directive below exists for a reason. If you don't understand a setting, don't skip it — understand it first, then apply it.

---

## Table of Contents

1. [Key Types: The Correct Choice](#1-key-types-the-correct-choice)
2. [Full sshd_config Hardening](#2-full-sshd_config-hardening)
3. [SSH Certificate Authority for Fleet Management](#3-ssh-certificate-authority-for-fleet-management)
4. [Two-Factor Authentication with PAM](#4-two-factor-authentication-with-pam)
5. [Client-Side ~/.ssh/config Best Practices](#5-client-side-sshconfig-best-practices)
6. [ssh-audit: Auditing Your SSH Configuration](#6-ssh-audit-auditing-your-ssh-configuration)
7. [fail2ban SSH Jail Configuration](#7-fail2ban-ssh-jail-configuration)
8. [known_hosts Management](#8-known_hosts-management)
9. [Operational Checklist](#9-operational-checklist)

---

## 1. Key Types: The Correct Choice

### The Short Answer

Use **Ed25519**. That is the correct answer. If you're generating a new key today for any purpose — user authentication, host keys, CA keys — it should be Ed25519. Everything else is a compromise or legacy support.

### Key Type Comparison

| Type | Key Size | Security Bits | Speed | Recommended |
|------|----------|---------------|-------|-------------|
| Ed25519 | 256 bits | ~128 | Fastest | **YES — use this** |
| RSA | 4096 bits | ~140 | Slow | Only for legacy compatibility |
| ECDSA (P-256/P-384/P-521) | 256-521 bits | 128-256 | Fast | **NO — avoid** |
| DSA | 1024 bits | <80 | Fast | **NEVER — broken** |

### Why Ed25519 is the Right Choice

Ed25519 uses the Curve25519 elliptic curve and the EdDSA signature scheme, designed by Daniel J. Bernstein and Tanja Lange. Here is why it is the correct default:

**1. No NIST curve controversy.** ECDSA uses NIST P-curves (P-256, P-384, P-521). These curves have known design concerns — the curve constants were generated through a process that Bernstein and others have argued may have been influenced by the NSA to introduce a backdoor. You cannot prove a backdoor exists, but you also cannot prove one does not. Curve25519 was designed in full public view with explicit, auditable parameter choices. Ed25519 sidesteps the controversy entirely.

**2. Smaller keys, smaller signatures.** An Ed25519 public key is 68 bytes. An RSA-4096 public key is 800+ bytes. Signatures are 64 bytes vs 512 bytes. This matters in authorized_keys files and certificates on large fleets.

**3. Faster.** Ed25519 signing and verification is significantly faster than RSA-4096. On a constrained bastion host handling thousands of connections, this adds up.

**4. Constant-time implementation.** Ed25519 is naturally resistant to timing side-channel attacks. RSA implementations require careful engineering to achieve the same property.

**5. No weak key generation.** RSA key generation involves random prime selection. A bad PRNG during generation can produce weak keys. Ed25519 key generation is just a hash of 32 random bytes — much simpler, much harder to get wrong.

**DSA is broken.** DSA at 1024 bits is below modern security margins. OpenSSH has deprecated it. Do not generate DSA keys. Do not accept DSA keys. Period.

**RSA-4096 is acceptable for legacy.** If you need to interoperate with ancient systems that don't support Ed25519 (e.g., some network appliances, old versions of PuTTY), RSA-4096 is the fallback. RSA-2048 is the absolute floor, and it's aging out. Never use RSA-1024.

### Key Generation Commands

```bash
# Ed25519 — use this for everything
ssh-keygen -t ed25519 -C "user@host $(date +%Y-%m-%d)"

# Ed25519 with explicit output path (for scripts, CA keys, etc.)
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "mario@workstation 2026-04-06"

# RSA-4096 — only for legacy compatibility
ssh-keygen -t rsa -b 4096 -C "user@host $(date +%Y-%m-%d)"

# NEVER generate these:
# ssh-keygen -t dsa                  # broken
# ssh-keygen -t ecdsa                # NIST curve controversy
# ssh-keygen -t rsa -b 2048          # marginal — use 4096 if you must use RSA
```

**Always use a passphrase on private keys.** The passphrase encrypts the key on disk. If your workstation is compromised or a backup is leaked, an encrypted key buys you time to rotate. Use a password manager to store the passphrase; don't try to memorize a weak one.

```bash
# Add or change passphrase on existing key
ssh-keygen -p -f ~/.ssh/id_ed25519
```

---

## 2. Full sshd_config Hardening

The file lives at `/etc/ssh/sshd_config`. Always validate before reloading:

```bash
sshd -t                        # syntax check
sshd -T | grep -i keyword      # dump effective config (great for debugging)
systemctl reload sshd          # graceful reload — doesn't drop existing sessions
```

**Keep a second terminal open** with an active SSH session before reloading. If you break sshd config and reload fails, you can roll back without locking yourself out.

### Complete Hardened sshd_config

```
# /etc/ssh/sshd_config
# Hardened configuration — last updated: 2026-04-06
# Validate: sshd -t
# Reload:   systemctl reload sshd

##############################################################################
# NETWORK AND PROTOCOL
##############################################################################

# Port — change away from 22 to reduce automated scanner noise.
# This is NOT security by obscurity as a defense — it doesn't stop a
# determined attacker. It DOES dramatically reduce brute-force log noise
# and decreases exposure to zero-day exploits in sshd that automated
# scanners exploit on port 22. Use a port in the 1024-65535 range.
# If you have a firewall, you still need the firewall rule updated.
Port 2222

# AddressFamily — restrict to IPv4 or IPv6 if you only use one.
# inet = IPv4 only, inet6 = IPv6 only, any = both (default)
# If your infrastructure is dual-stack, use 'any'.
# If you have no IPv6, use 'inet' to avoid a listening surface you
# don't monitor.
AddressFamily inet

# ListenAddress — bind to specific interface(s) instead of all.
# If your server has a management interface and a public interface,
# bind SSH only to the management interface.
# 0.0.0.0 = all IPv4 interfaces (less safe than specifying explicitly)
ListenAddress 0.0.0.0
# ListenAddress 10.10.0.5   # example: management NIC only

# Protocol — SSHv1 has been dead for years but some old configs still
# have 'Protocol 2,1' or 'Protocol 1'. OpenSSH >= 7.0 dropped v1 support
# entirely, so this directive is now a no-op in modern OpenSSH.
# Include it for clarity if you're auditing old systems.
# Protocol 2

##############################################################################
# HOST KEYS — what the server presents to prove its identity
##############################################################################

# Host keys live in /etc/ssh/. OpenSSH generates them on first boot.
# Only list key types you actually support. Ed25519 and RSA-4096.
# Remove the ECDSA host key if you want to enforce no NIST curves.
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key
# Deliberately omitting:
#   /etc/ssh/ssh_host_ecdsa_key    — NIST curve, see section 1
#   /etc/ssh/ssh_host_dsa_key      — broken

# Regenerate host keys (run on server, not client):
#   rm /etc/ssh/ssh_host_*
#   ssh-keygen -A             # regenerates all default key types
#   OR selectively:
#   ssh-keygen -t ed25519 -f /etc/ssh/ssh_host_ed25519_key -N ""
#   ssh-keygen -t rsa -b 4096 -f /etc/ssh/ssh_host_rsa_key -N ""

##############################################################################
# AUTHENTICATION — this section is the most security-critical
##############################################################################

# PermitRootLogin — the answer is 'no'. No exceptions for production.
# The root account is the highest-value target on any system. Attackers
# know it exists (unlike regular usernames). Allowing root login means
# a compromised key or cracked password immediately grants full control.
# Use sudo/su from a regular account instead.
#
# Values:
#   yes             — allow root login with password or key (never use this)
#   without-password — allow root login with key only (still wrong)
#   prohibit-password — same as without-password (OpenSSH 7.0+ alias)
#   forced-commands-only — root can only run specific commands via keys
#   no              — root cannot SSH in at all (correct)
PermitRootLogin no

# PasswordAuthentication — disable completely in favor of public key auth.
# Password auth is vulnerable to brute force, credential stuffing,
# phishing, and password reuse. Keys are not. There is no legitimate
# production reason to enable password auth on a properly managed server.
# Exception: initial bootstrap before keys are deployed (disable immediately after).
PasswordAuthentication no

# PubkeyAuthentication — must be yes (it's the default but be explicit).
PubkeyAuthentication yes

# AuthorizedKeysFile — where to look for authorized public keys.
# Default is .ssh/authorized_keys. The second path .ssh/authorized_keys2
# is a legacy holdover from SSH protocol v2 transition — you can drop it.
# %h = home directory, %u = username.
# For centralized key management, you can point to a system-wide location:
#   AuthorizedKeysFile /etc/ssh/authorized_keys/%u
# This lets you manage keys in /etc/ssh/ outside of user home directories,
# which is useful when home dirs are on NFS or user-controlled.
AuthorizedKeysFile .ssh/authorized_keys

# PermitEmptyPasswords — never. An account with no password that can
# SSH in is an open door. This setting exists only to remind you it
# exists so you can verify it's off.
PermitEmptyPasswords no

# ChallengeResponseAuthentication — keyboard-interactive authentication.
# This is the mechanism PAM uses for TOTP/2FA prompts.
# Set to 'no' unless you're implementing 2FA via PAM (see section 4).
# If you ARE using PAM-based 2FA, set to 'yes' and configure PAM.
# Note: In OpenSSH 8.7+, this was renamed to KbdInteractiveAuthentication.
# Both names work in most versions; use the one your version supports.
ChallengeResponseAuthentication no
# KbdInteractiveAuthentication no   # OpenSSH 8.7+ equivalent

# UsePAM — enables PAM for authentication. Required for PAM-based 2FA.
# If you're not using 2FA and have PasswordAuthentication no, this can
# be 'no'. However, PAM also handles session setup (pam_limits, pam_env,
# pam_loginuid for audit logging). Leaving it 'yes' is generally safe
# and preserves account locking via pam_tally2/pam_faillock.
# If you set UsePAM no with PasswordAuthentication no, ensure you have
# another mechanism for enforcing account policies.
UsePAM yes

# AuthenticationMethods — explicitly require the authentication chain.
# 'publickey' = key only
# 'publickey,keyboard-interactive' = key THEN 2FA (if using PAM TOTP)
# 'publickey,password' = key then password (pointless, don't use)
# For 2FA deployments, see section 4 for the correct configuration.
AuthenticationMethods publickey

# HostbasedAuthentication — trusts the client's hostname to authenticate.
# This is legacy, fragile, and depends on /etc/hosts.equiv or .rhosts.
# Always off.
HostbasedAuthentication no
IgnoreRhosts yes

##############################################################################
# SESSION AND CONNECTION CONTROLS
##############################################################################

# X11Forwarding — forwards the X11 display to the client.
# X11 forwarding has a long history of security issues. The forwarded
# connection can be used to inject keystrokes and read screen contents
# on the client. On headless servers (every production server), this
# should be off. If your developers need GUI forwarding, use a jump
# host or VNC/RDP with proper controls, or accept the risk explicitly.
X11Forwarding no

# AllowTcpForwarding — allows SSH tunnels (port forwarding).
# 'yes' = any tunnel direction (local -L, remote -R, dynamic -D)
# 'local' = only local port forwarding (-L)
# 'no' = no tunneling at all
# Tunneling bypasses firewalls. If your firewall policy restricts traffic
# to SSH, AllowTcpForwarding yes means an attacker (or rogue insider)
# can forward any port through the SSH connection and bypass those rules.
# Disable unless you have a specific operational need (bastion hosts,
# database jump connections, etc.).
AllowTcpForwarding no

# GatewayPorts — allows remote port forwards to bind on all interfaces
# (not just localhost) on the server. Default is 'no' (bind localhost only).
# This is dangerous — it lets SSH clients expose ports on your server's
# public interface. Almost never what you want.
GatewayPorts no

# PermitTunnel — controls TUN/TAP device forwarding (VPN over SSH).
# Unless you're running SSH-based VPN tunnels deliberately, turn this off.
PermitTunnel no

# PrintMotd — whether sshd prints /etc/motd on login.
# Disabling here if your shell's PAM config already prints it (avoids
# double-printing). Irrelevant to security but worth setting correctly.
PrintMotd no

# PrintLastLog — prints last login time/IP on login.
# Leave this on — it's a useful security signal (tells users if someone
# else has logged in as them).
PrintLastLog yes

# Banner — display a legal warning before authentication.
# Recommended for compliance (PCI-DSS, HIPAA, SOX all want warning banners).
# Create the file with your legal department's approved text.
# Banner /etc/ssh/banner.txt

##############################################################################
# TIMEOUTS AND LIMITS
##############################################################################

# LoginGraceTime — how long to wait for successful authentication before
# dropping the connection. Default is 120 seconds (way too long).
# An attacker can hold connections open for 2 minutes each, facilitating
# DoS. 30 seconds is ample for legitimate users.
LoginGraceTime 30

# MaxAuthTries — maximum authentication attempts per connection.
# After MaxAuthTries/2 failures, each failure is logged.
# After MaxAuthTries failures, the connection is dropped.
# Default is 6. Set to 3 for aggressive reduction of brute force window.
# Note: this is per TCP connection — fail2ban (section 7) handles the
# multi-connection case.
MaxAuthTries 3

# MaxSessions — maximum number of open sessions per network connection.
# With multiplexing (ControlMaster), one TCP connection can host many
# sessions. Limiting this prevents one compromised connection from
# becoming a multi-session pivot point.
MaxSessions 4

# MaxStartups — throttle unauthenticated connections.
# Format: start:rate:full
# start = connections before random drop kicks in
# rate  = percentage chance of dropping new connection
# full  = refuse all new connections above this count
# 10:30:60 means: up to 10 unauthenticated connections are fine;
# between 10 and 60, drop 30% of new ones randomly;
# at 60, refuse all new connections.
# This limits SYN-flood style attacks on SSH.
MaxStartups 10:30:60

# ClientAliveInterval — send a keepalive packet after this many seconds
# of silence from client. The client must respond.
# This detects dead connections (crashed clients, dropped NAT entries).
ClientAliveInterval 300

# ClientAliveCountMax — how many unanswered keepalives before disconnecting.
# With Interval=300 and Count=2, a dead session is dropped in ~10 minutes.
# This is a reasonable balance between keeping long-running sessions alive
# and cleaning up ghost connections.
ClientAliveCountMax 2

# TCPKeepAlive — OS-level TCP keepalive. This is different from SSH-level
# keepalives (ClientAlive*). TCP keepalive operates at the kernel level
# and can detect network failures, but is not encrypted and can be spoofed.
# Prefer ClientAlive* over TCPKeepAlive. Set TCPKeepAlive no since
# ClientAlive* covers the need more reliably.
TCPKeepAlive no

##############################################################################
# ACCESS CONTROL — whitelist approach
##############################################################################

# AllowUsers — whitelist of users who can SSH in.
# If set, ONLY these users can authenticate. All others are rejected
# regardless of key validity. This is a belt-and-suspenders control
# that catches misconfigured accounts.
# Supports glob patterns: admin* matches admin, admin1, admintools, etc.
# Can specify user@host to restrict a user to connections from a specific host.
AllowUsers deploy mario svc-monitoring

# AllowGroups — whitelist by group membership.
# Useful when you have many users and a 'sshusers' group policy.
# AllowUsers and AllowGroups are OR'd together — being in either list grants access.
# AllowGroups sshusers sudo

# DenyUsers / DenyGroups — blacklist approach.
# Processed BEFORE AllowUsers/AllowGroups. A user in DenyUsers is
# rejected even if they're also in AllowUsers.
# Useful for quickly blocking a compromised account without removing it.
# DenyUsers badactor compromised-account
# DenyGroups noSSH

# NOTE on processing order:
# DenyUsers → AllowUsers → DenyGroups → AllowGroups
# First match wins. If you use both Allow and Deny lists, test carefully.

##############################################################################
# CRYPTOGRAPHIC ALGORITHMS
##############################################################################

# These three directives are where you lock down the cryptographic surface.
# The goal: remove anything with known weaknesses, prefer modern algorithms.
# Use 'ssh-audit' (section 6) to validate what your server actually offers.
#
# All values below are for OpenSSH 8.x/9.x. Check 'man sshd_config' and
# 'ssh -Q cipher|mac|kex' for your version's supported list.

# Ciphers — symmetric encryption algorithms for the session.
# Prefer ChaCha20-Poly1305 (stream cipher + AEAD, designed by Bernstein,
# no NIST involvement, immune to CBC padding oracle attacks).
# AES-GCM is also good (authenticated encryption, hardware-accelerated on
# modern CPUs with AES-NI). AES-CTR is acceptable but lacks authentication.
# Remove: arcfour (RC4, broken), blowfish (64-bit block), 3des-cbc (meet-
# in-the-middle attacks), aes*-cbc (BEAST/POODLE variants).
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr

# MACs — message authentication codes for data integrity.
# Prefer -etm (Encrypt-then-MAC) variants — they authenticate the ciphertext
# before decryption, preventing padding oracle attacks that -mac variants
# are vulnerable to. The ETM variants are strictly better.
# hmac-sha2-512-etm is the gold standard. hmac-sha2-256-etm is also good.
# Remove: hmac-md5 (MD5 is broken), hmac-sha1 (collision attacks), any
# non-ETM variant if you can get away with it.
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com

# KexAlgorithms — key exchange algorithms used at session establishment.
# This is where the session key is negotiated. Use Diffie-Hellman with
# Curve25519 (sntrup761x25519 = post-quantum hybrid, excellent if supported)
# or curve25519-sha256. These avoid NIST curves and classical DH with
# known-weak groups (diffie-hellman-group1-sha1 = Logjam attack target).
# Remove: diffie-hellman-group1-sha1 (Logjam), diffie-hellman-group14-sha1
# (SHA1), ecdh-sha2-nistp* (NIST curves), diffie-hellman-group-exchange-sha1.
KexAlgorithms sntrup761x25519-sha512@openssh.com,curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512

# HostKeyAlgorithms — which host key algorithms the server will use and
# advertise. Should match your HostKey files.
HostKeyAlgorithms ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-256

# PubkeyAcceptedAlgorithms (OpenSSH 8.5+) or PubkeyAcceptedKeyTypes —
# which public key algorithms are accepted for user auth.
# This restricts what key types users can authenticate with.
PubkeyAcceptedAlgorithms ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-256

##############################################################################
# LOGGING
##############################################################################

# SyslogFacility — where auth events are sent.
# AUTH is standard. AUTHPRIV is more private (readable only by root) on
# systems that separate it. Either is fine; make sure your log aggregator
# picks up the right facility.
SyslogFacility AUTH

# LogLevel — verbosity of SSH logging.
# INFO is reasonable for production (logs auth success/failure, key used).
# VERBOSE adds fingerprints to auth logs — very useful for auditing which
# key was used when multiple keys exist.
# DEBUG* is for troubleshooting only — it logs too much for production.
LogLevel VERBOSE

##############################################################################
# SUBSYSTEMS
##############################################################################

# Subsystem — defines subsystem handlers. sftp is the most common.
# Use the internal-sftp subsystem (compiled into sshd) instead of
# the external /usr/lib/openssh/sftp-server binary.
# internal-sftp supports ChrootDirectory and is the correct choice
# for chrooted SFTP-only accounts.
Subsystem sftp internal-sftp

##############################################################################
# MATCH BLOCKS — per-user, per-group, per-address overrides
##############################################################################

# Match blocks allow overriding global settings for specific conditions.
# They are evaluated AFTER the global config. Only a subset of directives
# can be used in Match blocks.

# Example: SFTP-only chrooted users
# Match Group sftponly
#     ChrootDirectory /srv/sftp/%u
#     ForceCommand internal-sftp
#     AllowTcpForwarding no
#     X11Forwarding no

# Example: Allow password auth from management network only
# Match Address 10.10.0.0/24
#     PasswordAuthentication yes

# Example: Stricter settings for internet-facing auth
# Match Address *,!10.0.0.0/8,!192.168.0.0/16
#     MaxAuthTries 2
#     LoginGraceTime 15
```

### Applying the Config

```bash
# 1. Edit the config
sudo nano /etc/ssh/sshd_config

# 2. Test for syntax errors — ALWAYS do this before reloading
sudo sshd -t
# No output = no syntax errors

# 3. Check effective config (dump all active settings)
sudo sshd -T | less

# 4. Reload (graceful — existing sessions survive)
sudo systemctl reload sshd

# 5. Verify the process restarted cleanly
sudo systemctl status sshd
sudo journalctl -u sshd --since "1 minute ago"
```

---

## 3. SSH Certificate Authority for Fleet Management

Managing `authorized_keys` files individually across a large fleet is a maintenance nightmare. Keys accumulate, departed employees' keys linger, and there's no central revocation mechanism. SSH certificates solve this at scale.

### How SSH Certificates Work

Instead of copying a public key to every server's `authorized_keys`, you:
1. Create a CA key pair (one time, keep private key locked up tight).
2. Sign each user's public key with the CA, producing a certificate.
3. Configure servers to trust the CA — one line in `sshd_config`.
4. When a user authenticates, the server verifies the certificate was signed by the trusted CA.

Revoking access = invalidating the certificate (via expiry or KRL). No more hunting down `authorized_keys` entries across 500 servers.

### Step 1: Create the CA Key Pair

```bash
# Do this on a dedicated, offline (or highly restricted) system.
# The CA private key must never be on a server or workstation with
# internet access. Treat it like a root certificate authority.

# User CA — signs user keys (used by servers to authenticate users)
ssh-keygen -t ed25519 -f /etc/ssh/ca/user_ca -C "user-ca-prod-2026"

# Host CA — signs server host keys (used by clients to verify servers)
ssh-keygen -t ed25519 -f /etc/ssh/ca/host_ca -C "host-ca-prod-2026"

# Set strict permissions
chmod 600 /etc/ssh/ca/user_ca /etc/ssh/ca/host_ca
chmod 644 /etc/ssh/ca/user_ca.pub /etc/ssh/ca/host_ca.pub
```

### Step 2: Sign User Keys

```bash
# Sign mario's public key with the user CA
# -s = signing key (the CA private key)
# -I = identity string (appears in audit logs — make it meaningful)
# -n = principals (comma-separated list of usernames this cert is valid for)
# -V = validity period (+52w = 52 weeks from now; use short windows)
# -z = serial number (for revocation lists)

ssh-keygen -s /etc/ssh/ca/user_ca \
    -I "mario@workstation-2026" \
    -n mario,deploy \
    -V +52w \
    -z 1001 \
    ~/.ssh/id_ed25519.pub

# This produces ~/.ssh/id_ed25519-cert.pub
# The user presents this cert during authentication

# Verify the certificate
ssh-keygen -L -f ~/.ssh/id_ed25519-cert.pub
```

Example certificate inspection output:
```
/home/mario/.ssh/id_ed25519-cert.pub:
        Type: ssh-ed25519-cert-v01@openssh.com user certificate
        Public key: ED25519-CERT SHA256:abc123...
        Signing CA: ED25519 SHA256:xyz789... (using ssh-ed25519)
        Key ID: "mario@workstation-2026"
        Serial: 1001
        Valid: from 2026-04-06T00:00:00 to 2027-04-05T00:00:00
        Principals:
                mario
                deploy
        Critical Options: (none)
        Extensions:
                permit-pty
                permit-user-rc
                permit-X11-forwarding
                permit-agent-forwarding
                permit-port-forwarding
```

### Step 3: Configure Servers to Trust the User CA

```bash
# On every server — add to sshd_config:
echo "TrustedUserCAKeys /etc/ssh/ca/user_ca.pub" >> /etc/ssh/sshd_config

# Copy the CA public key to the server
scp /etc/ssh/ca/user_ca.pub server:/etc/ssh/ca/user_ca.pub
chmod 644 /etc/ssh/ca/user_ca.pub

# Reload sshd
systemctl reload sshd
```

In `sshd_config`:
```
# Trust certificates signed by this CA for user authentication
TrustedUserCAKeys /etc/ssh/ca/user_ca.pub
```

### Step 4: Sign Host Keys

```bash
# Sign each server's host key with the host CA
# -h = sign as a host certificate (not user certificate)
# -n = hostname(s) the certificate is valid for (FQDN and/or short name)

ssh-keygen -s /etc/ssh/ca/host_ca \
    -I "webserver-01.prod.example.com" \
    -h \
    -n "webserver-01,webserver-01.prod.example.com,10.0.1.10" \
    -V +52w \
    -z 2001 \
    /etc/ssh/ssh_host_ed25519_key.pub

# This produces /etc/ssh/ssh_host_ed25519_key-cert.pub
```

Tell sshd to use the host certificate:
```
# In sshd_config
HostCertificate /etc/ssh/ssh_host_ed25519_key-cert.pub
```

### Step 5: Configure Clients to Trust the Host CA

```bash
# In ~/.ssh/known_hosts (or /etc/ssh/ssh_known_hosts for system-wide):
# Format: @cert-authority <pattern> <CA public key>

echo "@cert-authority *.prod.example.com $(cat /etc/ssh/ca/host_ca.pub)" \
    >> ~/.ssh/known_hosts
```

This tells SSH clients: any server matching `*.prod.example.com` that presents a host certificate signed by `host_ca` is trusted. No more "are you sure you want to continue connecting?" prompts for new servers, and no more stale fingerprints when you rebuild a server.

### Certificate Revocation

SSH certificates support a Key Revocation List (KRL):

```bash
# Create a KRL (empty initially)
ssh-keygen -k -f /etc/ssh/ca/revoked_keys

# Add a certificate to the KRL by serial number
ssh-keygen -k -f /etc/ssh/ca/revoked_keys -u -z 1001

# Add a key file directly
ssh-keygen -k -f /etc/ssh/ca/revoked_keys -u /path/to/compromised_key.pub

# Reference the KRL in sshd_config
echo "RevokedKeys /etc/ssh/ca/revoked_keys" >> /etc/ssh/sshd_config
```

Distribute the KRL to all servers. This is the mechanism for immediate revocation when a key is compromised.

### Certificate Validity Best Practices

- **User certs:** 90 days to 1 year. Shorter = more rotation burden. Match your org's access review cycle.
- **Host certs:** 1-2 years. Hosts don't change identity. Automate renewal via cron.
- **Emergency certs:** 24-hour window for break-glass access.
- **Never issue certs without expiry** (`-V +unlimited`) — this defeats the revocation model.
- Use a serial number sequence so you can build a complete certificate inventory.

---

## 4. Two-Factor Authentication with PAM

2FA adds a second factor (TOTP code) after public key authentication. Even if an attacker steals a private key, they still need the TOTP device (phone/hardware token) to authenticate.

### Install google-authenticator-libpam

```bash
# Debian/Ubuntu
sudo apt install libpam-google-authenticator

# RHEL/Rocky/AlmaLinux
sudo dnf install google-authenticator

# Fedora
sudo dnf install google-authenticator
```

### User Setup

Each user who needs 2FA runs:

```bash
google-authenticator
```

Answer the prompts:
- Time-based tokens: **yes**
- Update ~/.google_authenticator: **yes**
- Disallow multiple uses of same token: **yes**
- Allow 30-second clock skew window: **yes** (accounts for small clock drift)
- Rate limiting (3 attempts/30s): **yes**

This writes `~/.google_authenticator` with the TOTP secret, emergency scratch codes, and settings. Back up the scratch codes.

### PAM Configuration

Edit `/etc/pam.d/sshd`:

```
# /etc/pam.d/sshd — modified for 2FA
# The order matters. Top-down processing.

# Standard auth stack — comment out or keep depending on your setup.
# If using keyboard-interactive for 2FA only (not password), you can
# keep @include common-auth as-is since PasswordAuthentication no
# means PAM password auth won't be triggered anyway.
@include common-auth

# Add Google Authenticator TOTP as required auth
# nullok = allow login without OTP if user hasn't set up 2FA yet
#          (remove 'nullok' once all users are enrolled)
# secret = path to the secret file (default is ~/.google_authenticator)
auth required pam_google_authenticator.so nullok

@include common-account
@include common-session
@include common-password
```

### sshd_config for 2FA

For publickey + TOTP, the user must:
1. Authenticate with their SSH key (publickey phase)
2. Enter their TOTP code (keyboard-interactive phase)

```
# In sshd_config:

# Enable keyboard-interactive (for the TOTP prompt)
ChallengeResponseAuthentication yes
# OpenSSH 8.7+:
# KbdInteractiveAuthentication yes

# Keep PAM enabled
UsePAM yes

# Require BOTH key auth AND keyboard-interactive (TOTP)
# This is the critical line — comma means AND (sequential)
AuthenticationMethods publickey,keyboard-interactive

# Do NOT have PasswordAuthentication yes here — keyboard-interactive
# is NOT password auth; it's the PAM challenge-response chain (TOTP)
PasswordAuthentication no
```

### Testing 2FA

```bash
# Test from client — you should be prompted for key passphrase (if any),
# then the TOTP code
ssh -v mario@server -p 2222

# Watch the server logs in real time
sudo journalctl -fu sshd
```

Expected log sequence:
```
Accepted publickey for mario from 10.0.1.5 port 54321 ssh2: ED25519 SHA256:...
Postponed keyboard-interactive for mario from 10.0.1.5 port 54321 ssh2
Accepted keyboard-interactive/pam for mario from 10.0.1.5 port 54321 ssh2
```

### Emergency Access if TOTP is Lost

Keep scratch codes from the `google-authenticator` setup in a secure password manager. If a user loses their TOTP device:

```bash
# As root — view (or remove) their google-authenticator config
# Temporary: disable 2FA for that user by removing their config
sudo mv /home/mario/.google_authenticator /home/mario/.google_authenticator.bak

# User can then log in with key only (if AuthenticationMethods has a fallback)
# Better: set up a Match block for emergency access from a specific IP
```

---

## 5. Client-Side ~/.ssh/config Best Practices

The client config at `~/.ssh/config` (or `/etc/ssh/ssh_config` system-wide) controls how the `ssh` client behaves. A well-crafted config eliminates repetitive flags and enforces security defaults.

### File Structure and Permissions

```bash
# Permissions must be right or SSH ignores the file
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/known_hosts
chmod 600 ~/.ssh/authorized_keys    # on servers
```

### Full ~/.ssh/config Example

```
# ~/.ssh/config
# SSH client configuration
# Order matters: first matching Host block wins for each directive.

##############################################################################
# GLOBAL DEFAULTS — apply to all connections unless overridden in a Host block
##############################################################################

Host *
    # Always prefer Ed25519, then RSA. This sets the order for key offer.
    IdentityFile ~/.ssh/id_ed25519
    IdentityFile ~/.ssh/id_rsa

    # Multiplexing — reuse an existing connection for subsequent sessions.
    # ControlMaster auto = create master connection on first connect, reuse after.
    # ControlPath = socket file for the multiplex control channel.
    # ControlPersist = keep the master connection alive this long after last
    # session closes (60s gives you time to reconnect quickly without hanging
    # a long-lived connection open indefinitely).
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h:%p
    ControlPersist 60s

    # Keepalives — send a packet every 60 seconds to prevent NAT timeout.
    # ServerAliveInterval complements (and works like) ClientAliveInterval
    # on the server side — this is the CLIENT sending keepalives.
    ServerAliveInterval 60
    ServerAliveCountMax 3

    # Use SSH certificate if present alongside the key
    # (automatically detected by ssh if -cert.pub file exists)

    # Disable forwarding by default — enable per-host when needed
    ForwardAgent no
    ForwardX11 no

    # Reject connections if host key is not in known_hosts
    # StrictHostKeyChecking yes  — safest; interactive sessions may annoy users
    # StrictHostKeyChecking ask  — prompt (default; reasonable)
    # StrictHostKeyChecking no   — never use in production scripts; MITM risk
    StrictHostKeyChecking ask

    # Hash known_hosts entries (see section 8)
    HashKnownHosts yes

    # Preferred algorithms — match your server's config
    Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
    MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com
    KexAlgorithms sntrup761x25519-sha512@openssh.com,curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512

    # Visual host key — display an ASCII art fingerprint on connect.
    # Humans are good at spotting visual anomalies; MITM'd keys look different.
    VisualHostKey yes

##############################################################################
# PRODUCTION SERVERS — direct access
##############################################################################

Host prod-web-01
    HostName webserver-01.prod.example.com
    Port 2222
    User deploy
    IdentityFile ~/.ssh/id_ed25519

Host prod-db-01
    HostName 10.0.2.10
    Port 2222
    User mario
    IdentityFile ~/.ssh/id_ed25519
    # No TCP forwarding needed — connect to DB via application, not tunnel

##############################################################################
# BASTION / JUMP HOST
##############################################################################

# The bastion host is your entry point into a private network.
# All connections to internal hosts should go through it.

Host bastion
    HostName bastion.example.com
    Port 22
    User mario
    IdentityFile ~/.ssh/id_ed25519
    ControlMaster auto
    ControlPath ~/.ssh/sockets/bastion_%r@%h:%p
    ControlPersist 10m    # keep bastion connection alive longer for tunneling

# ProxyJump — connect to internal-host by jumping through bastion.
# This replaces the old ProxyCommand with netcat. Much cleaner.
# ssh internal-host-01 will automatically connect through bastion.
Host internal-*
    User mario
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump bastion

Host internal-web-01
    HostName 10.10.1.5
    ProxyJump bastion

# Multi-hop: jump through bastion, then through an intermediate host
# Host deep-internal
#     HostName 192.168.100.5
#     ProxyJump bastion,internal-web-01

##############################################################################
# LEGACY / SPECIAL CASES
##############################################################################

# Old network appliance that only supports older algorithms
Host legacy-switch
    HostName 10.0.0.1
    User admin
    KexAlgorithms +diffie-hellman-group14-sha256
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
    # Do not use these settings globally — isolate them to this host

# GitHub — specific key, no agent forwarding needed
Host github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github
    ForwardAgent no
    ControlMaster no    # GitHub connections shouldn't be multiplexed
```

### Create the Socket Directory

```bash
mkdir -p ~/.ssh/sockets
chmod 700 ~/.ssh/sockets
```

### ForwardAgent Warning

`ForwardAgent yes` forwards your SSH agent to the remote server. This lets you SSH from the remote server to other hosts using your local keys. It is convenient and it is dangerous. If the remote server is compromised, the attacker can use your forwarded agent to authenticate to any other host your key grants access to.

Rules:
- Never enable ForwardAgent globally (`Host *`).
- Enable it only on trusted bastion hosts where you explicitly need it.
- Use ProxyJump instead of ForwardAgent for multi-hop connections — ProxyJump doesn't expose the agent to intermediate hosts.

---

## 6. ssh-audit: Auditing Your SSH Configuration

`ssh-audit` (by jtesta, maintained at https://github.com/jtesta/ssh-audit) connects to an SSH server and audits its advertised algorithms, checks for known weaknesses, and rates the configuration. It does NOT authenticate — it only examines the pre-auth handshake. Run it against your servers periodically and after config changes.

### Installation

```bash
# Method 1: pip (recommended — gets latest version)
pip3 install ssh-audit

# Method 2: From source
git clone https://github.com/jtesta/ssh-audit.git
cd ssh-audit
python3 ssh_audit.py --help

# Method 3: Docker
docker pull positronsecurity/ssh-audit
docker run -it --rm positronsecurity/ssh-audit <target>

# Method 4: Distro packages (may be older version)
# Ubuntu: apt install ssh-audit
# Note: distro packages are often 1-2 versions behind

# Verify installation
ssh-audit --version
```

### Running ssh-audit

```bash
# Basic scan — connect to server on default port 22
ssh-audit 192.168.1.10

# Specify port
ssh-audit -p 2222 192.168.1.10

# Scan with hostname
ssh-audit webserver.example.com

# Output as JSON (for automated processing)
ssh-audit --json webserver.example.com

# Output as CSV
ssh-audit --csv webserver.example.com

# Scan multiple hosts from a file
ssh-audit -T targets.txt

# Test client configuration (listen mode — audit what YOUR ssh client offers)
ssh-audit -L
# Then: ssh -p 2222 localhost

# Generate a remediation script (OpenSSH)
ssh-audit --fix webserver.example.com
```

### Understanding the Output

ssh-audit categorizes findings by color and severity:

```
# Example output (abbreviated)
# [color codes: green=good, yellow=warn, red=fail, purple=info]

(gen) banner: SSH-2.0-OpenSSH_9.3
(gen) software: OpenSSH 9.3
(gen) compatibility: OpenSSH 7.4+

# Host keys
(rec) -ssh-rsa                  -- `ssh-rsa` is disabled (sha1 hash)
(good) ssh-ed25519              -- available
(good) rsa-sha2-512             -- available

# Key exchange algorithms
(fail) diffie-hellman-group1-sha1   -- using Oakley Group 2 (1024-bit) -- vulnerable to Logjam
(warn) diffie-hellman-group14-sha1  -- using SHA-1 hash
(good) curve25519-sha256            -- good
(good) sntrup761x25519-sha512@openssh.com -- post-quantum hybrid, excellent

# Encryption algorithms (ciphers)
(fail) arcfour                  -- broken RC4 stream cipher
(warn) aes128-cbc               -- CBC mode -- potentially vulnerable to BEAST, Lucky13
(good) chacha20-poly1305@openssh.com -- excellent
(good) aes256-gcm@openssh.com   -- excellent

# Message authentication codes
(fail) hmac-md5                 -- broken MD5 hash
(warn) hmac-sha1                -- weak SHA-1
(good) hmac-sha2-512-etm@openssh.com -- encrypt-then-mac, excellent
(good) hmac-sha2-256-etm@openssh.com -- encrypt-then-mac, good

# Recommendations
(rec) Consider adding these kex algorithms: sntrup761x25519-sha512@openssh.com
(rec) Remove these ciphers: aes128-cbc, aes192-cbc, aes256-cbc
```

### Reading the Severity Levels

| Tag | Meaning | Action Required |
|-----|---------|-----------------|
| `(good)` | Algorithm is secure and recommended | Keep it |
| `(info)` | Informational, not a finding | Review |
| `(warn)` | Weak or deprecated, not immediately exploitable | Plan to remove |
| `(fail)` | Known vulnerable or broken | Remove immediately |
| `(rec)` | Recommended to add/change | Implement |

### Interpreting a Green Score

ssh-audit produces a policy score. A well-hardened server should show:
- All ciphers: chacha20 or AES-GCM only → no `(fail)` or `(warn)` in ciphers
- MACs: all ETM variants → no non-ETM, no MD5, no SHA1
- KEX: Curve25519 and/or sntrup761 → no DH group1/group14 with SHA1
- Host keys: ed25519 → no DSA, no RSA-SHA1

### Automated Auditing in CI/CD

```bash
#!/bin/bash
# audit-ssh.sh — run in CI or monitoring cron

TARGET="$1"
PORT="${2:-2222}"

RESULT=$(ssh-audit --json -p "$PORT" "$TARGET" 2>/dev/null)
FAILURES=$(echo "$RESULT" | python3 -c "
import json, sys
data = json.load(sys.stdin)
fails = [r for r in data.get('recommendations', []) if r['level'] == 'fail']
print(len(fails))
")

if [ "$FAILURES" -gt 0 ]; then
    echo "CRITICAL: $FAILURES failures on $TARGET:$PORT"
    ssh-audit -p "$PORT" "$TARGET"
    exit 1
fi
echo "OK: $TARGET:$PORT passes ssh-audit"
```

---

## 7. fail2ban SSH Jail Configuration

fail2ban monitors log files and bans IP addresses that show signs of brute-force attacks. For SSH, it watches for repeated authentication failures and uses iptables/nftables to block offending IPs.

### Installation

```bash
# Debian/Ubuntu
sudo apt install fail2ban

# RHEL/Rocky
sudo dnf install fail2ban

# Enable and start
sudo systemctl enable --now fail2ban
```

### Configuration Philosophy

Never edit `/etc/fail2ban/jail.conf` directly — it's overwritten on upgrades. Create `/etc/fail2ban/jail.local` to override settings.

### /etc/fail2ban/jail.local

```ini
# /etc/fail2ban/jail.local

[DEFAULT]
# Ban duration in seconds. 3600 = 1 hour.
# For persistent offenders, use 'bantime.increment = true' below.
bantime  = 3600

# Time window in which maxretry failures must occur to trigger a ban
findtime = 600

# Number of failures before banning
maxretry = 3

# Backend for log monitoring. 'auto' works for most systems.
# 'systemd' is better on systemd-based distros (reads journal directly)
backend = systemd

# Email notifications (optional)
# destemail = root@localhost
# sendername = fail2ban
# mta = sendmail
# action = %(action_mwl)s  # ban + email with whois + log lines

# Progressive ban times — increases bantime for repeat offenders
bantime.increment = true
bantime.multiplier = 2
bantime.maxtime = 86400  # cap at 24 hours

# Whitelist your management IPs — never ban these
ignoreip = 127.0.0.1/8 ::1 10.0.0.0/8 192.168.0.0/16


##############################################################################
# SSH JAIL
##############################################################################

[sshd]
enabled  = true
port     = 2222          # match your sshd Port directive
filter   = sshd
logpath  = /var/log/auth.log    # Debian/Ubuntu
# logpath = /var/log/secure    # RHEL/Rocky
backend  = systemd              # if using systemd journal
maxretry = 3
findtime = 600
bantime  = 3600

# Additional aggressive rules for known-bad patterns
[sshd-ddos]
enabled  = true
port     = 2222
filter   = sshd-ddos
logpath  = /var/log/auth.log
maxretry = 2
findtime = 120
bantime  = 7200
```

### Custom Filter for Aggressive Banning

Create `/etc/fail2ban/filter.d/sshd-extra.conf`:

```ini
# /etc/fail2ban/filter.d/sshd-extra.conf
# Catch additional SSH attack patterns

[Definition]
_daemon = sshd

failregex = ^%(__prefix_line)s[iI]nvalid user .+ from <HOST> port \d+\s*$
            ^%(__prefix_line)sUser .+ from <HOST> not allowed because not listed in AllowUsers\s*$
            ^%(__prefix_line)sConnection closed by invalid user .+ <HOST> port \d+ \[preauth\]\s*$
            ^%(__prefix_line)sFailed \S+ for .* from <HOST> port \d+\s*$
            ^%(__prefix_line)sRefused connect from <HOST>\s*$

ignoreregex =
```

### fail2ban Operations

```bash
# Check status of all jails
sudo fail2ban-client status

# Check SSH jail status (shows active bans)
sudo fail2ban-client status sshd

# Manually ban an IP
sudo fail2ban-client set sshd banip 203.0.113.42

# Manually unban an IP (careful with ignoreip)
sudo fail2ban-client set sshd unbanip 203.0.113.42

# Test a filter against a log file
sudo fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf

# Reload config after changes
sudo fail2ban-client reload

# View recent bans in the database
sudo fail2ban-client get sshd banlist

# Check ban counts and persistent offenders
sudo sqlite3 /var/lib/fail2ban/fail2ban.sqlite3 \
    "SELECT ip, failures, timeofban FROM bips ORDER BY failures DESC LIMIT 20;"
```

### iptables vs nftables Backend

Modern distros use nftables. fail2ban supports both:

```ini
# In jail.local [DEFAULT]:
# For iptables (older systems)
banaction = iptables-multiport

# For nftables (modern systems — Ubuntu 22.04+, RHEL 9+)
banaction = nftables-multiport
banaction_allports = nftables-allports
```

---

## 8. known_hosts Management

`~/.ssh/known_hosts` stores fingerprints of SSH servers you've connected to. It's your protection against MITM attacks — if a server's fingerprint changes (legitimate rebuild vs attacker interposing), SSH warns you.

### Hashing known_hosts

By default, known_hosts stores plaintext hostnames:
```
webserver.example.com ssh-ed25519 AAAA...
```

An attacker who reads your known_hosts file now knows which hosts you connect to. Hashing the entries obscures this:
```
|1|abc123==|xyz456== ssh-ed25519 AAAA...
```

Enable hashing:
```
# In ~/.ssh/config:
HashKnownHosts yes
```

This applies to NEW entries. Hash existing entries:
```bash
ssh-keygen -H -f ~/.ssh/known_hosts
# Remove old plaintext backup
rm ~/.ssh/known_hosts.old
```

Downside: you can't grep known_hosts for a hostname anymore. Use `ssh-keygen -F` instead:

```bash
# Find a host's entry in known_hosts
ssh-keygen -F webserver.example.com

# Find by IP
ssh-keygen -F 10.0.1.5
```

### Removing Stale or Changed Entries

```bash
# Remove a single host's entry (all key types)
ssh-keygen -R webserver.example.com
ssh-keygen -R 10.0.1.5

# Remove by hostname and IP (when a server has both)
ssh-keygen -R "[hostname]:2222"   # non-standard port

# After rebuilding a server — clear old key, add new one
ssh-keygen -R old-server.example.com
ssh-keyscan -H -p 2222 old-server.example.com >> ~/.ssh/known_hosts
```

### Adding Host Keys Securely

Never blindly accept a host key over an untrusted network. The right way to add a new server's key:

```bash
# Option 1: Compare fingerprint out-of-band
# On the server:
ssh-keygen -l -f /etc/ssh/ssh_host_ed25519_key.pub
# Output: 256 SHA256:abc123xyz... /etc/ssh/ssh_host_ed25519_key.pub (ED25519)
# Verify this fingerprint matches what the client sees on first connect.

# Option 2: Use ssh-keyscan to pre-populate known_hosts
# (do this from a trusted network or verify the fingerprint afterward)
ssh-keyscan -H -t ed25519 -p 2222 webserver.example.com >> ~/.ssh/known_hosts

# Option 3: Use SSH CA (section 3) — eliminates per-host fingerprint management
```

### System-Wide known_hosts

For fleet management, maintain a system-wide known_hosts file:

```bash
# /etc/ssh/ssh_known_hosts — applies to all users on the system
# Populate with ssh-keyscan across your fleet
for host in webserver-{01..20}.prod.example.com; do
    ssh-keyscan -H -t ed25519 -p 2222 "$host"
done >> /etc/ssh/ssh_known_hosts

# Or use your SSH CA (the @cert-authority line covers all signed hosts)
echo "@cert-authority *.prod.example.com $(cat /etc/ssh/ca/host_ca.pub)" \
    >> /etc/ssh/ssh_known_hosts
```

### Detecting MITM Attacks

If SSH warns you of a changed host key:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

Do NOT blindly run `ssh-keygen -R` and reconnect. Investigate:

1. Was the server legitimately rebuilt? Check with the team.
2. Is there a load balancer rotating between multiple servers (different host keys)?
3. Is there a DNS change that points the name at a different IP?
4. Could this be a genuine MITM attack?

Only after confirming it's legitimate should you clear and re-add the key.

---

## 9. Operational Checklist

Use this checklist when deploying a new server or auditing an existing one.

### Initial Server Setup

```bash
# [ ] Generate Ed25519 user key (on your workstation)
ssh-keygen -t ed25519 -C "mario@workstation $(date +%Y-%m-%d)"

# [ ] Copy key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 22 user@server

# [ ] Verify key-based auth works BEFORE disabling passwords
ssh -i ~/.ssh/id_ed25519 -p 22 user@server "echo 'key auth works'"

# [ ] Apply hardened sshd_config (section 2)
# [ ] Run sshd -t to validate
# [ ] Open a second SSH session (failsafe)
# [ ] systemctl reload sshd
# [ ] Test connection from another terminal

# [ ] Update firewall rules if Port changed
# UFW:
sudo ufw allow 2222/tcp
sudo ufw delete allow 22/tcp

# iptables:
sudo iptables -A INPUT -p tcp --dport 2222 -j ACCEPT
sudo iptables -D INPUT -p tcp --dport 22 -j ACCEPT

# nftables:
sudo nft add rule inet filter input tcp dport 2222 accept

# [ ] Install and configure fail2ban (section 7)
# [ ] Run ssh-audit and resolve all (fail) findings (section 6)
# [ ] If fleet-managed: enroll host in SSH CA (section 3)
# [ ] If 2FA required: install google-authenticator and configure PAM (section 4)
```

### Audit Existing Server

```bash
# [ ] Run ssh-audit
ssh-audit -p 2222 server.example.com

# [ ] Check effective sshd config
sudo sshd -T | grep -E "permitroot|passwordauth|pubkeyauth|allowtcp|x11forward|maxauthtries|logingrace"

# [ ] Verify no DSA or ECDSA host keys are loaded
sudo sshd -T | grep "hostkey"

# [ ] Check authorized_keys for old/unknown keys
for user in $(cut -d: -f1 /etc/passwd); do
    keyfile="/home/$user/.ssh/authorized_keys"
    [ -f "$keyfile" ] && echo "=== $user ===" && cat "$keyfile"
done

# [ ] Check for root authorized_keys
sudo cat /root/.ssh/authorized_keys 2>/dev/null

# [ ] Verify fail2ban is running and banning
sudo fail2ban-client status sshd

# [ ] Check recent auth failures
sudo grep "Failed password\|Invalid user" /var/log/auth.log | tail -20
# or on systemd:
sudo journalctl -u sshd | grep -E "Failed|Invalid" | tail -20

# [ ] Verify no extra listening services on SSH port
sudo ss -tlnp | grep sshd
```

### Key Rotation

```bash
# Rotate user key:
# 1. Generate new key
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_new -C "mario@workstation $(date +%Y-%m-%d)"

# 2. Add new key to all servers (while old key still works)
for server in webserver-{01..10}.prod.example.com; do
    ssh-copy-id -i ~/.ssh/id_ed25519_new.pub -p 2222 deploy@"$server"
done

# 3. Test new key
ssh -i ~/.ssh/id_ed25519_new -p 2222 deploy@webserver-01.prod.example.com

# 4. Remove old key from servers
OLD_KEY=$(cat ~/.ssh/id_ed25519.pub)
for server in webserver-{01..10}.prod.example.com; do
    ssh -i ~/.ssh/id_ed25519_new -p 2222 deploy@"$server" \
        "sed -i '/$OLD_KEY/d' ~/.ssh/authorized_keys"
done

# 5. Replace local key
mv ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.$(date +%Y%m%d).bak
mv ~/.ssh/id_ed25519_new ~/.ssh/id_ed25519
mv ~/.ssh/id_ed25519_new.pub ~/.ssh/id_ed25519.pub
```

---

## References

- OpenSSH sshd_config man page: `man 5 sshd_config`
- OpenSSH ssh_config man page: `man 5 ssh_config`
- ssh-audit: https://github.com/jtesta/ssh-audit
- SSH Academy (ssh.com): https://www.ssh.com/academy/ssh/sshd_config
- Mozilla SSH Guidelines: https://infosec.mozilla.org/guidelines/openssh
- NIST SP 800-153: Guidelines for Securing WLANs (references key management)
- Bernstein et al., "Ed25519: High-speed high-security signatures": https://ed25519.cr.yp.to/
- PAM google-authenticator: https://github.com/google/google-authenticator-libpam
- fail2ban: https://github.com/fail2ban/fail2ban
