---
name: gateway-resilience-observability
description: Spring Cloud Gateway 韧性 + 安全 + 可观测性实战：Resilience4j、OAuth2、OpenTelemetry
tags: [spring-cloud, gateway, resilience4j, oauth2, opentelemetry, observability, circuit-breaker]
---

## 概述

基于 **Spring Boot 3.5 + Spring Cloud Gateway + Java 25** 的现代 API 网关架构，集成 Resilience4j 熔断降级、Spring Security OAuth2 认证授权、Micrometer + OpenTelemetry 可观测性。适用于生产级微服务网关。

## 1. 韧性模式（Resilience4j）

### 熔断器（Circuit Breaker）

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: book-service
          uri: lb://book-service
          predicates:
            - Path=/books/**
          filters:
            - name: CircuitBreaker
              args:
                name: bookServiceCircuitBreaker
                fallbackUri: forward:/fallback/books
```

```yaml
# application.yml - Resilience4j 配置
resilience4j:
  circuitbreaker:
    configs:
      default:
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
        recordExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
```

### 重试（Retry）

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: book-service
          filters:
            - name: Retry
              args:
                retries: 3
                statuses: BAD_GATEWAY, SERVICE_UNAVAILABLE
                methods: GET
                backoff:
                  firstBackoff: 500ms
                  maxBackoff: 5s
                  factor: 2
```

### 限流（Rate Limiter）

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: book-service
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter:
                  replenishRate: 100
                  burstCapacity: 200
                  requestedTokens: 1
                keyResolver: "#{@userKeyResolver}"
```

```java
// 自定义 KeyResolver：按用户或 IP 限流
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> {
        // 优先使用认证用户，否则按 IP
        String userId = exchange.getPrincipal()
            .map(p -> p.getName())
            .blockOptional()
            .orElse(exchange.getRequest().getRemoteAddress().getHostString());
        return Mono.just(userId);
    };
}
```

## 2. OAuth2 安全认证

### 网关作为 OAuth2 Resource Server

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/spring-gateway
```

```java
// SecurityConfig.java - 安全配置
@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
            .authorizeExchange(exchanges -> exchanges
                .pathMatchers("/login/**", "/oauth2/**", "/public/**").permitAll()
                .pathMatchers(HttpMethod.GET, "/books/**").hasAuthority("SCOPE_book:read")
                .pathMatchers(HttpMethod.POST, "/books/**").hasAuthority("SCOPE_book:write")
                .anyExchange().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults())
            )
            .build();
    }
}
```

### Redis 会话管理

```yaml
spring:
  session:
    store-type: redis
    timeout: 30m
  data:
    redis:
      host: localhost
      port: 6379
```

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

## 3. 可观测性（OpenTelemetry + Micrometer）

### 依赖

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
<dependency>
    <groupId>net.ttddyy.observation</groupId>
    <artifactId>datasource-micrometer-spring-boot</artifactId>
</dependency>
```

### 配置 OTLP 导出

```yaml
management:
  tracing:
    sampling:
      probability: 1.0
  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces
  metrics:
    export:
      otlp:
        enabled: true
        endpoint: http://localhost:4318/v1/metrics
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
```

### 自定义观测（Gateway 过滤器链路追踪）

```java
@Component
public class TracingGatewayFilter implements GlobalFilter, Ordered {

    private final Tracer tracer;

    public TracingGatewayFilter(Tracer tracer) {
        this.tracer = tracer;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();
        return chain.filter(exchange)
            .doOnSuccess(v -> tracer.currentSpan()
                .ifPresent(span -> span.setAttribute("gateway.path", path)));
    }

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE - 10; // 在最后执行
    }
}
```

### 开发环境一键可观测（Arconia Dev Services）

使用 [Arconia OpenTelemetry](https://docs.arconia.io) 自动启动 Grafana LGTM 栈（Loki + Grafana + Tempo + Mimir）：

```shell
# 自动启动完整的可观测性平台
arconia dev

# 控制台会输出 Grafana 访问地址
# Access to the Grafana dashboard: http://localhost:38125
```

## 4. 完整项目结构

```
edge-service/                    # 网关服务
├── build.gradle
├── src/main/java/
│   ├── config/
│   │   └── SecurityConfig.java  # OAuth2 安全配置
│   ├── filter/
│   │   ├── TracingGatewayFilter.java
│   │   └── UserKeyResolver.java
│   └── Application.java
└── src/main/resources/
    └── application.yml          # Gateway 路由 + Resilience4j + OTel
```

## 注意事项

1. **Gateway 本身是响应式（WebFlux）**：所有过滤器、安全配置必须使用响应式 API（`Mono`/`Flux`），不能用 Servlet 注解
2. **熔断 fallback URI 必须不走 Gateway 路由**：`fallbackUri: forward:/fallback/books` 是本地 forward，不是在 Gateway 内再次路由
3. **OpenTelemetry 版本兼容**：Spring Boot 3.5 内置 Micrometer 1.15+，对应的 OTel exporter 版本需要匹配（推荐使用 Spring Boot BOM 管理版本）
4. **Redis Rate Limiter 依赖 Redis**：生产环境需确保 Redis 高可用，否则限流功能不可用
5. **OAuth2 退出登录**：需额外配置 `logoutHandler` 清除 Redis 中的 session
