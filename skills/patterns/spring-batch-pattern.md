---
name: spring-batch-pattern
description: Spring Batch 批量处理模式：大规模数据处理、分片、容错与最佳实践
tags: [patterns, spring-batch, batch-processing, chunk, performance, etl]
---

## 概述

Spring Batch 是 Java 生态中处理大容量批量操作的标准框架。本文总结在生产环境中使用 Spring Batch 的核心模式、性能优化策略和常见坑点。

## 核心概念

### Chunk 处理模型

```
Reader → Processor → Writer  （每个 Chunk 内事务包裹）
         ↑     ↓
      Item 处理完后统一提交
```

```java
@Bean
public Step importUserStep(
        JobRepository jobRepository,
        PlatformTransactionManager transactionManager) {

    return new StepBuilder("importUser", jobRepository)
            .<UserCsvRecord, User>chunk(1000, transactionManager)
            .reader(userItemReader())
            .processor(userItemProcessor())
            .writer(userItemWriter())
            .faultTolerant()
            .skip(ParseException.class).skipLimit(10)  // 跳过错行
            .retry(DeadlockLoserDataAccessException.class).retryLimit(3)
            .noRollback(ValidationException.class)     // 校验失败不回滚
            .listener(new ChunkListener() {
                @Override
                public void afterChunk(ChunkContext context) {
                    long count = context.getStepContext().getStepExecution().getWriteCount();
                    log.info("已处理 {} 条记录", count);
                }
            })
            .build();
}
```

## 常用 Reader 模式

### 数据库游标 Reader（大数据推荐）

```java
@Bean
public JdbcCursorItemReader<User> cursorReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<User>()
            .name("cursorReader")
            .dataSource(dataSource)
            .sql("SELECT id, name, email FROM users WHERE status = 'PENDING'")
            .fetchSize(5000)                            // JDBC 每次取 5000 行
            .rowMapper((rs, rowNum) -> User.builder()
                    .id(rs.getLong("id"))
                    .name(rs.getString("name"))
                    .email(rs.getString("email"))
                    .build())
            .build();
}
```

### 分页 Reader（避免长事务）

```java
@Bean
public JdbcPagingItemReader<User> pagingReader(DataSource dataSource) {
    return new JdbcPagingItemReaderBuilder<User>()
            .name("pagingReader")
            .dataSource(dataSource)
            .selectClause("SELECT id, name, email")
            .fromClause("FROM users")
            .whereClause("WHERE status = 'PENDING'")
            .sortKeys(Map.of("id", Order.ASCENDING))
            .pageSize(2000)
            .rowMapper(userRowMapper())
            .build();
}
```

### Reader 选择策略

| Reader 类型 | 优点 | 缺点 | 适用场景 |
|-------------|------|------|---------|
| `JdbcCursorItemReader` | 无游标偏移开销，适合全表扫描 | 单连接长事务 | 百万级以下 |
| `JdbcPagingItemReader` | 分页查询，事务短 | 排序开销，数据偏移问题 | 千万级以上 |
| `JpaCursorItemReader` | JPA 实体映射 | Hibernate Session 内存开销 | 与 JPA 集成 |
| `FlatFileItemReader` | CSV/固定宽度文件解析 | 不支持随机访问 | 文件导入 |
| `MultiResourceItemReader` | 多文件多线程处理 | 资源管理复杂 | 多文件批量导入 |
| `StaxEventItemReader` | 流式解析超大 XML | 不支持 XPATH 复杂查询 | XML 导入 |

## 批量写入优化

### JDBC 批量写入（推荐）

```java
@Bean
public JdbcBatchItemWriter<User> batchWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<User>()
            .dataSource(dataSource)
            .sql("INSERT INTO target_users (id, name, email, created_at) " +
                 "VALUES (:id, :name, :email, :createdAt)")
            .beanMapped()
            .assertUpdates(true)
            .build();
}
```

### 合并 Upsert 模式

```sql
-- PostgreSQL
INSERT INTO target_users (id, name, email, updated_at)
VALUES (:id, :name, :email, NOW())
ON CONFLICT (id) DO UPDATE
SET name = EXCLUDED.name,
    email = EXCLUDED.email,
    updated_at = NOW()
```

```java
// 使用 upsert SQL 配合 JdbcBatchItemWriter
@Bean
public JdbcBatchItemWriter<User> upsertWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<User>()
            .dataSource(dataSource)
            .sql("""
                INSERT INTO target_users (id, name, email, updated_at)
                VALUES (:id, :name, :email, NOW())
                ON CONFLICT (id) DO UPDATE SET
                    name = EXCLUDED.name,
                    email = EXCLUDED.email,
                    updated_at = NOW()
                """)
            .beanMapped()
            .build();
}
```

### 分区写入（多线程）

```java
@Bean
@StepScope
public JdbcBatchItemWriter<User> partitionedWriter(DataSource dataSource,
        @Value("#{stepExecutionContext['partition']}") Integer partition) {
    // 每个分区使用独立连接池
    return new JdbcBatchItemWriterBuilder<User>()
            .dataSource(dataSource)
            .sql("INSERT INTO target_users_${partition} (...)...")
            .build();
}
```

## 分区处理（Partitioning）

对于超大规模数据，使用分区并行处理：

```java
@Configuration
public class BatchPartitionConfig {

    @Bean
    public Step masterStep(
            JobRepository jobRepository,
            PartitionHandler partitionHandler) {

        return new StepBuilder("masterStep", jobRepository)
                .partitioner("workerStep", partitioner())
                .partitionHandler(partitionHandler)
                .gridSize(8)  // 8 个分区
                .taskExecutor(new SimpleAsyncTaskExecutor("batch-"))
                .build();
    }

    @Bean
    public Partitioner partitioner() {
        return gridSize -> {
            Map<String, ExecutionContext> partitions = new HashMap<>(gridSize);
            // 根据 ID 范围分片
            long totalCount = jdbcTemplate.queryForObject(
                    "SELECT COUNT(*) FROM source_table WHERE status='PENDING'",
                    Long.class);
            long pageSize = (totalCount + gridSize - 1) / gridSize;

            for (int i = 0; i < gridSize; i++) {
                ExecutionContext context = new ExecutionContext();
                context.putLong("minId", i * pageSize + 1);
                context.putLong("maxId", Math.min((i + 1) * pageSize, totalCount));
                partitions.put("partition-" + i, context);
            }
            return partitions;
        };
    }

    @Bean
    @StepScope
    public JdbcCursorItemReader<User> partitionReader(
            @Value("#{stepExecutionContext['minId']}") Long minId,
            @Value("#{stepExecutionContext['maxId']}") Long maxId) {

        return new JdbcCursorItemReaderBuilder<User>()
                .name("partitionReader-" + minId)
                .dataSource(dataSource)
                .sql("SELECT id, name, email FROM source_table " +
                     "WHERE status = 'PENDING' AND id BETWEEN ? AND ? " +
                     "ORDER BY id")
                .preparedStatementSetter(ps -> {
                    ps.setLong(1, minId);
                    ps.setLong(2, maxId);
                })
                .fetchSize(1000)
                .rowMapper(userRowMapper())
                .build();
    }
}
```

## 重启与容错

### 跳过策略

```java
// 自定义跳过策略：记录错误行号，跳过后继续
@Bean
public SkipPolicy customSkipPolicy() {
    return (Throwable t, int skipCount) -> {
        if (skipCount >= 100) {  // 最多跳过 100 条
            throw new SkipLimitExceededException("超过最大跳过数", skipCount, t);
        }
        if (t instanceof ParseException) {
            log.warn("跳过解析失败记录 #{}", skipCount + 1, t);
            return true;
        }
        if (t instanceof ValidationException) {
            log.warn("跳过校验失败记录 #{}", skipCount + 1, t);
            return true;
        }
        // 其他异常不跳过，导致 Job 失败
        return false;
    };
}
```

### Job 重启行为

```java
@Bean
public Job importJob(JobRepository jobRepository, Step importStep) {
    return new JobBuilder("importJob", jobRepository)
            .start(importStep)
            // 默认 RESTARTABLE — 失败后从断点恢复
            .preventRestart()              // 禁止重启（只需执行一次）
            .validator(parameters -> {     // 参数校验
                if (!parameters.containsKey("filePath")) {
                    throw new JobParametersInvalidException("缺少 filePath 参数");
                }
            })
            .incrementer(new RunIdIncrementer())  // 每次运行自增 ID
            .listener(new JobExecutionListener() {
                @Override
                public void beforeJob(JobExecution jobExecution) {
                    log.info("Job 启动：{}", jobExecution.getJobParameters());
                }
                @Override
                public void afterJob(JobExecution jobExecution) {
                    if (jobExecution.getStatus() == BatchStatus.FAILED) {
                        // 发送告警
                        alertService.sendBatchFailureAlert(jobExecution);
                    }
                }
            })
            .build();
}
```

## 性能调优

### Chunk Size 调优

```yaml
# 经验值（根据记录大小调整）
# 小记录 (< 1KB):    chunk-size: 5000~10000
# 中记录 (1~10KB):   chunk-size: 1000~5000
# 大记录 (> 10KB):   chunk-size: 200~1000
spring.batch.jdbc.initialize-schema: always
```

### 连接池配置

```yaml
# 分区处理时，确保连接池大小 >= 分区数
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000
      max-lifetime: 600000
```

### 关闭不必要的功能

```java
@Bean
public JobRepository jobRepository(
        DataSource dataSource,
        PlatformTransactionManager transactionManager) throws Exception {

    JobRepositoryFactoryBean factory = new JobRepositoryFactoryBean();
    factory.setDataSource(dataSource);
    factory.setTransactionManager(transactionManager);
    factory.setTablePrefix("BATCH_");

    // 关闭不必要的元数据写入（提升 30%+ 性能）
    factory.setIsolationLevelForCreate("ISOLATION_READ_COMMITTED");
    factory.setMaxVarCharLength(1000);
    factory.afterPropertiesSet();
    return factory.getObject();
}
```

## 监控

```java
@Bean
public StepExecutionListener performanceMonitor() {
    return new StepExecutionListener() {
        private final Map<String, Long> metrics = new ConcurrentHashMap<>();

        @Override
        public ExitStatus afterStep(StepExecution stepExecution) {
            long elapsed = stepExecution.getEndTime().getTime()
                         - stepExecution.getStartTime().getTime();
            long count = stepExecution.getWriteCount();

            log.info("Step {} 完成: {} 条记录, 耗时 {}ms, 吞吐量 {} 条/秒",
                    stepExecution.getStepName(),
                    count,
                    elapsed,
                    elapsed > 0 ? count * 1000 / elapsed : 0);
            return null;
        }
    };
}
```

### Micrometer 指标

```yaml
management:
  metrics:
    tags:
      application: ${spring.application.name}
    export:
      prometheus:
        enabled: true

# Spring Batch 自动暴露指标：
# spring.batch.job.completed
# spring.batch.job.failed
# spring.batch.step.completed
# spring.batch.step.failed
# spring.batch.chunk.read.count
# spring.batch.chunk.write.count
```

## 注意事项

1. **事务边界**：Chunk 内 Reader 在事务外（可无事务），Processor 和 Writer 在事务内。`Reader` 如果需要事务，用 `@Transactional(propagation=REQUIRES_NEW)`

2. **Reader 线程安全**：`JdbcCursorItemReader` 不是线程安全的。分区模式下每个分区使用独立的 Reader 实例

3. **重启 ID 稳定**：基于 ID 范围的分区确保数据在重启后仍然正确——ID 范围在分区步骤执行前固定

4. **跳过数与事务**：跳过计费是针对整个 JobExecution 的累计值，不是每步。如果 Step 重启，跳过计数重置

5. **Spring Batch 元数据表**：默认使用 `BATCH_` 前缀的表来管理 Job 执行状态。生产环境务必使用持久化数据源（非 H2内存）

6. **大事务问题**：Chunk Size 过大 → 单事务时间过长 → 锁竞争、MVCC 膨胀。根据数据行大小合理设置（参考上方调优表）

7. **多线程写入目标**：分区写入时目标表如果是同一张表，注意行锁冲突。可通过哈希分表、临时表合并等方式缓解
