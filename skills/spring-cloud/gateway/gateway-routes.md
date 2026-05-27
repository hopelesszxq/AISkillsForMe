---
name: gateway-routes
description: Spring Cloud Gateway 路由与过滤器
tags: [spring-cloud, gateway, routes]
---

## 路由配置
- 优先级：路由顺序决定匹配顺序
- Predicate 工厂：Path, Header, Method, Query, Cookie, RemoteAddr
- Filter 工厂：AddRequestHeader, StripPrefix, CircuitBreaker, Retry, RequestRateLimiter

## 常见坑
- Gateway 默认使用 Netty，非 Tomcat，注意依赖排除
- WebFlux 环境下不支持 Feign，需用 WebClient
- CORS 配置在 Gateway 层统一处理
