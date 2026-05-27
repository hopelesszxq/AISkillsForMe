---
name: pg-connection-vacuum
description: PostgreSQL 连接池调优、Vacuum 维护与性能监控实战
tags: [postgresql, database, connection-pool, vacuum, performance]
---

## 连接池配置调优

### HikariCP 最佳参数

```yaml
# Spring Boot application.yml
spring:
  datasource:
    type: com.zaxxer.hikari.HikariDataSource
    hikari:
      # 核心参数：池大小
      maximum-pool-size: 20        # 最大连接数（推荐 2*cpu+disk）
      minimum-idle: 5              # 最小空闲连接
      connection-timeout: 5000     # 获取连接超时（ms）
      idle-timeout: 300000         # 空闲连接超时（5分钟）
      max-lifetime: 900000         # 连接最大存活（15分钟，< 数据库连接超时）
      keepalive-time: 60000        # 心跳探测间隔（1分钟）

      # 诊断与优化
      pool-name: PgPool
      leak-detection-threshold: 10000  # 连接泄漏检测（10秒未归还）
      register-mbeans: true            # 暴露 JMX 监控指标
```

### 连接池大小计算公式

```
# 核心公式（基于 PostgreSQL 官方文档）
connections = ((core_count * 2) + effective_spindle_count)

# 示例：4核CPU + 1块SSD → 9个连接
# 示例：8核CPU + RAID10(4盘) → 20个连接

# 对于纯IO密集型任务，上述公式可能偏小
# 建议压测验证：从 2*cpu 开始，逐步增加，观察 TPS 拐点
```

### 连接池监控指标

```sql
-- 查看当前活跃/空闲/等待的连接数
SELECT
  state,
  count(*) AS connections,
  wait_event_type,
  wait_event
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY state, wait_event_type, wait_event
ORDER BY connections DESC;

-- 查看最大连接数配置
SHOW max_connections;          -- 默认 100，生产建议 200-500

-- 查看每个连接消耗的内存
-- 每个连接约消耗 2-10MB（sort_mem + work_mem 等）
SELECT name, setting, unit
FROM pg_settings
WHERE name IN ('work_mem', 'maintenance_work_mem', 'shared_buffers');
```

## PgBouncer 轻量连接池

高并发场景（>200 连接）需引入 PgBouncer，避免 PostgreSQL 进程数过多：

```ini
; pgbouncer.ini
[databases]
* = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# 连接池模式
# session: 归还连接 → 断开（适合短连接）
# transaction: 归还连接 → 归还到池（推荐 Web 应用）
# statement: 执行完语句即归还（最激进）
pool_mode = transaction

# 池大小配置
default_pool_size = 25         # 每个用户/数据库对的最大连接数
max_client_conn = 200          # 最大客户端连接数
reserve_pool_size = 5          # 预留连接数
reserve_pool_timeout = 5       # 预留连接超时（秒）

# 超时设置
server_idle_timeout = 600      # 服务端空闲连接超时（秒）
client_idle_timeout = 0        # 客户端空闲超时（0=不限制）
query_timeout = 30             # 查询超时（秒）
```

```yaml
# Spring Boot 通过 PgBouncer 连接
spring:
  datasource:
    url: jdbc:postgresql://127.0.0.1:6432/mydb
    hikari:
      maximum-pool-size: 10    # 比直连更小，因为 PgBouncer 已做汇聚
```

## Vacuum 与 Autovacuum 调优

### 为什么需要 Vacuum

PostgreSQL 的 MVCC 机制导致更新/删除后产生死元组（dead tuples），需要 Vacuum 回收：

- **VACUUM**：回收死元组空间，更新统计信息，不锁表
- **VACUUM FULL**：物理收缩表大小，**会锁表**（生产慎用）
- **ANALYZE**：更新统计信息，不回收空间

### Autovacuum 关键参数

```sql
-- 查看当前 autovacuum 配置
SELECT name, setting, unit, short_desc
FROM pg_settings
WHERE name LIKE 'autovacuum%';

-- 推荐生产配置
ALTER SYSTEM SET autovacuum = on;
ALTER SYSTEM SET autovacuum_max_workers = 4;          -- 并行 worker 数（建议 cpu/2）
ALTER SYSTEM SET autovacuum_naptime = '30s';          -- 检查间隔
ALTER SYSTEM SET autovacuum_vacuum_threshold = 100;   -- 触发阈值（死元组数）
ALTER SYSTEM SET autovacuum_vacuum_scale_factor = 0.05; -- 触发比例（5% 死元组）
ALTER SYSTEM SET autovacuum_vacuum_cost_limit = 1000; -- IO 成本限制
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = '2ms'; -- 每次操作的延迟
```

### 单表自定义 Autovacuum

对大表单独调整，避免默认参数不合适：

```sql
-- 大表（> 100GB）自动频繁 vacuum 会消耗大量 IO
-- 降低频率，提高阈值
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.1,    -- 10% 死元组才触发
  autovacuum_vacuum_threshold = 10000,     -- 至少 1万 死元组
  autovacuum_analyze_scale_factor = 0.05,
  autovacuum_vacuum_cost_limit = 2000      -- 提高 IO 预算，加快 vacuum
);

-- 小表（频繁更新）提高频率
ALTER TABLE user_sessions SET (
  autovacuum_vacuum_scale_factor = 0.01,   -- 1% 死元组即触发
  autovacuum_vacuum_threshold = 50,        -- 至少 50 个
  autovacuum_vacuum_cost_delay = '1ms'     -- IO 延迟更小
);
```

### 监控死元组与膨胀

```sql
-- 查看表的死元组比例（膨胀程度）
SELECT
  schemaname,
  relname,
  n_live_tup,
  n_dead_tup,
  round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
  last_autovacuum,
  last_autoanalyze
FROM pg_stat_user_tables
WHERE n_live_tup > 10000   -- 排除小表
ORDER BY dead_pct DESC
LIMIT 20;

-- 查看 autovacuum 工作情况
SELECT
  pid,
  datname,
  query,
  state,
  wait_event,
  NOW() - query_start AS duration
FROM pg_stat_activity
WHERE query LIKE 'VACUUM%' OR query LIKE 'autovacuum%'
ORDER BY query_start;
```

### 手动 Vacuum 策略

```sql
-- 生产环境：只 VACUUM，不要 FULL
VACUUM (ANALYZE, VERBOSE) orders;

-- 紧急情况：死元组比例 > 50%，阻塞了事务回卷
VACUUM (FREEZE, VERBOSE) orders;

-- VACUUM FULL 仅在如下场景使用：
-- 1. 维护窗口期（夜间低峰）
-- 2. 已用 DROP TABLE 删除了大量数据后
-- 3. 配合 pg_repack 在线整理（不锁表）
VACUUM FULL orders;  -- 会锁表，谨慎！
```

### pg_repack 在线整理表

```bash
# 安装 pg_repack 扩展
apt install postgresql-16-repack

# 在线重建表（不阻塞读写）
pg_repack -h localhost -d mydb -t orders
```

## 慢查询监控

```sql
-- 启用慢查询日志（可动态设置）
ALTER SYSTEM SET log_min_duration_statement = 1000;  -- 记录超过 1秒 的查询
ALTER SYSTEM SET log_autovacuum_min_duration = 1000;
ALTER SYSTEM SET log_lock_waits = on;
SELECT pg_reload_conf();

-- 使用 pg_stat_statements 分析
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- 查询最耗时的 SQL（前 20）
SELECT
  queryid,
  round(total_exec_time::numeric, 2) AS total_ms,
  calls,
  round(mean_exec_time::numeric, 2) AS avg_ms,
  round((100 * total_exec_time / SUM(total_exec_time) OVER ())::numeric, 2) AS pct,
  query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- 查询 IO 密集的 SQL
SELECT
  queryid,
  calls,
  shared_blks_read + shared_blks_hit AS total_blks,
  round(shared_blks_read * 100.0 / NULLIF(shared_blks_read + shared_blks_hit, 0), 2) AS read_pct,
  query
FROM pg_stat_statements
WHERE shared_blks_read > 0
ORDER BY shared_blks_read DESC
LIMIT 20;
```

## 注意事项

1. **不要盲目增大连接池**：PostgreSQL 每个连接都是独立进程，过多连接（>500）会导致上下文切换开销超过并行收益
2. **HikariCP maximum-pool-size** 与 **PgBouncer default_pool_size** 配合使用：应用层池 + 数据库连接池两层汇聚
3. **长事务阻塞 Vacuum**：运行中的事务会阻止 autovacuum 回收死元组，导致表膨胀。监控 `pg_stat_activity` 中 `state = 'idle in transaction'` 的连接
4. **事务 ID 回卷**：PostgreSQL 事务 ID 是 32 位（~20亿），需在达到 2^31 之前 FREEZE。监控 `SELECT datname, age(datfrozenxid) FROM pg_database;`，超过 10亿 需处理
5. **Vacuum 频率不是越高越好**：过于频繁的 autovacuum 会消耗大量 IO/CPU，大表建议调高 scale_factor 和 cost_limit
6. **`idle in transaction` 杀手**：应用层必须设置事务超时（`@Transactional(timeout = 30)`），防止连接长时间持有事务
