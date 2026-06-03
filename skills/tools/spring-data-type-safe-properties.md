---
name: spring-data-type-safe-properties
description: Spring Data 类型安全属性引用 —— 用方法引用替代字符串，编译期校验属性名
tags: [tools, spring-data, jpa, type-safe, java, querydsl]
---

## 概述

Spring Data 在 2026 年引入了**类型安全属性引用**（Type-safe Property References），允许用 Java 方法引用（`Person::getFirstName`）替代字符串（`"firstName"`）来引用实体属性，实现**编译期校验、IDE 友好重构、无需代码生成**。

> 来源：Spring 官方博客 *"Moving beyond Strings in Spring Data"*（Mark Paluch, 2026-02-27）

## 核心问题

传统字符串属性引用的问题：

```java
// ❌ 字符串方式：编译通过，运行时可能失败
Sort.by("firstName", "lastName");
where("address.country").is(…);

// 重构属性名后，字符串不会自动更新，测试才能发现
```

字符串方式的问题：
- **编译器不校验** —— 拼写错误、重命名后不一致只能在运行时暴露
- **IDE 重构无效** —— 字符串字面量没有语义上下文，重构工具无法更新
- **脆弱性强** —— 查询声明与领域模型距离越远，越容易遗漏

## 类型安全属性引用（Type-safe Property References）

### 基础用法：方法引用替代字符串

```java
// ✅ 类型安全方式：方法引用，编译器校验
Sort.by(Person::getFirstName, Person::getLastName)

// 编译器验证属性是否存在
// IDE 支持导航到属性定义
// 重构工具自动更新所有引用
```

### 嵌套属性路径

```java
// 字符串方式
Sort.by("address.country");

// ✅ 类型安全方式：组合 PropertyPath
Sort.by(PropertyPath.of(Person::getAddress).then(Address::getCountry))
```

### 编译期类型校验

```java
Sort.by(Person::getFirstName, Order::getOrderDate);
// ❌ 编译错误：incompatible owner types
// 因为泛型约束 <T> Sort by(TypedPropertyPath<T, ?>... properties)
// 要求所有属性路径属于同一个根类型 T
```

### 简写语法（属性引用）

```java
Sort.by(Person::address)         // 字段引用
Sort.by(Person::address / Address::city)  // 路径拼接语法（未来规划）
```

## 与已有方案对比

| 方案 | 类型安全 | 编译期校验 | 代码生成 | 额外依赖 | 跨模块支持 |
|---|---|---|---|---|---|
| 字符串属性 | ❌ | ❌ | 无 | 无 | ✅ |
| JPA Metamodel | ✅ | ✅ | 需要 APT | 需要 | 有限 |
| Querydsl | ✅ | ✅ | 需要 APT | 需要 | 需适配 |
| jOOQ | ✅ | ✅ | 需要插件 | 需要 | 表级 |
| **Spring Data 方法引用** | ✅ | ✅ | **无** | **无** | **✅ 原生支持** |

### Querydsl 示例（对比）

```java
// Querydsl 方式：需要 APT 代码生成
QPerson person = QPerson.person;
query.where(person.firstName.eq("John"));
```

### JPA Criteria API 示例（对比）

```java
// JPA Criteria：需要 APT 生成 Person_ 元模型
CriteriaBuilder cb = entityManager.getCriteriaBuilder();
CriteriaQuery<Person> query = cb.createQuery(Person.class);
Root<Person> person = query.from(Person.class);
query.where(cb.equal(person.get(Person_.firstName), "John"));
```

## 适用场景

- **排序（Sort）**：`Sort.by(Person::getFirstName)`
- **查询谓词**：`where(PropertyPath.of(Person::getFirstName).is("John"))`
- **投影（Projection）**：属性路径导航
- **更新操作**：类型安全的字段指定

## 注意事项

1. **版本要求**：需要 Spring Data 2026.x 版本（具体从 3.x/4.x 开始支持），检查当前 Spring Boot 发行版中的 Spring Data 版本
2. **方法引用 vs 字段引用**：方法引用要求存在 getter 方法；纯字段引用（如 `Person::address`）需 Java 16+ 的 `record` 或公开字段
3. **性能**：方法引用在 JIT 编译后与直接调用无异，无额外开销
4. **迁移建议**：推荐在新代码中使用；存量代码可逐步替换，先从经常重构的模块开始
5. **IDE 支持**：IntelliJ IDEA 2026.x 以上支持方法引用的自动补全和重构
