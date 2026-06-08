---
name: pg-hypopg
description: PostgreSQL HypoPG 扩展实战——创建假设索引测试查询优化效果而不消耗实际磁盘空间
tags: [postgresql, indexing, hypopg, performance, query-optimization, hypothetical-index]
---

## 概述

[HypoPG](https://github.com/HypoPG/hypopg) 是一个 PostgreSQL 扩展，允许创建**假设（虚拟）索引**——这些索引不实际存储在磁盘上，但能让 `EXPLAIN` 认为它们存在，从而展示如果创建该索引后查询计划会如何变化。

### 解决的核心问题

- 在生产库上创建索引前，先确认索引是否真的会被查询使用
- 避免"创建索引后发现查询计划没变"的浪费
- 测试多列索引、部分索引、覆盖索引的不同组合效果
- 无需大量磁盘空间和写入 IO 开销即可评估索引效果

## 安装

```bash
# 从源码安装
git clone https://github.com/HypoPG/hypopg.git
cd hypopg
make && sudo make install

# 在目标数据库中启用
psql -d mydb -c "CREATE EXTENSION hypopg;"
```

### 验证安装

```sql
SELECT * FROM pg_available_extensions WHERE name = 'hypopg';
-- name  | default_version | installed_version | comment
-- hypopg | 1.4             | 1.4              | Hypothetical indexes
```

## 基础用法

### 1. 创建假设索引

```sql
-- 创建假设的 B-tree 索引
SELECT * FROM hypopg_create_index('CREATE INDEX ON orders (user_id)');

-- 返回虚拟索引的 OID
-- indexrelid | indexname
-- 12345      | <12345>btree_orders_user_id
```

### 2. 查看假设索引

```sql
-- 列出所有假设索引
SELECT * FROM hypopg_list_indexes;

-- 只看数量
SELECT COUNT(*) FROM hypopg_list_indexes;
```

### 3. 用 EXPLAIN 评估效果

```sql
-- 先看在没有索引时的查询计划
EXPLAIN SELECT * FROM orders WHERE user_id = 42;
--                          QUERY PLAN
-- -------------------------------------------------------------
--  Seq Scan on orders  (cost=0.00..12345.00 rows=1 width=200)
--  (实际上全表扫描)

-- 创建假设索引
SELECT hypopg_create_index('CREATE INDEX ON orders (user_id)');

-- 再次查看查询计划（假设索引生效）
EXPLAIN SELECT * FROM orders WHERE user_id = 42;
--                          QUERY PLAN
-- -------------------------------------------------------------
--  Index Scan using <12345>btree_orders_user_id on orders
--    (cost=0.29..8.30 rows=1 width=200)
--  (计划使用了假设索引！)
```

> **注意**：假设索引名以 `<oid>` 开头，`EXPLAIN` 结果中会显示虚拟索引名，说明查询优化器认为索引存在。

### 4. 删除假设索引

```sql
-- 用 OID 删除
SELECT hypopg_drop_index(12345);

-- 删除所有假设索引（连接断开或事务回滚也会自动清除）
SELECT hypopg_reset();
```

## 高级场景

### 1. 测试复合索引的列顺序

```sql
-- 测试 (status, created_at) vs (created_at, status)
SELECT hypopg_create_index('CREATE INDEX ON orders (status, created_at)');
EXPLAIN SELECT * FROM orders WHERE status = 'PAID' ORDER BY created_at DESC;
-- 看是否用了 Index Scan Backward

-- 重置后测试另一种顺序
SELECT hypopg_reset();
SELECT hypopg_create_index('CREATE INDEX ON orders (created_at, status)');
EXPLAIN SELECT * FROM orders WHERE status = 'PAID' ORDER BY created_at DESC;
```

### 2. 测试部分索引

```sql
-- 只对活跃订单建立索引
SELECT hypopg_create_index('CREATE INDEX ON orders (created_at) WHERE status = ''ACTIVE''');

EXPLAIN SELECT * FROM orders WHERE status = 'ACTIVE' AND created_at > '2026-01-01';
-- 预期：Index Scan（使用部分索引）

EXPLAIN SELECT * FROM orders WHERE status = 'PAID' AND created_at > '2026-01-01';
-- 预期：Seq Scan（部分索引不匹配条件）
```

### 3. 测试覆盖索引（INCLUDE）

```sql
-- 测试 Index-Only Scan 是否可行
SELECT hypopg_create_index(
    'CREATE INDEX ON orders (user_id) INCLUDE (total_amount, status)'
);

EXPLAIN SELECT user_id, total_amount, status FROM orders WHERE user_id = 42;
-- 如果显示 "Index Only Scan"，说明覆盖索引有效
```

### 4. 测试多种索引类型

```sql
-- B-tree（默认）
SELECT hypopg_create_index('CREATE INDEX ON products (price)');

-- Hash 索引
SELECT hypopg_create_index('CREATE INDEX ON products USING hash (sku)');

-- GiST 索引（全文搜索/地理）
SELECT hypopg_create_index('CREATE INDEX ON documents USING gist (content_vector)');

-- BRIN 索引（时序数据）
SELECT hypopg_create_index('CREATE INDEX ON logs USING brin (created_at)');
```

## 实战工作流

### 生产环境索引优化流程

```sql
-- Step 1: 找出慢查询
-- 使用 pg_stat_statements 或 slow query log

-- Step 2: 用 EXPLAIN (analyze, buffers) 分析当前计划
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM orders 
WHERE status = 'PENDING' AND created_at < now() - interval '7 days';

-- Step 3: 创建假设索引测试
SELECT hypopg_create_index(
    'CREATE INDEX ON orders (status, created_at) WHERE status IN (''PENDING'', ''PROCESSING'')'
);

-- Step 4: 验证计划变化
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders 
WHERE status = 'PENDING' AND created_at < now() - interval '7 days';

-- Step 5: 确认有效后，创建真实索引
-- 注意：hypopg 的假设索引不能直接转为真实索引，需手动执行 CREATE INDEX
CREATE INDEX CONCURRENTLY idx_orders_status_created_at 
ON orders (status, created_at) 
WHERE status IN ('PENDING', 'PROCESSING');

-- Step 6: 清理假设索引
SELECT hypopg_reset();
```

### 结合 auto_explain 使用

```ini
# postgresql.conf
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = '1s'     # 记录超过 1 秒的查询
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_nested_statements = on
```

```sql
-- 从日志中发现慢查询后，用 HypoPG 测试索引方案
SELECT hypopg_create_index('CREATE INDEX ON slow_table (problem_column)');
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;  -- 看假设索引后的成本变化
```

## 注意事项与限制

### 已知限制

| 限制 | 说明 |
|------|------|
| **仅影响 EXPLAIN** | 假设索引不影响实际查询执行，只影响计划器决策展示 |
| **不支持 UNIQUE** | 假设索引不能是唯一索引（无实际数据，无法验证唯一性） |
| **不支持部分类型的 GiST/GIN** | 某些操作符类的 GiST/GIN 索引无法创建假设索引 |
| **会话级生命周期** | 假设索引只存在于当前会话，断开连接后自动清除 |
| **不消耗实际磁盘** | 但会消耗少量内存存储假设索引的元数据 |
| **不支持 EXCLUDE** | 排他约束索引无法创建假设版本 |

### 使用建议

1. **先 reset 再测试**：每次测试不同方案前执行 `hypopg_reset()`，避免旧假设干扰
2. **结合真实 EXPLAIN ANALYZE**：假设索引只能展示计划器预测，实际查询效果需用真实索引验证
3. **关注成本估算**：比较 `Seq Scan` 和 `Index Scan` 的 `cost` 差异，差异越大索引价值越高
4. **多列顺序实验**：复合索引的列顺序对查询性能影响巨大，HypoPG 可以快速试验各种组合
5. **不要替代真实测试**：创建真实索引后仍需用 `EXPLAIN (ANALYZE)` 验证实际性能提升

### 工作流总结

```
发现慢查询 → HypoPG 测试假设索引 → EXPLAIN 确认计划变化 → 
创建真实索引 → 验证效果 → 删除假设索引
```

## 参考

- [HypoPG GitHub](https://github.com/HypoPG/hypopg)
- [PostgreSQL Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
- [pg_stat_statements 慢查询日志](https://www.postgresql.org/docs/current/pgstatstatements.html)
