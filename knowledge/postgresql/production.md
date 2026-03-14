# PostgreSQL Production Configuration

> Comprehensive guide for running PostgreSQL in production environments with optimal performance, security, and reliability.

## Table of Contents

1. [Production Configuration](#production-configuration)
2. [Performance Tuning](#performance-tuning)
3. [Connection Management](#connection-management)
4. [Indexes and Query Optimization](#indexes-and-query-optimization)
5. [Backup and Recovery](#backup-and-recovery)
6. [Replication and High Availability](#replication-and-high-availability)
7. [Monitoring and Alerting](#monitoring-and-alerting)
8. [Security Hardening](#security-hardening)
9. [Maintenance Operations](#maintenance-operations)
10. [Troubleshooting](#troubleshooting)

---

## Production Configuration

### postgresql.conf Essential Settings

```ini
# postgresql.conf - Production Configuration

#------------------------------------------------------------------------------
# CONNECTION SETTINGS
#------------------------------------------------------------------------------
listen_addresses = '*'              # Allow external connections
port = 5432
max_connections = 200               # Adjust based on application needs
superuser_reserved_connections = 3

#------------------------------------------------------------------------------
# MEMORY SETTINGS
#------------------------------------------------------------------------------
# For 16GB RAM server - adjust proportionally
shared_buffers = 4GB                # 25% of RAM
effective_cache_size = 12GB         # 75% of RAM
work_mem = 64MB                     # Per-operation memory (careful with concurrency)
maintenance_work_mem = 1GB          # For VACUUM, CREATE INDEX
huge_pages = try                    # Enable huge pages if available

#------------------------------------------------------------------------------
# WAL SETTINGS
#------------------------------------------------------------------------------
wal_level = replica                 # Required for replication
max_wal_size = 4GB
min_wal_size = 1GB
wal_buffers = 64MB                  # 1/32 of shared_buffers
checkpoint_completion_target = 0.9
checkpoint_timeout = 10min

#------------------------------------------------------------------------------
# QUERY PLANNER
#------------------------------------------------------------------------------
random_page_cost = 1.1              # For SSD storage (default 4.0 for HDD)
effective_io_concurrency = 200      # For SSD (1 for HDD)
default_statistics_target = 100     # Increase for complex queries

#------------------------------------------------------------------------------
# PARALLELISM
#------------------------------------------------------------------------------
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_parallel_maintenance_workers = 4

#------------------------------------------------------------------------------
# LOGGING
#------------------------------------------------------------------------------
logging_collector = on
log_destination = 'stderr'
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_min_duration_statement = 1000   # Log queries > 1 second
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '

#------------------------------------------------------------------------------
# AUTOVACUUM
#------------------------------------------------------------------------------
autovacuum = on
autovacuum_max_workers = 4
autovacuum_naptime = 1min
autovacuum_vacuum_threshold = 50
autovacuum_analyze_threshold = 50
autovacuum_vacuum_scale_factor = 0.05      # 5% of table
autovacuum_analyze_scale_factor = 0.05
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 1000

#------------------------------------------------------------------------------
# REPLICATION
#------------------------------------------------------------------------------
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
hot_standby = on
```

### pg_hba.conf Security Configuration

```
# pg_hba.conf - Production Configuration

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local connections
local   all             postgres                                peer
local   all             all                                     peer

# IPv4 local connections
host    all             all             127.0.0.1/32            scram-sha-256

# Application server connections (specific IPs)
host    mydb            app_user        10.0.1.0/24             scram-sha-256

# Replication connections
host    replication     repl_user       10.0.2.0/24             scram-sha-256

# Monitoring (read-only)
host    all             monitor_user    10.0.3.0/24             scram-sha-256

# Deny all other connections
host    all             all             0.0.0.0/0               reject
```

### Docker Production Configuration

```yaml
# docker-compose.postgres-prod.yml
services:
  postgres:
    image: postgres:16-alpine
    container_name: postgres-prod
    restart: unless-stopped

    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD_FILE: /run/secrets/postgres_password
      POSTGRES_DB: mydb
      PGDATA: /var/lib/postgresql/data/pgdata

    secrets:
      - postgres_password

    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./postgresql.conf:/etc/postgresql/postgresql.conf:ro
      - ./pg_hba.conf:/etc/postgresql/pg_hba.conf:ro
      - postgres-logs:/var/log/postgresql

    command: >
      postgres
      -c config_file=/etc/postgresql/postgresql.conf
      -c hba_file=/etc/postgresql/pg_hba.conf

    deploy:
      resources:
        limits:
          memory: 8G
          cpus: "4"
        reservations:
          memory: 4G
          cpus: "2"

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

    shm_size: 256mb  # For shared memory

    networks:
      - database

    # Security
    user: "999:999"  # postgres user
    read_only: false  # Cannot be read-only for PostgreSQL

secrets:
  postgres_password:
    file: ./secrets/postgres_password.txt

volumes:
  postgres-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/ssd/postgres/data
  postgres-logs:

networks:
  database:
    internal: true
```

---

## Performance Tuning

### Memory Calculation Formula

```bash
#!/bin/bash
# memory-calculator.sh - Calculate optimal PostgreSQL memory settings

TOTAL_RAM_GB=${1:-16}  # Default 16GB

# Calculate settings
SHARED_BUFFERS=$((TOTAL_RAM_GB / 4))GB
EFFECTIVE_CACHE_SIZE=$((TOTAL_RAM_GB * 3 / 4))GB
MAINTENANCE_WORK_MEM=$((TOTAL_RAM_GB / 16))GB
WORK_MEM=$((TOTAL_RAM_GB * 1024 / 400))MB  # Assuming ~100 concurrent queries

echo "For ${TOTAL_RAM_GB}GB RAM:"
echo "  shared_buffers = ${SHARED_BUFFERS}"
echo "  effective_cache_size = ${EFFECTIVE_CACHE_SIZE}"
echo "  maintenance_work_mem = ${MAINTENANCE_WORK_MEM}"
echo "  work_mem = ${WORK_MEM}"
```

### Query Performance Analysis

```sql
-- Enable query statistics
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Find slowest queries
SELECT
    round((total_exec_time / 1000)::numeric, 2) AS total_seconds,
    calls,
    round((mean_exec_time)::numeric, 2) AS avg_ms,
    round((max_exec_time)::numeric, 2) AS max_ms,
    rows,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Find queries with most I/O
SELECT
    round((shared_blks_hit + shared_blks_read)::numeric, 2) AS total_buffers,
    round(100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0), 2) AS hit_ratio,
    calls,
    query
FROM pg_stat_statements
ORDER BY shared_blks_read DESC
LIMIT 20;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

### Table Statistics

```sql
-- Table size and bloat estimation
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS table_size,
    pg_size_pretty(pg_indexes_size(schemaname || '.' || tablename)) AS index_size,
    n_live_tup AS live_rows,
    n_dead_tup AS dead_rows,
    round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio,
    last_vacuum,
    last_autovacuum,
    last_analyze
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 20;

-- Index usage statistics
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS scans,
    idx_tup_read AS tuples_read,
    idx_tup_fetch AS tuples_fetched
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC
LIMIT 20;  -- Least used indexes (candidates for removal)

-- Unused indexes
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Buffer and Cache Analysis

```sql
-- Buffer cache hit ratio (should be > 99%)
SELECT
    sum(blks_hit) * 100.0 / sum(blks_hit + blks_read) AS buffer_hit_ratio
FROM pg_stat_database
WHERE datname = current_database();

-- Per-table buffer cache usage
CREATE EXTENSION IF NOT EXISTS pg_buffercache;

SELECT
    c.relname,
    pg_size_pretty(count(*) * 8192) AS buffered_size,
    round(100.0 * count(*) / (SELECT setting::int FROM pg_settings WHERE name = 'shared_buffers'), 2) AS buffer_percent,
    round(100.0 * sum(CASE WHEN b.isdirty THEN 1 ELSE 0 END) / count(*), 2) AS dirty_percent
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE c.relnamespace = (SELECT oid FROM pg_namespace WHERE nspname = 'public')
GROUP BY c.relname
ORDER BY count(*) DESC
LIMIT 20;
```

---

## Connection Management

### Connection Pooling with PgBouncer

```ini
# pgbouncer.ini
[databases]
mydb = host=postgres port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# Connection pool settings
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3

# Timeouts
server_idle_timeout = 600
client_idle_timeout = 0
query_timeout = 0
query_wait_timeout = 120

# Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60

# Admin
admin_users = pgbouncer_admin
stats_users = pgbouncer_stats
```

```yaml
# Docker Compose with PgBouncer
services:
  pgbouncer:
    image: bitnami/pgbouncer:1.21.0
    environment:
      POSTGRESQL_HOST: postgres
      POSTGRESQL_PORT: 5432
      POSTGRESQL_DATABASE: mydb
      PGBOUNCER_PORT: 6432
      PGBOUNCER_POOL_MODE: transaction
      PGBOUNCER_MAX_CLIENT_CONN: 1000
      PGBOUNCER_DEFAULT_POOL_SIZE: 20
    volumes:
      - ./pgbouncer/userlist.txt:/etc/pgbouncer/userlist.txt:ro
    ports:
      - "6432:6432"
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "pg_isready", "-h", "localhost", "-p", "6432"]
      interval: 10s
      timeout: 5s
      retries: 3
```

### Connection Monitoring

```sql
-- Current connections by state
SELECT
    state,
    count(*) AS connections,
    max(age(clock_timestamp(), query_start)) AS max_duration
FROM pg_stat_activity
WHERE pid != pg_backend_pid()
GROUP BY state
ORDER BY count(*) DESC;

-- Connections by application
SELECT
    application_name,
    client_addr,
    count(*) AS connections,
    sum(CASE WHEN state = 'active' THEN 1 ELSE 0 END) AS active,
    sum(CASE WHEN state = 'idle' THEN 1 ELSE 0 END) AS idle,
    sum(CASE WHEN state = 'idle in transaction' THEN 1 ELSE 0 END) AS idle_in_transaction
FROM pg_stat_activity
WHERE pid != pg_backend_pid()
GROUP BY application_name, client_addr
ORDER BY count(*) DESC;

-- Long-running queries
SELECT
    pid,
    now() - query_start AS duration,
    state,
    query
FROM pg_stat_activity
WHERE state != 'idle'
  AND query_start < now() - interval '5 minutes'
ORDER BY query_start;

-- Kill long-running query
SELECT pg_terminate_backend(pid);

-- Kill all idle connections for a user
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE usename = 'app_user'
  AND state = 'idle'
  AND query_start < now() - interval '10 minutes';
```

---

## Indexes and Query Optimization

### Index Types and When to Use

```sql
-- B-tree index (default, most common)
-- Use for: equality, range queries, ordering
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- Partial index (smaller, faster for specific conditions)
-- Use for: queries with WHERE clause that matches the condition
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';

-- Covering index (includes data columns)
-- Use for: index-only scans, avoiding table access
CREATE INDEX idx_orders_covering ON orders(user_id, status)
INCLUDE (total_amount, created_at);

-- GIN index (Generalized Inverted Index)
-- Use for: arrays, JSONB, full-text search
CREATE INDEX idx_products_tags ON products USING GIN(tags);
CREATE INDEX idx_users_metadata ON users USING GIN(metadata jsonb_path_ops);

-- GiST index (Generalized Search Tree)
-- Use for: geometric data, range types, full-text search
CREATE INDEX idx_locations_coords ON locations USING GIST(coordinates);

-- BRIN index (Block Range INdex)
-- Use for: large tables with naturally ordered data (e.g., timestamps)
CREATE INDEX idx_logs_created_at ON logs USING BRIN(created_at);

-- Hash index
-- Use for: equality comparisons only (rare use cases)
CREATE INDEX idx_sessions_token ON sessions USING HASH(token);

-- Expression index
-- Use for: queries on computed values
CREATE INDEX idx_users_lower_email ON users(lower(email));
CREATE INDEX idx_orders_year ON orders(date_part('year', created_at));

-- Multi-column index (order matters!)
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
-- This supports: WHERE user_id = X
--                WHERE user_id = X AND status = Y
-- Does NOT support efficiently: WHERE status = Y
```

### Query Optimization Patterns

```sql
-- EXPLAIN ANALYZE for query analysis
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = 123 AND status = 'completed';

-- Use EXISTS instead of IN for subqueries
-- Slow
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 1000);

-- Fast
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id AND o.total > 1000
);

-- Use CTEs for complex queries (materialized by default in PG12+)
WITH active_users AS MATERIALIZED (
    SELECT DISTINCT user_id
    FROM orders
    WHERE created_at > now() - interval '30 days'
)
SELECT u.*, au.user_id IS NOT NULL AS is_active
FROM users u
LEFT JOIN active_users au ON u.id = au.user_id;

-- Pagination with keyset (faster than OFFSET for large tables)
-- Instead of: SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 10000;
SELECT * FROM orders
WHERE id > 10000  -- last seen id
ORDER BY id
LIMIT 20;

-- Batch updates to avoid locking
UPDATE orders
SET processed = true
WHERE id IN (
    SELECT id FROM orders
    WHERE processed = false
    ORDER BY id
    LIMIT 1000
    FOR UPDATE SKIP LOCKED
);
```

### Index Maintenance

```sql
-- Reindex to fix bloat
REINDEX INDEX CONCURRENTLY idx_users_email;
REINDEX TABLE CONCURRENTLY users;

-- Check index bloat
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS scans
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Analyze table for better query plans
ANALYZE users;
ANALYZE VERBOSE users;

-- Vacuum and analyze
VACUUM ANALYZE users;
```

---

## Backup and Recovery

### Automated Backup Script

```bash
#!/bin/bash
set -euo pipefail

# backup-postgres.sh
BACKUP_DIR="/backups/postgres"
RETENTION_DAYS=30
POSTGRES_HOST="${POSTGRES_HOST:-localhost}"
POSTGRES_PORT="${POSTGRES_PORT:-5432}"
POSTGRES_USER="${POSTGRES_USER:-postgres}"
POSTGRES_DB="${POSTGRES_DB:-mydb}"
DATE=$(date +%Y-%m-%d_%H%M%S)

mkdir -p "$BACKUP_DIR"

# Full backup with custom format (compressed, parallel restore)
echo "Starting full backup..."
pg_dump \
    -h "$POSTGRES_HOST" \
    -p "$POSTGRES_PORT" \
    -U "$POSTGRES_USER" \
    -d "$POSTGRES_DB" \
    -Fc \
    -Z 9 \
    --no-owner \
    --no-acl \
    -f "$BACKUP_DIR/${POSTGRES_DB}_${DATE}.dump"

# Also create plain SQL backup for compatibility
pg_dump \
    -h "$POSTGRES_HOST" \
    -p "$POSTGRES_PORT" \
    -U "$POSTGRES_USER" \
    -d "$POSTGRES_DB" \
    --no-owner \
    --no-acl \
    | gzip > "$BACKUP_DIR/${POSTGRES_DB}_${DATE}.sql.gz"

# Backup roles and global objects
pg_dumpall \
    -h "$POSTGRES_HOST" \
    -p "$POSTGRES_PORT" \
    -U "$POSTGRES_USER" \
    --globals-only \
    | gzip > "$BACKUP_DIR/globals_${DATE}.sql.gz"

# Verify backup
pg_restore \
    --list "$BACKUP_DIR/${POSTGRES_DB}_${DATE}.dump" > /dev/null

# Calculate checksum
sha256sum "$BACKUP_DIR/${POSTGRES_DB}_${DATE}.dump" > \
    "$BACKUP_DIR/${POSTGRES_DB}_${DATE}.dump.sha256"

# Upload to S3 (optional)
if command -v aws &> /dev/null; then
    aws s3 cp "$BACKUP_DIR/${POSTGRES_DB}_${DATE}.dump" \
        "s3://my-backups/postgres/${DATE}/" \
        --storage-class STANDARD_IA
    aws s3 cp "$BACKUP_DIR/${POSTGRES_DB}_${DATE}.dump.sha256" \
        "s3://my-backups/postgres/${DATE}/"
fi

# Cleanup old backups
find "$BACKUP_DIR" -type f -name "*.dump" -mtime +$RETENTION_DAYS -delete
find "$BACKUP_DIR" -type f -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete
find "$BACKUP_DIR" -type f -name "*.sha256" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: ${POSTGRES_DB}_${DATE}.dump"
```

### Point-in-Time Recovery (PITR) Setup

```ini
# postgresql.conf for PITR
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
archive_timeout = 60  # Force archive every minute

# For continuous archiving to S3
# archive_command = 'wal-g wal-push %p'
```

```bash
# Restore to specific point in time
pg_restore \
    --dbname=mydb_restore \
    --clean \
    --if-exists \
    /backups/mydb_base.dump

# Apply WAL files up to target time
cat > /var/lib/postgresql/data/recovery.signal << EOF
EOF

cat > /var/lib/postgresql/data/postgresql.auto.conf << EOF
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2024-01-15 14:30:00 UTC'
recovery_target_action = 'promote'
EOF

# Restart PostgreSQL
pg_ctl restart
```

### Backup Verification Script

```bash
#!/bin/bash
# verify-backup.sh

BACKUP_FILE=${1:?Backup file required}
TEST_DB="backup_verify_$(date +%s)"

echo "Creating test database..."
createdb "$TEST_DB"

echo "Restoring backup..."
pg_restore \
    --dbname="$TEST_DB" \
    --clean \
    --if-exists \
    "$BACKUP_FILE"

echo "Verifying data..."
psql -d "$TEST_DB" -c "SELECT count(*) FROM users;"
psql -d "$TEST_DB" -c "SELECT count(*) FROM orders;"

echo "Running integrity check..."
psql -d "$TEST_DB" -c "
    SELECT schemaname, tablename
    FROM pg_tables
    WHERE schemaname = 'public';
"

echo "Cleaning up..."
dropdb "$TEST_DB"

echo "Backup verification complete!"
```

---

## Replication and High Availability

### Streaming Replication Setup

```ini
# Primary server postgresql.conf
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 1GB
synchronous_commit = on
synchronous_standby_names = 'replica1'
```

```ini
# Replica server postgresql.conf
hot_standby = on
primary_conninfo = 'host=primary port=5432 user=repl_user password=secret application_name=replica1'
primary_slot_name = 'replica1_slot'
```

```sql
-- On primary: Create replication user and slot
CREATE USER repl_user REPLICATION LOGIN PASSWORD 'secure_password';
SELECT pg_create_physical_replication_slot('replica1_slot');
```

```bash
# Initialize replica from primary
pg_basebackup \
    -h primary \
    -U repl_user \
    -D /var/lib/postgresql/data \
    -Fp \
    -Xs \
    -P \
    -R  # Create standby.signal and recovery settings
```

### Replication Monitoring

```sql
-- On primary: Check replication status
SELECT
    client_addr,
    application_name,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS replication_lag
FROM pg_stat_replication;

-- On replica: Check recovery status
SELECT
    pg_is_in_recovery() AS is_replica,
    pg_last_wal_receive_lsn() AS received_lsn,
    pg_last_wal_replay_lsn() AS replayed_lsn,
    pg_last_xact_replay_timestamp() AS last_replay_time,
    now() - pg_last_xact_replay_timestamp() AS replay_lag;

-- Check replication slots
SELECT
    slot_name,
    slot_type,
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
```

### Automatic Failover with Patroni

```yaml
# patroni.yml
scope: postgres-cluster
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: node1:8008

etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    synchronous_mode: true
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 200
        shared_buffers: 4GB
        effective_cache_size: 12GB
        work_mem: 64MB
        maintenance_work_mem: 1GB

  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: node1:5432
  data_dir: /var/lib/postgresql/data
  authentication:
    replication:
      username: repl_user
      password: repl_password
    superuser:
      username: postgres
      password: postgres_password
```

```yaml
# Docker Compose with Patroni
services:
  patroni1:
    image: patroni:latest
    hostname: node1
    environment:
      PATRONI_NAME: node1
      PATRONI_POSTGRESQL_CONNECT_ADDRESS: node1:5432
      PATRONI_RESTAPI_CONNECT_ADDRESS: node1:8008
    volumes:
      - ./patroni.yml:/etc/patroni/patroni.yml
      - patroni1-data:/var/lib/postgresql/data
    networks:
      - postgres-cluster

  patroni2:
    image: patroni:latest
    hostname: node2
    # Similar configuration...

  patroni3:
    image: patroni:latest
    hostname: node3
    # Similar configuration...

  haproxy:
    image: haproxy:alpine
    ports:
      - "5000:5000"  # Read-write
      - "5001:5001"  # Read-only
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    depends_on:
      - patroni1
      - patroni2
      - patroni3
```

---

## Monitoring and Alerting

### Key Metrics to Monitor

```sql
-- Database size
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

-- Transaction rates
SELECT
    datname,
    xact_commit + xact_rollback AS total_transactions,
    xact_commit AS commits,
    xact_rollback AS rollbacks,
    round(100.0 * xact_rollback / nullif(xact_commit + xact_rollback, 0), 2) AS rollback_percent
FROM pg_stat_database
WHERE datname = current_database();

-- Lock monitoring
SELECT
    pg_class.relname,
    pg_locks.locktype,
    pg_locks.mode,
    pg_locks.granted,
    pg_stat_activity.pid,
    pg_stat_activity.query,
    age(now(), pg_stat_activity.query_start) AS duration
FROM pg_locks
JOIN pg_class ON pg_locks.relation = pg_class.oid
JOIN pg_stat_activity ON pg_locks.pid = pg_stat_activity.pid
WHERE NOT pg_locks.granted
ORDER BY pg_stat_activity.query_start;

-- Deadlock detection
SELECT
    deadlocks,
    conflicts,
    temp_files,
    pg_size_pretty(temp_bytes) AS temp_bytes
FROM pg_stat_database
WHERE datname = current_database();

-- Checkpoint activity
SELECT
    checkpoints_timed,
    checkpoints_req,
    checkpoint_write_time,
    checkpoint_sync_time,
    buffers_checkpoint,
    buffers_clean,
    buffers_backend,
    maxwritten_clean
FROM pg_stat_bgwriter;
```

### Prometheus Postgres Exporter

```yaml
# docker-compose.monitoring.yml
services:
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:latest
    environment:
      DATA_SOURCE_NAME: "postgresql://monitor:password@postgres:5432/mydb?sslmode=disable"
    ports:
      - "9187:9187"
    command:
      - '--collector.database'
      - '--collector.locks'
      - '--collector.replication'
      - '--collector.stat_statements'
      - '--collector.stat_user_tables'
```

### Alerting Rules

```yaml
# prometheus/alerts/postgres.yml
groups:
  - name: postgres-alerts
    rules:
      - alert: PostgresqlDown
        expr: pg_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL is down"

      - alert: PostgresqlHighConnections
        expr: pg_stat_activity_count > (pg_settings_max_connections * 0.8)
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL connections > 80% of max"

      - alert: PostgresqlReplicationLag
        expr: pg_replication_lag > 60
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL replication lag > 60 seconds"

      - alert: PostgresqlLowDiskSpace
        expr: pg_database_size_bytes / node_filesystem_size_bytes > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL disk usage > 85%"

      - alert: PostgresqlSlowQueries
        expr: rate(pg_stat_statements_seconds_total[5m]) > 1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL slow queries detected"

      - alert: PostgresqlDeadlocks
        expr: increase(pg_stat_database_deadlocks[5m]) > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL deadlocks detected"

      - alert: PostgresqlHighBufferMiss
        expr: (pg_stat_database_blks_read / (pg_stat_database_blks_hit + pg_stat_database_blks_read)) > 0.03
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL buffer cache hit ratio < 97%"
```

---

## Security Hardening

### User and Role Management

```sql
-- Create application roles
CREATE ROLE app_read NOLOGIN;
GRANT CONNECT ON DATABASE mydb TO app_read;
GRANT USAGE ON SCHEMA public TO app_read;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO app_read;

CREATE ROLE app_write NOLOGIN;
GRANT app_read TO app_write;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_write;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT INSERT, UPDATE, DELETE ON TABLES TO app_write;

-- Create application user
CREATE USER app_user WITH PASSWORD 'secure_password';
GRANT app_write TO app_user;

-- Create read-only monitoring user
CREATE USER monitor_user WITH PASSWORD 'monitor_password';
GRANT app_read TO monitor_user;
GRANT pg_monitor TO monitor_user;

-- Revoke public schema access
REVOKE ALL ON SCHEMA public FROM PUBLIC;
GRANT USAGE ON SCHEMA public TO app_read;
```

### Row-Level Security

```sql
-- Enable RLS on sensitive tables
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see their own orders
CREATE POLICY orders_user_policy ON orders
    FOR ALL
    USING (user_id = current_setting('app.user_id')::int);

-- Policy: Admins can see all orders
CREATE POLICY orders_admin_policy ON orders
    FOR ALL
    TO admin_role
    USING (true);

-- Set user context in application
SET app.user_id = '123';
```

### SSL/TLS Configuration

```ini
# postgresql.conf
ssl = on
ssl_cert_file = '/etc/postgresql/ssl/server.crt'
ssl_key_file = '/etc/postgresql/ssl/server.key'
ssl_ca_file = '/etc/postgresql/ssl/ca.crt'
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'
ssl_prefer_server_ciphers = on
ssl_min_protocol_version = 'TLSv1.2'
```

```bash
# Generate SSL certificates
openssl req -new -x509 -days 365 -nodes \
    -out server.crt -keyout server.key \
    -subj "/CN=postgres.example.com"

chmod 600 server.key
chown postgres:postgres server.key server.crt
```

### Audit Logging with pgAudit

```sql
-- Install pgAudit extension
CREATE EXTENSION pgaudit;

-- Configure in postgresql.conf
-- pgaudit.log = 'write, ddl, role'
-- pgaudit.log_catalog = off
-- pgaudit.log_relation = on

-- Object-level auditing
ALTER TABLE sensitive_data SET (pgaudit.log = 'read, write');
```

---

## Maintenance Operations

### VACUUM and ANALYZE

```sql
-- Manual vacuum for specific table
VACUUM (VERBOSE, ANALYZE) orders;

-- Full vacuum (reclaims disk space, requires exclusive lock)
VACUUM FULL orders;  -- Avoid in production

-- Parallel vacuum (PostgreSQL 13+)
VACUUM (PARALLEL 4) orders;

-- Check tables needing vacuum
SELECT
    schemaname,
    relname,
    n_dead_tup,
    n_live_tup,
    round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

### Table Maintenance

```sql
-- Rebuild table to reclaim space (alternative to VACUUM FULL)
CREATE TABLE orders_new (LIKE orders INCLUDING ALL);
INSERT INTO orders_new SELECT * FROM orders;
BEGIN;
DROP TABLE orders;
ALTER TABLE orders_new RENAME TO orders;
COMMIT;

-- Online table rewrite with pg_repack
-- pg_repack --table orders mydb

-- Cluster table on index
CLUSTER orders USING idx_orders_created_at;
```

### Schema Changes

```sql
-- Add column with default (instant in PG11+)
ALTER TABLE users ADD COLUMN verified boolean DEFAULT false;

-- Add NOT NULL constraint safely
ALTER TABLE users ADD COLUMN status text;
UPDATE users SET status = 'active' WHERE status IS NULL;
ALTER TABLE users ALTER COLUMN status SET NOT NULL;

-- Create index concurrently (no locks)
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);

-- Drop index concurrently
DROP INDEX CONCURRENTLY idx_users_old;

-- Rename with minimal downtime
ALTER TABLE users RENAME TO users_old;
ALTER TABLE users_new RENAME TO users;
```

---

## Troubleshooting

### Common Issues

```sql
-- Find blocking queries
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query,
    now() - blocked.query_start AS blocked_duration
FROM pg_stat_activity blocked
JOIN pg_locks blocked_locks ON blocked.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON blocked_locks.locktype = blocking_locks.locktype
    AND blocked_locks.database IS NOT DISTINCT FROM blocking_locks.database
    AND blocked_locks.relation IS NOT DISTINCT FROM blocking_locks.relation
    AND blocked_locks.page IS NOT DISTINCT FROM blocking_locks.page
    AND blocked_locks.tuple IS NOT DISTINCT FROM blocking_locks.tuple
    AND blocked_locks.virtualxid IS NOT DISTINCT FROM blocking_locks.virtualxid
    AND blocked_locks.transactionid IS NOT DISTINCT FROM blocking_locks.transactionid
    AND blocked_locks.classid IS NOT DISTINCT FROM blocking_locks.classid
    AND blocked_locks.objid IS NOT DISTINCT FROM blocking_locks.objid
    AND blocked_locks.objsubid IS NOT DISTINCT FROM blocking_locks.objsubid
    AND blocked_locks.pid != blocking_locks.pid
JOIN pg_stat_activity blocking ON blocking.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- Terminate blocking session
SELECT pg_terminate_backend(blocking_pid);

-- Find long-running transactions
SELECT
    pid,
    now() - xact_start AS transaction_duration,
    now() - query_start AS query_duration,
    state,
    query
FROM pg_stat_activity
WHERE xact_start < now() - interval '10 minutes'
  AND state != 'idle'
ORDER BY xact_start;

-- Check table bloat
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
    pg_size_pretty(
        pg_total_relation_size(schemaname || '.' || tablename) -
        pg_relation_size(schemaname || '.' || tablename)
    ) AS bloat_estimate
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 20;

-- Check for corrupted indexes
SELECT
    indexrelid::regclass AS index,
    indisvalid AS valid,
    indisready AS ready
FROM pg_index
WHERE NOT indisvalid OR NOT indisready;

-- Repair invalid index
REINDEX INDEX CONCURRENTLY index_name;
```

### Performance Diagnostics

```sql
-- Current query statistics
SELECT
    pid,
    now() - query_start AS duration,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;

-- I/O timing (requires track_io_timing = on)
SELECT
    datname,
    blk_read_time,
    blk_write_time
FROM pg_stat_database
WHERE datname = current_database();

-- Table access patterns
SELECT
    schemaname,
    relname,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_tup_hot_upd
FROM pg_stat_user_tables
ORDER BY seq_scan DESC
LIMIT 20;
```

---

## Quick Reference

### Essential Commands

```bash
# Connect to database
psql -h localhost -U postgres -d mydb

# Backup
pg_dump -Fc mydb > backup.dump

# Restore
pg_restore -d mydb backup.dump

# Cluster management
pg_ctl status -D /var/lib/postgresql/data
pg_ctl reload -D /var/lib/postgresql/data
pg_ctl restart -D /var/lib/postgresql/data

# Vacuum
vacuumdb --analyze mydb

# Reindex
reindexdb mydb
```

### Key Configuration Parameters

| Parameter | Production Value | Description |
|-----------|-----------------|-------------|
| shared_buffers | 25% of RAM | Shared memory for caching |
| effective_cache_size | 75% of RAM | Planner's memory estimate |
| work_mem | RAM/max_connections/4 | Per-operation memory |
| maintenance_work_mem | 1-2GB | VACUUM, CREATE INDEX |
| max_connections | Based on needs | Limit connections |
| random_page_cost | 1.1 (SSD) | I/O cost estimate |
| effective_io_concurrency | 200 (SSD) | Parallel I/O ops |
| checkpoint_completion_target | 0.9 | Spread checkpoint writes |
| wal_buffers | 64MB | WAL buffer size |
