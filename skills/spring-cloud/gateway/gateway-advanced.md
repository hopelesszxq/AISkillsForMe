---
name: gateway-advanced
description: Gateway 高级特性：熔断、限流、自定义过滤器与聚合
tags: [spring-cloud, gateway, circuit-breaker, rate-limit, aggregation]
---

## 熔断降级 (Spring Cloud Circuit Breaker)

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/order/**
          filters:
            - name: CircuitBreaker
              args:
                name: orderServiceCB
                fallbackUri: forward:/fallback/order
            - name: Retry
              args:
                retries: 3
                statuses: BAD_GATEWAY, SERVICE_UNAVAILABLE
```

```java
@RestController
public class FallbackController {
    @GetMapping("/fallback/order")
    public Mono<Map<String, Object>> fallback() {
        return Mono.just(Map.of(
            "code", 503,
            "msg", "服务暂不可用，请稍后重试"
        ));
    }
}
```

## 请求限流 (RequestRateLimiter)

### 基于 Redis 的令牌桶限流

```yaml
filters:
  - name: RequestRateLimiter
    args:
      redis-rate-limiter.replenishRate: 10   # 每秒令牌数
      redis-rate-limiter.burstCapacity: 20   # 突发容量
      redis-rate-limiter.requestedTokens: 1  # 每次请求消耗
      key-resolver: "#{@userKeyResolver}"
```

```java
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> {
        String userId = exchange.getRequest().getHeaders()
            .getFirst("X-User-Id");
        return Mono.just(userId != null ? userId : "anonymous");
    };
}
```

## 自定义全局过滤器

```java
@Component
@Order(-1)
public class RequestLoggingFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        long start = System.currentTimeMillis();
        
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            long elapsed = System.currentTimeMillis() - start;
            log.info("{} {} -> {} [{}ms]",
                request.getMethod(), request.getURI(),
                exchange.getResponse().getStatusCode(), elapsed);
        }));
    }
}
```

## 网关聚合 (后端合并请求)

```java
@Component
public class AggregationFilter implements GlobalFilter, Ordered {
    private final WebClient webClient = WebClient.builder()
        .codecs(c -> c.defaultCodecs().maxInMemorySize(16 * 1024 * 1024))
        .build();

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();
        if (!path.startsWith("/api/aggregate")) {
            return chain.filter(exchange);
        }

        Mono<Map> orderMono = webClient.get()
            .uri("lb://order-service/api/order/user/{uid}", getUserId(exchange))
            .retrieve().bodyToMono(Map.class);
        Mono<Map> userMono = webClient.get()
            .uri("lb://user-service/api/user/{uid}", getUserId(exchange))
            .retrieve().bodyToMono(Map.class);
        
        return Mono.zip(orderMono, userMono)
            .map(tuple -> {
                Map<String, Object> result = new HashMap<>();
                result.put("orders", tuple.getT1());
                result.put("user", tuple.getT2());
                return result;
            })
            .flatMap(result -> writeJsonResponse(exchange, result));
    }
}
```

## 跨域配置

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins: "https://admin.example.com"
            allowedMethods: "*"
            allowedHeaders: "*"
            allowCredentials: true
            maxAge: 3600
```

## 注意事项

- **WebFlux 环境**：Gateway 基于 Netty + WebFlux，不要引入 Tomcat/Servlet 依赖
- **Feign 不可用**：内部服务调用请使用 `WebClient`（Reactive）或 `RestTemplate`（需配置）
- **过滤器顺序**：`@Order(-1)` 优先级最高；自定义 GatewayFilter 通过 `getOrder()` 控制
- **限流 Redis**：RequestRateLimiter 必须在 classpath 中有 `spring-boot-starter-data-redis-reactive`
- **超时配置**：CircuitBreaker 配合超时使用，避免慢调用阻塞网关线程
- **路由动态刷新**：可集成 Nacos 配置中心，结合 `RouteDefinitionRepository` 实现不停机更新
