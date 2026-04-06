# systemd Complete Guide — Production Reference

> Written from a senior sysadmin perspective. Opinionated where choices matter.
> Covers systemd 252+ (RHEL 9 / Ubuntu 22.04 / Debian 12 baseline).

---

## Table of Contents

1. [Mental Model](#mental-model)
2. [Unit File Anatomy](#unit-file-anatomy)
3. [[Unit] Section — Dependencies and Ordering](#unit-section)
4. [[Service] Section — Type and Lifecycle](#service-section)
5. [Exec Directives](#exec-directives)
6. [Restart Policies](#restart-policies)
7. [Environment and Identity](#environment-and-identity)
8. [Sandboxing — Full Reference](#sandboxing)
9. [Timer Units](#timer-units)
10. [Socket Activation](#socket-activation)
11. [Drop-in Overrides](#drop-in-overrides)
12. [journalctl Reference](#journalctl-reference)
13. [Production Service Template](#production-service-template)
14. [Operational Cheatsheet](#operational-cheatsheet)

---

## Mental Model

systemd manages the **lifecycle of units**. A unit is anything systemd tracks: services, timers, sockets, mounts, targets, devices, slices. Every unit is described by a plain-text file.

Units live in three places — listed in override priority order (highest last wins):

| Path | Purpose |
|---|---|
| `/usr/lib/systemd/system/` | Vendor/package-installed units — never edit these |
| `/run/systemd/system/` | Runtime-generated units — ephemeral, lost on reboot |
| `/etc/systemd/system/` | Admin-owned units and overrides — this is your domain |

**Always** put your custom units in `/etc/systemd/system/`. When a package ships a unit you want to modify, use a drop-in (see [Drop-in Overrides](#drop-in-overrides)) rather than editing the vendor file.

After modifying any unit file, run:

```bash
systemctl daemon-reload
```

This does not restart anything — it just re-reads unit files. Restart the unit separately if needed.

---

## Unit File Anatomy

Every unit file follows an INI-like format. Sections are `[Unit]`, `[Install]`, and one type-specific section (`[Service]`, `[Timer]`, `[Socket]`, etc.).

```ini
[Unit]
Description=My Application
Documentation=https://docs.example.com
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=myapp
ExecStart=/usr/bin/myapp --config /etc/myapp/config.toml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

The `[Install]` section is not processed at runtime — it tells `systemctl enable` where to create symlinks. `WantedBy=multi-user.target` is correct for the vast majority of services that should start in normal (non-graphical) multi-user mode.

---

## [Unit] Section

The `[Unit]` section declares metadata and relationships to other units. This is where dependency ordering lives.

### Description=

Human-readable name shown by `systemctl status` and `journalctl`. Be specific — "PostgreSQL 15 Database Server" beats "Database".

### Documentation=

URIs pointing to documentation. Multiple values are space- or newline-separated. Supports `http://`, `https://`, `man:`, `file:`, `info:`.

```ini
Documentation=https://docs.example.com man:myapp(8) file:/usr/share/doc/myapp/README
```

### Dependency Directives

This is where most people get it wrong. Understand the distinction between **ordering** and **requirement** — they are orthogonal.

#### Requires=

Hard requirement. If the listed unit fails to activate or is stopped, this unit is also stopped. If you list `Requires=postgresql.service` and Postgres crashes, your app unit is stopped too.

Use when your service **cannot function at all** without the dependency. This is stricter than most services need.

#### Wants=

Soft requirement. systemd will attempt to start the listed unit alongside this one, but if the dependency fails, this unit continues anyway. This is the **correct default for most dependencies**.

```ini
Wants=network-online.target
```

#### After= and Before=

Pure **ordering** directives — they say nothing about whether a dependency is required. `After=foo.service` means "start me after foo starts, stop me before foo stops."

**Critical rule:** `Requires=` and `Wants=` do NOT imply ordering. You almost always want both:

```ini
# Correct pattern
Wants=postgresql.service
After=postgresql.service
```

Without `After=`, systemd may start both units in parallel even if `Wants=` is set.

`Before=` is the inverse — less common, used by units that must prepare something before others start (e.g., a pre-flight check service).

#### BindsTo=

Stronger than `Requires=`. If the bound unit stops or enters failed state for **any reason** (including being explicitly stopped by an admin), this unit is also stopped. Use for tightly coupled pairs where one cannot meaningfully exist without the other — e.g., a service bound to a specific VPN tunnel interface.

```ini
BindsTo=sys-devices-virtual-net-tun0.device
After=sys-devices-virtual-net-tun0.device
```

#### PartOf=

Declares that this unit is part of a group. When the listed unit is restarted or stopped, this unit follows. But if **this** unit fails independently, the listed unit is not affected. Used for grouping units together for operational convenience.

#### Conflicts=

Mutual exclusion. If you start this unit and the listed unit is running, the listed unit is stopped first. Use for units that cannot coexist (e.g., two services binding the same port).

```ini
Conflicts=shutdown.target
```

### Dependency Ordering: The Full Picture

```
network.target          → network interfaces are up (not necessarily online)
network-online.target   → network is configured and reachable (requires NetworkManager-wait-online or similar)
remote-fs.target        → remote filesystems are mounted
nss-lookup.target       → DNS resolution is available
```

For services that make outbound connections, always use:
```ini
After=network-online.target
Wants=network-online.target
```

`network.target` is almost never what you want — it fires as soon as interfaces come up, before DHCP completes.

### ConditionPathExists= / AssertPathExists=

Conditions and assertions let you skip or abort unit activation:

```ini
# Skip silently if config missing
ConditionPathExists=/etc/myapp/config.toml

# Abort with failure if binary missing
AssertPathExists=/usr/bin/myapp
```

`Condition*` causes a silent skip (unit reports "condition failed", not "failed"). `Assert*` causes a hard failure.

---

## [Service] Section

### Service Types

The `Type=` directive controls how systemd determines that a service has finished starting. This is critically important — getting it wrong causes dependency issues where downstream services start before yours is actually ready.

#### Type=simple (default)

systemd considers the service started the moment `ExecStart` returns. The process is expected to stay in the foreground — systemd manages it directly.

**Use when:** Your daemon runs in the foreground natively (most modern Go, Rust, Python, Node.js services). This is the correct type for containerized-style processes.

**Do not use when:** The process forks into the background — readiness will be signaled immediately even though the real daemon may still be initializing.

```ini
Type=simple
ExecStart=/usr/bin/myapp serve
```

#### Type=forking

systemd expects the initial process to fork and exit. The parent exit signals "started." You almost always need to set `PIDFile=` so systemd can track the child.

**Use when:** Dealing with legacy daemons that double-fork (Apache httpd with MPM prefork, some C daemons, anything that calls `daemon(3)`). Prefer `Type=notify` or `Type=simple` for new software.

```ini
Type=forking
PIDFile=/run/myapp/myapp.pid
ExecStart=/usr/sbin/myapp -D
```

**Opinion:** If you're writing new software, never use this type. It's a workaround for 1990s daemon conventions.

#### Type=notify

The service is considered started when it sends `sd_notify(3)` with `READY=1` over a Unix socket that systemd provides via `$NOTIFY_SOCKET`. This is the **gold standard** for readiness signaling — it means "I am genuinely ready to accept requests."

**Use when:** Writing new daemons or when your software supports `sd_notify` (nginx 1.20+, PostgreSQL 14+ via `pg_ctl`, many modern services).

```ini
Type=notify
NotifyAccess=main   # or "exec" if forking is involved
ExecStart=/usr/bin/myapp
```

`NotifyAccess=` controls which processes may send notifications:
- `none` — notifications ignored
- `main` — only the main process (default)
- `exec` — any process spawned by ExecStart*
- `all` — any process in the service's cgroup

#### Type=oneshot

systemd waits for the process to exit before considering the unit active. Unlike `simple`, the unit can transition to "inactive" after the process exits (normal for tasks that run and complete).

**Use when:** Running scripts, cleanup tasks, setup tasks, anything that runs once and exits. Also the correct type for timer-triggered jobs.

```ini
Type=oneshot
RemainAfterExit=yes   # unit stays "active" after process exits — useful for setup units
ExecStart=/usr/local/bin/setup-networking.sh
ExecStop=/usr/local/bin/teardown-networking.sh
```

`RemainAfterExit=yes` is useful when you want `systemctl status` to show "active" for a setup unit that performed work but has no ongoing process.

#### Type=idle

Like `simple` but the process is delayed until all active jobs have dispatched. Rarely needed. Its original purpose was to prevent boot-time output interleaving with the console.

**Use when:** Essentially never. It's a niche option for boot-time cosmetics.

#### Type=dbus

The service is considered started when the specified `BusName=` appears on the D-Bus system bus. Implies a dependency on `dbus.service`.

**Use when:** The service registers a well-known D-Bus name as its readiness signal (common for desktop/session services, NetworkManager, BlueZ).

```ini
Type=dbus
BusName=org.freedesktop.NetworkManager
```

---

## Exec Directives

### ExecStart=

The main command. For all types except `oneshot`, exactly one `ExecStart=` is allowed. Must be an absolute path (or a path that resolves via `$PATH` when prefixed with certain flags).

```ini
ExecStart=/usr/bin/myapp --config /etc/myapp/config.toml
```

**Prefix characters:**

| Prefix | Meaning |
|---|---|
| `-` | Ignore non-zero exit code (e.g., `ExecStop=-/usr/bin/myapp stop`) |
| `@` | Pass argv[0] separately from the actual binary path |
| `+` | Run with full privileges (ignores User/Group sandboxing) |
| `!` | Similar to `+` but without ambient capabilities |
| `!!` | Similar but capability sets are filtered |

For `oneshot` units, you may list multiple `ExecStart=` lines — they execute sequentially.

### ExecStartPre=

Runs **before** `ExecStart`. If any `ExecStartPre=` command fails (non-zero exit), `ExecStart` is not run and the unit enters failed state — unless prefixed with `-`.

Common uses: wait for a dependency to be truly ready, create directories, validate config.

```ini
ExecStartPre=/usr/bin/myapp --validate-config /etc/myapp/config.toml
ExecStartPre=/usr/bin/install -d -m 0750 -o myapp -g myapp /var/run/myapp
```

### ExecStartPost=

Runs **after** `ExecStart` but the unit is not considered started until these commands also complete. Note: for `Type=simple`, this runs after the process is forked, not after readiness is confirmed. Use `Type=notify` if you need true post-readiness hooks.

### ExecReload=

Command to reload configuration without a full restart. Should signal the process to re-read its config.

```ini
ExecReload=/bin/kill -HUP $MAINPID
# Or for systemd-aware services:
ExecReload=/usr/bin/myapp reload
```

`$MAINPID` is a special variable substituted by systemd with the main process PID.

### ExecStop=

Command to stop the service. If not set, systemd sends `SIGTERM` then `SIGKILL` after `TimeoutStopSec`.

```ini
ExecStop=/usr/bin/myapp shutdown --graceful
```

### ExecStopPost=

Always runs after the service stops, regardless of how it stopped (clean exit, crash, killed). Ideal for cleanup that must happen even on failure.

```ini
ExecStopPost=/usr/local/bin/cleanup-pidfile.sh
ExecStopPost=-/bin/rm -f /run/myapp/myapp.pid
```

### TimeoutStartSec= and TimeoutStopSec=

```ini
TimeoutStartSec=30      # Fail if not ready within 30s (default 90s)
TimeoutStopSec=60       # Allow 60s for graceful shutdown before SIGKILL (default 90s)
TimeoutSec=30           # Shorthand that sets both
```

For services that take a long time to start (JVM warmup, database recovery), increase `TimeoutStartSec`. For services with expensive shutdown (flushing write-ahead logs, draining connections), increase `TimeoutStopSec`.

Set `TimeoutStopSec=0` only if you are certain the process always exits cleanly — otherwise you risk zombie cleanup issues.

### KillMode=

Controls what gets killed when stopping:

- `control-group` (default) — kill all processes in the cgroup (correct for most services)
- `process` — only kill the main process; child processes are left running
- `mixed` — send SIGTERM to main, SIGKILL to cgroup after timeout
- `none` — don't kill anything, rely solely on `ExecStop=`

**Opinion:** Almost never change this from the default. `control-group` is the right behavior — it ensures cleanup of all forked workers.

---

## Restart Policies

### Restart=

| Value | Restarts on |
|---|---|
| `no` | Never (default) |
| `on-success` | Only if exit code 0 |
| `on-failure` | Non-zero exit, signal, timeout, watchdog timeout |
| `on-abnormal` | Signal, timeout, watchdog — NOT on clean exit or non-zero exit code |
| `on-abort` | Only on uncaught signal |
| `on-watchdog` | Only on watchdog timeout |
| `always` | Every time, regardless of reason |

**Opinions:**

- `on-failure` is correct for almost all production services. It restarts on crashes and OOM kills, but not on intentional `systemctl stop`.
- `always` restarts even when you stop the service with `systemctl stop`, which fights you during maintenance. Avoid it unless you have a very specific need.
- `on-abnormal` is useful for services where a non-zero exit is a deliberate "I am done" signal (batch jobs inside a persistent wrapper).

```ini
Restart=on-failure
```

### RestartSec=

Delay between restart attempts. Defaults to 100ms — far too aggressive for most services.

```ini
RestartSec=5s       # Wait 5 seconds before restarting
```

For services that connect to databases or external APIs, use at least 5-10 seconds to give dependencies time to recover and to prevent hammering them.

### StartLimitBurst= and StartLimitIntervalSec=

Rate limiting for restarts. If the service starts more than `StartLimitBurst` times within `StartLimitIntervalSec`, systemd stops attempting restarts and puts the unit in "failed" state.

```ini
StartLimitIntervalSec=60s
StartLimitBurst=3
```

This means: if the service fails 3 times in 60 seconds, give up and alert. This is the correct behavior — a service restart-looping at 100ms would exhaust resources and mask the root cause.

To manually reset after hitting the limit:
```bash
systemctl reset-failed myapp.service
systemctl start myapp.service
```

**Note:** `StartLimitIntervalSec` lives in `[Unit]`, not `[Service]`, in systemd 230+. In older versions, it was in `[Service]`. When in doubt, put it in `[Unit]`.

### Watchdog

For `Type=notify` services, you can enable watchdog support:

```ini
WatchdogSec=30s
```

The service must call `sd_notify(WATCHDOG=1)` at least once per `WatchdogSec` interval. If it fails to do so, systemd considers the service hung and takes action per `Restart=`.

---

## Environment and Identity

### Environment=

Set individual environment variables inline:

```ini
Environment=APP_ENV=production
Environment=LOG_LEVEL=info PORT=8080
Environment="DATABASE_URL=postgres://localhost/mydb"
```

Multiple `Environment=` lines accumulate. Quote values containing spaces with double quotes.

**Security note:** Do not put secrets here. They appear in `systemctl show`, process environment (visible to other users via `/proc`), and the unit file itself. Use `EnvironmentFile=` with a mode-restricted file.

### EnvironmentFile=

Load variables from a file (one `KEY=VALUE` per line, `#` for comments):

```ini
EnvironmentFile=/etc/myapp/environment
EnvironmentFile=-/etc/myapp/environment.local   # dash = ignore if missing
```

Secure this file properly:
```bash
chmod 0600 /etc/myapp/environment
chown root:myapp /etc/myapp/environment
```

Variables set in `EnvironmentFile=` can be referenced in later directives:
```ini
EnvironmentFile=/etc/myapp/environment
ExecStart=/usr/bin/myapp --port ${PORT}
```

### PassEnvironment=

Pass specific variables from systemd's own environment (not common for system services, more relevant for user units):

```ini
PassEnvironment=HOME PATH LANG
```

### User= and Group=

Drop privileges after starting:

```ini
User=myapp
Group=myapp
```

**Best practice:** Create a dedicated system user with no shell and no home directory for every service:

```bash
useradd --system --no-create-home --shell /usr/sbin/nologin --user-group myapp
```

If the service needs to bind a privileged port (<1024) but otherwise should run unprivileged, use `AmbientCapabilities=CAP_NET_BIND_SERVICE` rather than running as root.

### WorkingDirectory=

Set the working directory for the service process:

```ini
WorkingDirectory=/var/lib/myapp
```

Defaults to the root directory `/` when `User=` is set (to avoid unexpected behavior if a home directory does not exist). Always set explicitly for services that use relative paths.

Use `WorkingDirectory=~` to use the user's home directory.

### SupplementaryGroups=

Add the service user to additional groups without changing its primary group:

```ini
SupplementaryGroups=ssl-cert video
```

---

## Sandboxing

This is where systemd truly shines as an init system. Properly sandboxed services are a core part of defense-in-depth. Apply as many of these as your service tolerates — the goal is minimum necessary access.

**General approach:** Start with the full set, run the service, check `journalctl -u myapp` for EPERM or EACCES errors, and relax only what is genuinely needed.

Use `systemd-analyze security myapp.service` to get an exposure score before and after sandboxing.

### NoNewPrivileges=yes

**What it does:** Prevents the service process (and all its children) from gaining new privileges via `setuid` binaries, file capabilities, or `prctl(PR_SET_SECUREBIT)`. Once set, cannot be unset.

**Impact:** Breaks `sudo`, `su`, `newgrp`, and any setuid binary called from within the service. For application services that should never escalate privileges, this is safe.

**Always set this.** There is almost no legitimate reason for an application service to spawn setuid processes.

```ini
NoNewPrivileges=yes
```

### PrivateTmp=yes

**What it does:** Mounts a private, service-specific tmpfs at `/tmp` and `/var/tmp`. The service's temporary files are invisible to other services and the rest of the system.

**Impact:** Temporary files created by the service are isolated. Files placed in `/tmp` by other processes are not visible to this service. Files written to the service's private `/tmp` are cleaned up when the service stops.

**Always set this** for any service that should not share temporary state with others.

```ini
PrivateTmp=yes
```

### ProtectSystem=

Controls write access to the operating system directory tree.

| Value | Behavior |
|---|---|
| `no` | No protection (default) |
| `yes` | Mount `/usr` and `/boot` read-only |
| `full` | Mount `/usr`, `/boot`, and `/etc` read-only |
| `strict` | Mount the entire filesystem read-only, except for `/dev`, `/proc`, `/sys` |

**Recommendation:** Use `strict` for all application services. Then use `ReadWritePaths=` to explicitly grant write access to the specific paths the service needs.

```ini
ProtectSystem=strict
ReadWritePaths=/var/lib/myapp /var/log/myapp /run/myapp
```

### ProtectHome=

Controls access to home directories.

| Value | Behavior |
|---|---|
| `no` | No protection (default) |
| `yes` | `/home`, `/root`, `/run/user` are empty and inaccessible |
| `read-only` | Home directories visible but read-only |
| `tmpfs` | Temporary empty tmpfs mounted over home directories |

**Recommendation:** `yes` for all services not running as a real user. Services running as `myapp` system user have no home directory to protect anyway, but this prevents accidental access to other users' homes.

```ini
ProtectHome=yes
```

### ReadWritePaths=

Exceptions to `ProtectSystem=strict`. List every path the service legitimately writes to:

```ini
ReadWritePaths=/var/lib/myapp /var/log/myapp /run/myapp /tmp/myapp-scratch
```

You can also use `ReadOnlyPaths=` to explicitly make paths read-only even when `ProtectSystem=` would not cover them, and `InaccessiblePaths=` to make paths completely invisible.

### PrivateDevices=yes

**What it does:** Mounts a minimal private `/dev` containing only pseudo-devices (`null`, `zero`, `full`, `random`, `urandom`, `tty`). Physical devices, block devices, and most character devices are hidden.

**Impact:** The service cannot access `/dev/sda`, `/dev/nvme0n1`, etc. Also implies `DevicePolicy=closed` in the cgroup.

**Always set this** for application services. Only skip for services that genuinely need raw device access (disk management tools, GPU compute services).

```ini
PrivateDevices=yes
```

### ProtectKernelTunables=yes

**What it does:** Makes `/proc/sys`, `/sys`, `/proc/sysrq-trigger`, `/proc/latency_stats`, `/proc/acpi`, `/proc/timer_stats`, `/proc/fs` read-only.

**Impact:** The service cannot modify kernel parameters via `sysctl`. Does not prevent reading them.

**Always set this.** Application services have no business modifying kernel tunables at runtime.

```ini
ProtectKernelTunables=yes
```

### ProtectKernelModules=yes

**What it does:** Prevents loading or unloading kernel modules (`CAP_SYS_MODULE` is stripped and `init_module`/`finit_module`/`delete_module` syscalls are blocked).

**Impact:** The service cannot load kernel modules. Breaks tools like `insmod`, `rmmod`, `modprobe`.

**Always set this** for application services.

```ini
ProtectKernelModules=yes
```

### ProtectKernelLogs=yes

**What it does:** Prevents access to the kernel log buffer (blocks `CAP_SYSLOG`, restricts `/proc/kmsg`, `/dev/kmsg`).

**Impact:** The service cannot read kernel messages. Set this for any service that does not need to read kernel logs.

```ini
ProtectKernelLogs=yes
```

### ProtectControlGroups=yes

**What it does:** Makes the cgroup filesystem (`/sys/fs/cgroup`) read-only for the service.

**Impact:** The service cannot modify cgroup settings, add processes to other cgroups, or escape its own cgroup. Prevents a class of container-escape-style attacks.

**Always set this** for application services.

```ini
ProtectControlGroups=yes
```

### RestrictAddressFamilies=

Limits which socket address families the service may use. This is a powerful control — if your service only speaks TCP/IP, deny everything else.

```ini
# Web service that only needs TCP/IP and Unix sockets
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

# Service with no network access
RestrictAddressFamilies=AF_UNIX
```

Common families:
- `AF_INET` — IPv4
- `AF_INET6` — IPv6
- `AF_UNIX` — Unix domain sockets
- `AF_NETLINK` — Kernel netlink (needed by tools that query the kernel network stack)
- `AF_PACKET` — Raw packet access (only for tcpdump-style tools)

Prefix with `~` to deny specific families and allow all others:
```ini
RestrictAddressFamilies=~AF_PACKET AF_NETLINK
```

### RestrictNamespaces=yes

**What it does:** Prevents the service from creating new kernel namespaces (network, PID, mount, user, UTS, IPC, cgroup namespaces).

**Impact:** The service cannot use namespace tricks to escape its sandbox, cannot create containers or overlay filesystems. Breaks tools that use user namespaces (some container runtimes, `unshare`, `bubblewrap`).

```ini
RestrictNamespaces=yes
```

You can also specify which namespaces to allow:
```ini
RestrictNamespaces=~CLONE_NEWUSER CLONE_NEWNET
```

### LockPersonality=yes

**What it does:** Locks the execution domain (personality) of the process. Prevents switching between Linux, BSD, and other execution personalities via `personality(2)`.

**Impact:** The service cannot use `personality()` to switch to a different syscall ABI. Eliminates a niche but real attack surface.

```ini
LockPersonality=yes
```

### MemoryDenyWriteExecute=yes

**What it does:** Prevents creating memory mappings that are both writable and executable simultaneously (`mmap` with `PROT_WRITE|PROT_EXEC`). Also blocks `mprotect` from adding `PROT_EXEC` to writable mappings.

**Impact:** Defeats a major class of memory corruption exploits (shellcode injection, JIT spray). **However:** This breaks JIT-compiled languages and runtimes — JVM (Java, Scala, Kotlin), V8 (Node.js, Deno), LuaJIT, PyPy, .NET, and anything using `libffi`.

```ini
# Safe for: C, C++ (non-JIT), Go, Rust, Python (CPython), Ruby (MRI)
MemoryDenyWriteExecute=yes

# Do NOT set for: Java, Node.js, .NET, Erlang/BEAM, LuaJIT, PyPy
```

When in doubt, test it. The service will fail with EACCES/EPERM if this breaks it.

### RestrictRealtime=yes

**What it does:** Prevents the service from setting real-time scheduling policies (`SCHED_FIFO`, `SCHED_RR`) that could starve other processes.

**Impact:** The service cannot use `sched_setscheduler()` to elevate to real-time priority. Breaks audio daemons, specialized real-time applications.

```ini
RestrictRealtime=yes
```

### SystemCallFilter=

Limits which syscalls the service may execute. Violations result in the process being killed with SIGSYS (and optionally logged).

systemd provides pre-defined syscall groups prefixed with `@`:

| Group | Contents |
|---|---|
| `@system-service` | Comprehensive set for typical daemon — use this as your baseline |
| `@basic-io` | read, write, close, etc. |
| `@file-system` | File operations |
| `@network-io` | Socket operations |
| `@process` | fork, exec, exit, etc. |
| `@signal` | Signal handling |
| `@ipc` | IPC primitives |
| `@sync` | fsync, fdatasync, etc. |
| `@aio` | Async I/O |
| `@privileged` | Privileged operations (generally deny this) |
| `@resources` | Resource limit changes |
| `@module` | Module loading (deny this) |
| `@raw-io` | Raw device I/O (deny this) |
| `@mount` | mount/umount (deny this) |
| `@reboot` | Reboot-related calls (deny this) |
| `@swap` | Swap management (deny this) |
| `@clock` | Clock setting (deny unless needed) |
| `@obsolete` | Old/obsolete syscalls (deny this) |
| `@cpu-emulation` | CPU emulation (deny unless needed) |

**Recommended baseline for a typical application service:**

```ini
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources @module @raw-io @mount @reboot @swap @obsolete @cpu-emulation
```

To return EPERM instead of killing the process (useful during debugging):
```ini
SystemCallErrorNumber=EPERM
```

**Important:** Test `SystemCallFilter=` carefully. Overly restrictive filters cause mysterious crashes. Start with `@system-service` and add exclusions one by one.

### CapabilityBoundingSet=

Sets the bounding set of Linux capabilities. The service cannot have any capability that is not in this set, regardless of file capabilities or other mechanisms.

```ini
# No capabilities at all (most restrictive — correct for most app services)
CapabilityBoundingSet=

# Only allow binding to privileged ports
CapabilityBoundingSet=CAP_NET_BIND_SERVICE

# Networking service that needs more
CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_NET_RAW
```

An empty `CapabilityBoundingSet=` is the right answer for application services running as an unprivileged user. Combined with `NoNewPrivileges=yes`, this creates a strong capability constraint.

### AmbientCapabilities=

Grants capabilities to the service process even when running as an unprivileged user, without requiring file capabilities on the binary. The capability must also be in `CapabilityBoundingSet=`.

```ini
# Allow binding port 443 without running as root
User=myapp
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
```

This replaces the old pattern of `authbind`, `setcap`, or running as root just to bind a privileged port. Use it.

---

## Timer Units

Timers replace cron for systemd-managed services. They are more flexible, integrate with journald logging, support missed-run catch-up, and can be introspected with `systemctl`.

Every timer unit has a corresponding service unit with the same basename. `backup.timer` triggers `backup.service`.

### OnCalendar= — Calendar Event Syntax

```ini
[Timer]
# Every day at midnight
OnCalendar=daily

# Every day at 02:00
OnCalendar=*-*-* 02:00:00

# Every Monday–Friday at 09:00
OnCalendar=Mon-Fri 09:00

# Every hour at :30
OnCalendar=*:30:00

# Every 15 minutes
OnCalendar=*:0/15

# First day of every month at 03:00
OnCalendar=*-*-1 03:00:00

# Every Sunday at 22:00
OnCalendar=Sun 22:00

# Specific date
OnCalendar=2026-12-31 23:59:00

# Every weekday, multiple times
OnCalendar=Mon-Fri 08:00,12:00,17:00
```

Validate calendar expressions before deploying:
```bash
systemd-analyze calendar "Mon-Fri 09:00"
```

### Monotonic Timers

Alternative to calendar — triggers relative to an event:

```ini
OnBootSec=15min          # 15 minutes after boot
OnActiveSec=1h           # 1 hour after timer activated
OnStartupSec=5min        # 5 minutes after systemd started (same as OnBootSec for system services)
OnUnitActiveSec=24h      # 24 hours after the service last ran
OnUnitInactiveSec=1h     # 1 hour after the service last completed
```

### Persistent=yes

If the system was off when a timer should have fired, `Persistent=yes` causes the timer to trigger immediately at next opportunity.

```ini
Persistent=yes
```

**Use for:** Backup jobs, log rotation, anything that must not be skipped. If your nightly backup timer fired while the server was down for maintenance, `Persistent=yes` ensures it runs as soon as the server comes back up.

**Do not use for:** Jobs that should only run at the scheduled time (e.g., sending a daily newsletter — you do not want to send it at 4pm because the server was down at midnight).

### RandomizedDelaySec=

Add a random delay up to the specified duration before triggering. Prevents multiple servers from hitting the same external resource simultaneously (thundering herd).

```ini
RandomizedDelaySec=1h
```

With `OnCalendar=daily` and `RandomizedDelaySec=1h`, the timer fires somewhere between midnight and 01:00, different each day.

**Always set this** on timer jobs that hit databases, APIs, or shared storage when multiple servers run the same timer.

### AccuracySec=

How accurately the timer fires. Default is 1 minute — systemd can coalesce timers to save wakeups. Reduce for time-sensitive jobs, leave at default for background tasks.

```ini
AccuracySec=1s     # Fire within 1 second of scheduled time
AccuracySec=10min  # Fire within 10 minutes (saves power, fine for backups)
```

### Complete Timer Example

```ini
# /etc/systemd/system/db-backup.timer
[Unit]
Description=Database Backup Timer
Documentation=https://wiki.example.com/ops/backups

[Timer]
OnCalendar=*-*-* 02:00:00
RandomizedDelaySec=30min
Persistent=yes
AccuracySec=1min

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/db-backup.service
[Unit]
Description=Database Backup Job
After=postgresql.service

[Service]
Type=oneshot
User=backup
ExecStart=/usr/local/bin/backup-database.sh
```

Enable and start:
```bash
systemctl enable --now db-backup.timer
systemctl list-timers --all    # See all timers and next fire time
```

---

## Socket Activation

Socket activation allows systemd to listen on a socket and start the service only when a connection arrives. Benefits: faster boot (service does not need to start until used), automatic restart with no dropped connections (socket stays open while service restarts), and privilege separation (socket binding done by systemd as root, service runs unprivileged).

### Socket Unit

```ini
# /etc/systemd/system/myapp.socket
[Unit]
Description=MyApp Socket

[Socket]
ListenStream=8080
# Or for Unix socket:
# ListenStream=/run/myapp/myapp.sock
# SocketUser=myapp
# SocketGroup=myapp
# SocketMode=0660

Accept=no    # Pass socket fd to a single service instance (correct for most apps)
# Accept=yes would fork a new service instance per connection (inetd style)

[Install]
WantedBy=sockets.target
```

### Triggered Service

The service does not need any special `ListenStream=` — it receives the pre-opened socket via `$LISTEN_FDS`:

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=MyApp Service
Requires=myapp.socket

[Service]
Type=simple
ExecStart=/usr/bin/myapp
# Application reads socket from sd_listen_fds() or via $LISTEN_FDS / $LISTEN_PID
```

### Accept=no vs Accept=yes

- `Accept=no` (recommended): One service instance handles all connections. The socket fd is passed directly. Works with servers that implement their own accept loop.
- `Accept=yes`: systemd accepts each connection and spawns a new service instance per connection (inetd model). The service unit must be a template (`myapp@.service`).

### Application Support

The service must use `sd_listen_fds(3)` (from libsystemd) or manually check `$LISTEN_FDS` and `$LISTEN_PID`. Most frameworks have built-in support:

- Go: `github.com/coreos/go-systemd/activation`
- Python: `systemd.daemon.listen_fds()`
- Node.js: `sd-listen-fds` package
- nginx, Apache, PostgreSQL: native support

Enable the socket (not the service directly):
```bash
systemctl enable --now myapp.socket
```

---

## Drop-in Overrides

Never edit vendor unit files in `/usr/lib/systemd/system/`. When a package update ships a new version, your edits are silently overwritten.

Instead, use drop-in overrides.

### The Right Way: systemctl edit

```bash
systemctl edit myapp.service
```

This opens `$EDITOR` on a new file at `/etc/systemd/system/myapp.service.d/override.conf`. Save and close — systemd reloads automatically (daemon-reload is run for you).

To override a full unit (replace rather than extend):
```bash
systemctl edit --full myapp.service
```

This copies the unit to `/etc/systemd/system/` and opens it. You are now fully responsible for maintaining this file through upgrades.

### Manual Drop-in Structure

```bash
mkdir -p /etc/systemd/system/myapp.service.d/
```

```ini
# /etc/systemd/system/myapp.service.d/override.conf
[Service]
# To CLEAR a list directive before setting new value:
ExecStart=
ExecStart=/usr/bin/myapp --new-flag

# To add environment variables:
Environment=LOG_LEVEL=debug

# To change resource limits:
LimitNOFILE=65536
```

**Critical:** For list-type directives (`ExecStart=`, `ExecStartPre=`, `Environment=`, etc.), you must first set an empty value to clear the inherited list before setting new values. Otherwise the new values are **appended** to the existing list.

### Viewing Effective Configuration

```bash
# Show the merged/effective unit configuration
systemctl cat myapp.service

# Show what drop-ins are applied
systemctl status myapp.service   # shows "Drop-In:" section

# Full property dump
systemctl show myapp.service
```

### Common Override Patterns

**Increase file descriptor limit for a package-managed service:**
```ini
# /etc/systemd/system/nginx.service.d/limits.conf
[Service]
LimitNOFILE=65536
```

**Add environment variables to a vendor service:**
```ini
# /etc/systemd/system/postgresql.service.d/environment.conf
[Service]
Environment=PGDATA=/data/postgresql/14/main
```

**Change restart policy on a package service:**
```ini
# /etc/systemd/system/redis.service.d/restart.conf
[Service]
Restart=on-failure
RestartSec=5s
```

---

## journalctl Reference

journald collects all output from systemd-managed services (stdout/stderr), kernel messages, and structured log data. It is your primary diagnostic tool.

### Basic Filtering

```bash
# Follow a specific service (like tail -f for a unit)
journalctl -u myapp.service -f

# Last 100 lines of a service
journalctl -u myapp.service -n 100

# All logs since boot
journalctl -b

# Previous boot (useful after a crash)
journalctl -b -1

# Specific time range
journalctl --since "2026-04-06 00:00:00" --until "2026-04-06 06:00:00"
journalctl --since "1 hour ago"
journalctl --since yesterday

# Multiple units
journalctl -u myapp.service -u postgresql.service
```

### Priority Filtering

```bash
# Show only errors and above
journalctl -p err
journalctl -p err..alert   # range from err down to alert

# Priority levels (syslog values):
# 0=emerg, 1=alert, 2=crit, 3=err, 4=warning, 5=notice, 6=info, 7=debug
journalctl -p warning       # warning and higher severity
journalctl -p debug         # everything including debug
```

### Output Formats

```bash
# JSON output (one JSON object per line)
journalctl -u myapp.service -o json

# Pretty-printed JSON (readable but verbose)
journalctl -u myapp.service -o json-pretty

# Just the message text, no metadata
journalctl -u myapp.service -o cat

# Short with microsecond timestamps
journalctl -u myapp.service -o short-precise

# Other formats: verbose, export, with-unit
```

### Practical Workflows

```bash
# No pager (pipe to other tools)
journalctl -u myapp.service --no-pager | grep -i error

# Show logs since service last started
journalctl -u myapp.service --since "$(systemctl show -p ActiveEnterTimestamp myapp.service | cut -d= -f2)"

# Count errors per minute (JSON + jq)
journalctl -u myapp.service -o json --no-pager | \
  jq -r '.PRIORITY + " " + .__REALTIME_TIMESTAMP' | \
  awk '$1 <= 3'   # err and above

# Kernel messages only
journalctl -k

# Follow multiple units
journalctl -f -u myapp.service -u postgresql.service

# Disk usage
journalctl --disk-usage
```

### Persistent Journal

By default, journald stores logs in memory (`/run/log/journal/`) and they are lost on reboot. Enable persistent storage:

```bash
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
# Or simply restart journald:
systemctl restart systemd-journald
```

Or configure in `/etc/systemd/journald.conf`:

```ini
[Journal]
Storage=persistent      # persistent | volatile | auto | none
SystemMaxUse=2G         # Max disk space for system journal
SystemKeepFree=1G       # Always keep this much free
SystemMaxFileSize=200M  # Max per-file size before rotation
MaxRetentionSec=30day   # Delete entries older than this
MaxFileSec=1week        # Rotate files older than this
Compress=yes            # Compress journal files
```

Apply changes:
```bash
systemctl restart systemd-journald
```

### Forwarding to syslog

To forward to a traditional syslog daemon (rsyslog, syslog-ng) alongside journald:

```ini
# /etc/systemd/journald.conf
[Journal]
ForwardToSyslog=yes
```

---

## Production Service Template

This is a complete, production-ready service unit demonstrating all major security and operational options. Copy and adapt it.

```ini
# /etc/systemd/system/myapp.service
#
# Production service template — myapp
# Adjust paths, user, and sandboxing to match your application.

[Unit]
Description=MyApp — Production Application Server
Documentation=https://docs.example.com/myapp https://github.com/example/myapp
After=network-online.target postgresql.service
Wants=network-online.target
Requires=postgresql.service

# Rate-limit restarts: max 3 failures in 60 seconds before giving up
StartLimitIntervalSec=60s
StartLimitBurst=3

[Service]
# ---------------------------------------------------------------------------
# Process type and identity
# ---------------------------------------------------------------------------
Type=notify
User=myapp
Group=myapp
SupplementaryGroups=ssl-cert

# ---------------------------------------------------------------------------
# Working environment
# ---------------------------------------------------------------------------
WorkingDirectory=/var/lib/myapp
RuntimeDirectory=myapp          # creates /run/myapp, owned by User=, cleaned up on stop
StateDirectory=myapp            # creates /var/lib/myapp, owned by User=, persists
LogsDirectory=myapp             # creates /var/log/myapp, owned by User=, persists
ConfigurationDirectory=myapp    # creates /etc/myapp, owned by root (read-only for service)

# ---------------------------------------------------------------------------
# Environment
# ---------------------------------------------------------------------------
Environment=APP_ENV=production
Environment=LOG_FORMAT=json
EnvironmentFile=/etc/myapp/environment   # Contains secrets (chmod 0600, chown root:myapp)

# ---------------------------------------------------------------------------
# Exec lifecycle
# ---------------------------------------------------------------------------
ExecStartPre=/usr/bin/myapp --check-config
ExecStart=/usr/bin/myapp serve --socket /run/myapp/myapp.sock
ExecReload=/bin/kill -HUP $MAINPID
ExecStopPost=-/usr/local/lib/myapp/cleanup.sh

# ---------------------------------------------------------------------------
# Restart policy
# ---------------------------------------------------------------------------
Restart=on-failure
RestartSec=10s
TimeoutStartSec=60s
TimeoutStopSec=30s
WatchdogSec=60s       # Requires application to call sd_notify(WATCHDOG=1) periodically

# ---------------------------------------------------------------------------
# Resource limits
# ---------------------------------------------------------------------------
LimitNOFILE=65536
LimitNPROC=512
MemoryMax=2G          # OOM-kill this service before others (cgroup limit)
CPUQuota=200%         # Allow up to 2 CPU cores worth of time
TasksMax=256          # Limit number of threads

# ---------------------------------------------------------------------------
# Security hardening
# ---------------------------------------------------------------------------

# Prevent privilege escalation via setuid/capabilities
NoNewPrivileges=yes

# Private temporary filesystem — invisible to other services
PrivateTmp=yes

# Make entire filesystem read-only except explicit ReadWritePaths
ProtectSystem=strict

# Block access to home directories
ProtectHome=yes

# Grant write access only to necessary paths
ReadWritePaths=/var/lib/myapp /var/log/myapp /run/myapp

# Hide physical devices; only expose null, zero, random, urandom, tty
PrivateDevices=yes

# Prevent modifying kernel parameters
ProtectKernelTunables=yes

# Prevent loading/unloading kernel modules
ProtectKernelModules=yes

# Prevent reading kernel log buffer
ProtectKernelLogs=yes

# Make cgroup filesystem read-only
ProtectControlGroups=yes

# Restrict to only the socket families this application uses
# Adjust to match your actual networking needs
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

# Prevent namespace creation (container escape vector)
RestrictNamespaces=yes

# Lock execution domain/personality
LockPersonality=yes

# Prevent W+X memory mappings (breaks JIT runtimes — remove for Java/Node.js)
MemoryDenyWriteExecute=yes

# Prevent real-time scheduling
RestrictRealtime=yes

# Allow only syscalls expected for a normal daemon
# Explicitly deny dangerous groups
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources @module @raw-io @mount @reboot @swap @obsolete @cpu-emulation
SystemCallArchitectures=native   # Prevent 32-bit syscall abuse on 64-bit systems

# Drop all capabilities
CapabilityBoundingSet=

# If the service needs to bind a privileged port uncomment:
# CapabilityBoundingSet=CAP_NET_BIND_SERVICE
# AmbientCapabilities=CAP_NET_BIND_SERVICE

# Misc
ProtectClock=yes         # Prevent changing system clock
ProtectHostname=yes      # Prevent changing hostname
PrivateIPC=yes           # Isolated IPC namespace (SysV IPC, POSIX MQ)
RemoveIPC=yes            # Remove IPC objects on service stop

# Hardened proc filesystem
ProcSubset=pid           # Only show own PIDs in /proc (requires Linux 5.8+)
ProtectProc=invisible    # /proc entries of other processes inaccessible

[Install]
WantedBy=multi-user.target
```

### Applying Security Incrementally

Start with a minimal service, then apply sandboxing one option at a time, restarting and testing after each. Use this workflow:

```bash
# Check current security exposure score (0 = fully locked, 10 = exposed)
systemd-analyze security myapp.service

# Watch for errors when testing new restrictions
journalctl -u myapp.service -f &

# Apply and test
systemctl daemon-reload && systemctl restart myapp.service

# Check for EPERM, EACCES, SIGSYS in logs
journalctl -u myapp.service -n 50 -p warning
```

---

## Operational Cheatsheet

### Service Management

```bash
# Start, stop, restart, reload
systemctl start myapp.service
systemctl stop myapp.service
systemctl restart myapp.service
systemctl reload myapp.service          # Reload config without restart
systemctl try-restart myapp.service     # Restart only if currently running
systemctl try-reload-or-restart myapp.service

# Enable/disable at boot
systemctl enable myapp.service          # Create symlinks
systemctl disable myapp.service         # Remove symlinks
systemctl enable --now myapp.service    # Enable and start immediately
systemctl disable --now myapp.service   # Disable and stop immediately

# Mask/unmask (prevent start even manually)
systemctl mask myapp.service
systemctl unmask myapp.service

# Status and inspection
systemctl status myapp.service
systemctl is-active myapp.service       # Returns exit code 0 if active
systemctl is-enabled myapp.service
systemctl is-failed myapp.service

# Reset failed state
systemctl reset-failed myapp.service

# List units
systemctl list-units --type=service
systemctl list-units --state=failed
systemctl list-timers --all
```

### Dependency Visualization

```bash
# Show what a unit depends on
systemctl list-dependencies myapp.service

# Show what depends on a unit
systemctl list-dependencies --reverse myapp.service

# Show full dependency tree
systemctl list-dependencies --all myapp.service
```

### Boot Analysis

```bash
# Overall boot time breakdown
systemd-analyze

# Blame: list units sorted by startup time
systemd-analyze blame

# Critical chain (longest startup path)
systemd-analyze critical-chain myapp.service

# Generate boot SVG timeline
systemd-analyze plot > /tmp/boot.svg

# Verify unit file syntax
systemd-analyze verify myapp.service

# Check security score
systemd-analyze security myapp.service
```

### systemd-run — Transient Units

Run a command as a transient systemd service without writing a unit file:

```bash
# Run in a transient service with sandboxing
systemd-run --unit=test-run \
  --property=User=nobody \
  --property=PrivateTmp=yes \
  --property=ProtectSystem=strict \
  /usr/bin/myapp --once

# Run as a transient timer (in 5 minutes)
systemd-run --on-active=5min /usr/local/bin/one-off-task.sh

# Interactive shell in a scope (same cgroup as your session)
systemd-run --scope --user bash
```

### Resource Control

```bash
# Set memory limit on running service (transient, until next restart)
systemctl set-property myapp.service MemoryMax=1G

# Persistent resource limits (writes a drop-in)
systemctl set-property --runtime myapp.service CPUQuota=50%

# Show cgroup hierarchy
systemd-cgls

# Show resource usage per service
systemd-cgtop
```

---

*Last reviewed: April 2026. Covers systemd 252+ (RHEL 9.x, Ubuntu 22.04+, Debian 12+).*
*Man pages: `man systemd.service`, `man systemd.exec`, `man systemd.unit`, `man systemd.timer`, `man systemd.socket`, `man journalctl`.*
