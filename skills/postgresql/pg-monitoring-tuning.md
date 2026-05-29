---
name: pg-monitoring-tuning
description: PostgreSQL 监控体系与性能调优实战：关键指标、慢查询分析、连接管理、参数调优
tags: [postgresql, database, monitoring, performance, tuning, dba]
---

## 概述

PostgreSQL 的性能问题通常来自：慢 SQL、连接风暴、索引失效、参数配置不当。本文覆盖从监控指标采集到参数调优的完整链路，帮助快速定位和解决生产环境性能问题。

## 1. 关键监控指标

### 核心 SQL 查询

```sql
-- 1. 活跃会话与等待事件
SELECT pid, state, wait_event_type, wait_event, 
       query_start, state_change, 
       LEFT(query, 100) AS query
FROM pg_stat_activity
WHERE state != 'idle'
  AND backend_type = 'client backend'
ORDER BY query_start;

-- 2. 锁等待监控
SELECT blocked.pid AS blocked_pid,
       blocked.query AS blocked_query,
       blocking.pid AS blocking_pid,
       blocking.query AS blocking_query,
       pg_blocking_pids(blocked.pid)
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking 
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));

-- 3. 慢查询 TOP 10（基于 pg_stat_statements）
SELECT queryid,
       ROUND(total_exec_time::numeric, 2) AS total_ms,
       calls,
       ROUND(mean_exec_time::numeric, 2) AS avg_ms,
       ROUND((100 * total_exec_time / SUM(total_exec_time) OVER ())::numeric, 2) AS pct,
       LEFT(query, 80) AS query
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat%'
ORDER BY total_exec_time DESC
LIMIT 10;

-- 4. 命中率检查
SELECT 'index hit rate' AS name,
       ROUND((sum(idx_blks_hit)::numeric / 
            NULLIF(sum(idx_blks_hit + idx_blks_read), 0) * 100), 2) AS ratio
FROM pg_statio_user_indexes
UNION ALL
SELECT 'table hit rate',
       ROUND((sum(heap_blks_hit)::numeric / 
            NULLIF(sum(heap_blks_hit + heap_blks_read), 0) * 100), 2)
FROM pg_statio_user_tables;
```

### 连接与事务

```sql
-- 5. 当前连接数（按数据库分组）
SELECT datname, state, COUNT(*) AS count
FROM pg_stat_activity
WHERE backend_type = 'client backend'
GROUP BY datname, state
ORDER BY count DESC;

-- 6. 长事务（运行超过 5 分钟的事务）
SELECT pid, 
       EXTRACT(EPOCH FROM (NOW() - xact_start)) / 60 AS minutes,
       state, 
       LEFT(query, 80)
FROM pg_stat_activity
WHERE xact_start < NOW() - INTERVAL '5 minutes'
  AND state = 'active'
ORDER BY xact_start;

-- 7. 未使用的索引（冗余索引排查）
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelid NOT IN (
    SELECT indexrelid FROM pg_stat_user_indexes WHERE idx_scan > 0
)
ORDER BY tablename;
```

## 2. pg_stat_statements 配置

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000
pg_stat_statements.track = all
pg_stat_statements.track_utility = off
pg_stat_statements.save = on

# 需要重启后创建扩展
# CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

## 3. auto_explain（慢查询自动日志）

```ini
# postgresql.conf
shared_preload_libraries = 'auto_explain'

# 记录执行时间超过 500ms 的查询及其执行计划
auto_explain.log_min_duration = '500ms'
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_nested_statements = on
auto_explain.log_timing = on
auto_explain.log_triggers = on

# 重启后生效，日志中会输出 EXPLAIN ANALYZE 结果
```

## 4. 核心参数调优

```ini
# postgresql.conf — 适用于 16GB 内存服务器的参考配置

# 内存配置（建议：总内存的 25%）
shared_buffers = '4GB'                # 建议 RAM 的 25%
effective_cache_size = '12GB'         # 建议 RAM 的 75%
work_mem = '32MB'                     # 每个排序/哈希操作的内存
maintenance_work_mem = '1GB'          # VACUUM/INDEX 操作内存

# 写入配置
wal_buffers = '16MB'                  # WAL 缓冲区
max_wal_size = '4GB'                 # 最大 WAL 大小
min_wal_size = '1GB'
checkpoint_completion_target = 0.9    # 分散检查点写入压力

# 并行查询
max_parallel_workers_per_gather = 2   # 每个查询并行度
max_parallel_workers = 8             # 总并行 worker 数
parallel_tuple_cost = 0.1            # 降低使优化器更倾向并行
parallel_setup_cost = 100

# 连接配置
max_connections = 200                 # 最大连接数（不要超过 500）
superuser_reserved_connections = 10   # 留给超级用户修复问题

# 自动清理（使用默认即可，以下为推荐值）
autovacuum = on
autovacuum_max_workers = 3
autovacuum_naptime = '60s'
autovacuum_vacuum_scale_factor = 0.01
autovacuum_vacuum_threshold = 50
autovacuum_analyze_scale_factor = 0.005
autovacuum_analyze_threshold = 50
```

## 5. 连接池配置建议

### 应用层（HikariCP）

```yaml
# application.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20          # 不要超过 (2 * CPU核数) + 1
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      leak-detection-threshold: 60000 # 连接泄漏检测
```

### 中间件层（PgBouncer）

```ini
# pgbouncer.ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# 事务级连接池（推荐 Web 应用）
pool_mode = transaction
default_pool_size = 40
max_client_conn = 200
reserve_pool_size = 10
reserve_pool_timeout = 3.0

# 连接生命周期
server_idle_timeout = 300
client_idle_timeout = 600
query_timeout = 30
```

## 6. 索引调优检查

```sql
-- 查找缺失的索引（需要 pg_stat_user_tables 统计）
SELECT relname AS table_name,
       seq_scan,
       seq_tup_read,
       seq_tup_read / NULLIF(seq_scan, 0) AS avg_rows_per_seq_scan,
       idx_scan
FROM pg_stat_user_tables
WHERE seq_scan > 1000 
  AND seq_tup_read > 100000
  AND idx_scan = 0
ORDER BY seq_tup_read DESC;

-- 重复索引检查
SELECT pg_size_pretty(SUM(pg_relation_size(idx))::bigint) AS total_size,
       string_agg(indexname, ', ') AS indexes
FROM (
    SELECT indrelid::regclass AS table_name,
           indexrelid::regclass AS indexname,
           pg_get_indexdef(indexrelid) AS def
    FROM pg_index
) t
GROUP BY table_name, def
HAVING COUNT(*) > 1;
```

## 7. 系统级监控脚本

```bash
#!/bin/bash
# pg_monitor.sh — 快速健康检查

echo "=== PostgreSQL 健康检查 ==="
echo "时间: $(date)"

psql -c "
SELECT 
    '连接数' AS metric,
    COUNT(*)::text AS value
FROM pg_stat_activity 
WHERE backend_type = 'client backend'
UNION ALL
SELECT 
    '活跃事务',
    COUNT(*)::text
FROM pg_stat_activity 
WHERE state = 'active' AND backend_type = 'client backend'
UNION ALL
SELECT 
    '等待事件',
    COUNT(*)::text
FROM pg_stat_activity 
WHERE wait_event IS NOT NULL AND state = 'active'
UNION ALL
SELECT 
    '长事务(>5min)',
    COUNT(*)::text
FROM pg_stat_activity 
WHERE xact_start < NOW() - INTERVAL '5 min'
  AND state = 'active';
"

echo "=== 磁盘使用 ==="
du -sh /var/lib/postgresql/data/base/*/ 2>/dev/null

echo "=== WAL 生成速率(MB/s) ==="
psql -c "
SELECT ROUND((pg_wal_lsn_diff(pg_current_wal_lsn(), stats.write_lsn) / 1024 / 1024)::numeric, 2)
FROM pg_stat_bgwriter;
" 2>/dev/null || echo "pg_stat_bgwriter 不可用"
```

## 注意事项

- **shared_buffers 不要超过 RAM 的 25%**：过大反而会因为内核缓存争用导致性能下降
- **max_connections 不要设置过高**：每个连接大约消耗 5~10MB 内存，建议配合 PgBouncer 使用
- **work_mem 是 per-operation 而非 per-connection**：排序/哈希操作每次分配，过高容易耗尽内存
- **pg_stat_statements 清空**：`SELECT pg_stat_statements_reset();` 在调优前后执行对比
- **检查点风暴**：如果发现周期性性能抖动，检查 `max_wal_size` 是否过小或 `checkpoint_completion_target` 不合理
- **autovacuum 不要关闭**：关闭会导致事务 ID 回卷（XID Wraparound），数据库变成只读
