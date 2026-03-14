# Spring Scheduling Reference

## Enable Scheduling

```java
@SpringBootApplication
@EnableScheduling
public class Application { }
```

## Basic Scheduled Tasks

```java
@Component
public class ScheduledTasks {

    // Fixed rate: runs every 5 seconds (regardless of previous execution)
    @Scheduled(fixedRate = 5000)
    public void fixedRateTask() {
        log.info("Fixed rate task - {}", Instant.now());
    }

    // Fixed delay: runs 5 seconds after previous execution completes
    @Scheduled(fixedDelay = 5000)
    public void fixedDelayTask() {
        log.info("Fixed delay task - {}", Instant.now());
    }

    // Initial delay: waits 10s before first execution
    @Scheduled(fixedRate = 5000, initialDelay = 10000)
    public void withInitialDelay() {
        log.info("Task with initial delay");
    }
}
```

## Cron Expressions

```java
@Component
public class CronTasks {

    // Every day at midnight
    @Scheduled(cron = "0 0 0 * * *")
    public void dailyTask() { }

    // Every hour at minute 0
    @Scheduled(cron = "0 0 * * * *")
    public void hourlyTask() { }

    // Every 15 minutes
    @Scheduled(cron = "0 */15 * * * *")
    public void every15Minutes() { }

    // Weekdays at 9 AM
    @Scheduled(cron = "0 0 9 * * MON-FRI")
    public void weekdayMorning() { }

    // First day of month at midnight
    @Scheduled(cron = "0 0 0 1 * *")
    public void monthlyTask() { }
}
```

## Cron Format

```
┌───────────── second (0-59)
│ ┌───────────── minute (0-59)
│ │ ┌───────────── hour (0-23)
│ │ │ ┌───────────── day of month (1-31)
│ │ │ │ ┌───────────── month (1-12 or JAN-DEC)
│ │ │ │ │ ┌───────────── day of week (0-7 or SUN-SAT)
│ │ │ │ │ │
* * * * * *

Special characters:
*   = any value
,   = value list (MON,WED,FRI)
-   = range (1-5)
/   = step (*/15 = every 15)
```

## Externalized Configuration

```yaml
# application.yml
scheduling:
  cleanup:
    cron: "0 0 2 * * *"
  report:
    rate: 3600000
```

```java
@Scheduled(cron = "${scheduling.cleanup.cron}")
public void cleanupTask() { }

@Scheduled(fixedRateString = "${scheduling.report.rate}")
public void reportTask() { }
```

## Async Scheduling

```java
@Configuration
@EnableAsync
@EnableScheduling
public class AsyncSchedulingConfig {

    @Bean
    public TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(10);
        scheduler.setThreadNamePrefix("scheduled-");
        scheduler.setErrorHandler(t -> log.error("Scheduled task error", t));
        return scheduler;
    }
}

@Component
public class AsyncTasks {

    @Async
    @Scheduled(fixedRate = 1000)
    public void asyncScheduledTask() {
        // Runs in separate thread, doesn't block scheduler
    }
}
```

## Dynamic Scheduling

```java
@Service
@RequiredArgsConstructor
public class DynamicSchedulerService {

    private final TaskScheduler taskScheduler;
    private final Map<String, ScheduledFuture<?>> scheduledTasks = new ConcurrentHashMap<>();

    public void scheduleTask(String taskId, Runnable task, String cronExpression) {
        cancelTask(taskId);
        ScheduledFuture<?> future = taskScheduler.schedule(
            task,
            new CronTrigger(cronExpression)
        );
        scheduledTasks.put(taskId, future);
    }

    public void cancelTask(String taskId) {
        ScheduledFuture<?> future = scheduledTasks.remove(taskId);
        if (future != null) {
            future.cancel(false);
        }
    }
}
```

## Conditional Scheduling

```java
@Scheduled(cron = "${scheduling.enabled:0 0 * * * *}")
@ConditionalOnProperty(name = "scheduling.enabled", havingValue = "true")
public void conditionalTask() { }
```

## Locking (ShedLock)

```xml
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
</dependency>
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-jdbc-template</artifactId>
</dependency>
```

```java
@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "10m")
public class ShedLockConfig {

    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(
            JdbcTemplateLockProvider.Configuration.builder()
                .withJdbcTemplate(new JdbcTemplate(dataSource))
                .usingDbTime()
                .build()
        );
    }
}

@Component
public class LockedTasks {

    @Scheduled(cron = "0 0 * * * *")
    @SchedulerLock(name = "hourlyReport", lockAtLeastFor = "5m", lockAtMostFor = "10m")
    public void hourlyReport() {
        // Only one instance runs this task
    }
}
```

## Error Handling

```java
@Configuration
public class SchedulingErrorConfig {

    @Bean
    public TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);
        scheduler.setErrorHandler(throwable -> {
            log.error("Scheduled task failed", throwable);
            // Send alert, metric, etc.
        });
        return scheduler;
    }
}
```

## Timezone

```java
@Scheduled(cron = "0 0 9 * * *", zone = "Europe/Rome")
public void morningTask() { }
```

## Testing

```java
@SpringBootTest
class ScheduledTasksTest {

    @SpyBean
    private ScheduledTasks scheduledTasks;

    @Test
    void shouldExecuteTask() throws Exception {
        // Wait for task execution
        await().atMost(Duration.ofSeconds(10))
            .untilAsserted(() ->
                verify(scheduledTasks, atLeast(1)).fixedRateTask()
            );
    }
}
```
