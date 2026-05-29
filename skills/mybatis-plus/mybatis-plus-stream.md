---
name: mybatis-plus-stream
description: MyBatis-Plus Stream — 流式编程风格数据库操作框架
tags: [mybatis-plus, stream, orm, java, query]
---

## 概述

MyBatis-Plus Stream 是基于 MyBatis-Plus 3.5.16+ 的增强框架，将 **Java Stream 编程风格**引入数据库操作。通过 Lambda 类型安全的链式 API，可以用 `stream().filter().sorted().limit().collect()` 的方式完成从简单查询到多表联查、分组聚合的各种操作。

> 项目地址：https://github.com/kamioj/mybatis-plus-stream-boot-starter
> 版本：4.1.2.0（2026-05）| 环境要求：JDK 17+、Spring Boot 3.x

## 核心特性

| 特性 | 说明 |
|------|------|
| 流式查询 | `stream().filter().sorted().limit().collect()` 链式调用 |
| 连表查询 | 内置 LEFT / RIGHT / INNER / CROSS JOIN，Lambda 类型安全 |
| 聚合函数 | 100+ SQL 函数（COUNT、SUM、AVG、字符串、日期、数学等） |
| 分页查询 | 单表分页、连表分页、分组分页，一行搞定 |
| 逻辑删除 | `withDeleted()` 一键切换查询模式 |
| 批量写入 | saveDuplicate / saveIgnore / saveReplace |
| 联合更新 | 支持多表 JOIN UPDATE |
| 零侵入 | 继承 IMysqlServiceBase 即用，无需修改现有代码 |

## 快速开始

### 1. 引入依赖

```xml
<dependency>
    <groupId>io.github.kamioj</groupId>
    <artifactId>mybatis-plus-stream-boot-starter</artifactId>
    <version>4.1.2.0</version>
</dependency>
```

### 2. 修改 Mapper

```java
// 将 BaseMapper 替换为 StreamBaseMapper（4.0 起；旧名 MysqlBaseMapper @Deprecated）
public interface UserMapper extends StreamBaseMapper<User> {
}
```

### 3. 修改 Service

```java
// 接口继承 IStreamService
public interface UserService extends IStreamService<User> {
}

// 实现类继承 StreamServiceImpl
@Service
public class UserServiceImpl extends StreamServiceImpl<UserMapper, User> implements UserService {
}
```

### 4. 开始使用

```java
// 流式查询 — 像 Java Stream 一样操作数据库
List<User> users = userService.stream()
    .filter(where -> where.eq(User::getRole, "admin"))
    .sorted(order -> order.orderDesc(User::getCreateTime))
    .limit(10)
    .collect(Collectors.toList());

// 按条件获取单个实体
User user = userService.get(where -> where.eq(User::getId, 1));

// 行锁查询（SELECT ... FOR UPDATE）
User user = userService.getByKeyForUpdate(1);
```

## 主要 API

### 查询（Get/GetValue）

```java
// 获取单个字段值
String name = userService.getValue(where -> where.eq(User::getId, 1), User::getUsername);

// 聚合函数取值
Integer total = userService.getValue(
    where -> where.eq(User::getStatus, 1),
    func -> func.count()
);
```

### 列表查询（List）

```java
// 条件 + 排序 + 限制
List<User> users = userService.list(
    where -> where.eq(User::getStatus, 1),
    order -> order.orderDesc(User::getCreateTime),
    10
);

// 映射到 DTO
List<UserVO> vos = userService.list(
    where -> where.eq(User::getRole, "user"),
    select -> select
        .select(User::getId, UserVO::getId)
        .select(User::getUsername, UserVO::getUsername),
    UserVO.class
);
```

### 连表查询（Join）

```java
// LEFT JOIN 查询
List<User> users = userService.listJoin(
    join -> join.leftJoin(Order.class, User::getId, Order::getUserId),
    where -> where.gt(Order::getAmount, 100)
);

// JOIN + DTO 映射
List<UserOrderVO> vos = userService.listJoin(
    join -> join.leftJoin(Order.class, User::getId, Order::getUserId),
    where -> where.eq(User::getStatus, 1),
    select -> select
        .select(User::getUsername, UserOrderVO::getUsername)
        .select(Order::getOrderNo, UserOrderVO::getOrderNo),
    UserOrderVO.class
);
```

### 分组聚合（Group）

```java
// 分组 + 聚合函数
List<UserStatVO> stats = userService.listGroup(
    group -> group.groupBy(User::getDeptId),
    where -> where.eq(User::getStatus, 1),
    select -> select
        .select(User::getDeptId, UserStatVO::getDeptId)
        .selectFunc(func -> func.count(), UserStatVO::getUserCount),
    UserStatVO.class
);
```

### 动态条件查询

```java
public List<User> searchUsers(String keyword, Integer status, String role) {
    return userService.list(where -> {
        if (keyword != null) {
            where.like(User::getUsername, keyword);
        }
        if (status != null) {
            where.eq(User::getStatus, status);
        }
        if (role != null) {
            where.eq(User::getRole, role);
        }
    });
}
```

### 批量写入

```java
// 跳过已存在的记录（根据唯一索引判断）
userService.saveIgnore(userList);

// 替换已存在的记录
userService.saveReplace(userList);

// 批量插入不返回自增 ID
userService.saveBatchWithoutId(userList);
```

## 项目结构（4.0+）

```
src/main/java/com/baomidou/mybatisplus/extension/
├── mapper/       StreamBaseMapper           # 增强 Mapper 入口
├── service/      IStreamService + impl/     # 核心 Service 接口（60+ 方法）
├── dialect/      SqlDialect SPI             # 方言扩展（MySQL/PG/DM/自定义）
├── core/         查询执行内核
├── wrapper/      Wrapper 家族（26 类）
├── stream/       流式 API
├── value/        单列投影值容器
├── metadata/     表/列/类型元数据
└── bo/           通用容器（PageVo, SortVo, BiList 等）
```

## 注意事项

1. **JDK 17+ 必须**：大量使用 Lambda 和 Stream API，不支持 JDK 8/11
2. **Spring Boot 3.x 必须**：基于 Spring Boot 3 + MyBatis-Plus 3.5.16+
3. **数据库方言**：4.0 起通过 SPI 扩展，内置 MySQL / PostgreSQL / 达梦三种
4. **AGPL 许可证**：使用 AGPL v3，商业闭源使用需注意合规
5. **不适用于超复杂 SQL**：嵌套子查询、递归 CTE 等场景还是需要原生 XML Mapper
6. **性能注意**：流式 API 的链式调用有少量封装开销，单次查询无感，高频批量查询建议用原生 LambdaQueryWrapper
