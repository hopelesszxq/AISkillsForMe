---
name: pg-autovacuum-tuning
description: PostgreSQL 18/19 Autovacuum 深度调优：防事务回卷、大表策略、等待事件分析与最佳配置
tags: [postgresql, vacuum, autovacuum, performance, tuning, dba]
---

## 概述

Autovacuum 是 PostgreSQL 最关键的"基础设施"进程。配置不当会导致：事务 ID 回卷（数据库宕机）、表膨胀、性能雪崩。PG18/19 引入了多项自动清理改进，但也增加了调优复杂度。

> ⚠️ 事务回卷（Transaction Wraparound）是 PostgreSQL 最严重的事故之一——一旦发生，数据库将进入只读模式强制清理。

## 一、PG18/19 Autovacuum 新特性

### PG18 增强
- **并行自动清理（Parallel Autovacuum）**：`autovacuum_max_parallel_workers` 参数，允许单表多 worker 并行清理（默认 2）
- **AIO 支持**：异步 I/O 子系统提升清理过程的读取效率
- **`vacuum_buffer_usage_limit`**：限制 VACUUM 的共享缓冲区使用量，避免挤占业务查询

### PG19 增强
- **REPACK 命令**：替代 VACUUM FULL，在线重建表且不阻塞写入（详见 `pg19-new-features.md`）
- **自动清理调度优化**：更智能的触发策略，减少对小表的频繁扫描

## 二、核心参数调优模板

```ini
# postgresql.conf — 生产环境 Autovacuum 配置

# --- 基础开关 ---
autovacuum = on                          # 必须开启
track_counts = on                        # 必须开启（默认 on）

# --- 资源控制 ---
autovacuum_max_workers = 4               # 总 worker 数：CPU 核心数 / 2
autovacuum_max_parallel_workers = 2      # PG18+ 单表并行 worker（大表推荐 2-4）
autovacuum_naptime = 30s                 # 调度间隔：大表多设为 45s-60s
autovacuum_vacuum_threshold = 100        # 触发阈值（小表）
autovacuum_vacuum_scale_factor = 0.01    # 触发比例（大表：表大小的 1%）
autovacuum_vacuum_insert_threshold = 500 # 仅插入表触发阈值（PG14+）
autovacuum_vacuum_insert_scale_factor = 0.01
autovacuum_vacuum_cost_limit = 2000      # PG18 默认提升到 2000（旧版 200）
autovacuum_vacuum_cost_delay = 2ms       # 延时：高 I/O 环境 5-10ms，SSD 可 0

# --- 防止回卷（最关键参数）---
autovacuum_freeze_max_age = 500000000    # 触发防回卷清理的年龄阈值（最大 20 亿）
autovacuum_multixact_freeze_max_age = 400000000
vacuum_freeze_min_age = 50000000         # 标记为冻结的最早 XID 年龄
vacuum_freeze_table_age = 150000000      # 触发全表冻结扫描的年龄

# --- PG18+ 新参数 ---
vacuum_buffer_usage_limit = 256          # VACUUM 最大使用 buffer（单位 8KB，默认 256=2MB）
autovacuum_work_mem = 256MB              # 每个 worker 可用内存（大表 512MB-1GB）
```

## 三、按表类型的分级策略

### 大表（>100GB）
```sql
-- 按表单独设置，覆盖全局参数
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.001,        -- 0.1% 即触发
    autovacuum_vacuum_threshold = 50000,           -- 最少 5 万行变化
    autovacuum_vacuum_cost_limit = 4000,           -- 更高 I/O 预算
    autovacuum_vacuum_cost_delay = 0,              -- 无延迟（SSD 环境）
    autovacuum_freeze_max_age = 300000000          -- 提前防回卷
);
```

### 频繁 INSERT 的日志表
```sql
ALTER TABLE audit_log SET (
    autovacuum_vacuum_insert_threshold = 100000,   -- 10 万条插入后触发
    autovacuum_vacuum_insert_scale_factor = 0.0,
    autovacuum_freeze_max_age = 200000000          -- 日志表降低防回卷阈值
);
```

### 小表（<1GB）
```sql
-- 小表用默认配置即可，过多清理反而浪费
ALTER TABLE configs SET (
    autovacuum_vacuum_scale_factor = 0.05,         -- 5% 才触发
    autovacuum_vacuum_threshold = 50
);
```

## 四、监控与诊断

### 4.1 查看当前 Autovacuum 进度
```sql
SELECT pid, datname, relid::regclass AS table_name,
       phase, heap_blks_total, heap_blks_scanned,
       heap_blks_vacuumed, index_vacuum_count,
       max_dead_tuples, num_dead_tuples,
       now() - query_start AS duration
FROM pg_stat_progress_vacuum
WHERE datname = current_database();
```

### 4.2 查看表膨胀风险
```sql
SELECT schemaname, relname,
       n_live_tup, n_dead_tup,
       ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
       ROUND(age(relfrozenxid)) AS xid_age,
       ROUND(age(relminmxid)) AS mxid_age,
       last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY dead_pct DESC
LIMIT 20;
```

### 4.3 等待事件分析
```sql
-- Autovacuum 相关的等待事件
SELECT pid, wait_event_type, wait_event, state,
       LEFT(query, 80) AS query,
       now() - query_start AS duration
FROM pg_stat_activity
WHERE query ILIKE '%autovacuum%'
   OR query ILIKE '%VACUUM%';
```

常见等待事件：

| 等待事件 | 含义 | 调优方向 |
|---------|------|---------|
| Vacuum:IndexCleanup | 索引清理中 | 增加 `autovacuum_work_mem` |
| Vacuum:Truncate | 截断末尾空页 | 正常行为，频繁则调小 `vacuum_truncate` |
| IO:DataFileRead | 读取数据文件 | 存储性能不足，考虑 SSD |
| LWLock:BufferMapping | 缓冲池争用 | 调大 `shared_buffers` 或减 `autovacuum_max_workers` |

## 五、防回卷（Anti-Wraparound）实战

### 触发条件
```
age(relfrozenxid) > autovacuum_freeze_max_age（默认 2 亿）
→ 自动触发防回卷清理（忽略其他所有限制，全力清理）
```

### 紧急处理步骤

```sql
-- 1. 检查最旧的表
SELECT relname, age(relfrozenxid) AS xid_age,
       pg_size_pretty(pg_total_relation_size(oid))
FROM pg_class
WHERE relkind = 'r' AND relfrozenxid != 0
ORDER BY age(relfrozenxid) DESC
LIMIT 10;

-- 2. 当 xid_age > 15 亿时，手动触发紧急清理
VACUUM (FREEZE, INDEX_CLEANUP ON, PARALLEL 4) orders;

-- 3. PG19 可用 REPACK 替代
REPACK TABLE orders;

-- 4. 环境允许时降低参数（临时）
-- ALTER SYSTEM SET autovacuum_freeze_max_age = '300000000';
-- SELECT pg_reload_conf();
```

### 回卷预防最佳实践
- **监控报警**：`age(relfrozenxid) > 10亿` 发出警告，`> 15亿` 立即处理
- **定期清理**：对大表执行定期 `VACUUM (FREEZE)` 维护窗口
- **PG18+**：利用并行 autovacuum 提高大表清理速度
- **分区表**：分区表逐分区清理，避免单次超长冻结

## 六、PG18/19 特定调优注意事项

1. **并行 Autovacuum 并非越快越好**：`autovacuum_max_parallel_workers` 设置过高会导致 I/O 风暴，建议从 2 起步观察
2. **`vacuum_buffer_usage_limit` 过小**：导致 VACUUM 反复读取数据页，建议大表设为 512-1024
3. **SSD 环境**：`autovacuum_vacuum_cost_delay = 0`、`autovacuum_vacuum_cost_limit = 10000` 充分压榨性能
4. **PG19 REPACK vs VACUUM FULL**：REPACK 不阻塞写入，推荐替代 VACUUM FULL；但需要额外磁盘空间（表的 1.5x）
5. **搭配 PG18 AIO 使用**：设置 `io_method = worker` 并分配足够 `io_workers` 提升清理 I/O 效率
