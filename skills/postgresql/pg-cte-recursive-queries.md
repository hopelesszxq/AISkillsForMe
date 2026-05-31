---
name: pg-cte-recursive-queries
description: PostgreSQL CTE（公用表表达式）与递归查询高级用法：WITH、递归 CTE、搜索深度、图遍历
tags: [postgresql, sql, cte, recursive, query-optimization, graph]
---

## 概述

PostgreSQL 的 CTE（Common Table Expression，公用表表达式）通过 `WITH` 子句实现。相比子查询，CTE 具有更好的可读性、可复用性，并支持 **递归查询**（WITH RECURSIVE），适用于树形结构、图遍历、层次数据等场景。

## 1. CTE 基础

### 1.1 基本语法

```sql
WITH cte_name AS (
    SELECT ...  -- CTE 查询体
)
SELECT * FROM cte_name;  -- 主查询引用 CTE
```

### 1.2 多 CTE 串联

```sql
WITH 
sales_summary AS (
    SELECT 
        department_id,
        SUM(amount) AS total_sales
    FROM orders
    WHERE order_date >= '2026-01-01'
    GROUP BY department_id
),
ranked_departments AS (
    SELECT 
        department_id,
        total_sales,
        RANK() OVER (ORDER BY total_sales DESC) AS rank
    FROM sales_summary
)
SELECT d.name, rs.total_sales, rs.rank
FROM departments d
JOIN ranked_departments rs USING (department_id)
WHERE rs.rank <= 5
ORDER BY rs.rank;
```

### 1.3 CTE vs 子查询性能

**⚠️ 关键区别**：PostgreSQL 默认将 CTE 作为 **优化边界**（Optimization Fence），即 CTE 被物化为临时结果集，阻止了谓词下推和 join 优化。PG12+ 可以通过 `NOT MATERIALIZED` 强制内联：

```sql
-- 默认 MATERIALIZED：CTE 结果被独立计算和存储
WITH expensive_cte AS (
    SELECT * FROM large_table WHERE status = 'active'
)
SELECT * FROM expensive_cte WHERE created_at > '2026-01-01';

-- NOT MATERIALIZED：CTE 被内联到主查询（优化器可以传递过滤条件）
WITH expensive_cte AS NOT MATERIALIZED (
    SELECT * FROM large_table WHERE status = 'active'
)
SELECT * FROM expensive_cte WHERE created_at > '2026-01-01';
```

**选择建议**：
- CTE 被引用多次：使用默认 `MATERIALIZED`（避免重复计算）
- CTE 只引用一次：使用 `NOT MATERIALIZED`（允许谓词下推）

## 2. 递归 CTE

### 2.1 语法结构

```sql
WITH RECURSIVE cte_name AS (
    -- 1. 非递归项（锚点查询）：基准行
    SELECT ...
    UNION ALL
    -- 2. 递归项：引用自身，在上一次结果基础上扩展
    SELECT ...
    FROM cte_name
    WHERE ...
)
SELECT * FROM cte_name;
```

执行流程：
1. 执行锚点查询，得到初始结果集
2. 将初始结果集作为输入，执行递归查询
3. 递归查询结果作为下一轮输入
4. 当递归查询返回空时停止

### 2.2 组织树结构查询

```sql
-- 表结构
CREATE TABLE org_chart (
    id          SERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    parent_id   INT REFERENCES org_chart(id),
    level       INT NOT NULL DEFAULT 0
);

INSERT INTO org_chart VALUES
(1, 'CEO',          NULL, 0),
(2, 'VP Engineering', 1, 1),
(3, 'VP Sales',       1, 1),
(4, 'Dev Lead',       2, 2),
(5, 'Senior Dev',     4, 3),
(6, 'Junior Dev',     4, 3),
(7, 'Sales Lead',     3, 2);

-- 递归查询：从 CEO 向下展开整个组织树
WITH RECURSIVE org_tree AS (
    -- 锚点：根节点
    SELECT 
        id, name, parent_id, level,
        ARRAY[id] AS path,
        0 AS depth
    FROM org_chart
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- 递归：找到子节点
    SELECT 
        e.id, e.name, e.parent_id, e.level,
        path || e.id,
        ot.depth + 1
    FROM org_chart e
    JOIN org_tree ot ON e.parent_id = ot.id
)
SELECT 
    repeat('  ', depth) || name AS tree_view,
    id,
    parent_id,
    depth,
    path::TEXT AS full_path
FROM org_tree
ORDER BY path;
```

结果：
```
tree_view        | id | parent_id | depth | full_path
-----------------+----+-----------+-------+-----------
CEO              |  1 |           |     0 | {1}
  VP Engineering |  2 |         1 |     1 | {1,2}
    Dev Lead     |  4 |         2 |     2 | {1,2,4}
      Senior Dev |  5 |         4 |     3 | {1,2,4,5}
      Junior Dev |  6 |         4 |     3 | {1,2,4,6}
  VP Sales       |  3 |         1 |     1 | {1,3}
    Sales Lead   |  7 |         3 |     2 | {1,3,7}
```

### 2.3 向上追溯（找祖先路径）

```sql
WITH RECURSIVE ancestors AS (
    -- 从目标节点开始
    SELECT id, name, parent_id, 0 AS depth
    FROM org_chart
    WHERE id = 6  -- Junior Dev
    
    UNION ALL
    
    SELECT e.id, e.name, e.parent_id, a.depth + 1
    FROM org_chart e
    JOIN ancestors a ON e.id = a.parent_id
)
SELECT * FROM ancestors ORDER BY depth DESC;

-- 结果
-- id=1 CEO, id=2 VP Engineering, id=4 Dev Lead, id=6 Junior Dev
```

### 2.4 物料清单（BOM）示例

```sql
CREATE TABLE bom (
    part_id     INT PRIMARY KEY,
    part_name   TEXT NOT NULL
);

CREATE TABLE bom_structure (
    parent_part_id  INT REFERENCES bom(part_id),
    child_part_id   INT REFERENCES bom(part_id),
    quantity        INT NOT NULL,
    PRIMARY KEY (parent_part_id, child_part_id)
);

-- 递归计算总用量
WITH RECURSIVE bom_explosion AS (
    -- 锚点：顶级产品
    SELECT 
        part_id,
        part_name,
        1 AS total_qty,
        ARRAY[part_id] AS visited
    FROM bom
    WHERE part_id = 100  -- 产品 A
    
    UNION ALL
    
    SELECT 
        b.part_id,
        b.part_name,
        be.total_qty * bs.quantity,
        visited || b.part_id
    FROM bom_structure bs
    JOIN bom b ON b.part_id = bs.child_part_id
    JOIN bom_explosion be ON be.part_id = bs.parent_part_id
    WHERE NOT (b.part_id = ANY(visited))  -- 防止循环
)
SELECT part_id, part_name, SUM(total_qty) AS total_required
FROM bom_explosion
WHERE part_id <> 100  -- 排除顶层产品本身
GROUP BY part_id, part_name
ORDER BY part_id;
```

## 3. 图遍历与路径检测

### 3.1 有向图路径搜索

```sql
CREATE TABLE graph_edges (
    from_node   INT NOT NULL,
    to_node     INT NOT NULL,
    weight      NUMERIC,
    PRIMARY KEY (from_node, to_node)
);

-- 查找所有从节点 1 到节点 6 的路径
WITH RECURSIVE paths AS (
    -- 起点
    SELECT 
        from_node AS current,
        ARRAY[from_node, to_node] AS path,
        weight AS total_weight
    FROM graph_edges
    WHERE from_node = 1
    
    UNION ALL
    
    SELECT 
        e.to_node,
        p.path || e.to_node,
        p.total_weight + e.weight
    FROM graph_edges e
    JOIN paths p ON e.from_node = p.current
    WHERE NOT (e.to_node = ANY(p.path))  -- 避免环路
)
SELECT * FROM paths
WHERE current = 6
ORDER BY total_weight;
```

### 3.2 检测环路

```sql
-- 如果存在环路，递归会陷入无限循环
-- 必须使用 path 数组检测环路
WITH RECURSIVE detect_cycle AS (
    SELECT 
        from_node, to_node,
        ARRAY[from_node, to_node] AS path,
        0 AS cycle_detected
    FROM graph_edges
    WHERE from_node = 1
    
    UNION ALL
    
    SELECT 
        e.to_node,
        p.path || e.to_node,
        CASE WHEN e.to_node = ANY(p.path) THEN 1 ELSE 0 END
    FROM graph_edges e
    JOIN detect_cycle p ON e.from_node = p.to_node
    WHERE cycle_detected = 0
)
SELECT * FROM detect_cycle WHERE cycle_detected = 1;
```

## 4. 递归 CTE 优化

### 4.1 添加搜索深度限制

```sql
WITH RECURSIVE org_tree AS (
    -- 锚点
    SELECT id, name, parent_id, 0 AS depth
    FROM org_chart WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT e.id, e.name, e.parent_id, ot.depth + 1
    FROM org_chart e
    JOIN org_tree ot ON e.parent_id = ot.id
    WHERE ot.depth < 5  -- 限制最大深度
)
SELECT * FROM org_tree;
```

### 4.2 使用索引加速

```sql
-- 递归 CTE 的性能取决于父-子连接的效率
-- 必须为 parent_id 创建索引
CREATE INDEX idx_org_chart_parent ON org_chart(parent_id);

-- 图遍历同样需要索引
CREATE INDEX idx_graph_from ON graph_edges(from_node);
CREATE INDEX idx_graph_to ON graph_edges(to_node);
```

### 4.3 使用 SEARCH 子句（PG14+）

PostgreSQL 14+ 支持 `SEARCH` 子句，简化排序控制：

```sql
-- PG14+ 语法：深度优先搜索
WITH RECURSIVE org_tree AS (
    SELECT id, name, parent_id
    FROM org_chart WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT e.id, e.name, e.parent_id
    FROM org_chart e
    JOIN org_tree ot ON e.parent_id = ot.id
)
SEARCH DEPTH FIRST BY name SET order_col
SELECT id, name, parent_id
FROM org_tree
ORDER BY order_col;

-- PG14+ 语法：广度优先搜索
SEARCH BREADTH FIRST BY name SET order_col
```

## 5. 注意事项

### 5.1 递归深度限制

PostgreSQL 默认递归深度无限制，但为防止失控，建议设置：

```sql
-- session 级别限制
SET session recursive_worktable_factor = 100;  -- 默认 100

-- 或者强制加深度限制
max_recursive_depth = 1000  -- PG15+ 配置参数
```

### 5.2 CTE 作为 DML 的目标

CTE 支持 `INSERT/UPDATE/DELETE` 返回数据（数据修改 CTE）：

```sql
WITH deleted_old AS (
    DELETE FROM orders
    WHERE created_at < '2024-01-01'
    RETURNING *
),
archived AS (
    INSERT INTO orders_archive SELECT * FROM deleted_old
    RETURNING *
)
SELECT COUNT(*) FROM archived;
```

### 5.3 善用 EXPLAIN ANALYZE

递归 CTE 的性能瓶颈通常出现在递归步中，使用 `EXPLAIN ANALYZE` 查看每轮迭代的耗时：

```sql
EXPLAIN (ANALYZE, BUFFERS)
WITH RECURSIVE t AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM t WHERE n < 10000
)
SELECT * FROM t;
```

### 5.4 递归 CTE 替代方案

对于简单层次查询，可考虑：
- `ltree` 扩展：专门用于树形数据，性能更好
- 嵌套集模型（Nested Set）：适用于读多写少的场景
- 物化路径（Materialized Path）：简单直接
