# Docker Production Best Practices

> Critical configurations and patterns for running Docker containers in production environments.

## Table of Contents

1. [Production Checklist](#production-checklist)
2. [Container Security Hardening](#container-security-hardening)
3. [Resource Management](#resource-management)
4. [High Availability Patterns](#high-availability-patterns)
5. [Logging and Monitoring](#logging-and-monitoring)
6. [Zero-Downtime Deployments](#zero-downtime-deployments)
7. [Backup and Recovery](#backup-and-recovery)
8. [Performance Optimization](#performance-optimization)
9. [Troubleshooting Production Issues](#troubleshooting-production-issues)

---

## Production Checklist

### Pre-Deployment Checklist

```markdown
## Image Security
- [ ] Use specific image tags (never `latest`)
- [ ] Pin to digest for critical images: `image@sha256:...`
- [ ] Scan images for vulnerabilities (Trivy, Scout, Snyk)
- [ ] Use minimal base images (Alpine, distroless, scratch)
- [ ] Run as non-root user
- [ ] No secrets baked into images

## Container Configuration
- [ ] Resource limits defined (CPU, memory)
- [ ] Health checks configured
- [ ] Restart policies set
- [ ] Read-only root filesystem where possible
- [ ] Capabilities dropped (cap_drop: ALL)
- [ ] Security options enabled (no-new-privileges)

## Networking
- [ ] Internal networks for backend services
- [ ] No unnecessary ports exposed
- [ ] TLS/SSL for all external communication
- [ ] Proper DNS configuration

## Data Management
- [ ] Named volumes for persistent data
- [ ] Backup strategy implemented
- [ ] Data encryption at rest
- [ ] Regular backup testing

## Monitoring & Logging
- [ ] Centralized logging configured
- [ ] Metrics collection enabled
- [ ] Alerting rules defined
- [ ] Log rotation configured

## Deployment
- [ ] CI/CD pipeline with image scanning
- [ ] Blue-green or rolling deployment strategy
- [ ] Rollback procedures documented
- [ ] Load balancer health checks configured
```

---

## Container Security Hardening

### Security-Focused Compose Configuration

```yaml
services:
  api:
    image: myapp:v1.2.3@sha256:abc123...  # Pinned to digest

    # Run as non-root
    user: "1000:1000"

    # Read-only filesystem
    read_only: true
    tmpfs:
      - /tmp:size=100M,mode=1777
      - /var/run:size=10M,mode=1777

    # Security options
    security_opt:
      - no-new-privileges:true
      - seccomp:./seccomp-profile.json

    # Drop all capabilities, add only required
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # Only if binding to ports < 1024

    # Resource limits (prevent DoS)
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 1G
          pids: 100  # Prevent fork bombs
        reservations:
          cpus: "0.5"
          memory: 256M

    # Health check
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

    # Network isolation
    networks:
      - internal

    # Restart policy
    restart: unless-stopped
```

### Custom Seccomp Profile

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_AARCH64"],
  "syscalls": [
    {
      "names": [
        "accept", "accept4", "access", "bind", "brk", "chdir",
        "chmod", "chown", "clock_gettime", "clone", "close",
        "connect", "dup", "dup2", "epoll_create", "epoll_ctl",
        "epoll_wait", "execve", "exit", "exit_group", "fchmod",
        "fchown", "fcntl", "fstat", "fsync", "futex", "getcwd",
        "getdents64", "getegid", "geteuid", "getgid", "getpeername",
        "getpid", "getppid", "getsockname", "getsockopt", "getuid",
        "ioctl", "listen", "lseek", "madvise", "mkdir", "mmap",
        "mprotect", "munmap", "nanosleep", "newfstatat", "open",
        "openat", "pipe", "poll", "pread64", "prlimit64", "pwrite64",
        "read", "readlink", "recvfrom", "recvmsg", "rename", "rmdir",
        "rt_sigaction", "rt_sigprocmask", "rt_sigreturn", "sched_getaffinity",
        "sched_yield", "select", "sendmsg", "sendto", "set_robust_list",
        "set_tid_address", "setgid", "setsockopt", "setuid", "shutdown",
        "sigaltstack", "socket", "stat", "statfs", "tgkill", "uname",
        "unlink", "wait4", "write", "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

### AppArmor Profile

```bash
# /etc/apparmor.d/docker-production
#include <tunables/global>

profile docker-production flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  network inet tcp,
  network inet udp,
  network inet icmp,

  deny @{PROC}/* w,
  deny @{PROC}/sys/kernel/** w,
  deny @{PROC}/sysrq-trigger rwx,
  deny mount,

  file,
  umount,

  # Application specific
  /app/** r,
  /app/data/** rw,
  /tmp/** rw,
}
```

```yaml
# Use AppArmor profile
services:
  app:
    security_opt:
      - apparmor:docker-production
```

### Secrets Management

```yaml
# Using Docker secrets (requires Swarm mode or external secret management)
services:
  api:
    secrets:
      - source: db_password
        target: /run/secrets/db_password
        uid: "1000"
        gid: "1000"
        mode: 0400
      - source: api_key
        target: /run/secrets/api_key
        mode: 0400
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password
      API_KEY_FILE: /run/secrets/api_key

secrets:
  db_password:
    external: true  # Managed by external system (Vault, AWS SM, etc.)
  api_key:
    file: /secure/path/api_key.txt  # Local file (for non-Swarm)
```

### Environment Variable Security

```yaml
services:
  api:
    # Never do this in production
    # environment:
    #   DATABASE_PASSWORD: mysecretpassword

    # Use env_file with proper permissions
    env_file:
      - path: /secure/config/.env.production
        required: true

    # Or reference from host environment
    environment:
      - DATABASE_URL  # Value comes from host
```

```bash
# .env.production file permissions
chmod 600 .env.production
chown root:docker .env.production
```

---

## Resource Management

### Memory Configuration

```yaml
services:
  java-app:
    image: eclipse-temurin:21-jre
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G
    environment:
      # JVM should use ~80% of container memory
      JAVA_OPTS: >
        -XX:+UseContainerSupport
        -XX:MaxRAMPercentage=80.0
        -XX:InitialRAMPercentage=50.0
        -XX:+ExitOnOutOfMemoryError

  node-app:
    image: node:20-alpine
    deploy:
      resources:
        limits:
          memory: 1G
        reservations:
          memory: 256M
    environment:
      # Node.js memory limit
      NODE_OPTIONS: "--max-old-space-size=800"

  python-app:
    image: python:3.11-slim
    deploy:
      resources:
        limits:
          memory: 512M
    # Python typically doesn't need explicit memory config
```

### CPU Configuration

```yaml
services:
  cpu-intensive:
    deploy:
      resources:
        limits:
          cpus: "4.0"
        reservations:
          cpus: "2.0"
    # Pin to specific CPUs for consistent performance
    cpuset: "0,1,2,3"

  io-intensive:
    deploy:
      resources:
        limits:
          cpus: "1.0"
        reservations:
          cpus: "0.25"
    # Lower priority for background tasks
    cpu_shares: 512  # Default is 1024
```

### Process Limits

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          pids: 100  # Max 100 processes

    # Ulimits
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
      nproc:
        soft: 4096
        hard: 8192
      memlock:
        soft: -1
        hard: -1  # Unlimited (for databases)
```

### Storage I/O Limits

```yaml
services:
  db:
    # Block I/O weight (10-1000, default 500)
    blkio_config:
      weight: 500
      weight_device:
        - path: /dev/sda
          weight: 400
      device_read_bps:
        - path: /dev/sda
          rate: '100mb'
      device_write_bps:
        - path: /dev/sda
          rate: '50mb'
      device_read_iops:
        - path: /dev/sda
          rate: 1000
      device_write_iops:
        - path: /dev/sda
          rate: 500
```

---

## High Availability Patterns

### Load Balancer with Health Checks

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    deploy:
      resources:
        limits:
          memory: 128M
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s
      timeout: 10s
      retries: 3
    depends_on:
      api:
        condition: service_healthy

  api:
    image: myapi:v1.2.3
    deploy:
      mode: replicated
      replicas: 3
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
```

```nginx
# nginx.conf
upstream api {
    least_conn;
    server api:3000 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;

    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    location / {
        proxy_pass http://api;
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 3;

        # Health check headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Database High Availability

```yaml
# PostgreSQL with replication
services:
  postgres-primary:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres-primary-data:/var/lib/postgresql/data
      - ./postgres/primary.conf:/etc/postgresql/postgresql.conf:ro
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    secrets:
      - db_password
    deploy:
      resources:
        limits:
          memory: 2G
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - database

  postgres-replica:
    image: postgres:16-alpine
    environment:
      PGUSER: ${POSTGRES_USER}
      PGPASSWORD_FILE: /run/secrets/db_password
    volumes:
      - postgres-replica-data:/var/lib/postgresql/data
      - ./postgres/replica.conf:/etc/postgresql/postgresql.conf:ro
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    secrets:
      - db_password
    deploy:
      resources:
        limits:
          memory: 2G
    depends_on:
      postgres-primary:
        condition: service_healthy
    networks:
      - database

volumes:
  postgres-primary-data:
  postgres-replica-data:

networks:
  database:
    internal: true
```

### Redis Cluster

```yaml
services:
  redis-master:
    image: redis:7-alpine
    command: >
      redis-server
      --appendonly yes
      --maxmemory 256mb
      --maxmemory-policy volatile-lru
      --save 900 1
      --save 300 10
      --save 60 10000
    volumes:
      - redis-data:/data
    deploy:
      resources:
        limits:
          memory: 300M
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - cache

  redis-replica:
    image: redis:7-alpine
    command: redis-server --replicaof redis-master 6379 --appendonly yes
    deploy:
      mode: replicated
      replicas: 2
      resources:
        limits:
          memory: 300M
    depends_on:
      redis-master:
        condition: service_healthy
    networks:
      - cache

networks:
  cache:
    internal: true
```

---

## Logging and Monitoring

### Centralized Logging with ELK

```yaml
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"
        labels: "service,environment"
    labels:
      service: "api"
      environment: "production"

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    user: root
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    deploy:
      resources:
        limits:
          memory: 256M
    depends_on:
      - elasticsearch

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    deploy:
      resources:
        limits:
          memory: 2G
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "127.0.0.1:5601:5601"
    depends_on:
      elasticsearch:
        condition: service_healthy
```

```yaml
# filebeat.yml
filebeat.inputs:
  - type: container
    paths:
      - '/var/lib/docker/containers/*/*.log'
    processors:
      - add_docker_metadata:
          host: "unix:///var/run/docker.sock"
      - decode_json_fields:
          fields: ["message"]
          target: "json"
          overwrite_keys: true

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "docker-logs-%{+yyyy.MM.dd}"

logging.level: warning
```

### Prometheus Metrics

```yaml
services:
  app:
    image: myapp:v1.2.3
    ports:
      - "3000:3000"
    labels:
      prometheus.scrape: "true"
      prometheus.port: "3000"
      prometheus.path: "/metrics"

  prometheus:
    image: prom/prometheus:v2.47.0
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alerts:/etc/prometheus/alerts:ro
      - prometheus-data:/prometheus
    ports:
      - "127.0.0.1:9090:9090"
    deploy:
      resources:
        limits:
          memory: 1G

  alertmanager:
    image: prom/alertmanager:v0.26.0
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    ports:
      - "127.0.0.1:9093:9093"

  grafana:
    image: grafana/grafana:10.2.0
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    environment:
      - GF_SECURITY_ADMIN_PASSWORD__FILE=/run/secrets/grafana_password
      - GF_INSTALL_PLUGINS=grafana-clock-panel
    secrets:
      - grafana_password
    ports:
      - "127.0.0.1:3001:3000"

volumes:
  prometheus-data:
  grafana-data:
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - '/etc/prometheus/alerts/*.yml'

scrape_configs:
  - job_name: 'docker-containers'
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
    relabel_configs:
      - source_labels: [__meta_docker_container_label_prometheus_scrape]
        regex: true
        action: keep
      - source_labels: [__meta_docker_container_label_prometheus_port]
        target_label: __address__
        regex: (.+)
        replacement: $1
      - source_labels: [__meta_docker_container_name]
        target_label: container_name
```

### Production Alerts

```yaml
# alerts/container-alerts.yml
groups:
  - name: container-alerts
    rules:
      - alert: ContainerHighCPU
        expr: rate(container_cpu_usage_seconds_total[5m]) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.container_name }}"
          description: "Container {{ $labels.container_name }} CPU usage is above 80%"

      - alert: ContainerHighMemory
        expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High memory usage on {{ $labels.container_name }}"
          description: "Container {{ $labels.container_name }} memory usage is above 90%"

      - alert: ContainerRestarting
        expr: increase(container_restart_count[1h]) > 3
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.container_name }} is restarting frequently"

      - alert: ContainerDown
        expr: absent(container_last_seen{name=~".+"}) == 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Container is down"
```

---

## Zero-Downtime Deployments

### Rolling Update Strategy

```yaml
services:
  api:
    image: myapi:${VERSION:-latest}
    deploy:
      mode: replicated
      replicas: 3
      update_config:
        parallelism: 1          # Update one container at a time
        delay: 10s              # Wait between updates
        failure_action: rollback
        monitor: 60s            # Monitor for this duration after update
        max_failure_ratio: 0.3  # Tolerate 30% failures
        order: start-first      # Start new before stopping old
      rollback_config:
        parallelism: 1
        delay: 10s
        failure_action: pause
        monitor: 60s
        order: stop-first
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
```

### Blue-Green Deployment Script

```bash
#!/bin/bash
set -euo pipefail

# blue-green-deploy.sh
COMPOSE_PROJECT_NAME=${COMPOSE_PROJECT_NAME:-myapp}
NEW_VERSION=${1:?Version required}
CURRENT_COLOR=$(docker compose ps --format json | jq -r '.[0].Name' | grep -oE 'blue|green' || echo "none")

if [ "$CURRENT_COLOR" = "blue" ] || [ "$CURRENT_COLOR" = "none" ]; then
    NEW_COLOR="green"
    OLD_COLOR="blue"
else
    NEW_COLOR="blue"
    OLD_COLOR="green"
fi

echo "Deploying $NEW_VERSION to $NEW_COLOR environment..."

# Deploy new version
VERSION=$NEW_VERSION COLOR=$NEW_COLOR docker compose -f docker-compose.yml \
    -f docker-compose.$NEW_COLOR.yml up -d

# Wait for health checks
echo "Waiting for health checks..."
for i in {1..30}; do
    if docker compose -f docker-compose.$NEW_COLOR.yml ps | grep -q "healthy"; then
        echo "New deployment is healthy"
        break
    fi
    if [ $i -eq 30 ]; then
        echo "Health check timeout, rolling back..."
        docker compose -f docker-compose.$NEW_COLOR.yml down
        exit 1
    fi
    sleep 10
done

# Switch traffic
echo "Switching traffic to $NEW_COLOR..."
docker exec nginx nginx -s reload

# Stop old deployment
if [ "$CURRENT_COLOR" != "none" ]; then
    echo "Stopping $OLD_COLOR deployment..."
    docker compose -f docker-compose.$OLD_COLOR.yml down
fi

echo "Deployment complete!"
```

### Graceful Shutdown

```javascript
// Node.js graceful shutdown
const server = http.createServer(app);
let isShuttingDown = false;

const gracefulShutdown = async (signal) => {
    console.log(`Received ${signal}, starting graceful shutdown...`);
    isShuttingDown = true;

    // Stop accepting new connections
    server.close(() => {
        console.log('HTTP server closed');
    });

    // Wait for existing connections (with timeout)
    const shutdownTimeout = setTimeout(() => {
        console.error('Shutdown timeout, forcing exit');
        process.exit(1);
    }, 30000);

    try {
        // Close database connections
        await db.close();

        // Close Redis connections
        await redis.quit();

        // Close message queue connections
        await messageQueue.close();

        clearTimeout(shutdownTimeout);
        console.log('Graceful shutdown complete');
        process.exit(0);
    } catch (error) {
        console.error('Error during shutdown:', error);
        process.exit(1);
    }
};

// Health check endpoint that respects shutdown state
app.get('/health', (req, res) => {
    if (isShuttingDown) {
        res.status(503).json({ status: 'shutting_down' });
    } else {
        res.status(200).json({ status: 'healthy' });
    }
});

process.on('SIGTERM', gracefulShutdown);
process.on('SIGINT', gracefulShutdown);
```

```dockerfile
# Dockerfile with proper signal handling
FROM node:20-alpine

# Install tini for proper signal handling
RUN apk add --no-cache tini

WORKDIR /app
COPY . .
RUN npm ci --only=production

# Use tini as init process
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]

# Or use dumb-init
# RUN apk add --no-cache dumb-init
# ENTRYPOINT ["dumb-init", "--"]
```

---

## Backup and Recovery

### Volume Backup Script

```bash
#!/bin/bash
set -euo pipefail

# backup-volumes.sh
BACKUP_DIR="/backups/$(date +%Y-%m-%d)"
RETENTION_DAYS=30

mkdir -p "$BACKUP_DIR"

# Backup PostgreSQL
echo "Backing up PostgreSQL..."
docker compose exec -T postgres pg_dump -U postgres mydb | \
    gzip > "$BACKUP_DIR/postgres-$(date +%H%M%S).sql.gz"

# Backup volumes
for volume in $(docker volume ls --format '{{.Name}}' | grep "^myapp_"); do
    echo "Backing up volume: $volume"
    docker run --rm \
        -v "$volume:/source:ro" \
        -v "$BACKUP_DIR:/backup" \
        alpine tar czf "/backup/$volume.tar.gz" -C /source .
done

# Encrypt backups
if [ -n "${BACKUP_ENCRYPTION_KEY:-}" ]; then
    for file in "$BACKUP_DIR"/*.gz; do
        openssl enc -aes-256-cbc -salt -pbkdf2 \
            -in "$file" -out "$file.enc" \
            -pass env:BACKUP_ENCRYPTION_KEY
        rm "$file"
    done
fi

# Upload to S3
if command -v aws &> /dev/null; then
    aws s3 sync "$BACKUP_DIR" "s3://my-backups/$(date +%Y-%m-%d)/" \
        --storage-class STANDARD_IA
fi

# Cleanup old backups
find /backups -type d -mtime +$RETENTION_DAYS -exec rm -rf {} +

echo "Backup complete: $BACKUP_DIR"
```

### Restore Script

```bash
#!/bin/bash
set -euo pipefail

# restore-volumes.sh
BACKUP_DATE=${1:?Backup date required (YYYY-MM-DD)}
BACKUP_DIR="/backups/$BACKUP_DATE"

if [ ! -d "$BACKUP_DIR" ]; then
    # Download from S3
    aws s3 sync "s3://my-backups/$BACKUP_DATE/" "$BACKUP_DIR/"
fi

# Decrypt if needed
for file in "$BACKUP_DIR"/*.enc; do
    [ -f "$file" ] || continue
    openssl enc -d -aes-256-cbc -pbkdf2 \
        -in "$file" -out "${file%.enc}" \
        -pass env:BACKUP_ENCRYPTION_KEY
done

# Stop services
docker compose stop

# Restore PostgreSQL
echo "Restoring PostgreSQL..."
docker compose up -d postgres
sleep 10
gunzip -c "$BACKUP_DIR"/postgres-*.sql.gz | \
    docker compose exec -T postgres psql -U postgres mydb

# Restore volumes
for backup in "$BACKUP_DIR"/*.tar.gz; do
    volume=$(basename "$backup" .tar.gz)
    echo "Restoring volume: $volume"
    docker run --rm \
        -v "$volume:/target" \
        -v "$BACKUP_DIR:/backup:ro" \
        alpine sh -c "rm -rf /target/* && tar xzf /backup/$(basename $backup) -C /target"
done

# Restart services
docker compose up -d

echo "Restore complete"
```

### Automated Backup with Cron

```yaml
services:
  backup:
    image: alpine:3.18
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./backup-scripts:/scripts:ro
      - backup-data:/backups
    environment:
      - BACKUP_ENCRYPTION_KEY_FILE=/run/secrets/backup_key
      - AWS_ACCESS_KEY_ID
      - AWS_SECRET_ACCESS_KEY
    secrets:
      - backup_key
    command: >
      sh -c "
        apk add --no-cache docker-cli aws-cli openssl &&
        echo '0 2 * * * /scripts/backup-volumes.sh' | crontab - &&
        crond -f
      "
    deploy:
      resources:
        limits:
          memory: 256M
```

---

## Performance Optimization

### Container Startup Optimization

```dockerfile
# Multi-stage build for smaller images
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build && npm prune --production

FROM node:20-alpine
WORKDIR /app

# Install only runtime dependencies
RUN apk add --no-cache tini

# Copy only necessary files
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

# Use non-root user
RUN addgroup -S app && adduser -S app -G app
USER app

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/main.js"]
```

### Network Optimization

```yaml
services:
  app:
    # Use host network for performance-critical services
    # network_mode: host  # Only if necessary

    # Or use macvlan for near-native performance
    networks:
      production:
        ipv4_address: 172.20.0.10

networks:
  production:
    driver: macvlan
    driver_opts:
      parent: eth0
    ipam:
      config:
        - subnet: 172.20.0.0/24
          gateway: 172.20.0.1
```

### Storage Optimization

```yaml
services:
  db:
    volumes:
      # Use named volumes for better performance
      - postgres-data:/var/lib/postgresql/data

    # Or use tmpfs for ephemeral data
    tmpfs:
      - /tmp:size=100M

    # Storage driver options
    storage_opt:
      size: 50G

volumes:
  postgres-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/ssd/postgres  # SSD storage
```

### JVM Optimization for Containers

```yaml
services:
  java-app:
    image: eclipse-temurin:21-jre
    environment:
      JAVA_OPTS: >
        -XX:+UseContainerSupport
        -XX:MaxRAMPercentage=75.0
        -XX:InitialRAMPercentage=50.0
        -XX:+UseG1GC
        -XX:MaxGCPauseMillis=200
        -XX:+UseStringDeduplication
        -XX:+ExitOnOutOfMemoryError
        -Djava.security.egd=file:/dev/./urandom
    deploy:
      resources:
        limits:
          memory: 2G
```

---

## Troubleshooting Production Issues

### Diagnostic Commands

```bash
# Container health and status
docker compose ps -a
docker inspect --format='{{.State.Health.Status}}' container_name
docker inspect --format='{{range .State.Health.Log}}{{.Output}}{{end}}' container_name

# Resource usage
docker stats --no-stream
docker compose top

# Network debugging
docker network inspect bridge
docker exec container_name netstat -tlnp
docker exec container_name ss -tlnp

# Logs
docker compose logs --tail=1000 --timestamps service_name
docker compose logs --since 1h service_name

# File system
docker exec container_name df -h
docker system df -v

# Process information
docker exec container_name ps aux
docker exec container_name cat /proc/1/status
```

### Debug Container

```yaml
services:
  debug:
    image: nicolaka/netshoot:latest
    profiles:
      - debug
    network_mode: container:api  # Share network with target container
    cap_add:
      - NET_ADMIN
      - SYS_PTRACE
    security_opt:
      - apparmor:unconfined
```

```bash
# Start debug container
docker compose --profile debug up -d debug

# Connect and debug
docker compose exec debug bash
# Inside: tcpdump, netstat, curl, dig, nslookup, etc.
```

### Common Issues and Solutions

```bash
# Issue: Container keeps restarting
docker logs --tail=100 container_name
docker inspect container_name | jq '.[0].State'

# Issue: Out of memory
docker stats container_name
docker inspect container_name | jq '.[0].HostConfig.Memory'

# Issue: Slow container startup
docker events --filter container=container_name
time docker compose up service_name

# Issue: Network connectivity
docker network ls
docker network inspect network_name
docker exec container_name ping other_container
docker exec container_name nslookup other_container

# Issue: Volume permissions
docker exec container_name ls -la /path/to/volume
docker exec container_name id
docker run --rm -v volume_name:/data alpine ls -la /data

# Issue: Port conflicts
docker port container_name
netstat -tlnp | grep PORT
lsof -i :PORT
```

### Performance Analysis

```bash
# CPU profiling
docker exec container_name top -H
docker exec container_name cat /proc/loadavg

# Memory analysis
docker exec container_name cat /proc/meminfo
docker exec container_name free -m

# I/O analysis
docker exec container_name iostat 1 5
docker exec container_name iotop

# Network analysis
docker exec container_name ss -s
docker exec container_name cat /proc/net/dev
```

---

## Quick Reference

### Production Compose Template

```yaml
services:
  api:
    image: myapi:${VERSION}@sha256:${DIGEST}
    user: "1000:1000"
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    tmpfs:
      - /tmp:size=100M,mode=1777
    deploy:
      mode: replicated
      replicas: 3
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
        reservations:
          memory: 256M
          cpus: "0.25"
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        order: start-first
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"
    networks:
      - internal

networks:
  internal:
    internal: true
```

### Essential Environment Variables

```bash
# .env.production
COMPOSE_PROJECT_NAME=myapp
COMPOSE_FILE=docker-compose.yml:docker-compose.prod.yml

# Application
NODE_ENV=production
LOG_LEVEL=info
PORT=3000

# Database
DATABASE_URL=postgres://user@db:5432/mydb

# Monitoring
PROMETHEUS_ENABLED=true
METRICS_PORT=9090
```
