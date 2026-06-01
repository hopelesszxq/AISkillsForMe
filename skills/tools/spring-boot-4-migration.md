---
name: spring-boot-4-migration
description: Spring Boot 4.x 升级迁移指南——Spring Framework 7、Security 7、Jackson 3、Hibernate 7、JDK 21+ 关键变更与兼容处理
tags: [tools, spring-boot, migration, java, spring-framework, spring-security]
---

## 概述

Spring Boot 4.0.0（2026 年发布）是继 Boot 3.x 之后的**重大版本升级**，配合 **Spring Framework 7.0**、**JDK 21+**（推荐 JDK 25）和全新的 Jakarta EE 11 生态，带来了以下核心变化：

| 组件 | Boot 3.x | Boot 4.x | 说明 |
|------|----------|----------|------|
| Spring Framework | 6.x | **7.0.x** | JDK 17+ 最低，推荐 21+ |
| JDK 最低版本 | 17 | **21** | 必须 JDK 21+ |
| Spring Security | 6.x | **7.0.x** | 重大 API 变更 |
| Spring Data | 2024.x | **2025.1.x** | 新特性 |
| Jackson | 2.x | **3.x** | 包名变更 |
| Hibernate | 6.x | **7.x** | Jakarta Persistence 3.2 |
| Tomcat | 10.x | **11.x** | Jakarta EE 11 |
| Spring Batch | 5.x | **6.x** | 新版本 |
| Spring Integration | 6.x | **7.x** | 新版本 |

## 一、JDK 升级要求

**最低 JDK 21**，推荐 JDK 25 以获得最佳虚拟线程性能。

```xml
<!-- pom.xml -->
<properties>
    <java.version>21</java.version>
    <!-- 或 -->
    <java.version>25</java.version>
</properties>
```

### 必须处理的变化

- Java 9+ 模块系统（如使用，需配置 `module-info.java`）
- `Removal` 级别的废弃 API（如 `finalize()`、`SecurityManager`）已被移除
- JDK 结构性强制封装（`--add-opens` 需求减少但某些库仍需要）

## 二、Spring Framework 7.0 关键变更

### 虚拟线程默认支持

```yaml
spring:
  threads:
    virtual:
      enabled: true  # 全局启用虚拟线程
```

Spring MVC 和 WebFlux 均已适配虚拟线程模式：

```java
// 配置虚拟线程的 TaskExecutor
@Bean
TaskExecutor taskExecutor() {
    return new SimpleAsyncTaskExecutor().withVirtualThreads();
}
```

### Jakarta EE 11 迁移

Boot 4.x 已全面迁移至 Jakarta EE 11：

```xml
<!-- Servlet API 更新 -->
<dependency>
    <groupId>jakarta.servlet</groupId>
    <artifactId>jakarta.servlet-api</artifactId>
    <version>6.1.0</version>  <!-- 从 5.0/6.0 升级 -->
    <scope>provided</scope>
</dependency>
```

### RestClient 成为一等公民

Spring Framework 7 将 `RestClient` 和 `RestTemplate` 统一管理：

```java
// 推荐使用：统一的 HTTP 接口
@Bean
RestClient.Builder restClientBuilder() {
    return RestClient.builder();
}

// 使用方式
@Service
class OrderClient {
    private final RestClient restClient;

    OrderClient(RestClient.Builder builder) {
        this.restClient = builder.baseUrl("https://api.example.com").build();
    }

    Order getOrder(Long id) {
        return restClient.get()
            .uri("/orders/{id}", id)
            .retrieve()
            .body(Order.class);
    }
}
```

## 三、Spring Security 7.0 迁移

### 配置 DSL 变化

```java
// Boot 3.x 方式（已废弃）
@Bean
SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/**").authenticated()
            .anyRequest().permitAll()
        )
        .build();
}

// Boot 4.x + Security 7 新方式
@Bean
SecurityFilterChain filterChain(ServerHttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth
            .anyRequest().denyAll()  // 默认全部拒绝的引用方式
        )
        .oauth2ResourceServer(OAuth2ResourceServerSpec::jwt)
        .build();
}
```

### 废弃 API 移除

- `WebSecurityConfigurerAdapter`（已在 Boot 3.x 废弃，现完全移除）
- `@EnableWebSecurity` 不再是必需注解
- `HttpSecurity.headers()` 默认安全头更新

## 四、Jackson 2 → Jackson 3 迁移

### 包名变更

```java
// Jackson 2.x → Jackson 3.x
// com.fasterxml.jackson.databind.ObjectMapper  →  com.fasterxml.jackson.databind.json.JsonMapper
// JsonInclude.Include.NON_NULL                  →  JsonInclude.Include.NON_NULL 基本不变

// 主要变化：ObjectMapper 构建方式
ObjectMapper mapper = JsonMapper.builder()
    .enable(SerializationFeature.INDENT_OUTPUT)
    .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
    .build();
```

### 自动配置适配

Spring Boot 4.x 自动使用 Jackson 3，无需手动切换：

```yaml
spring:
  jackson:
    default-property-inclusion: non_null
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Shanghai
```

> **注意**：如果项目中同时存在 Jackson 2 和 Jackson 3 的依赖，`@JsonTest` 可能失败。需排除旧 Jackson 2 依赖。

## 五、Hibernate 7.x 迁移

### 主要变化

- Jakarta Persistence 3.2（`jakarta.persistence` 包）
- Hibernate 类型 API 重构（`BasicType` → `BasicTypeReference`）
- 序列生成器默认改为 `GenerationType.SEQUENCE`

```xml
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>7.2.12.Final</version>
</dependency>
```

### 实体注解兼容

```java
// 所有 jakarta.persistence 注解不变
// 但 Hibernate 特有注解可能变化
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;
}
```

## 六、Gradle 适配

```groovy
// build.gradle.kts
plugins {
    java
    id("org.springframework.boot") version "4.0.6"
    id("io.spring.dependency-management") version "1.1.7"
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

## 七、升级步骤

```
1. 升级 JDK 至 21+（先验证 JDK 21 兼容性）
2. 确认所有依赖支持 Jakarta EE 11
3. 更新 spring-boot-starter-parent 至 4.0.6
4. 替换 Jackson 2 → Jackson 3 的 API 调用
5. 更新 Hibernate 和数据库驱动版本
6. 更新 Spring Security 配置
7. 测试所有集成点（RestTemplate/OpenFeign 等）
8. 启用虚拟线程进行性能测试
```

## 注意事项

1. **Spring Cloud 2025.x 兼容**：Spring Boot 4.x 需配合 Spring Cloud 2025.0.x（即 `Utopia` 版本线），旧版本 Spring Cloud 不兼容
2. **Thymeleaf 3.1.5+**：模板引擎需升级
3. **Groovy 5.0.5+**：如果使用 Groovy 脚本，需升级
4. **Kotlin 2.x+**：Kotlin 版本需同步升级
5. **Docker Compose 支持增强**：Boot 4.x 对 Artemis/ActiveMQ 等镜像的 Compose 支持已修复
6. **Actuator 端点变化**：`management.endpoints.web.exposure.include` 配置方式不变，但个别端点路径可能调整
7. **DevTools 变更**：DevTools 的 Restarter 不再支持无参 main 方法
8. **SSL 配置**：`@Ssl` 注解在 `@Bean` 方法中配合 `@ServiceConnection` 使用时需注意兼容性
