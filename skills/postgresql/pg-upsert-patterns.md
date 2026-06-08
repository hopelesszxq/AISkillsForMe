---
name: pg-upsert-patterns
description: PostgreSQL UPSERT（INSERT...ON CONFLICT / MERGE）实战模式与性能调优
tags: [postgresql, upsert, merge, insert, conflict, performance]
---

## 概述

PostgreSQL 的 UPSERT（INSERT ... ON CONFLICT）和 MERGE（PG15+）是实现"存在则更新，不存在则插入"的两种方式。本文覆盖常见场景的最佳实践、并发安全与性能优化。

## 一、INSERT ... ON CONFLICT（标准 UPSERT）

### 1.1 基础用法

```sql
-- 单条 UPSERT
INSERT INTO users (id, name, email, updated_at)
VALUES (1, '张三', 'zhangsan@example.com', NOW())
ON CONFLICT (id) DO UPDATE
SET name    = EXCLUDED.name,
    email   = EXCLUDED.email,
    updated_at = EXCLUDED.updated_at;
```

`EXCLUDED` 指代试图插入但冲突的那行数据。

### 1.2 批量 UPSERT（推荐）

```sql
INSERT INTO products (sku, name, price, stock, updated_at)
VALUES
    ('SKU001', '商品A', 29.9, 100, NOW()),
    ('SKU002', '商品B', 39.9, 200, NOW()),
    ('SKU003', '商品C', 49.9, 150, NOW())
ON CONFLICT (sku) DO UPDATE
SET name       = EXCLUDED.name,
    price      = EXCLUDED.price,
    stock      = products.stock + EXCLUDED.stock,  -- 累加库存
    updated_at = EXCLUDED.updated_at;
```

> **性能提示**：单条 INSERT 提交的开销远高于批量。建议单次 batch 控制在 500~2000 行，超过 10K 行建议使用 COPY + 临时表。

### 1.3 部分更新——仅当值发生变化时

```sql
INSERT INTO settings (user_id, config_key, config_value)
VALUES (42, 'theme', 'dark')
ON CONFLICT (user_id, config_key) DO UPDATE
SET config_value = EXCLUDED.config_value,
    updated_at   = NOW()
WHERE settings.config_value IS DISTINCT FROM EXCLUDED.config_value;
```

`IS DISTINCT FROM` 同时处理 NULL 和非 NULL 的比较，避免无意义的 UPDATE。

### 1.4 不更新、仅忽略（INSERT IGNORE 语义）

```sql
INSERT INTO logs (request_id, message, created_at)
VALUES ('req-001', 'some log', NOW())
ON CONFLICT (request_id) DO NOTHING;
```

适用于幂等写入场景（如 Webhook 回调、事件去重）。

## 二、MERGE 语句（PG 15+）

MERGE 语法更接近标准 SQL，可在一个语句中实现 INSERT/UPDATE/DELETE 的任意组合。

### 2.1 基础 MERGE

```sql
MERGE INTO inventory AS t
USING (VALUES ('SKU001', 50)) AS s(sku, qty)
ON t.sku = s.sku
WHEN MATCHED THEN
    UPDATE SET stock = t.stock + s.qty, updated_at = NOW()
WHEN NOT MATCHED THEN
    INSERT (sku, stock, created_at, updated_at)
    VALUES (s.sku, s.qty, NOW(), NOW());
```

### 2.2 MERGE 多条件分支

```sql
MERGE INTO orders AS t
USING (SELECT id, status FROM staging_orders WHERE batch_id = 20260601) AS s
ON t.id = s.id
WHEN MATCHED AND t.status = 'PENDING' THEN
    UPDATE SET status = s.status, updated_at = NOW()
WHEN MATCHED AND t.status = 'SHIPPED' THEN
    DO NOTHING
WHEN NOT MATCHED THEN
    INSERT (id, status, created_at, updated_at)
    VALUES (s.id, s.status, NOW(), NOW());
```

### 2.3 MERGE + DELETE（超时清理）

```sql
MERGE INTO sessions AS t
USING (SELECT id, last_active FROM active_sessions) AS s
ON t.id = s.id
WHEN MATCHED AND s.last_active < NOW() - INTERVAL '30 days' THEN
    DELETE
WHEN NOT MATCHED THEN
    INSERT (id, user_id, last_active)
    VALUES (s.id, 1, s.last_active);
```

## 三、高频数据导入场景

### 3.1 日志去重写入

```sql
-- 建表时创建唯一约束
CREATE TABLE event_log (
    event_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source      TEXT NOT NULL,
    event_type  TEXT NOT NULL,
    payload     JSONB,
    received_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (source, event_type, received_at)
);

-- 去重写入
INSERT INTO event_log (source, event_type, payload, received_at)
VALUES ($1, $2, $3, $4)
ON CONFLICT (source, event_type, received_at) DO NOTHING;
```

### 3.2 计数器累加

```sql
INSERT INTO page_views (page_slug, view_count, updated_at)
VALUES ('/home', 1, NOW())
ON CONFLICT (page_slug) DO UPDATE
SET view_count = page_views.view_count + 1,
    updated_at = NOW();
```

## 四、并发安全与锁机制

### 4.1 ON CONFLICT 的锁行为

- `ON CONFLICT DO UPDATE` 在冲突检测时获取行级锁（Row Share Lock）
- 高并发下的同一行 UPSERT 会引发锁等待，重试可提升吞吐

```sql
-- 建议配合重试模式（应用层或 PG savepoint）
BEGIN;
SAVEPOINT before_upsert;
INSERT INTO counters (id, val) VALUES (1, 1)
ON CONFLICT (id) DO UPDATE SET val = counters.val + 1;
COMMIT;
```

### 4.2 避免死锁

批量 UPSERT 时，**应用层对输入数据按主键排序**，可减少死锁概率：

```java
// Java 示例：排序后批量 upsert
List<Record> records = getRecords();
records.sort(Comparator.comparing(Record::getId));

// 使用 JDBC batch
try (PreparedStatement ps = conn.prepareStatement(
    "INSERT INTO t (id, val) VALUES (?, ?) ON CONFLICT (id) DO UPDATE SET val = EXCLUDED.val")) {
    for (Record r : records) {
        ps.setLong(1, r.getId());
        ps.setString(2, r.getVal());
        ps.addBatch();
    }
    ps.executeBatch();
}
```

## 五、性能对比

| 方式 | 场景 | 吞吐量（rows/s） | 说明 |
|------|------|-----------------|------|
| INSERT...ON CONFLICT batch | 批量 upsert | 80K~120K | 推荐，简单高效 |
| MERGE | 复杂条件逻辑 | 40K~80K | 灵活但解析开销略高 |
| 逐行 SELECT + UPDATE/INSERT | 应用层判断 | <5K | 不推荐，高并发差 |
| COPY + 临时表 + ON CONFLICT | 超大批量导入 | 200K+ | 最快方案 |

## 六、超大批量导入方案

```sql
-- 1. COPY 到临时表
CREATE TEMP TABLE tmp_products (LIKE products INCLUDING ALL);
COPY tmp_products FROM '/data/products.csv' WITH (FORMAT CSV, HEADER);

-- 2. 批量 UPSERT
INSERT INTO products (id, name, price, stock)
SELECT id, name, price, stock FROM tmp_products
ON CONFLICT (id) DO UPDATE
SET name  = EXCLUDED.name,
    price = EXCLUDED.price,
    stock = EXCLUDED.stock;

-- 3. 清理临时表
DROP TABLE tmp_products;
```

## 注意事项

1. **ON CONFLICT 必须指定约束**：必须是唯一索引或主键约束。不支持 ON CONFLICT WHERE 条件子句（PG 16+ 实验支持部分索引）。
2. **EXCLUDED 别名**：只能引用被插入的值，不能引用其他表。
3. **MERGE 的 WHERE 陷阱**：MERGE 的 WHEN MATCHED/WHEN NOT MATCHED 条件仅在对应的匹配状态内生效，不要混淆。
4. **触发器行为**：ON CONFLICT DO UPDATE 会触发 BEFORE UPDATE / AFTER UPDATE 触发器，而 DO NOTHING 不会触发任何 INSERT 触发器。
5. **CTID 变化**：ON CONFLICT DO UPDATE 实际是 DELETE + INSERT，行的 ctid 会变化，不支持在 update 中引用旧行除约束列以外的值。
6. **MERGE 的锁升级**：PG 15/16 的 MERGE 实现有时会获取比 ON CONFLICT 更强的锁，高并发场景优先选择 INSERT...ON CONFLICT。
