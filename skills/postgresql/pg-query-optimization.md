---
name: pg-query-optimization
description: PostgreSQL 执行计划深度解析与 SQL 优化实战技巧
tags: [postgresql, query, optimization, explain, index]
---

## EXPLAIN 执行计划深度解析

### 输出格式与关键指标

```sql
EXPLAIN (ANALYZE, BUFFERS, COSTS, FORMAT TEXT) 
SELECT * FROM orders WHERE status = 'PAID' AND create_time > '2025-01-01';
```

关键阅读指标：

| 指标 | 含义 | 诊断 |
|------|------|------|
| `cost=0.00..431.00` | 启动成本..总成本 | 是否偏离预期 |
| `rows=100` | 预估行数 | 与实际 `rows=50000` 对比看偏差 |
| `width=42` | 行平均宽度 | 宽表可能需垂直拆分 |
| `actual time=0.025..2.345` | 实际启动..完成时间 | 瓶颈点所在 |
| `loops=3` | 循环执行次数 | Nested Loop 中过高需警惕 |
| `buffers: shared hit=123 read=5` | 缓存命中/磁盘读取 | read 过多说明内存不足 |

### 常见扫描方式

```
Seq Scan on orders (cost=0.00..9876.00 rows=500000 width=42)
  Filter: ((status)::text = 'PAID'::text)
  -- × 全表扫描，大表性能灾难
```

```
Index Scan using idx_orders_status on orders (cost=0.29..150.00 rows=1000 width=42)
  Index Cond: ((status)::text = 'PAID'::text)
  -- ✓ 索引扫描，高效
```

```
Bitmap Heap Scan on orders (cost=50.00..200.00 rows=1000 width=42)
  Recheck Cond: ((status)::text = 'PAID'::text)
  -> Bitmap Index Scan on idx_orders_status (cost=0.00..49.75 rows=1000 width=0)
     -- ✓ 位图扫描，适合返回行占比较大时
```

### JOIN 策略

| 策略 | 触发条件 | 优化建议 |
|------|----------|----------|
| **Nested Loop** | 小表 join 大表，内表有索引 | ✓ 推荐，内表列加索引 |
| **Hash Join** | 两表都较大，无合适索引 | ✓ 适合等值 join |
| **Merge Join** | 两表已排序 | 需预先排序，较少用 |

```sql
-- Nested Loop (推荐)
EXPLAIN SELECT * FROM users u JOIN orders o ON u.id = o.user_id 
WHERE u.id = 100;
-- 输出：Nested Loop → 内层 Index Scan on orders (user_id索引)

-- Hash Join
EXPLAIN SELECT * FROM users u JOIN orders o ON u.id = o.user_id;
-- 输出：Hash Join → Hash (全表扫描) → Seq Scan
```

## 索引优化实战

### 复合索引列序指南

```sql
-- 等值条件列在前，范围条件列在后
CREATE INDEX idx_orders_status_create ON orders(status, create_time);
-- ✓ WHERE status='PAID' AND create_time > '2025-01-01' 用到索引

-- 错误示例
CREATE INDEX idx_orders_create_status ON orders(create_time, status);
-- × 范围查询在前，后面的 status 索引失效
```

### 覆盖索引（Index Only Scan）

```sql
-- 查询只需要 (status, amount, create_time)，把它们都包含在索引中
CREATE INDEX idx_orders_covering ON orders(status, amount, create_time);
-- 执行计划显示 Index Only Scan，无需回表
```

### 部分索引（Partial Index）

```sql
-- 只索引活跃订单（占全部数据的 10%）
CREATE INDEX idx_orders_active ON orders(create_time DESC) 
WHERE status NOT IN ('CANCELLED', 'REFUNDED');

-- 查询自动使用部分索引
SELECT * FROM orders WHERE status = 'PAID' ORDER BY create_time DESC;
```

### 函数索引

```sql
-- 手机号后四位查询
CREATE INDEX idx_user_phone_last4 ON users((SUBSTRING(phone FROM 8)));

-- 查询
SELECT * FROM users WHERE SUBSTRING(phone FROM 8) = '1234';
```

## 慢查询优化典型场景

### 场景 1：OFFSET 深分页

```sql
-- ❌ 传统分页（越往后越慢）
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 100000;

-- ✅ Keyset Pagination（游标分页）
SELECT * FROM orders WHERE id > 100000 ORDER BY id LIMIT 20;
-- 或带复合条件
SELECT * FROM orders 
WHERE (create_time, id) > ('2025-06-01', 100000) 
ORDER BY create_time, id LIMIT 20;
```

### 场景 2：JOIN 滥用导致性能差

```sql
-- ❌ 循环查询 N+1
for (Order order : orderList) {
    User user = userMapper.selectById(order.getUserId()); // N 次查询
}

-- ✅ 批量 IN 查询
List<Long> userIds = orderList.stream().map(Order::getUserId).collect(toList());
List<User> users = userMapper.selectBatchIds(userIds); // 1 次查询
Map<Long, User> userMap = users.stream().collect(toMap(User::getId, u -> u));
```

### 场景 3：函数包裹索引列

```sql
-- ❌ 函数导致索引失效
SELECT * FROM orders WHERE DATE(create_time) = '2025-06-01';

-- ✅ 范围查询，索引可用
SELECT * FROM orders 
WHERE create_time >= '2025-06-01' AND create_time < '2025-06-02';
```

### 场景 4：多表关联 OR 条件

```sql
-- ❌ OR 可能使索引失效
SELECT * FROM orders WHERE status = 'PAID' OR user_id = 100;

-- ✅ UNION ALL 分别走索引
SELECT * FROM orders WHERE status = 'PAID'
UNION ALL
SELECT * FROM orders WHERE user_id = 100 AND status <> 'PAID';
```

## 实战工具

### pg_stat_statements 查询 TOP 慢 SQL

```sql
-- 1. 启用 pg_stat_statements（需要重启）
-- postgresql.conf: shared_preload_libraries = 'pg_stat_statements'

-- 2. 查看 TOP 10 消耗最高的查询
SELECT queryid, query,
       calls, total_exec_time / calls AS avg_ms,
       rows, shared_blks_hit, shared_blks_read
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- 3. 重置统计
SELECT pg_stat_statements_reset();
```

### 索引使用情况诊断

```sql
-- 查看未使用的索引
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND schemaname NOT IN ('pg_catalog');

-- 查看缺失索引的查询
-- 结合 pg_stat_statements 中 seq_scan 高的表
SELECT relname, seq_scan, seq_tup_read, idx_scan
FROM pg_stat_user_tables
WHERE seq_scan > 1000 AND idx_scan = 0;
```

## 注意事项

1. **VACUUM 策略**：高频更新表设置 `autovacuum_vacuum_scale_factor=0.01` 更频繁触发
2. **work_mem 设置**：排序/哈希操作的内存，不宜全局设太大，可用 `SET LOCAL work_mem='64MB'` 为特定大查询调整
3. **JOIN 顺序**：PostgreSQL 查询优化器通常比人工选择更好，不要轻易加 `join_collapse_limit=1` 强制顺序
4. **analyze 时效性**：大表数据变更超过 10% 时执行 `ANALYZE` 更新统计信息
5. **parallel 配置**：`max_parallel_workers_per_gather=2`，OLTP 场景不宜开太高
