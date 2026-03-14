# Spring Batch Reference

## Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch</artifactId>
</dependency>
```

## Basic Job Configuration

```java
@Configuration
@EnableBatchProcessing
@RequiredArgsConstructor
public class BatchConfig {

    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;

    @Bean
    public Job importUsersJob(Step importStep) {
        return new JobBuilder("importUsersJob", jobRepository)
            .incrementer(new RunIdIncrementer())
            .start(importStep)
            .build();
    }

    @Bean
    public Step importStep(ItemReader<UserCsv> reader,
                          ItemProcessor<UserCsv, User> processor,
                          ItemWriter<User> writer) {
        return new StepBuilder("importStep", jobRepository)
            .<UserCsv, User>chunk(100, transactionManager)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .build();
    }
}
```

## Item Reader

```java
@Bean
public FlatFileItemReader<UserCsv> reader() {
    return new FlatFileItemReaderBuilder<UserCsv>()
        .name("userReader")
        .resource(new ClassPathResource("users.csv"))
        .delimited()
        .names("id", "name", "email")
        .targetType(UserCsv.class)
        .linesToSkip(1)  // Skip header
        .build();
}

// JDBC Reader
@Bean
public JdbcCursorItemReader<User> jdbcReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<User>()
        .name("jdbcReader")
        .dataSource(dataSource)
        .sql("SELECT id, name, email FROM users WHERE status = ?")
        .queryArguments("PENDING")
        .rowMapper(new BeanPropertyRowMapper<>(User.class))
        .build();
}

// JPA Reader
@Bean
public JpaPagingItemReader<User> jpaReader(EntityManagerFactory emf) {
    return new JpaPagingItemReaderBuilder<User>()
        .name("jpaReader")
        .entityManagerFactory(emf)
        .queryString("SELECT u FROM User u WHERE u.status = :status")
        .parameterValues(Map.of("status", "PENDING"))
        .pageSize(100)
        .build();
}
```

## Item Processor

```java
@Component
public class UserProcessor implements ItemProcessor<UserCsv, User> {

    @Override
    public User process(UserCsv item) throws Exception {
        // Return null to filter out item
        if (item.getEmail() == null) {
            return null;
        }

        return User.builder()
            .name(item.getName().toUpperCase())
            .email(item.getEmail().toLowerCase())
            .createdAt(LocalDateTime.now())
            .build();
    }
}

// Composite Processor
@Bean
public CompositeItemProcessor<UserCsv, User> compositeProcessor() {
    return new CompositeItemProcessorBuilder<UserCsv, User>()
        .delegates(
            new ValidationProcessor(),
            new TransformProcessor(),
            new EnrichmentProcessor()
        )
        .build();
}
```

## Item Writer

```java
@Bean
public JdbcBatchItemWriter<User> jdbcWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<User>()
        .dataSource(dataSource)
        .sql("INSERT INTO users (name, email, created_at) VALUES (:name, :email, :createdAt)")
        .beanMapped()
        .build();
}

// JPA Writer
@Bean
public JpaItemWriter<User> jpaWriter(EntityManagerFactory emf) {
    JpaItemWriter<User> writer = new JpaItemWriter<>();
    writer.setEntityManagerFactory(emf);
    return writer;
}

// Composite Writer
@Bean
public CompositeItemWriter<User> compositeWriter() {
    return new CompositeItemWriterBuilder<User>()
        .delegates(
            databaseWriter(),
            cacheWriter(),
            notificationWriter()
        )
        .build();
}
```

## Multi-Step Job

```java
@Bean
public Job multiStepJob(Step extractStep, Step transformStep, Step loadStep) {
    return new JobBuilder("etlJob", jobRepository)
        .start(extractStep)
        .next(transformStep)
        .next(loadStep)
        .build();
}

// Conditional flow
@Bean
public Job conditionalJob(Step step1, Step successStep, Step failureStep) {
    return new JobBuilder("conditionalJob", jobRepository)
        .start(step1)
        .on("COMPLETED").to(successStep)
        .from(step1).on("FAILED").to(failureStep)
        .end()
        .build();
}
```

## Listeners

```java
@Component
public class JobListener implements JobExecutionListener {

    @Override
    public void beforeJob(JobExecution jobExecution) {
        log.info("Job starting: {}", jobExecution.getJobInstance().getJobName());
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
            log.info("Job completed successfully");
        } else {
            log.error("Job failed: {}", jobExecution.getStatus());
        }
    }
}

@Component
public class StepListener implements StepExecutionListener {

    @Override
    public void beforeStep(StepExecution stepExecution) {
        log.info("Step starting: {}", stepExecution.getStepName());
    }

    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        log.info("Step completed: read={}, written={}, skipped={}",
            stepExecution.getReadCount(),
            stepExecution.getWriteCount(),
            stepExecution.getSkipCount());
        return stepExecution.getExitStatus();
    }
}
```

## Skip and Retry

```java
@Bean
public Step faultTolerantStep() {
    return new StepBuilder("faultTolerantStep", jobRepository)
        .<Input, Output>chunk(100, transactionManager)
        .reader(reader())
        .processor(processor())
        .writer(writer())
        .faultTolerant()
        .skip(ValidationException.class)
        .skipLimit(10)
        .retry(TransientDataAccessException.class)
        .retryLimit(3)
        .listener(skipListener())
        .build();
}
```

## Partitioning

```java
@Bean
public Step masterStep(Step workerStep, Partitioner partitioner) {
    return new StepBuilder("masterStep", jobRepository)
        .partitioner("workerStep", partitioner)
        .step(workerStep)
        .gridSize(4)
        .taskExecutor(taskExecutor())
        .build();
}

@Bean
public Partitioner partitioner() {
    return gridSize -> {
        Map<String, ExecutionContext> partitions = new HashMap<>();
        int totalRecords = getTotalRecords();
        int partitionSize = totalRecords / gridSize;

        for (int i = 0; i < gridSize; i++) {
            ExecutionContext context = new ExecutionContext();
            context.putInt("minId", i * partitionSize);
            context.putInt("maxId", (i + 1) * partitionSize - 1);
            partitions.put("partition" + i, context);
        }
        return partitions;
    };
}
```

## Running Jobs

```java
@Service
@RequiredArgsConstructor
public class BatchJobService {

    private final JobLauncher jobLauncher;
    private final Job importJob;

    public void runJob() throws Exception {
        JobParameters params = new JobParametersBuilder()
            .addString("date", LocalDate.now().toString())
            .addLong("timestamp", System.currentTimeMillis())
            .toJobParameters();

        jobLauncher.run(importJob, params);
    }
}
```

## Configuration

```yaml
spring:
  batch:
    job:
      enabled: false  # Don't run on startup
    jdbc:
      initialize-schema: always
```

## Testing

```java
@SpringBatchTest
@SpringBootTest
class BatchJobTest {

    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;

    @Test
    void testJob() throws Exception {
        JobExecution jobExecution = jobLauncherTestUtils.launchJob();

        assertEquals(BatchStatus.COMPLETED, jobExecution.getStatus());
        assertEquals(100, jobExecution.getStepExecutions()
            .iterator().next().getWriteCount());
    }

    @Test
    void testStep() throws Exception {
        JobExecution jobExecution = jobLauncherTestUtils.launchStep("importStep");

        assertEquals(BatchStatus.COMPLETED, jobExecution.getStatus());
    }
}
```
