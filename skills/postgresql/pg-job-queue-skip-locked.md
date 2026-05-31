---
name: pg-job-queue-skip-locked
description: 用 PostgreSQL SKIP LOCKED 实现轻量级任务队列，无需 Redis/RabbitMQ 即可在生产中可靠运行
tags: [postgresql, job-queue, skip-locked, background-job, performance]
---

## 要点

PostgreSQL 内置的 `SKIP LOCKED` 子句可以让你直接在数据库中实现**高性能任务队列**，无需引入 Redis 或 RabbitMQ 等额外组件。对于中小规模场景（每秒几百个任务），这是一个零运维成本的优雅方案。

核心思路：多个 Worker 同时 `SELECT FOR UPDATE SKIP LOCKED` 抢任务，每个 Worker 拿到不同的行，零冲突。

## 表结构设计

```sql
CREATE TABLE jobs (
    id          BIGSERIAL PRIMARY KEY,
    queue       TEXT NOT NULL DEFAULT 'default',   -- 队列名称
    payload     JSONB NOT NULL,                     -- 任务数据
    status      TEXT NOT NULL DEFAULT 'pending'     -- pending | running | done | failed
        CHECK (status IN ('pending', 'running', 'done', 'failed')),
    priority    INT NOT NULL DEFAULT 0,             -- 优先级（越大越优先）
    attempts    INT NOT NULL DEFAULT 0,             -- 重试次数
    max_retries INT NOT NULL DEFAULT 3,             -- 最大重试次数
    scheduled_at TIMESTAMPTZ NOT NULL DEFAULT now(),-- 可调度时间
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 核心索引：按状态 + 优先级 + 调度时间查找待处理任务
CREATE INDEX idx_jobs_pending
    ON jobs (priority DESC, scheduled_at)
    WHERE status = 'pending';
```

## Worker 抢任务 SQL（核心）

```sql
-- 每个 Worker 定期执行此查询获取任务
UPDATE jobs SET
    status = 'running',
    attempts = attempts + 1,
    updated_at = now()
WHERE id = (
    SELECT id
    FROM jobs
    WHERE status = 'pending'
      AND scheduled_at <= now()
    ORDER BY priority DESC, scheduled_at ASC
    FOR UPDATE SKIP LOCKED   -- 跳过已被其他 Worker 锁定的行
    LIMIT 1
)
RETURNING *;
```

`FOR UPDATE SKIP LOCKED` 让 N 个 Worker 同时拉取互不冲突的任务。没有它，Worker 之间会产生锁竞争，最终退化为串行处理。

## Java（Spring Boot）实现示例

```java
@Component
public class JobPoller {
    private final JdbcTemplate jdbc;

    @Scheduled(fixedRate = 100)  // 每个 Worker 每 100ms 轮询一次
    public void poll() {
        String sql = """
            UPDATE jobs SET status = 'running', attempts = attempts + 1, updated_at = now()
            WHERE id = (
                SELECT id FROM jobs
                WHERE status = 'pending' AND scheduled_at <= now()
                ORDER BY priority DESC, scheduled_at
                FOR UPDATE SKIP LOCKED LIMIT 1
            )
            RETURNING id, queue, payload::text
            """;

        Map<String, Object> job = jdbc.queryForMap(sql);
        if (job != null) {
            processJob(job);  // 异步处理任务
        }
    }

    private void processJob(Map<String, Object> job) {
        Long id = (Long) job.get("id");
        try {
            // 执行实际业务逻辑...
            jdbc.update("UPDATE jobs SET status = 'done', updated_at = now() WHERE id = ?", id);
        } catch (Exception e) {
            jdbc.update("""
                UPDATE jobs SET status = CASE
                    WHEN attempts >= max_retries THEN 'failed'
                    ELSE 'pending'
                END, updated_at = now() WHERE id = ?
                """, id);
        }
    }
}
```

## 高级模式

### 批量拉取（提高吞吐）

```sql
UPDATE jobs SET status = 'running', updated_at = now()
WHERE id IN (
    SELECT id FROM (
        SELECT id FROM jobs
        WHERE status = 'pending' AND scheduled_at <= now()
        ORDER BY priority DESC, scheduled_at
        FOR UPDATE SKIP LOCKED
        LIMIT 10       -- 一次拉取 10 个任务
    ) AS batch
)
RETURNING *;
```

### 分队列处理

```sql
-- 按不同 queue 名称拉取，实现队列隔离
WHERE status = 'pending' AND queue = 'email' AND scheduled_at <= now()
```

### 心跳 + 超时重新调度

```sql
-- 标记 running 超过 5 分钟仍未完成的任务为 pending，让其他 Worker 重新处理
UPDATE jobs SET
    status = 'pending',
    updated_at = now()
WHERE status = 'running'
  AND updated_at < now() - INTERVAL '5 minutes';
```

## 注意事项

- **事务提交后才可见**：Worker 更新任务状态后需提交事务，其他 Worker 才能看到变化
- **Worker 崩溃处理**：务必配合心跳机制（如上），防止 running 状态的任务永远卡死
- **SKIP LOCKED vs NOWAIT**：`SKIP LOCKED` 跳过锁冲突的行；`NOWAIT` 在遇到锁时直接报错。队列场景用 SKIP LOCKED
- **PostgreSQL 版本**：`FOR UPDATE SKIP LOCKED` 需要 PostgreSQL 9.5+
- **连接池大小**：Worker 数量 + 连接池大小需匹配。每个 Worker 需一个连接
- **适用规模**：本模式适合中小规模（<1000 任务/秒）。更高吞吐请使用 Redis Streams 或 RabbitMQ
- **分区建议**：如果 jobs 表增长到千万级，按 created_at 分区存储历史数据
