# Prometheus + Grafana + Loki Monitoring Stack — Production Setup Guide

> Written from a senior sysadmin perspective. This is not a "hello world" tutorial. Every decision
> here has a reason behind it. Read the annotations, don't just cargo-cult the YAML.

**Stack versions (as of early 2026 — verify latest before deploying):**
- Prometheus: 2.51+
- Node Exporter: 1.8+
- Alertmanager: 0.27+
- Grafana: 10.x (OSS)
- Loki: 3.x
- Promtail: 3.x (matches Loki version exactly — always keep these in sync)

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Node Exporter — Installation and Hardening](#node-exporter)
3. [Prometheus — Full Annotated Configuration](#prometheus-configuration)
4. [Alert Rules — Five Critical Rules with PromQL](#alert-rules)
5. [Alertmanager — Routing, Receivers, Inhibition](#alertmanager)
6. [Grafana — Installation, Provisioning, Dashboards](#grafana)
7. [Loki + Promtail — PLG Stack](#loki-promtail-plg-stack)
8. [Uptime Monitoring — UptimeRobot + healthchecks.io](#uptime-monitoring)
9. [Operational Runbook](#operational-runbook)

---

## Architecture Overview

```
                    ┌──────────────────────────────────────────────┐
                    │              Monitored Hosts                  │
                    │  node_exporter :9100   app metrics :8080      │
                    └────────────────┬─────────────────────────────┘
                                     │  scrape (pull)
                    ┌────────────────▼─────────────────────────────┐
                    │            Prometheus :9090                   │
                    │   TSDB on disk   →   Alert Rules              │
                    └──────┬─────────────────────┬─────────────────┘
                           │ query                │ fire alerts
               ┌───────────▼──────┐   ┌──────────▼───────────────┐
               │   Grafana :3000  │   │  Alertmanager :9093       │
               │  dashboards      │   │  route → email/Slack/PD   │
               └──────────────────┘   └──────────────────────────┘

                    ┌──────────────────────────────────────────────┐
                    │              Monitored Hosts                  │
                    │  systemd journal + /var/log/**/*.log          │
                    └────────────────┬─────────────────────────────┘
                                     │  tail (push)
                    ┌────────────────▼─────────────────────────────┐
                    │              Promtail :9080                   │
                    └────────────────┬─────────────────────────────┘
                                     │  HTTP push
                    ┌────────────────▼─────────────────────────────┐
                    │              Loki :3100                       │
                    │   object/filesystem log store                 │
                    └────────────────┬─────────────────────────────┘
                                     │ query (LogQL)
                    ┌────────────────▼─────────────────────────────┐
                    │   Grafana (same instance, Loki datasource)    │
                    └──────────────────────────────────────────────┘
```

**Design principles used here:**
- Prometheus runs on a dedicated monitoring VM (not on the nodes it monitors).
- Node Exporter runs on every target host. It must never be exposed to the internet — firewall port 9100 to the Prometheus server only.
- Grafana sits behind a reverse proxy (nginx with TLS). Never expose Grafana's port 3000 directly.
- Loki stores logs locally for small setups; for production at scale, use S3-compatible object storage.
- Alertmanager runs on the same VM as Prometheus. For HA, run two Alertmanager instances in a mesh.

---

## Node Exporter

### Why Node Exporter and not something else?

Node Exporter is the canonical way to expose Linux host metrics to Prometheus. It is a single static
binary with no runtime dependencies. It exposes ~1000 metrics covering CPU, memory, disk, network,
filesystem, systemd unit states, hardware sensors, and more. Don't use the Prometheus pushgateway
for host metrics — it is designed for batch jobs, not long-running services.

### Download and install

Always download from the official GitHub releases page. Never use a distro package for Node Exporter
because distros lag behind on versions and may patch the binary in unexpected ways.

```bash
# Set the version you want — check https://github.com/prometheus/node_exporter/releases
NODE_EXPORTER_VERSION="1.8.2"
ARCH="linux-amd64"

cd /tmp
wget "https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.${ARCH}.tar.gz"

# Verify the checksum — ALWAYS do this
wget "https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/sha256sums.txt"
sha256sum --check --ignore-missing sha256sums.txt

# Extract and install
tar -xzf "node_exporter-${NODE_EXPORTER_VERSION}.${ARCH}.tar.gz"
sudo cp "node_exporter-${NODE_EXPORTER_VERSION}.${ARCH}/node_exporter" /usr/local/bin/
sudo chown root:root /usr/local/bin/node_exporter
sudo chmod 755 /usr/local/bin/node_exporter
```

### Create a dedicated system user

Never run Node Exporter as root. Create a locked system account:

```bash
sudo useradd \
  --system \
  --no-create-home \
  --shell /bin/false \
  node_exporter
```

### systemd unit file

Create `/etc/systemd/system/node_exporter.service`:

```ini
[Unit]
Description=Prometheus Node Exporter
Documentation=https://prometheus.io/docs/guides/node-exporter/
After=network-online.target
Wants=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s

# The collectors to enable. These are the most useful ones for a general-purpose Linux server.
# Disable collectors you don't need to reduce cardinality and scrape time.
ExecStart=/usr/local/bin/node_exporter \
  --web.listen-address=":9100" \
  --web.telemetry-path="/metrics" \
  --collector.systemd \
  --collector.processes \
  --collector.filesystem.mount-points-exclude="^/(sys|proc|dev|host|etc|rootfs/var/lib/docker/containers|rootfs/var/lib/docker/overlay2|rootfs/run/docker/netns|rootfs/var/lib/docker/aufs)($$|/)" \
  --collector.filesystem.fs-types-exclude="^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs)$$"

# Security hardening — these are important
NoNewPrivileges=yes
ProtectHome=yes
ProtectSystem=strict
PrivateTmp=yes
PrivateDevices=yes
ProtectHostname=yes
ProtectClock=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectKernelLogs=yes
ProtectControlGroups=yes
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
RestrictNamespaces=yes
LockPersonality=yes
MemoryDenyWriteExecute=yes
RestrictRealtime=yes
RestrictSUIDSGID=yes
RemoveIPC=yes

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter
```

### Verify the metrics endpoint

```bash
# Quick sanity check — you should see thousands of lines of metrics
curl -s http://localhost:9100/metrics | head -50

# Check a specific metric
curl -s http://localhost:9100/metrics | grep '^node_cpu_seconds_total' | head -10

# Confirm it is listening on the right port
ss -tlnp | grep 9100
```

Expected output snippet:
```
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 12345.67
node_cpu_seconds_total{cpu="0",mode="iowait"} 23.4
node_cpu_seconds_total{cpu="0",mode="irq"} 0
node_cpu_seconds_total{cpu="0",mode="nice"} 0.01
node_cpu_seconds_total{cpu="0",mode="softirq"} 12.3
node_cpu_seconds_total{cpu="0",mode="steal"} 0
node_cpu_seconds_total{cpu="0",mode="system"} 234.5
node_cpu_seconds_total{cpu="0",mode="user"} 678.9
```

### Firewall rule (iptables / nftables)

```bash
# Allow Prometheus server (192.168.1.10) to scrape this node
sudo iptables -A INPUT -p tcp --dport 9100 -s 192.168.1.10 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 9100 -j DROP

# Make it persistent
sudo netfilter-persistent save
```

---

## Prometheus Configuration

### Installation

```bash
PROMETHEUS_VERSION="2.51.2"
ARCH="linux-amd64"

cd /tmp
wget "https://github.com/prometheus/prometheus/releases/download/v${PROMETHEUS_VERSION}/prometheus-${PROMETHEUS_VERSION}.${ARCH}.tar.gz"
wget "https://github.com/prometheus/prometheus/releases/download/v${PROMETHEUS_VERSION}/sha256sums.txt"
sha256sum --check --ignore-missing sha256sums.txt

tar -xzf "prometheus-${PROMETHEUS_VERSION}.${ARCH}.tar.gz"
sudo cp "prometheus-${PROMETHEUS_VERSION}.${ARCH}/prometheus" /usr/local/bin/
sudo cp "prometheus-${PROMETHEUS_VERSION}.${ARCH}/promtool" /usr/local/bin/
sudo chown root:root /usr/local/bin/prometheus /usr/local/bin/promtool

# Create directories
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo useradd --system --no-create-home --shell /bin/false prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
sudo chown -R prometheus:prometheus /etc/prometheus
```

### Full annotated prometheus.yml

Save to `/etc/prometheus/prometheus.yml`:

```yaml
# /etc/prometheus/prometheus.yml
# Validate this file before restarting: promtool check config /etc/prometheus/prometheus.yml

# ============================================================
# GLOBAL SECTION
# These defaults apply to all scrape jobs unless overridden.
# ============================================================
global:
  # How frequently to scrape targets. 15s is the sweet spot for most infrastructure.
  # 30s is fine for non-critical services. Never go below 10s unless you have a very
  # specific need — it increases storage and CPU significantly.
  scrape_interval: 15s

  # How frequently to evaluate alerting rules. Match scrape_interval unless you have
  # a reason (e.g., expensive rule evaluation on a slow machine).
  evaluation_interval: 15s

  # Timeout for scraping a single target. If a target takes longer than this to respond,
  # it is marked as down. Must be <= scrape_interval.
  scrape_timeout: 10s

  # External labels are attached to all time series AND to all alerts fired from this
  # Prometheus instance. Essential for multi-cluster setups so you know which
  # Prometheus generated a given alert or series.
  external_labels:
    cluster: "prod-eu-west-1"
    environment: "production"
    region: "eu-west-1"
    # Add a unique identifier for this Prometheus instance if running multiple
    prometheus_instance: "prom-01"

# ============================================================
# RULE FILES
# Alerting rules and recording rules. Keep them in separate files
# by concern — don't stuff everything into one file.
# ============================================================
rule_files:
  - "/etc/prometheus/rules/node_alerts.yml"
  - "/etc/prometheus/rules/app_alerts.yml"
  - "/etc/prometheus/rules/recording_rules.yml"
  # Glob patterns work: - "/etc/prometheus/rules/*.yml"

# ============================================================
# ALERTING
# Where to send fired alerts. Prometheus fires alerts to Alertmanager;
# it does NOT send emails/Slack directly.
# ============================================================
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "localhost:9093"   # Alertmanager on same host
            # - "alertmanager-02:9093"  # Second instance for HA
      # How long Prometheus will wait for Alertmanager to acknowledge an alert
      timeout: 10s
      # Path prefix if Alertmanager is behind a reverse proxy
      # path_prefix: /alertmanager
      # TLS config if Alertmanager has TLS enabled
      # tls_config:
      #   ca_file: /etc/prometheus/certs/ca.crt

# ============================================================
# SCRAPE CONFIGS
# Each job_name is a logical grouping of targets.
# Labels added here become permanent labels on the time series.
# ============================================================
scrape_configs:

  # ----------------------------------------------------------
  # Prometheus itself. Always scrape yourself — essential for
  # monitoring Prometheus's own health (memory, TSDB stats, etc.)
  # ----------------------------------------------------------
  - job_name: "prometheus"
    # Override the global scrape_interval for this specific job
    scrape_interval: 30s
    static_configs:
      - targets:
          - "localhost:9090"
        labels:
          host: "prom-server-01"

  # ----------------------------------------------------------
  # Node Exporter — all Linux hosts
  # ----------------------------------------------------------
  - job_name: "node_exporter"
    scrape_interval: 15s
    # The default metrics path is /metrics — only specify if different
    metrics_path: /metrics
    # scheme defaults to http. Change to https if node_exporter is TLS-enabled.
    scheme: http
    static_configs:
      - targets:
          - "10.0.1.10:9100"
          - "10.0.1.11:9100"
          - "10.0.1.12:9100"
        labels:
          # These labels are added to every metric from these targets.
          # Use them to group by role, rack, region, etc.
          role: "web"
          datacenter: "eu-west-1a"
      - targets:
          - "10.0.2.10:9100"
          - "10.0.2.11:9100"
        labels:
          role: "database"
          datacenter: "eu-west-1b"

    # relabel_configs run BEFORE scraping. They manipulate the target labels.
    # This is the most powerful and most misunderstood feature of Prometheus.
    relabel_configs:
      # Replace the instance label (which defaults to host:port) with just the hostname.
      # This makes dashboards cleaner when you have many ports.
      - source_labels: [__address__]
        regex: "([^:]+)(:[0-9]+)?"
        replacement: "$1"
        target_label: instance

      # Drop any target whose job label is "ignore" — useful for dynamic configs
      # where you temporarily want to stop scraping something.
      # - source_labels: [job]
      #   regex: "ignore"
      #   action: drop

    # metric_relabel_configs run AFTER scraping — they filter/transform the metrics
    # returned by the target. Use sparingly — it's better to fix the exporter.
    metric_relabel_configs:
      # Drop high-cardinality metrics you don't need. This saves significant storage.
      # Example: drop per-cpu idle metrics if you only care about aggregate CPU.
      # - source_labels: [__name__, mode]
      #   regex: "node_cpu_seconds_total;idle"
      #   action: drop

  # ----------------------------------------------------------
  # File-based service discovery
  # More scalable than static_configs for large environments.
  # Prometheus watches this file for changes and updates targets dynamically.
  # ----------------------------------------------------------
  # - job_name: "node_exporter_dynamic"
  #   file_sd_configs:
  #     - files:
  #         - "/etc/prometheus/targets/nodes/*.json"
  #       refresh_interval: 5m

  # ----------------------------------------------------------
  # Alertmanager — scrape its own metrics so you can monitor
  # the health of your alerting pipeline.
  # ----------------------------------------------------------
  - job_name: "alertmanager"
    scrape_interval: 30s
    static_configs:
      - targets:
          - "localhost:9093"

  # ----------------------------------------------------------
  # Grafana — scrape Grafana's built-in metrics endpoint
  # ----------------------------------------------------------
  - job_name: "grafana"
    scrape_interval: 30s
    static_configs:
      - targets:
          - "localhost:3000"

  # ----------------------------------------------------------
  # Example: Application with custom metrics
  # Your app exposes /metrics on port 8080 via client library
  # ----------------------------------------------------------
  - job_name: "my_application"
    scrape_interval: 15s
    metrics_path: /metrics
    static_configs:
      - targets:
          - "10.0.1.10:8080"
          - "10.0.1.11:8080"
        labels:
          app: "my-app"
          version: "2.3.1"
    relabel_configs:
      - source_labels: [__address__]
        regex: "([^:]+):[0-9]+"
        replacement: "$1"
        target_label: instance

# ============================================================
# REMOTE WRITE (optional)
# Send a copy of all metrics to a long-term storage backend
# (Thanos, Cortex, Mimir, Grafana Cloud, etc.)
# Only enable if you have a specific need — it adds overhead.
# ============================================================
# remote_write:
#   - url: "https://mimir.example.com/api/v1/push"
#     basic_auth:
#       username: "prometheus"
#       password_file: "/etc/prometheus/secrets/mimir_password"
#     # Only send specific metrics to reduce egress cost
#     write_relabel_configs:
#       - source_labels: [__name__]
#         regex: "node_.*|up|scrape_.*"
#         action: keep
#     queue_config:
#       # Tune these based on your network and remote write endpoint capacity
#       capacity: 10000
#       max_shards: 30
#       max_samples_per_send: 5000
#       batch_send_deadline: 5s
```

### systemd unit file for Prometheus

Save to `/etc/systemd/system/prometheus.service`:

```ini
[Unit]
Description=Prometheus Monitoring System
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target
Wants=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s

ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --storage.tsdb.retention.time=30d \
  --storage.tsdb.retention.size=50GB \
  --web.console.libraries=/usr/local/share/prometheus/console_libraries \
  --web.console.templates=/usr/local/share/prometheus/consoles \
  --web.listen-address=":9090" \
  --web.enable-lifecycle \
  --web.enable-admin-api \
  --log.level=info \
  --log.format=json

# --web.enable-lifecycle allows config reload via POST /-/reload
# This avoids needing to restart the service on config changes.
# IMPORTANT: Do NOT expose port 9090 to the internet if admin-api is enabled.

# Allow Prometheus to send signals to itself for reload
ExecReload=/bin/kill -HUP $MAINPID

NoNewPrivileges=yes
ProtectHome=yes
ProtectSystem=strict
ReadWritePaths=/var/lib/prometheus
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus

# Reload config without restart (requires --web.enable-lifecycle flag)
curl -X POST http://localhost:9090/-/reload

# Validate config before reloading
promtool check config /etc/prometheus/prometheus.yml
```

### Key flags explained

| Flag | Purpose | Recommendation |
|------|---------|----------------|
| `--storage.tsdb.retention.time` | Delete data older than this | 30d for most setups; use remote_write for longer |
| `--storage.tsdb.retention.size` | Max disk usage before oldest data is dropped | Set to ~80% of your disk size |
| `--storage.tsdb.path` | Where TSDB data lives | A dedicated disk/volume — not the OS disk |
| `--web.enable-lifecycle` | Enable `/-/reload` and `/-/quit` HTTP endpoints | Enable — essential for automation |
| `--web.enable-admin-api` | Enable `/-/admin` endpoints (delete series, etc.) | Enable only if needed; it can delete data |

---

## Alert Rules

### Directory structure

```
/etc/prometheus/rules/
├── node_alerts.yml      # Host-level alerts (CPU, memory, disk, etc.)
├── app_alerts.yml       # Application-specific alerts
└── recording_rules.yml  # Pre-computed aggregations for dashboard performance
```

### `/etc/prometheus/rules/node_alerts.yml`

```yaml
groups:
  - name: node_alerts
    # How often to evaluate these rules. Defaults to global evaluation_interval.
    interval: 15s
    rules:

      # ==============================================================
      # RULE 1: Instance Down
      # The simplest and most critical alert. If we can't scrape a target,
      # something is wrong — the host is down, node_exporter crashed,
      # or there's a network partition.
      #
      # The "up" metric is synthetic: Prometheus sets it to 1 if the scrape
      # succeeded and 0 if it failed. It always exists for every target.
      #
      # "for: 5m" means the condition must be true continuously for 5 minutes
      # before the alert fires. This prevents flapping on transient failures.
      # Do NOT set "for" to 0 for this alert — a single failed scrape can
      # happen due to timeout jitter.
      # ==============================================================
      - alert: InstanceDown
        expr: up == 0
        for: 5m
        labels:
          severity: critical
          team: infrastructure
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: >
            The target {{ $labels.instance }} (job={{ $labels.job }}) has been
            unreachable for more than 5 minutes. Check the host, the exporter,
            and network connectivity.
          runbook_url: "https://wiki.example.com/runbooks/instance-down"

      # ==============================================================
      # RULE 2: High CPU Usage
      # CPU utilization > 80% sustained for 5 minutes.
      #
      # How this PromQL works:
      # - rate(node_cpu_seconds_total{mode="idle"}[5m]) gives the per-second
      #   rate of CPU time spent idle over the last 5 minutes, per CPU core.
      # - avg by(instance) averages across all cores for a given host.
      # - Multiply by 100 to convert to percentage.
      # - Subtract from 100 to get "busy" percentage instead of "idle".
      #
      # Why avg instead of sum?
      # sum would give you total idle seconds across all cores. A 4-core host
      # would have sum of idle ≈ 4.0 when fully idle. avg normalizes to
      # 0.0-1.0 regardless of core count, which is what you want.
      #
      # Why mode="idle" instead of summing non-idle modes?
      # Idle is the complement of everything else and has a single series,
      # making the query simpler and less error-prone. Equivalent to:
      # 100 - sum(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance) /
      #        count(node_cpu_seconds_total{mode="idle"}) by (instance) * 100
      # ==============================================================
      - alert: HighCPU
        expr: >
          100 - (
            avg by(instance) (
              rate(node_cpu_seconds_total{mode="idle"}[5m])
            ) * 100
          ) > 80
        for: 5m
        labels:
          severity: warning
          team: infrastructure
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: >
            CPU usage on {{ $labels.instance }} is {{ printf "%.1f" $value }}%,
            which has exceeded 80% for more than 5 minutes.
            Check for runaway processes: ps aux --sort=-%cpu | head -20
          runbook_url: "https://wiki.example.com/runbooks/high-cpu"

      # Critical threshold — page someone if CPU is pegged at 95%
      - alert: CriticalCPU
        expr: >
          100 - (
            avg by(instance) (
              rate(node_cpu_seconds_total{mode="idle"}[5m])
            ) * 100
          ) > 95
        for: 2m
        labels:
          severity: critical
          team: infrastructure
        annotations:
          summary: "Critical CPU usage on {{ $labels.instance }}"
          description: >
            CPU usage on {{ $labels.instance }} is {{ printf "%.1f" $value }}%,
            critically high for 2+ minutes. Immediate investigation required.

      # ==============================================================
      # RULE 3: High Memory Usage
      # Memory utilization > 85% sustained for 5 minutes.
      #
      # How this PromQL works:
      # - node_memory_MemAvailable_bytes is what the kernel reports as actually
      #   available for new allocations (includes reclaimable cache/buffers).
      #   This is the RIGHT metric — not MemFree.
      # - node_memory_MemTotal_bytes is total physical RAM.
      # - (1 - available/total) gives the fraction in use.
      # - Multiply by 100 for percentage.
      #
      # Why MemAvailable and not MemFree?
      # Linux uses free memory for caching — MemFree is typically very low
      # even on a healthy system. MemAvailable accounts for reclaimable cache
      # and is what "free -m" shows in the "available" column. Alerting on
      # MemFree will give you constant false alarms.
      # ==============================================================
      - alert: HighMemory
        expr: >
          (
            1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
          ) * 100 > 85
        for: 5m
        labels:
          severity: warning
          team: infrastructure
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: >
            Memory usage on {{ $labels.instance }} is {{ printf "%.1f" $value }}%
            ({{ $value | humanize }}% used).
            Available: check `free -h` and `smem -r` for top consumers.
          runbook_url: "https://wiki.example.com/runbooks/high-memory"

      # ==============================================================
      # RULE 4: Disk Almost Full
      # Filesystem usage > 85% on any mounted filesystem.
      #
      # How this PromQL works:
      # - node_filesystem_avail_bytes is available space (for non-root users).
      # - node_filesystem_size_bytes is total filesystem size.
      # - (1 - avail/size) gives fraction used.
      # - Multiply by 100 for percentage.
      #
      # The fstype!="" filter drops pseudo-filesystems (tmpfs, devtmpfs, etc.)
      # that have 0 size and would cause division-by-zero or nonsensical alerts.
      # The mountpoint!="" filter drops some edge cases.
      #
      # Opinionated addition: filter out loop devices (snap packages on Ubuntu)
      # which are always 100% full by design. They clutter your alerts.
      # ==============================================================
      - alert: DiskAlmostFull
        expr: >
          (
            1 - node_filesystem_avail_bytes{
              fstype!="",
              fstype!~"tmpfs|squashfs|devtmpfs|overlay",
              mountpoint!=""
            } / node_filesystem_size_bytes{
              fstype!="",
              fstype!~"tmpfs|squashfs|devtmpfs|overlay",
              mountpoint!=""
            }
          ) * 100 > 85
        for: 5m
        labels:
          severity: warning
          team: infrastructure
        annotations:
          summary: "Disk almost full on {{ $labels.instance }}:{{ $labels.mountpoint }}"
          description: >
            Filesystem {{ $labels.mountpoint }} on {{ $labels.instance }} is
            {{ printf "%.1f" $value }}% full. Take action before it hits 100% —
            find large files: du -sh /* 2>/dev/null | sort -rh | head -20
          runbook_url: "https://wiki.example.com/runbooks/disk-full"

      - alert: DiskFull
        expr: >
          (
            1 - node_filesystem_avail_bytes{
              fstype!="",
              fstype!~"tmpfs|squashfs|devtmpfs|overlay",
              mountpoint!=""
            } / node_filesystem_size_bytes{
              fstype!="",
              fstype!~"tmpfs|squashfs|devtmpfs|overlay",
              mountpoint!=""
            }
          ) * 100 > 95
        for: 1m
        labels:
          severity: critical
          team: infrastructure
        annotations:
          summary: "Disk critically full on {{ $labels.instance }}:{{ $labels.mountpoint }}"
          description: >
            CRITICAL: Filesystem {{ $labels.mountpoint }} on {{ $labels.instance }}
            is {{ printf "%.1f" $value }}% full. Services will start failing imminently.

      # ==============================================================
      # RULE 5: High System Load
      # 15-minute load average exceeds 2x the number of CPU cores.
      #
      # How this PromQL works:
      # - node_load15 is the 15-minute load average (equivalent to `uptime`).
      # - count by(instance)(node_cpu_seconds_total{mode="idle"}) counts the
      #   number of CPU cores per instance. Each core has one "idle" series.
      # - on(instance) scopes the division to match by instance label.
      # - Dividing load by core count gives "load per core".
      # - A value > 1 means more work queued than cores available.
      # - We alert at > 2 to avoid noise on transient spikes.
      #
      # Why load15 instead of load1 or load5?
      # Load1 is too noisy — normal bursts trigger it constantly.
      # Load15 reflects sustained pressure and is actionable.
      # A system with load15 > 2x cores has been struggling for 15+ minutes.
      #
      # Note: on(instance) is a vector matching modifier. Without it, Prometheus
      # would try to match all label dimensions and likely return no results.
      # ==============================================================
      - alert: HighLoad
        expr: >
          node_load15 / on(instance) group_left()
          count by(instance)(node_cpu_seconds_total{mode="idle"}) > 2
        for: 15m
        labels:
          severity: warning
          team: infrastructure
        annotations:
          summary: "High system load on {{ $labels.instance }}"
          description: >
            System load on {{ $labels.instance }} is {{ printf "%.2f" $value }}x
            the number of CPU cores, sustained for 15+ minutes.
            Run `top` or `htop` to identify the culprit processes.
          runbook_url: "https://wiki.example.com/runbooks/high-load"
```

### Validate and reload

```bash
# Always validate before applying
promtool check rules /etc/prometheus/rules/node_alerts.yml

# Reload Prometheus config (no restart required)
curl -X POST http://localhost:9090/-/reload

# Check that the alerts appear in the UI
curl -s http://localhost:9090/api/v1/rules | python3 -m json.tool | head -100
```

### Recording rules (for dashboard performance)

Heavy PromQL queries in dashboards cause high CPU load on Prometheus. Pre-compute them with recording
rules. These run on the evaluation_interval and store results as new time series.

```yaml
# /etc/prometheus/rules/recording_rules.yml
groups:
  - name: node_recording_rules
    interval: 60s
    rules:
      # Pre-compute CPU usage per instance — used by dashboards
      - record: instance:node_cpu_utilisation:rate5m
        expr: >
          100 - (
            avg by(instance) (
              rate(node_cpu_seconds_total{mode="idle"}[5m])
            ) * 100
          )

      # Pre-compute memory usage per instance
      - record: instance:node_memory_utilisation:ratio
        expr: >
          1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

      # Pre-compute disk usage per instance and mountpoint
      - record: instance_mountpoint:node_filesystem_utilisation:ratio
        expr: >
          1 - node_filesystem_avail_bytes{fstype!~"tmpfs|squashfs|devtmpfs|overlay"}
              / node_filesystem_size_bytes{fstype!~"tmpfs|squashfs|devtmpfs|overlay"}
```

---

## Alertmanager

### Installation

```bash
ALERTMANAGER_VERSION="0.27.0"
ARCH="linux-amd64"

cd /tmp
wget "https://github.com/prometheus/alertmanager/releases/download/v${ALERTMANAGER_VERSION}/alertmanager-${ALERTMANAGER_VERSION}.${ARCH}.tar.gz"
wget "https://github.com/prometheus/alertmanager/releases/download/v${ALERTMANAGER_VERSION}/sha256sums.txt"
sha256sum --check --ignore-missing sha256sums.txt

tar -xzf "alertmanager-${ALERTMANAGER_VERSION}.${ARCH}.tar.gz"
sudo cp "alertmanager-${ALERTMANAGER_VERSION}.${ARCH}/alertmanager" /usr/local/bin/
sudo cp "alertmanager-${ALERTMANAGER_VERSION}.${ARCH}/amtool" /usr/local/bin/

sudo useradd --system --no-create-home --shell /bin/false alertmanager
sudo mkdir -p /etc/alertmanager /var/lib/alertmanager
sudo chown alertmanager:alertmanager /var/lib/alertmanager /etc/alertmanager
```

### Full annotated alertmanager.yml

Save to `/etc/alertmanager/alertmanager.yml`:

```yaml
# /etc/alertmanager/alertmanager.yml
# Validate: amtool check-config /etc/alertmanager/alertmanager.yml

# ============================================================
# GLOBAL SECTION
# Defaults for all receivers. Individual receivers can override these.
# ============================================================
global:
  # How long an alert remains resolved before Alertmanager forgets it.
  # After this time, if the same alert fires again, it will re-notify.
  resolve_timeout: 5m

  # SMTP settings for email notifications
  smtp_smarthost: "smtp.gmail.com:587"
  smtp_from: "alertmanager@example.com"
  smtp_auth_username: "alertmanager@example.com"
  # Use a file reference for secrets — never hardcode passwords in YAML
  smtp_auth_password_file: "/etc/alertmanager/secrets/smtp_password"
  smtp_require_tls: true

  # Slack webhook URL — create one in Slack App settings
  # slack_api_url: set per-receiver or use slack_api_url_file
  slack_api_url_file: "/etc/alertmanager/secrets/slack_webhook_url"

  # PagerDuty integration URL (usually don't change this)
  pagerduty_url: "https://events.pagerduty.com/v2/enqueue"

# ============================================================
# TEMPLATES
# Custom notification templates for cleaner messages.
# ============================================================
templates:
  - "/etc/alertmanager/templates/*.tmpl"

# ============================================================
# ROUTE TREE
# This is the heart of Alertmanager. Every firing alert walks
# this tree from the root until it finds a matching route.
# If no child route matches, the root route is used.
#
# The routing logic:
# 1. All alerts arrive at the root route.
# 2. Alertmanager checks child routes in order.
# 3. First matching child route handles the alert (unless continue: true).
# 4. If no child matches, root route handles it.
#
# group_by: Alerts with the same values for these labels are grouped
#           into a single notification. This prevents alert storms.
# group_wait: Wait this long after the first alert in a group fires
#             before sending the first notification. Allows other alerts
#             in the same group to arrive first (batching).
# group_interval: After the first notification, wait this long before
#                 sending another notification for the same group (with
#                 new or resolved alerts).
# repeat_interval: If nothing changes, re-notify after this interval.
#                  Set high enough to avoid spam, low enough to not miss
#                  ongoing issues.
# ============================================================
route:
  # Default receiver for alerts that don't match any child route
  receiver: "team-email"

  # Group alerts by these labels. All alerts with the same alertname,
  # cluster, and severity will be batched into one notification.
  group_by: ["alertname", "cluster", "severity"]

  # Wait up to 30s for more alerts in the same group before sending
  group_wait: 30s

  # After an initial notification, wait 5m before sending new notifications
  # for the same group (when new alerts fire or existing ones resolve)
  group_interval: 5m

  # If the alert is still firing and nothing has changed, re-notify after 4h
  repeat_interval: 4h

  routes:
    # ----------------------------------------------------------
    # Critical severity → PagerDuty (wake someone up)
    # ----------------------------------------------------------
    - match:
        severity: critical
      receiver: "pagerduty-critical"
      # Also send to Slack for visibility
      continue: true   # continue: true means keep checking other routes too
      group_wait: 10s   # Don't wait long for critical alerts
      repeat_interval: 1h

    # ----------------------------------------------------------
    # Slack for all infrastructure alerts
    # ----------------------------------------------------------
    - match:
        team: infrastructure
      receiver: "slack-infrastructure"
      group_by: ["alertname", "instance"]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 8h

    # ----------------------------------------------------------
    # Watchdog / Deadman's switch alert
    # This is a special alert that should ALWAYS be firing.
    # If it stops firing, your entire alerting pipeline is broken.
    # Route it to a receiver that expects to hear from it (e.g., healthchecks.io)
    # ----------------------------------------------------------
    - match:
        alertname: Watchdog
      receiver: "null"   # Suppress it from normal channels
      # In production, route Watchdog to a dead man's switch service

    # ----------------------------------------------------------
    # Inhibit email spam during maintenance
    # If you set an alert label maintenance="true", suppress emails
    # ----------------------------------------------------------
    - match:
        maintenance: "true"
      receiver: "null"

# ============================================================
# RECEIVERS
# Each receiver defines one or more notification channels.
# A receiver can have multiple configs of different types.
# ============================================================
receivers:

  # Null receiver — silently discards alerts
  - name: "null"

  # ----------------------------------------------------------
  # Email receiver
  # ----------------------------------------------------------
  - name: "team-email"
    email_configs:
      - to: "ops-team@example.com"
        send_resolved: true    # Also notify when alert resolves
        headers:
          Subject: "[{{ .Status | toUpper }}{{ if eq .Status \"firing\" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.SortedPairs.Values | join \" \" }}"
        # HTML body using Go template syntax
        html: |
          {{ range .Alerts }}
          <b>Alert:</b> {{ .Annotations.summary }}<br>
          <b>Severity:</b> {{ .Labels.severity }}<br>
          <b>Instance:</b> {{ .Labels.instance }}<br>
          <b>Description:</b> {{ .Annotations.description }}<br>
          <b>Started:</b> {{ .StartsAt }}<br>
          {{ if .Annotations.runbook_url }}<b>Runbook:</b> <a href="{{ .Annotations.runbook_url }}">{{ .Annotations.runbook_url }}</a>{{ end }}
          <hr>
          {{ end }}

  # ----------------------------------------------------------
  # Slack receiver
  # ----------------------------------------------------------
  - name: "slack-infrastructure"
    slack_configs:
      - channel: "#alerts-infrastructure"
        send_resolved: true
        icon_emoji: ":fire:"
        username: "Alertmanager"
        # Color-coded attachments based on severity
        color: '{{ if eq .Status "firing" }}{{ if eq (index .GroupLabels "severity") "critical" }}danger{{ else }}warning{{ end }}{{ else }}good{{ end }}'
        title: '{{ if eq .Status "firing" }}FIRING{{ else }}RESOLVED{{ end }}: {{ .GroupLabels.alertname }}'
        title_link: "http://grafana.example.com/alerting"
        text: |
          {{ range .Alerts }}
          *{{ .Annotations.summary }}*
          {{ .Annotations.description }}
          Instance: `{{ .Labels.instance }}`
          {{ if .Annotations.runbook_url }}Runbook: {{ .Annotations.runbook_url }}{{ end }}
          {{ end }}
        # Link to Prometheus for the triggering query
        actions:
          - type: button
            text: "View in Prometheus"
            url: "http://prometheus.example.com:9090/alerts"
          - type: button
            text: "Silence"
            url: "http://alertmanager.example.com:9093/#/silences/new"

  # ----------------------------------------------------------
  # PagerDuty receiver
  # ----------------------------------------------------------
  - name: "pagerduty-critical"
    pagerduty_configs:
      - routing_key_file: "/etc/alertmanager/secrets/pagerduty_routing_key"
        send_resolved: true
        severity: "{{ if eq .CommonLabels.severity \"critical\" }}critical{{ else }}warning{{ end }}"
        description: "{{ .CommonAnnotations.summary }}"
        details:
          # These become additional details in the PagerDuty incident
          instance: "{{ .CommonLabels.instance }}"
          cluster: "{{ .CommonLabels.cluster }}"
          environment: "{{ .CommonLabels.environment }}"
          description: "{{ .CommonAnnotations.description }}"
          runbook: "{{ .CommonAnnotations.runbook_url }}"
        # Link directly to Grafana dashboard
        links:
          - href: "http://grafana.example.com/d/rYdddlPWk/node-exporter-full"
            text: "Host Dashboard"

# ============================================================
# INHIBITION RULES
# Suppress (inhibit) certain alerts when other alerts are firing.
# This prevents alert storms where one root cause fires 50 alerts.
#
# How it works:
# If a "source" alert is firing, suppress any "target" alerts
# that share the same value for the "equal" labels.
#
# Example: If InstanceDown fires for host X, suppress all other
# alerts from host X — they're all caused by the same thing.
# ============================================================
inhibit_rules:

  # If a host is down, suppress all other alerts from that host.
  # There's no point in alerting about high CPU if the machine is unreachable.
  - source_matchers:
      - alertname = "InstanceDown"
    target_matchers:
      - alertname != "InstanceDown"   # Don't inhibit itself
    # Only inhibit alerts from the SAME instance
    equal:
      - instance
      - cluster

  # If a critical severity alert fires, suppress the corresponding warning.
  # Example: If DiskFull (critical) fires, suppress DiskAlmostFull (warning)
  # for the same mountpoint on the same host.
  - source_matchers:
      - severity = "critical"
    target_matchers:
      - severity = "warning"
    equal:
      - alertname
      - instance
      - cluster
```

### Alertmanager systemd unit

```ini
# /etc/systemd/system/alertmanager.service
[Unit]
Description=Prometheus Alertmanager
After=network-online.target
Wants=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
Restart=on-failure
RestartSec=5s

ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager \
  --web.listen-address=":9093" \
  --web.external-url="http://alertmanager.example.com:9093" \
  --log.level=info \
  --log.format=json

ExecReload=/bin/kill -HUP $MAINPID

NoNewPrivileges=yes
ProtectHome=yes
ProtectSystem=strict
ReadWritePaths=/var/lib/alertmanager
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now alertmanager

# Validate config
amtool check-config /etc/alertmanager/alertmanager.yml

# Reload config
curl -X POST http://localhost:9093/-/reload

# Test a receiver manually
amtool alert add alertname=TestAlert severity=warning instance=test-host \
  --alertmanager.url=http://localhost:9093
```

---

## Grafana

### Installation (Debian/Ubuntu via APT)

Always use the official Grafana APT repository. The distro packages are stale.

```bash
# Import GPG key and add repository
sudo apt-get install -y apt-transport-https software-properties-common wget gnupg
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | \
  sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install -y grafana

sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server
```

### Grafana configuration

Edit `/etc/grafana/grafana.ini` — key settings for production:

```ini
[server]
http_addr = 127.0.0.1     # Only listen on localhost; put nginx in front
http_port = 3000
domain = grafana.example.com
root_url = https://grafana.example.com/
enforce_domain = true

[security]
admin_user = admin
admin_password = change_this_immediately_and_use_a_password_manager
secret_key = generate_with_openssl_rand_hex_32    # openssl rand -hex 32
disable_gravatar = true
cookie_secure = true        # Requires HTTPS
cookie_samesite = strict

[users]
allow_sign_up = false        # Never allow self-registration in production
auto_assign_org_role = Viewer

[auth.anonymous]
enabled = false              # Disable anonymous access

[log]
mode = file
level = warn
format = json

[analytics]
reporting_enabled = false
check_for_updates = false
```

### Datasource provisioning (infrastructure as code)

Provisioned datasources are read-only in the UI — they cannot be accidentally deleted or modified.
This is the right way to manage datasources in production.

```yaml
# /etc/grafana/provisioning/datasources/prometheus.yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    uid: prometheus-main    # Stable UID — use this in dashboard JSON instead of name
    access: proxy           # Grafana server proxies requests to Prometheus (preferred)
    url: http://localhost:9090
    isDefault: true
    editable: false         # Prevent UI modification
    jsonData:
      # How long before queries time out
      timeInterval: "15s"   # Match your scrape_interval
      queryTimeout: "60s"
      httpMethod: POST      # POST is more efficient for large queries
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: tempo-main   # If you have Tempo for traces
    version: 1

  - name: Loki
    type: loki
    uid: loki-main
    access: proxy
    url: http://localhost:3100
    isDefault: false
    editable: false
    jsonData:
      maxLines: 1000
    version: 1
```

```bash
# Restart Grafana to pick up provisioned datasources
sudo systemctl restart grafana-server

# Verify datasource is available via API
curl -s -u admin:yourpassword http://localhost:3000/api/datasources | \
  python3 -m json.tool
```

### Dashboard provisioning

```yaml
# /etc/grafana/provisioning/dashboards/default.yaml
apiVersion: 1

providers:
  - name: "Infrastructure Dashboards"
    orgId: 1
    folder: "Infrastructure"
    folderUid: "infrastructure"
    type: file
    disableDeletion: true      # Prevent deletion from UI
    updateIntervalSeconds: 30  # Check for file changes every 30s
    allowUiUpdates: false      # Prevent UI modifications
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true
```

### Import dashboard via API

The Node Exporter Full dashboard (ID 1860) is the industry-standard starting point.
Never build a host metrics dashboard from scratch.

```bash
# Create the dashboards directory
sudo mkdir -p /var/lib/grafana/dashboards

# Download the Node Exporter Full dashboard JSON
curl -s "https://grafana.com/api/dashboards/1860/revisions/latest/download" \
  -o /var/lib/grafana/dashboards/node-exporter-full.json

# Fix the datasource UID to match your provisioned datasource
# The downloaded JSON uses ${DS_PROMETHEUS} as a variable — replace with your UID
sed -i 's/\${DS_PROMETHEUS}/prometheus-main/g' /var/lib/grafana/dashboards/node-exporter-full.json

# Alternatively, import via Grafana API (useful in CI/CD pipelines)
curl -X POST \
  -H "Content-Type: application/json" \
  -u admin:yourpassword \
  http://localhost:3000/api/dashboards/import \
  -d '{
    "dashboard": '"$(cat /var/lib/grafana/dashboards/node-exporter-full.json)"',
    "overwrite": true,
    "folderId": 0,
    "inputs": [
      {
        "name": "DS_PROMETHEUS",
        "type": "datasource",
        "pluginId": "prometheus",
        "value": "prometheus-main"
      }
    ]
  }'

sudo chown -R grafana:grafana /var/lib/grafana/dashboards
sudo systemctl restart grafana-server
```

### Other essential dashboards to import

| Grafana ID | Name | Purpose |
|-----------|------|---------|
| 1860 | Node Exporter Full | Comprehensive host metrics |
| 3662 | Prometheus 2.0 Overview | Monitor Prometheus itself |
| 9578 | Alertmanager | Monitor alerting pipeline |
| 13639 | Loki Logs | Log exploration and stats |
| 11835 | Grafana Internals | Monitor Grafana performance |

---

## Loki + Promtail (PLG Stack)

### Why PLG instead of ELK?

Loki does not index log content — it only indexes labels (like Prometheus). This makes it much cheaper
to operate than Elasticsearch. The trade-off: full-text search is slower because it has to scan log
lines. For most operational use cases (finding errors in a known service over a short time window),
Loki is more than adequate and costs a fraction of ELK.

The PLG stack is: **P**romtail (collector) + **L**oki (storage/query) + **G**rafana (UI).

### Loki installation

```bash
LOKI_VERSION="3.0.0"
ARCH="linux-amd64"

cd /tmp
wget "https://github.com/grafana/loki/releases/download/v${LOKI_VERSION}/loki-linux-amd64.zip"
wget "https://github.com/grafana/loki/releases/download/v${LOKI_VERSION}/loki-linux-amd64.zip.sha256"
sha256sum --check loki-linux-amd64.zip.sha256

unzip loki-linux-amd64.zip
sudo cp loki-linux-amd64 /usr/local/bin/loki
sudo chmod 755 /usr/local/bin/loki

sudo useradd --system --no-create-home --shell /bin/false loki
sudo mkdir -p /etc/loki /var/lib/loki/chunks /var/lib/loki/index /var/lib/loki/wal
sudo chown -R loki:loki /var/lib/loki /etc/loki
```

### `/etc/loki/loki-local-config.yaml`

```yaml
# /etc/loki/loki-local-config.yaml
# This is a single-binary, local filesystem configuration.
# For production at scale, replace filesystem storage with S3/GCS and
# use a distributed deployment (microservices mode).

auth_enabled: false   # Set to true in multi-tenant environments

server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  log_level: warn
  log_format: json

# ============================================================
# COMMON CONFIG — shared settings for all components
# ============================================================
common:
  instance_addr: 127.0.0.1
  path_prefix: /var/lib/loki
  storage:
    filesystem:
      chunks_directory: /var/lib/loki/chunks
      rules_directory: /var/lib/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

# ============================================================
# QUERY RANGE — caching and limits for query performance
# ============================================================
query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

# ============================================================
# SCHEMA CONFIG
# Defines how and where logs are stored over time.
# The schema version affects index and chunk format.
# NEVER change a period's config after logs have been written to it —
# create a new period with a future from date instead.
# ============================================================
schema_config:
  configs:
    - from: "2024-01-01"
      store: tsdb          # TSDB index — more efficient than BoltDB
      object_store: filesystem
      schema: v13          # Latest schema version — use this for new deployments
      index:
        prefix: loki_index_
        period: 24h        # Create a new index every 24 hours

# ============================================================
# STORAGE CONFIG
# ============================================================
storage_config:
  tsdb_shipper:
    active_index_directory: /var/lib/loki/index
    cache_location: /var/lib/loki/cache
    cache_ttl: 24h
  filesystem:
    directory: /var/lib/loki/chunks

# ============================================================
# COMPACTOR — runs periodically to merge and compact chunks
# ============================================================
compactor:
  working_directory: /var/lib/loki/compactor
  compaction_interval: 10m
  retention_enabled: true
  retention_delete_delay: 2h
  retention_delete_worker_count: 150
  delete_request_store: filesystem

# ============================================================
# LIMITS / RETENTION
# ============================================================
limits_config:
  # Global retention — delete logs older than this
  retention_period: 30d

  # Per-stream rate limiting — tune based on your log volume
  ingestion_rate_mb: 16
  ingestion_burst_size_mb: 32

  # Maximum number of label names per log stream
  max_label_names_per_series: 15

  # Maximum query time range
  max_query_length: 721h   # 30 days

  # Reject logs from the future (clock skew protection)
  reject_old_samples: true
  reject_old_samples_max_age: 168h   # 7 days

  # Split large queries into smaller pieces for better performance
  split_queries_by_interval: 15m

# ============================================================
# RULER — for LogQL alerting rules (optional)
# ============================================================
ruler:
  alertmanager_url: http://localhost:9093

# ============================================================
# ANALYTICS — disable telemetry
# ============================================================
analytics:
  reporting_enabled: false
```

### Loki systemd unit

```ini
# /etc/systemd/system/loki.service
[Unit]
Description=Loki Log Aggregation System
After=network-online.target
Wants=network-online.target

[Service]
User=loki
Group=loki
Type=simple
Restart=on-failure
RestartSec=5s

ExecStart=/usr/local/bin/loki \
  -config.file=/etc/loki/loki-local-config.yaml

NoNewPrivileges=yes
ProtectHome=yes
ProtectSystem=strict
ReadWritePaths=/var/lib/loki
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
```

### Promtail installation

Promtail runs on every host that generates logs (same hosts as Node Exporter).

```bash
PROMTAIL_VERSION="3.0.0"   # Must match Loki version exactly

cd /tmp
wget "https://github.com/grafana/loki/releases/download/v${PROMTAIL_VERSION}/promtail-linux-amd64.zip"
unzip promtail-linux-amd64.zip
sudo cp promtail-linux-amd64 /usr/local/bin/promtail
sudo chmod 755 /usr/local/bin/promtail

sudo useradd --system --no-create-home --shell /bin/false promtail
# Promtail needs to read log files — add it to the systemd-journal group
sudo usermod -a -G systemd-journal promtail
# Add to adm group to read /var/log/* files
sudo usermod -a -G adm promtail

sudo mkdir -p /etc/promtail /var/lib/promtail
sudo chown promtail:promtail /var/lib/promtail /etc/promtail
```

### `/etc/promtail/promtail-config.yaml`

This is the most important Promtail config you'll write. It covers both systemd journal scraping
and traditional file-based log scraping.

```yaml
# /etc/promtail/promtail-config.yaml

# ============================================================
# SERVER
# Promtail exposes its own metrics and health endpoints.
# ============================================================
server:
  http_listen_port: 9080
  grpc_listen_port: 0        # Disable gRPC unless needed
  log_level: warn
  log_format: json

# ============================================================
# POSITIONS FILE
# Tracks how far Promtail has read into each log file.
# If you delete this file, Promtail will re-read all logs from the start.
# IMPORTANT: This must be on a persistent path, not /tmp.
# ============================================================
positions:
  filename: /var/lib/promtail/positions.yaml
  # Sync positions to disk every 10s. If Promtail crashes between syncs,
  # you may get duplicate log entries — this is by design (at-least-once).
  sync_period: 10s
  ignore_invalid_yaml: false

# ============================================================
# CLIENTS
# Where to send logs. Multiple clients send to multiple Loki instances.
# ============================================================
clients:
  - url: "http://loki-server:3100/loki/api/v1/push"
    # Backoff configuration for when Loki is unavailable
    backoff_config:
      min_period: 500ms
      max_period: 5m
      max_retries: 10
    # Batch logs before sending to reduce HTTP overhead
    batchwait: 1s        # Wait up to 1s to fill a batch
    batchsize: 1048576   # Max batch size: 1MB
    # External labels added to ALL log streams from this Promtail instance
    external_labels:
      host: "${HOSTNAME}"   # Populated from environment variable
      environment: "production"
      cluster: "prod-eu-west-1"
    # Timeout for sending a batch
    timeout: 10s

# ============================================================
# SCRAPE CONFIGS
# Define what logs to collect and how to label them.
# ============================================================
scrape_configs:

  # ----------------------------------------------------------
  # Source 1: systemd journal
  # This captures ALL systemd service logs in one place.
  # Perfect for getting logs from services that don't write to files
  # (most modern systemd services).
  # ----------------------------------------------------------
  - job_name: "systemd-journal"
    journal:
      # Read from system journal (not per-user journals)
      path: /var/log/journal
      # How far back to read on first start. "now" means only new entries.
      # Use a duration like "24h" to backfill on new hosts.
      since: "now"
      # Include all journal fields as labels (can cause high cardinality — be selective)
      json: false
      # Maximum age of journal entries to ship (prevents re-shipping old entries)
      max_age: 12h
      # Label the stream with the systemd unit name (most useful journal field)
      labels:
        job: "systemd-journal"
        host: "${HOSTNAME}"

    # Pipeline stages transform log entries before sending to Loki.
    # Each stage is applied in order.
    pipeline_stages:
      # Extract useful fields from the journal entry structure
      - json:
          expressions:
            # The actual log message
            message: MESSAGE
            # The systemd unit that produced the log
            unit: SYSLOG_IDENTIFIER
            # Priority: 0=emerg, 1=alert, 2=crit, 3=err, 4=warning, 5=notice, 6=info, 7=debug
            priority: PRIORITY
            # PID of the process
            pid: _PID
            # Transport type (journal, syslog, kernel, etc.)
            transport: _TRANSPORT

      # Create a label from the unit name so you can filter by service in LogQL
      - labels:
          unit:
          transport:

      # Map numeric priority to human-readable level
      - template:
          source: priority
          template: |
            {{ if eq .Value "0" }}emerg{{ else if eq .Value "1" }}alert{{ else if eq .Value "2" }}critical{{ else if eq .Value "3" }}error{{ else if eq .Value "4" }}warning{{ else if eq .Value "5" }}notice{{ else if eq .Value "6" }}info{{ else }}debug{{ end }}

      # Use the mapped level as a label
      - labels:
          level: priority

      # Drop debug-level logs unless you're troubleshooting
      # This reduces log volume significantly
      - drop:
          source: level
          expression: "debug"

      # Use the journal message as the final log line
      - output:
          source: message

  # ----------------------------------------------------------
  # Source 2: Nginx access logs
  # File-based scraping for traditional log files.
  # ----------------------------------------------------------
  - job_name: "nginx-access"
    static_configs:
      - targets:
          - localhost
        labels:
          job: "nginx"
          log_type: "access"
          host: "${HOSTNAME}"
          # __path__ is a special label that tells Promtail which file(s) to tail
          __path__: "/var/log/nginx/access.log"

    pipeline_stages:
      # Parse nginx combined log format
      # 127.0.0.1 - frank [10/Oct/2000:13:55:36 -0700] "GET /apache_pb.gif HTTP/1.0" 200 2326
      - regex:
          expression: >
            ^(?P<remote_addr>[\w\.]+) - (?P<remote_user>\S+) \[(?P<time_local>[^\]]+)\]
            "(?P<method>\S+) (?P<request>[^\s"]+) (?P<protocol>[^"]+)"
            (?P<status>\d+) (?P<bytes_sent>\d+)
            "(?P<http_referer>[^"]*)" "(?P<http_user_agent>[^"]*)"

      # Promote selected parsed fields to Loki labels
      # WARNING: Be conservative with labels — high cardinality (like remote_addr)
      # will explode your Loki storage and query performance.
      - labels:
          method:
          status:

      # Add a level label based on HTTP status code
      - template:
          source: status
          template: |
            {{ if lt (int .Value) 400 }}info{{ else if lt (int .Value) 500 }}warning{{ else }}error{{ end }}

      - labels:
          level: status

      # Parse the timestamp from the log line for accurate time ordering
      - timestamp:
          source: time_local
          format: "02/Jan/2006:15:04:05 -0700"

  # ----------------------------------------------------------
  # Source 3: Application logs (JSON format)
  # Most modern applications log structured JSON — handle it properly.
  # ----------------------------------------------------------
  - job_name: "my-application"
    static_configs:
      - targets:
          - localhost
        labels:
          job: "my-app"
          host: "${HOSTNAME}"
          # Glob pattern — picks up all rotated log files too
          __path__: "/var/log/my-app/*.log"

    pipeline_stages:
      # Parse JSON log line
      # Expected format: {"level":"error","ts":1234567890.123,"msg":"something failed","trace_id":"abc123"}
      - json:
          expressions:
            level: level
            timestamp: ts
            message: msg
            trace_id: trace_id
            component: component

      # Make level and component into Loki labels for filtering
      - labels:
          level:
          component:

      # Parse Unix timestamp
      - timestamp:
          source: timestamp
          format: Unix

      # Drop routine health check log noise
      - drop:
          source: message
          expression: "health check"

      - output:
          source: message

  # ----------------------------------------------------------
  # Source 4: Auth logs — for security monitoring
  # ----------------------------------------------------------
  - job_name: "auth-logs"
    static_configs:
      - targets:
          - localhost
        labels:
          job: "auth"
          host: "${HOSTNAME}"
          __path__: "/var/log/auth.log"

    pipeline_stages:
      # Tag failed SSH authentication attempts
      - regex:
          expression: "(?P<auth_result>Failed password|Accepted password|Invalid user)"
      - labels:
          auth_result:
```

### Promtail systemd unit

```ini
# /etc/systemd/system/promtail.service
[Unit]
Description=Promtail Log Shipper
After=network-online.target
Wants=network-online.target

[Service]
User=promtail
Group=promtail
Type=simple
Restart=on-failure
RestartSec=5s

Environment=HOSTNAME=%H

ExecStart=/usr/local/bin/promtail \
  -config.file=/etc/promtail/promtail-config.yaml \
  -config.expand-env=true

# Allow reading of log files
ReadOnlyPaths=/var/log /run/log/journal

NoNewPrivileges=yes
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now promtail
sudo systemctl status promtail

# Check Promtail is shipping logs
curl -s http://localhost:9080/metrics | grep promtail_sent_entries_total
```

### LogQL — practical query examples

LogQL syntax: `{label_selector} | filter | parser | format`

**Basic log selection:**
```logql
# All nginx logs
{job="nginx"}

# All error logs from nginx
{job="nginx", level="error"}

# Logs from a specific host
{host="web-01", job="systemd-journal"}

# All SSH authentication failures
{job="auth"} |= "Failed password"

# Case-insensitive search for "error" anywhere in the line
{job="nginx"} |~ "(?i)error"

# Exclude health check noise
{job="my-app"} != "health check"

# Complex filter: errors that are NOT database connection timeouts
{job="my-app", level="error"} != "connection timeout" |= "database"
```

**Rate queries (for dashboards and alerts):**
```logql
# Log ingestion rate per minute (total)
rate({job="nginx"}[5m])

# Error rate per minute for nginx
rate({job="nginx", level="error"}[5m])

# Ratio of errors to total requests (error rate %)
sum(rate({job="nginx", status=~"5.."}[5m])) /
sum(rate({job="nginx"}[5m])) * 100

# Error count by component (bar chart)
sum by(component) (count_over_time({job="my-app", level="error"}[1h]))
```

**Label filter after parsing:**
```logql
# Parse JSON on the fly and filter by a field not indexed as a label
{job="my-app"} | json | response_time > 1000

# Filter nginx logs for slow requests (>5s) parsed from the log line
{job="nginx"} | regexp `bytes_sent=(?P<bytes>\d+)` | bytes > 100000

# Find all requests to a specific path
{job="nginx"} | logfmt | path="/api/v2/users"
```

**Metric queries (turn logs into metrics):**
```logql
# Count log lines per level over time (useful for anomaly detection)
sum by(level) (rate({job="systemd-journal"}[5m]))

# 99th percentile response time (if response_time is a label)
# (Loki doesn't support quantile over log content — use Prometheus for this)

# Alert when error rate spikes (in Loki ruler or Grafana alert)
sum(rate({job="nginx", level="error"}[5m])) > 10
```

---

## Uptime Monitoring

### Why you need external uptime monitoring

Prometheus monitors your internal systems from the inside. But if your entire monitoring VM goes
down, or if there's a network partition between your monitoring and your services, you'll have no
alerts. External uptime monitoring checks your services from outside your infrastructure, from
multiple geographic locations, and alerts through a completely separate channel (not through your
Alertmanager).

External uptime monitoring and Prometheus/Alertmanager are complementary — run both.

### UptimeRobot (free tier)

UptimeRobot free tier gives you:
- 50 monitors
- 5-minute check interval (60s on paid plans)
- HTTP/HTTPS, keyword, port, and ping checks
- Email, Slack, and webhook alerting
- 90-day log retention

**Setup via web UI (https://uptimerobot.com):**

1. Create an account.
2. Add Monitor → HTTP(s)
3. Friendly Name: "Production API"
4. URL: `https://api.example.com/health`
5. Monitoring Interval: 5 minutes (free tier limit)
6. Under "Alert Contacts": add your email. Also add a Slack webhook for team visibility.
7. Optional: "Keyword Monitor" — checks that a specific string appears in the response body.
   Use this to verify your health endpoint returns `{"status":"ok"}` and not a cached error page.

**HTTP check configuration (what to set):**

```
Monitor Type: HTTPS
URL: https://api.example.com/health
Keyword: "healthy"          ← check response body contains this word
Expected Status Code: 200   ← only 200, not 2xx
Timeout: 30 seconds
Check Interval: 5 minutes
Alert: After 1 failure (for critical endpoints — noisy but safe)
       After 3 failures (for less critical endpoints — avoids flapping)
```

**Why to use keyword monitoring, not just status code:**

A load balancer can return 200 OK even when all backends are down (returning its own error page).
A reverse proxy can return 200 with a cached response. Always check response content, not just status.

**Recommended monitors for a typical web stack:**

| Monitor | URL | Keyword |
|---------|-----|---------|
| Homepage | `https://example.com` | `Welcome` |
| API health | `https://api.example.com/health` | `ok` |
| CDN | `https://cdn.example.com/static/ping.txt` | `pong` |
| Admin panel | `https://admin.example.com` | `Dashboard` |
| Login page | `https://example.com/login` | `password` |

**UptimeRobot API — for automation:**

```bash
# Create a monitor via API (useful in IaC/Ansible)
curl -X POST "https://api.uptimerobot.com/v2/newMonitor" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "api_key=YOUR_API_KEY" \
  --data-urlencode "friendly_name=Production API" \
  --data-urlencode "url=https://api.example.com/health" \
  --data-urlencode "type=2" \       # 2 = HTTP(s)
  --data-urlencode "keyword_type=2" \   # 2 = keyword exists
  --data-urlencode "keyword_value=healthy" \
  --data-urlencode "interval=5"
```

### healthchecks.io — cron job monitoring

healthchecks.io solves a different problem: detecting when a scheduled job (cron, systemd timer,
backup script) FAILS TO RUN. Prometheus can tell you when a service is down, but it can't easily
tell you that your nightly backup didn't run. healthchecks.io uses a "dead man's switch" approach:
your job pings the service on success; if no ping arrives within the expected window, you get alerted.

**Free tier:** 20 checks, 3 team members, 6-month log retention.

**Concept:**
1. Create a check at https://healthchecks.io
2. Set the expected period (e.g., "daily at 2am") and grace period (e.g., 1 hour)
3. At the end of your job, `curl` the unique URL
4. If healthchecks.io doesn't receive the curl within period + grace, it alerts you

**Basic cron ping pattern:**

```bash
# In your cron job or backup script:
# Run the job, AND THEN ping healthchecks.io ONLY on success.
# The && means the curl only runs if the backup command succeeds.

# Example: nightly database backup
0 2 * * * /opt/scripts/backup.sh && \
  curl -fsS --retry 3 --max-time 10 \
  "https://hc-ping.com/YOUR-UNIQUE-UUID" > /dev/null 2>&1

# If you want to ping on failure too (for diagnostics):
0 2 * * * /opt/scripts/backup.sh \
  && curl -fsS "https://hc-ping.com/YOUR-UUID" \
  || curl -fsS "https://hc-ping.com/YOUR-UUID/fail"
```

**Ping with exit code capture (more robust pattern):**

```bash
#!/bin/bash
# /opt/scripts/backup-with-healthcheck.sh
set -euo pipefail

HEALTHCHECK_URL="https://hc-ping.com/YOUR-UNIQUE-UUID"

# Signal start of job (optional, but lets healthchecks.io track duration)
curl -fsS --retry 3 "${HEALTHCHECK_URL}/start" > /dev/null 2>&1 || true

# Run the actual job
/opt/scripts/backup.sh
EXIT_CODE=$?

# Ping with exit code — healthchecks.io marks it failed if code != 0
curl -fsS --retry 3 "${HEALTHCHECK_URL}/${EXIT_CODE}" > /dev/null 2>&1 || true

exit $EXIT_CODE
```

**Recommended checks to set up:**

| Check Name | Schedule | Grace | What it monitors |
|-----------|----------|-------|-----------------|
| Nightly DB backup | Daily | 2h | Database backup completion |
| SSL cert renewal | Daily | 24h | Certbot/Let's Encrypt renewal |
| Log rotation | Daily | 4h | logrotate running |
| Prometheus TSDB compaction | Every 2h | 30m | Prometheus self-health |
| Offsite backup sync | Daily | 4h | rsync to offsite storage |

**Self-hosted alternative:** The healthchecks.io software is open source. You can run it yourself
with Docker:

```bash
docker run -d \
  --name healthchecks \
  -p 8000:8000 \
  -e SECRET_KEY="$(openssl rand -hex 32)" \
  -e DB=sqlite \
  -e DB_NAME=/data/hc.sqlite3 \
  -v /var/lib/healthchecks:/data \
  healthchecks/healthchecks:latest
```

---

## Operational Runbook

### Day-to-day operations

```bash
# Check health of all monitoring components
systemctl status prometheus alertmanager grafana-server loki promtail node_exporter

# Reload Prometheus config (no restart, no data loss)
curl -X POST http://localhost:9090/-/reload

# Reload Alertmanager config
curl -X POST http://localhost:9093/-/reload

# Check current alerts
curl -s http://localhost:9090/api/v1/alerts | python3 -m json.tool
amtool alert query --alertmanager.url=http://localhost:9093

# Silence an alert for maintenance
amtool silence add alertname=InstanceDown instance=web-01 \
  --alertmanager.url=http://localhost:9093 \
  --duration=2h \
  --comment="Planned maintenance window"

# List active silences
amtool silence query --alertmanager.url=http://localhost:9093

# Check TSDB stats (disk usage, series count)
curl -s http://localhost:9090/api/v1/status/tsdb | python3 -m json.tool

# Check Prometheus targets and their scrape status
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool

# Check Loki ingestion stats
curl -s http://localhost:3100/metrics | grep loki_ingester_streams_created_total
```

### Prometheus disk sizing guide

As a rough rule: each metric sample costs about 1.5 bytes on disk (after compression). Calculate:

```
disk_usage_GB = (series_count * samples_per_second * bytes_per_sample * retention_seconds) / 1e9

Example: 10,000 series, scraping every 15s, 30-day retention:
= (10000 * (1/15) * 1.5 * 2592000) / 1e9
= (10000 * 0.0667 * 1.5 * 2592000) / 1e9
≈ 2.6 GB
```

Add 20% headroom and use `--storage.tsdb.retention.size` as a hard cap.

### Troubleshooting checklist

**Prometheus not scraping a target:**
```bash
# Check the target status in the UI
curl -s "http://localhost:9090/api/v1/targets" | python3 -m json.tool | grep -A5 "web-01"

# Try scraping manually from the Prometheus server
curl -s http://web-01:9100/metrics | head -20

# Check firewall
nc -zv web-01 9100

# Check Node Exporter logs on the target
journalctl -u node_exporter -n 50
```

**Alert not firing when it should:**
```bash
# Test the PromQL expression directly
curl -sg 'http://localhost:9090/api/v1/query?query=up==0' | python3 -m json.tool

# Check rule evaluation errors
curl -s http://localhost:9090/api/v1/rules | python3 -m json.tool | grep -i error

# Check Prometheus logs
journalctl -u prometheus -n 100 | grep -i error
```

**Alertmanager not sending notifications:**
```bash
# Check Alertmanager logs
journalctl -u alertmanager -n 100

# Check config is valid
amtool check-config /etc/alertmanager/alertmanager.yml

# Manually send a test alert
amtool alert add alertname=TestAlert severity=warning instance=localhost \
  --alertmanager.url=http://localhost:9093

# Check receiver configuration
amtool config routes test --alertmanager.url=http://localhost:9093 \
  alertname=TestAlert severity=warning
```

**Loki not receiving logs:**
```bash
# Check Promtail is shipping
curl -s http://localhost:9080/metrics | grep -E 'promtail_(sent|dropped)_entries'

# Check Loki is accepting pushes
curl -s http://localhost:3100/ready

# Check Loki ingestion stats
curl -s http://localhost:3100/loki/api/v1/label

# View Promtail logs
journalctl -u promtail -n 50
```

### Backup recommendations

```bash
# Backup Prometheus data (cold backup — stop service first)
sudo systemctl stop prometheus
sudo tar -czf /backup/prometheus-$(date +%Y%m%d).tar.gz /var/lib/prometheus
sudo systemctl start prometheus

# Backup configs (do this daily, store in git)
sudo tar -czf /backup/monitoring-configs-$(date +%Y%m%d).tar.gz \
  /etc/prometheus \
  /etc/alertmanager \
  /etc/grafana \
  /etc/loki \
  /etc/promtail

# Export Grafana dashboards via API (for all dashboards)
curl -s -u admin:password http://localhost:3000/api/search?type=dash-db | \
  python3 -c "import sys,json; [print(d['uid']) for d in json.load(sys.stdin)]" | \
  while read uid; do
    curl -s -u admin:password "http://localhost:3000/api/dashboards/uid/${uid}" \
      > "/backup/grafana-dashboard-${uid}.json"
  done
```

---

## Summary: File Locations Reference

| Component | Config | Data | Logs |
|-----------|--------|------|------|
| Prometheus | `/etc/prometheus/prometheus.yml` | `/var/lib/prometheus` | `journalctl -u prometheus` |
| Alert Rules | `/etc/prometheus/rules/*.yml` | — | — |
| Alertmanager | `/etc/alertmanager/alertmanager.yml` | `/var/lib/alertmanager` | `journalctl -u alertmanager` |
| Node Exporter | `/etc/systemd/system/node_exporter.service` | — | `journalctl -u node_exporter` |
| Grafana | `/etc/grafana/grafana.ini` | `/var/lib/grafana` | `/var/log/grafana/grafana.log` |
| Grafana Provisioning | `/etc/grafana/provisioning/` | — | — |
| Loki | `/etc/loki/loki-local-config.yaml` | `/var/lib/loki` | `journalctl -u loki` |
| Promtail | `/etc/promtail/promtail-config.yaml` | `/var/lib/promtail/positions.yaml` | `journalctl -u promtail` |

## Summary: Port Reference

| Component | Port | Protocol | Notes |
|-----------|------|----------|-------|
| Node Exporter | 9100 | HTTP | Firewall to Prometheus only |
| Prometheus | 9090 | HTTP | Firewall to internal network; behind nginx for external access |
| Alertmanager | 9093 | HTTP | Firewall to internal network |
| Grafana | 3000 | HTTP | Behind nginx with TLS |
| Loki | 3100 | HTTP | Firewall to Promtail hosts and Grafana |
| Promtail | 9080 | HTTP | Internal only (Prometheus scrapes its own metrics) |
