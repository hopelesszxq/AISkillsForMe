---
name: pg-extended-statistics
description: PostgreSQL 扩展统计信息（Extended Statistics）：多列相关性优化查询计划的实战指南
tags: [postgresql, performance, query-optimization, statistics, planner]
---

## 概述

PostgreSQL 查询优化器依赖列级统计信息估算 WHERE 条件的选择率。但当多个列具有**相关性**时（例如：`city` 和 `zip_code`），独立列统计会高估选择率，导致优化器选择低效的计划。

**扩展统计信息（Extended Statistics）** 告诉优化器列之间的实际依赖关系，在多列过滤、GROUP BY、关联查询中显著提升计划质量。

## 什么时候需要扩展统计

### 典型场景

| 场景 | 问题 | 扩展统计解决方案 |
|------|------|-----------------|
| 多列过滤 | `WHERE city='北京' AND zip='100000'` — 优化器假设独立分布，行数估算严重偏差 | `dependencies` 统计 |
| 关联列 GROUP BY | `GROUP BY make, model` — 假设笛卡尔积 distinct 值，估算过高 | `ndistinct` 统计 |
| 非均匀分布 | WHERE 条件筛出的值集有偏斜 | `mcv` 统计 |

### 快速诊断：是否需要扩展统计

```sql
-- 对比实际值 vs 优化器估算
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM orders 
WHERE status = 'shipped' AND payment_method = 'credit_card';

-- 如果 rows= 与实际行数相差 10x 以上，考虑扩展统计
```

## 三种扩展统计类型

### 1. `dependencies` — 函数依赖

当列 A 的值能唯一确定列 B 的值时，称为函数依赖。

```sql
CREATE STATISTICS st_zip_city (dependencies)
ON zip_code, city FROM addresses;

-- 优化器查询依赖关系
ANALYZE addresses;
```

**何时有效**：一对多关系（如 `zip_code → city`、`country → region`）。

### 2. `ndistinct` — 多列不同值计数

当 GROUP BY 多列时，优化器需要知道列组合的唯一值数量。

```sql
-- 无扩展统计：优化器猜测 n_distinct = n_col1 * n_col2（通常过高）
-- 有扩展统计：精确值
CREATE STATISTICS st_orders_status_nd (ndistinct)
ON status, payment_method FROM orders;

ANALYZE orders;

-- 查看估算结果
SELECT stxname, stxndistinct 
FROM pg_statistic_ext 
WHERE stxname = 'st_orders_status_nd';
```

### 3. `mcv` — 多列最常见值列表

当多列组合的分布呈偏斜时（某些组合出现频率远高于其他），`mcv` 能记录这些高频率组合。

```sql
-- 记录最常见的 (category, region, season) 组合
CREATE STATISTICS st_sales_mcv (mcv)
ON category, region, season FROM sales;

ANALYZE sales;
```

## 实战：识别需要扩展统计的列

```sql
-- 1. 找到估算偏差大的查询
SELECT queryid, query, 
       rows * 100 / (rows + 1) AS est_rows,
       actual_rows
FROM pg_stat_statements 
WHERE actual_rows > 1000 
  AND ABS(rows - actual_rows) / NULLIF(actual_rows, 0) > 5;

-- 2. 分析关联列相关性
SELECT 
  tablename,
  attname AS column1,
  'correlated_column_name' AS column2,
  correlation
FROM pg_stats 
WHERE tablename = 'orders'
  AND correlation IS NOT NULL
ORDER BY ABS(correlation) DESC;
```

## 完整示例

```sql
-- 创建测试表
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    status TEXT NOT NULL,        -- pending, paid, shipped, delivered, cancelled
    payment_method TEXT NOT NULL,-- credit_card, alipay, wechat, bank_transfer
    amount NUMERIC(10,2),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 写入带相关性的数据（不同状态下支付方式分布不同）
INSERT INTO orders (status, payment_method, amount)
SELECT 
  CASE WHEN random() < 0.4 THEN 'shipped'
       WHEN random() < 0.65 THEN 'delivered'  
       WHEN random() < 0.8 THEN 'pending'
       WHEN random() < 0.9 THEN 'paid'
       ELSE 'cancelled'
  END,
  CASE WHEN random() < 0.5 THEN 'credit_card'
       WHEN random() < 0.75 THEN 'alipay'
       ELSE 'wechat'
  END,
  random() * 1000
FROM generate_series(1, 100000);

ANALYZE orders;

-- 查看无扩展统计的计划
EXPLAIN (ANALYZE, BUFFERS)
SELECT COUNT(*) FROM orders 
WHERE status = 'cancelled' AND payment_method = 'credit_card';
-- 注意 rows= 的估算值

-- 创建扩展统计
CREATE STATISTICS st_orders_status_pay (dependencies, ndistinct, mcv)
ON status, payment_method FROM orders;

ANALYZE orders;

-- 再次查看计划
EXPLAIN (ANALYZE, BUFFERS)
SELECT COUNT(*) FROM orders 
WHERE status = 'cancelled' AND payment_method = 'credit_card';
-- 估算值应更接近实际行数
```

## 监控扩展统计

```sql
-- 查看所有已创建的扩展统计
SELECT 
  s.stxname,
  s.stxnamespace::regnamespace AS schema_name,
  s.stxkind AS stat_types,   -- d=依赖, n=ndistinct, m=mcv
  t.relname AS table_name,
  s.stxkeys                 -- 列引用编码
FROM pg_statistic_ext s
JOIN pg_class t ON t.oid = s.stxrelid;

-- 查看扩展统计的详细内容
SELECT * FROM pg_statistic_ext_data 
WHERE stxoid = 'st_orders_status_pay'::regclass;
```

## 注意事项

1. **ANALYZE 开销**：扩展统计在 `ANALYZE` 时计算，会额外消耗 CPU 和内存。对于大表，只对高频查询涉及的关键列组合创建
2. **不适用于表达式索引**：扩展统计不支持 `WHERE lower(name) = 'xxx'` 这种表达式过滤
3. **Dependencies 的限制**：函数依赖统计需要至少 1000 行采样，小表效果有限
4. **MCV 条目上限**：默认最多记录 100 个最常见值组合，可通过 `ALTER STATISTICS ... SET STATISTICS` 调大
5. **需要显式 ANALYZE**：创建统计后不会自动触发 ANALYZE，必须手动执行
6. **升级注意**：pg17+ 增强了依赖统计的精度，pg19 的 REPACK 不会重建扩展统计

```sql
-- 调大 MCV 条目数
ALTER STATISTICS st_sales_mcv SET STATISTICS 500;
ANALYZE sales;
```

## 参考

- [PostgreSQL 文档 — Extended Statistics](https://www.postgresql.org/docs/current/planner-stats.html#PLANNER-STATS-EXTENDED)
- [PostgreSQL 文档 — CREATE STATISTICS](https://www.postgresql.org/docs/current/sql-createstatistics.html)
- [PostgreSQL 19 REPACK 特性](pg19-new-features.md)
