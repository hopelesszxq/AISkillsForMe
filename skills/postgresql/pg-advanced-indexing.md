---
name: pg-advanced-indexing
description: PostgreSQL 高级索引与查询优化实战
tags: [postgresql, database, indexing, performance]
---

## 索引类型详解

| 索引类型 | 适用场景 | 示例 |
|---|---|---|
| B-tree | 默认，等值 + 范围查询 | 主键、外键、排序字段 |
| GIN | 全文检索、JSONB、数组 | `@@` 全文搜索、`@>` JSON 包含 |
| GiST | 空间数据、范围类型 | 地理坐标、日期范围排除约束 |
| BRIN | 超大表、数据天然有序 | 日志表（按时间插入） |
| Hash | 等值查询（很少用） | 长字符串等值匹配 |
| SP-GiST | 非均衡数据分布 | GIS、电话前缀搜索 |

## JSONB 索引策略

```sql
-- 创建 GIN 索引（支持 JSONB 包含操作）
CREATE INDEX idx_users_metadata ON users USING GIN (metadata);

-- 查询示例
SELECT * FROM users WHERE metadata @> '{"role": "admin"}';
SELECT * FROM users WHERE metadata -> 'address' ->> 'city' = 'Beijing';

-- 路径索引（只索引特定路径，更小更快）
CREATE INDEX idx_users_city ON users ((metadata -> 'address' ->> 'city'));

-- JSONB 存在性查询（检查 key 是否存在）
SELECT * FROM users WHERE metadata ? 'phone';
```

## 部分索引（Partial Index）

```sql
-- 只索引活跃用户（status = 'active' 的约占 10%）
CREATE INDEX idx_users_active ON users(email) WHERE status = 'active';

-- 软删除场景：只索引未删除记录
CREATE INDEX idx_orders_pending ON orders(created_at) 
WHERE deleted_at IS NULL AND status = 'pending';

-- 查询自动使用部分索引（必须匹配 WHERE 条件）
EXPLAIN ANALYZE
SELECT * FROM orders 
WHERE status = 'pending' AND deleted_at IS NULL
ORDER BY created_at LIMIT 20;
```

## 覆盖索引（INCLUDE）

```sql
-- 避免回表查询
CREATE INDEX idx_orders_user_status ON orders(user_id, status) 
INCLUDE (total_amount, created_at);

-- 以下查询只需扫描索引，无需回表
EXPLAIN ANALYZE
SELECT user_id, total_amount, created_at 
FROM orders WHERE user_id = 123 AND status = 'paid';
```

## 函数索引

```sql
-- 忽略大小写查询
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'admin@example.com';

-- 日期截断条件
CREATE INDEX idx_orders_created_date ON orders((created_at::date));
SELECT * FROM orders WHERE created_at::date = '2025-01-01';

-- 联合字段排序
CREATE INDEX idx_users_full_name ON users((last_name || ' ' || first_name));
```

## 复合索引最左前缀原则

```sql
-- 创建复合索引
CREATE INDEX idx_a_b_c ON t(a, b, c);

-- 能用到索引的查询:
-- WHERE a = 1              ✓
-- WHERE a = 1 AND b = 2    ✓
-- WHERE a = 1 AND b = 2 AND c = 3  ✓
-- WHERE a IN (1,2)         ✓ (多条件扫描)

-- 不能用到索引的查询:
-- WHERE b = 2               ✗（跳过了最左列）
-- WHERE c = 3               ✗
-- WHERE b = 2 AND c = 3    ✗

-- 排序利用索引:
-- ORDER BY a, b, c         ✓（同方向）
-- ORDER BY a DESC, b DESC  ✓
-- ORDER BY a, b DESC       ✗（方向不一致）
```

## 查询优化技巧

```sql
-- 1. 使用 LIMIT + 游标分页替代 OFFSET（Keyset Pagination）
-- ❌ 传统分页（OFFSET 越大越慢）
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 200000;

-- ✅ 游标分页（稳定性能）
SELECT * FROM orders 
WHERE id > :last_seen_id 
ORDER BY id LIMIT 20;

-- 2. 禁止隐式类型转换
-- ❌ 索引失效
SELECT * FROM users WHERE phone = 13800138000;  -- phone 是 varchar
-- ✅ 正确写法
SELECT * FROM users WHERE phone = '13800138000';

-- 3. 聚合查询优化
-- ❌ count(distinct) 精确但慢
SELECT COUNT(DISTINCT user_id) FROM orders;
-- ✅ 近似统计（HyperLogLog）
SELECT COUNT(*) FROM (
    SELECT user_id FROM orders GROUP BY user_id
) t;

-- 4. 使用物化视图加速复杂报表
CREATE MATERIALIZED VIEW mv_order_daily AS
SELECT created_at::date AS day,
       COUNT(*) AS order_count,
       SUM(total_amount) AS revenue
FROM orders
GROUP BY created_at::date
WITH DATA;

REFRESH MATERIALIZED VIEW CONCURRENTLY mv_order_daily;
```

## 注意事项

1. **索引不是越多越好**：每个索引影响写入性能（INSERT/UPDATE/DELETE 需要同步维护索引）
2. **定期维护**：`VACUUM` / `ANALYZE` 保持统计信息新鲜，auto-vacuum 默认开启
3. **查看索引使用率**：`pg_stat_user_indexes` 表可以看哪些索引从未被使用
4. **避免冗余索引**：`(a, b)` 复合索引已覆盖 `(a)` 单列索引的场景
5. **B-tree 索引对 IS NULL 有效**（PostgreSQL 8.3+），但复合索引需注意 NULL 位置
