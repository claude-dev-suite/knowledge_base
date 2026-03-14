# Spring Session - JDBC

## Configuration

### Dependencies
```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

### application.yml
```yaml
spring:
  session:
    store-type: jdbc
    jdbc:
      initialize-schema: always  # always, embedded, never
      table-name: SPRING_SESSION
      cleanup-cron: "0 * * * * *"  # Every minute
      save-mode: on-set-attribute
      flush-mode: on-save
    timeout: 30m

  datasource:
    url: jdbc:postgresql://localhost:5432/sessions
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

### Java Configuration

```java
@Configuration
@EnableJdbcHttpSession(
    maxInactiveIntervalInSeconds = 1800,
    tableName = "SPRING_SESSION",
    cleanupCron = "0 * * * * *",
    flushMode = FlushMode.ON_SAVE,
    saveMode = SaveMode.ON_SET_ATTRIBUTE
)
public class JdbcSessionConfig {

    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

## Database Schema

Spring Session creates two tables:

### PostgreSQL
```sql
CREATE TABLE SPRING_SESSION (
    PRIMARY_ID CHAR(36) NOT NULL,
    SESSION_ID CHAR(36) NOT NULL,
    CREATION_TIME BIGINT NOT NULL,
    LAST_ACCESS_TIME BIGINT NOT NULL,
    MAX_INACTIVE_INTERVAL INT NOT NULL,
    EXPIRY_TIME BIGINT NOT NULL,
    PRINCIPAL_NAME VARCHAR(100),
    CONSTRAINT SPRING_SESSION_PK PRIMARY KEY (PRIMARY_ID)
);

CREATE UNIQUE INDEX SPRING_SESSION_IX1 ON SPRING_SESSION (SESSION_ID);
CREATE INDEX SPRING_SESSION_IX2 ON SPRING_SESSION (EXPIRY_TIME);
CREATE INDEX SPRING_SESSION_IX3 ON SPRING_SESSION (PRINCIPAL_NAME);

CREATE TABLE SPRING_SESSION_ATTRIBUTES (
    SESSION_PRIMARY_ID CHAR(36) NOT NULL,
    ATTRIBUTE_NAME VARCHAR(200) NOT NULL,
    ATTRIBUTE_BYTES BYTEA NOT NULL,
    CONSTRAINT SPRING_SESSION_ATTRIBUTES_PK PRIMARY KEY (SESSION_PRIMARY_ID, ATTRIBUTE_NAME),
    CONSTRAINT SPRING_SESSION_ATTRIBUTES_FK FOREIGN KEY (SESSION_PRIMARY_ID)
        REFERENCES SPRING_SESSION(PRIMARY_ID) ON DELETE CASCADE
);
```

### MySQL
```sql
CREATE TABLE SPRING_SESSION (
    PRIMARY_ID CHAR(36) NOT NULL,
    SESSION_ID CHAR(36) NOT NULL,
    CREATION_TIME BIGINT NOT NULL,
    LAST_ACCESS_TIME BIGINT NOT NULL,
    MAX_INACTIVE_INTERVAL INT NOT NULL,
    EXPIRY_TIME BIGINT NOT NULL,
    PRINCIPAL_NAME VARCHAR(100),
    CONSTRAINT SPRING_SESSION_PK PRIMARY KEY (PRIMARY_ID)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC;

CREATE UNIQUE INDEX SPRING_SESSION_IX1 ON SPRING_SESSION (SESSION_ID);
CREATE INDEX SPRING_SESSION_IX2 ON SPRING_SESSION (EXPIRY_TIME);
CREATE INDEX SPRING_SESSION_IX3 ON SPRING_SESSION (PRINCIPAL_NAME);

CREATE TABLE SPRING_SESSION_ATTRIBUTES (
    SESSION_PRIMARY_ID CHAR(36) NOT NULL,
    ATTRIBUTE_NAME VARCHAR(200) NOT NULL,
    ATTRIBUTE_BYTES BLOB NOT NULL,
    CONSTRAINT SPRING_SESSION_ATTRIBUTES_PK PRIMARY KEY (SESSION_PRIMARY_ID, ATTRIBUTE_NAME),
    CONSTRAINT SPRING_SESSION_ATTRIBUTES_FK FOREIGN KEY (SESSION_PRIMARY_ID)
        REFERENCES SPRING_SESSION(PRIMARY_ID) ON DELETE CASCADE
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC;
```

## Custom Table Names

```java
@Configuration
@EnableJdbcHttpSession(tableName = "APP_SESSION")
public class JdbcSessionConfig {
}
```

```yaml
spring:
  session:
    jdbc:
      table-name: APP_SESSION
```

## Session Cleanup

### Scheduled Cleanup
```yaml
spring:
  session:
    jdbc:
      cleanup-cron: "0 */5 * * * *"  # Every 5 minutes
```

### Manual Cleanup
```java
@Service
@RequiredArgsConstructor
public class SessionCleanupService {

    private final JdbcTemplate jdbcTemplate;

    @Scheduled(cron = "0 0 * * * *")  // Every hour
    public void cleanupExpiredSessions() {
        long now = System.currentTimeMillis();

        int deleted = jdbcTemplate.update(
            "DELETE FROM SPRING_SESSION WHERE EXPIRY_TIME < ?",
            now
        );

        log.info("Cleaned up {} expired sessions", deleted);
    }
}
```

## Session Lookup by Principal

```java
@Service
@RequiredArgsConstructor
public class SessionService {

    private final JdbcIndexedSessionRepository sessionRepository;

    public Map<String, Session> findByUsername(String username) {
        return sessionRepository.findByPrincipalName(username);
    }

    public void invalidateAllSessions(String username) {
        Map<String, Session> sessions = findByUsername(username);
        sessions.keySet().forEach(sessionRepository::deleteById);
    }
}
```

## Save Mode Options

| Mode | Description |
|------|-------------|
| ON_SET_ATTRIBUTE | Save when attribute is set |
| ON_GET_ATTRIBUTE | Save when attribute is accessed |
| ALWAYS | Save after every request |

```yaml
spring:
  session:
    jdbc:
      save-mode: on-set-attribute
```

## Flush Mode Options

| Mode | Description |
|------|-------------|
| ON_SAVE | Flush at end of request |
| IMMEDIATE | Flush immediately on change |

```yaml
spring:
  session:
    jdbc:
      flush-mode: on-save
```

## Transaction Configuration

```java
@Configuration
@EnableJdbcHttpSession
public class JdbcSessionConfig {

    @Bean
    public TransactionOperations transactionOperations(
            PlatformTransactionManager transactionManager) {
        return new TransactionTemplate(transactionManager);
    }
}
```

## Performance Optimization

### Connection Pooling
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      idle-timeout: 300000
      connection-timeout: 20000
      max-lifetime: 1200000
```

### Indexing
```sql
-- Additional indexes for performance
CREATE INDEX idx_session_expiry ON SPRING_SESSION (EXPIRY_TIME);
CREATE INDEX idx_session_principal ON SPRING_SESSION (PRINCIPAL_NAME);
CREATE INDEX idx_session_last_access ON SPRING_SESSION (LAST_ACCESS_TIME);
```

## Best Practices

| Do | Don't |
|----|-------|
| Use connection pooling | Create connections per session |
| Index PRINCIPAL_NAME | Full table scans for lookups |
| Configure cleanup schedule | Let expired sessions accumulate |
| Use appropriate save-mode | Save on every access |
| Monitor table size | Ignore table growth |
| Use transactions | Skip transaction management |
