---
name: mybatis-plus-multi-tenant
description: MyBatis-Plus 多租户插件实战：数据隔离策略、自定义处理器、动态租户上下文与性能优化
tags: [mybatis-plus, multi-tenant, saas, database, orm]
---

## 概述

MyBatis-Plus 内置多租户 SQL 拦截器（`TenantLineInnerInterceptor`），通过自动追加 WHERE 条件实现**行级数据隔离**，无需在每个 Mapper 中手动拼接租户条件。适用于 SaaS 多租户系统的 Shared Database + Shared Schema 模式。

## 1. 基础配置

### 分页拦截器 + 多租户拦截器

```java
@Configuration
public class MyBatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 多租户拦截器（必须在分页之前）
        interceptor.addInnerInterceptor(new TenantLineInnerInterceptor(
            new CustomTenantHandler()
        ));

        // 分页拦截器
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.POSTGRE_SQL));

        return interceptor;
    }
}
```

### 租户处理器（TenantLineHandler）

```java
@Component
public class CustomTenantHandler implements TenantLineHandler {

    private static final String TENANT_COLUMN = "tenant_id";

    @Override
    public String getTenantIdColumn() {
        return TENANT_COLUMN;
    }

    @Override
    public Expression getTenantId() {
        // 从当前上下文中获取租户 ID
        Long tenantId = TenantContext.getCurrentTenantId();
        if (tenantId == null) {
            throw new RuntimeException("当前租户未设置");
        }
        return new LongValue(tenantId);
    }

    @Override
    public boolean ignoreTable(String tableName) {
        // 忽略不需要租户隔离的表
        return List.of("sys_config", "sys_dict", "sys_log", "flyway_schema_history")
            .contains(tableName);
    }
}
```

## 2. 租户上下文（ThreadLocal）

```java
public class TenantContext {

    private static final ThreadLocal<Long> CURRENT_TENANT = new ThreadLocal<>();

    public static void setCurrentTenantId(Long tenantId) {
        CURRENT_TENANT.set(tenantId);
    }

    public static Long getCurrentTenantId() {
        return CURRENT_TENANT.get();
    }

    public static void clear() {
        CURRENT_TENANT.remove();
    }
}
```

### Web 拦截器设置租户上下文

```java
@Component
public class TenantInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, 
                             HttpServletResponse response, 
                             Object handler) {
        // 从请求头/Token 中解析租户 ID
        String tenantId = request.getHeader("X-Tenant-Id");
        if (tenantId != null) {
            TenantContext.setCurrentTenantId(Long.parseLong(tenantId));
        }
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler,
                                Exception ex) {
        TenantContext.clear(); // 防止内存泄漏
    }
}
```

## 3. 复杂场景处理

### 手动排除租户条件（特定查询）

```java
@Mapper
public interface UserMapper extends BaseMapper<User> {

    // 使用 @SqlParser 注解忽略租户过滤（旧版）
    @SqlParser(filter = true)
    @Select("SELECT * FROM sys_user WHERE status = 1")
    List<User> listAllUsers();

    // 新版：使用自定义方法 + 临时禁用
    default List<User> listCrossTenant() {
        // 在调用前通过 TenantContext 标记忽略
        return selectList(Wrappers.emptyWrapper());
    }
}
```

### 新版临时禁用租户（推荐方式）

```java
// 使用 MybatisPlusInterceptor 的 clear/restore 机制
public class TenantUtils {

    private static final MybatisPlusInterceptor interceptor;

    static {
        interceptor = SpringContextHolder.getBean(MybatisPlusInterceptor.class);
    }

    public static <T> T withoutTenant(Supplier<T> supplier) {
        try {
            interceptor.removeInnerInterceptor(TenantLineInnerInterceptor.class);
            return supplier.get();
        } finally {
            interceptor.addInnerInterceptor(new TenantLineInnerInterceptor(
                new CustomTenantHandler()
            ));
        }
    }
}

// 使用
// List<User> allUsers = TenantUtils.withoutTenant(() -> userService.list());
```

### 动态忽略表

```java
@Component
public class DynamicTenantHandler implements TenantLineHandler {

    private final Set<String> ignoreTables = new HashSet<>();

    public void addIgnoreTable(String tableName) {
        ignoreTables.add(tableName);
    }

    public void removeIgnoreTable(String tableName) {
        ignoreTables.remove(tableName);
    }

    @Override
    public boolean ignoreTable(String tableName) {
        return ignoreTables.contains(tableName);
    }

    // ... 其他方法
}
```

## 4. DDL 语句处理

多租户拦截器默认不处理 DDL（CREATE TABLE / ALTER TABLE）。如需对 DDL 做租户隔离，可自定义 SQL 注入器：

```java
@Component
public class TenantDDLInjector extends DefaultSqlInjector {

    @Override
    public List<AbstractMethod> getMethodList(Class<?> mapperClass, TableInfo tableInfo) {
        List<AbstractMethod> methods = super.getMethodList(mapperClass, tableInfo);
        // 添加自定义 DDL 方法
        return methods;
    }
}
```

## 5. SQL 示例（拦截效果）

```sql
-- 原始 SQL
SELECT * FROM orders WHERE status = 1

-- 拦截后（自动追加 tenant_id）
SELECT * FROM orders WHERE status = 1 AND tenant_id = 1001

-- 插入自动追加
INSERT INTO orders (order_no, amount) VALUES ('NO001', 100)
-- 拦截后
INSERT INTO orders (tenant_id, order_no, amount) VALUES (1001, 'NO001', 100)

-- 更新自动追加
UPDATE orders SET status = 2 WHERE id = 1
-- 拦截后
UPDATE orders SET status = 2 WHERE id = 1 AND tenant_id = 1001
```

## 注意事项

- **租户字段必须建索引**：`CREATE INDEX idx_orders_tenant ON orders(tenant_id)`
- **ThreadLocal 必须清理**：请求结束后务必清除，防止内存泄漏
- **ignoreTable 判断尽量用 HashSet**：避免每次 SQL 解析都做字符串匹配
- **分页 + 多租户顺序**：多租户拦截器必须注册在分页拦截器之前
- **JOIN 查询注意**：多表 JOIN 时只会给主表加租户条件，副表需要手动处理
- **批量插入性能**：批量插入时租户 ID 会自动填充到每行，无需担心
- **@SqlParser 已废弃**：3.5.x+ 推荐使用拦截器动态移除方式
