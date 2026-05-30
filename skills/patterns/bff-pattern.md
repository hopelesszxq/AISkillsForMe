---
name: bff-pattern
description: BFF（Backend for Frontend）模式：Spring Cloud Gateway 实现多端网关隔离与聚合
tags: [patterns, bff, gateway, microservice, frontend]
---

## 概述

BFF（Backend for Frontend）模式是微服务架构中的一种 API 网关模式——**为每种前端客户端（Web、Mobile、桌面）提供独立的后端网关**。核心思想：不要让移动端和桌面端共享同一个后端 API，而是各自有专属的适配层。

### 为什么需要 BFF

| 问题 | 单网关方案 | BFF 方案 |
|------|-----------|---------|
| 移动端需要聚合数据 | 网关臃肿，if/else 分支多 | 移动 BFF 按需聚合，精简响应 |
| Web 端需要大屏数据 | 移动端被迫下载 Web 的数据 | Web BFF 独立返回完整数据 |
| 不同端认证方式不同 | 网关耦合多种认证 | 每个 BFF 独立管理认证策略 |
| 前端团队独立部署 | 网关变更需要协调 | BFF 可独立迭代、独立发布 |

## 架构设计

```
                    ┌──────────────┐
                    │   Frontend   │
                    │ (Web/Mobile) │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  BFF Layer   │  ← 每个端独立的 BFF 服务
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Order    │ │ User     │ │ Product  │
        │ Service  │ │ Service  │ │ Service  │
        └──────────┘ └──────────┘ └──────────┘
```

## Spring Cloud Gateway 实现 BFF

### 1. 多 BFF 路由配置

```yaml
# application-web-bff.yml (Web 端 BFF)
spring:
  cloud:
    gateway:
      routes:
        # Web 端需要完整订单 + 商品详情
        - id: web-order-aggregation
          uri: lb://web-bff-service
          predicates:
            - Path=/api/web/orders/**
          filters:
            - name: CircuitBreaker
              args:
                name: webOrderCircuitBreaker
                fallbackUri: forward:/fallback/orders
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
```

```yaml
# application-mobile-bff.yml (移动端 BFF)
spring:
  cloud:
    gateway:
      routes:
        # 移动端只需要精简订单摘要
        - id: mobile-order-aggregation
          uri: lb://mobile-bff-service
          predicates:
            - Path=/api/mobile/orders/**
          filters:
            - name: CircuitBreaker
              args:
                name: mobileOrderCircuitBreaker
                fallbackUri: forward:/fallback/orders-mobile
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 200  # 移动端更高 QPS
```

### 2. Nacos 多端配置隔离

```yaml
# Nacos 中每个 BFF 独立的配置文件
# Data ID: web-bff-service-dev.yaml
# Data ID: mobile-bff-service-dev.yaml

# 在微服务中激活不同 BFF
spring:
  application:
    name: ${BFF_TYPE:web-bff-service}  # 通过环境变量切换
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
```

### 3. Web BFF 聚合服务代码

```java
@RestController
@RequestMapping("/api/web/orders")
public class WebBffController {

    @Autowired
    private WebClient webClient;

    @GetMapping("/{orderId}")
    public Mono<WebOrderDetailVO> getOrderDetail(@PathVariable String orderId) {
        // Web BFF：聚合订单 + 商品详情 + 物流信息（大屏展示）
        Mono<OrderDTO> order = webClient.get()
            .uri("lb://order-service/api/orders/{id}", orderId)
            .retrieve()
            .bodyToMono(OrderDTO.class);

        Mono<ProductDTO> product = webClient.get()
            .uri("lb://product-service/api/products/{id}", orderId)
            .retrieve()
            .bodyToMono(ProductDTO.class);

        Mono<LogisticsDTO> logistics = webClient.get()
            .uri("lb://logistics-service/api/logistics/{id}", orderId)
            .retrieve()
            .bodyToMono(LogisticsDTO.class);

        return Mono.zip(order, product, logistics)
            .map(tuple -> WebOrderDetailVO.aggregate(
                tuple.getT1(), tuple.getT2(), tuple.getT3()
            ));
    }
}
```

### 4. Mobile BFF 精简服务

```java
@RestController
@RequestMapping("/api/mobile/orders")
public class MobileBffController {

    @Autowired
    private WebClient webClient;

    @GetMapping("/{orderId}")
    public Mono<MobileOrderBriefVO> getOrderBrief(@PathVariable String orderId) {
        // Mobile BFF：只返回订单状态和金额（流量敏感）
        return webClient.get()
            .uri("lb://order-service/api/orders/{id}/brief", orderId)
            .retrieve()
            .bodyToMono(MobileOrderBriefVO.class)
            .timeout(Duration.ofSeconds(2));  // 移动端超时更短
    }
}
```

## BFF + OAuth2 令牌中继

### Gateway 统一认证

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: web-bff
          uri: lb://web-bff-service
          predicates:
            - Path=/api/web/**
          filters:
            - TokenRelay  # 将 OAuth2 令牌传递给下游
```

```java
@Configuration
@EnableWebFluxSecurity
public class BffSecurityConfig {

    @Bean
    public SecurityWebFilterChain webBffSecurity(ServerHttpSecurity http) {
        return http
            .authorizeExchange(exchanges -> exchanges
                .pathMatchers("/api/web/**").authenticated()
                .pathMatchers("/api/mobile/**").permitAll()  // 移动端用 API Key
            )
            .oauth2Login(Customizer.withDefaults())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

### 移动端 BFF 使用 JWT 认证

```java
@Component
public class MobileAuthFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String apiKey = exchange.getRequest().getHeaders()
            .getFirst("X-API-Key");

        if (apiKey == null || !validateApiKey(apiKey)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        // 验证通过后注入用户 ID
        exchange.getAttributes().put("userId", extractUserId(apiKey));
        return chain.filter(exchange);
    }
}
```

## BFF 注意事项

1. **不要过度拆分**：通常只需要 2-3 个 BFF（Web、Mobile、第三方 API），每个端一个足矣
2. **BFF 不做业务逻辑**：BFF 只做数据聚合、格式转换、裁剪，不做业务判断
3. **超时策略差异化**：移动端网络不稳定，超时应比 Web 端短（2s vs 5s）
4. **缓存策略不同**：移动端缓存更激进（离线场景），Web 端缓存更保守
5. **BFF 团队归属**：BFF 应该归前端团队维护，而不是后端团队——这是 BFF 的核心价值
6. **避免 BFF 变成臃肿网关**：每个 BFF 保持轻量，超过 500 行就考虑拆模块
7. **版本管理**：BFF 接口版本与前端版本挂钩，建议 `/api/{bff-type}/v1/...`
8. **错误处理**：BFF 对下游服务错误做降级，不要透传 500 给前端
