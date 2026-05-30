---
name: feign-migration-spring6
description: OpenFeign 从 Spring Boot 2.x 迁移到 3.x 实战指南：Jakarta EE、HTTP Client 5、配置变化与兼容处理
tags: [spring-cloud, feign, migration, spring-boot-3, jakarta, httpclient]
---

## 概述

Spring Boot 3.x / Spring Framework 6 将 Java EE 迁移到 **Jakarta EE**（javax → jakarta 命名空间），同时 OpenFeign 从 11.x 升级到 12.x+，底层 HTTP 客户端也发生了变化。本文为从 Spring Boot 2.x + Feign 10.x 迁移到 Spring Boot 3.x + Feign 12.x+ 提供完整指南。

## 版本对应关系

| Spring Boot | Feign 版本 | 命名空间 | HTTP 客户端 |
|---|---|---|---|
| 2.3.x - 2.7.x | 10.x - 11.x | javax.* | Apache HttpClient 4.x / OkHttp 3.x |
| 3.0.x - 3.1.x | 12.x | jakarta.* | Apache HttpClient 5.x / OkHttp 4.x |
| 3.2.x - 3.4.x | 12.x - 13.x | jakarta.* | Apache HttpClient 5.x / OkHttp 4.x |

## Maven 依赖变更

### Spring Boot 2.x

```xml
<!-- spring-cloud-starter-openfeign 自动引入 Feign 10.x -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>

<!-- 如使用 OkHttp 作为底层 -->
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-okhttp</artifactId>
</dependency>
```

### Spring Boot 3.x

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>

<!-- Jakarta 兼容 -->
<dependency>
    <groupId>jakarta.servlet</groupId>
    <artifactId>jakarta.servlet-api</artifactId>
    <scope>provided</scope>
</dependency>

<!-- HTTP Client 5 (推荐) -->
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-hc5</artifactId>
</dependency>

<!-- 或 OkHttp 4.x -->
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-okhttp</artifactId>
</dependency>
```

## 代码迁移要点

### 1. 导入路径变更

```java
// Spring Boot 2.x (javax)
import javax.servlet.http.HttpServletRequest;
import javax.validation.Valid;
import javax.validation.constraints.NotBlank;

// Spring Boot 3.x (jakarta)
import jakarta.servlet.http.HttpServletRequest;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
```

### 2. Feign 客户端接口

```java
// Spring Boot 2.x
// 无变化，但内部使用 javax 注解的需替换
@FeignClient(name = "user-service", url = "${user.service.url}")
public interface UserClient {

    @GetMapping("/users/{id}")
    User getUser(@PathVariable("id") Long id);

    @PostMapping("/users")
    User createUser(@RequestBody @Valid User user);
}
```

### 3. 自定义配置类

```java
// Spring Boot 3.x + Feign 12.x
@Configuration
public class FeignConfig {

    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }

    @Bean
    public RequestInterceptor requestInterceptor() {
        return requestTemplate -> {
            requestTemplate.header("X-Source", "gateway");
            // 从请求上下文获取 TraceId
            String traceId = MDC.get("traceId");
            if (traceId != null) {
                requestTemplate.header("X-Trace-Id", traceId);
            }
        };
    }

    // Feign 12.x 推荐使用 Apache HttpClient 5
    @Bean
    public Client feignClient(HttpClient httpClient) {
        return new ApacheHttp5Client(httpClient);
    }

    @Bean
    public HttpClient httpClient() {
        return HttpClientBuilder.create()
            .setMaxConnTotal(200)
            .setMaxConnPerRoute(50)
            .setConnectionTimeToLive(30, TimeUnit.SECONDS)
            .setDefaultRequestConfig(RequestConfig.custom()
                .setConnectTimeout(5000)
                .setResponseTimeout(10000)
                .build())
            .build();
    }
}
```

### 4. 配置项变化

```yaml
# Spring Boot 2.x
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 10000
        loggerLevel: basic
  compression:
    request:
      enabled: true
    response:
      enabled: true
  httpclient:
    enabled: true
    max-connections: 200
    max-connections-per-route: 50

# Spring Boot 3.x (配置不变，但底层实现变了)
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 10000
        loggerLevel: basic
  compression:
    request:
      enabled: true
    response:
      enabled: true
  httpclient:
    hc5:
      enabled: true  # Spring Boot 3.x 使用 hc5 替代 httpclient
    max-connections: 200
    max-connections-per-route: 50
```

## 常见兼容性问题

### 1. ClassNotFoundException: javax.servlet.*

```
java.lang.NoClassDefFoundError: javax/servlet/ServletException
```

**原因**: Feign 12.x 不传递 javax.servlet API。
**解决**: 添加 Jakarta 兼容依赖或替换为 jakarta 等价类。

### 2. Hystrix 不再可用

```xml
<!-- Spring Boot 2.x 可能使用了 Hystrix -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

Spring Cloud 2022.0.x+ 移除了 Hystrix 支持。

```xml
<!-- ✅ 迁移到 Resilience4j -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

### 3. Feign 重试器变化

```java
// Feign 10.x
@Bean
public Retryer feignRetryer() {
    return new Retryer.Default(100, 1000, 3);
}

// Feign 12.x 保持不变（接口兼容）
@Bean
public Retryer feignRetryer() {
    return new Retryer.Default(100, 1000, 3);
}
```

### 4. 异步 Feign 的变化

```java
// Spring Boot 3.x 使用 CompletableFuture
@FeignClient(name = "async-service", async = true)
public interface AsyncUserClient {

    CompletableFuture<User> getUser(@RequestParam("id") Long id);
}
```

## 迁移检查清单

- [ ] Spring Boot 版本升级到 3.x
- [ ] Spring Cloud 版本切换到 2022.0.x+
- [ ] javax.* 导入已全部替换为 jakarta.*
- [ ] Hystrix → Resilience4j 迁移完成
- [ ] Feign 版本 >= 12.x
- [ ] HTTP 客户端更新到 HC5 或 OkHttp 4
- [ ] tomcat-embed-jasper / javax.servlet 依赖已替换
- [ ] 自定义错误解码器（ErrorDecoder）验证兼容性
- [ ] 集成测试覆盖所有 Feign 调用
