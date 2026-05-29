---
name: mybatis-plus-optimistic-lock
description: MyBatis-Plus 乐观锁实战：配置、重试策略、并发冲突处理与性能优化
tags: [mybatis-plus, optimistic-lock, concurrency, orm]
---

## 概述

乐观锁（Optimistic Locking）通过版本号或时间戳机制解决并发更新冲突，在读多写少的场景中比悲观锁（SELECT ... FOR UPDATE）性能更好。MyBatis-Plus 内置 `OptimisticLockerInnerInterceptor` 插件，只需简单配置即可实现。

## 1. 基础配置

### 引入拦截器

```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 乐观锁插件（必须放在分页插件前面）
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());

        // 分页插件
        MybatisPlusInterceptor pagination = new MybatisPlusInterceptor();
        pagination.addInnerInterceptor(new PaginationInnerInterceptor(DbType.POSTGRE_SQL));
        interceptor.addInnerInterceptor(pagination);

        return interceptor;
    }
}
```

### 实体类配置

```java
@Data
@TableName("sys_user")
public class User {

    @TableId(type = IdType.AUTO)
    private Long id;

    private String name;
    private String email;

    /**
     * 乐观锁版本号字段
     * 支持类型：int, Integer, long, Long, Date, Timestamp, LocalDateTime
     * 每次 UPDATE 时自动 version +1
     */
    @Version
    @TableField(fill = FieldFill.INSERT)
    private Integer version;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.UPDATE)
    private LocalDateTime updateTime;
}
```

## 2. 自动填充默认版本号

```java
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        // INSERT 时版本号默认 1
        this.strictInsertFill(metaObject, "version", Integer.class, 1);
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }
}
```

## 3. 使用示例

```java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    /**
     * 乐观锁更新 - 自动校验版本号
     * MyBatis-Plus 自动生成的 SQL:
     *   UPDATE sys_user SET name=?, email=?, version=version+1
     *   WHERE id=? AND version=?
     * 返回 0 表示版本冲突，更新失败
     */
    @Transactional
    public boolean updateUser(User user) {
        // 先查询出当前版本
        User existing = userMapper.selectById(user.getId());
        if (existing == null) {
            throw new BusinessException("用户不存在");
        }

        // 从 DB 中拿到最新版本号
        user.setVersion(existing.getVersion());

        // 更新时会自动校验 version，失败返回 0
        int rows = userMapper.updateById(user);
        return rows > 0;
    }
}
```

## 4. 重试策略（处理并发冲突）

```java
@Service
public class RetryableUserService {

    private static final int MAX_RETRIES = 3;

    @Autowired
    private UserMapper userMapper;

    /**
     * 带重试的乐观锁更新
     * 适用场景：高并发下少量冲突（如库存扣减）
     */
    @Transactional
    public boolean updateUserWithRetry(User user) {
        for (int attempt = 1; attempt <= MAX_RETRIES; attempt++) {
            try {
                User existing = userMapper.selectById(user.getId());
                user.setVersion(existing.getVersion());

                int rows = userMapper.updateById(user);
                if (rows > 0) {
                    return true;
                }

                // 版本冲突，短暂等待后重试
                if (attempt < MAX_RETRIES) {
                    Thread.sleep(50 * attempt); // 退避：50ms → 100ms → 150ms
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException("重试中断", e);
            }
        }
        // 记录冲突日志，人工介入或走降级
        log.warn("乐观锁更新失败，user={}, 已达最大重试次数", user.getId());
        throw new OptimisticLockException("用户数据已被他人修改，请刷新后重试");
    }
}
```

## 5. Spring Retry 声明式重试

```java
@Service
@Slf4j
public class RetryableOrderService {

    @Autowired
    private OrderMapper orderMapper;

    /**
     * 使用 @Retryable 声明式重试（需引入 spring-retry）
     *
     * 依赖：
     * <dependency>
     *     <groupId>org.springframework.retry</groupId>
     *     <artifactId>spring-retry</artifactId>
     * </dependency>
     */
    @Retryable(
        retryFor = OptimisticLockException.class,
        maxAttempts = 4,
        backoff = @Backoff(delay = 50, multiplier = 2, maxDelay = 500)
    )
    @Transactional
    public void deductStock(Long productId, int quantity, Integer currentVersion) {
        Order order = orderMapper.selectById(productId);
        if (currentVersion != order.getVersion()) {
            throw new OptimisticLockException("版本冲突");
        }

        // 直接使用 LambdaQueryWrapper 手动带 version 条件
        order.setStock(order.getStock() - quantity);
        LambdaUpdateWrapper<Order> wrapper = Wrappers.lambdaUpdate(Order.class)
            .eq(Order::getId, productId)
            .eq(Order::getVersion, currentVersion);
        int rows = orderMapper.update(order, wrapper);

        if (rows == 0) {
            throw new OptimisticLockException("库存扣减冲突");
        }
    }

    @Recover
    public void recover(OptimisticLockException e, Long productId, int quantity, Integer version) {
        log.error("扣减库存失败（重试耗尽）：productId={}, quantity={}", productId, quantity);
        // 写入死信队列或失败表，人工处理
    }
}
```

## 6. 批量操作注意事项

```java
@Transactional
public void batchUpdateWithVersion(List<User> users) {
    // ❌ 错误写法：批量 updateById 会逐条更新，但版本号逐条校验
    // users.forEach(user -> userMapper.updateById(user));

    // ✅ 正确写法：使用批量更新，保留每条记录的版本号
    for (User user : users) {
        User existing = userMapper.selectById(user.getId());
        user.setVersion(existing.getVersion());
        int rows = userMapper.updateById(user);
        if (rows == 0) {
            throw new OptimisticLockException("用户 " + user.getId() + " 版本冲突");
        }
    }
}
```

## 7. 与逻辑删除共存

```java
@Data
@TableName("sys_user")
public class User {

    @TableId
    private Long id;

    @Version
    private Integer version;

    @TableLogic
    private Integer deleted;  // 0=正常, 1=已删除

    // 逻辑删除的 UPDATE 不会加版本号校验
    // MyBatis-Plus 自动处理：DELETE 实质上走 UPDATE set deleted=1，不加 version 条件
}
```

## 注意事项

### 工作原理
- `OptimisticLockerInnerInterceptor` 会自动拦截 Entity 的 `updateById` 和 `update(T entity, Wrapper)` 方法
- 生成的 SQL 为：`UPDATE table SET col1=?, col2=?, version=version+1 WHERE id=? AND version=?`
- 仅当 `version` 字段值匹配时才更新成功，`updateById` 返回 `0 `表示冲突

### 限制与坑点
1. **仅支持 `updateById` 和带 Wrapper 的 `update`**：自定义 XML Mapper 中的 UPDATE 不会自动加版本校验，需要手动写 `AND version = #{version}`
2. **不支持批量更新**：`saveBatch` 底层是逐条 INSERT，与乐观锁无关；批量 UPDATE 用 Wrapper 需要手动带条件
3. **Wrapper 覆盖**：如果用 `LambdaUpdateWrapper` 手动设置 version 值，插件不会自动覆盖
4. **类型必须匹配**：Entity 中的 `@Version` 字段类型必须和 DB 字段一致，否则 SQL 拼接异常

### 选型建议
| 方案 | 适用场景 | 复杂度 |
|------|---------|--------|
| MyBatis-Plus 插件 | 标准 CRUD，低并发冲突 | 低 |
| 手动 Wrapper + version | 复杂条件更新，自定义 SQL | 中 |
| 悲观锁 FOR UPDATE | 高冲突、短事务（库存扣减） | 低 |
| Redis 分布式锁 | 跨服务、跨 DB 的并发控制 | 高 |
