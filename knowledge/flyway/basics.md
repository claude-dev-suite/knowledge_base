# Flyway Database Migrations

> Official Documentation: https://documentation.red-gate.com/fd

## Overview

Flyway is a database migration tool that uses SQL scripts and Java code to track, manage, and apply database schema changes. It ensures database schema consistency across all environments by versioning migrations and tracking which have been applied.

Key concepts:
- **Migrations**: SQL or Java files that modify the database schema
- **Schema History Table**: `flyway_schema_history` tracks applied migrations
- **Checksums**: Detect if previously applied migrations were modified
- **Baselines**: Starting point for existing databases

---

## Setup with Spring Boot

### Maven Dependency

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>

<!-- For PostgreSQL -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>

<!-- For MySQL/MariaDB -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-mysql</artifactId>
</dependency>
```

### Gradle Dependency

```groovy
implementation 'org.flywaydb:flyway-core'
implementation 'org.flywaydb:flyway-database-postgresql' // For PostgreSQL
```

### Directory Structure

```
src/main/resources/
└── db/
    └── migration/
        ├── V1__create_users_table.sql
        ├── V2__add_email_column.sql
        └── R__create_audit_view.sql
```

---

## Migration File Naming

### Versioned Migrations

Format: `V<version>__<description>.sql`

```
V1__create_users.sql           # Version 1
V1.1__add_index.sql            # Version 1.1
V2__create_orders.sql          # Version 2
V2.0.1__fix_constraint.sql     # Version 2.0.1
V20231115__daily_migration.sql # Date-based version
```

Rules:
- Prefix `V` (uppercase) is required
- Version can use dots, underscores, or integers
- Double underscore `__` separates version from description
- Description uses underscores for spaces
- Extension must be `.sql`

### Repeatable Migrations

Format: `R__<description>.sql`

```
R__create_views.sql
R__refresh_materialized_views.sql
R__update_stored_procedures.sql
```

Repeatable migrations run whenever their checksum changes and always run after versioned migrations.

### Undo Migrations (Teams/Enterprise)

Format: `U<version>__<description>.sql`

```
U1__create_users.sql  # Undoes V1__create_users.sql
```

---

## Configuration Properties

### application.yml

```yaml
spring:
  flyway:
    enabled: true
    url: jdbc:postgresql://localhost:5432/mydb
    user: ${DB_USER}
    password: ${DB_PASSWORD}

    # Locations
    locations: classpath:db/migration,filesystem:/opt/migrations

    # Baseline
    baseline-on-migrate: true
    baseline-version: '0'
    baseline-description: 'Initial baseline'

    # Behavior
    out-of-order: false
    validate-on-migrate: true
    clean-disabled: true

    # Schema
    schemas: public,audit
    default-schema: public
    create-schemas: true

    # Table
    table: flyway_schema_history

    # Placeholders
    placeholder-replacement: true
    placeholders:
      schema_name: public
      table_prefix: app_
```

### application.properties

```properties
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
spring.flyway.baseline-on-migrate=true
spring.flyway.baseline-version=0
spring.flyway.out-of-order=false
spring.flyway.validate-on-migrate=true
spring.flyway.clean-disabled=true
spring.flyway.schemas=public
spring.flyway.table=flyway_schema_history
```

### Key Properties Explained

| Property | Default | Description |
|----------|---------|-------------|
| `enabled` | true | Enable/disable Flyway |
| `locations` | classpath:db/migration | Migration script locations |
| `baseline-on-migrate` | false | Baseline existing database on first migrate |
| `baseline-version` | 1 | Version to baseline |
| `out-of-order` | false | Allow migrations to run out of order |
| `validate-on-migrate` | true | Validate checksums on migrate |
| `clean-disabled` | true | Disable clean command (safety) |
| `target` | latest | Target version to migrate to |

---

## Versioned vs Repeatable Migrations

### Versioned Migrations

- Run exactly once
- Must have unique version numbers
- Applied in version order
- Checksum validated (changes cause errors)

```sql
-- V1__create_users.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
```

### Repeatable Migrations

- Run whenever checksum changes
- No version number
- Run after all versioned migrations
- Sorted by description alphabetically

```sql
-- R__create_user_statistics_view.sql
CREATE OR REPLACE VIEW user_statistics AS
SELECT
    DATE(created_at) as registration_date,
    COUNT(*) as user_count
FROM users
GROUP BY DATE(created_at);
```

Use cases for repeatable migrations:
- Views
- Stored procedures
- Functions
- Reference/seed data

---

## Baseline Migrations

Baselines establish a starting point for existing databases.

### When to Use

- Adopting Flyway on an existing database
- Skipping already-applied migrations in production

### Configuration

```yaml
spring:
  flyway:
    baseline-on-migrate: true
    baseline-version: '1'
    baseline-description: 'Existing production schema'
```

### Manual Baseline Command

```bash
# Maven
mvn flyway:baseline -Dflyway.baselineVersion=5

# Gradle
./gradlew flywayBaseline -Dflyway.baselineVersion=5
```

### Programmatic Baseline

```java
@Bean
public FlywayMigrationStrategy flywayMigrationStrategy() {
    return flyway -> {
        if (!flyway.info().applied().length > 0) {
            flyway.baseline();
        }
        flyway.migrate();
    };
}
```

---

## Callbacks

Callbacks execute custom code at specific points during the migration lifecycle.

### Available Callbacks

| Callback | Timing |
|----------|--------|
| beforeMigrate | Before migrate runs |
| afterMigrate | After successful migrate |
| afterMigrateError | After migrate fails |
| beforeEachMigrate | Before each migration |
| afterEachMigrate | After each migration |
| beforeClean | Before clean runs |
| afterClean | After clean completes |
| beforeInfo | Before info runs |
| afterInfo | After info completes |
| beforeValidate | Before validate runs |
| afterValidate | After validate completes |
| beforeBaseline | Before baseline runs |
| afterBaseline | After baseline completes |
| beforeRepair | Before repair runs |
| afterRepair | After repair completes |

### SQL Callbacks

Create callback files in your migration directory:

```sql
-- beforeMigrate.sql
SELECT 'Starting migration at ' || NOW();

-- afterMigrate.sql
REFRESH MATERIALIZED VIEW CONCURRENTLY user_statistics;
ANALYZE users;
```

### Java Callbacks

```java
import org.flywaydb.core.api.callback.Callback;
import org.flywaydb.core.api.callback.Context;
import org.flywaydb.core.api.callback.Event;
import org.springframework.stereotype.Component;

@Component
public class FlywayAuditCallback implements Callback {

    private static final Logger log = LoggerFactory.getLogger(FlywayAuditCallback.class);

    @Override
    public boolean supports(Event event, Context context) {
        return event == Event.AFTER_MIGRATE || event == Event.AFTER_MIGRATE_ERROR;
    }

    @Override
    public boolean canHandleInTransaction(Event event, Context context) {
        return true;
    }

    @Override
    public void handle(Event event, Context context) {
        if (event == Event.AFTER_MIGRATE) {
            log.info("Migration completed successfully. Applied {} migrations.",
                context.getMigrationInfo().length);
        } else if (event == Event.AFTER_MIGRATE_ERROR) {
            log.error("Migration failed!");
            // Send alert, rollback actions, etc.
        }
    }

    @Override
    public String getCallbackName() {
        return "AuditCallback";
    }
}
```

---

## Placeholders

Placeholders allow dynamic values in migrations.

### Configuration

```yaml
spring:
  flyway:
    placeholder-replacement: true
    placeholder-prefix: '${'
    placeholder-suffix: '}'
    placeholders:
      schema_name: public
      admin_email: admin@example.com
      table_prefix: app_
```

### Usage in Migrations

```sql
-- V1__create_config.sql
CREATE TABLE ${table_prefix}configuration (
    id SERIAL PRIMARY KEY,
    key VARCHAR(100) NOT NULL,
    value TEXT,
    created_by VARCHAR(255) DEFAULT '${admin_email}'
);

INSERT INTO ${table_prefix}configuration (key, value)
VALUES ('schema', '${schema_name}');
```

### Built-in Placeholders

| Placeholder | Description |
|-------------|-------------|
| `${flyway:defaultSchema}` | Default schema |
| `${flyway:user}` | Database user |
| `${flyway:database}` | Database name |
| `${flyway:timestamp}` | Current timestamp |
| `${flyway:filename}` | Current migration filename |

---

## Clean and Repair Commands

### Clean Command

Drops all objects in configured schemas. **DANGEROUS - disabled by default.**

```yaml
spring:
  flyway:
    clean-disabled: false  # Enable (use with caution!)
```

```bash
mvn flyway:clean  # Drops everything!
```

### Repair Command

Fixes the schema history table by:
- Removing failed migrations
- Realigning checksums
- Fixing descriptions

```bash
# Maven
mvn flyway:repair

# Gradle
./gradlew flywayRepair
```

Common repair scenarios:
- Migration file was modified after being applied
- Migration failed mid-way and left dirty state
- Need to sync checksum after intentional file change

---

## Java-Based Migrations

For complex migrations requiring programmatic logic.

### Naming Convention

```
V1__Description.java
R__Description.java
```

### Example Implementation

```java
package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;

public class V3__MigrateUserData extends BaseJavaMigration {

    @Override
    public void migrate(Context context) throws Exception {
        try (var statement = context.getConnection().createStatement()) {
            // Complex data transformation
            try (var rs = statement.executeQuery(
                    "SELECT id, full_name FROM users WHERE first_name IS NULL")) {

                try (var update = context.getConnection().prepareStatement(
                        "UPDATE users SET first_name = ?, last_name = ? WHERE id = ?")) {

                    while (rs.next()) {
                        String[] parts = rs.getString("full_name").split(" ", 2);
                        update.setString(1, parts[0]);
                        update.setString(2, parts.length > 1 ? parts[1] : "");
                        update.setLong(3, rs.getLong("id"));
                        update.executeUpdate();
                    }
                }
            }
        }
    }
}
```

### Spring-Aware Java Migration

```java
package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class V4__ImportExternalData extends BaseJavaMigration {

    @Autowired
    private ExternalDataService externalDataService;

    @Override
    public void migrate(Context context) throws Exception {
        var data = externalDataService.fetchData();
        // Process and insert data
    }
}
```

Enable Spring injection:

```java
@Configuration
public class FlywayConfig {

    @Bean
    public FlywayConfigurationCustomizer flywayCustomizer(ApplicationContext context) {
        return configuration -> configuration
            .javaMigrationClassProvider(new SpringJavaMigrationClassProvider(context));
    }
}
```

---

## Multiple Databases/Schemas

### Multiple Schemas in Single Database

```yaml
spring:
  flyway:
    schemas: public,audit,reporting
    default-schema: public
    create-schemas: true
```

```sql
-- V1__create_multi_schema.sql
CREATE SCHEMA IF NOT EXISTS audit;
CREATE SCHEMA IF NOT EXISTS reporting;

CREATE TABLE public.users (id SERIAL PRIMARY KEY, name VARCHAR(100));
CREATE TABLE audit.user_changes (id SERIAL PRIMARY KEY, user_id INT, changed_at TIMESTAMP);
CREATE TABLE reporting.user_summary (total_users INT, last_updated TIMESTAMP);
```

### Multiple Databases

Configure separate Flyway instances:

```java
@Configuration
public class MultiDatabaseFlywayConfig {

    @Bean
    @Primary
    public Flyway primaryFlyway(@Qualifier("primaryDataSource") DataSource dataSource) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration/primary")
            .load();
    }

    @Bean
    public Flyway auditFlyway(@Qualifier("auditDataSource") DataSource dataSource) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration/audit")
            .table("flyway_schema_history_audit")
            .load();
    }

    @Bean
    public FlywayMigrationInitializer primaryFlywayInitializer(@Qualifier("primaryFlyway") Flyway flyway) {
        return new FlywayMigrationInitializer(flyway);
    }

    @Bean
    public FlywayMigrationInitializer auditFlywayInitializer(@Qualifier("auditFlyway") Flyway flyway) {
        return new FlywayMigrationInitializer(flyway);
    }
}
```

Directory structure for multiple databases:

```
src/main/resources/db/migration/
├── primary/
│   ├── V1__create_users.sql
│   └── V2__create_orders.sql
└── audit/
    ├── V1__create_audit_log.sql
    └── V2__create_audit_triggers.sql
```

---

## CLI and Maven Commands

### Maven Plugin Configuration

```xml
<plugin>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-maven-plugin</artifactId>
    <version>10.0.0</version>
    <configuration>
        <url>jdbc:postgresql://localhost:5432/mydb</url>
        <user>${db.user}</user>
        <password>${db.password}</password>
        <locations>
            <location>classpath:db/migration</location>
        </locations>
    </configuration>
</plugin>
```

### Common Commands

```bash
# Apply pending migrations
mvn flyway:migrate

# Show migration status
mvn flyway:info

# Validate applied migrations
mvn flyway:validate

# Repair schema history
mvn flyway:repair

# Create baseline
mvn flyway:baseline

# Drop all objects (if enabled)
mvn flyway:clean
```

---

## Best Practices

1. **Never modify applied migrations** - Create new migrations for changes
2. **Use meaningful descriptions** - `V5__add_user_email_verification` not `V5__changes`
3. **Keep migrations small and focused** - One logical change per migration
4. **Always test migrations** - Run against a copy of production data
5. **Use transactions** - Wrap DDL in transactions when supported
6. **Version control migrations** - Commit alongside application code
7. **Disable clean in production** - Set `clean-disabled: true`
8. **Use baselines for existing databases** - Don't recreate history
9. **Separate DDL and DML** - Structure changes vs data changes
10. **Use repeatable migrations for views/procedures** - Easier maintenance
11. **Include rollback strategy** - Document manual rollback steps
12. **Use placeholders for environment differences** - Schema names, users, etc.

---

## Common Pitfalls

### 1. Modifying Applied Migrations

**Problem**: Changing a migration after it's been applied causes checksum mismatch.

**Solution**: Use `flyway:repair` or create a new migration.

### 2. Missing Double Underscore

**Problem**: `V1_create_users.sql` (single underscore) is ignored.

**Solution**: Use `V1__create_users.sql` (double underscore).

### 3. Out-of-Order Migrations

**Problem**: Adding V2 after V3 was applied fails by default.

**Solution**: Enable `out-of-order: true` or renumber migrations.

### 4. Case Sensitivity

**Problem**: `v1__create.sql` doesn't work on case-sensitive filesystems.

**Solution**: Always use uppercase `V` and `R` prefixes.

### 5. Running Clean in Production

**Problem**: `flyway:clean` drops all data.

**Solution**: Always set `clean-disabled: true` in production.

### 6. Large Migrations Without Transactions

**Problem**: Failed migration leaves database in inconsistent state.

**Solution**: Use database transactions or break into smaller migrations.

### 7. Hardcoded Environment Values

**Problem**: Schema names, users differ between environments.

**Solution**: Use placeholders: `${schema_name}`, `${admin_user}`.

### 8. Ignoring Validation Errors

**Problem**: Suppressing validation hides real issues.

**Solution**: Fix the root cause - repair or create corrective migration.

### 9. Not Testing Against Production Data

**Problem**: Migration works on empty database, fails on real data.

**Solution**: Test with anonymized production data copy.

### 10. Concurrent Migration Execution

**Problem**: Multiple instances try to migrate simultaneously.

**Solution**: Flyway uses locking, but ensure single deployment or use `group: true`.

---

## Schema History Table

Flyway tracks migrations in `flyway_schema_history`:

```sql
SELECT installed_rank, version, description, type, script,
       checksum, installed_on, execution_time, success
FROM flyway_schema_history
ORDER BY installed_rank;
```

| Column | Description |
|--------|-------------|
| installed_rank | Order of installation |
| version | Migration version |
| description | Migration description |
| type | SQL, JDBC, BASELINE, etc. |
| script | Script filename |
| checksum | File checksum for validation |
| installed_on | Timestamp of installation |
| execution_time | Duration in milliseconds |
| success | Boolean success flag |
