---
name: spring-boot-41-security-observability
description: Spring Boot 4.1 安全与可观测增强：InetAddress 过滤、证书监控、LazyConnectionDataSourceProxy、OpenTelemetry 环境变量
tags: [tools, spring-boot, security, observability, otel, datasource]
---

## 概述

Spring Boot 4.1.0-RC1（2026-04-23 发布）在安全与可观测方面引入了多项实用增强。本文聚焦于 **InetAddress HTTP 客户端过滤**、**SslMeterBinder 证书监控**、**LazyConnectionDataSourceProxy** 和 **OpenTelemetry SDK 环境变量支持**，这些功能在 `spring-boot-41-features.md` 中未涵盖。

## 一、InetAddress HTTP 客户端过滤（安全）

Spring Boot 4.1 支持对 HTTP 客户端（RestClient/RestTemplate/WebClient）进行 IP 地址范围过滤，防止 SSRF 攻击。

### 配置方式

```yaml
spring:
  http:
    client:
      inet-address-filter:
        # 只允许访问内网 IP 段（白名单模式）
        allowed-addresses: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
        # 或使用黑名单模式（拒绝特定地址）
        # blocked-addresses: 169.254.0.0/16, 127.0.0.0/8
        
        # 默认拒绝所有链路本地地址
        block-link-local: true
        # 默认拒绝回环地址（生产建议开启）
        block-loopback: true
```

### 编程式配置

```java
@Configuration
public class HttpClientSecurityConfig {

    @Bean
    public InetAddressFilter httpClientFilter() {
        return InetAddressFilter.builder()
            .allowAddresses("10.0.0.0/8", "172.16.0.0/12")
            .allowAddresses("192.168.0.0/16")
            .blockLinkLocal(true)
            .blockLoopback(true)
            .build();
    }
}
```

### 使用场景

| 场景 | 建议配置 |
|------|---------|
| 微服务间调用 | 白名单内网段 |
| 外部 API 集成 | 白名单特定公网 IP |
| 通用安全加固 | block-loopback=true + block-link-local=true |

### 注意事项

- **默认仅生效于自动配置的 HTTP 客户端**：自定义 `RestTemplateBuilder` 需手动注入 `InetAddressFilter`
- **不会影响 JDBC 连接**：数据库连接不受此过滤影响
- **日志记录**：被拦截的请求会在 TRACE 级别记录，可用于安全审计

## 二、SslMeterBinder 证书到期监控（可观测）

新增 `SslMeterBinder`，自动监控 TrustStore 中证书的到期时间，暴露 Prometheus 指标。

### 自动生效

```yaml
# 默认已启用，无需额外配置
management:
  endpoints:
    web:
      exposure:
        include: metrics
```

### 暴露的指标

```text
# HELP ssl_certificate_days_remaining 证书剩余有效期天数
# TYPE ssl_certificate_days_remaining gauge
ssl_certificate_days_remaining{alias="my-cert",issuer="CN=MyCA"} 87.0

# HELP ssl_certificate_expiration_timestamp 证书到期时间戳
# TYPE ssl_certificate_expiration_timestamp gauge
ssl_certificate_expiration_timestamp{alias="my-cert"} 1.736e+09
```

### 自定义

```java
@Configuration
public class SslMetricsConfig {

    @Bean
    public SslMeterBinder sslMeterBinder(SslBundleRegistry registry) {
        // 指定要监控的 SslBundle
        return new SslMeterBinder(registry, 
            SslBundleRegistrar.forBundle("server"));
    }
}
```

### Prometheus 告警规则示例

```yaml
groups:
  - name: ssl-certificates
    rules:
      - alert: CertificateExpiringSoon
        expr: ssl_certificate_days_remaining < 30
        for: 1h
        annotations:
          summary: "证书 {{ $labels.alias }} 将在 30 天内到期"
```

## 三、LazyConnectionDataSourceProxy（连接池优化）

Spring Boot 4.1 支持 `LazyConnectionDataSourceProxy`，延迟实际数据库连接创建到第一次 SQL 执行时。

### 启用

```yaml
spring:
  datasource:
    lazy-connection-data-source-proxy: true  # 默认 false
```

### 工作原理

```
传统流程:
  @Transactional 开始 → 立即获取连接 → 业务逻辑 → SQL 执行 → 释放连接
  
LazyConnectionDataSourceProxy:
  @Transactional 开始 → 不获取连接 → 业务逻辑 → SQL 执行时获取连接 → 释放连接
```

### 适用场景

```java
@Service
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;

    @Transactional
    public OrderResult processOrder(String orderId) {
        // 检查缓存 — 不需要数据库连接
        OrderCache cache = cacheService.get(orderId);
        if (cache != null) {
            // 无数据库操作，不会创建连接
            return cache.toResult();
        }
        
        // 只有这里才会真正获取数据库连接
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        return OrderResult.from(order);
    }
}
```

### 注意事项

- **只读事务**：适用于大量只读检查后可能跳过数据库操作的场景
- **Hibernate 多租户兼容**：Spring Framework 7.0.7 修复了与 Hibernate 多租户的兼容性问题
- **不建议与事务传播 REQUIRES_NEW 混用**：可能导致连接获取时机混乱
- **性能影响**：对于每次都执行 SQL 的场景，会有微小额外开销（代理层），建议实测对比

## 四、OpenTelemetry SDK 环境变量支持

Spring Boot 4.1 原生支持 OpenTelemetry SDK 环境变量规范，无需手动配置 OTEL 属性。

```yaml
# application.yml — 通过 Spring 属性自动映射到 OTEL 环境变量
otel:
  sdk:
    disabled: false
  traces:
    exporter: otlp
  metrics:
    exporter: otlp
  logs:
    exporter: otlp
  exporter:
    otlp:
      endpoint: http://otel-collector:4318
      protocol: http/protobuf
  resource:
    attributes:
      service.name: my-application
      service.version: 1.0.0
      deployment.environment: production
```

### 等效的 OTEL 环境变量方式

```bash
# 也可以直接用 OTEL 环境变量（与 Spring 属性等价）
export OTEL_SDK_DISABLED=false
export OTEL_TRACES_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
export OTEL_RESOURCE_ATTRIBUTES=service.name=my-application,service.version=1.0.0
```

### 优先级规则

```
Spring 属性 > OTEL 环境变量 > OTEL SDK 默认值
```

### 迁移建议

```java
// 从手动配置迁移到自动配置
// ❌ 旧方式（Spring Boot 4.0.x 需要手动）
// @Bean
// public OpenTelemetry openTelemetry() {
//     return OpenTelemetrySdk.builder()
//         .setTracerProvider(...)
//         .build();
// }

// ✅ 新方式（4.1 自动装配，只需配置属性）
@Configuration
@ConditionalOnProperty(name = "otel.sdk.disabled", havingValue = "false", matchIfMissing = true)
public class OtelAutoConfig {
    // 无需手动创建 OpenTelemetry 实例
}
```

## 五、依赖坐标

```xml
<!-- Spring Boot 4.1.x BOM -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.1.0-RC1</version>
    <relativePath/>
</parent>
```

## 注意事项

1. **InetAddress 过滤不阻断 gRPC**：gRPC 走 HTTP/2，过滤规则对 `GrpcChannel` 无效
2. **SslMeterBinder 需要 Actuator**：依赖 `spring-boot-starter-actuator`，务必检查是否引入
3. **LazyConnectionDataSourceProxy 禁用场景**：使用 `HikariCP` 连接池的健康检查时，建议保持默认关闭
4. **OpenTelemetry 环境变量**：确保 `otel-exporter-otlp` 相关依赖已添加，否则自动配置不生效
5. **版本状态**：4.1.0-RC1 为里程碑版本，生产环境建议等待 GA 稳定版
