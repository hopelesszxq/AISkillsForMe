---
name: mybatis-plus-iservice-chain
description: MyBatis-Plus IService 链式查询与 Service 层最佳实践：LambdaChain、saveOrUpdate、批量操作、事务与性能优化
tags: [mybatis-plus, iservice, chain, service-layer, orm, java]
---

## 概述

MyBatis-Plus 的 `IService<T>` 接口提供了丰富的 CRUD 封装和链式查询能力，但很多开发者只用了基础的 `save()`/`list()`/`page()` 方法，对高阶链式操作、批量处理、事务集成并不熟悉。本文深入 IService 层的所有实用特性。

## 1. IService 核心方法分层

### 基础 CRUD

```java
public interface UserService extends IService<User> {
}
```

| 分类 | 方法 | 说明 |
|------|------|------|
| 新增 | `save(entity)` / `saveBatch(list)` / `saveOrUpdate(entity)` | 单条/批量/存在即更新 |
| 删除 | `removeById(id)` / `remove(queryWrapper)` / `removeByIds(ids)` | 按 ID/条件删除 |
| 更新 | `updateById(entity)` / `update(wrapper, entity)` | 按 ID/条件更新 |
| 查询 | `getById(id)` / `list(wrapper)` / `page(page, wrapper)` | 单条/列表/分页 |
| 计数 | `count(wrapper)` | 条件计数 |

### 链式查询（LambdaChain）

```java
// ===== 链式查询 =====

// 单表查询：lambdaQuery() 返回 LambdaQueryChainWrapper
User user = userService.lambdaQuery()
    .eq(User::getUsername, "admin")
    .eq(User::getStatus, 1)
    .select(User::getId, User::getUsername, User::getEmail)
    .one();  // 返回单个实体

// 列表查询
List<User> users = userService.lambdaQuery()
    .like(User::getUsername, "test")
    .ge(User::getCreateTime, LocalDateTime.now().minusDays(7))
    .orderByDesc(User::getId)
    .last("limit 10")
    .list();

// 分页查询
Page<User> page = userService.lambdaQuery()
    .eq(User::getDeptId, deptId)
    .page(new Page<>(1, 20));  // 直接返回 Page 对象

// 统计
long count = userService.lambdaQuery()
    .eq(User::getStatus, 1)
    .count();
```

### 链式更新

```java
// ===== 链式更新 =====

// 条件更新
boolean updated = userService.lambdaUpdate()
    .eq(User::getId, 1)
    .set(User::getEmail, "new@example.com")
    .set(User::getUpdateTime, LocalDateTime.now())
    .update();  // 执行更新

// 批量条件更新
userService.lambdaUpdate()
    .eq(User::getDeptId, 100)
    .set(User::getStatus, 0)
    .update();

// 存在则更新，不存在则插入
userService.saveOrUpdate(user, new LambdaUpdateWrapper<User>()
    .eq(User::getUsername, user.getUsername()));
```

### 链式删除

```java
// ===== 链式删除 =====
userService.lambdaQuery()
    .eq(User::getStatus, 2)       // 逻辑删除状态：已删除
    .lt(User::getUpdateTime, LocalDateTime.now().minusMonths(6))
    .remove();  // 物理删除（实际调用 remove）

// 也支持 chain() 方法（别名）
userService.query().eq("status", 1).list();      // 字符串方式
userService.update().eq("id", 1).set("name", "x").update();  // 字符串方式
```

## 2. 批量操作最佳实践

### saveBatch 源码原理

```java
// 默认分批大小：1000
userService.saveBatch(userList);                   // 默认 1000 一批
userService.saveBatch(userList, 500);              // 自定义 500 一批

// 原理（伪代码）
// 1. 将 list 切分为 batchSize 大小的小批次
// 2. 每批调用 save() → sqlSession.flushStatements()
// 3. 每批结束后提交（取决于事务管理器）
```

> **重要**：`saveBatch` 默认**不在全局事务中**，每批独立提交。需要事务时参考下文。

### 真正的批量 INSERT 性能优化

```java
// === 方式一：使用内置批量（默认逐条 INSERT）===
// Mapper 层实际是逐条 INSERT，通过批量提交减少网络往返
userService.saveBatch(list);

// === 方式二：使用 MP 的批量 SQL 注入器（真正的批量 INSERT）===
// 需要自定义 SQL 注入器，生成 INSERT INTO ... VALUES (...), (...), (...)
// 参考：mybatis-plus-sql-injector.md

// === 方式三：JDBC 连接参数开启 rewriteBatch || 对于 MySQL ===
// jdbc:mysql://localhost:3306/db?rewriteBatchedStatements=true
// 配合 MyBatis 的 batchExecutorType
SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH);
// 在 session 中执行批量操作
```

### saveOrUpdate 批量

```java
// 批量 saveOrUpdate（每个元素判断 ID 是否存在）
// 对每条记录执行：如果 ID 存在则 UPDATE，否则 INSERT
userService.saveOrUpdateBatch(userList);

// 自定义批量大小的 saveOrUpdate
userService.saveOrUpdateBatch(userList, 500);
```

## 3. 事务集成

### 链式操作中的事务

```java
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {

    // 方式一：@Transactional 包裹 chain 操作
    @Transactional(rollbackFor = Exception.class)
    public void batchUpdateWithTx(List<User> users) {
        // lambdaUpdate 中的 update() 会使用当前事务
        users.forEach(user -> 
            this.lambdaUpdate()
                .eq(User::getId, user.getId())
                .set(User::getEmail, user.getEmail())
                .update());
    }

    // 方式二：结合 saveOrUpdateBatch 在事务中
    @Transactional(rollbackFor = Exception.class)
    public void saveOrUpdateWithTx(List<User> users) {
        this.saveOrUpdateBatch(users);
    }
}
```

### 批量插入的事务控制

```java
@Transactional(rollbackFor = Exception.class)
public boolean saveLargeBatch(List<User> list) {
    // 自行拆分批次，全部在同一个事务中
    List<List<User>> batches = Lists.partition(list, 1000);
    for (List<User> batch : batches) {
        baseMapper.insertBatchSomeColumn(batch);  // 需要 SQL 注入器支持
    }
    return true;
}
```

## 4. Service 层继承与扩展

### 自定义 Service 基类

```java
public interface BaseService<T> extends IService<T> {
    // 自定义通用方法
    Page<T> pageWithSort(Page<T> page, Wrapper<T> wrapper, String sortField, boolean asc);
    boolean softDeleteById(Serializable id);
}

public class BaseServiceImpl<M extends BaseMapper<T>, T> extends ServiceImpl<M, T> implements BaseService<T> {

    @Override
    public Page<T> pageWithSort(Page<T> page, Wrapper<T> wrapper, String sortField, boolean asc) {
        if (StringUtils.isNotBlank(sortField)) {
            page.addOrder(new OrderItem(sortField, asc));
        }
        return baseMapper.selectPage(page, wrapper);
    }

    @Override
    public boolean softDeleteById(Serializable id) {
        T entity = getById(id);
        if (entity == null) return false;
        // 通过反射或接口设置 deleted 字段
        if (entity instanceof SoftDeletable) {
            ((SoftDeletable) entity).setDeleted(true);
            return updateById(entity);
        }
        return false;
    }
}
```

## 5. Service 层性能优化

### 避免 N+1 查询

```java
// ❌ 错误：循环中逐个查询
List<Order> orders = orderService.list();
for (Order order : orders) {
    User user = userService.getById(order.getUserId());  // N+1
}

// ✅ 正确：批量查询
List<Order> orders = orderService.list();
Set<Long> userIds = orders.stream().map(Order::getUserId).collect(Collectors.toSet());
List<User> users = userService.listByIds(userIds);
Map<Long, User> userMap = users.stream().collect(Collectors.toMap(User::getId, u -> u));
```

### 使用 select 只查需要的字段

```java
// ❌ 查询所有字段（如果表有 30 个字段）
List<User> users = userService.list(queryWrapper);

// ✅ 只查需要的字段
List<User> users = userService.lambdaQuery()
    .select(User::getId, User::getUsername, User::getEmail)  // 只查 3 个字段
    .eq(User::getStatus, 1)
    .list();
```

## 6. IService vs Mapper 直接调用

| 场景 | IService | Mapper |
|------|----------|--------|
| 简单 CRUD | ✅ 优先使用 | ❌ 不推荐 |
| 复杂 SQL/多表 JOIN | ❌ 不直接支持 | ✅ 自定义 XML |
| 批量操作 | ✅ saveBatch | ✅ insertBatchSomeColumn |
| 链式查询 | ✅ lambdaQuery | ❌ 需要手动封装 |
| 分页查询 | ✅ page(wrapper) | ✅ selectPage(page, wrapper) |
| 动态 SQL | ❌ | ✅ @Select + script |

## 注意事项

- `saveBatch` 默认走逐条 INSERT（通过批量提交减少网络往返），不是真正的批量 INSERT
- `saveOrUpdate` 先根据 ID 查库判断存在性，批量场景下效率低，建议自行实现唯一键冲突处理
- `lambdaQuery().one()` 如果查询返回多条数据会抛出 `TooManyResultsException`
- 链式操作不支持跨表 JOIN，复杂 SQL 仍需 Mapper XML
- ServiceImpl 中 `baseMapper` 是泛型 Mapper 实例，可以随时降级到 Mapper 层操作
