---
name: pg18-new-features
description: PostgreSQL 18 新特性：UUIDv7、虚拟生成列、RETURNING OLD/NEW、并行逻辑复制
tags: [postgresql, pg18, replication, uuid, generated-column, performance]
---

## 概述

PostgreSQL 18（预计 2025 年下半年发布）进一步强化了数据仓库与实时复制能力。核心变化：UUIDv7 内置支持、虚拟生成列（VIRTUAL）、`RETURNING OLD/NEW` DML 扩展、并行逻辑复制默认启用等。

## 1. UUIDv7 内置支持

### 功能说明

UUIDv7 生成**时间有序的 UUID**，使 B-tree 索引在写入大表时保持"热"，避免随机 UUID 导致的索引页分裂和写入放大。

```sql
-- 主键默认使用 UUIDv7
CREATE TABLE retail.orders (
    order_id    uuid PRIMARY KEY DEFAULT uuidv7(),
    customer_id uuid NOT NULL,
    order_ts    timestamptz NOT NULL DEFAULT now(),
    unit_price  numeric(12,2) NOT NULL
);

-- UUIDv7 是有序的，因此以下查询索引友好
SELECT * FROM retail.orders
WHERE order_id > '0195f000-0000-0000-0000-000000000000'
ORDER BY order_id
LIMIT 100;
```

### 与传统 UUID 对比

| 特性 | UUIDv4（随机） | UUIDv7（时间有序） |
|---|---|---|
| 索引写入性能 | ❌ 随机插入导致页分裂 | ✅ 顺序插入，索引紧凑 |
| 冲突概率 | 极低 | 极低 |
| 可排序 | ❌ | ✅ 按生成时间排序 |
| 适用场景 | 传统 OLTP | 大型事实表、时序数据 |

## 2. 虚拟生成列（VIRTUAL Generated Columns）

### 功能说明

PostgreSQL 18 支持 `GENERATED ALWAYS AS (...) VIRTUAL`，计算列值不实际存储，仅在查询时计算，节省磁盘空间。

```sql
CREATE TABLE retail.orders (
    order_id     uuid PRIMARY KEY DEFAULT uuidv7(),
    unit_price   numeric(12,2) NOT NULL,
    quantity     integer NOT NULL CHECK (quantity > 0),
    total_amount numeric(14,2) GENERATED ALWAYS AS (unit_price * quantity) VIRTUAL,
    month_bucket date GENERATED ALWAYS AS (date_trunc('month', order_ts)::date) VIRTUAL
);

-- 查询时自动使用计算值
SELECT order_id, total_amount, month_bucket
FROM retail.orders
WHERE month_bucket = '2025-06-01'
LIMIT 10;
```

### 存储列 vs 虚拟列

| 特性 | STORED | VIRTUAL |
|---|---|---|
| 磁盘占用 | ✅ 占用 | ❌ 不占用 |
| 写入性能 | ❌ 每次写入都计算 | ✅ 只读时计算 |
| 索引支持 | ✅ 可直接建索引 | ❌ 不能直接建索引（可用表达式索引替代） |

## 3. RETURNING OLD/NEW 扩展

### 功能说明

PostgreSQL 18 在 `UPDATE`/`DELETE` 的 `RETURNING` 子句中支持 `OLD`/`NEW` 关键字，方便审计、CDC 和 ETL 场景。

```sql
-- 更新时同时返回旧值和新值
UPDATE retail.customers
SET full_name = 'Budi S.'
WHERE email = 'budi@example.com'
RETURNING 'OLD' AS which_old, OLD.full_name,
          'NEW' AS which_new, NEW.full_name;

-- 删除时返回被删除行的旧值
DELETE FROM retail.orders
WHERE order_ts < now() - interval '1 year'
RETURNING 'DELETED' AS action, OLD.order_id, OLD.total_amount;
```

### 应用场景

- **审计日志**：记录变更前后的值
- **CDC 系统**：直接获取变更数据，无需额外查询
- **ETL 增量同步**：一次性拿到新增/变更数据

## 4. 并行逻辑复制（默认启用）

### 功能说明

PostgreSQL 18 将并行逻辑应用（Parallel Logical Apply）设为默认行为。订阅者使用多个 worker 并行回放来自发布者的变更，大幅缩短大表初始同步和持续复制的延迟。

```sql
-- 发布者端创建发布
CREATE PUBLICATION pub_retail
FOR TABLE retail.customers, retail.orders;

-- 订阅者端创建订阅（自动使用并行apply）
CREATE SUBSCRIPTION sub_retail
CONNECTION 'host=pg18_pub port=5432 dbname=analytics user=admin password=pubpass'
PUBLICATION pub_retail
WITH (copy_data = true);

-- 查看复制状态
SELECT subname, status, received_lsn, latest_end_time
FROM pg_stat_subscription;
```

### 5. idle_replication_slot_timeout

PostgreSQL 18 新增 `idle_replication_slot_timeout` 参数，当订阅者长时间断开时自动清理复制槽，防止 WAL 日志堆积撑爆磁盘。

```ini
# postgresql.conf
idle_replication_slot_timeout = '15min'
```

## 注意事项

1. **UUIDv7 依赖时钟**：系统时间回拨可能导致 UUID 生成顺序混乱，多节点部署需确保 NTP 同步
2. **虚拟列不可直接索引**：如需对虚拟列做范围查询，使用表达式索引 `CREATE INDEX ON t((col1 + col2))`
3. **RETURNING OLD/NEW** 仅在行级触发器或订阅的上下文中生效，普通查询无需此语法
4. **并行逻辑复制** 需要足够的 CPU 和内存，订阅者 `max_parallel_workers` 需合理配置
5. **升级注意**：从 PG17 升级到 PG18 需运行 `pg_upgrade`，逻辑复制槽会保留
