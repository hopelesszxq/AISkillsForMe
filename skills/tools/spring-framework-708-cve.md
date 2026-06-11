---
name: spring-framework-708-cve
description: Spring Framework 7.0.8 安全更新：16 个 CVE 修复详情与升级指南
tags: [tools, spring-framework, security, cve, vulnerability, websocket, spel, webflux]
---

## 概述

Spring Framework 7.0.8（2026-06-08 发布）是一次**大规模安全维护版本**，一次性修复了 **16 个 CVE 漏洞**。这是 2026 年 AI 安全扫描浪潮（482 份安全报告/月）后的首次集中安全补丁发布。

受影响的版本：Spring Framework 6.x / 7.0.x（低于 7.0.8 的所有版本）

对应的 Spring Boot 版本：Spring Boot 4.1.0（2026-06-10）已包含此修复。

## 16 个 CVE 分类详解

### WebSocket 安全问题（2个）

| CVE | 类型 | 严重程度 | 描述 |
|-----|------|---------|------|
| CVE-2026-41838 | 可预测 Session ID | 中 | WebSocket 模块使用可预测的 Session ID，攻击者可劫持连接 |
| CVE-2026-41839 | Session Fixation | 高 | WebFlux 中可通过 Session Fixation 实现权限提升 |

**修复**：WebSocket Session ID 改用 `JdkIdGenerator`（UUID 随机生成）。

### WebFlux 安全问题（2个）

| CVE | 类型 | 严重程度 | 描述 |
|-----|------|---------|------|
| CVE-2026-41840 | 拒绝服务 | 中 | Multipart 请求导致 DoS |
| CVE-2026-41841 | 信息泄露 | 中 | 静态资源缓存泄露敏感信息（MVC + WebFlux） |

### 静态资源安全问题（3个，MVC + WebFlux 均受影响）

| CVE | 类型 | 严重程度 | 描述 |
|-----|------|---------|------|
| CVE-2026-41842 | 拒绝服务 | 中 | 版本化资源请求导致 DoS |
| CVE-2026-41843 | 路径遍历 | 高 | 版本化静态资源存在路径穿越漏洞 |
| CVE-2026-41844 | 开放重定向 | 中 | Spring MVC 和 WebFlux 中的 Open Redirect |

**修复**：增加对静态资源路径的安全校验，记录不安全的静态资源位置警告。

### XSS 安全问题（2个）

| CVE | 类型 | 严重程度 | 描述 |
|-----|------|---------|------|
| CVE-2026-41845 | XSS | 高 | `JavaScriptUtils` 中的跨站脚本攻击 |
| CVE-2026-41846 | XSS | 高 | JSP Form Tags 中的跨站脚本攻击 |

### SpEL 表达式引擎安全问题（4个）

| CVE | 类型 | 严重程度 | 描述 |
|-----|------|---------|------|
| CVE-2026-41848 | 拒绝服务 | 中 | `AntPathMatcher` 导致的 DoS |
| CVE-2026-41850 | 算法复杂度 DoS | 高 | SpEL 表达式的算法复杂度拒绝服务 |
| CVE-2026-41851 | 无界缓存 DoS | 中 | SpEL 中无界缓存导致的 DoS |
| CVE-2026-41852 | 任意方法调用 | **严重** | SpEL 表达式中可任意调用方法 |

> **CVE-2026-41852 风险最高**：攻击者可构造恶意 SpEL 表达式执行任意方法调用，需优先修复。

### 其他安全隐患（3个）

| CVE | 类型 | 严重程度 | 描述 |
|-----|------|---------|------|
| CVE-2026-41853 | Multipart 请求走私 | 高 | HTTP Multipart 请求走私（MVC + WebFlux） |
| CVE-2026-41854 | SSRF | 高 | `UriComponentsBuilder` 服务端请求伪造 |
| CVE-2026-41855 | 不安全反序列化 | **严重** | Jackson JMS Converter 的不安全反序列化 |

> **CVE-2026-41855 风险极高**：通过 Jackson JMS 消息转换器可实现 RCE（远程代码执行）。

## 升级指南

### 1. Maven 依赖更新

```xml
<!-- Spring Framework BOM -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-framework-bom</artifactId>
    <version>7.0.8</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>

<!-- 或使用 Spring Boot BOM（已包含） -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>4.1.0</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

### 2. Gradle 依赖更新

```kotlin
// Spring Framework
implementation(platform("org.springframework:spring-framework-bom:7.0.8"))

// 或 Spring Boot（推荐）
implementation(platform("org.springframework.boot:spring-boot-dependencies:4.1.0"))
```

### 3. 受影响版本范围

| 框架分支 | 受影响版本 | 修复版本 |
|---------|-----------|---------|
| Spring Framework 7.0.x | < 7.0.8 | **7.0.8** |
| Spring Framework 6.2.x | < 6.2.19 | **6.2.19** |
| Spring Framework 6.1.x | < 6.1.23 | **6.1.23** |

## 防护建议

### 1. 立即行动优先级

```
1️⃣ 升级 Spring Framework 到 7.0.8 / 6.2.19（今天）
2️⃣ 如果无法立即升级，先修复 CVE-2026-41855（不安全反序列化）
3️⃣ 再修复 CVE-2026-41852（SpEL 任意方法调用）
4️⃣ 检查 WebSocket / WebFlux 端点是否暴露
```

### 2. 临时缓解措施（无法升级时）

```java
// 限制 SpEL 表达式评估
@Bean
public StandardEvaluationContext customEvaluationContext() {
    StandardEvaluationContext context = new StandardEvaluationContext();
    // 限制可访问的方法
    context.setMethodResolvers(new ReflectiveMethodResolver(false));
    return context;
}

// 禁用 Jackson JMS 中的默认反序列化
@Bean
public MappingJackson2MessageConverter jacksonJmsConverter() {
    MappingJackson2MessageConverter converter = new MappingJackson2MessageConverter();
    converter.setDeserializationType(MessageType.JSON);
    // 仅允许受信任的类型
    converter.setAllowedListPatterns(Arrays.asList("com.example.**"));
    return converter;
}
```

### 3. 安全配置检查

```yaml
# application.yaml
spring:
  web:
    resources:
      # 避免暴露敏感静态资源
      static-locations: classpath:/static/, classpath:/public/
  mvc:
    # 限制版本化资源访问
    static-path-pattern: /**
```

### 4. 验证升级是否成功

```bash
# 检查 Spring Framework 版本
mvn dependency:tree | grep spring-core

# 输出应显示 7.0.8
# org.springframework:spring-core:jar:7.0.8:compile
```

## 相关链接

- [Spring Framework 7.0.8 Release](https://github.com/spring-projects/spring-framework/releases/tag/v7.0.8)
- [Spring Security Advisories](https://spring.io/security)
- [Spring and Security In The Times Of AI](https://spring.io/blog/2026/06/01/spring_and_security_in_the_times_of_ai)
- Spring Boot 4.1.0 已包含此安全更新
