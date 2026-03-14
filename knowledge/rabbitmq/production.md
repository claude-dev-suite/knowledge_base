# RabbitMQ - Production Operations

## Cluster Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     RabbitMQ Cluster                             │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Node 1     │  │   Node 2     │  │   Node 3     │          │
│  │   (Disk)     │──│   (Disk)     │──│   (RAM)      │          │
│  │              │  │              │  │              │          │
│  │ Queues: A,B  │  │ Queues: C,D  │  │ Queues: E,F  │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                 Shared Metadata                           │  │
│  │  - Exchange definitions                                   │  │
│  │  - Binding definitions                                    │  │
│  │  - Virtual hosts                                          │  │
│  │  - User/permissions                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## High Availability

### Quorum Queues (Recommended)

Replicated queues based on Raft consensus.

```javascript
// Node.js
await channel.assertQueue('orders', {
  durable: true,
  arguments: {
    'x-queue-type': 'quorum',
    'x-quorum-initial-group-size': 3,
    'x-delivery-limit': 5
  }
});
```

```bash
# CLI
rabbitmqadmin declare queue name=orders \
  arguments='{"x-queue-type":"quorum","x-quorum-initial-group-size":3}'
```

### Quorum Queue Features
- Automatic leader election
- Data safety (WAL-based)
- Poison message handling (delivery-limit)
- Memory limit per queue

### Classic Mirrored Queues (Legacy)

```bash
# Policy for mirroring
rabbitmqctl set_policy ha-all "^ha\." \
  '{"ha-mode":"all","ha-sync-mode":"automatic"}' \
  --apply-to queues
```

## Security

### TLS Configuration

```ini
# rabbitmq.conf
listeners.ssl.default = 5671

ssl_options.cacertfile = /path/to/ca_certificate.pem
ssl_options.certfile   = /path/to/server_certificate.pem
ssl_options.keyfile    = /path/to/server_key.pem
ssl_options.verify     = verify_peer
ssl_options.fail_if_no_peer_cert = true

# Disable non-TLS
listeners.tcp = none
```

### Authentication

```ini
# rabbitmq.conf
auth_mechanisms.1 = PLAIN
auth_mechanisms.2 = AMQPLAIN

# LDAP authentication
auth_backends.1 = ldap
auth_backends.2 = internal

auth_ldap.servers.1 = ldap.example.com
auth_ldap.user_dn_pattern = cn=${username},ou=users,dc=example,dc=com
```

### Authorization

```bash
# Create user
rabbitmqctl add_user app_user password

# Set permissions (configure, write, read patterns)
rabbitmqctl set_permissions -p /production app_user "^app-.*" "^app-.*" "^app-.*"

# Set user tags
rabbitmqctl set_user_tags app_user monitoring
```

### Virtual Host Isolation

```bash
# Create vhost
rabbitmqctl add_vhost /service-a
rabbitmqctl add_vhost /service-b

# Service-specific permissions
rabbitmqctl set_permissions -p /service-a service-a-user ".*" ".*" ".*"
```

## Monitoring

### Key Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `queue_messages` | Total messages in queue | > 10000 |
| `queue_messages_ready` | Messages ready for delivery | > 5000 |
| `queue_messages_unacknowledged` | Delivered but unacked | > 1000 |
| `queue_consumers` | Consumer count | = 0 |
| `message_stats.publish` | Publish rate | Anomaly |
| `message_stats.deliver_get` | Consume rate | Anomaly |
| `connection_count` | Total connections | > 1000 |
| `channel_count` | Total channels | > 5000 |
| `mem_used` | Memory usage | > 80% |
| `disk_free` | Available disk | < 2GB |

### Prometheus Integration

```yaml
# docker-compose.yml
services:
  rabbitmq:
    image: rabbitmq:3-management
    environment:
      RABBITMQ_ENABLED_PLUGINS: "rabbitmq_prometheus"
    ports:
      - "15692:15692"  # Prometheus metrics
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'rabbitmq'
    static_configs:
      - targets: ['rabbitmq:15692']
```

### Alerting Rules

```yaml
groups:
  - name: rabbitmq
    rules:
      - alert: RabbitMQQueueFilling
        expr: rabbitmq_queue_messages > 10000
        for: 5m
        labels:
          severity: warning

      - alert: RabbitMQNoConsumers
        expr: rabbitmq_queue_consumers == 0
        for: 2m
        labels:
          severity: critical

      - alert: RabbitMQHighMemory
        expr: rabbitmq_process_resident_memory_bytes / rabbitmq_resident_memory_limit_bytes > 0.8
        for: 5m
        labels:
          severity: warning

      - alert: RabbitMQDiskLow
        expr: rabbitmq_disk_space_available_bytes < 2147483648
        for: 1m
        labels:
          severity: critical
```

## Performance Tuning

### Memory Configuration

```ini
# rabbitmq.conf
# Memory threshold (40% of RAM)
vm_memory_high_watermark.relative = 0.4

# Or absolute
vm_memory_high_watermark.absolute = 2GB

# Paging threshold
vm_memory_high_watermark_paging_ratio = 0.75
```

### Disk Alarm

```ini
# rabbitmq.conf
# Minimum free disk space
disk_free_limit.absolute = 2GB

# Or relative to RAM
disk_free_limit.relative = 1.5
```

### Connection/Channel Limits

```ini
# rabbitmq.conf
# Max connections
# connection_max = 10000

# Channel max per connection
channel_max = 128

# Heartbeat
heartbeat = 60
```

### Queue Performance

```javascript
// Lazy queues (disk-based, lower memory)
await channel.assertQueue('large-queue', {
  arguments: {
    'x-queue-mode': 'lazy'
  }
});

// Message TTL
await channel.assertQueue('temp-queue', {
  arguments: {
    'x-message-ttl': 86400000  // 24 hours
  }
});

// Max length
await channel.assertQueue('bounded-queue', {
  arguments: {
    'x-max-length': 10000,
    'x-overflow': 'reject-publish'  // or 'drop-head'
  }
});
```

## Operations

### Cluster Management

```bash
# Join cluster
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster rabbit@node1
rabbitmqctl start_app

# Check cluster status
rabbitmqctl cluster_status

# Remove node
rabbitmqctl forget_cluster_node rabbit@node3
```

### Queue Management

```bash
# List queues
rabbitmqctl list_queues name messages consumers memory

# Purge queue
rabbitmqctl purge_queue queue_name

# Delete queue
rabbitmqctl delete_queue queue_name

# Move queue leader (quorum)
rabbitmq-queues rebalance all
```

### User Management

```bash
# List users
rabbitmqctl list_users

# Change password
rabbitmqctl change_password username newpassword

# Delete user
rabbitmqctl delete_user username
```

## Backup & Recovery

### Definitions Export

```bash
# Export definitions
rabbitmqctl export_definitions /path/to/definitions.json

# Import definitions
rabbitmqctl import_definitions /path/to/definitions.json

# Via API
curl -u admin:password http://localhost:15672/api/definitions > definitions.json
```

### Message Backup

```bash
# Shovel plugin for message migration
rabbitmq-plugins enable rabbitmq_shovel
rabbitmq-plugins enable rabbitmq_shovel_management

# Configure shovel via API or management UI
```

### Disaster Recovery

```bash
# 1. Stop nodes
rabbitmqctl stop_app

# 2. Backup mnesia directory
cp -r /var/lib/rabbitmq/mnesia backup/

# 3. Backup definitions
rabbitmqctl export_definitions definitions.json

# 4. Restore
cp -r backup/mnesia /var/lib/rabbitmq/
rabbitmqctl import_definitions definitions.json
rabbitmqctl start_app
```

## Production Checklist

### Security
- [ ] TLS enabled for all connections
- [ ] Default guest user disabled
- [ ] Per-application users with minimal permissions
- [ ] Virtual hosts for isolation
- [ ] Management UI restricted

### Reliability
- [ ] Quorum queues for important data
- [ ] Dead letter exchanges configured
- [ ] Message persistence enabled
- [ ] Publisher confirms in use
- [ ] Consumer acknowledgments manual

### Performance
- [ ] Memory limits configured
- [ ] Disk alarm thresholds set
- [ ] Connection limits appropriate
- [ ] Lazy queues for large backlogs
- [ ] Prefetch optimized

### Operations
- [ ] Monitoring dashboards
- [ ] Alerting configured
- [ ] Backup procedures tested
- [ ] Cluster health checks
- [ ] Log aggregation

### Configuration

```ini
# rabbitmq.conf - Production settings
listeners.tcp.default = 5672
management.tcp.port = 15672

# Clustering
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_k8s
cluster_formation.k8s.host = kubernetes.default.svc.cluster.local

# Memory
vm_memory_high_watermark.relative = 0.4
vm_memory_high_watermark_paging_ratio = 0.75

# Disk
disk_free_limit.absolute = 2GB

# Connections
channel_max = 128
heartbeat = 60

# TLS
listeners.ssl.default = 5671
ssl_options.cacertfile = /certs/ca.pem
ssl_options.certfile = /certs/server.pem
ssl_options.keyfile = /certs/server-key.pem
ssl_options.verify = verify_peer
ssl_options.fail_if_no_peer_cert = true

# Logging
log.file.level = info
log.console = true
log.console.level = warning
```

## References

- [Production Checklist](https://www.rabbitmq.com/production-checklist.html)
- [Monitoring](https://www.rabbitmq.com/monitoring.html)
- [Clustering Guide](https://www.rabbitmq.com/clustering.html)
- [Quorum Queues](https://www.rabbitmq.com/quorum-queues.html)
