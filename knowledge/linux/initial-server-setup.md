# Initial Server Setup: Ubuntu 22.04 / 24.04
## Production-Ready Checklist for a Senior Sysadmin

**Scope:** A numbered, opinionated runbook taking a fresh Ubuntu 22.04 LTS or 24.04 LTS server from first root login to a hardened, production-ready baseline. Every command is tested on both LTS releases unless noted. Run everything as root unless the step explicitly says otherwise.

---

## Table of Contents

1. [First Login as Root](#1-first-login-as-root)
2. [Hostname, Locale, and Timezone](#2-hostname-locale-and-timezone)
3. [Create a Sudo User](#3-create-a-sudo-user)
4. [SSH Key Setup](#4-ssh-key-setup)
5. [Harden SSH](#5-harden-ssh)
6. [UFW Firewall](#6-ufw-firewall)
7. [fail2ban](#7-fail2ban)
8. [Unattended Security Upgrades](#8-unattended-security-upgrades)
9. [Swap Space](#9-swap-space)
10. [Kernel / sysctl Hardening](#10-kernel--sysctl-hardening)
11. [logrotate](#11-logrotate)
12. [NTP / Time Sync](#12-ntp--time-sync)
13. [journald Limits](#13-journald-limits)
14. [rkhunter Baseline](#14-rkhunter-baseline)
15. [AppArmor Status](#15-apparmor-status)
16. [Idempotent Bash Setup Script](#16-idempotent-bash-setup-script)

---

## 1. First Login as Root

When a cloud provider hands you a fresh server you typically SSH in as root with password auth or an injected key.

```bash
ssh root@<SERVER_IP>
```

**Immediately do:**

```bash
# Confirm you are root
whoami
# → root

# Check the OS version — confirms Ubuntu 22.04 or 24.04
lsb_release -a
cat /etc/os-release

# Update the package index and apply all pending updates FIRST
# Never skip this. Unpatched packages on a fresh server are a liability.
apt update && apt full-upgrade -y

# Remove orphaned packages
apt autoremove -y
apt autoclean
```

> **Opinion:** Run `full-upgrade` (not just `upgrade`) on the first boot. It handles kernel transitions and held-back packages that `upgrade` skips. Reboot immediately after if a new kernel was installed.

```bash
# Reboot if the kernel was updated
[ -f /var/run/reboot-required ] && reboot
```

After reboot, re-login as root and continue.

---

## 2. Hostname, Locale, and Timezone

Getting these right at the start prevents subtle bugs in logs, cron, and certificates.

### 2.1 Hostname

```bash
# Set a meaningful FQDN-style hostname
hostnamectl set-hostname web01.example.com

# Verify
hostnamectl status
hostname -f

# Update /etc/hosts — critical for many tools that resolve the hostname locally
# Replace the old short hostname with the new one
sed -i "s/^127\.0\.1\.1.*/127.0.1.1\tweb01.example.com web01/" /etc/hosts

# Confirm
cat /etc/hosts | grep "127.0.1.1"
```

### 2.2 Locale

Ubuntu 22.04/24.04 cloud images ship with `C.UTF-8` or a minimal locale set. Many tools, particularly Python and Ruby applications, need a proper locale.

```bash
# Install locale data
apt install -y locales

# Generate the locale you need (en_US.UTF-8 is the safe default)
locale-gen en_US.UTF-8
update-locale LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8

# Reload the environment
. /etc/default/locale

# Verify
locale
# You should see LANG=en_US.UTF-8 on every line
```

> **Opinion:** Always set `LC_ALL` and `LANG` together. Missing `LC_ALL` causes inconsistent behavior across sudo sessions and cron jobs.

### 2.3 Timezone

```bash
# List available timezones if you need to search
timedatectl list-timezones | grep America

# Set the timezone (use UTC for servers unless there is a strong business reason not to)
timedatectl set-timezone UTC

# Verify
timedatectl status
date
```

> **Opinion:** Use UTC on every server. Period. Application-layer code can convert to local time. Mixed timezones in a cluster are a support nightmare, especially when correlating logs.

---

## 3. Create a Sudo User

Never use root for day-to-day operations. Create a dedicated admin user now, before locking down SSH.

```bash
# Replace "deploy" with your chosen username
NEWUSER="deploy"

# Create the user with a home directory and bash shell
adduser --gecos "" $NEWUSER
# You will be prompted to set a password — use a strong one
# (It will be disabled for SSH login after step 4, but keep it for sudo prompts)

# Add user to the sudo group
usermod -aG sudo $NEWUSER

# Verify group membership
id $NEWUSER
# Should show: groups=...,27(sudo),...

# Optionally: also add to the adm group for log reading
usermod -aG adm $NEWUSER
```

### 3.1 Validate sudo access before locking root out

Open a **second terminal** and test before proceeding:

```bash
# In a NEW terminal window/tab:
ssh $NEWUSER@<SERVER_IP>

# After login:
sudo -v
# Should prompt for password and succeed

sudo whoami
# → root
```

> **Warning:** Do not close your root session until you have confirmed the new user can `sudo`. Locking yourself out of a cloud VM is a bad day.

---

## 4. SSH Key Setup

Password-based SSH auth will be disabled in step 5. Get your key on the server now.

### 4.1 From your local machine (preferred method)

```bash
# If you don't have an SSH key pair yet, generate one on your local machine:
ssh-keygen -t ed25519 -C "your_email@example.com" -f ~/.ssh/id_ed25519
# Or RSA 4096 for legacy compatibility:
# ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# Copy your public key to the server for the new user
ssh-copy-id -i ~/.ssh/id_ed25519.pub $NEWUSER@<SERVER_IP>
```

### 4.2 Manual method (when ssh-copy-id is unavailable)

On the **server as root**:

```bash
NEWUSER="deploy"

# Create the SSH directory with correct permissions
mkdir -p /home/$NEWUSER/.ssh
chmod 700 /home/$NEWUSER/.ssh

# Paste your public key here
echo "ssh-ed25519 AAAA...your_public_key_here... comment" \
  >> /home/$NEWUSER/.ssh/authorized_keys

# Set correct permissions — SSH will silently refuse to use keys with wrong perms
chmod 600 /home/$NEWUSER/.ssh/authorized_keys
chown -R $NEWUSER:$NEWUSER /home/$NEWUSER/.ssh
```

### 4.3 Verify key-based login BEFORE proceeding

```bash
# From your local machine — must succeed WITHOUT a password prompt
ssh -i ~/.ssh/id_ed25519 $NEWUSER@<SERVER_IP>
```

> **Hard stop:** If key auth fails, debug it now. Do not proceed to step 5 until you can log in without a password.

**Debug checklist:**
- `~/.ssh/` on server: mode `700`
- `authorized_keys` on server: mode `600`
- Owner of both is the user, not root
- Public key content is on a **single line** (no line breaks)
- Run `ssh -vvv` locally to see exactly where auth fails

---

## 5. Harden SSH

This is the highest-impact hardening step. Do it wrong and you lock yourself out.

```bash
# Back up the original config
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

# Edit the SSH daemon config
nano /etc/ssh/sshd_config
```

**Required settings — change or confirm each one:**

```ini
# Disable root login entirely
PermitRootLogin no

# Require public key authentication
PubkeyAuthentication yes

# Disable password authentication
PasswordAuthentication no

# Disable challenge-response (covers PAM password fallback)
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no

# Disable empty passwords
PermitEmptyPasswords no

# Disable X11 forwarding (enable only if you need it)
X11Forwarding no

# Disable agent forwarding unless needed
AllowAgentForwarding no

# Set a login grace time (seconds before unauthenticated connections are dropped)
LoginGraceTime 30

# Limit authentication attempts per connection
MaxAuthTries 3

# Limit concurrent unauthenticated connections
MaxStartups 10:30:60

# Restrict which users can log in (optional but recommended)
AllowUsers deploy

# Log level — INFO is fine for most cases, VERBOSE for debugging
LogLevel INFO

# Protocol — SSH2 only (SSH1 is ancient, broken, should never appear but be explicit)
Protocol 2
```

### 5.1 Optional: Change the SSH port

Changing the port provides no real security (obscurity is not security) but does reduce log noise from automated scanners. Only do this if your team is comfortable managing non-standard SSH.

```bash
# In /etc/ssh/sshd_config:
Port 2222
# or any port above 1024 that is not in use

# If you change the port, update UFW BEFORE restarting sshd:
ufw allow 2222/tcp
ufw delete allow 22/tcp
```

### 5.2 Validate and apply

```bash
# Validate config syntax — always do this before restarting
sshd -t
# No output = config is valid

# Restart the SSH daemon
systemctl restart ssh
# On some Ubuntu versions the service is named "sshd":
# systemctl restart sshd

# Verify it is running
systemctl status ssh
```

### 5.3 Test from a new terminal (keep current session open)

```bash
ssh -i ~/.ssh/id_ed25519 deploy@<SERVER_IP>
# Must connect without a password prompt

# Confirm root login is blocked
ssh root@<SERVER_IP>
# Must be denied
```

> **Opinion:** Set `AllowUsers` to an explicit list. It creates a documented allowlist and means adding a new system user (e.g., for an application) does not automatically grant SSH access.

---

## 6. UFW Firewall

Ubuntu's Uncomplicated Firewall wraps `iptables` (22.04) or `nftables` (24.04) with sane defaults.

```bash
# Install UFW if it isn't present (it is on Ubuntu cloud images)
apt install -y ufw

# IMPORTANT: Configure rules BEFORE enabling, or you will lock yourself out

# Deny all incoming by default
ufw default deny incoming

# Allow all outgoing by default
ufw default allow outgoing

# Allow SSH — if you changed the port in step 5, use that port number instead
ufw allow 22/tcp comment 'SSH'

# Allow HTTP and HTTPS
ufw allow 80/tcp comment 'HTTP'
ufw allow 443/tcp comment 'HTTPS'

# Enable UFW — answer "y" when prompted
ufw enable

# Verify status
ufw status verbose
```

**Example output you want to see:**

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
```

### 6.1 Common additional rules

```bash
# Allow from a specific IP only (e.g., database access from app server)
ufw allow from 10.0.0.5 to any port 5432 comment 'PostgreSQL from app01'

# Allow a range of IPs
ufw allow from 10.0.0.0/24 to any port 3306 comment 'MySQL from internal network'

# Delete a rule by rule number
ufw status numbered
ufw delete <NUMBER>

# Reload after changes
ufw reload
```

> **Opinion:** Start with the minimal set of open ports and expand on demand. Every open port is an attack surface. Do not `allow 3306` globally just because "we might need it later."

---

## 7. fail2ban

fail2ban monitors log files and bans IPs that show brute-force patterns.

```bash
# Install
apt install -y fail2ban

# Copy the default config to a local override file
# Never edit jail.conf directly — it gets overwritten on upgrades
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Edit `/etc/fail2ban/jail.local`:

```bash
nano /etc/fail2ban/jail.local
```

**Key settings to configure in `[DEFAULT]` section:**

```ini
[DEFAULT]
# Ban duration in seconds (default 600 = 10 min; set to -1 for permanent)
bantime  = 3600

# Time window in which failures are counted
findtime  = 600

# Number of failures before banning
maxretry = 5

# Your own IP — never ban yourself
ignoreip = 127.0.0.1/8 ::1
# Add your home/office IP:
# ignoreip = 127.0.0.1/8 ::1 203.0.113.10
```

**Enable the SSH jail — add or confirm this section:**

```ini
[sshd]
enabled  = true
port     = ssh
# If you changed the SSH port, specify it:
# port    = 2222
logpath  = /var/log/auth.log
maxretry = 3
```

```bash
# Enable and start fail2ban
systemctl enable fail2ban
systemctl start fail2ban

# Check status
systemctl status fail2ban
fail2ban-client status
fail2ban-client status sshd
```

### 7.1 Useful fail2ban commands

```bash
# View currently banned IPs
fail2ban-client status sshd

# Unban an IP manually
fail2ban-client set sshd unbanip <IP_ADDRESS>

# Watch the fail2ban log in real time
tail -f /var/log/fail2ban.log
```

> **Opinion:** `maxretry = 3` for SSH is aggressive but correct. Legitimate users do not fat-finger their SSH key. If you use Ansible or other automation that connects frequently, add the automation host to `ignoreip`.

---

## 8. Unattended Security Upgrades

Security patches should apply automatically. The window between a CVE publication and exploitation is measured in hours, not days.

```bash
# Install
apt install -y unattended-upgrades apt-listchanges

# Configure — this interactive dialog asks what to enable
# Answer "Yes" to automatically download and install stable updates
dpkg-reconfigure --priority=low unattended-upgrades
```

Edit `/etc/apt/apt.conf.d/50unattended-upgrades`:

```bash
nano /etc/apt/apt.conf.d/50unattended-upgrades
```

**Recommended configuration:**

```
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}";
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
    "${distro_id}ESM:${distro_codename}-infra-security";
};

// Automatically remove unused kernel packages
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";

// Remove dependencies that are no longer needed
Unattended-Upgrade::Remove-Unused-Dependencies "true";

// Automatically reboot if required (set to false if you prefer controlled reboots)
Unattended-Upgrade::Automatic-Reboot "false";

// Reboot at this time if Automatic-Reboot is true
Unattended-Upgrade::Automatic-Reboot-Time "03:00";

// Email alerts — set to your email address
Unattended-Upgrade::Mail "ops@example.com";
Unattended-Upgrade::MailReport "on-change";
```

Edit `/etc/apt/apt.conf.d/20auto-upgrades` (created by dpkg-reconfigure or create manually):

```bash
cat > /etc/apt/apt.conf.d/20auto-upgrades << 'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
EOF
```

```bash
# Test the configuration
unattended-upgrades --dry-run --debug 2>&1 | head -50

# Verify the service is running
systemctl status unattended-upgrades
```

> **Opinion:** Set `Automatic-Reboot "false"` on production servers. Instead, check `cat /var/run/reboot-required` in your monitoring and schedule reboots during maintenance windows. Surprise reboots at 3 AM because a kernel was patched cause incidents.

---

## 9. Swap Space

Cloud VMs often ship with no swap. For most workloads, 2 GB of swap is a reasonable safety net. It prevents OOM kills at the cost of performance degradation (which at least tells you the server is under-provisioned rather than silently crashing).

```bash
# Check if swap already exists
swapon --show
free -h

# If swap exists and is adequately sized, skip to swappiness tuning below
```

**Create a 2 GB swap file:**

```bash
# Allocate the file — fallocate is fast but mkswap also works on regular files
fallocate -l 2G /swapfile

# Verify
ls -lh /swapfile

# Set permissions — swap files must not be world-readable
chmod 600 /swapfile

# Initialize as swap
mkswap /swapfile

# Activate swap
swapon /swapfile

# Verify
swapon --show
free -h
```

**Make swap permanent across reboots:**

```bash
# Add to /etc/fstab (check it isn't already there first)
grep -q '/swapfile' /etc/fstab || echo '/swapfile none swap sw 0 0' >> /etc/fstab

# Verify fstab entry
grep swap /etc/fstab
```

**Tune swappiness:**

The default swappiness of 60 is too aggressive for servers — it starts swapping when RAM is 40% free. For a server (not a desktop), set it to 10.

```bash
# Check current value
cat /proc/sys/vm/swappiness

# Set immediately
sysctl vm.swappiness=10

# Make permanent
echo "vm.swappiness=10" >> /etc/sysctl.d/99-swappiness.conf

# For database servers (PostgreSQL, MySQL), also tune cache pressure
echo "vm.vfs_cache_pressure=50" >> /etc/sysctl.d/99-swappiness.conf
sysctl -p /etc/sysctl.d/99-swappiness.conf
```

> **Opinion:** Swap is a last resort, not a resource planning substitute. If your server is hitting swap regularly, add RAM. But the 2 GB file costs almost nothing and has saved many services from OOM kills during unexpected load spikes.

---

## 10. Kernel / sysctl Hardening

Kernel parameters that reduce the attack surface and improve network resilience.

```bash
cat > /etc/sysctl.d/99-security.conf << 'EOF'
###############################################################################
# Network hardening
###############################################################################

# Enable SYN cookies to protect against SYN flood attacks
net.ipv4.tcp_syncookies = 1

# Disable IP source routing (used in some spoofing attacks)
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0

# Disable ICMP redirect acceptance (prevents MITM via routing manipulation)
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# Do not send ICMP redirects (this is not a router)
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Ignore bogus ICMP error responses
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Log packets with impossible source addresses (martians)
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# Disable IP forwarding — this is not a router
# Set to 1 ONLY if this server acts as a NAT gateway or VPN server
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# Protect against IP spoofing via reverse path filtering
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Increase the range of ephemeral ports available
net.ipv4.ip_local_port_range = 1024 65535

# Reduce TIME_WAIT state duration for faster TCP reuse
net.ipv4.tcp_fin_timeout = 15

# Enable TCP fast open for connections (reduces latency)
net.ipv4.tcp_fastopen = 3

# Tune TCP keepalive to detect dead connections faster
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15

###############################################################################
# Kernel hardening
###############################################################################

# Restrict dmesg access to root only (hides kernel addresses from unprivileged users)
kernel.dmesg_restrict = 1

# Restrict kernel pointer exposure in /proc
kernel.kptr_restrict = 2

# Disable magic SysRq key (useful in emergencies, but a security risk on shared servers)
kernel.sysrq = 0

# Restrict ptrace to direct parent processes only
# 1 = only parent can ptrace child; 2 = only root; 3 = disabled entirely
kernel.yama.ptrace_scope = 1

# Disable core dumps for setuid programs
fs.suid_dumpable = 0

# Protect against symlink and hardlink attacks (Ubuntu 20.04+ has these on by default)
fs.protected_symlinks = 1
fs.protected_hardlinks = 1

# Increase system file descriptor limit
fs.file-max = 2097152

###############################################################################
# Memory hardening
###############################################################################

# Randomize memory layout to make exploits harder
kernel.randomize_va_space = 2
EOF

# Apply all settings immediately
sysctl -p /etc/sysctl.d/99-security.conf

# Verify a key setting
sysctl net.ipv4.tcp_syncookies
# → net.ipv4.tcp_syncookies = 1
```

> **Opinion:** `kernel.yama.ptrace_scope = 1` will occasionally frustrate developers who use debuggers or profilers like `perf` or `strace` on processes they don't own. That is intentional. Legitimate debugging workflows can be accommodated with `sudo`.

---

## 11. logrotate

Ubuntu ships with logrotate preconfigured. Verify it is working and tune if needed.

```bash
# Check that logrotate is installed
which logrotate
logrotate --version

# View the main config
cat /etc/logrotate.conf

# List app-specific configs
ls -la /etc/logrotate.d/

# Test logrotate in dry-run mode — shows what would happen
logrotate -d /etc/logrotate.conf 2>&1 | head -40

# Verify the daily cron job or systemd timer
# Ubuntu 22.04+: logrotate runs via a systemd timer
systemctl status logrotate.timer
systemctl status logrotate.service

# On older systems it may be a cron job:
cat /etc/cron.daily/logrotate
```

**Check and tune the global logrotate config:**

```bash
nano /etc/logrotate.conf
```

Confirm these settings exist (add or adjust as needed):

```
# Rotate weekly by default
weekly

# Keep 4 weeks of backlogs
rotate 4

# Create new log files after rotating
create

# Compress rotated logs
compress

# Delay compression by one rotation (lets processes finish writing)
delaycompress

# Do not rotate empty log files
notifempty

# Do not error if the log file is missing
missingok
```

**For high-traffic applications, create a custom logrotate config:**

```bash
cat > /etc/logrotate.d/myapp << 'EOF'
/var/log/myapp/*.log {
    daily
    rotate 14
    compress
    delaycompress
    notifempty
    missingok
    sharedscripts
    postrotate
        systemctl reload myapp 2>/dev/null || true
    endscript
}
EOF
```

---

## 12. NTP / Time Sync

Accurate time is non-negotiable. It affects TLS certificate validation, log correlation, Kerberos, and distributed systems.

### 12.1 systemd-timesyncd (default, sufficient for most servers)

```bash
# Check status
timedatectl status
systemctl status systemd-timesyncd

# Show sync details
timedatectl show-timesync --all 2>/dev/null || \
  journalctl -u systemd-timesyncd --no-pager -n 20
```

Configure the NTP servers (edit `/etc/systemd/timesyncd.conf`):

```bash
nano /etc/systemd/timesyncd.conf
```

```ini
[Time]
NTP=time1.google.com time2.google.com time3.google.com time4.google.com
FallbackNTP=0.ubuntu.pool.ntp.org 1.ubuntu.pool.ntp.org
RootDistanceMaxSec=5
PollIntervalMinSec=32
PollIntervalMaxSec=2048
```

```bash
# Restart to apply
systemctl restart systemd-timesyncd

# Force sync now
timedatectl set-ntp true

# Confirm synchronized
timedatectl | grep "synchronized"
# → System clock synchronized: yes
```

### 12.2 chrony (recommended alternative for high-precision or NTP server roles)

```bash
# Install chrony (it replaces systemd-timesyncd)
apt install -y chrony

# Disable and stop systemd-timesyncd
systemctl disable systemd-timesyncd
systemctl stop systemd-timesyncd

# Configure chrony
nano /etc/chrony/chrony.conf
```

Add preferred NTP servers at the top:

```
server time1.google.com iburst
server time2.google.com iburst
server time3.google.com iburst
server time4.google.com iburst
```

```bash
# Enable and start chrony
systemctl enable chrony
systemctl start chrony

# Check sync status
chronyc tracking
chronyc sources -v
```

> **Opinion:** For most VMs, `systemd-timesyncd` with Google's NTP servers is fine. Use `chrony` if you need sub-millisecond accuracy, if you are building an internal NTP server, or if you are running distributed databases (Cassandra, CockroachDB) that are sensitive to clock skew.

---

## 13. journald Limits

By default, `journald` will consume up to 10% of your disk. On a server with 20 GB of disk, that is 2 GB of logs. Cap it explicitly.

```bash
# Check current disk usage
journalctl --disk-usage

# Edit the journald config
nano /etc/systemd/journald.conf
```

**Recommended settings:**

```ini
[Journal]
# Maximum disk space for persistent journal storage
SystemMaxUse=500M

# Maximum disk space for a single journal file
SystemMaxFileSize=50M

# How long to keep journal entries
MaxRetentionSec=2week

# Keep the runtime journal (in /run) small
RuntimeMaxUse=100M

# Compress journal entries
Compress=yes

# Forward to syslog (keep off unless you are running a syslog daemon)
ForwardToSyslog=no
```

```bash
# Apply by restarting journald
systemctl restart systemd-journald

# Vacuum immediately to enforce the new limits
journalctl --vacuum-size=500M
journalctl --vacuum-time=2weeks

# Verify new disk usage
journalctl --disk-usage
```

---

## 14. rkhunter Baseline

Rootkit Hunter scans for known rootkits, backdoors, and suspicious local changes.

```bash
# Install
apt install -y rkhunter

# Update the database of known good file hashes and rootkit signatures
rkhunter --update

# Build the baseline of known-good file properties
# Run this AFTER the system is fully configured (all packages installed)
# This "photographs" the current state — run it now and after any major change
rkhunter --propupd

# Run a full check
rkhunter --check --skip-keypress

# Review the log
cat /var/log/rkhunter.log | grep -E "Warning|Error|Infected"
```

**Configure rkhunter:**

```bash
nano /etc/rkhunter.conf
```

Key settings to review:

```ini
# Email alerts
MAIL-ON-WARNING="ops@example.com"

# Update mirrors
UPDATE_MIRRORS=1
MIRRORS_MODE=0

# Allow hidden files in these directories (common false positives)
ALLOWHIDDENDIR=/dev/.udev
ALLOWHIDDENFILE=/dev/.blkid.tab
ALLOWHIDDENFILE=/dev/.blkid.tab.old

# SSH allowed config (match your sshd_config)
ALLOW_SSH_PROT_V1=0
```

**Schedule daily rkhunter checks:**

```bash
cat > /etc/cron.daily/rkhunter-check << 'EOF'
#!/bin/bash
/usr/bin/rkhunter --cronjob --update --quiet
EOF
chmod +x /etc/cron.daily/rkhunter-check
```

> **Opinion:** rkhunter produces false positives. Review the log after the first run and use `ALLOWHIDDENDIR`, `ALLOWHIDDENFILE`, and `SCRIPTWHITELIST` directives to suppress known-good items. An alert-fatigued ops team ignores all alerts.

---

## 15. AppArmor Status

AppArmor provides mandatory access control (MAC) — it confines programs to a limited set of resources. Ubuntu ships with AppArmor enabled and profiles for common services.

```bash
# Check if AppArmor is enabled
aa-status

# Or
systemctl status apparmor

# List all loaded profiles
apparmor_status 2>/dev/null || aa-status

# Check for profiles in complain mode vs enforce mode
aa-status | grep -E "enforce|complain"
```

**Install additional tools:**

```bash
apt install -y apparmor-utils apparmor-profiles apparmor-profiles-extra
```

**Manage profiles:**

```bash
# Put a profile in enforce mode
aa-enforce /etc/apparmor.d/usr.bin.firefox

# Put a profile in complain mode (logs violations but doesn't block)
aa-complain /etc/apparmor.d/usr.bin.firefox

# Disable a specific profile
aa-disable /etc/apparmor.d/usr.bin.firefox

# Reload all profiles
systemctl reload apparmor
```

**For services you install (nginx, MySQL, etc.):**

```bash
# nginx profile example — enforce it
apt install -y nginx
aa-enforce /etc/apparmor.d/usr.sbin.nginx
```

> **Opinion:** Do not disable AppArmor. Cloud vendors and Ubuntu themselves ship it enabled because it provides meaningful defense-in-depth. The performance overhead is negligible. If a specific profile causes problems, put it in `complain` mode and investigate rather than disabling it wholesale.

---

## 16. Idempotent Bash Setup Script

This script encodes all of the above steps. It is safe to run multiple times — each step checks whether the action is already done before doing it.

**Usage:**
```bash
# Download or copy to the server, then:
chmod +x server-setup.sh
sudo ./server-setup.sh
```

```bash
#!/usr/bin/env bash
# =============================================================================
# server-setup.sh
# Idempotent Ubuntu 22.04 / 24.04 initial server hardening script
# Run as root. Safe to run multiple times.
# =============================================================================
set -euo pipefail

# ── Configuration ─────────────────────────────────────────────────────────────
NEWUSER="${NEWUSER:-deploy}"
HOSTNAME="${SERVER_HOSTNAME:-web01.example.com}"
TIMEZONE="${SERVER_TZ:-UTC}"
LOCALE="${SERVER_LOCALE:-en_US.UTF-8}"
SSH_PORT="${SSH_PORT:-22}"
SWAP_SIZE="${SWAP_SIZE:-2G}"
ALERT_EMAIL="${ALERT_EMAIL:-ops@example.com}"
YOUR_PUBLIC_KEY="${YOUR_PUBLIC_KEY:-}"   # Set this or pre-populate authorized_keys

# ── Helpers ───────────────────────────────────────────────────────────────────
log()  { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
ok()   { echo "[OK]  $*"; }
skip() { echo "[SKIP] $*"; }

require_root() {
  if [[ $EUID -ne 0 ]]; then
    echo "ERROR: Run this script as root." >&2
    exit 1
  fi
}

# ── 1. System update ──────────────────────────────────────────────────────────
system_update() {
  log "Updating package index and upgrading packages..."
  apt-get update -qq
  DEBIAN_FRONTEND=noninteractive apt-get full-upgrade -y -qq
  apt-get autoremove -y -qq
  apt-get autoclean -qq
  ok "System updated"
}

# ── 2. Hostname, locale, timezone ─────────────────────────────────────────────
configure_system() {
  log "Configuring hostname: $HOSTNAME"
  current_hostname=$(hostnamectl --static 2>/dev/null || hostname)
  if [[ "$current_hostname" != "$HOSTNAME" ]]; then
    hostnamectl set-hostname "$HOSTNAME"
    short="${HOSTNAME%%.*}"
    if ! grep -q "127.0.1.1" /etc/hosts; then
      echo "127.0.1.1	$HOSTNAME $short" >> /etc/hosts
    else
      sed -i "s/^127\.0\.1\.1.*/127.0.1.1\t$HOSTNAME $short/" /etc/hosts
    fi
    ok "Hostname set to $HOSTNAME"
  else
    skip "Hostname already $HOSTNAME"
  fi

  log "Configuring locale: $LOCALE"
  apt-get install -y -qq locales
  if ! locale -a 2>/dev/null | grep -qi "$(echo $LOCALE | tr '[:upper:]' '[:lower:]' | tr '.' '_')"; then
    locale-gen "$LOCALE"
  fi
  update-locale LANG="$LOCALE" LC_ALL="$LOCALE"
  ok "Locale configured"

  log "Configuring timezone: $TIMEZONE"
  current_tz=$(timedatectl show --property=Timezone --value 2>/dev/null || cat /etc/timezone)
  if [[ "$current_tz" != "$TIMEZONE" ]]; then
    timedatectl set-timezone "$TIMEZONE"
    ok "Timezone set to $TIMEZONE"
  else
    skip "Timezone already $TIMEZONE"
  fi
}

# ── 3. Sudo user ──────────────────────────────────────────────────────────────
create_user() {
  log "Creating user: $NEWUSER"
  if id "$NEWUSER" &>/dev/null; then
    skip "User $NEWUSER already exists"
  else
    adduser --disabled-password --gecos "" "$NEWUSER"
    ok "User $NEWUSER created"
  fi

  if ! groups "$NEWUSER" | grep -qw sudo; then
    usermod -aG sudo,adm "$NEWUSER"
    ok "User $NEWUSER added to sudo and adm groups"
  else
    skip "User $NEWUSER already in sudo group"
  fi

  # Deploy SSH key if provided
  if [[ -n "$YOUR_PUBLIC_KEY" ]]; then
    SSH_DIR="/home/$NEWUSER/.ssh"
    AUTH_KEYS="$SSH_DIR/authorized_keys"
    mkdir -p "$SSH_DIR"
    chmod 700 "$SSH_DIR"
    if ! grep -qF "$YOUR_PUBLIC_KEY" "$AUTH_KEYS" 2>/dev/null; then
      echo "$YOUR_PUBLIC_KEY" >> "$AUTH_KEYS"
      ok "SSH public key added for $NEWUSER"
    else
      skip "SSH key already present for $NEWUSER"
    fi
    chmod 600 "$AUTH_KEYS"
    chown -R "$NEWUSER:$NEWUSER" "$SSH_DIR"
  fi
}

# ── 5. SSH hardening ──────────────────────────────────────────────────────────
harden_ssh() {
  log "Hardening SSH..."
  SSHD="/etc/ssh/sshd_config"
  [[ -f "${SSHD}.bak" ]] || cp "$SSHD" "${SSHD}.bak"

  declare -A settings=(
    ["PermitRootLogin"]="no"
    ["PubkeyAuthentication"]="yes"
    ["PasswordAuthentication"]="no"
    ["ChallengeResponseAuthentication"]="no"
    ["KbdInteractiveAuthentication"]="no"
    ["PermitEmptyPasswords"]="no"
    ["X11Forwarding"]="no"
    ["LoginGraceTime"]="30"
    ["MaxAuthTries"]="3"
    ["MaxStartups"]="10:30:60"
    ["Protocol"]="2"
  )

  for key in "${!settings[@]}"; do
    val="${settings[$key]}"
    if grep -qE "^#?${key}\s" "$SSHD"; then
      sed -i "s|^#\?${key}\s.*|${key} ${val}|" "$SSHD"
    else
      echo "${key} ${val}" >> "$SSHD"
    fi
  done

  if [[ "$SSH_PORT" != "22" ]]; then
    sed -i "s|^#\?Port\s.*|Port ${SSH_PORT}|" "$SSHD"
    grep -q "^Port " "$SSHD" || echo "Port $SSH_PORT" >> "$SSHD"
  fi

  sshd -t && ok "SSH config valid"
  systemctl restart ssh 2>/dev/null || systemctl restart sshd
  ok "SSH hardened and restarted"
}

# ── 6. UFW firewall ───────────────────────────────────────────────────────────
configure_ufw() {
  log "Configuring UFW..."
  apt-get install -y -qq ufw

  ufw --force reset
  ufw default deny incoming
  ufw default allow outgoing
  ufw allow "${SSH_PORT}/tcp" comment 'SSH'
  ufw allow 80/tcp comment 'HTTP'
  ufw allow 443/tcp comment 'HTTPS'
  ufw --force enable
  ok "UFW configured and enabled"
  ufw status verbose
}

# ── 7. fail2ban ───────────────────────────────────────────────────────────────
configure_fail2ban() {
  log "Configuring fail2ban..."
  apt-get install -y -qq fail2ban

  if [[ ! -f /etc/fail2ban/jail.local ]]; then
    cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
  fi

  cat > /etc/fail2ban/jail.d/sshd-local.conf << EOF
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 5
ignoreip = 127.0.0.1/8 ::1

[sshd]
enabled  = true
port     = ${SSH_PORT}
logpath  = /var/log/auth.log
maxretry = 3
EOF

  systemctl enable fail2ban
  systemctl restart fail2ban
  ok "fail2ban configured and started"
}

# ── 8. Unattended upgrades ────────────────────────────────────────────────────
configure_unattended_upgrades() {
  log "Configuring unattended-upgrades..."
  apt-get install -y -qq unattended-upgrades apt-listchanges

  cat > /etc/apt/apt.conf.d/20auto-upgrades << 'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
EOF

  # Patch 50unattended-upgrades safely
  sed -i 's|//Unattended-Upgrade::Remove-Unused-Kernel-Packages|Unattended-Upgrade::Remove-Unused-Kernel-Packages|' \
    /etc/apt/apt.conf.d/50unattended-upgrades 2>/dev/null || true
  sed -i 's|//Unattended-Upgrade::Remove-Unused-Dependencies|Unattended-Upgrade::Remove-Unused-Dependencies|' \
    /etc/apt/apt.conf.d/50unattended-upgrades 2>/dev/null || true

  systemctl enable unattended-upgrades
  systemctl restart unattended-upgrades
  ok "Unattended-upgrades configured"
}

# ── 9. Swap ───────────────────────────────────────────────────────────────────
configure_swap() {
  log "Configuring swap..."
  if swapon --show | grep -q '/swapfile'; then
    skip "Swap already active on /swapfile"
  else
    fallocate -l "$SWAP_SIZE" /swapfile
    chmod 600 /swapfile
    mkswap /swapfile
    swapon /swapfile
    ok "Swap created: $SWAP_SIZE"
  fi

  if ! grep -q '/swapfile' /etc/fstab; then
    echo '/swapfile none swap sw 0 0' >> /etc/fstab
    ok "Swap added to /etc/fstab"
  else
    skip "Swap already in /etc/fstab"
  fi

  sysctl -w vm.swappiness=10 > /dev/null
  if ! grep -q 'vm.swappiness' /etc/sysctl.d/99-swappiness.conf 2>/dev/null; then
    echo "vm.swappiness=10" > /etc/sysctl.d/99-swappiness.conf
    echo "vm.vfs_cache_pressure=50" >> /etc/sysctl.d/99-swappiness.conf
  fi
  ok "Swappiness set to 10"
}

# ── 10. sysctl hardening ──────────────────────────────────────────────────────
apply_sysctl() {
  log "Applying kernel/sysctl hardening..."
  cat > /etc/sysctl.d/99-security.conf << 'EOF'
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
kernel.sysrq = 0
kernel.yama.ptrace_scope = 1
fs.suid_dumpable = 0
fs.protected_symlinks = 1
fs.protected_hardlinks = 1
fs.file-max = 2097152
kernel.randomize_va_space = 2
EOF

  sysctl -p /etc/sysctl.d/99-security.conf > /dev/null
  ok "sysctl hardening applied"
}

# ── 12. NTP ───────────────────────────────────────────────────────────────────
configure_ntp() {
  log "Configuring NTP..."
  NTP_CONF="/etc/systemd/timesyncd.conf"
  if ! grep -q "time1.google.com" "$NTP_CONF" 2>/dev/null; then
    cat > "$NTP_CONF" << 'EOF'
[Time]
NTP=time1.google.com time2.google.com time3.google.com time4.google.com
FallbackNTP=0.ubuntu.pool.ntp.org 1.ubuntu.pool.ntp.org
RootDistanceMaxSec=5
PollIntervalMinSec=32
PollIntervalMaxSec=2048
EOF
    systemctl restart systemd-timesyncd
    ok "NTP configured"
  else
    skip "NTP already configured"
  fi
  timedatectl set-ntp true
}

# ── 13. journald limits ───────────────────────────────────────────────────────
configure_journald() {
  log "Configuring journald limits..."
  JCONF="/etc/systemd/journald.conf"
  if ! grep -q "SystemMaxUse=500M" "$JCONF" 2>/dev/null; then
    cat > "$JCONF" << 'EOF'
[Journal]
SystemMaxUse=500M
SystemMaxFileSize=50M
MaxRetentionSec=2week
RuntimeMaxUse=100M
Compress=yes
ForwardToSyslog=no
EOF
    systemctl restart systemd-journald
    journalctl --vacuum-size=500M --vacuum-time=2weeks > /dev/null 2>&1 || true
    ok "journald limits configured"
  else
    skip "journald already configured"
  fi
}

# ── 14. rkhunter ──────────────────────────────────────────────────────────────
configure_rkhunter() {
  log "Configuring rkhunter..."
  apt-get install -y -qq rkhunter
  rkhunter --update --quiet 2>/dev/null || true
  rkhunter --propupd --quiet
  ok "rkhunter baseline established"

  CRON_SCRIPT="/etc/cron.daily/rkhunter-check"
  if [[ ! -f "$CRON_SCRIPT" ]]; then
    cat > "$CRON_SCRIPT" << 'EOF'
#!/bin/bash
/usr/bin/rkhunter --cronjob --update --quiet
EOF
    chmod +x "$CRON_SCRIPT"
    ok "rkhunter daily cron installed"
  else
    skip "rkhunter cron already exists"
  fi
}

# ── 15. AppArmor ──────────────────────────────────────────────────────────────
check_apparmor() {
  log "Checking AppArmor..."
  apt-get install -y -qq apparmor-utils apparmor-profiles 2>/dev/null || true

  if systemctl is-active --quiet apparmor 2>/dev/null; then
    ok "AppArmor is active"
    aa-status --verbose 2>/dev/null | head -10 || true
  elif aa-status &>/dev/null; then
    ok "AppArmor is running (via kernel)"
  else
    echo "WARNING: AppArmor does not appear to be running. Investigate."
  fi
}

# ── Main ──────────────────────────────────────────────────────────────────────
main() {
  require_root
  log "=== Ubuntu Server Hardening Script ==="
  log "    Target user : $NEWUSER"
  log "    Hostname    : $HOSTNAME"
  log "    Timezone    : $TIMEZONE"
  log "    SSH port    : $SSH_PORT"
  log "    Swap size   : $SWAP_SIZE"
  echo ""

  system_update
  configure_system
  create_user
  harden_ssh
  configure_ufw
  configure_fail2ban
  configure_unattended_upgrades
  configure_swap
  apply_sysctl
  configure_ntp
  configure_journald
  configure_rkhunter
  check_apparmor

  echo ""
  log "=== Setup complete. ==="
  log "Reboot recommended if kernel was updated:"
  log "  cat /var/run/reboot-required && reboot"
  echo ""
  log "Verify you can SSH as $NEWUSER before closing this session:"
  log "  ssh -i ~/.ssh/id_ed25519 ${NEWUSER}@<SERVER_IP>"
}

main "$@"
```

---

## Quick Reference: Post-Setup Verification Checklist

Run these commands to confirm everything is working after setup or after a reboot.

```bash
# System
hostnamectl status
timedatectl status
locale
free -h && swapon --show

# Services
systemctl is-active ssh fail2ban unattended-upgrades systemd-timesyncd apparmor
systemctl is-enabled ssh fail2ban unattended-upgrades

# Firewall
ufw status verbose

# fail2ban
fail2ban-client status sshd

# sysctl spot-check
sysctl net.ipv4.tcp_syncookies kernel.dmesg_restrict kernel.yama.ptrace_scope

# SSH config sanity
sshd -T | grep -E "permitrootlogin|passwordauthentication|pubkeyauthentication|port"

# Journal disk usage
journalctl --disk-usage

# AppArmor
aa-status | head -10

# rkhunter last run
ls -la /var/log/rkhunter.log
```

---

## Common Pitfalls and How to Avoid Them

| Pitfall | Consequence | Fix |
|---|---|---|
| Disabling password auth before testing key login | Locked out of server | Always test key login in a new terminal before restarting sshd |
| Enabling UFW before allowing SSH | Locked out of server | Add SSH rule before `ufw enable` |
| Setting `Automatic-Reboot "true"` in unattended-upgrades on production | Unexpected service outage | Keep it `false`, monitor `/var/run/reboot-required` |
| Running `rkhunter --propupd` AFTER installing new packages | Next check reports false positives as real threats | Run `--propupd` whenever you make major package changes |
| Using `vm.swappiness=60` (default) on a database server | Excessive swapping degrades DB performance | Set to 10 for application servers, 1 for dedicated DB servers |
| Ignoring AppArmor profile complaints | Missing opportunities to detect misbehaving software | Monitor `/var/log/audit/audit.log` or `dmesg | grep apparmor` |
| Not setting `MaxStartups` in sshd | Connection exhaustion from scanner bots | Set `MaxStartups 10:30:60` |

---

## References

- DigitalOcean: Initial Server Setup with Ubuntu 22.04 — https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-22-04
- Ubuntu Server Docs: Security — Users — https://ubuntu.com/server/docs/security-users
- Ubuntu Security Guide (usg) — https://ubuntu.com/security/certifications/docs/usg
- CIS Ubuntu Linux 22.04 LTS Benchmark — https://www.cisecurity.org/benchmark/ubuntu_linux
- Lynis — server auditing tool: `apt install lynis && lynis audit system`
- sshd_config(5) man page: `man 5 sshd_config`
- UFW manual: `man ufw`
- fail2ban wiki: https://github.com/fail2ban/fail2ban/wiki
