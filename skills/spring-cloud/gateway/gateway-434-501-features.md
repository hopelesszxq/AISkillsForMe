---
name: gateway-434-501-features
description: Spring Cloud Gateway 4.3.4 & 5.0.x 新特性：Forwarded Header、MVC CircuitBreaker、StripContextPath、CodecCustomizer
tags: [spring-cloud, gateway, circuit-breaker, mvc, rfc7239, strip-context-path, codec]
---

## 概述

Spring Cloud Gateway 4.3.4（2026-04-01）和 5.0.x 系列分别针对 4.3.x 和 5.0.x 系列进行了 Bug 修复和功能增强。5.0.x 作为 SC2025 之后的版本，引入了一些重要新特性。

## Gateway 5.0.x 新特性

### 1. Forwarded Header RFC 7239 支持

Gateway 5.0.1 新增了对 `Forwarded` 头中 `by` 参数的支持（RFC 7239 标准）：

```yaml
spring:
  cloud:
    gateway:
      forwarded:
        enabled: true  # 开启 Forwarded Header 处理
```

当上游服务需要识别原始客户端信息时，Gateway 会自动生成符合 RFC 7239 的 `Forwarded` 头：

```
Forwarded: by=gateway-id;for=192.168.1.100;host=api.example.com;proto=https
```

### 2. MVC CircuitBreaker 增强

新增两个关键行为支持：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: resilient-service
          uri: http://backend:8080
          predicates:
            - Path=/api/**
          filters:
            - name: CircuitBreaker
              args:
                name: myCircuitBreaker
                fallbackUri: forward:/fallback
                notPermittedBehavior: THROTTLE  # 新增：未允许请求的处理方式
                resumeWithoutError: true          # 新增：无错误时恢复
```

**`notPermittedBehavior`** 支持值：
- `THROTTLE` — 对未允许的请求限流降级（默认）
- `FAIL_FAST` — 直接快速失败

**`resumeWithoutError`**：当断路器从 HALF_OPEN 检测到无错误时，是否恢复通行。

### 3. 毫秒级时间谓词（WebMVC）

WebMVC 模式下支持毫秒级时间戳作为时间谓词参数：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: time-based-route
          uri: http://backend:8080
          predicates:
            - After=1749398400000  # 毫秒时间戳
            - Before=1749484800000
```

支持 epoch millisecond 值，兼容 Java 8+ 的 `Instant.ofEpochMilli()`。

## Gateway 4.3.x Bugfix

### v4.3.4 修复

- **SetRequestUriGatewayFilterFactory 快捷配置失效**：修复 shortcut 配置方式无法设置请求 URI 的问题

```yaml
# 修复前可能不生效
filters:
  - SetRequestUri=/custom-path

# 修复后正常
filters:
  - name: SetRequestUri
    args:
      uri: /custom-path
```

### v4.3.3 修复

- **URL 编码参数处理**：修复 FormFilter 中 URL 编码参数处理错误

## Gateway 5.0.2 新特性（2026-06-11）

### 1. StripContextPath 过滤器

当 WebMVC Gateway 配置了 `server.servlet.context-path` 时，`StripPrefix` 和 `RewritePath` 过滤器操作的是包含 context-path 的完整 URI，而 `Path` 路由谓词匹配的是去掉 context-path 后的路径，导致行为不一致。

**新增 `StripContextPath` 过滤器**，在路径操作前先剥离 context-path：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: app-service
          uri: http://backend:8080
          predicates:
            - Path=/myapp/api/**
          filters:
            - StripContextPath  # ✅ 先剥离 /myapp 后再处理 StripPrefix
            - StripPrefix=1
```

**工作原理**：

```
请求: /myapp/api/users/123
       │
       ▼ StripContextPath
       /api/users/123          ← 剥离 /myapp 上下文路径
       │
       ▼ StripPrefix=1
       /users/123              ← 正确剥离第一段
```

**注意**：仅在 WebMVC 模式下生效（Reactive 模式无 context-path 概念）。

### 2. CodecCustomizer 支持 Body Filter 编码

Body 过滤器（`ModifyRequestBody`、`ModifyResponseBody`）默认使用框架级 codec 配置。5.0.2 允许通过 `CodecCustomizer` Bean 自定义 body 编码：

```java
@Bean
public CodecCustomizer bodyCodecCustomizer() {
    return configurer -> {
        configurer.customCodecs().register(new Jackson2JsonDecoder());
    };
}
```

**应用场景**：
- 自定义 JSON 序列化/反序列化配置
- 集成 protobuf、avro 等编解码器
- 全局 codec 日志/监控

## Gateway v5.0.x 其他修复

- **ApiVersionHolder 支持**：新增对 ApiVersionHolder 的支持
- **Micrometer Tracing 依赖修正**：Gateway 指标需 `spring-boot-micrometer-tracing` 可选依赖
- **Jackson 2 模块兼容**：修复 Gateway MVC 中 Jackson 2 模块的使用问题

## 注意事项

1. **v4.3.x vs v5.0.x**：v4.3.x 对应 Spring Cloud 2024.x，v5.0.x 对应 SC2025.x。根据 Spring Cloud 版本选择合适的分支
2. **SC2025 迁移注意**：Gateway 5.0.x 默认关闭了 X-Forwarded-* 头自动注入，需显式开启或使用新的 Forwarded 头机制
3. **MVC CircuitBreaker 行为变更**：新增 `notPermittedBehavior` 默认为 `THROTTLE`，升级后需确认对现有降级逻辑无影响
