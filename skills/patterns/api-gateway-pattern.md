---
name: api-gateway-pattern
description: API 网关模式深度解析：网关分层架构、路由策略、限流熔断、请求转换与网关选型对比
tags: [patterns, api-gateway, microservice, architecture, routing]
---

## 网关的核心职责

| 职责 | 说明 | 典型实现 |
|------|------|----------|
| **路由转发** | 根据请求路径、Header 路由到后端服务 | Spring Cloud Gateway, Kong |
| **认证鉴权** | 统一拦截 JWT/OAuth2/API Key | Keycloak, Spring Security |
| **限流熔断** | 保护后端不被流量冲垮 | Sentinel, Resilience4j |
| **请求/响应转换** | Header 改写、协议转换（gRPC→HTTP） | Envoy, Apisix |
| **观测性** | 全链路追踪、日志聚合、Metrics | OpenTelemetry, Prometheus |
| **负载均衡** | 多实例分发 | RoundRobin, LeastConn, Consistent Hash |

## 一、网关分层架构（L1-L4）

```
        ┌─────────────────────────────────┐
        │        L1: 入口网关 (Edge)       │  ← 面向公网，TLS termination, WAF
        │   Kong / Nginx + Lua / Cloudflare │
        └──────────────┬──────────────────┘
        ┌──────────────┴──────────────────┐
        │      L2: 业务网关 (Domain)        │  ← 面向内部业务域
        │   Spring Cloud Gateway / Zuul     │
        └──────────────┬──────────────────┘
        ┌──────────────┴──────────────────┐
        │   L3: 微服务网关 (Service Mesh)   │  ← Sidecar 代理
        │   Envoy / Istio / Linkerd        │
        └──────────────┬──────────────────┘
        ┌──────────────┴──────────────────┐
        │         L4: 后端服务              │
        │   user-svc / order-svc / pay-svc │
        └─────────────────────────────────┘
```

## 二、路由策略

### 1. 基于路径路由

```yaml
# Spring Cloud Gateway 配置
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Request-Source, gateway

        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
            - Header=X-Version, v2    # 版本路由
          filters:
            - StripPrefix=1
```

### 2. 基于权重的灰度发布

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service-v1
          uri: lb://user-service-v1
          predicates:
            - Path=/api/users/**
            - Weight=user-group, 90    # 90% 流量
          filters:
            - StripPrefix=1
        - id: user-service-v2
          uri: lb://user-service-v2
          predicates:
            - Path=/api/users/**
            - Weight=user-group, 10    # 10% 流量
          filters:
            - StripPrefix=1
```

### 3. 基于 Header/Query 的高级路由

```yaml
# 租户隔离路由
spring:
  cloud:
    gateway:
      routes:
        - id: tenant-a
          uri: lb://service-tenant-a
          predicates:
            - Path=/api/**
            - Header=X-Tenant-Id, tenant-a
        - id: tenant-b
          uri: lb://service-tenant-b
          predicates:
            - Path=/api/**
            - Header=X-Tenant-Id, tenant-b
```

## 三、限流与熔断策略

### 1. 分布式限流（基于 Redis + Lua）

```java
@Component
public class RedisRateLimiterFilter implements GlobalFilter, Ordered {
    private final RedisTemplate<String, String> redis;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String clientIp = exchange.getRequest().getRemoteAddress().getAddress().getHostAddress();
        String key = "ratelimit:" + clientIp;

        // Lua 脚本：令牌桶
        String lua = """
            local key = KEYS[1]
            local limit = tonumber(ARGV[1])     -- 桶容量
            local rate = tonumber(ARGV[2])       -- 速率
            local now = redis.call('TIME')[1]

            local last = redis.call('GET', key .. ':last') or now
            local tokens = redis.call('GET', key .. ':tokens') or limit

            local elapsed = now - last
            local newTokens = math.min(limit, tokens + elapsed * rate)

            if newTokens >= 1 then
                redis.call('SET', key .. ':tokens', newTokens - 1)
                redis.call('SET', key .. ':last', now)
                return 1
            else
                return 0
            end
        """;

        boolean allowed = redis.execute(
            new DefaultRedisScript<>(lua, Boolean.class),
            List.of(key), "100", "10"  // 容量100，每秒10个
        );

        if (!allowed) {
            exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }
}
```

### 2. 熔断降级

```yaml
# Resilience4j + Spring Cloud Gateway
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
```

## 四、网关选型对比

| 特性 | Spring Cloud Gateway | Kong | Envoy | Apisix |
|------|---------------------|------|-------|--------|
| **语言** | Java/Reactive | Lua+Nginx | C++ | Lua+Nginx |
| **性能** | 中（15K req/s） | 高（30K+） | 极高（100K+） | 高（40K+） |
| **配置方式** | YAML/Java DSL | Admin API + Kong Manager | xDS API | Admin API + Dashboard |
| **热更新** | 不支持（需重启） | 支持（API 调用） | 支持（xDS） | 支持 |
| **动态路由** | 需 Nacos 动态刷新 | 原生支持 | 原生支持 | 原生支持 |
| **服务发现** | Nacos/Eureka/Consul | DB/Consul/DNS | xDS/Consul | DNS/Consul |
| **WAF** | 无内置 | 插件 | LUA Filter | 插件 |
| **gRPC** | 需适配 | 插件 | 原生 | 原生 |
| **运维复杂度** | 低（Spring 体系） | 中 | 高 | 中 |
| **社区** | 活跃（Spring） | 活跃 | CNCF | Apache |

## 五、常见坑点

### 1. 网关成为瓶颈

```yaml
# 解决方案：提升吞吐量
spring:
  codec:
    max-in-memory-size: 16MB    # 默认 256KB，大文件 API 报错
  cloud:
    gateway:
      httpclient:
        connect-timeout: 3000
        response-timeout: 10s
        pool:
          type: elastic          # 弹性连接池
          max-connections: 1000  # 默认 45，明显不够
```

### 2. 全局过滤器的性能陷阱

```java
// ❌ 错误：对静态资源做 JWT 校验（影响性能）
@Bean
public GlobalFilter globalJwtFilter() {
    return (exchange, chain) -> {
        String path = exchange.getRequest().getURI().getPath();
        // 跳过公开路径
        if (path.startsWith("/public/") || path.startsWith("/health")) {
            return chain.filter(exchange);  // 提前返回
        }
        // JWT 校验逻辑...
    };
}
```

### 3. 超时传播

- 网关超时应大于后端服务超时，否则客户端收到 504 而非业务错误
- 建议规则：`客户端超时 > 网关超时 > 后端服务超时`

### 4. 链路追踪 ID 传递

```java
// 网关传递 TraceId，确保全链路可追踪
@Bean
public GlobalFilter traceIdFilter() {
    return (exchange, chain) -> {
        String traceId = Optional.ofNullable(
            exchange.getRequest().getHeaders().getFirst("X-Trace-Id")
        ).orElse(UUID.randomUUID().toString().replace("-", ""));

        exchange.getAttributes().put("traceId", traceId);
        ServerHttpRequest mutated = exchange.getRequest().mutate()
            .header("X-Trace-Id", traceId)
            .build();
        return chain.filter(exchange.mutate().request(mutated).build());
    };
}
```

## 注意事项

1. **网关不是 ESB**：不要把业务逻辑（数据转换、状态编排）放进网关，网关只做路由和横切关注点
2. **避免网关自调用**：微服务间的 RPC 调用不应再经过网关，改用 Feign/Service Mesh
3. **健康检查与优雅下线**：网关必须支持后端服务的优雅下线，避免流量打到已停止的实例
4. **限流参数调优**：生产环境建议根据压测结果设置限流阈值，初始值放宽松再逐步收紧
5. **多级网关安全**：边缘网关做 WAF + DDoS 防护，内部网关做业务鉴权
