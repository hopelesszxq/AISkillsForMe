---
name: pg-soft-delete
description: PostgreSQL 软删除设计模式完整指南：逻辑删除 vs 物理删除、部分唯一索引、查询过滤与性能优化
tags: [postgresql, soft-delete, partial-index, unique-index, performance]
---

## 概述

软删除（逻辑删除）通过 `deleted_at` 标记而非物理删除数据，适合需要恢复/审计的场景。但 PostgreSQL 中软删除处理不当会导致索引膨胀、唯一约束破坏和查询性能下降。

## 1. 软删除核心设计

### 推荐方案：deleted_at + partial unique index

```sql
-- 基础表结构
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) NOT NULL,
    username    VARCHAR(100) NOT NULL,
    deleted_at  TIMESTAMPTZ,        -- NULL = 未删除, 有值 = 已删除时间
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 部分唯一索引：只对未删除记录强制唯一
CREATE UNIQUE INDEX uk_users_email_active
    ON users(email)
    WHERE deleted_at IS NULL;

CREATE UNIQUE INDEX uk_users_username_active
    ON users(username)
    WHERE deleted_at IS NULL;
```

### 查询必须带 deleted_at 过滤

```sql
-- ✅ 正确：自动使用部分索引
SELECT * FROM users
WHERE email = 'alice@example.com' AND deleted_at IS NULL;

-- ❌ 错误：WHERE 条件不匹配部分索引定义，走全表扫描
SELECT * FROM users WHERE email = 'alice@example.com';
```

## 2. 两种软删除实现对比

| 方案 | 实现 | 优点 | 缺点 |
|------|------|------|------|
| **NULL-marked** | `deleted_at TIMESTAMPTZ NULL` | 部分索引友好，查询清晰，默认不走删除数据 | 需要每个查询加 `WHERE deleted_at IS NULL` |
| **Boolean flag** | `is_deleted BOOLEAN DEFAULT FALSE` | 更直观 | 部分索引仍需额外条件，默认查询可能扫到大量已删数据 |

**推荐统一使用 `deleted_at` 方案**，天然区分活跃记录。

## 3. MyBatis-Plus 自动过滤

```java
@TableName(value = "users", autoResultMap = true)
public class User {
    private Long id;
    private String email;
    private String username;

    @TableField(fill = FieldFill.INSERT)
    @TableLogic(value = "null", delval = "now()")
    private LocalDateTime deletedAt;  // LocalDateTime 接收 TIMESTAMPTZ
}
```

### 配置全局逻辑删除

```yaml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted_at
      logic-delete-value: "now()"    # 删除时写入当前时间
      logic-not-delete-value: "null" # 未删除为 NULL
```

### 注意：MyBatis-Plus 4.x 变更

Spring Boot 4 + MyBatis-Plus 4.x 中 `@TableLogic` 的 `value` 和 `delval` 支持 SpEL 表达式：

```java
@TableLogic(value = "null", delval = "#{@nowProvider.get()}")
private LocalDateTime deletedAt;
```

## 4. 部分唯一索引解决重复注册问题

常见场景：用户注销后重新注册同一邮箱。

```sql
-- 已删除用户允许相同 email
INSERT INTO users (email, username, deleted_at)
VALUES ('alice@example.com', 'alice_old', now());

-- 新注册使用相同 email：不会唯一冲突（因为部分索引不包含已删记录）
INSERT INTO users (email, username)
VALUES ('alice@example.com', 'alice_new');
-- ✅ 成功，uk_users_email_active 不索引已删行
```

## 5. 性能优化与维护

### 定期清理已删数据

```sql
-- 归档 90 天前的软删除数据
WITH archived AS (
    DELETE FROM users
    WHERE deleted_at IS NOT NULL
      AND deleted_at < now() - INTERVAL '90 days'
    RETURNING *
)
INSERT INTO users_archive SELECT * FROM archived;
```

### 防止部分索引失效

PostgreSQL 部分索引要求**查询的 WHERE 条件必须完全匹配**索引定义：

```sql
-- 索引定义 WHERE deleted_at IS NULL

-- ✅ 走索引
SELECT * FROM users WHERE deleted_at IS NULL AND email = 'x@x.com';

-- ❌ 不走索引（条件顺序不同不影响，但缺条件不行）
SELECT * FROM users WHERE email = 'x@x.com';

-- ❌ 不走索引（参数化写法用 COALESCE）
SELECT * FROM users
WHERE email = 'x@x.com'
  AND (deleted_at IS NULL OR deleted_at IS NOT NULL);  -- 破坏匹配
```

### 统计信息优化

大量软删除记录后需 ANALYZE：

```sql
-- 软删除比例高达 50%+ 时
ANALYZE users;

-- 或直接设置部分索引的填充因子
CREATE UNIQUE INDEX uk_users_email_active
    ON users(email) WHERE deleted_at IS NULL
    WITH (fillfactor = 70);  -- 预留 30% 空间减少页分裂
```

## 6. 替代方案：分区 + 物理删除

对于极高删除率的表（如每日 80% 行被删），考虑分区方案：

```sql
-- 按 deleted_at 分区
CREATE TABLE orders (
    id BIGSERIAL,
    status TEXT,
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ
) PARTITION BY RANGE (deleted_at);

-- 活跃分区（deleted_at IS NULL 的行）
CREATE TABLE orders_active PARTITION OF orders
    FOR VALUES FROM (MINVALUE) TO ('1970-01-01');

-- 已删分区
CREATE TABLE orders_deleted_202601 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- 直接 truncate 过期分区 = 物理删除
TRUNCATE orders_deleted_202501;
```

## 注意事项

- **触发器/级联问题**：软删除不触发外键级联删除，需业务层处理关联数据
- **审计一致性**：建议 `deleted_at` 与 `deleted_by` 配合使用
- **查询默认过滤**：ORM 层务必全局配置逻辑删除，避免遗漏
- **批量删除性能**：`UPDATE ... WHERE deleted_at IS NULL` 会锁大量行，考虑分批
- **唯一索引冲突**：重构唯一约束时，必须重建部分索引而非简单 DROP/CREATE
