# MySQL Production Configuration

> Comprehensive guide for running MySQL in production with optimal performance, security, and reliability.

## Table of Contents

1. [Production Configuration](#production-configuration)
2. [Performance Tuning](#performance-tuning)
3. [InnoDB Optimization](#innodb-optimization)
4. [Query Optimization](#query-optimization)
5. [Replication and High Availability](#replication-and-high-availability)
6. [Backup and Recovery](#backup-and-recovery)
7. [Security Hardening](#security-hardening)
8. [Monitoring and Alerting](#monitoring-and-alerting)
9. [Maintenance Operations](#maintenance-operations)

---

## Production Configuration

### my.cnf Essential Settings

```ini
# /etc/mysql/mysql.conf.d/mysqld.cnf
[mysqld]

#------------------------------------------------------------------------------
# BASIC SETTINGS
#------------------------------------------------------------------------------
user                           = mysql
port                           = 3306
bind-address                   = 0.0.0.0
datadir                        = /var/lib/mysql
socket                         = /var/run/mysqld/mysqld.sock
pid-file                       = /var/run/mysqld/mysqld.pid

# Character Set
character-set-server           = utf8mb4
collation-server               = utf8mb4_unicode_ci
init_connect                   = 'SET NAMES utf8mb4'

#------------------------------------------------------------------------------
# CONNECTION SETTINGS
#------------------------------------------------------------------------------
max_connections                = 500
max_connect_errors             = 100000
wait_timeout                   = 600
interactive_timeout            = 600
connect_timeout                = 10
max_allowed_packet             = 256M
thread_cache_size              = 128

#------------------------------------------------------------------------------
# MEMORY SETTINGS (for 16GB RAM server)
#------------------------------------------------------------------------------
# InnoDB Buffer Pool - 70% of RAM for dedicated MySQL server
innodb_buffer_pool_size        = 11G
innodb_buffer_pool_instances   = 8

# Other buffers
key_buffer_size                = 256M          # For MyISAM indexes
sort_buffer_size               = 4M
read_buffer_size               = 2M
read_rnd_buffer_size           = 8M
join_buffer_size               = 4M
tmp_table_size                 = 256M
max_heap_table_size            = 256M

#------------------------------------------------------------------------------
# INNODB SETTINGS
#------------------------------------------------------------------------------
innodb_file_per_table          = ON
innodb_flush_log_at_trx_commit = 1             # ACID compliance
innodb_log_buffer_size         = 64M
innodb_log_file_size           = 1G            # Larger = better write performance
innodb_log_files_in_group      = 2
innodb_flush_method            = O_DIRECT
innodb_io_capacity             = 2000          # For SSD
innodb_io_capacity_max         = 4000
innodb_read_io_threads         = 8
innodb_write_io_threads        = 8
innodb_thread_concurrency      = 0             # Auto
innodb_lock_wait_timeout       = 50
innodb_deadlock_detect         = ON
innodb_print_all_deadlocks     = ON

#------------------------------------------------------------------------------
# LOGGING
#------------------------------------------------------------------------------
log_error                      = /var/log/mysql/error.log
log_error_verbosity            = 2

# Slow query log
slow_query_log                 = ON
slow_query_log_file            = /var/log/mysql/slow.log
long_query_time                = 1
log_queries_not_using_indexes  = ON
log_throttle_queries_not_using_indexes = 10

# Binary logging (for replication)
server-id                      = 1
log_bin                        = /var/log/mysql/mysql-bin
binlog_format                  = ROW
binlog_expire_logs_days        = 7
max_binlog_size                = 1G
sync_binlog                    = 1

# General query log (disable in production)
general_log                    = OFF

#------------------------------------------------------------------------------
# QUERY CACHE (deprecated in MySQL 8.0, use ProxySQL instead)
#------------------------------------------------------------------------------
# query_cache_type             = OFF
# query_cache_size             = 0

#------------------------------------------------------------------------------
# REPLICATION SETTINGS
#------------------------------------------------------------------------------
gtid_mode                      = ON
enforce_gtid_consistency       = ON
log_slave_updates              = ON
relay_log                      = /var/log/mysql/mysql-relay-bin
relay_log_recovery             = ON

#------------------------------------------------------------------------------
# PERFORMANCE SCHEMA
#------------------------------------------------------------------------------
performance_schema             = ON
performance_schema_consumer_events_statements_history_long = ON
```

### Docker Production Configuration

```yaml
# docker-compose.mysql-prod.yml
services:
  mysql:
    image: mysql:8.0
    container_name: mysql-prod
    restart: unless-stopped

    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root_password
      MYSQL_DATABASE: mydb
      MYSQL_USER: app_user
      MYSQL_PASSWORD_FILE: /run/secrets/mysql_app_password

    secrets:
      - mysql_root_password
      - mysql_app_password

    volumes:
      - mysql-data:/var/lib/mysql
      - ./my.cnf:/etc/mysql/conf.d/custom.cnf:ro
      - mysql-logs:/var/log/mysql

    command: >
      --default-authentication-plugin=caching_sha2_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci

    deploy:
      resources:
        limits:
          memory: 12G
          cpus: "4"
        reservations:
          memory: 8G
          cpus: "2"

    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p$$MYSQL_ROOT_PASSWORD"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

    networks:
      - database

    cap_drop:
      - ALL
    cap_add:
      - DAC_OVERRIDE
      - SETGID
      - SETUID

secrets:
  mysql_root_password:
    file: ./secrets/mysql_root_password.txt
  mysql_app_password:
    file: ./secrets/mysql_app_password.txt

volumes:
  mysql-data:
  mysql-logs:

networks:
  database:
    internal: true
```

---

## Performance Tuning

### Memory Calculation

```bash
#!/bin/bash
# mysql-memory-calculator.sh

TOTAL_RAM_GB=${1:-16}
CONNECTIONS=${2:-500}

# InnoDB Buffer Pool: 70% of RAM for dedicated server
BUFFER_POOL=$((TOTAL_RAM_GB * 70 / 100))G

# Per-connection memory (estimate)
PER_CONN_MB=4
TOTAL_CONN_MB=$((CONNECTIONS * PER_CONN_MB))

echo "For ${TOTAL_RAM_GB}GB RAM with ${CONNECTIONS} connections:"
echo "  innodb_buffer_pool_size = ${BUFFER_POOL}"
echo "  innodb_buffer_pool_instances = $((TOTAL_RAM_GB / 2))"
echo "  Estimated per-connection memory = ${TOTAL_CONN_MB}MB"
echo ""
echo "Recommended settings:"
echo "  key_buffer_size = 256M"
echo "  sort_buffer_size = 4M"
echo "  read_buffer_size = 2M"
echo "  join_buffer_size = 4M"
echo "  tmp_table_size = 256M"
```

### Status Variables Analysis

```sql
-- Check buffer pool usage
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';

-- Buffer pool hit ratio (should be > 99%)
SELECT
    (1 - (
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    )) * 100 AS buffer_pool_hit_ratio;

-- Connection usage
SELECT
    @@max_connections AS max_connections,
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Threads_connected') AS current_connections,
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Max_used_connections') AS max_used_connections;

-- Table cache usage
SHOW GLOBAL STATUS LIKE 'Open%tables%';
SHOW GLOBAL STATUS LIKE 'Opened_tables';

-- Temporary tables
SELECT
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Created_tmp_disk_tables') /
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Created_tmp_tables') * 100
    AS tmp_disk_table_ratio;
```

---

## InnoDB Optimization

### InnoDB Configuration for SSD

```ini
[mysqld]
# SSD-optimized settings
innodb_io_capacity             = 2000
innodb_io_capacity_max         = 4000
innodb_flush_neighbors         = 0       # Disable for SSD
innodb_read_ahead_threshold    = 0       # Disable for SSD

# Parallelism
innodb_read_io_threads         = 8
innodb_write_io_threads        = 8
innodb_purge_threads           = 4

# Checkpoints
innodb_max_dirty_pages_pct     = 75
innodb_max_dirty_pages_pct_lwm = 10
```

### InnoDB Monitoring

```sql
-- InnoDB engine status
SHOW ENGINE INNODB STATUS\G

-- InnoDB metrics
SELECT NAME, COUNT, STATUS
FROM information_schema.INNODB_METRICS
WHERE STATUS = 'enabled'
ORDER BY COUNT DESC
LIMIT 20;

-- Buffer pool pages
SELECT
    POOL_ID,
    POOL_SIZE,
    FREE_BUFFERS,
    DATABASE_PAGES,
    MODIFIED_DATABASE_PAGES
FROM information_schema.INNODB_BUFFER_POOL_STATS;

-- Row lock waits
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- Transaction history
SELECT
    trx_id,
    trx_state,
    trx_started,
    trx_rows_locked,
    trx_rows_modified,
    trx_query
FROM information_schema.INNODB_TRX
ORDER BY trx_started;
```

---

## Query Optimization

### Index Strategies

```sql
-- Composite indexes (leftmost prefix rule)
CREATE INDEX idx_orders_user_status_date ON orders(user_id, status, created_at);
-- Supports: WHERE user_id = ?
--           WHERE user_id = ? AND status = ?
--           WHERE user_id = ? AND status = ? AND created_at > ?

-- Covering index
CREATE INDEX idx_users_email_active ON users(email, is_active, id);

-- Full-text index
ALTER TABLE articles ADD FULLTEXT INDEX ft_content (title, body);

-- Prefix index for long strings
CREATE INDEX idx_url_prefix ON pages(url(100));

-- Invisible index (test before dropping)
ALTER TABLE users ALTER INDEX idx_email INVISIBLE;
-- Test queries
ALTER TABLE users ALTER INDEX idx_email VISIBLE;  -- or drop if not needed
```

### Query Analysis

```sql
-- Explain with analyze
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123 AND status = 'completed';

-- JSON format for detailed info
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE user_id = 123;

-- Check index usage for a table
SHOW INDEX FROM orders;

-- Unused indexes
SELECT
    object_schema,
    object_name,
    index_name,
    count_star AS query_count
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
  AND count_star = 0
  AND object_schema = 'mydb'
ORDER BY object_name, index_name;
```

### Query Optimization Patterns

```sql
-- Use EXISTS instead of IN for subqueries
-- Slow
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders WHERE total > 1000);

-- Fast
SELECT * FROM users u WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.total > 1000
);

-- Pagination with keyset (faster than OFFSET)
-- Slow for large offsets
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 100000;

-- Fast with keyset
SELECT * FROM orders WHERE id > 100000 ORDER BY id LIMIT 20;

-- Batch processing
UPDATE orders
SET processed = 1
WHERE id IN (
    SELECT id FROM (
        SELECT id FROM orders WHERE processed = 0 LIMIT 1000
    ) tmp
);

-- Or use prepared statements in application code with LIMIT
```

---

## Replication and High Availability

### Master-Slave Replication Setup

```sql
-- On Master: Create replication user
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'secure_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';
FLUSH PRIVILEGES;

-- Get binary log position
SHOW MASTER STATUS;
```

```sql
-- On Slave: Configure replication
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='master_host',
    SOURCE_PORT=3306,
    SOURCE_USER='repl_user',
    SOURCE_PASSWORD='secure_password',
    SOURCE_AUTO_POSITION=1;  -- For GTID-based replication

START REPLICA;
SHOW REPLICA STATUS\G
```

### Monitoring Replication

```sql
-- Check replication status
SHOW REPLICA STATUS\G

-- Key metrics to check:
-- Replica_IO_Running: Yes
-- Replica_SQL_Running: Yes
-- Seconds_Behind_Source: 0 (or low)

-- Check replication lag
SELECT
    @master := (SELECT VARIABLE_VALUE FROM performance_schema.global_status
                WHERE VARIABLE_NAME = 'Seconds_Behind_Master') AS seconds_behind;
```

### MySQL Group Replication

```ini
# my.cnf for Group Replication
[mysqld]
# Group Replication settings
plugin_load_add                    = group_replication.so
group_replication_group_name       = "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
group_replication_start_on_boot    = OFF
group_replication_local_address    = "node1:33061"
group_replication_group_seeds      = "node1:33061,node2:33061,node3:33061"
group_replication_bootstrap_group  = OFF
group_replication_single_primary_mode = ON

# Required settings
disabled_storage_engines           = "MyISAM,BLACKHOLE,FEDERATED,ARCHIVE"
gtid_mode                          = ON
enforce_gtid_consistency           = ON
binlog_checksum                    = NONE
```

```sql
-- Bootstrap first node
SET GLOBAL group_replication_bootstrap_group = ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group = OFF;

-- Join other nodes
START GROUP_REPLICATION;

-- Check group status
SELECT * FROM performance_schema.replication_group_members;
```

---

## Backup and Recovery

### mysqldump for Logical Backups

```bash
#!/bin/bash
# backup-mysql.sh
set -euo pipefail

BACKUP_DIR="/backups/mysql"
RETENTION_DAYS=30
DATE=$(date +%Y-%m-%d_%H%M%S)
MYSQL_USER="${MYSQL_USER:-backup}"
MYSQL_PASSWORD="${MYSQL_PASSWORD}"
MYSQL_HOST="${MYSQL_HOST:-localhost}"
DATABASE="${DATABASE:-mydb}"

mkdir -p "$BACKUP_DIR"

# Full backup with routines and triggers
mysqldump \
    -h "$MYSQL_HOST" \
    -u "$MYSQL_USER" \
    -p"$MYSQL_PASSWORD" \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --set-gtid-purged=OFF \
    --quick \
    --lock-tables=false \
    "$DATABASE" | gzip > "$BACKUP_DIR/${DATABASE}_${DATE}.sql.gz"

# Verify backup
gunzip -t "$BACKUP_DIR/${DATABASE}_${DATE}.sql.gz"

# Checksum
sha256sum "$BACKUP_DIR/${DATABASE}_${DATE}.sql.gz" > "$BACKUP_DIR/${DATABASE}_${DATE}.sql.gz.sha256"

# Upload to S3
if command -v aws &> /dev/null; then
    aws s3 cp "$BACKUP_DIR/${DATABASE}_${DATE}.sql.gz" "s3://my-backups/mysql/${DATE}/"
fi

# Cleanup old backups
find "$BACKUP_DIR" -type f -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: ${DATABASE}_${DATE}.sql.gz"
```

### Percona XtraBackup for Physical Backups

```bash
#!/bin/bash
# xtrabackup.sh - Physical backup (no locks, faster restore)

# Full backup
xtrabackup --backup \
    --target-dir=/backups/full \
    --user=backup \
    --password=secret

# Prepare backup
xtrabackup --prepare --target-dir=/backups/full

# Incremental backup
xtrabackup --backup \
    --target-dir=/backups/inc1 \
    --incremental-basedir=/backups/full \
    --user=backup \
    --password=secret

# Prepare incremental
xtrabackup --prepare --apply-log-only --target-dir=/backups/full
xtrabackup --prepare --target-dir=/backups/full --incremental-dir=/backups/inc1

# Restore
systemctl stop mysql
rm -rf /var/lib/mysql/*
xtrabackup --copy-back --target-dir=/backups/full
chown -R mysql:mysql /var/lib/mysql
systemctl start mysql
```

### Point-in-Time Recovery

```bash
# Enable binary logging in my.cnf
# log_bin = /var/log/mysql/mysql-bin
# binlog_expire_logs_days = 7

# List binary logs
mysqlbinlog --no-defaults /var/log/mysql/mysql-bin.000001 | head -50

# Restore to specific point in time
mysql -u root -p < /backups/full_backup.sql

mysqlbinlog \
    --no-defaults \
    --stop-datetime="2024-01-15 14:30:00" \
    /var/log/mysql/mysql-bin.000001 \
    /var/log/mysql/mysql-bin.000002 | mysql -u root -p
```

---

## Security Hardening

### User Management

```sql
-- Create application user with minimal privileges
CREATE USER 'app_user'@'10.0.1.%' IDENTIFIED BY 'secure_password';
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'app_user'@'10.0.1.%';

-- Create read-only user
CREATE USER 'readonly'@'%' IDENTIFIED BY 'readonly_password';
GRANT SELECT ON mydb.* TO 'readonly'@'%';

-- Create backup user
CREATE USER 'backup'@'localhost' IDENTIFIED BY 'backup_password';
GRANT SELECT, SHOW VIEW, TRIGGER, LOCK TABLES, RELOAD, PROCESS ON *.* TO 'backup'@'localhost';

-- Remove default accounts
DROP USER IF EXISTS ''@'localhost';
DROP USER IF EXISTS ''@'%';
DROP USER IF EXISTS 'root'@'%';

-- Set password expiration
ALTER USER 'app_user'@'10.0.1.%' PASSWORD EXPIRE INTERVAL 90 DAY;

-- Require SSL
ALTER USER 'app_user'@'10.0.1.%' REQUIRE SSL;

-- View user privileges
SHOW GRANTS FOR 'app_user'@'10.0.1.%';
```

### Secure Configuration

```ini
[mysqld]
# Disable remote root login
skip-grant-tables            = OFF

# Require secure transport
require_secure_transport     = ON

# SSL configuration
ssl-ca                       = /etc/mysql/ssl/ca.pem
ssl-cert                     = /etc/mysql/ssl/server-cert.pem
ssl-key                      = /etc/mysql/ssl/server-key.pem
tls_version                  = TLSv1.2,TLSv1.3

# Disable dangerous features
local_infile                 = OFF
symbolic-links               = OFF
skip-symbolic-links

# Password validation
validate_password.policy     = STRONG
validate_password.length     = 14
validate_password.mixed_case_count = 1
validate_password.number_count = 1
validate_password.special_char_count = 1
```

### Audit Logging

```ini
# Install audit plugin
[mysqld]
plugin-load-add              = audit_log.so
audit_log_file               = /var/log/mysql/audit.log
audit_log_format             = JSON
audit_log_policy             = LOGINS
audit_log_rotate_on_size     = 100M
```

---

## Monitoring and Alerting

### Key Metrics to Monitor

```sql
-- Connection metrics
SHOW GLOBAL STATUS WHERE Variable_name IN (
    'Threads_connected',
    'Threads_running',
    'Max_used_connections',
    'Aborted_connects',
    'Aborted_clients'
);

-- Query metrics
SHOW GLOBAL STATUS WHERE Variable_name IN (
    'Questions',
    'Slow_queries',
    'Com_select',
    'Com_insert',
    'Com_update',
    'Com_delete'
);

-- InnoDB metrics
SHOW GLOBAL STATUS WHERE Variable_name IN (
    'Innodb_buffer_pool_reads',
    'Innodb_buffer_pool_read_requests',
    'Innodb_row_lock_waits',
    'Innodb_row_lock_time_avg',
    'Innodb_deadlocks'
);

-- Replication metrics
SHOW REPLICA STATUS\G
```

### Prometheus MySQL Exporter

```yaml
# docker-compose.monitoring.yml
services:
  mysql-exporter:
    image: prom/mysqld-exporter:latest
    environment:
      DATA_SOURCE_NAME: "exporter:password@(mysql:3306)/"
    ports:
      - "9104:9104"
    command:
      - '--collect.auto_increment.columns'
      - '--collect.binlog_size'
      - '--collect.global_status'
      - '--collect.global_variables'
      - '--collect.info_schema.innodb_metrics'
      - '--collect.info_schema.processlist'
      - '--collect.info_schema.tables'
      - '--collect.slave_status'
```

### Alert Rules

```yaml
# prometheus/alerts/mysql.yml
groups:
  - name: mysql-alerts
    rules:
      - alert: MySQLDown
        expr: mysql_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MySQL is down"

      - alert: MySQLHighConnections
        expr: mysql_global_status_threads_connected / mysql_global_variables_max_connections > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MySQL connections > 80%"

      - alert: MySQLReplicationLag
        expr: mysql_slave_status_seconds_behind_master > 60
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MySQL replication lag > 60s"

      - alert: MySQLSlowQueries
        expr: rate(mysql_global_status_slow_queries[5m]) > 0.1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "MySQL slow queries detected"
```

---

## Maintenance Operations

### OPTIMIZE and ANALYZE

```sql
-- Analyze table statistics
ANALYZE TABLE orders;

-- Optimize table (reclaim space, rebuild indexes)
OPTIMIZE TABLE orders;

-- For InnoDB, OPTIMIZE does:
ALTER TABLE orders ENGINE=InnoDB;

-- Online DDL
ALTER TABLE orders ADD INDEX idx_status (status), ALGORITHM=INPLACE, LOCK=NONE;
```

### Maintenance Script

```bash
#!/bin/bash
# mysql-maintenance.sh

MYSQL_USER="maintenance"
MYSQL_PASSWORD="password"
DATABASE="mydb"

# Get list of tables
TABLES=$(mysql -u $MYSQL_USER -p"$MYSQL_PASSWORD" -N -e \
    "SELECT table_name FROM information_schema.tables WHERE table_schema='$DATABASE' AND engine='InnoDB'")

for table in $TABLES; do
    echo "Analyzing $table..."
    mysql -u $MYSQL_USER -p"$MYSQL_PASSWORD" -e "ANALYZE TABLE $DATABASE.$table"

    echo "Optimizing $table..."
    mysql -u $MYSQL_USER -p"$MYSQL_PASSWORD" -e "OPTIMIZE TABLE $DATABASE.$table"
done

# Check for table problems
mysqlcheck -u $MYSQL_USER -p"$MYSQL_PASSWORD" --analyze --optimize $DATABASE
```

### Managing Large Tables

```sql
-- Partition large tables
ALTER TABLE logs
PARTITION BY RANGE (TO_DAYS(created_at)) (
    PARTITION p202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
    PARTITION p202402 VALUES LESS THAN (TO_DAYS('2024-03-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Drop old partition (instant)
ALTER TABLE logs DROP PARTITION p202401;

-- Add new partition
ALTER TABLE logs REORGANIZE PARTITION p_future INTO (
    PARTITION p202403 VALUES LESS THAN (TO_DAYS('2024-04-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Archive old data
INSERT INTO logs_archive SELECT * FROM logs WHERE created_at < '2024-01-01';
DELETE FROM logs WHERE created_at < '2024-01-01';
```

---

## Quick Reference

### Essential Commands

```bash
# Connect
mysql -u user -p -h host database

# Backup
mysqldump -u user -p database > backup.sql

# Restore
mysql -u user -p database < backup.sql

# Check tables
mysqlcheck -u user -p --check database

# Show processes
mysqladmin -u user -p processlist

# Kill query
mysqladmin -u user -p kill <process_id>
```

### Key Configuration Parameters

| Parameter | Production Value | Description |
|-----------|-----------------|-------------|
| innodb_buffer_pool_size | 70% of RAM | Main memory cache |
| max_connections | Based on needs | Max concurrent connections |
| innodb_flush_log_at_trx_commit | 1 | ACID compliance |
| sync_binlog | 1 | Durable binary log |
| innodb_io_capacity | 2000 (SSD) | I/O operations/sec |
| slow_query_log | ON | Log slow queries |
| long_query_time | 1 | Slow query threshold |
