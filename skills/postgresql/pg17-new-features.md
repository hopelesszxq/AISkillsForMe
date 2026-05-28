---
name: pg17-new-features
description: PostgreSQL 17 新特性详解：增量备份、JSON_TABLE、性能提升与 SQL/JSON 增强
tags: [postgresql, pg17, performance, json, backup, vacuum]
---

## 概述

PostgreSQL 17（2024年9月发布）在性能、开发者体验、运维管理方面有大量改进。以下是 Java/Spring 开发者最需要关注的新特性。

## 1. JSON_TABLE 支持

### 功能说明

将 JSON 数据转换为关系型表格，无需使用 `jsonb_to_recordset` 或子查询。

### 示例

```sql
-- PG17 之前：使用 jsonb_to_recordset
SELECT * FROM jsonb_to_recordset(
    '[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]'
) AS t(id int, name text);

-- PG17：使用 JSON_TABLE
SELECT * FROM JSON_TABLE(
    '[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]'::jsonb,
    '$[*]' COLUMNS (
        id int PATH '$.id',
        name text PATH '$.name'
    )
) AS jt;
```

### 结合表查询

```sql
-- 从订单表的 JSON 字段提取行项目
SELECT o.order_id, jt.product_id, jt.quantity, jt.price
FROM orders o,
JSON_TABLE(o.items_json, '$[*]' COLUMNS (
    product_id int PATH '$.productId',
    quantity int PATH '$.qty',
    price numeric(10,2) PATH '$.price'
)) AS jt
WHERE o.order_date >= '2025-01-01';
```

## 2. 增量备份（pg_basebackup）

### 功能说明

PG17 引入内置的增量备份功能，无需额外工具（如 pgBackRest）。

### 配置

```bash
# 启用 WAL 归档
wal_level = replica
archive_mode = on
archive_command = 'cp %p /backup/wal/%f'

# 开启增量备份支持
summarize_wal = on   # PG17 新增，为增量备份生成 WAL 摘要
```

### 使用

```bash
# 创建全量基础备份
pg_basebackup -D /backup/full -Ft -z -P

# 创建增量备份（基于上一次备份）
pg_basebackup -D /backup/inc1 -i /backup/full -Ft -z -P

# 查看备份链
pg_verifybackup --manifest /backup/full/backup_manifest
```

## 3. MERGE 增强（WHEN NOT MATCHED BY SOURCE）

### 示例

```sql
-- PG17 支持同步源表删除
MERGE INTO target t
USING source s ON t.id = s.id
WHEN MATCHED THEN
    UPDATE SET t.name = s.name, t.updated_at = now()
WHEN NOT MATCHED THEN
    INSERT (id, name) VALUES (s.id, s.name)
WHEN NOT MATCHED BY SOURCE THEN  -- PG17 新增
    DELETE;
```

### 使用场景

```sql
-- 缓存同步：用源表数据刷新缓存表，删除过期数据
MERGE INTO product_cache c
USING product p ON c.product_id = p.id
WHEN MATCHED AND c.stock != p.stock THEN
    UPDATE SET stock = p.stock, version = version + 1
WHEN NOT MATCHED THEN
    INSERT (product_id, stock, version)
    VALUES (p.id, p.stock, 1)
WHEN NOT MATCHED BY SOURCE THEN
    DELETE;
```

## 4. 性能提升

### 并行查询优化

```sql
-- PG17 改进了并行查询的计划生成
-- 更智能地选择 Bitmap Scan vs. Index Scan
-- 并行 VACUUM 现在处理更多场景

-- 查看并行查询配置
SHOW max_parallel_workers;
SHOW max_parallel_workers_per_gather;
SHOW parallel_tuple_cost;
```

### B-tree 索引优化

```sql
-- PG17 减少了 B-tree 索引的 WAL 写入量（减少约 20%）
-- 空值索引性能提升
CREATE INDEX idx_users_email ON users(email) WHERE email IS NOT NULL;
```

### 流式 I/O

```sql
-- PG17 新增 streaming read API，大表全表扫描更快
-- Seq Scan 性能提升 10-20%
SET enable_streaming_reads = on;  -- 默认开启
```

## 5. VACUUM 改进

### 新增参数

```sql
-- PG17 VACUUM 自动调整缓冲区大小
VACUUM (VERBOSE, BUFFER_USAGE_LIMIT '2GB') my_table;

-- 新选项：SHRINK（仅 PG17+）
-- 回收表末端的空洞空间（不阻塞读写）
ALTER TABLE my_table SET (vacuum_index_cleanup = auto);
```

### 系统视图增强

```sql
-- 查看索引使用效率（PG17 新增列）
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    last_idx_scan   -- PG17 新增
FROM pg_stat_user_indexes
WHERE last_idx_scan IS NULL
   OR last_idx_scan < now() - interval '7 days'
ORDER BY last_idx_scan NULLS FIRST;
```

## 6. SQL/JSON 标准增强

```sql
-- PG17 支持 JSON EXISTS
SELECT json_exists(
    '{"name":"Alice","age":30}'::jsonb,
    '$.name'
);

-- JSON_VALUE 支持默认值
SELECT json_value(
    '{"name":"Alice"}'::jsonb,
    '$.age'
    DEFAULT 0 ON ERROR
) AS age;

-- JSON 正则表达式支持
SELECT json_query(
    '["a","b","c"]'::jsonb,
    '$ ? (@ like_regex "^a")'
);
```

## 7. 逻辑复制增强

```sql
-- PG17 逻辑复制支持两阶段提交
-- 发布端：
CREATE PUBLICATION my_pub FOR ALL TABLES;
ALTER PUBLICATION my_pub SET (two_phase_commit = true);

-- 订阅端：
CREATE SUBSCRIPTION my_sub
CONNECTION 'host=primary port=5432 dbname=mydb'
PUBLICATION my_pub
WITH (two_phase_commit = true);

-- 查看复制进度
SELECT * FROM pg_stat_subscription;

-- PG17 新增：订阅端支持 TRUNCATE
ALTER SUBSCRIPTION my_sub SET (truncate = true);
```

## 8. 开发者体验改进

### EXPLAIN 增强

```sql
-- PG17 EXPLAIN 支持 SERIALIZABLE 和更详细的 JSON 输出
EXPLAIN (ANALYZE, SERIALIZE, FORMAT JSON)
SELECT * FROM orders WHERE user_id = 42;

-- 新增 GENERIC_PLAN 显示
EXPLAIN (GENERIC_PLAN)
SELECT * FROM users WHERE id = $1;
```

### 正则表达式增强

```sql
-- PG17 支持非捕获组、原子组等
SELECT 'hello123' ~ 'hello(?:[0-9]+)';  -- true
SELECT 'aaaa' ~ '(?>a+)b';              -- 原子组防止回溯
```

## 升级注意

1. **pg_upgrade 支持跨大版本直接升级**：PG16 → PG17 不再需要中间版本
2. **检查废弃功能**：
   - `plpython2` 已移除
   - `wal_compression=always` 改为默认值
3. **兼容性检查**：
   ```bash
   # 升级前运行
   pg_upgrade --check
   ```
4. **Rolling upgrade with logical replication** 在生产中更安全
