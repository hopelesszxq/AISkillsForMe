---
name: mybatis-plus-multitenant
description: MyBatis-Plus 多租户插件与数据权限——TenantLineInnerInterceptor、DataPermissionInterceptor 实战
tags: [mybatis-plus, multitenant, data-permission, saas, interceptor]
---

## 概述

MyBatis-Plus 提供内置的多租户拦截器 `TenantLineInnerInterceptor`，可自动为 SQL 追加租户过滤条件，无需手动拼接 `WHERE tenant_id = ?`。结合自定义数据权限拦截器，可实现行级数据隔离。

## 一、多租户插件（租户行级隔离）

### 1.1 基础配置

```java
@Configuration
@MapperScan("com.example.mapper")
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 多租户拦截器（必须先于分页拦截器注册）
        interceptor.addInnerInterceptor(new TenantLineInnerInterceptor(
            new TenantLineHandler() {
                @Override
                public Expression getTenantId() {
                    // 从当前上下文获取租户 ID
                    Long tenantId = TenantContextHolder.getCurrentTenantId();
                    if (tenantId == null) {
                        throw new RuntimeException("缺少租户信息");
                    }
                    return new LongValue(tenantId);
                }

                @Override
                public String getTenantIdColumn() {
                    // 租户列名，默认 tenant_id
                    return "tenant_id";
                }

                @Override
                public boolean ignoreTable(String tableName) {
                    // 忽略不需要加租户条件的基础表
                    return Set.of(
                        "sys_config", "sys_dict", "sys_region",
                        "flyway_schema_history"
                    ).contains(tableName);
                }
            }
        ));

        // 分页插件（必须在租户之后）
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));

        return interceptor;
    }
}
```

### 1.2 租户上下文持有者

```java
public class TenantContextHolder {

    private static final ThreadLocal<Long> TENANT_ID = new ThreadLocal<>();

    public static void setCurrentTenantId(Long tenantId) {
        TENANT_ID.set(tenantId);
    }

    public static Long getCurrentTenantId() {
        return TENANT_ID.get();
    }

    public static void clear() {
        TENANT_ID.remove();
    }

    // 在请求过滤器中设置
    public static class TenantFilter implements Filter {
        @Override
        public void doFilter(ServletRequest request, ServletResponse response,
                             FilterChain chain) throws IOException, ServletException {
            try {
                // 从 JWT Token、Header 或子域名获取租户 ID
                HttpServletRequest req = (HttpServletRequest) request;
                String tenantId = req.getHeader("X-Tenant-Id");
                if (tenantId != null) {
                    TenantContextHolder.setCurrentTenantId(Long.parseLong(tenantId));
                }
                chain.doFilter(request, response);
            } finally {
                TenantContextHolder.clear(); // 必须清理！
            }
        }
    }
}
```

### 1.3 部分表忽略租户条件

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

    interceptor.addInnerInterceptor(new TenantLineInnerInterceptor(
        new TenantLineHandler() {
            // ... getTenantId, getTenantIdColumn ...

            @Override
            public boolean ignoreTable(String tableName) {
                // 方式1：静态忽略列表
                if (tableName.startsWith("sys_") || tableName.startsWith("dict_")) {
                    return true;
                }
                // 方式2：通过注解动态忽略
                return DynamicTableContext.shouldIgnoreTenant(tableName);
            }
        }
    ));
    interceptor.addInnerInterceptor(new PaginationInnerInterceptor());
    return interceptor;
}
```

### 1.4 关联查询时插入租户条件

多租户插件会自动将租户条件注入到 JOIN 查询中：

```sql
-- 原始 SQL
SELECT u.*, o.order_amount
FROM user u
JOIN orders o ON u.id = o.user_id
WHERE u.status = 'ACTIVE'

-- 自动追加后
SELECT u.*, o.order_amount
FROM user u
JOIN orders o ON u.id = o.user_id
WHERE u.status = 'ACTIVE'
  AND u.tenant_id = 1001
  AND o.tenant_id = 1001
```

### 1.5 特殊处理：INSERT 语句

默认情况下，INSERT 语句会自动补充租户列：

```java
// 自动追加 tenant_id 列和值
// INSERT INTO user (name, age, tenant_id) VALUES (?, ?, 1001)
userMapper.insert(new User().setName("张三").setAge(25));
```

如需禁止某些表的 INSERT 自动注入租户字段：

```java
@Override
public boolean ignoreInsert(String tableName, List<Column> columns) {
    return "audit_log".equals(tableName);
}
```

## 二、数据权限插件（行级数据过滤）

租户插件解决的是「不同租户之间的数据隔离」，而数据权限解决的是「同一租户内不同角色能看哪些数据」。

### 2.1 自定义数据权限拦截器

```java
@Slf4j
public class DataPermissionInnerInterceptor implements InnerInterceptor {

    @Override
    public void beforeQuery(Executor executor, MappedStatement ms,
                            Object parameter, RowBounds rowBounds,
                            ResultHandler resultHandler, BoundSql boundSql) {
        // 获取当前用户的数据权限范围
        DataScope scope = DataScopeContext.get();
        if (scope == null || !scope.isEnabled()) {
            return; // 不限制
        }

        // 解析并重写 SQL
        String originalSql = boundSql.getSql();
        String rewrittenSql = DataPermissionSqlParser.rewrite(originalSql, scope);
        if (!rewrittenSql.equals(originalSql)) {
            // 通过反射替换 BoundSql 中的 SQL
            ReflectUtil.setFieldValue(boundSql, "sql", rewrittenSql);
        }
    }
}
```

### 2.2 使用注解驱动数据权限

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DataPermission {
    /**
     * 需要过滤的表别名
     */
    String tableAlias() default "t";

    /**
     * 部门字段名
     */
    String deptField() default "dept_id";

    /**
     * 用户字段名
     */
    String userField() default "create_by";
}

// AOP 实现
@Aspect
@Component
public class DataPermissionAspect {

    @Around("@annotation(dataPermission)")
    public Object around(ProceedingJoinPoint joinPoint,
                         DataPermission dataPermission) throws Throwable {
        try {
            // 根据当前用户角色构造数据权限范围
            UserDetails user = SecurityUtils.getCurrentUser();
            DataScope scope = buildDataScope(user, dataPermission);
            DataScopeContext.set(scope);
            return joinPoint.proceed();
        } finally {
            DataScopeContext.clear();
        }
    }

    private DataScope buildDataScope(UserDetails user, DataPermission dp) {
        // 超级管理员 -> 全量
        if (user.isAdmin()) {
            return DataScope.all();
        }
        // 部门管理员 -> 本部门及子部门
        if (user.isDeptAdmin()) {
            return DataScope.deptAndChildren(user.getDeptId());
        }
        // 普通用户 -> 仅本人数据
        return DataScope.self(user.getUserId());
    }
}
```

### 2.3 常用的数据权限模式

| 模式 | SQL 追加条件 | 适用场景 |
|------|-------------|---------|
| 全部数据 | 不加条件 | 超级管理员 |
| 本部门及子部门 | `dept_id IN (子部门ID列表)` | 部门经理 |
| 本部门 | `dept_id = ?` | 部门主管 |
| 仅本人 | `create_by = ?` | 普通员工 |
| 自定义 | `区域编码前缀匹配` | 区域销售 |

## 三、动态表名插件

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

    // 动态表名（按月分表场景）
    interceptor.addInnerInterceptor(new DynamicTableNameInnerInterceptor(
        new TableNameHandler() {
            @Override
            public String dynamicTableName(String sql, String tableName) {
                String suffix = TableNameContext.get();
                if (suffix != null && !"order".equals(tableName)) {
                    return tableName; // 只处理 order 表
                }
                return tableName + "_" + suffix; // order → order_202606
            }
        }
    ));

    return interceptor;
}
```

```java
// 使用
TableNameContext.set("202606");
orderMapper.selectList(null); // → SELECT * FROM order_202606
TableNameContext.clear();
```

## 四、多插件注册顺序（重要）

```java
@Bean
public MybatisPlusInterceptor interceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    // 1. 多租户（最先执行，给所有 SQL 加租户条件）
    interceptor.addInnerInterceptor(tenantLineInterceptor());
    // 2. 数据权限（在租户基础上再加行级过滤）
    interceptor.addInnerInterceptor(dataPermissionInterceptor());
    // 3. 动态表名（修改表名）
    interceptor.addInnerInterceptor(dynamicTableNameInterceptor());
    // 4. 分页（最后执行，在 COUNT/分页之前已经确定了完整 SQL）
    interceptor.addInnerInterceptor(paginationInterceptor());
    // 5. 乐观锁
    interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
    return interceptor;
}
```

**错误顺序的后果**：
- 租户排在分页之后 → 分页 COUNT 查询未加租户条件 → 统计租户数据泄露
- 分页排在乐观锁之后 → 乐观锁版本号更新可能被分页影响

## 五、注意事项

- **分页与租户顺序**：`TenantLineInnerInterceptor` 必须在 `PaginationInnerInterceptor` 之前注册，否则分页 COUNT 查询不会携带租户条件
- **INSERT 自动补列**：默认自动为 INSERT 补充租户列，但由于 SQL 解析局限，不支持 `INSERT ... SELECT` 和 `REPLACE INTO`
- **JOIN 性能**：多租户为每个 JOIN 表追加 WHERE 条件，当关联表超过 5 张时建议用 `ignoreTable` 忽略非核心业务表
- **测试覆盖**：每个 Mapper 方法应该写租户单元测试，验证生成的 SQL 包含 `AND tenant_id = ?`
- **排除注册**：不需要租户隔离的 Mapper 方法可标记 `@InterceptorIgnore(tenantLine = "1")` 跳过拦截
- **ThreadLocal 清理**：`TenantContextHolder` 必须在请求结束的 `finally` 块中清理，防止内存泄漏和线程污染
- **Spring Boot 4 兼容**：MyBatis-Plus 3.5.15+ 支持 Spring Boot 4.0.0，但需使用 `mybatis-plus-spring-boot4-starter`
