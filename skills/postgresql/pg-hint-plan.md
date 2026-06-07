---
name: pg-hint-plan
description: PostgreSQL pg_hint_plan 扩展：强制查询计划、扫描方式与连接顺序的手动优化
tags: [postgresql, pg_hint_plan, query-plan, optimizer, hint, performance]
---

## 概述

`pg_hint_plan` 是 PostgreSQL 第三方扩展，允许在 SQL 中嵌入**优化器提示（Hints）**，强制指定扫描方式、连接方法、连接顺序等。当 PostgreSQL 优化器因统计信息不准确或复杂查询产生次优计划时，这是最直接的解决方案。

> 官方文档：https://ossc-db.github.io/pg_hint_plan/
> 支持版本：PostgreSQL 12 ~ 18

## 安装

```sql
-- 编译安装
-- 下载源码后：
-- make
-- make install

-- 加载扩展
CREATE EXTENSION pg_hint_plan;

-- 查看版本
SELECT * FROM pg_hint_plan_info();
```

```ini
# postgresql.conf 配置
shared_preload_libraries = 'pg_hint_plan'
pg_hint_plan.enable_hint = on          # 全局启用 hint
pg_hint_plan.enable_hint_tables = on   # 允许 hint 表
pg_hint_plan.parse_messages = warning  # hint 解析失败时的日志级别
pg_hint_plan.debug_level = warning     # hint 调试输出
```

## Hint 语法

### 基本格式

Hint 写在 SQL 语句开头的**块注释**中，以 `/*+` 开头、`*/` 结尾：

```sql
/*+
 *   Hint 内容
 */
SELECT ...
```

### 扫描方式 Hints

| Hint | 含义 | 示例 |
|------|------|------|
| `SeqScan(table)` | 强制全表扫描 | `/*+ SeqScan(orders) */` |
| `NoSeqScan(table)` | 禁止全表扫描 | `/*+ NoSeqScan(orders) */` |
| `IndexScan(table index)` | 强制索引扫描 | `/*+ IndexScan(orders idx_orders_status) */` |
| `IndexOnlyScan(table index)` | 仅索引扫描 | `/*+ IndexOnlyScan(orders idx_orders_date) */` |
| `BitmapScan(table index)` | 位图扫描 | `/*+ BitmapScan(orders idx_orders_status) */` |
| `TidScan(table)` | TID 扫描 | `/*+ TidScan(orders) */` |
| `NoBitmapScan(table)` | 禁止位图扫描 | `/*+ NoBitmapScan(orders) */` |

### 连接方法 Hints

| Hint | 含义 | 示例 |
|------|------|------|
| `NestLoop(table1 table2)` | 嵌套循环连接 | `/*+ NestLoop(o u) */` |
| `HashJoin(table1 table2)` | 哈希连接 | `/*+ HashJoin(o u) */` |
| `MergeJoin(table1 table2)` | 排序合并连接 | `/*+ MergeJoin(o u) */` |
| `NoNestLoop(table1 table2)` | 禁止嵌套循环 | `/*+ NoNestLoop(o u) */` |

### 连接顺序 Hints

| Hint | 含义 | 示例 |
|------|------|------|
| `Leading(table1 table2 ...)` | 指定连接顺序 | `/*+ Leading(o u i) */` |
| `Leading((table1 table2) table3)` | 指定分组连接顺序 | `/*+ Leading((o u) i) */` |

### 其他 Hints

| Hint | 含义 | 示例 |
|------|------|------|
| `Rows(table1 table2 #num)` | 强制预估行数 | `/*+ Rows(o u #1000) */` |
| `Parallel(table #workers)` | 强制并行度 | `/*+ Parallel(o 4) */` |
| `Set(parameter value)` | 设置 GUC 参数 | `/*+ Set(enable_hashjoin off) */` |

## 实战案例

### 案例1：修复优化器选择的错误扫描方式

优化器对 `status='PAID'` 选择性过于乐观，使用了全表扫描：

```sql
-- 原始查询（Seq Scan，耗时 12s）
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE status = 'PAID'
  AND create_time >= '2025-01-01'
ORDER BY create_time DESC
LIMIT 100;

-- 优化后：强制使用复合索引
/*+
 *  IndexScan(orders idx_orders_status_date)
 */
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE status = 'PAID'
  AND create_time >= '2025-01-01'
ORDER BY create_time DESC
LIMIT 100;
```

### 案例2：修复多表连接中错误的连接方法

优化器错误地选择了 Nested Loop（大量循环），应改为 Hash Join：

```sql
-- 原始执行计划：Nested Loop (35ms → 当数据增长到 1M 行时变为 28s)
SELECT o.id, u.name, i.product_name
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN order_items i ON o.id = i.order_id
WHERE o.status = 'PAID';

-- 强制 Hash Join + 指定连接顺序
/*+
 *  HashJoin(o u)
 *  HashJoin(o i)
 *  Leading(o u i)
 */
EXPLAIN ANALYZE
SELECT o.id, u.name, i.product_name
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN order_items i ON o.id = i.order_id
WHERE o.status = 'PAID';
```

### 案例3：解决"参数嗅探"导致的计划不稳定

同一个查询因不同输入参数值导致优化器选择不同的计划，某些参数下性能极差：

```sql
-- 问题：status='PAID'(低基数) 和 status='PENDING'(高基数) 需要不同计划
-- 方案1：使用通用计划（PostgreSQL 12+）
/*+
 *  IndexScan(orders idx_orders_status)
 */
PREPARE find_orders(text) AS
SELECT * FROM orders
WHERE status = $1
ORDER BY create_time DESC
LIMIT 100;

-- 方案2：CROSS JOIN + ROWS hint 补偿统计偏差
/*+
 *  Rows(orders #10000)
 *  Leading(orders)
 */
SELECT * FROM orders
WHERE status = 'PENDING'
ORDER BY create_time DESC
LIMIT 100;
```

### 案例4：优化多表聚合查询

```sql
-- 报告查询：按用户统计订单总额
EXPLAIN ANALYZE
/*+
 *  HashJoin(u o)
 *  Leading(u o)
 *  HashJoin(u o i)
 *  Rows(u o #5000)
 */
SELECT
    u.id,
    u.name,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(i.amount), 0) AS total_amount
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
LEFT JOIN order_items i ON i.order_id = o.id
WHERE u.created_at >= '2025-01-01'
GROUP BY u.id, u.name
ORDER BY total_amount DESC
LIMIT 20;
```

## 通过 Hint 表统一管理

对于不方便修改 SQL 的生产环境（ORM 生成的 SQL、第三方工具），可通过 hint 表进行管理：

```sql
-- 创建 hint 表
CREATE TABLE hint_plan.hints (
    id SERIAL PRIMARY KEY,
    norm_query_hash VARCHAR(32),  -- 归一化 SQL 的 MD5
    hint_text TEXT,               -- hint 内容
    enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- 注册 hint（自动匹配所有相同结构的 SQL）
INSERT INTO hint_plan.hints (norm_query_hash, hint_text)
VALUES (
    md5('SELECT * FROM orders WHERE status = $1 AND create_time >= $2 ORDER BY create_time DESC LIMIT $3'),
    'IndexScan(orders idx_orders_status_date)'
);

-- 查看 hint 命中情况
SELECT * FROM pg_hint_plan_hints_usage;
```

### 获取 SQL 的归一化哈希

```sql
-- 获取当前查询的归一化 SQL
SELECT query_hash, query
FROM pg_stat_statements
WHERE query LIKE '%orders%'
ORDER BY total_exec_time DESC
LIMIT 5;
```

## 监控 Hint 是否生效

```sql
-- 查看 hint 解析结果
EXPLAIN (ANALYZE, HINTS true)
/*+ IndexScan(orders idx_orders_status_date) */
SELECT * FROM orders WHERE status = 'PAID';

-- 输出中会显示 hint 是否被应用
-- "Hint Status: Used" 表示生效
-- "Hint Status: Not used" 表示未生效（检查拼写或表别名是否匹配）
```

## 注意事项

- **表别名必须匹配**：Hint 中使用的是 SQL 中的表名/别名，而非实际表名。如果 SQL 使用了别名，hint 中也必须用别名
- **复合索引全名**：索引名必须是全名，不支持模糊匹配
- **不覆盖统计信息更新**：添加 hint 后仍需 `ANALYZE` 更新统计信息，让优化器做出更好的无 hint 计划
- **版本升级后检查**：PG 大版本升级后优化器行为可能变化，原 hint 可能不再需要或失效
- **不要过度使用**：只在明确了优化器选择错误时才使用 hint，否则会阻碍优化器后续的改进
- **测试环境验证**：所有 hint 必须在与生产环境数据量级相同的测试环境中验证
- **备库兼容性**：hint 在主库生效，备库只读查询同样可以使用
- **ORM 场景**：使用 JPA/Hibernate 时，可通过 `@Query(nativeQuery=true)` 或 `@QueryHints` 添加 hint；MyBatis-Plus 可在 XML 中直接写 hint 注释
