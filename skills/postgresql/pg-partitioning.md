---
name: pg-partitioning
description: PostgreSQL 表分区实战与性能优化
tags: [postgresql, database, partitioning, performance]
---

## 分区类型总览

| 分区类型 | 适用场景 | 示例 |
|---|---|---|
| **RANGE** | 时间序列数据（日志、订单） | 按月/按天分区 |
| **LIST** | 枚举值分区（地区、状态） | 按省份、按状态 |
| **HASH** | 数据均匀分布、无自然分区键 | 按 ID 哈希 |
| **复合分区** | 多维度分区 | 先 RANGE 再 LIST |

## 声明式分区（Declarative Partitioning）

### RANGE 分区（按月）

```sql
-- 创建主表
CREATE TABLE orders (
    id              BIGSERIAL,
    order_no        VARCHAR(32) NOT NULL,
    user_id         BIGINT NOT NULL,
    total_amount    DECIMAL(12,2),
    status          VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, created_at)  -- 分区键必须包含在主键中
) PARTITION BY RANGE (created_at);

-- 创建子分区表（每月一个）
CREATE TABLE orders_2026_01 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE orders_2026_02 PARTITION OF orders
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

CREATE TABLE orders_2026_03 PARTITION OF orders
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');

-- 默认分区（捕获未匹配的数据）
CREATE TABLE orders_default PARTITION OF orders DEFAULT;

-- 查询自动分区裁剪
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE created_at >= '2026-01-15' AND created_at < '2026-02-15';
-- 只扫描 orders_2026_01 和 orders_2026_02，忽略其他分区
```

### LIST 分区

```sql
CREATE TABLE user_login_log (
    id          BIGSERIAL,
    user_id     BIGINT NOT NULL,
    region      VARCHAR(10) NOT NULL,  -- 地区编码
    login_time  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ip_address  INET,
    PRIMARY KEY (id, region)
) PARTITION BY LIST (region);

CREATE TABLE log_north PARTITION OF user_login_log
    FOR VALUES IN ('BJ', 'TJ', 'HB');

CREATE TABLE log_south PARTITION OF user_login_log
    FOR VALUES IN ('GD', 'SZ', 'GX');

CREATE TABLE log_other PARTITION OF user_login_log DEFAULT;
```

### HASH 分区

```sql
CREATE TABLE events (
    id          BIGSERIAL,
    event_type  VARCHAR(50),
    payload     JSONB,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (id, event_type)
) PARTITION BY HASH (id);

CREATE TABLE events_p0 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE events_p1 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE events_p2 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE events_p3 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

## 子分区（复合分区）

```sql
-- 先按年 RANGE 分区，再按区域 LIST 子分区
CREATE TABLE audit_log (
    id          BIGSERIAL,
    action      VARCHAR(50),
    region      VARCHAR(10),
    operator    VARCHAR(64),
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (id, created_at, region)
) PARTITION BY RANGE (created_at);

-- 2026 年的分区，再按区域子分区
CREATE TABLE audit_log_2026 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01')
    PARTITION BY LIST (region);

CREATE TABLE audit_log_2026_bj PARTITION OF audit_log_2026
    FOR VALUES IN ('BJ');
CREATE TABLE audit_log_2026_sh PARTITION OF audit_log_2026
    FOR VALUES IN ('SH');
CREATE TABLE audit_log_2026_gd PARTITION OF audit_log_2026
    FOR VALUES IN ('GD');
```

## 分区管理维护

### 自动创建分区（pg_partman 扩展）

```sql
-- 安装扩展
CREATE EXTENSION pg_partman;

-- 配置按月自动创建分区（提前创建未来 3 个月）
SELECT partman.create_parent(
    p_parent_table := 'public.orders',
    p_control := 'created_at',
    p_type := 'native',
    p_interval := '1 month',
    p_premake := 3
);

-- 更新分区配置
UPDATE partman.part_config
SET retention = '6 months',          -- 保留 6 个月
    retention_keep_table = false     -- 删除超过保留期的分区
WHERE parent_table = 'public.orders';

-- 手动运行维护（可放 cron）
SELECT partman.run_maintenance();
```

### 手动分区管理

```sql
-- 创建新分区
CREATE TABLE orders_2026_04 PARTITION OF orders
    FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');

-- 删除旧分区（DROP 比 DELETE 快几个数量级）
DROP TABLE IF EXISTS orders_2025_01;

-- 从分区表中分离（先分离再转成普通表）
ALTER TABLE orders DETACH PARTITION orders_2025_06;
-- 分离后可以独立操作，也可以重新 ATTACH
ALTER TABLE orders ATTACH PARTITION orders_2025_06
    FOR VALUES FROM ('2025-06-01') TO ('2025-07-01');
```

## 分区索引

```sql
-- 每个子分区自动继承主表的索引定义
-- 但也可以为不同分区创建不同索引

-- 在子分区上创建本地索引
CREATE INDEX idx_orders_2026_01_user ON orders_2026_01(user_id);
CREATE INDEX idx_orders_2026_02_user ON orders_2026_02(user_id);

-- PostgreSQL 12+ 支持在主表上创建统一索引
CREATE INDEX idx_orders_user_id ON orders(user_id) LOCAL;
-- LOCAL 表示在每个分区上创建本地索引

-- 全局索引（PostgreSQL 不支持，只能用本地索引）
```

## 分区裁剪优化

```sql
-- 需要分区键在 WHERE 条件中显式出现才能裁剪
-- ✅ 能裁剪
EXPLAIN SELECT * FROM orders
WHERE created_at >= '2026-01-01' AND created_at < '2026-03-01';

-- ❌ 不能裁剪（分区键被函数包裹）
EXPLAIN SELECT * FROM orders
WHERE DATE(created_at) >= '2026-01-01';

-- ✅ 改为范围查询
EXPLAIN SELECT * FROM orders
WHERE created_at >= '2026-01-01'::timestamptz;

-- ❌ 不能裁剪（使用 OR 条件）
EXPLAIN SELECT * FROM orders
WHERE created_at >= '2026-02-01' OR user_id = 123;

-- 分区键 + 非分区键组合查询
-- ✅ 先按分区键裁剪，再在分区内查询
EXPLAIN SELECT * FROM orders
WHERE created_at BETWEEN '2026-01-01' AND '2026-02-01'
  AND user_id = 12345;
-- 只扫描 orders_2026_01 分区上的索引
```

## 性能对比数据

| 场景 | 单表 1 亿行 | 按月分区（12 个月） |
|---|---|---|
| 全表扫描 | 45s | 分区裁剪后 ~3.5s |
| 按时间范围删除 | DELETE 5 分钟 | `DROP TABLE` < 1s |
| VACUUM 开销 | 极大，影响业务 | 单个分区小，影响范围小 |
| 索引维护 | 大索引重建耗时 | 按需重建小分区索引 |

## 注意事项

1. **分区键必须在主键中**：PostgreSQL 要求所有唯一索引（包括主键）必须包含分区键
2. **分区数限制**：建议单表分区不超过 1000 个，过多分区影响计划器性能
3. **外键限制**：分区表不能引用其他表的外键（子分区不能有外键）
4. **UPDATE 移动分区**：如果 UPDATE 导致记录移动到新分区（如 created_at 更新），PostgreSQL 不支持，会报错
5. **BEFORE ROW 触发器**：分区表不支持 BEFORE ROW 触发器
6. **分区裁剪依赖查询条件**：`WHERE` 条件必须包含分区键且不能是函数包装形式
7. **批量插入性能**：分区表导入大量数据时，每个 INSERT 都要路由到正确分区，建议使用 `COPY` 直接写入子分区
8. **运维自动化**：生产环境务必使用 `pg_partman` 或定时任务自动管理分区生命周期
