# Spring Batch

> **File:** `21_Spring_Batch.md`
> **Part:** 7 — Advanced Topics
> **Prerequisites:** Spring Core, Spring Boot Fundamentals, JDBC/JPA basics
> **Estimated Study Time:** 8–10 hours (read + code + practice)

---

## Table of Contents

1. [What is Spring Batch?](#1-what-is-spring-batch)
2. [Batch Processing Concepts](#2-batch-processing-concepts)
3. [Spring Batch Architecture](#3-spring-batch-architecture)
4. [Job, Step, JobLauncher, JobRepository](#4-job-step-joblauncher-jobrepository)
5. [ItemReader, ItemProcessor, ItemWriter](#5-itemreader-itemprocessor-itemwriter)
6. [Chunk-Oriented Processing](#6-chunk-oriented-processing)
7. [Tasklet Processing](#7-tasklet-processing)
8. [Job Parameters & JobExecutionContext](#8-job-parameters--jobexecutioncontext)
9. [Job & Step Listeners](#9-job--step-listeners)
10. [Skip, Retry & Restart Logic](#10-skip-retry--restart-logic)
11. [Job Scheduling](#11-job-scheduling)
12. [Partitioning & Parallel Processing](#12-partitioning--parallel-processing)
13. [Spring Batch Admin & Monitoring](#13-spring-batch-admin--monitoring)
14. [Interview Questions](#14-interview-questions)
15. [Summary Cheat Sheet](#15-summary-cheat-sheet)

---

## 1. What is Spring Batch?

**Spring Batch** is a lightweight, comprehensive framework for **batch processing** — processing large volumes of data in chunks, with transaction management, retry, skip, restart, and monitoring capabilities.

### Why Spring Batch?

| Without Batch | With Spring Batch |
|---------------|-------------------|
| Manual loops + JDBC | Chunk-oriented processing |
| No restart capability | Restart from where it failed |
| Manual transaction mgmt | Declarative transaction boundaries |
| No skip/retry | Built-in skip/retry policies |
| No monitoring | Job repository + metrics |
| Manual error handling | Declarative exception handling |

### Use Cases

- **ETL** (Extract-Transform-Load) pipelines
- **Data migration** between systems
- **Report generation** from large datasets
- **Bulk imports/exports** (CSV, XML, JSON, DB)
- **Scheduled data synchronization**
- **Financial transactions** (end-of-day processing)

### Key Characteristics

- **Chunk-oriented** processing (read → process → write in transactions)
- **Restartable** — resume from last successful point
- **Scalable** — partitioning, parallel steps
- **Declarative** — XML/Java config
- **Comprehensive** — logging, tracing, monitoring
- **Enterprise-ready** — transaction management, skip/retry

---

## 2. Batch Processing Concepts

### Core Terminology

| Term | Description |
|------|-------------|
| **Job** | A batch process — the top-level container |
| **Step** | A phase of a job (read-process-write or tasklet) |
| **JobInstance** | A logical run of a job (identified by job name + parameters) |
| **JobExecution** | A single attempt to run a JobInstance |
| **StepExecution** | A single attempt to run a Step |
| **JobParameters** | Input parameters identifying a job instance |
| **JobRepository** | Metadata store for job/step executions |
| **JobLauncher** | Entry point to launch jobs |
| **ItemReader** | Reads input data |
| **ItemProcessor** | Transforms/filters data |
| **ItemWriter** | Writes output data |
| **Chunk** | A batch of items processed within one transaction |

### Batch vs Online Processing

| Aspect | Batch | Online (Web) |
|--------|-------|--------------|
| Data volume | Large | Small per request |
| Latency | Minutes/hours | Milliseconds |
| Concurrency | Sequential/partitioned | Highly concurrent |
| Failure handling | Restart, skip | Retry, rollback |
| State | Persistent (JobRepository) | Stateless (usually) |
| Transaction | Long/chunked | Short |

---

## 3. Spring Batch Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         JOB                                  │
│  ┌───────────────────────────────────────────────────────┐   │
│  │                       STEP 1                          │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │   │
│  │  │ Item     │→ │ Item     │→ │ Item     │            │   │
│  │  │ Reader   │  │ Processor│  │ Writer   │            │   │
│  │  └──────────┘  └──────────┘  └──────────┘            │   │
│  │         [ CHUNK: read/process N items, write in TX ] │   │
│  └───────────────────────────────────────────────────────┘   │
│                              │                               │
│                              ▼                               │
│  ┌───────────────────────────────────────────────────────┐   │
│  │                       STEP 2                          │   │
│  │              (Tasklet or Chunk)                       │   │
│  └───────────────────────────────────────────────────────┘   │
│                              │                               │
│                              ▼                               │
│                     JobRepository                            │
│                   (BATCH_JOB_INSTANCE,                       │
│                    BATCH_JOB_EXECUTION,                      │
│                    BATCH_STEP_EXECUTION, ...)                │
└─────────────────────────────────────────────────────────────┘
```

### Major Components

| Component | Responsibility |
|-----------|----------------|
| `JobLauncher` | Launches jobs |
| `Job` | Top-level batch process |
| `Step` | Phase of a job |
| `JobRepository` | Persists metadata |
| `PlatformTransactionManager` | Manages transactions |
| `ItemReader` / `ItemProcessor` / `ItemWriter` | Data pipeline |
| `JobExplorer` | Read-only view of repository |
| `JobOperator` | Administrative operations (stop, restart) |

### Maven Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch</artifactId>
</dependency>
```

### Enabling Batch

```java
@SpringBootApplication
@EnableBatchProcessing   // optional in Spring Boot 3+ (auto-configured)
public class BatchApplication {
    public static void main(String[] args) {
        SpringApplication.run(BatchApplication.class, args);
    }
}
```

> **Note:** Since Spring Boot 3 / Spring Batch 5, `@EnableBatchProcessing` is no longer required — auto-configuration handles it. But you may still add it for custom configuration.

---

## 4. Job, Step, JobLauncher, JobRepository

### Defining a Job

```java
@Configuration
public class BatchConfig {
    
    @Bean
    public Job importUserJob(JobRepository jobRepository,
                             Step step1,
                             Step step2,
                             JobCompletionNotificationListener listener) {
        return new JobBuilder("importUserJob", jobRepository)
            .listener(listener)
            .start(step1)
            .next(step2)
            .build();
    }
}
```

**JobBuilder fluent API:**

```java
new JobBuilder("myJob", jobRepository)
    .start(step1)
    .on("COMPLETED").to(step2)
    .from(step2).on("FAILED").to(step3)
    .from(step3).end()
    .build();
```

### Defining a Step

**Chunk-oriented step:**

```java
@Bean
public Step chunkStep(JobRepository jobRepository,
                      PlatformTransactionManager txManager,
                      ItemReader<User> reader,
                      ItemProcessor<User, User> processor,
                      ItemWriter<User> writer) {
    return new StepBuilder("chunkStep", jobRepository)
        .<User, User>chunk(10, txManager)
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .build();
}
```

**Tasklet step:**

```java
@Bean
public Step taskletStep(JobRepository jobRepository,
                        PlatformTransactionManager txManager) {
    return new StepBuilder("taskletStep", jobRepository)
        .tasklet((contribution, chunkContext) -> {
            System.out.println("Hello from tasklet!");
            return RepeatStatus.FINISHED;
        }, txManager)
        .build();
}
```

### JobLauncher

```java
@Autowired
private JobLauncher jobLauncher;

@Autowired
private Job myJob;

public void runJob() throws Exception {
    JobParameters params = new JobParametersBuilder()
        .addString("inputFile", "/data/input.csv")
        .addLong("timestamp", System.currentTimeMillis())
        .toJobParameters();
    
    JobExecution execution = jobLauncher.run(myJob, params);
    System.out.println("Status: " + execution.getStatus());
}
```

### JobRepository

The `JobRepository` persists metadata about jobs and steps. By default, Spring Batch creates these tables:

| Table | Purpose |
|-------|---------|
| `BATCH_JOB_INSTANCE` | Unique job runs |
| `BATCH_JOB_EXECUTION` | Each attempt of a job instance |
| `BATCH_JOB_EXECUTION_PARAMS` | Job parameters |
| `BATCH_JOB_EXECUTION_CONTEXT` | Job-level context |
| `BATCH_STEP_EXECUTION` | Each step execution |
| `BATCH_STEP_EXECUTION_CONTEXT` | Step-level context |

**Schema initialization:**

```properties
spring.batch.jdbc.initialize-schema=always   # or embedded (default), never
```

**Custom schema scripts:**
```
org/springframework/batch/core/schema-postgresql.sql
org/springframework/batch/core/schema-mysql.sql
org/springframework/batch/core/schema-oracle.sql
```

---

## 5. ItemReader, ItemProcessor, ItemWriter

### ItemReader

Reads items one at a time; returns `null` when input is exhausted.

```java
public interface ItemReader<T> {
    T read() throws Exception, UnexpectedInputException, ParseException;
}
```

**FlatFileItemReader (CSV):**

```java
@Bean
public FlatFileItemReader<User> reader() {
    return new FlatFileItemReaderBuilder<User>()
        .name("userItemReader")
        .resource(new ClassPathResource("users.csv"))
        .delimited()
        .names("id", "name", "email", "age")
        .targetType(User.class)
        .linesToSkip(1)   // skip header
        .build();
}
```

**JdbcCursorItemReader:**

```java
@Bean
public JdbcCursorItemReader<User> jdbcReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<User>()
        .name("jdbcReader")
        .dataSource(dataSource)
        .sql("SELECT id, name, email, age FROM users WHERE active = true")
        .rowMapper((rs, rowNum) -> new User(
            rs.getLong("id"),
            rs.getString("name"),
            rs.getString("email"),
            rs.getInt("age")))
        .build();
}
```

**JpaPagingItemReader:**

```java
@Bean
public JpaPagingItemReader<User> jpaReader(EntityManagerFactory emf) {
    return new JpaPagingItemReaderBuilder<User>()
        .name("jpaReader")
        .entityManagerFactory(emf)
        .queryString("SELECT u FROM User u WHERE u.active = true")
        .pageSize(100)
        .build();
}
```

**Common readers:**

| Reader | Use Case |
|--------|----------|
| `FlatFileItemReader` | CSV / delimited / fixed-width files |
| `JsonItemReader` | JSON files |
| `StaxEventItemReader` | XML files |
| `JdbcCursorItemReader` | Stream DB rows (cursor) |
| `JdbcPagingItemReader` | Paged DB rows |
| `JpaPagingItemReader` | Paged JPA entities |
| `HibernateCursorItemReader` | Hibernate streaming |
| `MongoItemReader` | MongoDB |
| `KafkaItemReader` | Kafka topics |
| `MultiResourceItemReader` | Multiple files |
| `ListItemReader` | List (testing) |

### ItemProcessor

Transforms or filters items. Return `null` to **filter out** the item.

```java
public interface ItemProcessor<I, O> {
    O process(I item) throws Exception;
}
```

**Example — transform + filter:**

```java
@Component
public class UserProcessor implements ItemProcessor<User, UserDTO> {
    
    @Override
    public UserDTO process(User user) {
        // Filter: skip underage
        if (user.getAge() < 18) {
            return null;   // filtered out, not written
        }
        
        // Transform
        return new UserDTO(
            user.getId(),
            user.getName().toUpperCase(),
            maskEmail(user.getEmail()),
            user.getAge()
        );
    }
    
    private String maskEmail(String email) {
        return email.replaceAll("(.).*@", "$1***@");
    }
}
```

### ItemWriter

Writes items in batches (one call per chunk).

```java
public interface ItemWriter<T> {
    void write(Chunk<? extends T> chunk) throws Exception;
}
```

**FlatFileItemWriter (CSV out):**

```java
@Bean
public FlatFileItemWriter<UserDTO> writer() {
    return new FlatFileItemWriterBuilder<UserDTO>()
        .name("userItemWriter")
        .resource(new FileSystemResource("/output/users-out.csv"))
        .delimited()
        .delimiter(",")
        .names("id", "name", "email", "age")
        .headerCallback(w -> w.write("id,name,email,age"))
        .build();
}
```

**JdbcBatchItemWriter:**

```java
@Bean
public JdbcBatchItemWriter<UserDTO> jdbcWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<UserDTO>()
        .dataSource(dataSource)
        .sql("INSERT INTO user_dto (id, name, email, age) " +
             "VALUES (:id, :name, :email, :age)")
        .beanMapped()
        .build();
}
```

**JpaItemWriter:**

```java
@Bean
public JpaItemWriter<User> jpaWriter(EntityManagerFactory emf) {
    JpaItemWriter<User> writer = new JpaItemWriter<>();
    writer.setEntityManagerFactory(emf);
    return writer;
}
```

**Common writers:**

| Writer | Use Case |
|--------|----------|
| `FlatFileItemWriter` | CSV / delimited |
| `JsonFileItemWriter` | JSON |
| `StaxEventItemWriter` | XML |
| `JdbcBatchItemWriter` | Batch INSERT via JDBC |
| `JpaItemWriter` | JPA persist/merge |
| `HibernateItemWriter` | Hibernate persist/merge |
| `MongoItemWriter` | MongoDB |
| `KafkaItemWriter` | Kafka |
| `CompositeItemWriter` | Multiple writers |
| `ClassifierCompositeItemWriter` | Route by item type |
| `MultiResourceItemWriter` | Split output across files |

---

## 6. Chunk-Oriented Processing

### How It Works

```
For each chunk of size N:
  1. Begin transaction
  2. Loop N times:
       item = reader.read()
       if item == null: break
       processed = processor.process(item)
       if processed != null: buffer.add(processed)
  3. writer.write(buffer)
  4. Commit transaction
  5. Update StepExecution metadata
Repeat until reader returns null
```

### Diagram

```
Step
 │
 ├── Chunk 1: [R R R R R P P P P P W] COMMIT
 ├── Chunk 2: [R R R R R P P P P P W] COMMIT
 ├── Chunk 3: [R R R R R P P P P P W] COMMIT
 └── Chunk 4: [R R R null] COMMIT (partial chunk)
```

### Configuration

```java
@Bean
public Step step(JobRepository jobRepository,
                 PlatformTransactionManager txManager) {
    return new StepBuilder("step", jobRepository)
        .<User, UserDTO>chunk(100, txManager)   // chunk size = 100
        .reader(reader())
        .processor(processor())
        .writer(writer())
        .build();
}
```

### Chunk Size Trade-offs

| Chunk Size | Pros | Cons |
|-----------|------|------|
| Small (1–10) | Frequent commits, low rollback cost | Overhead of TX per chunk |
| Medium (50–500) | Balanced | — |
| Large (1000+) | Fewer commits, faster | Larger rollback on failure, more memory |

**Rule of thumb:** Chunk size = **performance target / item processing time**.

### Commit Interval

The chunk size = **commit interval** — number of items per transaction.

### `ItemStream` Interface

For resources needing open/close:

```java
public interface ItemStream {
    void open(ExecutionContext ctx) throws Exception;
    void update(ExecutionContext ctx) throws Exception;
    void close() throws Exception;
}
```

Most readers/writers implement `ItemStream` for restart support.

---

## 7. Tasklet Processing

A **Tasklet** is a single unit of work without chunk semantics — good for:

- File cleanup
- Stored procedure calls
- Sending notifications
- Simple DB updates

### Inline Tasklet

```java
@Bean
public Step cleanupStep(JobRepository jobRepository,
                        PlatformTransactionManager txManager) {
    return new StepBuilder("cleanupStep", jobRepository)
        .tasklet((contribution, chunkContext) -> {
            // Do work
            File tempDir = new File("/tmp/batch");
            FileUtils.cleanDirectory(tempDir);
            
            return RepeatStatus.FINISHED;   // or CONTINUABLE
        }, txManager)
        .build();
}
```

### Tasklet Class

```java
@Component
public class FileCleanupTasklet implements Tasklet {
    
    @Override
    public RepeatStatus execute(StepContribution contribution,
                                ChunkContext chunkContext) throws Exception {
        String path = (String) chunkContext.getStepContext()
            .getJobParameters().get("path");
        
        Files.walk(Paths.get(path))
             .filter(Files::isRegularFile)
             .forEach(p -> { try { Files.delete(p); } catch (Exception e) {} });
        
        return RepeatStatus.FINISHED;
    }
}
```

### RepeatStatus

| Value | Meaning |
|-------|---------|
| `FINISHED` | Tasklet done |
| `CONTINUABLE` | Call again (loop) |

### Chunk vs Tasklet

| Aspect | Chunk | Tasklet |
|--------|-------|---------|
| Data pipeline | ✅ | ❌ |
| Transaction | Per chunk | Per tasklet |
| Use | ETL, bulk processing | Cleanup, setup, notification |
| Restart | Position tracking | Manual (via context) |

---

## 8. Job Parameters & JobExecutionContext

### JobParameters

Identify a specific **JobInstance**. Same job + same params = same instance (won't re-run unless restarted).

```java
JobParameters params = new JobParametersBuilder()
    .addString("inputFile", "/data/input.csv")
    .addLong("runDate", System.currentTimeMillis())   // unique per run
    .addDouble("threshold", 0.75)
    .addDate("startDate", new Date())
    .toJobParameters();
```

**Accessing:**

```java
@Component
public class MyTasklet implements Tasklet {
    @Override
    public RepeatStatus execute(StepContribution contrib, ChunkContext ctx) {
        String file = ctx.getStepContext().getJobParameters()
                        .get("inputFile").toString();
        return RepeatStatus.FINISHED;
    }
}
```

**Spring Boot auto-run:**

```properties
spring.batch.job.name=importUserJob
spring.batch.job.enabled=true
```

**Command line:**
```bash
java -jar app.jar --spring.batch.job.name=importUserJob inputFile=/data/in.csv
```

### JobExecutionContext vs StepExecutionContext

| Context | Scope | Persistence |
|---------|-------|-------------|
| `JobExecutionContext` | Whole job, all steps | BATCH_JOB_EXECUTION_CONTEXT |
| `StepExecutionContext` | Single step | BATCH_STEP_EXECUTION_CONTEXT |

**Storing:**

```java
// Job-level (persists across steps)
chunkContext.getStepContext().getStepExecution()
    .getJobExecution().getExecutionContext()
    .put("totalProcessed", 12345);

// Step-level (persists for restart within a step)
chunkContext.getStepContext().getStepExecution()
    .getExecutionContext()
    .put("lastId", 500);
```

**Reading:**

```java
ExecutionContext jobCtx = chunkContext.getStepContext()
    .getStepExecution().getJobExecution().getExecutionContext();
Object total = jobCtx.get("totalProcessed");
```

### Promoting Step → Job

```java
ExecutionContext stepCtx = chunkContext.getStepContext()
    .getStepExecution().getExecutionContext();
stepCtx.put("count", 100);

// Promotion listener (use ExecutionContextPromotionListener)
```

```java
@Bean
public ExecutionContextPromotionListener promotionListener() {
    ExecutionContextPromotionListener l = new ExecutionContextPromotionListener();
    l.setKeys(new String[]{"count"});
    return l;
}
```

---

## 9. Job & Step Listeners

### Listener Interfaces

| Listener | Level | Methods |
|----------|-------|---------|
| `JobExecutionListener` | Job | `beforeJob`, `afterJob` |
| `StepExecutionListener` | Step | `beforeStep`, `afterStep` |
| `ChunkListener` | Chunk | `beforeChunk`, `afterChunk`, `afterChunkError` |
| `ItemReadListener` | Item | `beforeRead`, `afterRead`, `onReadError` |
| `ItemProcessListener` | Item | `beforeProcess`, `afterProcess`, `onProcessError` |
| `ItemWriteListener` | Item | `beforeWrite`, `afterWrite`, `onWriteError` |
| `SkipListener` | Skip | `onSkipInRead`, `onSkipInProcess`, `onSkipInWrite` |

### Job Listener Example

```java
@Component
public class JobCompletionNotificationListener 
        implements JobExecutionListener {
    
    @Override
    public void beforeJob(JobExecution jobExecution) {
        System.out.println("Job starting: " + jobExecution.getJobInstance().getJobName());
    }
    
    @Override
    public void afterJob(JobExecution jobExecution) {
        if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
            System.out.println("Job finished. Duration: " +
                (jobExecution.getEndTime().getTime() - 
                 jobExecution.getStartTime().getTime()) + " ms");
        } else {
            System.err.println("Job failed: " + jobExecution.getExitStatus());
        }
    }
}
```

### Registering Listeners

**Job listener:**

```java
new JobBuilder("job", repo)
    .listener(jobListener)
    .start(step)
    .build();
```

**Step listener:**

```java
new StepBuilder("step", repo)
    .<I, O>chunk(10, tx)
    .listener(stepListener)
    .listener(chunkListener)
    .listener(itemReadListener)
    .build();
```

### Annotations (Spring Batch 4.2+)

```java
@Component
public class AnnotatedListeners {
    
    @BeforeJob
    public void beforeJob(JobExecution je) { }
    
    @AfterJob
    public void afterJob(JobExecution je) { }
    
    @BeforeStep
    public void beforeStep(StepExecution se) { }
    
    @AfterStep
    public ExitStatus afterStep(StepExecution se) { return null; }
    
    @BeforeChunk
    public void beforeChunk(ChunkContext ctx) { }
    
    @AfterChunk
    public void afterChunk(ChunkContext ctx) { }
    
    @OnReadError
    public void onReadError(Exception ex) { }
    
    @OnSkipInRead
    public void onSkipInRead(Throwable t) { }
}
```

### Multiple Listeners & Ordering

```java
.listener(firstListener)
.listener(secondListener)
```

Or use `@Order` on `@Component` listeners.

---

## 10. Skip, Retry & Restart Logic

### Fault Tolerance

Spring Batch provides three fault-tolerance strategies:

1. **Skip** — ignore and continue
2. **Retry** — attempt again
3. **Restart** — resume from checkpoint

### Skip

```java
@Bean
public Step step(JobRepository jobRepository,
                 PlatformTransactionManager txManager) {
    return new StepBuilder("step", jobRepository)
        .<User, UserDTO>chunk(10, txManager)
        .reader(reader())
        .processor(processor())
        .writer(writer())
        .faultTolerant()
        .skip(FlatFileParseException.class)
        .skip(ValidationException.class)
        .skipLimit(100)                        // max 100 skips, then fail
        .noSkip(FileNotFoundException.class)   // never skip these
        .build();
}
```

### Retry

```java
.faultTolerant()
.retry(DeadlockLoserDataAccessException.class)
.retryLimit(3)
.noRetry(FileNotFoundException.class)
.backOffPolicy(new ExponentialBackOffPolicy())
```

Or custom:

```java
.retryPolicy(new SimpleRetryPolicy(3, 
    Map.of(DeadlockLoserDataAccessException.class, true)))
```

### Skip + Retry Together

```java
.faultTolerant()
.skip(ValidationException.class)
.skipLimit(100)
.retry(TransientDataAccessException.class)
.retryLimit(3)
```

**Order:** Retry first (same item), then skip (move on).

### SkipListener

```java
@Component
public class MySkipListener implements SkipListener<User, UserDTO> {
    
    @Override
    public void onSkipInRead(Throwable t) {
        log.warn("Skipped in read: {}", t.getMessage());
    }
    
    @Override
    public void onSkipInProcess(User item, Throwable t) {
        log.warn("Skipped in process: {}", item.getId());
    }
    
    @Override
    public void onSkipInWrite(UserDTO item, Throwable t) {
        log.warn("Skipped in write: {}", item.getId());
    }
}
```

### Restart

A failed job can be **restarted** with the same `JobParameters`:

```java
JobExecution execution = jobLauncher.run(job, params);
// If FAILED, calling again with same params restarts
```

**Restart requires:**

1. `JobRepository` configured (default in Boot)
2. `ItemReader`/`ItemWriter` implement `ItemStream`
3. Same `JobParameters`

**Controlling restart:**

```java
@Bean
public Job job(JobRepository repo) {
    return new JobBuilder("job", repo)
        .start(step)
        .preventRestart()              // disallow restart
        .restartable(true)             // (default)
        .build();
}
```

**JobExplorer to find failed jobs:**

```java
@Autowired
private JobExplorer jobExplorer;

public void findFailedJobs() {
    List<JobInstance> instances = jobExplorer.findJobInstancesByJobName(
        "importUserJob", 0, 10);
    for (JobInstance inst : instances) {
        List<JobExecution> execs = jobExplorer.getJobExecutions(inst);
        execs.stream()
            .filter(e -> e.getStatus() == BatchStatus.FAILED)
            .forEach(e -> restart(e));
    }
}
```

**JobOperator for programmatic control:**

```java
@Autowired
private JobOperator jobOperator;

Long executionId = jobOperator.startNextInstance("importUserJob");
jobOperator.stop(executionId);
jobOperator.restart(executionId);
jobOperator.abandon(executionId);
```

### ExitStatus vs BatchStatus

| Status | Meaning |
|--------|---------|
| `COMPLETED` | Successful |
| `FAILED` | Failed |
| `STOPPED` | Manually stopped |
| `STARTED` | Running |
| `STARTING` | Starting |
| `ABANDONED` | Abandoned |

`ExitStatus` = customizable string (e.g., `COMPLETED WITH SKIPS`).

---

## 11. Job Scheduling

### Option 1: Spring `@Scheduled`

```java
@Component
public class JobScheduler {
    
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job myJob;
    
    @Scheduled(cron = "0 0 2 * * ?")   // 2 AM daily
    public void runJob() throws Exception {
        JobParameters params = new JobParametersBuilder()
            .addLong("time", System.currentTimeMillis())
            .toJobParameters();
        jobLauncher.run(myJob, params);
    }
}
```

Enable scheduling:

```java
@Configuration
@EnableScheduling
public class SchedulerConfig { }
```

**Cron format:** `second minute hour day-of-month month day-of-week`

Examples:
- `0 0 2 * * ?` — 2 AM daily
- `0 */15 * * * *` — every 15 minutes
- `0 0 0 1 * ?` — 1st of month at midnight

### Option 2: Quartz

**Dependency:**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

**Configuration:**

```java
@Configuration
public class QuartzConfig {
    
    @Bean
    public JobDetail jobDetail() {
        return JobBuilder.newJob(QuartzJobLauncher.class)
            .withIdentity("myBatchJob")
            .storeDurably()
            .build();
    }
    
    @Bean
    public Trigger trigger(JobDetail jobDetail) {
        return TriggerBuilder.newTrigger()
            .forJob(jobDetail)
            .withIdentity("myTrigger")
            .withSchedule(CronScheduleBuilder.cronSchedule("0 0 2 * * ?"))
            .build();
    }
}
```

**Quartz job:**

```java
public class QuartzJobLauncher extends QuartzJobBean {
    
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job myBatchJob;
    
    @Override
    protected void executeInternal(JobExecutionContext context) {
        JobParameters params = new JobParametersBuilder()
            .addLong("time", System.currentTimeMillis())
            .toJobParameters();
        jobLauncher.run(myBatchJob, params);
    }
}
```

> ⚠️ By default, Quartz's `SpringBeanJobFactory` doesn't autowire. Use `AutowireCapableBeanFactory`:

```java
public class AutowiringSpringBeanJobFactory extends SpringBeanJobFactory 
        implements ApplicationContextAware {
    
    private AutowireCapableBeanFactory beanFactory;
    
    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        beanFactory = ctx.getAutowireCapableBeanFactory();
    }
    
    @Override
    protected Object createJobInstance(TriggerFiredBundle bundle) 
            throws Exception {
        Object job = super.createJobInstance(bundle);
        beanFactory.autowireBean(job);
        return job;
    }
}
```

### Option 3: External Scheduler

- Kubernetes CronJob
- Jenkins
- Airflow
- Linux cron

Call a REST endpoint or run `java -jar app.jar --spring.batch.job.name=job`.

---

## 12. Partitioning & Parallel Processing

### Partitioning

Split a step's work across multiple threads/processes.

```
        Master Step
             │
    ┌────────┼────────┐
    ▼        ▼        ▼
  Step 1   Step 2   Step 3  (parallel on partitions)
```

**Configuration:**

```java
@Bean
public Step masterStep(JobRepository jobRepository,
                       PlatformTransactionManager txManager,
                       Step slaveStep) {
    return new StepBuilder("masterStep", jobRepository)
        .partitioner("slaveStep", partitioner())
        .step(slaveStep)
        .gridSize(4)              // 4 partitions
        .taskExecutor(taskExecutor())
        .build();
}

@Bean
public Partitioner partitioner() {
    return gridSize -> {
        Map<String, ExecutionContext> result = new HashMap<>();
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putInt("partitionIndex", i);
            ctx.putLong("minId", i * 1000L);
            ctx.putLong("maxId", (i + 1) * 1000L - 1);
            result.put("partition" + i, ctx);
        }
        return result;
    };
}

@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
    exec.setCorePoolSize(4);
    exec.setMaxPoolSize(4);
    exec.initialize();
    return exec;
}
```

**Slave step uses partition context:**

```java
@Bean
@StepScope
public JdbcPagingItemReader<User> reader(
        DataSource ds,
        @Value("#{stepExecutionContext['minId']}") Long minId,
        @Value("#{stepExecutionContext['maxId']}") Long maxId) {
    return new JdbcPagingItemReaderBuilder<User>()
        .dataSource(ds)
        .name("pagingReader")
        .selectClause("SELECT *")
        .fromClause("FROM users")
        .whereClause("id BETWEEN " + minId + " AND " + maxId)
        .sortKeys(Map.of("id", Order.ASCENDING))
        .pageSize(100)
        .rowMapper(userRowMapper)
        .build();
}
```

### Multi-threaded Step

Simpler than partitioning — same step, multiple threads:

```java
@Bean
public Step multithreadedStep(JobRepository repo,
                              PlatformTransactionManager tx) {
    return new StepBuilder("multiStep", repo)
        .<User, User>chunk(100, tx)
        .reader(reader())
        .writer(writer())
        .taskExecutor(taskExecutor())   // concurrent chunks
        .throttleLimit(4)
        .build();
}
```

> ⚠️ **Reader must be thread-safe** (most are not!). Use `SynchronizedItemStreamReader`:

```java
@Bean
public SynchronizedItemStreamReader<User> syncReader() {
    SynchronizedItemStreamReader<User> r = new SynchronizedItemStreamReader<>();
    r.setDelegate(unsynchronizedReader());
    return r;
}
```

### Parallel Steps (Split)

Run different steps concurrently:

```java
@Bean
public Job parallelJob(JobRepository repo, Step step1, Step step2, Step step3) {
    Flow flow1 = new FlowBuilder<Flow>("flow1").from(step1).build();
    Flow flow2 = new FlowBuilder<Flow>("flow2").from(step2).build();
    Flow splitFlow = new FlowBuilder<Flow>("splitFlow")
        .split(new SimpleAsyncTaskExecutor())
        .add(flow1, flow2)
        .build();
    
    return new JobBuilder("parallelJob", repo)
        .start(splitFlow)
        .next(step3)
        .end()
        .build();
}
```

### Remote Chunking vs Remote Partitioning

| Aspect | Remote Chunking | Remote Partitioning |
|--------|-----------------|---------------------|
| Architecture | Master reads, workers process/write | Each worker reads its partition |
| Data transfer | Items sent over network | Metadata only |
| Bottleneck | Master (single reader) | No central bottleneck |
| Complexity | Lower | Higher |
| Use case | Medium scale | Very large scale |

---

## 13. Spring Batch Admin & Monitoring

### Spring Boot Actuator Integration

**Dependency:**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

**Endpoints:**

```properties
management.endpoints.web.exposure.include=health,info,metrics,batch
```

**Available:**
- `GET /actuator/batch/jobs` — list jobs
- `GET /actuator/batch/jobs/{name}` — job executions
- `GET /actuator/batch/jobExecutions/{id}` — details

### Micrometer Metrics

Spring Batch auto-registers metrics:

- `spring.batch.job` — job executions (tag: `status`, `job.name`)
- `spring.batch.step` — step executions

**Prometheus scrape:**

```
spring_batch_job_duration_seconds_sum{job_name="importUserJob"} 45.2
```

### Custom Monitoring

**JobExecutionListener:**

```java
@Component
public class MetricsListener implements JobExecutionListener {
    
    private final MeterRegistry registry;
    
    public MetricsListener(MeterRegistry registry) {
        this.registry = registry;
    }
    
    @Override
    public void afterJob(JobExecution execution) {
        registry.counter("batch.job.completed",
            "job", execution.getJobInstance().getJobName(),
            "status", execution.getStatus().toString())
            .increment();
        
        registry.timer("batch.job.duration",
            "job", execution.getJobInstance().getJobName())
            .record(Duration.between(
                execution.getStartTime().toInstant(),
                execution.getEndTime().toInstant()));
    }
}
```

### Querying the Job Repository

```java
@Autowired
private JobExplorer jobExplorer;

public void report() {
    List<JobInstance> instances = jobExplorer.findJobInstancesByJobName(
        "importUserJob", 0, 100);
    
    for (JobInstance inst : instances) {
        for (JobExecution exec : jobExplorer.getJobExecutions(inst)) {
            System.out.printf("Execution %d: %s (%d reads, %d writes)%n",
                exec.getId(),
                exec.getStatus(),
                exec.getStepExecutions().stream()
                    .mapToLong(StepExecution::getReadCount).sum(),
                exec.getStepExecutions().stream()
                    .mapToLong(StepExecution::getWriteCount).sum());
        }
    }
}
```

### Retry via JobOperator

```java
@Autowired private JobOperator jobOperator;

public void restartFailedJob(Long executionId) throws Exception {
    JobExecution exec = jobExplorer.getJobExecution(executionId);
    if (exec.getStatus() == BatchStatus.FAILED) {
        jobOperator.restart(executionId);
    }
}
```

### Production Best Practices

1. **Use a database** JobRepository (not in-memory) in production
2. **Enable restart** — never lose progress
3. **Monitor with Micrometer** — alert on failures
4. **Set reasonable chunk sizes** (100–1000)
5. **Use `@StepScope`** for late binding of parameters
6. **Avoid stateful readers** in multithreaded steps
7. **Clean old job executions** periodically:
   ```sql
   DELETE FROM BATCH_JOB_EXECUTION WHERE CREATE_TIME < NOW() - INTERVAL 30 DAY;
   ```
8. **Secure Actuator** endpoints
9. **Log job parameters** — helps debugging
10. **Test with `JobLauncherTestUtils`**

---

## 14. Interview Questions

### Q1. What is Spring Batch and when would you use it?

**Answer:** Spring Batch is a framework for batch processing large volumes of data with support for chunk-oriented processing, transactions, restart, skip/retry, and monitoring. Use it for ETL, data migration, report generation, bulk imports/exports, and scheduled data synchronization — anywhere you process thousands to millions of records reliably.

### Q2. Explain the architecture of Spring Batch.

**Answer:** A **Job** is the top-level process containing one or more **Steps**. Each Step is either chunk-oriented (ItemReader → ItemProcessor → ItemWriter) or tasklet-based. The **JobRepository** persists metadata (JobInstance, JobExecution, StepExecution). **JobLauncher** starts jobs. **JobExplorer** reads metadata. **JobOperator** performs admin operations.

### Q3. What is chunk-oriented processing?

**Answer:** Process items in batches (chunks): read N items, optionally process them, write the batch in a single transaction. The chunk size = commit interval. This gives efficient batching, bounded memory, and restart capability. Failed chunks roll back cleanly.

### Q4. Difference between Tasklet and Chunk?

**Answer:** Tasklet = single unit of work (no read/process/write). Chunk = read → process → write loop in transactions. Use Tasklet for cleanup, notifications, procedures. Use Chunk for ETL/data processing.

### Q5. How does Spring Batch achieve restartability?

**Answer:** 1) `JobRepository` persists execution state after each chunk. 2) `ItemReader`/`ItemWriter` implement `ItemStream` and store position in `ExecutionContext`. 3) On restart with the same `JobParameters`, the job resumes from the last checkpoint. Requires same params and DB-backed repository.

### Q6. Difference between JobInstance, JobExecution, StepExecution?

**Answer:** 
- **JobInstance** = logical run of a job (identified by name + JobParameters).
- **JobExecution** = one attempt to run a JobInstance (may fail and be retried).
- **StepExecution** = one attempt to run a Step within a JobExecution.

### Q7. How do you implement skip and retry?

**Answer:** In `StepBuilder`, `.faultTolerant().skip(Exception.class).skipLimit(100).retry(TransientException.class).retryLimit(3)`. Retry attempts the same item; skip moves past it. Use `SkipListener` to log skips.

### Q8. What is a Partitioner?

**Answer:** Splits a step's work into partitions (by grid size), each with its own `ExecutionContext`. The master step runs each partition as a slave step (possibly in parallel via TaskExecutor). Enables horizontal scaling for large datasets.

### Q9. Difference between multi-threaded step and partitioning?

**Answer:** Multi-threaded step: single step with concurrent chunk processing using a `TaskExecutor`. Reader must be thread-safe (`SynchronizedItemStreamReader`). Partitioning: each partition is a separate step execution with its own reader/writer/context. Partitioning scales better, but is more complex.

### Q10. How do you schedule a Spring Batch job?

**Answer:** 1) `@Scheduled` + `@EnableScheduling` (simple). 2) **Quartz** for clustered scheduling with persistent triggers. 3) External scheduler (Kubernetes CronJob, Jenkins, Airflow) calling `java -jar` or a REST endpoint.

### Q11. What is @StepScope?

**Answer:** A step-scoped bean — created per step execution, allows late binding of `JobParameters` and `ExecutionContext` via `@Value("#{jobParameters['x']}")` or `@Value("#{stepExecutionContext['y']}")`. Essential for partitioned/parameterized readers.

### Q12. How do you pass data between steps?

**Answer:** Via `JobExecutionContext`:
```java
stepCtx.getStepExecution().getJobExecution().getExecutionContext().put("key", value);
```
Or promote step → job context with `ExecutionContextPromotionListener`.

### Q13. What are the different Job statuses?

**Answer:** `STARTING`, `STARTED`, `STOPPING`, `STOPPED`, `FAILED`, `COMPLETED`, `ABANDONED`. `ExitStatus` is a customizable string; `BatchStatus` is the enum.

### Q14. How do you prevent a job from restarting?

**Answer:** `.preventRestart()` on `JobBuilder`. Or set `restartable(false)`. But this is rarely recommended — restartability is a core benefit.

### Q15. How do you test Spring Batch jobs?

**Answer:** Use `JobLauncherTestUtils`:
```java
@Autowired JobLauncherTestUtils utils;
@Autowired JobRepository repository;

@Test void testJob() throws Exception {
    JobExecution exec = utils.launchJob(new JobParametersBuilder()
        .addString("inputFile", "test.csv").toJobParameters());
    assertThat(exec.getStatus()).isEqualTo(BatchStatus.COMPLETED);
}

@Test void testStep() {
    JobExecution exec = utils.launchStep("step1");
}
```

### Q16. What is the JobRepository and why is it needed?

**Answer:** It's the metadata store for all job/step executions. Enables restart, monitoring, and auditing. In production, use a **database** (not in-memory). Spring Boot auto-configures it with your DataSource.

### Q17. How do you handle a failing ItemWriter?

**Answer:** 1) Configure retry for transient exceptions. 2) Skip for permanent ones. 3) Ensure the writer is idempotent OR uses proper transaction boundaries. 4) Log failed items via `SkipListener` for later review.

### Q18. What is the difference between `@EnableBatchProcessing` in Boot 2 vs 3?

**Answer:** In Spring Boot 2, `@EnableBatchProcessing` was required. In Spring Boot 3 / Spring Batch 5, auto-configuration sets it up automatically. Adding it disables auto-config, so omit unless you need custom infrastructure beans.

### Q19. How do you process millions of records efficiently?

**Answer:** 1) Appropriate chunk size (100–1000). 2) Partitioning across threads/instances. 3) Paging readers (`JdbcPagingItemReader`/`JpaPagingItemReader`). 4) Batch writes (`JdbcBatchItemWriter`). 5) Disable unnecessary logging. 6) Index source DB appropriately.

### Q20. What are common Spring Batch anti-patterns?

**Answer:** 
- Using `JpaItemWriter` with default `merge` (slow) — use `persist` mode
- Non-thread-safe readers in multi-threaded steps
- In-memory JobRepository in production
- Chunk size too large (memory) or too small (overhead)
- Not using `ItemStream` (breaks restart)
- Not cleaning old executions (DB bloat)

---

## 15. Summary Cheat Sheet

### Maven Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
```

### Minimal Job Config

```java
@Configuration
public class BatchConfig {
    
    @Bean
    public Job job(JobRepository repo, Step step) {
        return new JobBuilder("myJob", repo)
            .start(step)
            .build();
    }
    
    @Bean
    public Step step(JobRepository repo, PlatformTransactionManager tx,
                     ItemReader<User> r, ItemProcessor<User, User> p,
                     ItemWriter<User> w) {
        return new StepBuilder("step", repo)
            .<User, User>chunk(100, tx)
            .reader(r).processor(p).writer(w)
            .faultTolerant()
            .skip(Exception.class).skipLimit(10)
            .retry(TransientDataAccessException.class).retryLimit(3)
            .build();
    }
}
```

### Key Annotations

| Annotation | Purpose |
|-----------|---------|
| `@EnableBatchProcessing` | Enable batch (Boot 2; auto in Boot 3) |
| `@StepScope` | Per-step bean scope |
| `@JobScope` | Per-job bean scope |
| `@BeforeJob` / `@AfterJob` | Job listener methods |
| `@BeforeStep` / `@AfterStep` | Step listener methods |
| `@BeforeChunk` / `@AfterChunk` | Chunk listener methods |
| `@OnReadError` / `@OnProcessError` / `@OnWriteError` | Error listener |
| `@OnSkipInRead` / `@OnSkipInProcess` / `@OnSkipInWrite` | Skip listeners |

### Job Launch Snippet

```java
JobParameters params = new JobParametersBuilder()
    .addString("inputFile", "input.csv")
    .addLong("time", System.currentTimeMillis())
    .toJobParameters();
jobLauncher.run(myJob, params);
```

### Batch Status Values

```
COMPLETED | FAILED | STOPPED | STARTED | STARTING | STOPPING | ABANDONED
```

### Common Readers/Writers

**Readers:**
`FlatFileItemReader`, `JsonItemReader`, `StaxEventItemReader`, `JdbcCursorItemReader`, `JdbcPagingItemReader`, `JpaPagingItemReader`, `MongoItemReader`, `KafkaItemReader`

**Processors:**
`ItemProcessor<I,O>` (custom)

**Writers:**
`FlatFileItemWriter`, `JsonFileItemWriter`, `StaxEventItemWriter`, `JdbcBatchItemWriter`, `JpaItemWriter`, `MongoItemWriter`, `CompositeItemWriter`

### Fault Tolerance

```java
.faultTolerant()
.skip(X.class).skipLimit(N)
.retry(Y.class).retryLimit(N)
.noSkip(Z.class).noRetry(W.class)
```

### Partitioning

```java
.partitioner("slaveStep", partitioner())
.step(slaveStep)
.gridSize(4)
.taskExecutor(taskExecutor())
```

### Key Tables

```
BATCH_JOB_INSTANCE
BATCH_JOB_EXECUTION
BATCH_JOB_EXECUTION_PARAMS
BATCH_JOB_EXECUTION_CONTEXT
BATCH_STEP_EXECUTION
BATCH_STEP_EXECUTION_CONTEXT
```

### Configuration Properties

```properties
spring.batch.jdbc.initialize-schema=always
spring.batch.job.enabled=true
spring.batch.job.name=myJob
spring.batch.job.names=job1,job2
```

### Testing

```java
@SpringBootTest
class JobTest {
    @Autowired JobLauncherTestUtils utils;
    
    @Test void testJob() throws Exception {
        JobExecution exec = utils.launchJob();
        assertThat(exec.getStatus()).isEqualTo(BatchStatus.COMPLETED);
    }
}
```

---

## Cross-References

- **Previous file:** `20_Spring_WebFlux_Reactive.md` — Reactive Programming
- **Next file:** `22_Spring_Messaging.md` — JMS, AMQP, Kafka
- **Related:** `05_Spring_Transaction_Management.md`, `10_Spring_Data_JPA.md`

---

## Practice Exercises

1. Create a CSV → DB import job with `FlatFileItemReader` + `JdbcBatchItemWriter`.
2. Add skip and retry policies to handle malformed rows.
3. Implement a `JobExecutionListener` that logs duration and record counts.
4. Build a partitioned job that reads from 4 partitions of a table in parallel.
5. Schedule the job with `@Scheduled` and Quartz.
6. Restart a failed job programmatically with `JobOperator`.
7. Expose batch metrics via Micrometer and Prometheus.
8. Write integration tests with `JobLauncherTestUtils`.

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [20_Spring_WebFlux_Reactive.md](./20_Spring_WebFlux_Reactive.md)
- **Next →:** [22_Spring_Messaging.md](./22_Spring_Messaging.md)
- **Related:** [04_Spring_JDBC.md](./04_Spring_JDBC.md), [22_Spring_Messaging.md](./22_Spring_Messaging.md), [26_Spring_Best_Practices.md](./26_Spring_Best_Practices.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
