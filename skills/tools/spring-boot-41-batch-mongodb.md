---
name: spring-boot-41-batch-mongodb
description: Spring Boot 4.1 MongoDB 支持 Spring Batch JobRepository——零 SQL 依赖的批处理元数据存储
tags: [tools, spring-boot, spring-boot-41, spring-batch, mongodb, batch-processing]
---

## 概述

Spring Boot 4.1（2026-06-10 发布）引入了 **`spring-boot-starter-batch-data-mongodb`** 自动配置，使 Spring Batch 的 `JobRepository` 可以使用 **MongoDB** 作为后端存储，不再强制依赖关系型数据库。这对于已经使用 MongoDB 的团队来说，消除了为 Batch 元数据单独维护 SQL 数据库的痛点。

> 本特性由 Spring Batch 的 `JobRepository` 接口解耦（从 JDBC 抽象为独立接口）配合 Boot 4.1 的零配置自动装配实现。

## 添加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch-data-mongodb</artifactId>
</dependency>
```

只需这一个依赖，Spring Boot 自动配置会完成以下工作：
- 自动检测 MongoDB 连接（复用 `spring-boot-starter-data-mongodb` 配置）
- 创建所需的集合（JobExecution、StepExecution、JobInstance 等）
- 配置 MongoDB 事务管理器（需要 MongoDB 副本集）
- 替换默认的 JDBC `JobRepository`

## 配置

### 基础配置

```yaml
spring:
  data:
    mongodb:
      host: localhost
      port: 27017
      database: batch-db
  batch:
    jdbc:
      initialize-schema: never   # 关闭 JDBC Schema 初始化
    data:
      mongodb:
        schema:
          initialize: true       # 自动创建 MongoDB 集合（默认 false）
```

### 排除 JDBC 自动配置

引入 MongoDB Starter 后，JDBC 的 Batch 自动配置仍会尝试启动。需要显式排除：

```java
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.batch.BatchJdbcAutoConfiguration;

@SpringBootApplication(exclude = {BatchJdbcAutoConfiguration.class})
public class BatchApplication {
    public static void main(String[] args) {
        SpringApplication.run(BatchApplication.class, args);
    }
}
```

如果不排除，Spring Boot 会同时配置 JDBC 和 MongoDB 两个 `JobRepository`，导致冲突。

### Docker Compose 示例

```yaml
services:
  mongodb:
    image: mongo:7
    command: mongod --replSet rs0  # MongoDB Batch 需要副本集支持事务
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_DATABASE: batch-db

  # 初始化副本集（首次启动后执行）
  mongo-init:
    image: mongo:7
    depends_on:
      - mongodb
    entrypoint: >
      mongosh --host mongodb:27017 --eval
      "rs.initiate({_id:'rs0', members:[{_id:0, host:'mongodb:27017'}]})"
```

## 完整示例

### Application 类

```java
@SpringBootApplication(exclude = {BatchJdbcAutoConfiguration.class})
public class BatchApplication {
    public static void main(String[] args) {
        SpringApplication.run(BatchApplication.class, args);
    }
}
```

### Job 定义

```java
@Configuration
public class CustomerImportJob {

    @Bean
    public Job importJob(JobRepository jobRepository, Step importStep) {
        return new JobBuilder("importCustomerJob", jobRepository)
                .start(importStep)
                .build();
    }

    @Bean
    public Step importStep(
            JobRepository jobRepository,
            MongoTransactionManager transactionManager) {  // 自动注入 MongoDB 事务管理器

        return new StepBuilder("importStep", jobRepository)
                .<String, Customer>chunk(100, transactionManager)
                .reader(flatFileReader())
                .processor(processor())
                .writer(jdbcWriter())
                .build();
    }

    @Bean
    public FlatFileItemReader<String> flatFileReader() {
        return new FlatFileItemReaderBuilder<String>()
                .name("customerReader")
                .resource(new ClassPathResource("customers.csv"))
                .lineMapper((line, num) -> line)
                .build();
    }

    @Bean
    public ItemProcessor<String, Customer> processor() {
        return line -> {
            String[] parts = line.split(",");
            return new Customer(null, parts[0], parts[1]);
        };
    }

    @Bean
    public JdbcBatchItemWriter<Customer> jdbcWriter(DataSource dataSource) {
        return new JdbcBatchItemWriterBuilder<Customer>()
                .dataSource(dataSource)
                .sql("INSERT INTO customers (name, email) VALUES (:name, :email)")
                .beanMapped()
                .build();
    }
}
```

## 工作的 MongoDB 集合

启用 `spring.batch.data.mongodb.schema.initialize=true` 后，MongoDB 中会自动创建以下集合：

| 集合名 | 用途 |
|--------|------|
| `jobInstance` | 作业实例元数据 |
| `jobExecution` | 作业执行元数据 |
| `stepExecution` | 步骤执行元数据 |
| `jobParameters` | 作业参数 |

## 注意事项

1. **MongoDB 必须为副本集**：MongoDB Batch 事务支持依赖副本集（单节点也可配置为副本集）
2. **必须排除 JDBC 配置**：`BatchJdbcAutoConfiguration` 不排除会导致双 JobRepository 冲突
3. **JDBC 和 MongoDB 可共存**：JobRepository 用 MongoDB，ItemWriter 仍可用 JDBC 写关系型数据库
4. **迁移现有 Batch**：从 JDBC 切换到 MongoDB 时，历史 JobExecution 数据不会自动迁移
5. **监控兼容**：Spring Batch Admin / Micrometer 监控指标在 MongoDB 模式下同样可用
6. **性能考虑**：相比 JDBC，MongoDB 在 Batch 元数据读写方面可能有一定性能差异，建议在大规模作业下进行基准测试
