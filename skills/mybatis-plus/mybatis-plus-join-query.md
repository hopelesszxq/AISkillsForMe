---
name: mybatis-plus-join-query
description: MyBatis-Plus 多表关联查询实战：自定义 SQL + 分页、注解 @TableName 多表、Wrapper 联表扩展与数据权限集成
tags: [mybatis-plus, join, sql, pagination, multi-table, query]
---

## MyBatis-Plus 多表关联查询

MyBatis-Plus 原生 Lambda 查询只支持**单表操作**，多表关联需要借助自定义 SQL 或扩展手段。本文覆盖主流方案。

## 1. 自定义 XML 实现 JOIN 查询（推荐生产方案）

### Mapper XML

```xml
<!-- UserMapper.xml -->
<mapper namespace="com.example.mapper.UserMapper">
    
    <!-- 关联查询结果映射 -->
    <resultMap id="UserDeptResultMap" type="com.example.dto.UserDeptDTO">
        <id property="id" column="id"/>
        <result property="username" column="username"/>
        <result property="email" column="email"/>
        <result property="deptName" column="dept_name"/>
        <result property="deptCode" column="dept_code"/>
    </resultMap>
    
    <!-- 左连接查询 -->
    <select id="selectUserWithDept" resultMap="UserDeptResultMap">
        SELECT u.id, u.username, u.email,
               d.dept_name, d.dept_code
        FROM sys_user u
        LEFT JOIN sys_dept d ON u.dept_id = d.id
        <where>
            <if test="username != null and username != ''">
                AND u.username LIKE CONCAT('%', #{username}, '%')
            </if>
            <if test="deptCode != null">
                AND d.dept_code = #{deptCode}
            </if>
            <if test="status != null">
                AND u.status = #{status}
            </if>
        </where>
        ORDER BY u.create_time DESC
    </select>
    
</mapper>
```

### Java 接口

```java
@Mapper
public interface UserMapper extends BaseMapper<User> {
    
    // 分页查询用户+部门
    IPage<UserDeptDTO> selectUserWithDept(
        Page<?> page, 
        @Param("username") String username,
        @Param("deptCode") String deptCode,
        @Param("status") Integer status
    );
}
```

### DTO

```java
@Data
public class UserDeptDTO {
    private Long id;
    private String username;
    private String email;
    private String deptName;
    private String deptCode;
}
```

### 调用

```java
@Service
public class UserService {
    
    @Autowired
    private UserMapper userMapper;
    
    public IPage<UserDeptDTO> queryUsers(UserQuery query) {
        Page<UserDeptDTO> page = new Page<>(query.getPageNum(), query.getPageSize());
        return userMapper.selectUserWithDept(
            page, 
            query.getUsername(), 
            query.getDeptCode(),
            query.getStatus()
        );
    }
}
```

## 2. @TableName 多表关联（动态表名 SQL 注入器）

### 跨表查询器

```java
import com.baomidou.mybatisplus.core.metadata.TableInfoHelper;
import com.baomidou.mybatisplus.extension.plugins.inner.InnerInterceptor;

/**
 * 通过自定义拦截器实现关联查询
 */
@Component
public class JoinQueryInterceptor implements InnerInterceptor {
    
    @Override
    public void beforeQuery(Executor executor, MappedStatement ms, Object parameter,
                            RowBounds rowBounds, ResultHandler resultHandler,
                            BoundSql boundSql) {
        // 获取原始 SQL
        String originalSql = boundSql.getSql();
        
        // 如果参数包含 join 标记，自动追加 JOIN 语句
        if (parameter instanceof JoinQueryParam) {
            JoinQueryParam joinParam = (JoinQueryParam) parameter;
            String joinSql = buildJoinSql(joinParam);
            // 修改 BoundSql（需反射）
            ReflectUtil.setFieldValue(boundSql, "sql", originalSql + joinSql);
        }
    }
    
    private String buildJoinSql(JoinQueryParam param) {
        StringBuilder sb = new StringBuilder();
        for (JoinTable join : param.getJoins()) {
            sb.append(" ").append(join.getType().name())
              .append(" JOIN ").append(join.getTableName())
              .append(" ON ").append(join.getOnCondition());
        }
        return sb.toString();
    }
}
```

> ⚠️ **注意**：这种拦截器方式侵入性强，建议仅用于简单场景。复杂关联优先用 XML 方式。

## 3. MyBatis-Plus Join 扩展包

社区有专门的 `mybatis-plus-join` 扩展，支持流式 JOIN 查询：

```xml
<dependency>
    <groupId>com.github.yulichang</groupId>
    <artifactId>mybatis-plus-join-boot-starter</artifactId>
    <version>1.4.11</version>
</dependency>
```

### 使用扩展包

```java
// 继承 MPJBaseMapper 而非 BaseMapper
public interface UserMapper extends MPJBaseMapper<User> {
}

// 链式 JOIN 查询
@Service
public class UserService {
    
    @Autowired
    private UserMapper userMapper;
    
    public List<UserDeptDTO> getUsersWithDept() {
        return userMapper.selectJoinList(UserDeptDTO.class,
            new MPJLambdaWrapper<User>()
                .selectAll(User.class)
                .select(Dept::getDeptName, Dept::getDeptCode)
                .leftJoin(Dept.class, Dept::getId, User::getDeptId)
                .eq(User::getStatus, 1)
                .like(User::getUsername, "admin")
                .orderByAsc(User::getCreateTime)
        );
    }
    
    // 分页 JOIN 查询
    public IPage<UserDeptDTO> pageUsersWithDept(UserQuery query) {
        Page<UserDeptDTO> page = new Page<>(query.getPageNum(), query.getPageSize());
        return userMapper.selectJoinPage(page, UserDeptDTO.class,
            new MPJLambdaWrapper<User>()
                .selectAll(User.class)
                .select(Dept::getDeptName, Dept::getDeptCode)
                .leftJoin(Dept.class, Dept::getId, User::getDeptId)
                .like(query.getUsername() != null, User::getUsername, query.getUsername())
                .eq(query.getStatus() != null, User::getStatus, query.getStatus())
        );
    }
}
```

## 4. 数据权限 + JOIN 查询集成

### 自定义拦截器实现数据权限过滤

```java
@Component
public class DataScopeInterceptor implements InnerInterceptor {
    
    @Override
    public void beforeQuery(Executor executor, MappedStatement ms, Object parameter,
                            RowBounds rowBounds, ResultHandler resultHandler,
                            BoundSql boundSql) {
        // 获取当前用户数据权限
        DataScope scope = SecurityUtils.getCurrentDataScope();
        if (scope == null || scope.isAdmin()) {
            return; // 管理员不限制
        }
        
        String originalSql = boundSql.getSql();
        String filteredSql = applyDataScope(originalSql, scope);
        
        ReflectUtil.setFieldValue(boundSql, "sql", filteredSql);
    }
    
    private String applyDataScope(String sql, DataScope scope) {
        // 追加部门过滤条件
        // SELECT * FROM sys_user WHERE ... → 追加 AND dept_id IN (scope_dept_ids)
        if (sql.contains("sys_user") && CollectionUtil.isNotEmpty(scope.getDeptIds())) {
            String deptFilter = " AND (dept_id IN (" + 
                scope.getDeptIds().stream()
                    .map(String::valueOf)
                    .collect(Collectors.joining(",")) + 
                ") OR create_by = " + scope.getUserId() + ")";
            sql = sql.replace("WHERE", "WHERE " + deptFilter + " AND");
        }
        return sql;
    }
}
```

## 5. 性能优化

### JOIN 查询优化原则

| 原则 | 说明 |
|------|------|
| 只查必要列 | 避免 `SELECT *`，只查需要的字段 |
| 小表驱动大表 | JOIN 顺序：小结果集作为驱动表 |
| 索引覆盖关联键 | JOIN 条件列和 WHERE 过滤列加索引 |
| 分页查总数 | COUNT 使用覆盖索引，避免 JOIN 导致的性能损失 |

### 分页 COUNT 优化

```xml
<!-- 自定义 COUNT 查询，避免全表 JOIN 统计 -->
<select id="selectUserWithDeptCount" resultType="java.lang.Long">
    SELECT COUNT(1) FROM sys_user u
    <where>
        <if test="username != null">
            AND u.username LIKE CONCAT('%', #{username}, '%')
        </if>
        <!-- 注意：COUNT 不 JOIN，直接在主表统计 -->
    </where>
</select>
```

## 注意事项

### 1. 分页问题
- MyBatis-Plus 分页插件自动生成 COUNT 查询时，如果原 SQL 包含 JOIN，COUNT 也会包含 JOIN
- 大表 JOIN 的分页查询建议**手写 COUNT** 或使用 COUNT 优化策略

### 2. DTO vs Entity
- JOIN 查询结果不要映射到 Entity，用 DTO 接收
- 避免 Entity 中被无关字段污染

### 3. 扩展包限制
- `mybatis-plus-join` 扩展包对复杂 JOIN（子查询、UNION）支持有限
- 多表 JOIN 超过 3 张表时，建议回归 XML 手写

### 4. 多租户冲突
- 使用多租户插件时，JOIN 查询可能自动追加租户过滤
- 跨租户查询需要临时关闭多租户拦截器：
```java
// 临时关闭多租户
try {
    TenantContextHolder.setIgnore(true);
    return userMapper.selectJoinList(...);
} finally {
    TenantContextHolder.setIgnore(false);
}
```
