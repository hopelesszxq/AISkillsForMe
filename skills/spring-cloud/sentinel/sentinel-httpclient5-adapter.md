---
name: sentinel-httpclient5-adapter
description: Sentinel 1.8.10 Apache HttpClient 5.x 和 Spring RestClient 适配器集成实战
tags: [spring-cloud, sentinel, httpclient, restclient, circuit-breaker, adaptation]
---

## 概述

Sentinel 1.8.10（2026-05-21 发布）新增了 **Apache HttpClient 5.x 适配器**和 **Spring RestClient 适配器**，补齐了 Sentinel 在最新 HTTP 客户端生态中的覆盖。同时新增了**可选的精确匹配优化开关**，可跳过正则资源匹配以提升性能。

## 1. Apache HttpClient 5.x 适配器

### 添加依赖

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-apache-httpclient5-adapter</artifactId>
    <version>1.8.10</version>
</dependency>
```

### 配置 HttpClient 5

```java
import org.apache.hc.client5.http.impl.classic.HttpClientBuilder;
import com.alibaba.csp.sentinel.adapter.apache.httpclient5.SentinelHttpClientBuilder;

// 方式一：通过 Sentinel 包装的 Builder 创建
HttpClientBuilder builder = SentinelHttpClientBuilder.create()
    .setMaxTotal(200)
    .setDefaultMaxPerRoute(50);

CloseableHttpClient client = builder.build();

// 方式二：手动注册 Sentinel 拦截器
HttpClientBuilder builder2 = HttpClientBuilder.create();
builder2.addRequestInterceptorLast(new SentinelApacheHttpClient5Interceptor());
builder2.addResponseInterceptorLast(new SentinelApacheHttpClient5Interceptor());
CloseableHttpClient client2 = builder2.build();
```

### Spring Boot 自动配置

```java
@Configuration
public class HttpClient5Config {

    @Bean
    public CloseableHttpClient sentinelHttpClient5() {
        return SentinelHttpClientBuilder.create()
            .setMaxTotal(200)
            .setDefaultMaxPerRoute(20)
            .build();
    }
}
```

### 资源命名规则

```
# HTTP 方法 + URI 路径
GET:/api/orders/{id}
POST:/api/orders
```

## 2. Spring RestClient 适配器

### 添加依赖

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-spring-restclient-adapter</artifactId>
    <version>1.8.10</version>
</dependency>
```

### 配置 RestClient

```java
import org.springframework.web.client.RestClient;
import com.alibaba.csp.sentinel.adapter.spring.restclient.SentinelRestClientBuilder;

// 方式一：通过 Sentinel Builder
RestClient client = SentinelRestClientBuilder.create()
    .baseUrl("https://api.example.com")
    .build();

// 方式二：手动添加 Sentinel 拦截器
RestClient client2 = RestClient.builder()
    .requestInterceptor(new SentinelRestClientInterceptor())
    .build();
```

### Spring Boot 自动注入示例

```java
@Service
public class OrderService {
    
    private final RestClient restClient;
    
    public OrderService(RestClient.Builder builder) {
        this.restClient = SentinelRestClientBuilder.from(builder)
            .baseUrl("https://order-service")
            .build();
    }
    
    public Order getOrder(Long id) {
        return restClient.get()
            .uri("/orders/{id}", id)
            .retrieve()
            .body(Order.class);
    }
}
```

## 3. 精确资源匹配优化（1.8.10+）

### 问题背景

当 Sentinel 的资源数量很大时（数万级别），每次请求都要遍历正则匹配列表查找资源，可能成为瓶颈。

### 开启精确匹配优先（非正则模式）

```properties
# application.properties
# 当资源名已有精确匹配时，跳过正则匹配
# 默认 false（保持向后兼容）
csp.sentinel.prefer.exact.match=true
```

### 工作原理

```
请求: GET:/api/orders/123

精确匹配模式 (prefer.exact.match=true):
  1. 查找精确匹配 "GET:/api/orders/123" → 有规则则应用
  2. 如果没有精确匹配，回退到正则匹配
  3. 性能优化：避免热点资源每次都走正则

传统模式 (prefer.exact.match=false):
  1. 总是先检查所有正则表达式的匹配
  2. 虽然灵活，但大量资源时性能差
```

### 使用建议

| 场景 | 推荐设置 | 原因 |
|------|---------|------|
| 高并发核心接口 | `true` | 减少正则匹配开销 |
| RESTful 动态参数多 | `true` | 精确匹配 + 按需配置正则 |
| 大量自定义正则规则 | `false` | 保持向后兼容 |
| 迁移期 | `false` | 确保规则完全迁移后再切换 |

## 注意事项

1. **HttpClient 4 到 5 迁移**：`sentinel-apache-httpclient4-adapter` 依然可用，但建议新项目直接使用 5.x 适配器
2. **RestClient vs RestTemplate**：Spring 官方推荐新项目使用 RestClient，Spring Boot 4.x 中 RestTemplate 适配器同样可用
3. **精确匹配迁移**：在现有生产环境开启 `prefer.exact.match=true` 前，建议先在灰度环境验证规则覆盖完整性
4. **依赖冲突**：HttpClient 5.x 适配器需确保项目中无 HttpClient 4.x 的冲突依赖
