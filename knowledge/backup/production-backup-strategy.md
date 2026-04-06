# Production Backup Strategy

## Opinionated Senior Sysadmin Notes

Let's get one thing out of the way immediately: **a backup you have never restored from is not a backup — it's a hope.** The entire point of this document is to get you to the point where you have tested, verified, encrypted, offsite backups that you have actually restored from. Everything else is commentary.

I've watched teams spend weeks building elaborate backup pipelines, only to discover during an actual incident that the restore process was broken, the GPG key was lost, or the S3 bucket policy blocked reads. Test restores are not optional. They are the backup. The dump files are just artifacts.

This document is opinionated. Where there are multiple ways to do something, I'll tell you which one I use and why. Feel free to deviate — but know why you're deviating.

---

## Table of Contents

1. [The 3-2-1 Rule](#the-3-2-1-rule)
2. [RPO and RTO Planning](#rpo-and-rto-planning)
3. [PostgreSQL Backups](#postgresql-backups)
4. [MongoDB Backups](#mongodb-backups)
5. [MySQL / MariaDB Backups](#mysql--mariadb-backups)
6. [Filesystem Backups with rsync](#filesystem-backups-with-rsync)
7. [Cloud Upload with rclone](#cloud-upload-with-rclone)
8. [GPG Encryption](#gpg-encryption)
9. [Complete Backup Script](#complete-backup-script)
10. [IAM Policy for S3 Backup User](#iam-policy-for-s3-backup-user)
11. [Restore Runbook](#restore-runbook)
12. [Backup Verification Checklist](#backup-verification-checklist)
13. [Alerting and Monitoring](#alerting-and-monitoring)
14. [Common Mistakes and How to Avoid Them](#common-mistakes-and-how-to-avoid-them)

---

## The 3-2-1 Rule

The 3-2-1 rule is the minimum viable backup strategy. If you are not meeting all three criteria, you do not have a real backup strategy — you have a wishful thinking strategy.

### What 3-2-1 Means

**3 copies of your data**

- Copy 1: The live production data (your database, your filesystem)
- Copy 2: A local backup — fast to restore, useful for day-to-day mistakes
- Copy 3: An offsite backup — survives datacenter fires, ransomware, hardware failures

**2 different storage media**

This is the most commonly misunderstood requirement. "Different media" means different storage technology or at least different failure domains. Examples of valid combinations:
- NVMe SSD (production) + spinning disk NAS (local backup) + object storage (offsite)
- EBS volume + another EBS volume in a different AZ + S3 (this is borderline — S3 is truly different media)
- Local disk + tape + cloud

What does NOT count as different media:
- Two directories on the same disk
- Two EBS volumes in the same availability zone attached to the same instance
- RAID arrays — RAID is not a backup

**1 offsite copy**

"Offsite" means physically separate from your primary location. Cloud object storage (S3, Backblaze B2, Wasabi) is the standard answer for most teams. It should be in a different geographic region, not just a different availability zone.

### Why 3-2-1 Is a Minimum, Not a Target

For most production systems, I push teams toward 3-2-1-1-0:
- 3 copies
- 2 different media
- 1 offsite
- 1 air-gapped or immutable copy (S3 Object Lock, B2 Object Lock, or literal offline tape)
- 0 errors on verify (automated restore testing passes)

Ransomware specifically targets backup infrastructure. If your backup server is mounted and writable from a compromised machine, ransomware will encrypt your backups too. Immutable storage — where objects cannot be deleted or overwritten for a defined period — is the mitigation.

### Implementation Checklist

- [ ] Local backup running daily (pg_dump, mongodump, mysqldump)
- [ ] Offsite upload running daily (rclone to S3 or B2)
- [ ] Encryption applied before upload (GPG, see below)
- [ ] S3 Object Lock or B2 File Lock enabled on backup bucket
- [ ] Backup user has write-only or append-only IAM permissions (cannot delete)
- [ ] Separate restoration user has read permissions
- [ ] Monthly test restore performed and documented

---

## RPO and RTO Planning

Before you write a single line of backup script, you need to have an honest conversation with stakeholders about two numbers. Everything else flows from these.

### Recovery Point Objective (RPO)

RPO is the maximum acceptable age of data you can recover. It answers: **how much data can we afford to lose?**

Examples:
- RPO = 0: No data loss tolerated. Requires synchronous replication, not backups alone.
- RPO = 1 hour: You can lose up to 1 hour of transactions.
- RPO = 24 hours: Daily backups are sufficient.

RPO drives backup **frequency**.

| RPO | Backup approach |
|-----|----------------|
| 0 (zero data loss) | Synchronous replication + WAL shipping |
| < 5 minutes | Streaming replication + WAL archiving (PITR) |
| < 1 hour | Hourly pg_dump + mongodump |
| < 24 hours | Daily backups |
| < 1 week | Weekly backups (probably not acceptable for production) |

**My opinionated default for most production systems:** RPO = 1 hour, achieved via hourly dumps of critical databases, continuous WAL archiving for PostgreSQL, and daily full backups uploaded offsite. This gives you point-in-time recovery within PostgreSQL and hourly granularity for everything else.

### Recovery Time Objective (RTO)

RTO is the maximum acceptable downtime. It answers: **how long can we be down?**

Examples:
- RTO = 15 minutes: You need hot standby, automated failover, and a pre-configured restore environment.
- RTO = 4 hours: Manual restore process is acceptable, but must be documented and practiced.
- RTO = 24 hours: You have time to provision new infrastructure during the recovery.

RTO drives restore **speed** and **automation**.

| RTO | Required infrastructure |
|-----|------------------------|
| < 1 minute | Active-active multi-region, no restore needed |
| < 15 minutes | Hot standby with automated failover |
| < 1 hour | Warm standby + documented restore runbook + recent local backup |
| < 4 hours | Manual restore from local backup + tested runbook |
| < 24 hours | Restore from offsite backup |

**My opinionated default:** RTO = 2 hours for most production systems. This means a warm standby for databases, a local copy of the most recent backup, and a restore runbook that has been executed within the last 30 days. Not the last 30 days on paper — actually executed.

### Documenting Your SLAs

Write these numbers down in a document that stakeholders have reviewed and signed off on. This is not optional. When an incident happens at 3 AM, you need to know whether you are racing against a 15-minute RTO or a 4-hour RTO. That changes every decision you make.

```
# Service: api-production
# Database: PostgreSQL 16 (primary), MongoDB 7 (events)
# RPO: 1 hour
# RTO: 2 hours
# Backup schedule: hourly (PostgreSQL), daily (MongoDB)
# Offsite upload: daily at 03:00 UTC
# Last test restore: 2026-03-15 — passed, took 47 minutes
# Contact during incident: sysadmin@company.com, +1-555-0100
```

---

## PostgreSQL Backups

PostgreSQL has the richest backup tooling of any open-source database. Use it. There is no excuse for not having working PostgreSQL backups.

### pg_dump: Plain Text Format

```bash
pg_dump -U postgres -h localhost mydb > /backup/mydb_$(date +%Y%m%d_%H%M%S).sql
```

**When to use plain text:**
- When you need a human-readable dump you can inspect with a text editor
- When you need maximum portability across PostgreSQL versions
- When you need to apply the backup to a different database system
- For schema-only exports you'll edit by hand

**When NOT to use plain text:**
- Large databases — text dumps are slow and large
- When you need parallel restore — plain text cannot use pg_restore at all
- For daily operational backups — use custom format instead

Plain text dumps can only be restored with psql:
```bash
psql -U postgres -h localhost mydb < /backup/mydb_20260101_030000.sql
```

### pg_dump: Custom Format (-Fc)

This is my default for production database backups.

```bash
pg_dump -U postgres -h localhost -Fc mydb > /backup/mydb_$(date +%Y%m%d_%H%M%S).dump
```

**Why custom format:**
- Compressed by default (uses zlib internally)
- Supports parallel restore with `pg_restore -j`
- Supports selective restore (restore only specific tables, schemas)
- Faster than plain text
- Can be inspected with `pg_restore -l` without actually restoring

Inspect the table of contents:
```bash
pg_restore -l /backup/mydb_20260101_030000.dump
```

Restore with parallel workers:
```bash
pg_restore -U postgres -h localhost -d mydb -j 8 -c /backup/mydb_20260101_030000.dump
```

The `-c` flag drops existing objects before recreating them. Use it when restoring into an existing database that has stale schema. Omit it when restoring into a fresh empty database.

### pg_dump: Directory Format (-Fd) with Parallel Dump

```bash
pg_dump -U postgres -h localhost -Fd -j 4 mydb -f /backup/mydb_$(date +%Y%m%d_%H%M%S)/
```

**When to use directory format:**
- Large databases where parallel dump significantly reduces backup window
- When you want each table dumped as a separate file (allows selective restore)
- When you want to checkpoint and resume a dump

`-j 4` uses 4 parallel workers. Each worker handles one table at a time. For a database with many large tables, this can cut your backup time by 3-4x.

**Caveat:** Directory dumps produce a directory tree, not a single file. This complicates encryption and upload. My approach is to tar the directory first:

```bash
tar -czf /backup/mydb_$(date +%Y%m%d_%H%M%S).tar.gz /backup/mydb_dump_dir/
```

Restore a directory format dump with parallel workers:
```bash
pg_restore -U postgres -h localhost -d mydb -j 8 -c /backup/mydb_20260101_030000/
```

### pg_restore Options Reference

```bash
pg_restore [options] filename

Key options:
  -d dbname       Target database
  -j N            Parallel restore with N workers (only works with -Fc or -Fd)
  -c              Drop objects before recreating (--clean)
  -C              Create the database from the dump (--create)
  -e              Exit on first error (--exit-on-error) — use this in scripts
  -s              Schema only, no data
  -a              Data only, no schema
  -t tablename    Restore only this table
  -n schema       Restore only this schema
  -U username     Connect as this user
  -h hostname     Connect to this host
  -p port         Connect on this port
  -v              Verbose output
  --no-owner      Skip ownership commands (useful when restoring as different user)
  --no-privileges Skip GRANT/REVOKE commands
  --if-exists     Use IF EXISTS when dropping objects (avoids errors on clean)
```

**Recommended restore command for production:**
```bash
pg_restore \
  -U postgres \
  -h localhost \
  -d mydb \
  -j 8 \
  -c \
  --if-exists \
  -e \
  -v \
  /backup/mydb_20260101_030000.dump \
  2>&1 | tee /var/log/restore_$(date +%Y%m%d_%H%M%S).log
```

Always tee the output to a log file. When a restore fails partway through, you need to know exactly where it stopped.

### WAL Archiving and Point-in-Time Recovery (PITR)

For RPO under 5 minutes, pg_dump is not enough. You need continuous WAL archiving.

WAL (Write-Ahead Log) is PostgreSQL's transaction log. By archiving WAL segments continuously, you can replay them on top of a base backup to recover to any point in time.

**postgresql.conf settings:**
```ini
wal_level = replica
archive_mode = on
archive_command = 'rclone copy %p s3-backup:my-backup-bucket/wal/%f'
archive_timeout = 300  # Archive at least every 5 minutes even if no WAL activity
```

**Creating a base backup:**
```bash
pg_basebackup -U replicator -h localhost -D /backup/basebackup -Ft -z -P
```

**Restoring to a point in time:**

1. Stop PostgreSQL
2. Replace data directory with base backup
3. Create `recovery.signal` in data directory
4. Set `recovery_target_time` in `postgresql.conf`
5. Configure `restore_command` to retrieve WAL from archive
6. Start PostgreSQL — it replays WAL up to the target time

```ini
# In postgresql.conf during recovery
restore_command = 'rclone copy s3-backup:my-backup-bucket/wal/%f %p'
recovery_target_time = '2026-04-05 14:30:00 UTC'
recovery_target_action = 'promote'
```

WAL archiving is complex to set up correctly. Consider using pgBackRest or Barman for production PITR — they handle the complexity for you and are well-tested.

### PostgreSQL Backup Sizing

Always verify your dump size is reasonable. A sudden 50% size decrease in your dumps is a warning sign that something is wrong with the database or the dump process.

```bash
# Compare today's dump to yesterday's
ls -lh /backup/mydb_*.dump | tail -5
```

---

## MongoDB Backups

### mongodump / mongorestore

```bash
# Full database dump
mongodump \
  --host localhost \
  --port 27017 \
  --username backupuser \
  --password "$(cat /etc/backup/mongo_password)" \
  --authenticationDatabase admin \
  --db mydb \
  --out /backup/mongo/mydb_$(date +%Y%m%d_%H%M%S)

# Dump with compression (produces .bson.gz files)
mongodump \
  --host localhost \
  --port 27017 \
  --username backupuser \
  --password "$(cat /etc/backup/mongo_password)" \
  --authenticationDatabase admin \
  --db mydb \
  --gzip \
  --out /backup/mongo/mydb_$(date +%Y%m%d_%H%M%S)
```

**Restoring:**
```bash
mongorestore \
  --host localhost \
  --port 27017 \
  --username adminuser \
  --password "$(cat /etc/backup/mongo_admin_password)" \
  --authenticationDatabase admin \
  --db mydb \
  --gzip \
  --drop \
  /backup/mongo/mydb_20260101_030000/mydb/
```

The `--drop` flag drops existing collections before restoring. Use it when restoring into a database with stale data.

### MongoDB Backup Caveats

**mongodump is not consistent for sharded clusters.** For sharded MongoDB, you must either:
1. Use MongoDB Atlas backups (if using Atlas)
2. Use `mongodump` with the mongos process and `--oplog` for consistency
3. Stop the balancer and dump each shard individually

For replica sets, mongodump is consistent at the point of connection. No additional steps needed.

**Replica set dump with oplog for point-in-time recovery:**
```bash
mongodump \
  --host rs0/mongo1:27017,mongo2:27017,mongo3:27017 \
  --username backupuser \
  --password "$(cat /etc/backup/mongo_password)" \
  --authenticationDatabase admin \
  --oplog \
  --gzip \
  --out /backup/mongo/full_$(date +%Y%m%d_%H%M%S)
```

The `--oplog` flag includes the oplog in the dump, which allows you to replay operations that happened during the dump window, giving you a consistent snapshot.

**MongoDB Atlas:** If you use Atlas, use Atlas Cloud Backups instead of mongodump. Atlas provides consistent backups with PITR and handles the infrastructure. mongodump should be your fallback for self-hosted.

### MongoDB Backup User Setup

Create a dedicated backup user with minimal permissions:

```javascript
use admin
db.createUser({
  user: "backupuser",
  pwd: "strong-random-password-here",
  roles: [
    { role: "backup", db: "admin" }
  ]
})
```

The `backup` built-in role provides `mongodump` access without admin privileges.

---

## MySQL / MariaDB Backups

### mysqldump with Production-Safe Options

The minimal acceptable mysqldump command for a production InnoDB database:

```bash
mysqldump \
  --user=backupuser \
  --password="$(cat /etc/backup/mysql_password)" \
  --host=localhost \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --hex-blob \
  --default-character-set=utf8mb4 \
  mydb > /backup/mysql/mydb_$(date +%Y%m%d_%H%M%S).sql
```

**Option breakdown:**

`--single-transaction`: Wraps the dump in a single transaction, providing a consistent snapshot without locking tables. **Required for InnoDB.** Without this, you get an inconsistent dump. Never dump InnoDB without `--single-transaction`.

`--routines`: Includes stored procedures and functions. Without this, your restore will be missing application logic.

`--triggers`: Includes trigger definitions. Without this, triggers are missing on restore.

`--events`: Includes scheduled events. Without this, cron-like database jobs are missing on restore.

`--hex-blob`: Exports BLOB fields as hexadecimal. Prevents encoding issues in text dumps.

`--default-character-set=utf8mb4`: Ensures correct character encoding throughout the dump.

**Dump all databases with metadata:**
```bash
mysqldump \
  --user=root \
  --password="$(cat /etc/backup/mysql_root_password)" \
  --all-databases \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --hex-blob \
  --master-data=2 \
  > /backup/mysql/all_databases_$(date +%Y%m%d_%H%M%S).sql
```

`--master-data=2` includes the binary log position as a comment, enabling point-in-time recovery. Use `--source-data=2` for MySQL 8.0.26+.

**Restoring:**
```bash
mysql \
  --user=root \
  --password="$(cat /etc/backup/mysql_root_password)" \
  --host=localhost \
  mydb < /backup/mysql/mydb_20260101_030000.sql
```

Or for all databases:
```bash
mysql \
  --user=root \
  --password="$(cat /etc/backup/mysql_root_password)" \
  --host=localhost \
  < /backup/mysql/all_databases_20260101_030000.sql
```

### MySQL Backup User

```sql
CREATE USER 'backupuser'@'localhost' IDENTIFIED BY 'strong-random-password';
GRANT SELECT, SHOW VIEW, TRIGGER, LOCK TABLES, EVENT, PROCESS ON *.* TO 'backupuser'@'localhost';
GRANT RELOAD ON *.* TO 'backupuser'@'localhost';
FLUSH PRIVILEGES;
```

### MySQL Binary Log Backup

For PITR, enable binary logging and archive the binary logs:

```ini
# /etc/mysql/mysql.conf.d/mysqld.cnf
[mysqld]
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = ROW
expire_logs_days = 7
sync_binlog = 1
```

Archive binary logs separately from dumps:
```bash
# Copy new binary logs to backup location
rsync -a /var/log/mysql/mysql-bin.* /backup/mysql/binlogs/
```

---

## Filesystem Backups with rsync

rsync is the workhorse of filesystem backups. Used correctly, it produces efficient incremental backups with nearly the same disk usage as a single full backup.

### Basic rsync Backup

```bash
rsync -avz --delete \
  /var/www/html/ \
  /backup/filesystem/www/
```

`-a`: Archive mode (recursive, preserves permissions, timestamps, symlinks, owner, group)
`-v`: Verbose
`-z`: Compress during transfer (useful for SSH, wasteful for local disks)
`--delete`: Delete files in destination that no longer exist in source

### --link-dest: Incremental Backups with Hardlinks

This is the rsync technique that makes incremental backups efficient. Each daily backup appears to be a full copy but only uses disk space for files that changed.

```bash
#!/bin/bash
BACKUP_ROOT="/backup/filesystem"
SOURCE="/var/www/html"
DATE=$(date +%Y%m%d_%H%M%S)
DEST="${BACKUP_ROOT}/${DATE}"
LINK="${BACKUP_ROOT}/latest"

rsync -avz \
  --link-dest="${LINK}" \
  --delete \
  "${SOURCE}/" \
  "${DEST}/"

# Update the 'latest' symlink
ln -sfn "${DEST}" "${LINK}"
```

How `--link-dest` works:
1. rsync compares the source with `--link-dest` (yesterday's backup)
2. Files that are identical (same content, same mtime) are hardlinked from the previous backup directory
3. Changed and new files are copied fresh
4. The result: each backup directory looks like a complete copy, but unchanged files share disk blocks

Disk usage comparison:
- Without `--link-dest`: 100GB × 30 days = 3TB
- With `--link-dest` (assuming 5% daily change rate): 100GB + (5GB × 29) = ~245GB

### --bwlimit: Bandwidth Throttling

When running backups over a shared network link, throttle bandwidth to avoid impacting production traffic:

```bash
rsync -avz \
  --bwlimit=50000 \
  --link-dest="${LINK}" \
  "${SOURCE}/" \
  "${DEST}/"
```

`--bwlimit=50000` limits transfer to 50,000 KiB/s (~50 MB/s). Adjust based on your link capacity.

**Schedule bandwidth limits by time of day:**
```bash
# In crontab — full speed at night, limited during day
# 02:00 backup — no limit
0 2 * * * /usr/local/bin/backup.sh
# 14:00 incremental — limited
0 14 * * * rsync --bwlimit=10000 ...
```

### SSH Transport

rsync over SSH is the standard for remote backups:

```bash
rsync -avz \
  -e "ssh -i /root/.ssh/backup_key -o StrictHostKeyChecking=yes" \
  --link-dest=/backup/latest \
  /var/www/html/ \
  backupuser@backup-server.example.com:/backup/webserver/
```

**SSH key setup for backup:**
```bash
# On backup server: create dedicated backup user
useradd -r -m -s /bin/bash backupuser

# On source server: generate dedicated backup key (no passphrase for automated use)
ssh-keygen -t ed25519 -f /root/.ssh/backup_key -N "" -C "backup@webserver"

# On backup server: authorize the key
mkdir -p /home/backupuser/.ssh
cat /root/.ssh/backup_key.pub >> /home/backupuser/.ssh/authorized_keys
chmod 700 /home/backupuser/.ssh
chmod 600 /home/backupuser/.ssh/authorized_keys
```

**Restrict the SSH key to rsync only:**

In `authorized_keys`, prefix the key with restrictions:
```
command="rrsync /backup/webserver/",restrict ssh-ed25519 AAAAC3Nz... backup@webserver
```

`rrsync` is a restricted rsync wrapper that limits the key to rsync operations in the specified directory. This is proper security hygiene — a compromised source server cannot use the backup key to do anything other than push files to the designated backup directory.

### Exclude Patterns

```bash
rsync -avz \
  --exclude='*.log' \
  --exclude='*.tmp' \
  --exclude='.git/' \
  --exclude='node_modules/' \
  --exclude='__pycache__/' \
  --exclude='*.pyc' \
  --exclude='.env' \
  --exclude='vendor/' \
  --exclude='cache/' \
  --exclude='tmp/' \
  --exclude-from='/etc/backup/rsync_excludes.txt' \
  "${SOURCE}/" \
  "${DEST}/"
```

Using an exclude file (`/etc/backup/rsync_excludes.txt`) keeps the script clean:
```
*.log
*.tmp
.git/
node_modules/
__pycache__/
*.pyc
.env
vendor/
cache/
tmp/
*.swp
.DS_Store
Thumbs.db
```

### rsync Dry Run

Always test rsync commands with `--dry-run` before running for real:

```bash
rsync -avzn \
  --link-dest="${LINK}" \
  "${SOURCE}/" \
  "${DEST}/"
```

`-n` is the short form of `--dry-run`. It shows what would be transferred without actually doing it.

---

## Cloud Upload with rclone

rclone is the Swiss Army knife of cloud storage. It supports 40+ providers and gives you a consistent interface across all of them.

### Installation

```bash
# Official install script (always verify the script before running)
curl https://rclone.org/install.sh | sudo bash

# Or via package manager (may not be latest)
sudo apt install rclone          # Debian/Ubuntu
sudo dnf install rclone          # Fedora/RHEL
brew install rclone              # macOS

# Verify
rclone version
```

### Configuring S3

```bash
rclone config
```

Interactive walkthrough for S3:
```
n/s/q> n
name> s3-backup
Storage> s3
provider> AWS
env_auth> false
access_key_id> AKIAIOSFODNN7EXAMPLE
secret_access_key> wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
region> us-east-1
endpoint>
location_constraint>
acl> private
```

The config is stored at `~/.config/rclone/rclone.conf`. For a service running as root, this is at `/root/.config/rclone/rclone.conf`.

For server-side encryption with customer-managed keys, add to the config:
```ini
[s3-backup]
type = s3
provider = AWS
access_key_id = AKIAIOSFODNN7EXAMPLE
secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
region = us-east-1
acl = private
server_side_encryption = aws:kms
sse_kms_key_id = arn:aws:kms:us-east-1:123456789012:key/mrk-1234
```

### Configuring Backblaze B2

B2 is significantly cheaper than S3 for storage (about 6x cheaper as of 2026). The download bandwidth is free when using Cloudflare as a CDN in front of it, making it excellent for backup storage.

```bash
rclone config
```

```
n/s/q> n
name> b2-backup
Storage> b2
account> 001a2b3c4d5e6f7a8b9c0d1e2f3a4b5c
key> K001abcdefghijklmnopqrstuvwxyz1234567
```

Or directly in `rclone.conf`:
```ini
[b2-backup]
type = b2
account = 001a2b3c4d5e6f7a8b9c0d1e2f3a4b5c
key = K001abcdefghijklmnopqrstuvwxyz1234567
hard_delete = false
```

`hard_delete = false` keeps deleted files as hidden versions rather than permanently deleting them. This gives you a safety net if you accidentally delete a backup — you can recover it from the B2 console.

### rclone sync vs rclone copy — CRITICAL DIFFERENCE

I cannot stress this enough: **understand the difference before you run rclone sync in production.**

**rclone copy:**
- Copies files from source to destination
- Does NOT delete anything at the destination
- Safe for uploading backups
- Destination can have files that don't exist in source — they stay

**rclone sync:**
- Makes destination identical to source
- **DELETES files in destination that don't exist in source**
- Dangerous for backup uploads because it can delete old backups
- Correct use: when you want destination to be an exact mirror

**For backup uploads, use rclone copy:**
```bash
# CORRECT — copies today's backup without touching previous days
rclone copy \
  /backup/postgres/mydb_20260101_030000.dump.gpg \
  s3-backup:my-backup-bucket/postgres/

# DANGEROUS — will delete all previous backups not in /backup/postgres/
rclone sync \
  /backup/postgres/ \
  s3-backup:my-backup-bucket/postgres/
```

**When rclone sync IS appropriate:**
- Mirroring a filesystem to cloud (you want exact parity)
- Syncing a single day's backup directory where you've already named it with a date
- When you handle retention separately in S3 lifecycle rules

If you use `rclone sync`, test it with `--dry-run` first every single time until you are 100% certain of what it will delete.

### rclone Performance Options

```bash
rclone copy \
  --progress \
  --transfers 16 \
  --checkers 32 \
  --bwlimit 100M \
  --retries 5 \
  --low-level-retries 10 \
  --stats 30s \
  /backup/postgres/ \
  s3-backup:my-backup-bucket/postgres/
```

`--progress`: Show real-time transfer progress. Essential for interactive use. Disable in cron scripts (use `--stats 0` instead).

`--transfers 16`: Number of parallel file transfers. Default is 4. For many small files, increase this. For a few large files, the default is fine.

`--checkers 32`: Number of parallel stat operations for checking which files need transferring. Increase for buckets with many files.

`--bwlimit 100M`: Limit to 100 MB/s. Use units: K, M, G. Can specify time windows: `--bwlimit "08:00,10M 18:00,off 23:00,100M"`.

`--retries 5`: Retry failed transfers up to 5 times. rclone handles transient errors automatically.

`--stats 30s`: Print transfer stats every 30 seconds. Good for log files.

### B2 Lifecycle Rules for Cost Optimization

B2 lifecycle rules automatically delete or hide old file versions. This is how you implement retention without running cleanup scripts.

In the B2 console or via API:
- Keep file versions for 7 days (matches your 7 daily retention)
- Or configure per-prefix rules for different retention

For more sophisticated retention (7 daily, 4 weekly, 12 monthly), implement this in your backup script's cleanup function (see the complete script below) rather than relying on lifecycle rules, since lifecycle rules cannot distinguish between daily and monthly backups without a naming convention.

**Naming convention for lifecycle-managed retention:**
```
backups/daily/YYYY-MM-DD/
backups/weekly/YYYY-WW/
backups/monthly/YYYY-MM/
```

Then apply different lifecycle rules to each prefix.

---

## GPG Encryption

Every backup leaving your infrastructure must be encrypted. Non-negotiable. Even if your cloud provider encrypts at rest, they hold the keys. You should hold the keys.

GPG asymmetric encryption is the right approach for backup encryption:
- Encrypt with the recipient's public key
- Only the holder of the private key can decrypt
- The backup server never needs to store the private key
- The private key lives in a secure location (offline, HSM, or encrypted key store)

### GPG Key Generation

Generate a dedicated key for backup encryption. Do NOT use your personal GPG key.

```bash
# Interactive key generation
gpg --full-generate-key

# Recommended settings:
# Key type: RSA and RSA (option 1)
# Key size: 4096 bits
# Expiration: 2y (set an expiration, renew annually)
# Name: Backup Encryption Key
# Email: backup@company.com
# Comment: Production backup encryption 2025
```

Or non-interactively with a parameter file:
```bash
cat > /tmp/backup_keygen_params << 'EOF'
%echo Generating backup encryption key
Key-Type: RSA
Key-Length: 4096
Subkey-Type: RSA
Subkey-Length: 4096
Name-Real: Backup Encryption Key
Name-Email: backup@company.com
Name-Comment: Production backup encryption
Expire-Date: 2y
%no-protection
%commit
%echo done
EOF

gpg --batch --generate-key /tmp/backup_keygen_params
rm /tmp/backup_keygen_params
```

`%no-protection` creates the key without a passphrase. For backup automation, a passphrase on the GPG key defeats the purpose (you'd have to store the passphrase somewhere, making it no more secure). Protect the key by controlling access to the private key file itself.

### Export the Key

```bash
# Export public key (distribute to all backup servers)
gpg --armor --export backup@company.com > /etc/backup/backup-public.asc

# Export private key (store SECURELY — offline storage, password manager, HSM)
# This key is needed for restore. If you lose it, you cannot restore.
gpg --armor --export-secret-keys backup@company.com > backup-private.asc
chmod 600 backup-private.asc
# Store this in: 1Password/Bitwarden, encrypted USB drive, printed QR code in safe
```

### Import Key on Restore Server

```bash
# Import public key (on backup servers — for encryption)
gpg --import /etc/backup/backup-public.asc

# Trust the key
gpg --edit-key backup@company.com
# gpg> trust
# Your decision? 5 (ultimate trust for our own key)
# gpg> quit

# Import private key (on restore server — for decryption)
gpg --import backup-private.asc
```

### Encrypting Backups

**The critical pipeline — pipe directly from pg_dump:**

```bash
pg_dump -U postgres -Fc mydb \
  | gzip \
  | gpg --encrypt --recipient backup@company.com --trust-model always \
  > /backup/postgres/mydb_$(date +%Y%m%d_%H%M%S).dump.gz.gpg
```

This pipeline:
1. Dumps the database directly to stdout (no intermediate file)
2. Compresses with gzip (pg_dump -Fc is already compressed, so skip gzip for -Fc)
3. Encrypts with the backup public key
4. Writes the encrypted file to disk

For `-Fc` format (already compressed internally):
```bash
pg_dump -U postgres -Fc mydb \
  | gpg --encrypt --recipient backup@company.com --trust-model always \
  > /backup/postgres/mydb_$(date +%Y%m%d_%H%M%S).dump.gpg
```

For plain SQL (compresses well):
```bash
pg_dump -U postgres mydb \
  | gzip \
  | gpg --encrypt --recipient backup@company.com --trust-model always \
  > /backup/postgres/mydb_$(date +%Y%m%d_%H%M%S).sql.gz.gpg
```

### Decrypting on Restore

```bash
# Decrypt and decompress
gpg --decrypt /backup/postgres/mydb_20260101_030000.dump.gpg \
  > /tmp/mydb_restore.dump

# Or pipe directly to pg_restore
gpg --decrypt /backup/postgres/mydb_20260101_030000.dump.gpg \
  | pg_restore -U postgres -d mydb_restore -j 8

# For SQL format
gpg --decrypt /backup/postgres/mydb_20260101_030000.sql.gz.gpg \
  | gunzip \
  | psql -U postgres mydb_restore
```

---

## Complete Backup Script

This is a production-ready backup script. Read it completely before deploying. Adjust variables for your environment.

```bash
#!/usr/bin/env bash
# /usr/local/bin/backup.sh
#
# Production backup script
# Runs via cron: 0 2 * * * root /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
#
# Dependencies: pg_dump, mongodump, mysqldump, gpg, rclone, curl
# Config: /etc/backup/backup.conf

set -euo pipefail

# ─── Configuration ─────────────────────────────────────────────────────────────

CONFIG_FILE="/etc/backup/backup.conf"
[[ -f "${CONFIG_FILE}" ]] && source "${CONFIG_FILE}"

# Defaults (override in /etc/backup/backup.conf)
BACKUP_ROOT="${BACKUP_ROOT:-/var/backups/db}"
GPG_RECIPIENT="${GPG_RECIPIENT:-backup@company.com}"
RCLONE_REMOTE="${RCLONE_REMOTE:-s3-backup:my-backup-bucket}"
HEALTHCHECK_URL="${HEALTHCHECK_URL:-}"  # Set to healthchecks.io URL
LOG_FILE="${LOG_FILE:-/var/log/backup.log}"
KEEP_DAILY="${KEEP_DAILY:-7}"
KEEP_WEEKLY="${KEEP_WEEKLY:-4}"
KEEP_MONTHLY="${KEEP_MONTHLY:-12}"

# Databases to back up
PG_DATABASES="${PG_DATABASES:-mydb}"            # space-separated
PG_HOST="${PG_HOST:-localhost}"
PG_USER="${PG_USER:-backupuser}"
PGPASSFILE="/etc/backup/.pgpass"

MONGO_DATABASES="${MONGO_DATABASES:-}"          # space-separated; empty = skip
MONGO_HOST="${MONGO_HOST:-localhost}"
MONGO_USER="${MONGO_USER:-backupuser}"
MONGO_PASSWORD_FILE="/etc/backup/mongo_password"

MYSQL_DATABASES="${MYSQL_DATABASES:-}"          # space-separated; empty = skip
MYSQL_HOST="${MYSQL_HOST:-localhost}"
MYSQL_USER="${MYSQL_USER:-backupuser}"
MYSQL_PASSWORD_FILE="/etc/backup/mysql_password"

# ─── Setup ─────────────────────────────────────────────────────────────────────

DATE=$(date +%Y%m%d_%H%M%S)
DAY_OF_WEEK=$(date +%u)    # 1=Monday, 7=Sunday
DAY_OF_MONTH=$(date +%d)   # 01-31

BACKUP_DIR="${BACKUP_ROOT}/${DATE}"
mkdir -p "${BACKUP_DIR}"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

die() {
    log "FATAL: $*"
    # Ping healthchecks.io failure endpoint if configured
    if [[ -n "${HEALTHCHECK_URL}" ]]; then
        curl -fsS --retry 3 "${HEALTHCHECK_URL}/fail" -d "Backup failed: $*" || true
    fi
    exit 1
}

check_dependencies() {
    local deps=(pg_dump gpg rclone curl)
    [[ -n "${MONGO_DATABASES}" ]] && deps+=(mongodump)
    [[ -n "${MYSQL_DATABASES}" ]] && deps+=(mysqldump)
    for dep in "${deps[@]}"; do
        command -v "${dep}" &>/dev/null || die "Missing dependency: ${dep}"
    done
}

# ─── PostgreSQL Backups ─────────────────────────────────────────────────────────

backup_postgres() {
    [[ -z "${PG_DATABASES}" ]] && return 0
    log "Starting PostgreSQL backups..."

    export PGPASSFILE="${PGPASSFILE}"

    for db in ${PG_DATABASES}; do
        log "  Dumping PostgreSQL database: ${db}"
        local outfile="${BACKUP_DIR}/pg_${db}_${DATE}.dump.gpg"

        # Pipe dump through GPG encryption — no plaintext on disk
        PGHOST="${PG_HOST}" pg_dump \
            -U "${PG_USER}" \
            -Fc \
            "${db}" \
            | gpg \
                --batch \
                --yes \
                --encrypt \
                --recipient "${GPG_RECIPIENT}" \
                --trust-model always \
            > "${outfile}" \
            || die "PostgreSQL dump failed for database: ${db}"

        local size
        size=$(du -sh "${outfile}" | cut -f1)
        log "  Completed: ${outfile} (${size})"
    done
}

# ─── MongoDB Backups ─────────────────────────────────────────────────────────

backup_mongo() {
    [[ -z "${MONGO_DATABASES}" ]] && return 0
    log "Starting MongoDB backups..."

    local mongo_pass
    mongo_pass=$(cat "${MONGO_PASSWORD_FILE}")

    for db in ${MONGO_DATABASES}; do
        log "  Dumping MongoDB database: ${db}"
        local dump_dir="${BACKUP_DIR}/mongo_${db}_tmp"
        local outfile="${BACKUP_DIR}/mongo_${db}_${DATE}.tar.gz.gpg"

        mongodump \
            --host "${MONGO_HOST}" \
            --username "${MONGO_USER}" \
            --password "${mongo_pass}" \
            --authenticationDatabase admin \
            --db "${db}" \
            --gzip \
            --out "${dump_dir}" \
            || die "MongoDB dump failed for database: ${db}"

        # Tar, compress, and encrypt
        tar -czf - -C "${dump_dir}" . \
            | gpg \
                --batch \
                --yes \
                --encrypt \
                --recipient "${GPG_RECIPIENT}" \
                --trust-model always \
            > "${outfile}" \
            || die "MongoDB encryption failed for database: ${db}"

        rm -rf "${dump_dir}"

        local size
        size=$(du -sh "${outfile}" | cut -f1)
        log "  Completed: ${outfile} (${size})"
    done
}

# ─── MySQL Backups ─────────────────────────────────────────────────────────

backup_mysql() {
    [[ -z "${MYSQL_DATABASES}" ]] && return 0
    log "Starting MySQL backups..."

    local mysql_pass
    mysql_pass=$(cat "${MYSQL_PASSWORD_FILE}")

    for db in ${MYSQL_DATABASES}; do
        log "  Dumping MySQL database: ${db}"
        local outfile="${BACKUP_DIR}/mysql_${db}_${DATE}.sql.gz.gpg"

        mysqldump \
            --user="${MYSQL_USER}" \
            --password="${mysql_pass}" \
            --host="${MYSQL_HOST}" \
            --single-transaction \
            --routines \
            --triggers \
            --events \
            --hex-blob \
            "${db}" \
            | gzip \
            | gpg \
                --batch \
                --yes \
                --encrypt \
                --recipient "${GPG_RECIPIENT}" \
                --trust-model always \
            > "${outfile}" \
            || die "MySQL dump failed for database: ${db}"

        local size
        size=$(du -sh "${outfile}" | cut -f1)
        log "  Completed: ${outfile} (${size})"
    done
}

# ─── Upload to Cloud ─────────────────────────────────────────────────────────

upload_to_cloud() {
    log "Uploading backups to cloud: ${RCLONE_REMOTE}"

    rclone copy \
        --stats 60s \
        --transfers 4 \
        --checkers 8 \
        --retries 5 \
        --low-level-retries 10 \
        "${BACKUP_DIR}/" \
        "${RCLONE_REMOTE}/$(date +%Y/%m/%d)/" \
        || die "rclone upload failed"

    log "Upload complete"
}

# ─── Retention Cleanup ─────────────────────────────────────────────────────────
# Strategy:
#   Daily: keep last KEEP_DAILY days
#   Weekly: keep last KEEP_WEEKLY Sunday snapshots
#   Monthly: keep last KEEP_MONTHLY first-of-month snapshots
#
# Naming convention: backup directories are named YYYYMMDD_HHMMSS
# We keep:
#   - All dirs from the last 7 days
#   - Sunday dirs from the last 4 weeks (for weekly)
#   - The first backup dir of each month for the last 12 months

cleanup_local() {
    log "Running retention cleanup on ${BACKUP_ROOT}..."

    # Get list of all backup directories (sorted oldest first)
    local all_dirs
    mapfile -t all_dirs < <(find "${BACKUP_ROOT}" -maxdepth 1 -type d -name '????????_??????' | sort)

    local to_keep=()
    local now_epoch
    now_epoch=$(date +%s)

    for dir in "${all_dirs[@]}"; do
        local dirname
        dirname=$(basename "${dir}")
        local dir_date="${dirname:0:8}"  # YYYYMMDD
        local dir_epoch
        dir_epoch=$(date -d "${dir_date}" +%s 2>/dev/null) || continue

        local age_days=$(( (now_epoch - dir_epoch) / 86400 ))

        # Keep if within KEEP_DAILY days
        if [[ ${age_days} -le ${KEEP_DAILY} ]]; then
            to_keep+=("${dir}")
            continue
        fi

        # Keep weekly (Sunday = day 7) if within KEEP_WEEKLY weeks
        local day_of_week
        day_of_week=$(date -d "${dir_date}" +%u)
        local age_weeks=$(( age_days / 7 ))
        if [[ "${day_of_week}" == "7" && ${age_weeks} -le ${KEEP_WEEKLY} ]]; then
            to_keep+=("${dir}")
            continue
        fi

        # Keep monthly (1st of month) if within KEEP_MONTHLY months
        local day_of_month="${dir_date:6:2}"
        local age_months=$(( age_days / 30 ))
        if [[ "${day_of_month}" == "01" && ${age_months} -le ${KEEP_MONTHLY} ]]; then
            to_keep+=("${dir}")
            continue
        fi
    done

    # Delete directories not in keep list
    for dir in "${all_dirs[@]}"; do
        local keep=false
        for k in "${to_keep[@]}"; do
            [[ "${dir}" == "${k}" ]] && keep=true && break
        done
        if [[ "${keep}" == "false" ]]; then
            log "  Deleting old backup: ${dir}"
            rm -rf "${dir}"
        fi
    done

    log "Retention cleanup complete. Kept ${#to_keep[@]} of ${#all_dirs[@]} backup sets."
}

# ─── Healthcheck Ping ─────────────────────────────────────────────────────────

ping_healthcheck() {
    if [[ -z "${HEALTHCHECK_URL}" ]]; then
        log "No healthcheck URL configured, skipping ping"
        return 0
    fi

    log "Pinging healthcheck: ${HEALTHCHECK_URL}"
    curl -fsS --retry 3 "${HEALTHCHECK_URL}" \
        -d "Backup completed successfully at $(date --iso-8601=seconds)" \
        || log "WARNING: healthcheck ping failed (backup itself succeeded)"
}

# ─── Main ─────────────────────────────────────────────────────────────────────

main() {
    log "====== Backup started: ${DATE} ======"
    check_dependencies
    backup_postgres
    backup_mongo
    backup_mysql
    upload_to_cloud
    cleanup_local
    ping_healthcheck
    log "====== Backup completed successfully ======"
}

main "$@"
```

### Config File

```bash
# /etc/backup/backup.conf
# Permissions: chmod 640 /etc/backup/backup.conf

BACKUP_ROOT="/var/backups/db"
GPG_RECIPIENT="backup@company.com"
RCLONE_REMOTE="s3-backup:my-company-backups"
HEALTHCHECK_URL="https://hc-ping.com/your-uuid-here"
LOG_FILE="/var/log/backup.log"

PG_DATABASES="mydb analytics"
PG_HOST="localhost"
PG_USER="backupuser"

MONGO_DATABASES="events"
MONGO_HOST="localhost"
MONGO_USER="backupuser"

MYSQL_DATABASES=""   # No MySQL on this server
```

### .pgpass File

```
# /etc/backup/.pgpass
# Format: hostname:port:database:username:password
# Permissions: chmod 600 /etc/backup/.pgpass
localhost:5432:mydb:backupuser:strongpassword123
localhost:5432:analytics:backupuser:strongpassword123
localhost:5432:*:backupuser:strongpassword123
```

### Crontab

```crontab
# /etc/cron.d/backup
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Daily backup at 02:00
0 2 * * * root /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

# Rotate backup log monthly
0 0 1 * * root /usr/bin/logrotate /etc/logrotate.d/backup
```

### logrotate Config

```
# /etc/logrotate.d/backup
/var/log/backup.log {
    monthly
    rotate 12
    compress
    missingok
    notifempty
}
```

---

## IAM Policy for S3 Backup User

Create a dedicated IAM user with minimal permissions. This user should only be able to write backups to the specific backup bucket. It should NOT have delete permissions — use lifecycle rules or a separate cleanup process with a separate key.

### IAM Policy JSON

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowBackupWrite",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::my-company-backups",
        "arn:aws:s3:::my-company-backups/*"
      ]
    },
    {
      "Sid": "AllowRetentionCleanup",
      "Effect": "Allow",
      "Action": [
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-company-backups/*"
      ],
      "Condition": {
        "StringLike": {
          "s3:prefix": [
            "postgres/*",
            "mongo/*",
            "mysql/*"
          ]
        }
      }
    }
  ]
}
```

**Separate read-only restore policy (attach to a different user/role, used only during actual restores):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowBackupRead",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::my-company-backups",
        "arn:aws:s3:::my-company-backups/*"
      ]
    }
  ]
}
```

### Bucket Policy for Object Lock (Immutable Backups)

Enable Object Lock at the bucket level to prevent deletion or modification:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDeletionDuringRetentionPeriod",
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "s3:DeleteObject",
        "s3:DeleteObjectVersion"
      ],
      "Resource": "arn:aws:s3:::my-company-backups/*",
      "Condition": {
        "Null": {
          "s3:object-lock-remaining-retention-days": "false"
        }
      }
    },
    {
      "Sid": "DenyUnencryptedPuts",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-company-backups/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }
  ]
}
```

### S3 Lifecycle Rules

Apply via AWS CLI or console to implement retention at the bucket level:

```json
{
  "Rules": [
    {
      "ID": "ExpireDailyBackups",
      "Filter": { "Prefix": "daily/" },
      "Status": "Enabled",
      "Expiration": { "Days": 7 }
    },
    {
      "ID": "ExpireWeeklyBackups",
      "Filter": { "Prefix": "weekly/" },
      "Status": "Enabled",
      "Expiration": { "Days": 28 }
    },
    {
      "ID": "ExpireMonthlyBackups",
      "Filter": { "Prefix": "monthly/" },
      "Status": "Enabled",
      "Expiration": { "Days": 365 }
    },
    {
      "ID": "DeleteIncompleteMultipartUploads",
      "Filter": { "Prefix": "" },
      "Status": "Enabled",
      "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 1 }
    }
  ]
}
```

Apply with AWS CLI:
```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-company-backups \
  --lifecycle-configuration file://lifecycle.json
```

---

## Restore Runbook

**This runbook must be tested. If you have not run through this runbook in the past 30 days, it is expired.** Date your test restores. Document what broke. Fix it. Test again.

### Pre-Restore Checklist

- [ ] Identify the target backup file (date, database, format)
- [ ] Confirm you have the GPG private key accessible
- [ ] Confirm the restore target server has enough disk space (at least 3x the backup file size)
- [ ] Confirm the rclone remote is configured on the restore server
- [ ] Notify stakeholders of planned restore and estimated downtime
- [ ] Create a snapshot of the current database state if partial data might be salvageable

### Step 1: Identify the Backup

```bash
# List available backups in S3
rclone ls s3-backup:my-company-backups/ | sort | tail -50

# Or list by date prefix
rclone ls s3-backup:my-company-backups/2026/04/01/

# List local backups
ls -lh /var/backups/db/ | sort
```

### Step 2: Download the Backup

```bash
# Download specific backup file
rclone copy \
  s3-backup:my-company-backups/2026/04/01/pg_mydb_20260401_020000.dump.gpg \
  /tmp/restore/

# Verify the file arrived intact (check file size matches)
ls -lh /tmp/restore/
```

### Step 3: Import GPG Key (if not already on restore server)

```bash
# Import private key from secure storage
gpg --import /path/to/backup-private.asc

# Verify key is available
gpg --list-secret-keys backup@company.com

# Set trust level
echo "$(gpg --fingerprint backup@company.com | grep fingerprint | tr -d ' ' | cut -d= -f2):6:" \
  | gpg --import-ownertrust
```

### Step 4: Decrypt the Backup

```bash
# Test decryption (verify key works, do not restore yet)
gpg --decrypt /tmp/restore/pg_mydb_20260401_020000.dump.gpg \
  | pg_restore --list \
  | head -30
# This should show the table of contents without restoring anything

# Full decrypt to temporary file (optional — can pipe directly to pg_restore)
gpg --decrypt /tmp/restore/pg_mydb_20260401_020000.dump.gpg \
  > /tmp/restore/pg_mydb_20260401_020000.dump
```

### Step 5: PostgreSQL Restore

**Option A: Restore into existing database (replacing current data)**

```bash
# Stop application connections to the database
# Terminate active connections
psql -U postgres -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='mydb' AND pid <> pg_backend_pid();"

# Restore with parallel workers
gpg --decrypt /tmp/restore/pg_mydb_20260401_020000.dump.gpg \
  | pg_restore \
    -U postgres \
    -h localhost \
    -d mydb \
    -j 8 \
    -c \
    --if-exists \
    -e \
    -v \
  2>&1 | tee /var/log/restore_$(date +%Y%m%d_%H%M%S).log
```

**Option B: Restore into a new database (safest — keeps original intact)**

```bash
# Create new database
psql -U postgres -c "CREATE DATABASE mydb_restore;"

# Restore
gpg --decrypt /tmp/restore/pg_mydb_20260401_020000.dump.gpg \
  | pg_restore \
    -U postgres \
    -h localhost \
    -d mydb_restore \
    -j 8 \
    -e \
    -v \
  2>&1 | tee /var/log/restore_$(date +%Y%m%d_%H%M%S).log

# Verify data looks correct
psql -U postgres -d mydb_restore -c "\dt"
psql -U postgres -d mydb_restore -c "SELECT COUNT(*) FROM users;"

# Once satisfied, rename databases
psql -U postgres -c "ALTER DATABASE mydb RENAME TO mydb_old_$(date +%Y%m%d);"
psql -U postgres -c "ALTER DATABASE mydb_restore RENAME TO mydb;"

# Drop old database after you're confident the restore is good
# psql -U postgres -c "DROP DATABASE mydb_old_20260401;"
```

### Step 6: MongoDB Restore

```bash
# Decrypt
gpg --decrypt /tmp/restore/mongo_events_20260401_020000.tar.gz.gpg \
  > /tmp/restore/mongo_events.tar.gz

# Extract
mkdir -p /tmp/restore/mongo_events
tar -xzf /tmp/restore/mongo_events.tar.gz -C /tmp/restore/mongo_events

# Restore
mongorestore \
  --host localhost \
  --username adminuser \
  --password "$(cat /etc/backup/mongo_admin_password)" \
  --authenticationDatabase admin \
  --db events \
  --gzip \
  --drop \
  /tmp/restore/mongo_events/events/
```

### Step 7: MySQL Restore

```bash
# Decrypt and decompress, pipe to mysql
gpg --decrypt /tmp/restore/mysql_mydb_20260401_020000.sql.gz.gpg \
  | gunzip \
  | mysql \
    --user=root \
    --password="$(cat /etc/backup/mysql_root_password)" \
    mydb
```

### Step 8: Verify the Restore

Never call a restore complete until you have verified the data. At minimum:

```bash
# PostgreSQL
psql -U postgres -d mydb -c "SELECT COUNT(*) FROM users;"
psql -U postgres -d mydb -c "SELECT COUNT(*) FROM orders WHERE created_at > NOW() - INTERVAL '7 days';"
psql -U postgres -d mydb -c "\dt"  # List all tables
psql -U postgres -d mydb -c "SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) FROM pg_tables WHERE schemaname='public' ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC LIMIT 10;"

# MongoDB
mongosh --eval "db.events.countDocuments()" events
mongosh --eval "db.events.findOne()" events

# MySQL
mysql -u root -p mydb -e "SELECT COUNT(*) FROM users;"
mysql -u root -p mydb -e "SHOW TABLES;"
```

### Step 9: Resume Application Traffic

```bash
# If you swapped database names, update application configs if needed
# Restart application services
systemctl restart myapp
systemctl restart nginx

# Monitor application logs for errors
journalctl -fu myapp | head -100
```

### Step 10: Post-Restore Documentation

Write down what happened:
```
# /var/log/restore_incident_20260401.txt
Date: 2026-04-01
Restored by: [name]
Reason for restore: [description]
Backup used: pg_mydb_20260401_020000.dump.gpg (from 02:00 UTC)
Data loss: approximately 0 rows (backup was from 02:00, incident occurred at 02:47)
Restore started: 03:15 UTC
Restore completed: 03:52 UTC (37 minutes total)
Verification: row counts matched expected ranges
Application traffic resumed: 03:58 UTC
Total downtime: ~1 hour 11 minutes
```

---

## Backup Verification Checklist

**Monthly test restore drill. Non-negotiable. Block time on the calendar.**

The most common failure modes I've seen:
- GPG private key is lost, inaccessible, or expired
- The backup file is corrupt (silent corruption during write or upload)
- pg_restore fails on schema changes that happened after the backup format was tested
- rclone credentials rotated without updating backup config
- The restore server doesn't have enough disk space
- pg_dump was running but silently failing (check exit codes!)

Every single one of these was discovered during a test restore — not during an actual incident. Test your restores.

### Monthly Test Restore Drill

Perform this drill on a **non-production** system. Do it monthly. Document results.

```
DATE: ________________
PERFORMED BY: ________________
SYSTEMS TESTED: ________________
```

**Pre-drill verification**

- [ ] Confirm backup script ran successfully last night (check `/var/log/backup.log`)
- [ ] Confirm backup completed with exit code 0 (no silent failures)
- [ ] Confirm backup files exist in local storage (`ls -lh /var/backups/db/`)
- [ ] Confirm backup files exist in cloud storage (`rclone ls s3-backup:my-company-backups/ | tail -20`)
- [ ] Confirm healthcheck ping was received (check healthchecks.io dashboard)
- [ ] Confirm backup file sizes are in expected range (alert on >50% decrease)

**GPG key verification**

- [ ] Private key is accessible (check secure storage location, e.g., 1Password)
- [ ] Private key is not expired (`gpg --list-secret-keys backup@company.com`)
- [ ] Test decrypt: `gpg --decrypt [recent-backup-file] | head -c 100 | hexdump` — should produce output

**Restore test — PostgreSQL**

- [ ] Download most recent PostgreSQL backup from cloud
- [ ] Decrypt successfully
- [ ] `pg_restore --list` shows expected tables
- [ ] Restore into test database (`mydb_test`)
- [ ] Row count check: compare critical table counts to production (within expected range)
- [ ] Schema check: `\dt` shows expected tables, no missing tables
- [ ] Query a recent record: select a row from past week that you know should exist
- [ ] Test application connectivity to restored database (smoke test)
- [ ] Drop test database after verification

**Restore test — MongoDB** (if applicable)

- [ ] Download most recent MongoDB backup from cloud
- [ ] Decrypt successfully
- [ ] Restore into test database
- [ ] Collection count check: `db.getCollectionNames().length`
- [ ] Document count check: compare to production for key collections
- [ ] Query a recent document

**Restore test — MySQL** (if applicable)

- [ ] Download most recent MySQL backup from cloud
- [ ] Decrypt and decompress pipeline works
- [ ] Restore into test database
- [ ] Table existence check: `SHOW TABLES`
- [ ] Row count check for key tables

**Infrastructure checks**

- [ ] Confirm local backup disk usage is within expected range
- [ ] Confirm old backups are being deleted (retention policy working)
- [ ] Confirm cloud storage costs are within budget
- [ ] Confirm backup user credentials have not expired
- [ ] Confirm IAM/bucket policies have not changed
- [ ] Review rclone logs for any transfer errors

**Timing**

- [ ] Record time to complete each restore step (for RTO planning)
- [ ] Total restore time is within RTO: `____ minutes` (RTO: `____ minutes`)
- [ ] If restore time exceeds RTO, create ticket to improve restore speed

**Documentation**

- [ ] Update this checklist with any new failure modes discovered
- [ ] Update restore runbook if any steps were wrong or missing
- [ ] Sign off: `Drill completed on ________ by ________. All checks passed: YES / NO`
- [ ] If checks failed: open incident ticket, do not mark as passed until resolved

---

## Alerting and Monitoring

Backups need monitoring just like production services. Silent failure is the most dangerous failure mode.

### healthchecks.io Integration

healthchecks.io is a dead-man's switch service. You ping it after every successful backup. If it doesn't receive a ping within the expected interval, it alerts you.

```bash
# Free tier supports multiple checks
# Set up a check for each server/database combination
# Expected schedule: daily, grace period: 1 hour
# Alert via email, Slack, PagerDuty

HEALTHCHECK_URL="https://hc-ping.com/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# Ping on success (in backup script)
curl -fsS "${HEALTHCHECK_URL}" -d "Backup OK"

# Ping on failure
curl -fsS "${HEALTHCHECK_URL}/fail" -d "Backup FAILED: ${error_message}"

# Ping on start (to detect script that hangs before completion)
curl -fsS "${HEALTHCHECK_URL}/start"
```

### Backup Size Monitoring

Alert when backup size drops by more than 25%:

```bash
#!/bin/bash
# /usr/local/bin/check-backup-size.sh
BACKUP_DIR="/var/backups/db"
ALERT_EMAIL="sysadmin@company.com"

# Get today's and yesterday's backup sizes
TODAY=$(find "${BACKUP_DIR}" -maxdepth 2 -name "*.gpg" -newer "${BACKUP_DIR}/$(ls -t "${BACKUP_DIR}" | sed -n '2p')" | xargs du -sc | tail -1 | cut -f1)
YESTERDAY=$(find "${BACKUP_DIR}/$(ls -t "${BACKUP_DIR}" | sed -n '2p')" -name "*.gpg" | xargs du -sc | tail -1 | cut -f1)

if [[ ${YESTERDAY} -gt 0 ]]; then
    RATIO=$(echo "scale=2; ${TODAY} * 100 / ${YESTERDAY}" | bc)
    if (( $(echo "${RATIO} < 75" | bc -l) )); then
        echo "WARNING: Backup size dropped by more than 25%. Today: ${TODAY}KB, Yesterday: ${YESTERDAY}KB" \
          | mail -s "Backup Size Alert - $(hostname)" "${ALERT_EMAIL}"
    fi
fi
```

### Prometheus / Grafana Integration

If you use Prometheus, expose backup metrics:

```bash
# Write metrics to node_exporter textfile collector
METRICS_FILE="/var/lib/node_exporter/textfile_collector/backup.prom"

cat > "${METRICS_FILE}" << EOF
# HELP backup_last_success_timestamp Unix timestamp of last successful backup
# TYPE backup_last_success_timestamp gauge
backup_last_success_timestamp{job="database_backup"} $(date +%s)

# HELP backup_size_bytes Size of most recent backup in bytes
# TYPE backup_size_bytes gauge
backup_size_bytes{job="database_backup",database="mydb"} $(stat -c%s /var/backups/db/latest/pg_mydb_*.dump.gpg)
EOF
```

---

## Common Mistakes and How to Avoid Them

**1. Testing backups but never restores**

The backup succeeded. The encrypted file exists in S3. You feel safe. But you have never decrypted it. The GPG key is in a 1Password vault that was transferred to a new owner who didn't know what the key was for and deleted it six months ago. This is how people lose their data. Test restores.

**2. Using pg_dump without checking the exit code**

pg_dump exits with a non-zero code on failure, but if you don't check it in your script, you happily encrypt and upload an empty or partial file. Use `set -euo pipefail` and check that output files are non-empty.

```bash
pg_dump ... > /tmp/dump.sql || die "pg_dump failed"
[[ -s /tmp/dump.sql ]] || die "pg_dump produced empty file"
```

**3. Storing GPG private key on the backup server**

If the backup server is compromised, the attacker has your private key and can decrypt all your backups. The private key should be in offline storage or an HSM. Restore servers get the private key only during actual restore operations.

**4. Not testing GPG key expiration**

GPG keys expire. Set a calendar reminder to renew the backup encryption key 60 days before expiration. If the key expires and you don't notice, your next backup attempt silently fails or produces unencryptable output.

**5. rclone sync deleting old backups**

I've seen this happen. A sysadmin uses `rclone sync` thinking it's like rsync. It deletes all old backups in the destination that don't exist locally. Use `rclone copy` for backup uploads.

**6. Not limiting the backup user's IAM permissions**

If your backup server is compromised and the attacker has your S3 credentials, they can delete all your backups. Give the backup user `PutObject` but not `DeleteObject`. Use a separate restore user with read-only permissions.

**7. mysqldump without --single-transaction**

Without `--single-transaction`, mysqldump acquires table locks for the duration of the dump. For a 50GB database, this means 30 minutes of write locks on your production tables. Always use `--single-transaction` for InnoDB.

**8. Backing up to the same server as the database**

A disk failure, RAID corruption, or ransomware attack takes out both your database and your backup simultaneously. The local backup must be on a different physical device, network share, or remote server. At minimum, a different disk. Preferably, a different machine.

**9. No offsite backup**

"The database server is in AWS so it's safe" — until someone with admin access to your AWS account deletes the RDS instance and its snapshots. Or until a regional outage. Offsite means a different provider, not a different availability zone.

**10. Not documenting the restore procedure**

At 3 AM during an outage, under pressure, with your hands shaking, you will not remember the exact pg_restore flags. The restore runbook must be written, reviewed, and practiced before the incident. Keep it in a location that is accessible even when your primary systems are down (a printed copy, a different cloud document, a second laptop).

---

## Summary

To recap the non-negotiables:

1. **Test restores, not just backups.** A backup you have not restored from is not a backup.
2. **Encrypt before upload.** GPG with asymmetric keys, private key stored offline.
3. **Use rclone copy, not rclone sync**, when uploading backups.
4. **Implement 3-2-1**: local copy, cloud copy, separate failure domains.
5. **Know your RPO and RTO** before choosing backup frequency and infrastructure.
6. **Run monthly restore drills** and document the results with timing.
7. **Monitor with dead-man's switches** (healthchecks.io) so silent failures alert you.
8. **Limit IAM permissions** — backup user writes only, restore user reads only.
9. **Use `--single-transaction`** for MySQL/MariaDB InnoDB backups.
10. **Use `pg_dump -Fc` with `pg_restore -j`** for fast parallel PostgreSQL restores.

The backup infrastructure is only as good as your last successful test restore. Schedule the drill. Do the drill. Document the results.
