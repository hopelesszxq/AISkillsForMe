---
name: mybatis-plus-auto-fill
description: MyBatis-Plus MetaObjectHandler 自动填充深度实践：审计字段、自定义策略、多租户填充
tags: [mybatis-plus, orm, database, auto-fill, audit]
---

## 概述

MyBatis-Plus 的 MetaObjectHandler（元对象处理器）提供 **自动填充** 功能，可在 INSERT 或 UPDATE 时自动为指定字段赋值，适用于：创建时间/更新时间、创建人/更新人、逻辑删除标记、租户 ID 等审计字段。

## 1. 基础配置

### 实体类注解

```java
import com.baomidou.mybatisplus.annotation.FieldFill;
import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableLogic;
import java.time.LocalDateTime;

@Data
public class User {

    private Long id;

    private String name;

    @TableField(fill = FieldFill.INSERT) // 插入时填充
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE) // 插入和更新时填充
    private LocalDateTime updateTime;

    @TableField(fill = FieldFill.INSERT)
    private Long createBy;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private Long updateBy;

    @TableField(fill = FieldFill.INSERT)
    @TableLogic
    private Integer deleted; // 逻辑删除标记（0-正常，1-删除）

    @TableField(fill = FieldFill.INSERT)
    private Long tenantId; // 多租户 ID

    @Version
    @TableField(fill = FieldFill.INSERT)
    private Integer version; // 乐观锁版本号初始值
}
```

FieldFill 可选值：
| 枚举值 | 触发时机 |
|---|---|
| `FieldFill.DEFAULT` | 默认不填充 |
| `FieldFill.INSERT` | 插入时填充 |
| `FieldFill.UPDATE` | 更新时填充 |
| `FieldFill.INSERT_UPDATE` | 插入和更新时填充 |

### 处理器实现

```java
import com.baomidou.mybatisplus.core.handlers.MetaObjectHandler;
import org.apache.ibatis.reflection.MetaObject;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;

@Component
public class MyMetaObjectHandler implements MetaObjectHandler {

    /**
     * 插入填充策略
     */
    @Override
    public void insertFill(MetaObject metaObject) {
        // strictInsertFill: 只有字段上声明了 fill = INSERT/INSERT_UPDATE 且字段值为 null 时才填充
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "createBy", Long.class, getCurrentUserId());
        this.strictInsertFill(metaObject, "updateBy", Long.class, getCurrentUserId());
        this.strictInsertFill(metaObject, "deleted", Integer.class, 0);
        this.strictInsertFill(metaObject, "tenantId", Long.class, getCurrentTenantId());
        this.strictInsertFill(metaObject, "version", Integer.class, 1);
    }

    /**
     * 更新填充策略
     */
    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
        this.strictUpdateFill(metaObject, "updateBy", Long.class, getCurrentUserId());
    }

    /**
     * 获取当前用户 ID（需根据实际认证框架实现）
     */
    private Long getCurrentUserId() {
        // 例如从 Spring Security 获取
        // SecurityContextHolder.getContext().getAuthentication()
        return 1L; // 示例
    }

    private Long getCurrentTenantId() {
        return 1L; // 示例
    }
}
```

## 2. 填充模式详解

### strictInsertFill vs insertFill

```java
// strictInsertFill — 仅当字段值为 null 时填充（推荐）
this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());

// insertFill — 无论字段是否有值都填充（会覆盖已有值）
this.insertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
```

**⚠️ 坑**：`strictInsertFill` 依赖字段类型精确匹配。如果 entity 字段是 `Date` 类型但提供了 `LocalDateTime` 值，则不生效。建议统一使用 `LocalDateTime`。

### 支持 Java 8 日期类型

```java
// 如需支持 Date 类型
this.strictInsertFill(metaObject, "createDate", Date.class, new Date());
```

### 使用 setFieldValByName 直接赋值

```java
@Override
public void insertFill(MetaObject metaObject) {
    // 更灵活的方式，不强制要求类型匹配
    Object val = getFieldValByName("createTime", metaObject);
    if (val == null) {
        setFieldValByName("createTime", LocalDateTime.now(), metaObject);
    }
}
```

## 3. 高级场景

### 3.1 获取操作类型（INSERT vs UPDATE）

```java
@Override
public void insertFill(MetaObject metaObject) {
    // 通过 metaObject 获取原始 SQL 类型
    String sqlType = metaObject.getOriginalObject().getClass().getSimpleName();
    // 实际填充逻辑
}
```

### 3.2 根据用户角色动态填充

```java
@Component
public class SmartMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        // 管理员可以手动指定创建人，普通用户由系统填充
        User currentUser = getCurrentUser();
        if (currentUser != null && !currentUser.isAdmin()) {
            // 只有非管理员才自动填充
            this.strictInsertFill(metaObject, "createBy", Long.class, currentUser.getId());
            this.strictInsertFill(metaObject, "deptId", Long.class, currentUser.getDeptId());
        }
    }
}
```

### 3.3 配合多租户

当 MyBatis-Plus 多租户插件和自动填充同时使用时，tenantId 可通过自动填充赋值，也可通过多租户拦截器自动追加 WHERE 条件：

```java
// 自动填充（INSERT 时设置租户 ID）
this.strictInsertFill(metaObject, "tenantId", Long.class, getCurrentTenantId());

// 多租户插件配置（读操作自动追加 WHERE tenant_id = ?）
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new TenantLineInnerInterceptor(
        new TenantLineHandler() {
            @Override
            public String getTenantIdColumn() {
                return "tenant_id";
            }
            @Override
            public Expression getTenantId() {
                return new LongValue(getCurrentTenantId());
            }
        }
    ));
    return interceptor;
}
```

### 3.4 批量操作时自动填充

MyBatis-Plus 的批量插入（`saveBatch`）底层通过 `sqlSession.insert` 逐个执行，自动填充仍然生效。但 JDBC 批量模式（`rewriteBatchedStatements=true`）可能会跳过拦截器：

```yaml
# application.yml — 确保批量插入时拦截器生效
mybatis-plus:
  configuration:
    default-executor-type: simple  # 不要用 BATCH 模式，否则 MetaObjectHandler 可能不触发
```

如果必须使用 BATCH 模式，需手动填充：

```java
List<User> users = getUsers();
users.forEach(user -> {
    user.setCreateTime(LocalDateTime.now());
    user.setUpdateTime(LocalDateTime.now());
});
userService.saveBatch(users, 1000); // BATCH 模式下已手动赋值，无需 MetaObjectHandler
```

## 4. 注意事项

### 4.1 自动填充和 @TableField 默认值

如果实体字段已有默认值，且使用 `strictInsertFill`，由于字段值不是 null，填充不会生效：

```java
// 不会自动填充，因为默认值不为 null
private LocalDateTime createTime = LocalDateTime.now();
```

**解决**：移除字段默认值，全部由 MetaObjectHandler 管理。

### 4.2 线程安全问题

不要在 MetaObjectHandler 中使用实例变量存储用户信息，应使用 ThreadLocal 或从请求上下文中获取：

```java
@Component
public class SafeMetaObjectHandler implements MetaObjectHandler {

    // ❌ 错误：实例变量在多线程下会被覆盖
    // private Long currentUserId;

    @Override
    public void insertFill(MetaObject metaObject) {
        // ✅ 正确：每次都从安全上下文中获取
        Long userId = SecurityContextHolder.getCurrentUserId();
        this.strictInsertFill(metaObject, "createBy", Long.class, userId);
    }
}
```

### 4.3 单元测试

```java
@SpringBootTest
class MetaObjectHandlerTest {

    @Autowired
    private UserMapper userMapper;

    @Test
    void testInsertFill() {
        User user = new User();
        user.setName("test");
        userMapper.insert(user);

        User saved = userMapper.selectById(user.getId());
        assertNotNull(saved.getCreateTime());  // 自动填充
        assertNotNull(saved.getUpdateTime());
    }
}
```

### 4.4 性能建议

- 频繁写入场景下，时间字段自动填充比应用层赋值更可靠且性能差异可忽略
- 避免在自动填充中执行远程调用或数据库查询（会拖慢每个 INSERT/UPDATE）
- 对于万亿级大表，考虑将审计字段放在业务层显式赋值，减少反射开销
