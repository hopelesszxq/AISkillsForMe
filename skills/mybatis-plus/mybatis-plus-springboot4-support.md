---
name: mybatis-plus-3516-springboot4
description: MyBatis-Plus 3.5.15~3.5.16 新特性：Jackson 3 支持、Spring Boot 4 兼容、CrudRepository 优化
tags: [mybatis-plus, orm, spring-boot-4, jackson, upgrade]
---

## 概述

MyBatis-Plus 3.5.14~3.5.16 版本引入了对 **Spring Boot 4.0.0** 和 **Jackson 3.0** 的全面支持，标志着 MyBatis-Plus 进入新一代 Spring Boot 生态的关键升级。

## 主要新特性

### 1. Spring Boot 4.0.0 支持（3.5.14+）

MyBatis-Plus 3.5.14 增加了 BOM 管理，3.5.15 开始全面支持 Spring Boot 4.0.0：

```xml
<!-- Spring Boot 4 专用 Starter -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot4-starter</artifactId>
    <version>3.5.16</version>
</dependency>

<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot4-starter-test</artifactId>
    <version>3.5.16</version>
    <scope>test</scope>
</dependency>
```

BOM 管理方式：
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-bom</artifactId>
            <version>3.5.16</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 2. Jackson 3.0 支持

3.5.15 开始支持 Jackson 3.0，修复了 `Jackson3TypeHandler` 自定义 `ObjectMapper` 无效的问题：

```java
@Bean
public Jackson3TypeHandler jackson3TypeHandler(ObjectMapper objectMapper) {
    Jackson3TypeHandler handler = new Jackson3TypeHandler();
    // 自定义 ObjectMapper 现在可以生效
    handler.setObjectMapper(objectMapper);
    return handler;
}
```

**依赖升级**：jackson 升级至 2.20.1（同时支持 Jackson 3 路径）

### 3. CrudRepository 批量操作优化

3.5.15 优化了 `CrudRepository` 批量执行前的事务判断逻辑：

```java
public interface UserRepository extends CrudRepository<User, Long> {
}

// 批量插入 — 非事务环境会自动关闭连接
userRepository.saveBatch(userList);
```

核心优化：非事务环境中执行批量操作后及时关闭数据库连接，避免连接泄漏。

### 4. 代码生成器增强

- **元数据构建调整**：重构代码生成器元数据构建逻辑
- **Enjoy 模板 bug 修复**：修复 Enjoy 模板生成 XML 错误
- **模块名空值处理**：修复 `PackageConfig` 指定模块为空时拼接路径错误

### 5. 依赖升级汇总

| 组件 | 3.5.14 | 3.5.16 |
|------|--------|--------|
| Spring Boot 3 | - | 3.5.9 |
| mybatis-spring | - | 4.0.0 |
| fastjson | - | 2.0.60 |
| jackson | - | 2.20.1 |
| gson | - | 2.13.2 |
| PostgreSQL driver | - | 42.7.8 |
| MySQL driver | - | 9.5.0 |
| H2 | - | 2.4.240 |
| SQLite JDBC | - | 3.51.1.0 |

## 注意事项

1. **Spring Boot 4 Starter 不可混淆**：Spring Boot 3.x 项目用 `mybatis-plus-spring-boot3-starter`，Spring Boot 4.x 用 `mybatis-plus-spring-boot4-starter`
2. **Jackson 3 兼容性**：已在 XML/JSON 处理器中测试，但自定义序列化器需确保兼容
3. **配置加密修复**：3.5.16 修复了配置加密无法在环境变量中使用的问题
