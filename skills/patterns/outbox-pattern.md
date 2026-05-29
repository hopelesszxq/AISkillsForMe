---
name: outbox-pattern
description: 事务性发件箱模式（Transactional Outbox Pattern）：可靠事件发布、Exactly-Once 投递、Spring Boot 实现
tags: [patterns, outbox, event-driven, transactional, spring-boot, kafka]
---

## 概述

事务性发件箱模式解决微服务中"写数据库 + 发消息"的原子性问题。当 Kafka/RabbitMQ 不可用时，确保不会丢失事件；当应用崩溃时，防止出现"事务已提交但事件未发送"的数据不一致。

## 问题场景

### ❌ 反模式：数据库操作后直接发消息

```java
@Service
public class PaymentService {
    @Autowired
    private KafkaTemplate<String, PaymentEvent> kafkaTemplate;

    @Transactional
    public void processPayment(Payment payment) {
        // Step 1: 保存支付记录
        paymentRepository.save(payment);
        // Step 2: 发布事件到 Kafka
        kafkaTemplate.send("payment.completed", new PaymentEvent(payment));
        // Step 3: 返回成功
    }
}
```

**三个故障模式**：
1. Kafka 不可用时，整个支付失败（错误的状态依赖）
2. Kafka 异常被吞掉，事件丢失（静默数据丢失）
3. 应用在 Step 1 和 Step 2 之间崩溃，事务提交但事件从未发送（无法恢复）

## 解决方案：Outbox Pattern

### 核心思想

- 删除 Kafka 从支付流程的关键路径
- 支付服务和发件箱事件在 **同一个数据库事务** 中写入
- 后台独立进程负责将发件箱事件发布到消息队列
- PostgreSQL ACID 保证写操作原子性

### 数据库表结构

```sql
CREATE TABLE outbox_events (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id  VARCHAR(100) NOT NULL,   -- 业务聚合 ID（如订单ID）
    aggregate_type VARCHAR(100) NOT NULL,  -- 聚合类型（如 'ORDER'）
    event_type    VARCHAR(100) NOT NULL,   -- 事件类型（如 'ORDER_CREATED'）
    topic         VARCHAR(100) NOT NULL,   -- 目标 Topic
    payload       JSONB NOT NULL,          -- 消息体
    sent          BOOLEAN NOT NULL DEFAULT FALSE,
    created_at    TIMESTAMP NOT NULL DEFAULT NOW(),
    sent_at       TIMESTAMP
);

-- ⚠️ 关键索引：高效查询未发送事件
CREATE INDEX idx_outbox_unsent ON outbox_events (sent, created_at)
    WHERE sent = FALSE;
```

> 使用 `WHERE sent = FALSE` 的**部分索引**，避免全表扫描。未发送事件只是全部数据的一小部分。

### Spring Boot 实现：支付服务

```java
@Service
public class PaymentService {

    @Autowired
    private PaymentRepository paymentRepository;
    @Autowired
    private OutboxRepository outboxRepository;

    @Transactional
    public PaymentResponse processPayment(PaymentRequest request) {
        // Step 1: 构建支付记录
        Payment payment = buildPayment(request);

        // Step 2: 写支付记录
        paymentRepository.save(payment);

        // Step 3: 同事务写入发件箱
        OutboxEvent event = OutboxEvent.builder()
                .aggregateId(payment.getId())
                .aggregateType("PAYMENT")
                .eventType("PAYMENT_COMPLETED")
                .topic("payment.completed")
                .payload(buildPayload(payment))
                .sent(false)
                .build();
        outboxRepository.save(event);

        // Step 4: 事务提交 — 支付 + 事件一起持久化
        return PaymentResponse.success(payment.getId());
    }
}
```

> 支付服务完全不感知 Kafka。它只操作 PostgreSQL。这是解耦的关键。

### 后台发件箱处理进程

```java
@Component
public class OutboxProcessor {

    @Autowired
    private OutboxRepository outboxRepository;
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Scheduled(fixedDelay = 5000)  // 每 5 秒执行
    public void processOutbox() {
        List<OutboxEvent> events = outboxRepository
                .findBySentFalseOrderByCreatedAtAsc();

        for (OutboxEvent event : events) {
            try {
                // 发送到 Kafka 并等待确认
                kafkaTemplate.send(
                    event.getTopic(),
                    event.getAggregateId(),
                    event.getPayload()
                ).get(10, TimeUnit.SECONDS);

                // Kafka 确认后标记已发送
                event.setSent(true);
                event.setSentAt(LocalDateTime.now());
                outboxRepository.save(event);

            } catch (Exception e) {
                // 发送失败，下次循环重试
                log.error("Failed to publish event {}: {}", event.getId(), e.getMessage());
            }
        }
    }
}
```

### 故障场景推演

| 场景 | 结果 |
|------|------|
| Kafka 宕机时支付 | 支付成功，事件留在 outbox 表，恢复后自动发送 |
| 支付服务崩溃 | 事务回滚，支付和事件都未写入，幂等性安全 |
| 处理器崩溃 | 事件保持 sent=false，重启后重新处理 |
| Kafka 间歇故障 | 事件重试直到成功，消费者需幂等 |

## 排序语义与分区策略

生产环境下需要注意排序问题：

### 正确的排序：按业务键

```sql
-- 同一个 aggregate_id 内按创建时间排序
SELECT * FROM outbox_events
WHERE sent = FALSE
ORDER BY aggregate_id, created_at ASC;
```

### 分区处理（多实例）

```java
// 按 aggregate_id 哈希分片，每个实例处理自己的分区
@Component
public class PartitionedOutboxProcessor {

    private final String instanceId;

    public PartitionedOutboxProcessor(@Value("${outbox.partition:0}") int partition,
                                       @Value("${outbox.partitions:4}") int partitions) {
        this.instanceId = partition + "-of-" + partitions;
    }

    @Scheduled(fixedDelay = 5000)
    public void processPartition() {
        // 只处理属于本分区的 aggregrate_id
        List<OutboxEvent> events = outboxRepository
                .findUnsentByPartition(instanceId);
        // ...处理逻辑
    }
}
```

## 幂等消费者

发件箱模式保证 **至少一次投递**（At-Least-Once），消费者必须实现幂等：

```java
@Component
public class PaymentConsumer {

    @Autowired
    private PaymentRepository paymentRepository;

    @KafkaListener(topics = "payment.completed")
    public void onPaymentCompleted(PaymentEvent event) {
        // 用事件 ID 去重
        if (paymentRepository.existsByEventId(event.getId())) {
            log.info("Duplicate event skipped: {}", event.getId());
            return;
        }
        // 处理业务逻辑
        processPaymentEvent(event);
    }
}
```

## 注意事项

### 1. 发件箱表清理
- 已发送事件会不断累积，需定期清理
- 策略：删除 `sent_at < NOW() - INTERVAL '7 days'` 的记录
- 或使用分区表按时间滚动

### 2. 监控告警
- 监控 `count(*) WHERE sent = FALSE` 的积压量
- 积压持续增长说明 Kafka 或 Processor 故障
- 设置阈值告警（如超过 1000 条未发送）

### 3. 处理器 HA
- 多实例部署时，通过数据库**乐观锁**或**分区策略**避免重复处理
- 使用 `SELECT ... FOR UPDATE SKIP LOCKED`（PostgreSQL 9.5+）

```sql
-- PostgreSQL 跳过已锁定行，适合多消费者
BEGIN;
SELECT * FROM outbox_events
WHERE sent = FALSE
ORDER BY created_at ASC
LIMIT 10
FOR UPDATE SKIP LOCKED;
-- 处理...
COMMIT;
```

### 4. 替代方案
| 方案 | 优点 | 缺点 |
|------|------|------|
| **Debezium + CDC** | 无代码侵入，监听 binlog | 引入 Kafka Connect 复杂度 |
| **Spring Modulith** | 内置 Outbox 支持 | 限 Spring 生态 |
| **Namastack Outbox** | Java/Kotlin 专用库 | 外部依赖 |
