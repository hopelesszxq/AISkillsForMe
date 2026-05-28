---
name: pg-logical-replication
description: PostgreSQL 逻辑复制（Logical Replication）实战：CDC、跨版本同步与冲突处理
tags: [postgresql, database, replication, cdc, logical]
---

## 概述

PostgreSQL 逻辑复制基于 **发布（Publication）** 和 **订阅（Subscription）** 模型，支持：

- 跨大版本升级（如 PG12 → PG17）零停机迁移
- 选择性复制（只复制部分表）
- 实时 CDC（Change Data Capture）数据同步到数仓或消息队列
- 双向同步（需注意冲突处理）

## 架构图

```
发布端 (Publisher)                   订阅端 (Subscriber)
┌────────────────────┐              ┌────────────────────┐
│   WAL → 逻辑解码    │ ── 网络流 ──→ │   apply worker     │
│   pgoutput 插件     │              │   写入目标表        │
│   发布: my_pub      │              │   订阅: my_sub      │
└────────────────────┘              └────────────────────┘
```

## 基础配置

### 发布端配置

```bash
# postgresql.conf 修改
wal_level = logical           # 必须设为 logical（默认 replica）
max_replication_slots = 10    # 至少 ≥ 订阅数，预留余量
max_wal_senders = 10          # 至少 ≥ 订阅数
# 以上参数需要重启生效
```

```sql
-- 重启后验证
SHOW wal_level;
-- 预期输出: logical
```

### 创建发布

```sql
-- 发布所有表（仅限已存在的表）
CREATE PUBLICATION my_pub FOR ALL TABLES;

-- 发布指定表（推荐，更可控）
CREATE PUBLICATION order_pub FOR TABLE orders, order_items;

-- 发布指定表 + WHERE 条件过滤（PG15+）
CREATE PUBLICATION active_order_pub
FOR TABLE orders WHERE (status IN ('PAID', 'SHIPPED'));

-- 只发布 INSERT（不发布 UPDATE/DELETE）
CREATE PUBLICATION insert_only_pub
FOR TABLE orders
WITH (publish = 'insert');
```

### 查看发布

```sql
-- 查看所有发布
SELECT * FROM pg_publication;

-- 查看发布包含的表
SELECT * FROM pg_publication_tables;
```

### 订阅端配置

```sql
-- 创建订阅（连接到发布端）
CREATE SUBSCRIPTION my_sub
CONNECTION 'host=192.168.1.10 port=5432 dbname=mydb user=repl_user password=xxx'
PUBLICATION my_pub;

-- 复制数据到指定表空间
CREATE SUBSCRIPTION my_sub
CONNECTION '...'
PUBLICATION my_pub
WITH (create_slot = true, copy_data = true, origin = none);
```

### 查看订阅

```sql
-- 查看所有订阅
SELECT * FROM pg_subscription;

-- 查看复制进度
SELECT * FROM pg_stat_subscription;

-- 查看复制槽
SELECT slot_name, slot_type, active, restart_lsn
FROM pg_replication_slots;
```

## 零停机迁移方案

跨大版本升级时，逻辑复制是最安全的方式：

```
1. 新版本 PG 作为订阅端
2. 旧版本 PG 作为发布端
3. 全量数据首次同步 (copy_data=true)
4. 增量持续同步（追平落后）
5. 验证数据一致性
6. 切换连接字符串
7. 删除订阅和发布
```

```sql
-- 步骤1：在旧库创建发布
CREATE PUBLICATION migration_pub FOR ALL TABLES;

-- 步骤2：在新库创建订阅（自动全量+增量）
CREATE SUBSCRIPTION migration_sub
CONNECTION 'host=old-db port=5432 dbname=mydb user=repl password=xxx'
PUBLICATION migration_pub
WITH (copy_data = true);

-- 步骤3：监控延迟
SELECT
    pid,
    application_name,
    state,
    sync_state,
    pg_wal_lsn_diff(
        pg_current_wal_lsn(),
        write_lsn
    ) AS write_lag_bytes
FROM pg_stat_replication;

-- 步骤4：验证数据一致性后切换
-- 新库验证完后，停旧库写流量，等待最后延迟追平
ALTER SUBSCRIPTION migration_sub DISABLE;

-- 步骤5：清理
DROP SUBSCRIPTION migration_sub;
DROP PUBLICATION migration_pub;
```

## CDC 实战：同步到消息队列

结合 pgoutput + Debezium + Kafka 实现实时 CDC：

```yaml
# docker-compose.yml 示例
version: '3.8'
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: secret
    command:
      - "postgres"
      - "-c"
      - "wal_level=logical"
      - "-c"
      - "max_replication_slots=10"
      - "-c"
      - "max_wal_senders=10"

  debezium:
    image: debezium/connect:2.7
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: "1"
      CONFIG_STORAGE_TOPIC: "debezium_configs"
      OFFSET_STORAGE_TOPIC: "debezium_offsets"
      STATUS_STORAGE_TOPIC: "debezium_statuses"
```

```json
// Debezium connector 配置
{
  "name": "order-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "myuser",
    "database.password": "secret",
    "database.dbname": "mydb",
    "database.server.name": "pg-server",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_slot",
    "publication.name": "debezium_pub",
    "table.include.list": "public.orders,public.order_items",
    "publication.autocreate.mode": "filtered",
    "decimal.handling.mode": "string",
    "tombstones.on.delete": "false"
  }
}
```

## 冲突处理

逻辑复制遇到冲突时，订阅端 worker 会报错并停止复制。

### 常见冲突类型

| 冲突类型 | 触发条件 | 解决方案 |
|---------|---------|---------|
| **主键冲突** | 订阅端已存在相同 PK 的记录，发布端又 INSERT | 订阅端提前清理冲突数据 |
| **数据不存在** | UPDATE/DELETE 的目标行在订阅端不存在 | 检查数据同步完整性 |
| **外键冲突** | 插入数据引用了订阅端不存在的 FK | 确保 FK 表先同步或复制到同一订阅 |
| **检查约束** | 订阅端有更严格的 CHECK 约束 | 保持两端约束一致 |

### 冲突策略

```sql
-- PG16+ 支持冲突自动跳过
CREATE SUBSCRIPTION my_sub
CONNECTION '...'
PUBLICATION my_pub
WITH (
    -- PG16+ 冲突处理选项
    conflict_resolution = 'apply'     -- 可选: 'apply' (覆盖), 'skip' (跳过)
    -- 或者异步处理：
    -- disable_on_error = false        -- 出错不自动禁用订阅
);
```

### 手动解决冲突

```sql
-- 1. 查看订阅错误
SELECT subname, subenabled, subskiptransaction
FROM pg_subscription;

-- 2. 查看日志中的冲突详情
-- 在 PG 日志中搜索 "logical replication"

-- 3. 跳过冲突事务（跳过单条冲突 SQL）
ALTER SUBSCRIPTION my_sub SKIP TRANSACTION '<lsn>';

-- 4. 或直接禁用/启用来重新同步
ALTER SUBSCRIPTION my_sub DISABLE;
-- 手动在订阅端修复数据
ALTER SUBSCRIPTION my_sub ENABLE;

-- 5. 如果冲突过多，重建订阅
DROP SUBSCRIPTION my_sub;
-- 注意：不会自动清理复制槽，需要手动
SELECT pg_drop_replication_slot('my_sub');
```

## PG17 逻辑复制增强

```sql
-- PG17 支持两阶段提交（需要发布端和订阅端都开启）
CREATE PUBLICATION my_pub FOR ALL TABLES
WITH (two_phase_commit = true);

CREATE SUBSCRIPTION my_sub
CONNECTION '...'
PUBLICATION my_pub
WITH (two_phase_commit = true);

-- PG17 订阅端支持 TRUNCATE
ALTER SUBSCRIPTION my_sub SET (truncate = true);

-- PG17 新增 pg_stat_subscription_stats 视图
SELECT * FROM pg_stat_subscription_stats;
```

## 性能优化

```bash
# postgresql.conf 调优

# 订阅端并行应用（PG14+）
max_parallel_apply_workers_per_subscription = 4

# 增加接收缓冲区
wal_receiver_buffer_size = 1GB        # 默认 64MB

# 减少 WAL 保留量
# 定期清理不再需要的复制槽
```

```sql
-- 监控复制延迟（秒级）
SELECT
    CASE
        WHEN pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn()
            THEN 0
        ELSE EXTRACT(SECOND FROM now() - pg_last_xact_replay_timestamp())
    END AS replication_lag_seconds;

-- 查看复制统计
SELECT slot_name, database, active,
       pg_size_pretty(
           pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
       ) AS retained_wal
FROM pg_replication_slots
WHERE slot_type = 'logical';
```

## 注意事项

1. **DDL 不同步**：逻辑复制只同步 DML（INSERT/UPDATE/DELETE/TRUNCATE），不会同步 DDL（ALTER TABLE 等）。需要手动在两端执行 DDL，或使用 pglogical 扩展
2. **序列不同步**：自增序列（SERIAL/BIGSERIAL）的值不会同步，订阅端需要提前设置序列起点
3. **大对象不同步**：BYTEA/LO 类型不受影响，但 pg_largeobject 不被复制
4. **DDL 锁影响**：发布端执行 DDL 时，会等待所有复制槽确认，可能导致写阻塞。建议在低峰期执行 DDL
5. **复制槽管理**：未消费的 WAL 会堆积，导致磁盘满。监控复制槽延迟，及时清理废弃订阅
6. **网络延迟敏感**：逻辑复制依赖网络稳定性，建议在可信内网中使用
7. **订阅端只读建议**：订阅端表的数据由复制管理，不要在订阅端直接写入，否则容易产生冲突
8. **部分表复制**：`CREATE PUBLICATION ... FOR TABLE ...` 只复制列名匹配的列，表结构不同时可能失败
