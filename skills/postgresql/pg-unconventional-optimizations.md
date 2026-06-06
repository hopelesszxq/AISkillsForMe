---
name: pg-unconventional-optimizations
description: PostgreSQL 非传统优化技巧：check constraint 排除、函数索引降基数、Hash 索引唯一约束、虚拟生成列
tags: [postgresql, performance, indexing, optimization, hash-index, constraint-exclusion]
---

## 要点

常规优化三板斧（加索引、改查询、反范式）之外的几种"创造性"优化方案，可在特定场景大幅提升性能并节约存储。

### 1. constraint_exclusion — 利用 CHECK 约束跳过全表扫描

当查询条件与表上的 CHECK 约束矛盾时，PostgreSQL 默认仍会扫描全表。开启 `constraint_exclusion` 可让优化器识别并跳过无需扫描的表。

```sql
-- 默认只对分区表生效
SET constraint_exclusion TO 'on';  -- 对所有表生效

-- 示例：plan 字段有 CHECK(plan IN ('free', 'pro'))
-- 查询 plan = 'Pro'（大写，不可能匹配）
EXPLAIN ANALYZE SELECT * FROM users WHERE plan = 'Pro';
-- 开启前：Seq Scan on users (rows=0, 但扫了全表)
-- 开启后：Result (One-Time Filter: false) — 0.008ms 完成
```

**适用场景**：BI/报表环境，用户会写 ad-hoc 查询，容易因大小写/拼写错误产生不可能匹配的条件。

### 2. 函数索引降基数 — 更小更快的索引

按 datetime 字段分组做日报时，用全精度 `timestamptz` 索引浪费空间。改为函数索引仅索引日期部分，大小可缩小 3 倍以上。

```sql
-- 原始：214MB B-tree
CREATE INDEX sale_sold_at_ix ON sale (sold_at);

-- 优化：66MB 函数索引（利用 B-tree deduplication）
CREATE INDEX sale_sold_at_date_ix ON sale (
  (date_trunc('day', sold_at AT TIME ZONE 'UTC'))::date
);

-- 查询必须使用完全相同的表达式才能命中索引
SELECT date_trunc('day', sold_at AT TIME ZONE 'UTC'), SUM(charged)
FROM sale
WHERE date_trunc('day', sold_at AT TIME ZONE 'UTC')::date
      BETWEEN '2025-01-01' AND '2025-01-31'
GROUP BY 1;
```

### 3. 虚拟生成列（PG18+）— 解决函数索引的"纪律问题"

函数索引的脆弱性在于查询必须使用完全相同的表达式。PG18 引入虚拟生成列，将表达式定义为列，确保应用层始终使用正确的表达式。

```sql
-- PG 18+ 添加虚拟生成列
ALTER TABLE sale ADD sold_at_date DATE
GENERATED ALWAYS AS (date_trunc('day', sold_at AT TIME ZONE 'UTC'));

-- 直接使用虚拟列查询，自动命中函数索引
SELECT sold_at_date, SUM(charged)
FROM sale
WHERE sold_at_date BETWEEN '2025-01-01' AND '2025-01-31'
GROUP BY 1;
```

> **注意**：PG18 不支持在虚拟生成列上直接创建索引（`ERROR: indexes on virtual generated columns are not supported`），仍需在表达式上建索引。PG19 可能会支持。

### 4. Hash 索引实现唯一约束 — 大文本字段节省 5 倍空间

PostgreSQL 不支持 `UNIQUE HASH INDEX`，但可用**排除约束（EXCLUSION CONSTRAINT）**等效实现。

```sql
-- 大文本 URL 字段的 B-tree 唯一索引：154MB
CREATE UNIQUE INDEX urls_url_unique_ix ON urls (url);

-- 用排除约束 + Hash 索引替代：32MB（5 倍空间节省）
ALTER TABLE urls ADD CONSTRAINT urls_url_unique_hash
EXCLUDE USING HASH (url WITH =);

-- 查询依然可以使用该索引
EXPLAIN ANALYZE SELECT * FROM urls WHERE url = 'https://example.com';
-- Index Scan using urls_url_unique_hash
```

#### 限制

| 场景 | 是否支持 |
|------|---------|
| 外键引用 `REFERENCES urls(url)` | ❌ 不支持 |
| `INSERT ... ON CONFLICT (url) DO NOTHING` | ❌ 需要 `ON CONFLICT ON CONSTRAINT urls_url_unique_hash` |
| `INSERT ... ON CONFLICT ... DO UPDATE` | ❌ 不支持（可用 `MERGE` 替代） |
| `MERGE INTO ... WHEN MATCHED THEN UPDATE` | ✅ 支持 |

```sql
-- 替代方案：用 MERGE 实现 upsert
MERGE INTO urls t
USING (VALUES (1000004, 'https://example.com')) AS s(id, url)
ON t.url = s.url
WHEN MATCHED THEN UPDATE SET id = s.id
WHEN NOT MATCHED THEN INSERT (id, url) VALUES (s.id, s.url);
```

## 代码示例

```sql
-- 完整示例：constraint_exclusion 测试
CREATE TABLE users (
    id      INT PRIMARY KEY,
    username TEXT NOT NULL,
    plan    TEXT NOT NULL,
    CONSTRAINT plan_check CHECK (plan IN ('free', 'pro'))
);

INSERT INTO users
SELECT n, 'user' || n, (ARRAY['free','pro'])[ceil(random()*2)]
FROM generate_series(1, 100000) AS t(n);

-- 开启前：全表扫描
EXPLAIN ANALYZE SELECT * FROM users WHERE plan = 'Pro';

-- 开启后：0ms 返回
SET constraint_exclusion TO 'on';
EXPLAIN ANALYZE SELECT * FROM users WHERE plan = 'Pro';
```

## 注意事项

1. **constraint_exclusion 有规划开销**：默认只对分区表启用（`partition` 模式）。全表开启后，简单查询的规划时间可能增加，建议仅在 BI/报表库启用。
2. **函数索引脆弱性**：查询表达式必须与索引定义完全一致（包括时间时区处理），建议配合虚拟生成列或视图使用。
3. **Hash 索引限制**：不支持唯一约束声明，需用排除约束绕过；不支持 `ON CONFLICT DO UPDATE`，需要用 `MERGE`；不能作为外键目标。
4. **空间-速度权衡**：Hash 索引只能做等值查询（`=`），不支持范围查询，适合大文本、长字符串的唯一性检查场景。
