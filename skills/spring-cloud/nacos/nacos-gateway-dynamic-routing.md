---
name: nacos-gateway-dynamic-routing
description: Nacos + Spring Cloud Gateway 动态路由：配置中心管理路由、灰度发布、多环境隔离
tags: [spring-cloud, nacos, gateway, dynamic-route, gray-release, load-balance]
---

## 概述

将 Spring Cloud Gateway 的路由规则托管到 Nacos 配置中心，实现**运行时动态刷新路由**，无需重启网关即可增、删、改路由策略。结合 Nacos 的灰度配置能力和 Gateway 的负载均衡，实现精细化的流量治理。

## 架构

```
                    ┌─────────────┐
                    │   Nacos     │
                    │  Config     │  ← 路由规则存储在 Nacos
                    └──────┬──────┘
                           │ 监听配置变更
                    ┌──────▼──────┐
                    │   Gateway   │  ← 动态刷新路由
                    └──────┬──────┘
                           │ 负载均衡
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Order    │ │ User     │ │ Product  │
        │ Service  │ │ Service  │ │ Service  │
        └──────────┘ └──────────┘ └──────────┘
              │            │            │
              └────────────┼────────────┘
                           ▼
                    ┌──────────────┐
                    │ Nacos        │
                    │ Discovery    │  ← 服务实例注册
                    └──────────────┘
```

## 1. 基础依赖

```xml
<dependencies>
    <!-- Spring Cloud Gateway -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <!-- Nacos 服务发现（支持 lb:// 路由） -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    <!-- Nacos 配置中心（动态路由数据源） -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    </dependency>
    <!-- Actuator（可选，用于查看路由状态） -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependency>
```

## 2. 基础配置

```yaml
# bootstrap.yml（Nacos 配置优先加载）
spring:
  application:
    name: gateway-service
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
        # 路由配置的 Data ID: gateway-service-routes.yaml
        # 监听该配置，变更后自动刷新路由
        shared-configs:
          - data-id: gateway-service-routes.yaml
            group: DEFAULT_GROUP
            refresh: true    # 关键：自动刷新
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: ${NACOS_NAMESPACE:public}

# application.yml
server:
  port: 8080
spring:
  cloud:
    gateway:
      # 注意：不要在 application.yml 写静态路由
      # 所有路由由 Nacos 动态管理
```

## 3. Nacos 中配置动态路由

在 Nacos 控制台创建 Data ID：`gateway-service-routes.yaml`

```yaml
# gateway-service-routes.yaml (Nacos Config)
spring:
  cloud:
    gateway:
      routes:
        # === 订单服务 ===
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/order/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: orderCircuitBreaker
                fallbackUri: forward:/fallback/order

        # === 用户服务 ===
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200

        # === 商品服务（灰度版本） ===
        - id: product-service-v2
          uri: lb://product-service
          predicates:
            - Path=/api/product/**
            - Header=X-Gray-Version, v2
          filters:
            - StripPrefix=1
          metadata:
            version: v2
            gray: true

        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/api/product/**
          filters:
            - StripPrefix=1
          metadata:
            version: v1
```

## 4. 动态路由事件监听

Gateway 默认通过 `@RefreshScope` + Nacos `refresh: true` 自动刷新路由。如果需要自定义刷新逻辑：

```java
@Component
public class DynamicRouteEventListener {

    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;
    @Autowired
    private RouteDefinitionLocator routeDefinitionLocator;

    /**
     * 监听 Nacos 配置变更事件
     */
    @EventListener
    public void onNacosConfigChange(NacosConfigReceivedEvent event) {
        // Nacos 配置已变更，Gateway 的 Routes 会自动刷新
        // 可在此处添加自定义处理（如日志、告警）
        log.info("Nacos config changed, routes will be refreshed: {}", event.getDataId());
    }

    /**
     * 手动重新加载路由
     */
    public void refreshRoutes() {
        routeDefinitionLocator.getRouteDefinitions()
            .flatMap(route -> {
                log.info("Removing route: {}", route.getId());
                return routeDefinitionWriter.delete(Mono.just(route.getId()));
            })
            .then(Mono.defer(() -> {
                log.info("All routes removed, gateway will reload from Nacos");
                return Mono.empty();
            }))
            .subscribe();
    }
}
```

## 5. 基于 Nacos 元数据的灰度路由

利用 Nacos 服务实例的元数据实现灰度发布：

```yaml
# product-service application.yml（灰度实例）
spring:
  cloud:
    nacos:
      discovery:
        metadata:
          version: v2
          gray: "true"
```

```java
/**
 * 灰度路由过滤器：根据请求头 + Nacos 元数据，路由到指定版本的服务
 */
@Component
public class GrayLoadBalanceFilter implements GlobalFilter, Ordered {

    @Autowired
    private LoadBalancerClient loadBalancerClient;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String grayVersion = exchange.getRequest().getHeaders()
            .getFirst("X-Gray-Version");

        if (StringUtils.hasText(grayVersion)) {
            // 根据元数据选择灰度实例
            ServiceInstance instance = loadBalancerClient.choose("product-service",
                new MetadataBasedPredicate(grayVersion));

            if (instance != null) {
                // 改写路由到灰度实例
                exchange.getAttributes().put("serviceInstance", instance);
            }
        }

        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE - 10;
    }
}
```

## 6. 多环境路由隔离

通过 Nacos Namespace 或 Group 实现环境隔离：

```yaml
# bootstrap.yml 按环境加载不同 Namespace
spring:
  cloud:
    nacos:
      config:
        namespace: ${NACOS_NAMESPACE:public}  # dev/test/prod 不同 Namespace
        shared-configs:
          - data-id: gateway-service-routes.yaml
            group: ${NACOS_GROUP:DEFAULT_GROUP}
            refresh: true
```

```text
Nacos Namespace 隔离方案：
├── dev (Namespace)
│   └── gateway-service-routes.yaml     ← 开发环境路由
├── test (Namespace)
│   └── gateway-service-routes.yaml     ← 测试环境路由
└── prod (Namespace)
    └── gateway-service-routes.yaml     ← 生产环境路由
```

## 7. 路由健康状态查看

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: gateway,health,info
  endpoint:
    gateway:
      enabled: true
```

```bash
# 查看当前路由列表
curl http://localhost:8080/actuator/gateway/routes

# 查看某个路由详情
curl http://localhost:8080/actuator/gateway/routes/order-service

# 手动刷新路由（配合自定义 endpoint）
curl -X POST http://localhost:8080/actuator/gateway/refresh
```

## 注意事项

1. **Nacos 配置必须可刷新**：`shared-configs` 中设置 `refresh: true`，否则路由变更不生效
2. **不要在 application.yml 定义静态路由**：否则会和 Nacos 动态路由冲突，导致路由混乱
3. **路由 ID 必须唯一**：重复的 route ID 会导致路由覆盖，生产上要制定命名规范 `{service}-{version}`
4. **Gateway 是 WebFlux 环境**：不要在 Gateway 中使用 Feign、RestTemplate 等阻塞调用，改用 WebClient
5. **灰度发布组合使用**：Nacos 元数据灰度 + Gateway Header 路由 + Sentinel 流量控制，三者配合实现完整灰度
6. **Nacos 配置变更延迟**：Nacos client 默认 3000ms 长轮询间隔，配置变更到路由生效有 3s+ 延迟，对时效性要求高的场景需调优
7. **备份本地路由**：在 application.yml 中保留 fallback 路由配置，防止 Nacos 宕机后 Gateway 无路由可用
