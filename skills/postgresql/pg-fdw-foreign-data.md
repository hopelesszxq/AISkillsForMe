---
name: pg-fdw-foreign-data
description: PostgreSQL FDW 实践：postgres_fdw 跨库查询、数据联邦写入性能调优与多数据源整合
tags: [postgresql, fdw, foreign-data-wrapper, postgres_fdw, data-integration,跨库查询]
---

## 概述

PostgreSQL FDW（Foreign Data Wrapper，外部数据包装器）允许在 PG 中直接查询**远程数据库**或其他异构数据源（MySQL、MongoDB、CSV 等），就像查询本地表一样。最常用的是 `postgres_fdw`，用于连接其他 PostgreSQL 实例。

> 适用场景：微服务分库查询、报表系统跨库汇总、数据迁移中间态、读写分离的读库聚合。

## 1. postgres_fdw 基础配置

### 安装与创建扩展

```sql
-- 在本地数据库中执行
CREATE EXTENSION IF NOT EXISTS postgres_fdw;
```

### 创建外部服务器

```sql
-- 定义远程连接
CREATE SERVER remote_production
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (
    host '10.0.1.100',
    port '5432',
    dbname 'orders_db'
);
```

### 用户映射

```sql
-- 本地用户到远程用户的映射
CREATE USER MAPPING FOR local_user
SERVER remote_production
OPTIONS (
    user 'remote_user',
    password 'remote_password'
);

-- 使用 PUBLIC 允许所有用户通过同一映射连接
CREATE USER MAPPING FOR PUBLIC
SERVER remote_production
OPTIONS (
    user 'readonly_user',
    password 'readonly_pass'
);
```

### 创建外部表

```sql
-- 方式一：手动建表（推荐，可按需选择字段）
CREATE FOREIGN TABLE remote_orders (
    id            BIGINT,
    order_no      VARCHAR(32),
    user_id       BIGINT,
    amount        NUMERIC(12,2),
    status        VARCHAR(16),
    created_at    TIMESTAMP
)
SERVER remote_production
OPTIONS (
    schema_name 'public',
    table_name  'orders'
);

-- 方式二：导入特定表（自动生成结构）
IMPORT FOREIGN SCHEMA public
LIMIT TO (orders, order_items)
FROM SERVER remote_production
INTO public;
```

## 2. 查询优化与下推

### 检查是否下推

```sql
-- 使用 EXPLAIN VERBOSE 查看执行计划
EXPLAIN (VERBOSE, ANALYZE)
SELECT * FROM remote_orders
WHERE created_at >= '2025-01-01'
  AND status = 'PAID'
ORDER BY created_at DESC
LIMIT 100;

-- 应显示类似：
-- Foreign Scan on remote_orders
--   -> Remote SQL: SELECT ... WHERE (created_at >= '2025-01-01'::timestamp) AND (status = 'PAID'::text)
--   下推成功，只有少量结果回传到本地
```

### 确保谓词下推

```sql
-- ✅ 可下推：WHERE、LIMIT、JOIN 条件中的列引用
-- ❌ 不可下推：函数调用（非内置函数）、类型不匹配的比较

-- ✅ 下推友好
WHERE amount > 1000
AND status IN ('PAID', 'REFUNDING')

-- ❌ 本地执行（无法下推）
WHERE my_custom_func(amount) > 0
```

### JOIN 优化：远程 JOIN vs 本地 JOIN

```sql
-- 如果两个表在同一远程服务器，设置 fetch_size 让小表拉到本地 JOIN
-- 或将 JOIN 下推（PG 18+ 支持跨 FDW JOIN 下推）

-- 设置每次拉取的行数（默认 100，提高后可减少往返）
ALTER SERVER remote_production OPTIONS (ADD fetch_size '500');
```

## 3. 写入操作

### 可更新外部表的条件

- `postgres_fdw` 默认支持 INSERT、UPDATE、DELETE
- 需远程表有主键或唯一约束
- `UPDATE` 和 `DELETE` 需使用 `RETURNING` 支持

```sql
-- INSERT
INSERT INTO remote_orders (order_no, user_id, amount, status)
VALUES ('ORD-2025-001', 1001, 299.00, 'PAID');

-- UPDATE（需远程表有主键）
UPDATE remote_orders
SET status = 'SHIPPED'
WHERE id = 12345;

-- DELETE
DELETE FROM remote_orders WHERE id = 12345;
```

### 批量操作性能

```sql
-- 批量插入（推荐用批量，减少网络往返）
INSERT INTO remote_orders (order_no, user_id, amount, status)
SELECT generate_series(1, 1000) AS order_no,
       1001 AS user_id,
       10.00 AS amount,
       'PAID' AS status;

-- 使用 FDW batch_size 参数
ALTER SERVER remote_production OPTIONS (ADD batch_size '100');
-- batch_size 控制单次插入批处理的行数，默认 1，建议 100-500
```

## 4. 常用参数调优

```sql
-- 服务器级别参数
ALTER SERVER remote_production OPTIONS (
    SET fetch_size '500',        -- SQL 查询每次拉取的行数（默认 100）
    SET batch_size '200',        -- 批量 INSERT 每次提交行数（默认 1）
    SET keep_connections 'true'  -- 复用连接，减少连接开销（默认 false）
);

-- Session 级覆盖
SET postgres_fdw.fetch_size TO 1000;
SET postgres_fdw.batch_size TO 500;
```

| 参数 | 默认值 | 推荐值 | 说明 |
|------|--------|--------|------|
| `fetch_size` | 100 | 500-2000 | 每次游标读取行数，越大减少网络往返 |
| `batch_size` | 1 | 100-500 | INSERT 每次批量提交行数 |
| `keep_connections` | false | true | 事务间保持远程连接 |
| `use_remote_estimate` | false | true | 从远程获取统计信息优化查询计划 |
| `fdw_startup_cost` | 100 | - | FDW 评估的启动成本，高值抑制使用 FDW 计划 |

```sql
-- 启用远程估算
ALTER FOREIGN TABLE remote_orders OPTIONS (ADD use_remote_estimate 'true');
```

## 5. 其他 FDW 扩展

| FDW | 适用数据源 | 安装 |
|-----|-----------|------|
| **mysql_fdw** | MySQL / MariaDB | `CREATE EXTENSION mysql_fdw;` |
| **mongo_fdw** | MongoDB | 需要编译安装 |
| **file_fdw** | CSV/文本文件 | PG 自带 |
| **oracle_fdw** | Oracle | 需要编译和 OCI |
| **tds_fdw** | SQL Server / Sybase | `CREATE EXTENSION tds_fdw;` |

### file_fdw 示例：查询 CSV 文件

```sql
CREATE EXTENSION file_fdw;

CREATE SERVER file_server FOREIGN DATA WRAPPER file_fdw;

CREATE FOREIGN TABLE csv_import (
    id    INTEGER,
    name  TEXT,
    email TEXT
)
SERVER file_server
OPTIONS (
    filename '/data/import/users.csv',
    format  'csv',
    header  'true'
);

-- 直接查询 CSV
SELECT * FROM csv_import WHERE name LIKE '张%';
```

### mysql_fdw 示例

```sql
CREATE EXTENSION mysql_fdw;

CREATE SERVER mysql_server
FOREIGN DATA WRAPPER mysql_fdw
OPTIONS (host '127.0.0.1', port '3306');

CREATE USER MAPPING FOR local_user
SERVER mysql_server
OPTIONS (username 'app_user', password 'secret');

CREATE FOREIGN TABLE mysql_users (
    id      INTEGER,
    name    TEXT,
    created TIMESTAMP
)
SERVER mysql_server
OPTIONS (dbname 'app_db', table_name 'users');
```

## 6. 实战：报表跨库聚合

```sql
-- 场景：订单在 orders_db，用户明细在 users_db
-- 需要跨数据库关联查询

-- 在报表库中注册两个外部表
CREATE FOREIGN TABLE f_orders (...) SERVER server_orders;
CREATE FOREIGN TABLE f_users (...) SERVER server_users;

-- 跨库查询
SELECT
    u.name AS user_name,
    u.level AS user_level,
    COUNT(o.id) AS order_count,
    SUM(o.amount) AS total_amount
FROM f_users u
JOIN f_orders o ON u.id = o.user_id
WHERE o.created_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY u.id, u.name, u.level
ORDER BY total_amount DESC
LIMIT 20;
```

> ⚠️ **跨 FDW JOIN 性能注意事项**：PG17+ 改进了跨 FDW JOIN 下推，但建议仍按「小表拉到本地 JOIN」思路设计查询。若两个外部表在同一远程服务器，PG 会自动将 JOIN 下推。

## 注意事项

### ⚠️ 事务行为
- FDW 查询默认使用远程**读已提交**隔离级别
- 本地和远程事务**不完全一致**：本地 ROLLBACK 不会自动回滚远程（2PC 除外）
- 跨 FDW 的分布式事务需使用 **2PC（两阶段提交）**，PG 18+ 支持增强

### ⚠️ 连接池管理
- `keep_connections = true` 会保持远程连接，但连接数有限
- 多个本地会话共享同一个 FDW 连接
- 监控远程 PG 的 `max_connections`，避免耗尽

### ⚠️ 执行计划异常
- 外部表统计信息默认从远程获取（`use_remote_estimate = true` 时）
- 如果远程表数据分布变化大，本地执行计划可能不准确
- 定期在远程库执行 `ANALYZE` 保持统计信息新鲜

### ⚠️ 权限管理
- 远程用户授予最小权限（如只读用户仅 SELECT）
- 本地通过 `USER MAPPING` 控制访问凭证
- 敏感表不要用 `IMPORT FOREIGN SCHEMA` 全量导入

### ⚠️ 性能瓶颈
- FDW 不适合 OLTP 高频查询（每次查询有网络开销）
- 适合报表场景、数据同步、管理后台
- 大数据量（>10万行）建议在远程做好聚合再查
