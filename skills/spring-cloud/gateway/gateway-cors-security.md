---
name: gateway-cors-security
description: Spring Cloud Gateway CORS 跨域配置与安全防护最佳实践（安全响应头、CSRF、请求校验）
tags: [spring-cloud, gateway, cors, security, headers, csrf]
---

## 概述

Spring Cloud Gateway 作为微服务统一入口，CORS 配置和安全响应头是生产环境的必备能力。错误配置会导致跨域请求被拦截或产生安全漏洞。本文覆盖 Gateway 中的 CORS 全局/路由级配置、安全响应头、CSRF 防护和请求校验。

## CORS 配置

### 全局 CORS（推荐）

全局配置适用于所有路由：

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            # 生产环境应限定具体域名
            allowed-origins:
              - "https://admin.example.com"
              - "https://app.example.com"
            allowed-methods:
              - GET
              - POST
              - PUT
              - DELETE
              - OPTIONS
            allowed-headers:
              - Authorization
              - Content-Type
              - X-Requested-With
              - Accept
            exposed-headers:
              - X-Total-Count
              - X-Pagination
            allow-credentials: true
            max-age: 3600
```

### 路由级 CORS

不同路由配置不同的跨域策略：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: admin-service
          uri: lb://admin-service
          predicates:
            - Path=/api/admin/**
          filters:
            - name: DedupeResponseHeader
              args:
                strategy: RETAIN_FIRST
            # 路由级 CORS 通过自定义过滤器实现
```

### Java 配置方式

```java
@Configuration
public class GatewayCorsConfig {

    @Bean
    public CorsWebFilter corsWebFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOriginPatterns(List.of("*"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);

        return new CorsWebFilter(source);
    }
}
```

## 安全响应头

Gateway 应统一添加安全响应头，防范常见 Web 攻击：

### 全局过滤器添加安全头

```java
@Component
public class SecurityHeadersGlobalFilter implements GlobalFilter, Ordered {

    private static final Map<String, String> SECURITY_HEADERS = Map.of(
        "X-Content-Type-Options", "nosniff",
        "X-Frame-Options", "DENY",
        "X-XSS-Protection", "1; mode=block",
        "Strict-Transport-Security", "max-age=31536000; includeSubDomains",
        "Referrer-Policy", "no-referrer-when-downgrade",
        "Permissions-Policy", "camera=(), microphone=(), geolocation=()"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpResponse response = exchange.getResponse();
        SECURITY_HEADERS.forEach(response.getHeaders()::set);
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -1; // 最先执行
    }
}
```

### 动态 Content-Security-Policy

```java
// 根据路由动态设置 CSP
@Component
public class CspHeaderFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();
        String csp;
        if (path.startsWith("/api/admin")) {
            csp = "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'";
        } else {
            csp = "default-src 'self'; script-src 'self'";
        }
        exchange.getResponse().getHeaders().set("Content-Security-Policy", csp);
        return chain.filter(exchange);
    }
}
```

## 常见坑点

### 1. allowCredentials 与 allowedOrigins 冲突

```yaml
# ❌ 错误：allowCredentials=true 时不能使用 "*"
allowed-origins: "*"
allow-credentials: true

# ✅ 正确：使用具体域名或 allowedOriginPatterns
allowed-origins: "https://app.example.com"
allow-credentials: true
```

`allowCredentials(true)` 与 `allowedOrigins("*")` 同时使用会导致浏览器拒绝请求，报错：

```
The value of the 'Access-Control-Allow-Origin' header must not be the wildcard '*'
```

### 2. OPTIONS 预检请求处理

Gateway 默认处理 OPTIONS 请求，但需要注意：

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        add-to-simple-url-handler-mapping: true  # 确保预检请求能被正确处理
```

### 3. 重复 CORS 头

当下游服务也配置了 CORS 时，响应头可能重复：

```yaml
filters:
  - name: DedupeResponseHeader
    args:
      name: Access-Control-Allow-Origin Access-Control-Allow-Credentials
      strategy: RETAIN_FIRST
```

### 4. 生产环境不允许开放所有来源

```yaml
# ❌ 生产环境禁止
allowed-origins: "*"

# ✅ 限定为实际使用的域名
allowed-origins:
  - "https://app.example.com"
  - "https://admin.example.com"
```

## 请求校验过滤器

防止恶意请求进入后端服务：

```java
@Component
public class RequestValidationFilter implements GlobalFilter {

    private static final Pattern SQL_INJECTION = 
        Pattern.compile("(?i)(\\b(select|insert|update|delete|drop|alter|union|exec)\\b)");
    
    private static final Pattern XSS_PATTERN = 
        Pattern.compile("(?i)(<script|<iframe|onerror|onclick|javascript:)");

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String query = exchange.getRequest().getURI().getQuery();
        if (query != null) {
            if (SQL_INJECTION.matcher(query).find() || XSS_PATTERN.matcher(query).find()) {
                exchange.getResponse().setStatusCode(HttpStatus.BAD_REQUEST);
                return exchange.getResponse().setComplete();
            }
        }
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return 0;
    }
}
```

## 完整配置示例

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        add-to-simple-url-handler-mapping: true
        cors-configurations:
          '[/**]':
            allowed-origins:
              - "https://app.example.com"
            allowed-methods: "*"
            allowed-headers: "*"
            allow-credentials: true
            max-age: 3600
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Origin Access-Control-Allow-Credentials, RETAIN_FIRST
      routes:
        - id: api-service
          uri: lb://api-service
          predicates:
            - Path=/api/**
          filters:
            - StripPrefix=1
```

## 检查清单

- [ ] 生产环境 CORS 来源已限定具体域名
- [ ] 安全响应头已通过全局过滤器添加
- [ ] 预检请求正常响应（OPTIONS）
- [ ] 无重复 CORS 头问题
- [ ] 已配置 CSP 策略
- [ ] 大请求体大小已限制
- [ ] SQL 注入/XSS 基本校验已配置
