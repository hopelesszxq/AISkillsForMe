---
name: pg-as-dead-letter-queue
description: 使用 PostgreSQL 作为事件驱动系统的死信队列 — 可查询、可重试、可审计的 DLQ 方案
tags: [postgresql, event-driven, dlq, dead-letter-queue, kafka, pattern, skip-locked]
---

## 要点

在 Kafka/RabbitMQ 等消息系统中，用 PostgreSQL 取代 topic DLQ 存储失败事件，利用 PG 的 SQL 查询能力实现可观测、可重试的死信处理。

### 为什么用 PG 做 DLQ？

| 维度 | Kafka/RabbitMQ Topic DLQ | PostgreSQL DLQ |
|------|--------------------------|----------------|
| 消息查询 | 需额外消费者 + 工具 | 直接 `SELECT * FROM dlq_events WHERE status = 'PENDING'` |
| 按原因检索 | 困难，需解析消息体 | `WHERE error_reason LIKE '%timeout%'` |
| 按时间回溯 | 有限 | `WHERE created_at > NOW() - INTERVAL '7 days'` |
| 批量重试 | 需自定义客户端 | `UPDATE ... WHERE status = 'PENDING' LIMIT 50` |
| 审计追溯 | 消息过期后被删除 | 数据持久保留，可长期分析 |

## 代码示例

### 1. DLQ 表设计

```sql
CREATE TABLE dlq_events (
    id               BIGSERIAL PRIMARY KEY,
    event_type       VARCHAR(255) NOT NULL,
    payload          JSONB NOT NULL,
    error_reason     TEXT NOT NULL,
    error_stacktrace TEXT,
    status           VARCHAR(20) NOT NULL DEFAULT 'PENDING',
        -- PENDING / SUCCEEDED / FAILED_PERMANENT
    retry_count      INT NOT NULL DEFAULT 0,
    max_retries      INT NOT NULL DEFAULT 240,
    retry_after      TIMESTAMPTZ NOT NULL,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 索引策略
CREATE INDEX idx_dlq_status             ON dlq_events (status);
CREATE INDEX idx_dlq_status_retry_after ON dlq_events (status, retry_after);
CREATE INDEX idx_dlq_event_type         ON dlq_events (event_type);
CREATE INDEX idx_dlq_created_at         ON dlq_events (created_at);
```

### 2. 重试调度器（FOR UPDATE SKIP LOCKED）

```java
// 使用 Spring Data JPA + ShedLock 实现分布式调度

@Scheduled(fixedRateString = "${dlq.retry.fixed-rate:21600000}")
@SchedulerLock(name = "dlqRetryJob", lockAtLeastFor = "1h")
@Transactional
public void retryDlqEvents() {
    List<DlqEvent> events = dlqEventRepository.findEligibleForRetry(
        "PENDING", LocalDateTime.now(), PageRequest.of(0, 50));
    
    for (DlqEvent event : events) {
        try {
            processEvent(event);
            event.setStatus("SUCCEEDED");
        } catch (Exception e) {
            event.setRetryCount(event.getRetryCount() + 1);
            if (event.getRetryCount() >= event.getMaxRetries()) {
                event.setStatus("FAILED_PERMANENT");
            } else {
                // 指数退避
                event.setRetryAfter(
                    LocalDateTime.now().plusHours(6));
            }
        }
    }
    dlqEventRepository.saveAll(events);
}
```

### 3. SKIP LOCKED 查询（防止多实例重复处理）

```java
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "-2"))
@Query(value = """
    SELECT * FROM dlq_events
    WHERE status = :status
      AND retry_after <= :now
      AND retry_count < max_retries
    ORDER BY retry_after ASC
    LIMIT :limit
    FOR UPDATE SKIP LOCKED
    """, nativeQuery = true)
List<DlqEvent> findEligibleForRetry(
    @Param("status") String status,
    @Param("now") LocalDateTime now,
    Pageable pageable);
```

### 4. 消费者层指数退避（在写入 DLQ 前重试）

```java
// 消费失败后先重试 3 次，再写入 DLQ
ExponentialBackOffWithMaxRetries backOff =
    new ExponentialBackOffWithMaxRetries(3);
backOff.setInitialInterval(2000L);   // 2 秒
backOff.setMultiplier(2.0);          // 指数增长
backOff.setMaxInterval(30000L);      // 最大 30 秒
backOff.setMaxAttempts(3);           // 最多重试 3 次

// 重试 3 次仍失败 → 写入 DLQ
if (!retryTemplate.execute(ctx -> processMessage(message))) {
    dlqService.sendToDlq(message, lastError);
}
```

### 5. 配置项

```yaml
dlq:
  retry:
    enabled: true
    max-retries: 240        # 240次 ≈ 60天（6小时间隔）
    batch-size: 50
    fixed-rate: 21600000    # 6小时（毫秒）
```

## 工作原理

```
Kafka Consumer → 处理失败 → 指数退避重试(3次)
                                    ↓ 仍失败
                              INSERT INTO dlq_events
                                    ↓
                        调度器每隔6小时扫描 PENDING 记录
                        使用 FOR UPDATE SKIP LOCKED 避免冲突
                                    ↓
                             重试处理成功 → status=SUCCEEDED
                             重试超限 → status=FAILED_PERMANENT
```

## 注意事项

1. **不是替代消息系统**：PG DLQ 做的是"兜底仓储"，高吞吐事件流仍由 Kafka/RabbitMQ 承载，PG 只存失败事件。
2. **避免 DLQ 洪泛**：一定要在消费者层实现指数退避重试（3 次以上），只将确实无法恢复的失败事件写入 DLQ。
3. **SKIP LOCKED 最佳实践**：小批次（20-50 条）+ 适当间隔，避免长时间锁住大量行。
4. **FAILED_PERMANENT 监控**：设置告警，当 FAILED_PERMANENT 数量突增时及时人工介入。
5. **清理策略**：已 SUCCEEDED 的事件定期删除（TTL 30 天），或归档到冷存储。
6. **PG 版本要求**：`FOR UPDATE SKIP LOCKED` 需要 PG 9.5+（现代版本都支持）。
