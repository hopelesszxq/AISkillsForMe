---
name: spring-modulith-outbox
description: Spring Modulith 2.1.0 事务性发件箱：Namastack Outbox 集成与 JobRunr 事件外部化
tags: [patterns, spring-modulith, outbox, event-driven, spring-boot, jobrunr]
---

## 概述

Spring Modulith 2.1.0 GA（2026-06-11 发布）在模块化单体框架中引入了**内建的事务性发件箱（Outbox）支持**，通过 **Namastack Outbox** 库实现可靠的事件发布，同时支持通过 **JobRunr** 将事件外部化到消息中间件。这标志着 Spring Modulith 从"模块验证+事件总线"向"可靠事件驱动架构平台"的重要进化。

> 已有技能 `spring-modulith.md` 覆盖了 2.1.0-RC1 及之前的基础概念，本文聚焦 2.1.0 GA 新增的 Outbox 和事件外部化能力。

## 版本更新总览

| 特性 | 说明 |
|------|------|
| Namastack Outbox 集成 | 零额外配置的事务性发件箱 |
| JobRunr 事件外部化 | 将事件异步投递到 RabbitMQ/Kafka |
| EPR 模式默认开启 | JDBC 事件发布注册表 schema 自动创建 |
| Spring Boot 4.1 | 基线从 4.0 升级到 4.1 |
| 测试增强 | `PublishedEvents` 跨线程可见 |

## 一、Namastack Outbox 集成

### 1.1 添加依赖

```xml
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-core</artifactId>
</dependency>

<!-- Namastack Outbox 自动配置 -->
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-outbox-namastack</artifactId>
</dependency>
```

### 1.2 基础用法

一旦添加 `spring-modulith-outbox-namastack` 依赖，模块间事件自动走事务性发件箱路由，**无需修改任何业务代码**：

```java
// 模块 A：发布事件（自动进入发件箱表）
@Service
public class OrderService {
    private final ApplicationEventPublisher events;

    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        // 此事件自动写入 outbox 表，与业务事务一起提交
        events.publishEvent(new OrderPlaced(order.getId()));
    }
}

// 模块 B：消费事件（从发件箱表读取）
@Service
public class NotificationService {
    @TransactionalEventListener
    void on(OrderPlaced event) {
        notificationClient.send(event);
    }
}
```

### 1.3 发件箱工作原理

```
@Transactional 方法执行
    │
    ├── 1. 业务数据库操作（INSERT/UPDATE）
    ├── 2. ApplicationEventPublisher.publishEvent()
    │        └── 事件序列化 → 写入 outbox 表（同一事务）
    │
    ├── 3. 事务提交（业务数据 + 事件 一次性写入）
    │
    └── 4. Outbox 后台线程轮询
            ├── 读取未完成事件
            ├── 调用模块内 @TransactionalEventListener
            └── 标记为 COMPLETED（或重试）
```

### 1.4 事务边界

Spring Modulith 会在 `@Transactional` 方法返回后，在**同一事务边界内**将事件持久化到 outbox 表，确保"业务操作 → 事件入库"的原子性：

```yaml
spring:
  modulith:
    outbox:
      namastack:
        # 轮询间隔（默认 1000ms）
        polling-interval: 500
        # 每次批处理最大事件数
        batch-size: 50
        # 重试延迟（毫秒）
        retry-interval: 5000
        # 最大重试次数
        max-retries: 3
```

### 1.5 数据库表结构（自动创建）

```sql
-- 发件箱表（Spring Modulith 自动管理）
CREATE TABLE IF NOT EXISTS event_publication (
    id                   UUID PRIMARY KEY,
    serialized_event     TEXT NOT NULL,
    event_type           VARCHAR(512) NOT NULL,
    listener_id          VARCHAR(512) NOT NULL,
    completion_date      TIMESTAMP,
    publication_date     TIMESTAMP NOT NULL,
    status               VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    -- 2.1.0 新增：SHA-256 事件哈希（Neo4j 场景）
    event_hash           VARCHAR(128)
);
```

> EPR 模式在 2.1.0 中**默认开启**（`spring.modulith.events.jdbc.schema-initialization.enabled=true`），无需手动配置即可自动建表。

## 二、JobRunr 事件外部化

### 2.1 为什么需要事件外部化

模块间通过 Spring 事件总线是同步的（或事务级别的异步），当需要将事件投递到**外部消息系统**（RabbitMQ、Kafka）或**跨服务边界**时，需要事件外部化。

### 2.2 JobRunr 集成

```xml
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-outbox-jobrunr</artifactId>
</dependency>

<!-- JobRunr 依赖（自动引入） -->
<dependency>
    <groupId>org.jobrunr</groupId>
    <artifactId>jobrunr-spring-boot-4-starter</artifactId>
</dependency>
```

### 2.3 配置 JobRunr 外部化

```yaml
spring:
  modulith:
    outbox:
      externalization:
        # 启用事件外部化
        enabled: true
        # 目标 Topic/Exchange 解析策略
        routing-resolver: fully-qualified-name
        # 支持的消息中间件
        broker: rabbitmq  # 或 kafka

jobrunr:
  background-job-server:
    enabled: true
    worker-count: 4
    poll-interval-in-seconds: 5
  database:
    type: sql
    table-prefix: jobrunr_
```

### 2.4 外部化事件投递流程

```java
// 无需额外代码，所有模块间事件自动被 JobRunr 拾取并投递到外部

// 自定义事件路由（可选）
@Bean
public ExternalizedEventRoutingCallback routingCallback() {
    return event -> {
        if (event instanceof OrderPlaced) {
            // 投递到 RabbitMQ exchange
            return RoutingTarget.of("exchange:order.events", "order.placed");
        }
        if (event instanceof PaymentCompleted) {
            return RoutingTarget.of("topic:payment.events", "payment.completed");
        }
        return null; // 走默认路由：类全限定名
    };
}
```

### 2.5 RabbitMQ 发布确认

Spring Modulith 2.1.0 要求 RabbitMQ `publisher-confirm-type=CORRELATED`，确保外部化事件至少投递一次：

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated
    publisher-returns: true
  modulith:
    outbox:
      externalization:
        # 阻塞等待外部化确认（防止事务提交后投递失败）
        block-for-externalization: true
```

## 三、EPR 模式变更（2.1.0 破坏性变化）

### 3.1 默认开启 Schema 初始化

```yaml
# 2.1.0 之前（需手动开启）
spring.modulith.events.jdbc.schema-initialization.enabled=true

# 2.1.0 开始（默认 true，如需禁用）
spring.modulith.events.jdbc.schema-initialization.enabled=false
```

**影响**：如果你的应用使用内存数据库（H2）做测试，且不希望自动建表，需显式禁用。

### 3.2 Flyway 迁移优化

```yaml
# 2.1.0 改进：只有包含 Flyway 迁移文件的模块才会执行迁移
# 避免空模块尝试执行 Flyway 迁移
```

### 3.3 EPR Schema 初始化顺序

Spring Modulith 2.1.0 确保 JDBC EPR schema 初始化在 Boot 的文件式 SQL 初始化**之后**执行，避免冲突：

```yaml
spring:
  sql:
    init:
      mode: always
  modulith:
    events:
      jdbc:
        schema-initialization:
          enabled: true  # 自动在 Boot SQL 之后执行
```

## 四、测试增强

### 4.1 跨线程事件可见

2.1.0 改进了 `PublishedEvents` 和 `Scenario` 测试 API，默认可以观察到**所有线程**发布的事件：

```java
@ApplicationModuleTest
class OrderModuleTest {

    @Autowired
    Scenario scenario;

    @Test
    void shouldPublishEventOnOrderPlacement() {
        // 异步/跨线程发布的事件也能被观察到
        scenario.waitForEvent(OrderPlaced.class, 1000)
            .matching(event -> event.orderId() == 1L)
            .toArrive();
    }
}
```

### 4.2 Slice Test 兼容

```java
// 2.1.0 支持与 Boot 4.1 的 @WebMvcTest / @DataJpaTest 等 Slice Test 组合使用
@WebMvcTest(OrderController.class)
@ApplicationModuleTest(module = "order")
class OrderControllerSliceTest {
    // 只加载 order 模块的 Web 层 Bean
}
```

## 五、与其他 Outbox 实现的对比

| 特性 | Spring Modulith Outbox (2.1.0) | 手动 Outbox 实现 | Debezium Outbox |
|------|------|------|------|
| 实现复杂度 | 零代码（加依赖即可） | 中高（需手写轮询/重试） | 高（需部署 CDC 组件） |
| 事务保证 | ✅ Spring 事务自动管理 | ✅ 需手动保证 | ✅ 通过 WAL/CDC |
| 外部消息系统 | ✅ JobRunr → RabbitMQ/Kafka | ✅ 自定义 | ✅ Kafka 直接 |
| 重试机制 | ✅ 内建（NameStack/JobRunr） | ❌ 需自建 | ✅ Kafka 内置 |
| 数据库支持 | JPA/JDBC/Neo4j | 任意 | Debezium 支持的数据库 |
| 学习成本 | 低（Spring 原生体验） | 中 | 高 |
| 适合场景 | 模块化单体 → 微服务过渡期 | 任意架构 | 高吞吐、异构系统 |

## 注意事项

1. **JDK 21+ 必需**：Spring Modulith 2.1.0 需要 Spring Boot 4.1 + JDK 21+
2. **Namastack 并非唯一选择**：如果不希望引入 Namastack，可继续使用 Spring Modulith 内建的 EPR（事件发布注册表）+ JobRunr 外部化
3. **事务边界要清晰**：事件只有在 `@Transactional` 方法内才能保证原子性写入 outbox 表
4. **Outbox 表清理**：`event_publication` 表中的已处理事件不会自动删除，建议定时清理：
   ```sql
   DELETE FROM event_publication
   WHERE completion_date < NOW() - INTERVAL '7 days';
   ```
5. **生产环境建议**：JobRunr 外部化时，确保 RabbitMQ/Kafka 高可用，避免事件投递失败阻塞业务
6. **升级注意**：从 2.1.0-RC1 升级到 GA 时，`schema-initialization.enabled` 默认为 `true`，需检查是否和现有 Flyway/Liquibase 脚本冲突
7. **SHA-256 哈希**：Neo4j EPR 使用 SHA-256 替代了旧版哈希算法，升级后需重建索引

## 参考

- [Spring Modulith 2.1.0 Release Notes](https://github.com/spring-projects/spring-modulith/releases/tag/2.1.0)
- [Namastack Outbox](https://namastack.dev/)
- [JobRunr Documentation](https://www.jobrunr.io/)
- [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
