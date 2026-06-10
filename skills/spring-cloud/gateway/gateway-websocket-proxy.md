---
name: gateway-websocket-proxy
description: Spring Cloud Gateway WebSocket 代理配置、负载均衡与常见坑点
tags: [spring-cloud, gateway, websocket, load-balancing, proxy]
---

## 概述

Spring Cloud Gateway 原生支持 WebSocket 代理，可通过简单的路由配置将 WebSocket 请求（ws:// / wss://）转发到后端服务。WebSocket 连接是长连接，与 HTTP 短连接的代理有显著差异。

## 路由配置

### 基本 WebSocket 路由

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: websocket-chat
          uri: lb://chat-service
          predicates:
            - Path=/ws/chat/**
          filters:
            - StripPrefix=1
```

关键点：
- **协议**：URI 使用 `lb://` 或 `http://`/`https://`（Gateway 自动升级为 WebSocket）
- **路径断言**：WebSocket 端点路径需要精确匹配
- **StripPrefix**：根据实际后端端点路径决定是否需要

### 带验证的 WebSocket 路由

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: websocket-secure
          uri: lb://notification-service
          predicates:
            - Path=/ws/notifications/**
            - Header=Sec-WebSocket-Protocol, v1
          filters:
            - name: Retry
              args:
                retries: 3
                statuses: BAD_GATEWAY
                methods: GET
```

## 架构原理

```text
客户端 WebSocket       Spring Cloud Gateway       后端服务
    |                       |                       |
    |--- HTTP Upgrade --->  |                       |
    |  (101 Switching)      |                       |
    |<-- 101 OK ---------|                         |
    |                       |--- WebSocket conn --> |
    |                       |   (TCP 透传)          |
    |<=== WebSocket 双向数据流 ====================>|
    |                       |                       |
```

Gateway 在 WebSocket 代理中的角色：
1. 拦截 HTTP Upgrade 请求（101 Switching Protocols）
2. 建立到后端的 WebSocket 连接
3. 在客户端和后端之间透传 WebSocket 帧
4. 连接断开时清理资源

## Java 自定义处理器

```java
@Configuration
public class WebSocketRouteConfig {

    @Bean
    public RouteLocator websocketRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("ws-monitor", r -> r
                .path("/ws/monitor/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .setRequestHeader("X-Proxy", "gateway")
                    // WebSocket 专用的重试过滤器
                    .retry(config -> {
                        config.setRetries(2);
                        config.setStatuses(HttpStatus.BAD_GATEWAY);
                        config.setMethods(HttpMethod.GET);
                    })
                )
                .uri("lb://monitor-service"))
            .build();
    }
}
```

## 负载均衡

WebSocket 是长连接，连接建立后不再重新负载均衡。需要特别注意 Session 亲和性：

### 方案1：基于 IP 的负载均衡

```yaml
spring:
  cloud:
    loadbalancer:
      configurations: health-check
    gateway:
      routes:
        - id: ws-sticky
          uri: lb:ws://sticky-service
          predicates:
            - Path=/ws/sticky/**
          filters:
            # 添加客户端 IP 头，后端可据此做会话亲和
            - AddRequestHeader=X-Forwarded-For, #{remoteAddr}
```

### 方案2：后端 Session 管理（推荐）

后端服务使用分布式 Session + Spring Session，允许多实例任意接收 WebSocket 连接：

```java
// 后端 Spring Boot 配置
@Configuration
@EnableRedisWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(chatHandler(), "/chat")
            .setAllowedOrigins("*")
            // 使用 Redis 存储 Session，支持水平扩展
            .withSockJS();
    }
}
```

## 安全配置

### CORS 配置

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/ws/**]':
            allowed-origins: "https://your-frontend.com"
            allowed-methods: GET
            allow-credentials: true
```

### 认证过滤

```java
@Bean
public RouteLocator securedWebSocketRoutes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("ws-secured", r -> r
            .path("/ws/secured/**")
            .filters(f -> f
                .stripPrefix(1)
                // 在 Upgrade 请求阶段验证 Token
                .filter((exchange, chain) -> {
                    ServerHttpRequest request = exchange.getRequest();
                    String token = request.getHeaders()
                        .getFirst("Sec-WebSocket-Protocol");

                    if (token == null || !validateToken(token)) {
                        exchange.getResponse().setStatusCode(
                            HttpStatus.UNAUTHORIZED);
                        return exchange.getResponse().setComplete();
                    }
                    return chain.filter(exchange);
                })
            )
            .uri("lb://secured-service"))
        .build();
}

private boolean validateToken(String token) {
    // JWT 验证逻辑
    return token != null && token.startsWith("v1.");
}
```

## 配置项调优

```yaml
# application.yml — Gateway WebSocket 相关配置
spring:
  cloud:
    gateway:
      # WebSocket 最大帧大小（默认 65536）
      websocket:
        max-frame-size: 262144  # 256KB
        # 代理 ping/pong 消息（默认 true）
        forward-ping-pong: true
        # 连接空闲超时（毫秒，默认无限制）
        idle-timeout: 300000  # 5分钟

# Netty 全局配置（WebSocket 底层）
server:
  netty:
    max-chunk-size: 256KB
    max-initial-line-length: 8KB
```

## 常见坑点

### 1. WebSocket MVC vs WebFlux

Spring Cloud Gateway **仅支持 WebFlux（响应式）模式**。如果在 `spring-webmvc` 下使用 Gateway，WebSocket 代理不可用。

```xml
<!-- ✅ 正确：Gateway 必须使用 WebFlux -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

<!-- ❌ 不要同时引入 spring-boot-starter-web -->
<!-- <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency> -->
```

### 2. 连接泄漏

WebSocket 连接断开时，Gateway 可能残留未关闭的连接。4.x 版本已修复大多数问题，但仍建议监控连接数：

```yaml
# 启用 Gateway 指标
management:
  endpoints:
    web:
      exposure:
        include: gateway
  endpoint:
    gateway:
      enabled: true
```

```shell
# 查看当前路由连接数
curl -s http://localhost:8080/actuator/gateway/routes | jq '.[] | select(.route_id | contains("ws"))'
```

### 3. 超时配置

Gateway 的默认超时可能不适合 WebSocket 长连接：

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        # WebSocket 不需要 response timeout
        response-timeout: 300s
        # 连接池
        pool:
          type: elastic
          max-idle-time: 600s
```

### 4. 路径冲突

同时存在 HTTP REST 和 WebSocket 路由时，确保路径无歧义：

```yaml
# ✅ 推荐：明确区分路径前缀
- Path=/api/**      → HTTP REST
- Path=/ws/**       → WebSocket

# ❌ 避免：同一路径同时配置 HTTP 和 WebSocket
- Path=/chat/**     → 不确定是 HTTP 还是 WebSocket
```

## 调试技巧

```shell
# 测试 WebSocket 连接
curl -v -H "Connection: Upgrade" \
        -H "Upgrade: websocket" \
        -H "Sec-WebSocket-Version: 13" \
        -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
        -H "Host: localhost:8080" \
        http://localhost:8080/ws/chat/test

# 使用 wscat 测试（推荐）
# npm install -g wscat
wscat -c ws://localhost:8080/ws/chat
> {"type": "ping"}
< {"type": "pong", "ts": 1718000000}
```
