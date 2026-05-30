---
name: gateway-sc2025-migration
description: Spring Cloud 2025.0.0 Gateway 重大变更：模块重命名、配置前缀迁移、X-Forwarded-* 默认关闭
tags: [spring-cloud, gateway, migration, breaking-change, upgrade]
---

## 概述

Spring Cloud 2025.0.0 对 Gateway 模块进行了**重大重构**，包括模块/Starter 重命名、配置属性前缀变更、安全默认值调整。从 2024.x 升级时需要仔细处理这些变更。

## 一、模块与 Starter 重命名

旧名已废弃，使用会输出日志警告。

| 弃用的 Artifact | 新 Artifact |
|------|-------|
| spring-cloud-gateway-server | spring-cloud-gateway-server-webflux |
| spring-cloud-gateway-server-mvc | spring-cloud-gateway-server-webmvc |
| spring-cloud-starter-gateway-server | spring-cloud-starter-gateway-server-webflux |
| spring-cloud-starter-gateway-server-mvc | spring-cloud-starter-gateway-server-webmvc |
| spring-cloud-gateway-mvc | spring-cloud-gateway-proxyexchange-webmvc |
| spring-cloud-gateway-webflux | spring-cloud-gateway-proxyexchange-webflux |

### Maven 迁移示例

```xml
<!-- 旧：spring-cloud-gateway-server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-gateway-server-webflux</artifactId>
</dependency>

<!-- 旧：spring-cloud-starter-gateway -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway-server-webflux</artifactId>
</dependency>

<!-- 旧：spring-cloud-gateway-mvc (Proxy Exchange) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-gateway-proxyexchange-webmvc</artifactId>
</dependency>
```

## 二、配置属性前缀变更

所有配置属性前缀已调整以匹配新模块名。

| 模块/Starter | 弃用的前缀 | 新前缀 |
|-------------|-----------|--------|
| spring-cloud-starter-gateway-server-webflux | spring.cloud.gateway.* | spring.cloud.gateway.server.webflux.* |
| spring-cloud-starter-gateway-server-webmvc | spring.cloud.gateway.mvc.* | spring.cloud.gateway.server.webmvc.* |
| spring-cloud-gateway-proxyexchange-webflux | spring.cloud.gateway.proxy.* | spring.cloud.gateway.proxy-exchange.webflux.* |
| spring-cloud-gateway-proxyexchange-webmvc | spring.cloud.gateway.proxy.* | spring.cloud.gateway.proxy-exchange.webmvc.* |

### 使用 Properties Migrator 辅助迁移

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-properties-migrator</artifactId>
    <scope>runtime</scope>
</dependency>
```

启动后会自动打印弃用属性和对应新属性的映射日志。迁移完成后移除该依赖。

### application.yml 示例

```yaml
# 旧配置（2024.x）
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
      globalcors:
        cors-configurations:
          '[/**]':
            allowed-origins: "https://app.example.com"

# 新配置（2025.0.0+）
spring:
  cloud:
    gateway:
      server:
        webflux:
          routes:
            - id: user-service
              uri: lb://user-service
              predicates:
                - Path=/api/user/**
          globalcors:
            cors-configurations:
              '[/**]':
                allowed-origins: "https://app.example.com"
```

## 三、X-Forwarded-* 与 Forwarded 头默认关闭

2025.0.0 开始，Gateway 默认**不信任**任何代理附加的 `X-Forwarded-*` 和 `Forwarded` 请求头，需显式配置可信来源。

### 配置可信代理

```properties
# Spring Cloud Gateway Server WebFlux
spring.cloud.gateway.server.webflux.trusted-proxies=10\\.0\\.0\\..*

# Spring Cloud Gateway Server WebMVC
spring.cloud.gateway.server.webmvc.trusted-proxies=10\\.0\\.0\\..*
```

### 多可信 IP 段

```yaml
spring:
  cloud:
    gateway:
      server:
        webflux:
          trusted-proxies: |
            10\\.0\\.0\\..*,
            172\\.16\\.0\\..*,
            192\\.168\\.1\\..*
```

⚠️ 不配置此项时，客户端可以伪造 `X-Forwarded-For` 等头信息，导致错误的内网 IP 判断。

## 四、升级检查清单

- [ ] 更新 `pom.xml` / `build.gradle` 中的 Gateway 依赖 artifactId
- [ ] 添加 `spring-boot-properties-migrator` 运行时依赖
- [ ] 启动应用，检查日志中的弃用属性警告
- [ ] 按新前缀更新 `application.yml` 中的所有 Gateway 配置
- [ ] 添加 `trusted-proxies` 配置以启用 X-Forwarded-* 头
- [ ] 确认所有自定义 Gateway Filter/GlobalFilter 兼容性
- [ ] 移除 `spring-boot-properties-migrator` 依赖

## 注意事项

- 新 artifact name 和 旧 artifact name **可以同时存在**于 classpath，但建议尽快统一为新名
- Proxy Exchange 模块（`spring-cloud-gateway-mvc` → `spring-cloud-gateway-proxyexchange-webmvc`）适用于非响应式场景（Servlet stack）
- WebFlux 模块适用于响应式场景（默认，基于 Netty）
- 旧前缀在当前版本仍受支持，但会在未来版本移除
