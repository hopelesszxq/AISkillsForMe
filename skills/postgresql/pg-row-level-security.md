---
name: pg-row-level-security
description: PostgreSQL 行级安全(RLS)实战：多租户隔离、细粒度行权限控制与策略管理
tags: [postgresql, security, rls, multi-tenancy, authorization, policy]
---

## 概述

PostgreSQL 行级安全性（Row-Level Security, RLS）允许在数据库层面基于用户/角色/应用上下文自动过滤行记录。相比应用层过滤，RLS 在 SQL 解析阶段注入 WHERE 条件，**无法被绕过**，是合规审计和多租户隔离的首选方案。

> RLS 在 PostgreSQL 9.5 引入，PG 15+ 支持 `FORCE ROW LEVEL SECURITY` 对超级用户也生效。

## 基础配置

### 1. 启用 RLS

```sql
-- 对指定表启用 RLS
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- 可选：对超级用户也强制 RLS（PG 15+）
ALTER TABLE orders FORCE ROW LEVEL SECURITY;
```

### 2. 创建安全策略

```sql
-- 策略语法
CREATE POLICY policy_name ON table_name
    [FOR { ALL | SELECT | INSERT | UPDATE | DELETE }]
    [TO { role_name | PUBLIC | CURRENT_USER | SESSION_USER }]
    [USING (using_expression)]
    [WITH CHECK (check_expression)];
```

- `USING` — 控制现有行的可见性（SELECT, UPDATE, DELETE）
- `WITH CHECK` — 控制新插入或修改行的校验（INSERT, UPDATE）

## 场景一：多租户隔离（最常见）

每个 tenant_id 只能看到自己的数据：

```sql
-- 1. 建表
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    tenant_id   TEXT NOT NULL,
    customer    TEXT NOT NULL,
    amount      DECIMAL(10,2),
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- 2. 创建索引
CREATE INDEX idx_orders_tenant ON orders(tenant_id);

-- 3. 启用 RLS
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- 4. 创建策略
CREATE POLICY tenant_isolation ON orders
    FOR ALL
    USING (tenant_id = current_setting('app.tenant_id')::TEXT)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::TEXT);
```

### Spring Boot 集成

```java
@Component
public class TenantContextFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        try {
            String tenantId = extractTenant(request);
            // 设置 PostgreSQL session 变量
            try (Statement stmt = dataSource.getConnection().createStatement()) {
                stmt.execute("SET app.tenant_id = '" + tenantId + "'");
            }
            TenantContext.set(tenantId);
            chain.doFilter(request, response);
        } finally {
            TenantContext.clear();
        }
    }
}
```

或使用 Hibernate/MyBatis 拦截器自动设置：

```sql
-- 连接池层面设置（Druid/HikariCP 连接初始化 SQL）
SET app.tenant_id = 'tenant_a';
```

## 场景二：基于角色的行权限

```sql
-- 不同角色可见范围不同
CREATE POLICY role_based_access ON documents AS PERMISSIVE
    FOR SELECT
    USING (
        CASE WHEN current_role = 'admin' THEN
            TRUE  -- 管理员可见全部
        WHEN current_role = 'manager' THEN
            department_id = current_setting('app.department_id')::INT
        ELSE
            creator = current_user  -- 普通用户仅见自己创建
        END
    );
```

## 场景三：软删除 + RLS 结合

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;

-- 自动过滤已删除记录（应用层无需写 WHERE deleted_at IS NULL）
CREATE POLICY exclude_deleted ON users
    FOR SELECT
    USING (deleted_at IS NULL);

-- 管理员可查看已删除
CREATE POLICY admin_see_deleted ON users
    FOR SELECT
    TO admin_role
    USING (true);
```

## 场景四：读写分离策略

```sql
-- 所有人可查看已发布的文章
CREATE POLICY read_published ON articles
    FOR SELECT
    USING (status = 'published');

-- 作者只能修改自己的草稿
CREATE POLICY update_own_draft ON articles
    FOR UPDATE
    USING (author_id = current_user::BIGINT AND status = 'draft')
    WITH CHECK (author_id = current_user::BIGINT);
```

## 性能优化

```sql
-- 1. 确保策略列有索引（RLS 每次查询都会评估 USING 表达式）
CREATE INDEX idx_orders_tenant ON orders(tenant_id);

-- 2. 使用 IMMUTABLE 函数加速策略判断
CREATE OR REPLACE FUNCTION current_tenant_id() RETURNS TEXT
    LANGUAGE SQL IMMUTABLE PARALLEL SAFE
AS $$
    SELECT current_setting('app.tenant_id', true);
$$;

CREATE POLICY tenant_isolation_fast ON orders
    FOR ALL
    USING (tenant_id = current_tenant_id());

-- 3. 启用分区裁剪 + RLS（PG 12+）
CREATE TABLE measurements (
    id BIGSERIAL,
    tenant_id TEXT NOT NULL,
    measured_at TIMESTAMPTZ
) PARTITION BY LIST (tenant_id);

CREATE TABLE measurements_tenant_a
    PARTITION OF measurements FOR VALUES IN ('tenant_a');
CREATE TABLE measurements_tenant_b
    PARTITION OF measurements FOR VALUES IN ('tenant_b');

ALTER TABLE measurements ENABLE ROW LEVEL SECURITY;
-- 每个分区表自动继承父表 RLS
```

## 监控与调试

```sql
-- 查看所有 RLS 策略
SELECT schemaname, tablename, policyname, permissive, roles, cmd, qual
FROM pg_policies
ORDER BY tablename;

-- 测试 RLS 效果
BEGIN;
SET LOCAL app.tenant_id = 'tenant_a';
SELECT * FROM orders;  -- 只返回 tenant_a 的数据
ROLLBACK;

-- 查看查询计划是否使用了 RLS
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders;
-- RLS 会显示为 "Filter: (tenant_id = CURRENT_SETTING('app.tenant_id'::text))"
```

## 注意事项

- **超级用户绕过 RLS**：PG 15 之前超级用户自动绕过 RLS。PG 15+ 可使用 `FORCE ROW LEVEL SECURITY` 对超级用户也生效
- **外键与 RLS**：RLS 不影响外键约束检查（在 SQL 层面），这可能导致 "幽灵行" 问题
- **批量导入**：使用 `COPY` 命令时 RLS 不生效，批量导入数据需自行确保租户隔离
- **函数安全性**：如果 USING 表达式中调用了 VOLATILE 函数，会导致无法使用缓存执行计划
- **聚合函数**：`COUNT(*)` 等聚合函数同样受 RLS 限制，结果正确
- **ANALYZE 统计信息**：RLS 过滤后的行数不会影响 ANALYZE 的统计信息收集
- **迁移注意事项**：
  - 启用 RLS 前确保所有应用查询都携带正确的租户上下文
  - 使用 `ALTER TABLE ... DISABLE ROW LEVEL SECURITY;` 临时关闭进行数据迁移
  - 启用 RLS 后，原来通过应用层过滤的查询会**双重过滤**，但不影响结果正确性
