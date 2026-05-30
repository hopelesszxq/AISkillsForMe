---
name: pg18-optimizer-monitoring
description: PostgreSQL 18 优化器与监控增强：AIO子系统、Skip Scan、自连接消除、Vacuum改进
tags: [postgresql, pg18, performance, optimizer, monitoring, vacuum, aio]
---

## 概述

PostgreSQL 18 正式版（2025-09-25 发布）除 UUIDv7、虚拟生成列、RETURNING OLD/NEW 等已知特性外，还引入了大量优化器和监控层面的增强。本文聚焦这些**更底层但对性能影响巨大**的变化。

> 已有技能 `pg18-new-features.md` 覆盖了 UUIDv7、虚拟生成列、RETURNING OLD/NEW、并行逻辑复制等特性，本文作为补充。

## 1. 异步 I/O 子系统（AIO）

### 功能说明

PG18 引入全新的异步 I/O 子系统（`io_method`），允许后端进程**批量排队多个读请求**，显著提升顺序扫描、位图堆扫描、Vacuum 等操作的 I/O 效率。

```ini
# postgresql.conf
# 启用 AIO（默认 off）
io_method = 'io_uring'        # 可选: 'sync'(默认), 'io_uring', 'posix_aio'
io_combine_limit = '512kB'    # I/O 合并上限
io_max_combine_limit = '8MB'  # 最大合并大小
effective_io_concurrency = 16 # PG18 默认值从 1 提升到 16
maintenance_io_concurrency = 16
```

### 监控 AIO 状态

新增系统视图 `pg_aios`，查看当前异步 I/O 的文件句柄使用情况：

```sql
-- 查看正在进行的 AIO 请求
SELECT * FROM pg_aios;

-- 查看 AIO 使用统计
SELECT io_method, 
       count(*) AS active_requests,
       sum(pending_bytes) AS total_pending
FROM pg_aios
GROUP BY io_method;
```

### 性能影响

| 场景 | sync（默认） | io_uring | 提升 |
|------|-------------|----------|------|
| 顺序扫描（大表） | 基准 | -30% 耗时 | ~30% |
| Vacuum 全表扫描 | 基准 | -40% 耗时 | ~40% |
| 位图堆扫描 | 基准 | -25% 耗时 | ~25% |

## 2. B-tree Skip Scan

### 功能说明

PG18 支持对**多列 B-tree 索引**进行 "Skip Scan" 查找。当查询条件跳过索引的前导列时，优化器不再需要全索引扫描，而是直接"跳"到匹配项。

```sql
-- 创建多列索引
CREATE INDEX idx_orders_status_date ON orders(status, created_at);

-- 传统上，以下查询无法高效使用 idx_orders_status_date 的第一列
-- 因为 WHERE 中没有 status 条件。PG18 通过 Skip Scan 解决。
SELECT * FROM orders
WHERE created_at >= '2026-01-01'
ORDER BY created_at
LIMIT 100;

-- Skip Scan 会自动遍历 status 的各种不同值，
-- 对每个 status 值执行 B-tree 定位查找
```

### 适用场景

- 多列索引但查询条件未覆盖前导列
- 前导列只有非等值条件（如 `>`、`<`）
- 数据分布稀疏，跳过大量无效页

### 与传统索引扫描对比

```sql
-- 查看执行计划，确认是否使用了 Skip Scan
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders
WHERE created_at >= '2026-01-01'
ORDER BY created_at
LIMIT 100;

-- 输出中应包含 "Skip Scan" 字样
--                                   QUERY PLAN
-- --------------------------------------------------------------------
--  Limit  (cost=...)
--    ->  Skip Scan using idx_orders_status_date on orders  (cost=...)
--          Skip Condition: (status IS NOT NULL)
```

## 3. 优化器增强

### 3.1 自连接消除

优化器**自动移除不必要的表自连接**：

```sql
-- PG18 可以自动将以下查询优化为单表扫描
SELECT a.*
FROM orders a, orders b
WHERE a.id = b.id
  AND a.created_at > '2026-01-01';
-- 等价于:
SELECT * FROM orders WHERE created_at > '2026-01-01';
```

可通过 `enable_self_join_elimination = off` 关闭。

### 3.2 DISTINCT 键重排序

允许 `SELECT DISTINCT` 的列在内部重排序以避免排序操作：

```sql
-- 如果 (category, status) 有索引，但查询写的是 DISTINCT status, category
-- PG18 可以自动重排序以匹配索引
SELECT DISTINCT status, category FROM orders;
-- 可通过 enable_distinct_reordering = off 关闭
```

### 3.3 HAVING 下推到 WHERE

允许 `GROUPING SETS` 上的 `HAVING` 子句下推到 `WHERE` 进行提前过滤：

```sql
SELECT department, SUM(salary)
FROM employees
GROUP BY GROUPING SETS ((department), ())
HAVING SUM(salary) > 100000;
-- PG18 将 HAVING 条件提前过滤，减少中间结果
```

### 3.4 Right Semi Join

新增 **Right Semi Join** 计划类型，半连接（EXISTS / IN 子查询）有更多优化选择：

```sql
-- 优化器可选择 Right Semi Join
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

### 3.5 MERGE JOIN + 增量排序

MERGE JOIN 现在可以使用增量排序（incremental sort），减少内存占用：

```sql
-- 当排序的某部分已经有序时，增量排序可减少排序开销
SET enable_incremental_sort = on;
```

### 3.6 分区查询优化

- 查询访问**大量分区**时的计划效率显著提升
- 分区连接（partitionwise join）在更多场景启用，且内存消耗降低
- 分区查询的成本估算更准确

## 4. Vacuum 重要改进

### 4.1 冻结所有可见页

普通 VACUUM 现在可以**冻结所有可见（all-visible）页面**，即使当前不需要冻结。这减少后续全表冻结的开销：

```ini
# 控制冻结激进程度（默认 0，表示不主动冻结）
vacuum_max_eager_freeze_failure_rate = 0.01
# 也可在表级别设置
```

```sql
ALTER TABLE orders SET (vacuum_max_eager_freeze_failure_rate = 0.01);
```

### 4.2 vacuum_truncate 配置

新增 GUC 参数 `vacuum_truncate`，控制 Vacuum 是否截断文件末尾的空页：

```ini
vacuum_truncate = on   # 默认 on，设置为 off 可减少 I/O 抖动
```

### 4.3 VACUUM/ANALYZE ONLY 选项

VACUUM 和 ANALYZE 可只处理分区表本身而不处理子分区：

```sql
-- 仅处理父表（分区表）的元数据，不处理子分区
VACUUM ONLY partitioned_table;
ANALYZE ONLY partitioned_table;
```

这在自动清理（autovacuum）场景中很有用，因为 autovacuum 默认不处理分区表本身。

## 5. 监控增强

### 5.1 连接日志粒度提升

`log_connections` 从布尔值扩展为支持详细级别，可报告连接各阶段的耗时：

```ini
log_connections = 'verbose'   # 可选: off, on(默认), verbose
```

verbose 模式下，日志会输出 DNS 解析、SSL 握手、认证等各阶段的耗时。

### 5.2 关系统计信息管理

新增 4 个函数用于**手动管理统计信息**：

```sql
-- 恢复关系级统计信息到指定值
SELECT pg_restore_relation_stats('orders', '{
  "relpages": 10000,
  "reltuples": 5000000
}');

-- 恢复列级统计信息
SELECT pg_restore_attribute_stats('orders', 'status', '{
  "n_distinct": 5,
  "null_frac": 0.0
}');

-- 清除统计信息（让下次 ANALYZE 重新计算）
SELECT pg_clear_relation_stats('orders');
SELECT pg_clear_attribute_stats('orders', 'status');
```

这允许在无法执行 ANALYZE 的环境中手动设置、恢复或清除统计信息。

### 5.3 锁性能改进

查询访问大量关系的**锁性能得到显著改善**，涉及分区表或大量外键场景受益明显。

## 6. 其他实用性改进

| 特性 | 说明 |
|------|------|
| GIN 索引并行创建 | `CREATE INDEX CONCURRENTLY` 支持 GIN 索引 |
| IN (VALUES ...) → ANY | 优化器自动转换以获得更好的统计信息 |
| OR 子句转数组 | `WHERE a = 1 OR a = 2` 转换为 `a = ANY(ARRAY[1,2])` 加速索引处理 |
| generate_series() 估算改进 | 对数值和时间戳类型的行数估算更准确 |
| SQL 函数计划缓存改进 | 减少 SQL 语言函数的重复计划开销 |

## 注意事项

1. **AIO 依赖内核支持**：`io_uring` 需要 Linux 5.1+ 内核，生产环境建议 Linux 6.0+
2. **Skip Scan 非万能**：当前导列的不同值非常多时，Skip Scan 效率可能不如全表扫描，优化器会自动判断
3. **VACUUM 冻结策略**：`vacuum_max_eager_freeze_failure_rate` 设置过小会导致 Vacuum 做过多额外工作
4. **统计信息函数权限**：`pg_restore_*` 函数需要超级用户或 `pg_maintain` 角色
5. **分区 ONLY 选项**：使用 `ONLY` 后，autovacuum 仍会处理子分区
6. **升级建议**：升级 PG18 后，建议先启用 `io_method = 'io_uring'` 和 `enable_self_join_elimination = on`（默认开启）以获得最大性能收益
