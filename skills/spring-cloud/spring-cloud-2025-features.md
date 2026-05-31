---
name: spring-cloud-2025-features
description: Spring Cloud 2025.0.0 新特性详解（虚拟线程、GraalVM、Spring Boot 3.4+）
tags: [spring-cloud, microservice, gateway, feign]
---

## Spring Cloud 2025.0.0 新特性详解

Spring Cloud 2025.0.0 是继 Spring Cloud 2024.0 之后的重大版本，配合 **Spring Boot 3.4+** 和 **JDK 21**，带来了多项重要更新。

> **版本代号**：2025.0.0 (又名 `Utopia-SR0`)

## 一、关键升级

| 组件 | 版本 | 说明 |
|------|------|------|
| Spring Boot | 3.4.x+ | 最低要求 Spring Boot 3.4 |
| JDK | 17+ (推荐 21) | 最低要求 JDK 17，推荐 JDK 21 以使用虚拟线程 |
| Spring Cloud Gateway | 4.2.x | WebFlux + Netty，虚拟线程支持 |
| Spring Cloud OpenFeign | 4.2.x | 完全响应式/虚拟线程模式 |
| Spring Cloud LoadBalancer | 4.2.x | 性能优化 |
| Spring Cloud Circuit Breaker | 4.2.x | Resilience4j 2.x 集成 |

## 二、虚拟线程（Virtual Threads）全面支持

Spring Cloud 2025 对 JDK 21 虚拟线程做了全面适配，网关和 Feign 可以直接使用虚拟线程：

### Spring Cloud Gateway 虚拟线程模式

```yaml
# application.yml - 启用虚拟线程
spring:
  threads:
    virtual:
      enabled: true

# Gateway 路由示例
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - name: CircuitBreaker
              args:
                name: userServiceCB
                fallbackUri: forward:/fallback/users
```

```java
// 自定义过滤器 - 使用虚拟线程
@Component
public class VirtualThreadFilter implements GlobalFilter, Ordered {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // Gateway 自身基于 WebFlux（反应式），
        // 但在调用阻塞服务时可以通过虚拟线程优化
        return chain.filter(exchange);
    }
    
    // 使用虚拟线程异步处理耗时操作
    public CompletableFuture<String> blockingOperation() {
        return CompletableFuture.supplyAsync(() -> {
            // 这里会在虚拟线程中执行
            return callExternalService();
        });
    }
}
```

### OpenFeign + 虚拟线程

```java
// Feign 接口 - 虚拟线程模式下自动使用虚拟线程
@FeignClient(name = "user-service", url = "${user-service.url}")
public interface UserClient {
    
    @GetMapping("/users/{id}")
    User getUser(@PathVariable Long id);
    
    @PostMapping("/users")
    User createUser(@RequestBody User user);
}

// 配置
@Configuration
public class FeignConfig {
    
    @Bean
    public Feign.Builder feignBuilder() {
        return Feign.builder()
            // 使用虚拟线程执行器
            .executor(Executors.newVirtualThreadPerTaskExecutor())
            .retryer(new Retryer.Default(100, 1000, 3));
    }
}
```

## 三、GraalVM 原生镜像改进

Spring Cloud 2025 对 GraalVM Native Image 的兼容性大幅提升：

```xml
<!-- pom.xml - GraalVM 原生编译 -->
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
</plugin>
```

```bash
# 编译原生镜像
mvn -Pnative native:compile

# 启动（毫秒级）
./target/gateway-service
```

**支持的原生编译组件：**
- ✅ Spring Cloud Gateway（已验证）
- ✅ Spring Cloud OpenFeign（新增）
- ✅ Spring Cloud LoadBalancer
- ✅ Spring Cloud Circuit Breaker
- ⚠️ Spring Cloud Config Client（部分限制）
- ❌ Spring Cloud Config Server（暂不支持）

## 四、Gateway 性能优化

### 4.1 Netty 线程模型优化

```yaml
# Spring Cloud Gateway - 性能调优
spring:
  cloud:
    gateway:
      httpclient:
        # 连接池配置
        pool:
          type: elastic
          max-connections: 1000
          acquire-timeout: 45000
        # 响应超时
        response-timeout: 10s
        # Netty 线程数 (默认: CPU核心数*2)
      threads:
        event-loop: 16
```

### 4.2 路由配置简化

```yaml
# 2025 新语法 - 更简洁的路由配置
spring:
  cloud:
    gateway:
      routes:
        # 支持直接指定负载均衡策略
        - id: service-route
          uri: lb://backend-service
          predicates:
            - Path=/api/**
          filters:
            # 内置限流过滤器改进
            - name: RequestRateLimiter
              args:
                redis-rate-limiter:
                  replenishRate: 100
                  burstCapacity: 200
            # 响应缓存
            - CacheResponse
            # 请求重试（支持虚拟线程）
            - name: Retry
              args:
                retries: 3
                statuses: BAD_GATEWAY, SERVICE_UNAVAILABLE
```

## 五、OpenFeign 增强

### 5.1 HTTP/2 支持

```java
// Feign 配置 HTTP/2
@Configuration
public class FeignHttp2Config {
    
    @Bean
    public Client feignClient() {
        return new Http2Client();
    }
}
```

### 5.2 声明式重试策略

```java
@FeignClient(name = "inventory-service")
@Retryable(retryFor = InventoryException.class, maxAttempts = 3)
public interface InventoryClient {
    
    @GetMapping("/stock/{sku}")
    Stock getStock(@PathVariable String sku);
    
    // 特定方法的单独重试策略
    @Retryable(retryFor = TimeoutException.class, maxAttempts = 5, backoff = @Backoff(delay = 500))
    @PostMapping("/reserve")
    Reservation reserve(@RequestBody ReserveRequest request);
}
```

## 六、服务发现增强

### 6.1 多注册中心支持

```yaml
# 同时注册到 Nacos 和 Eureka
spring:
  cloud:
    service-registry:
      auto-registration:
        enabled: true
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
    netflix:
      eureka:
        client:
          service-url:
            defaultZone: http://localhost:8761/eureka/
```

## 七、Spring Cloud 2025 版本兼容矩阵

| Spring Cloud 版本 | Spring Boot 版本 | JDK | Spring Cloud Alibaba |
|-------------------|------------------|-----|---------------------|
| 2025.0.0 | 3.4.x | 17+ (推荐21) | 2025.0.0 |
| 2024.0.x | 3.3.x | 17+ | 2023.0.x |
| 2023.0.x | 3.2.x | 17+ | 2022.0.x |

## 注意事项

1. **虚拟线程不是万能的**：Gateway 基于 WebFlux 反应式编程模型，本身已经是异步非阻塞。虚拟线程主要优化 Feign 客户端等使用 `RestTemplate` 或阻塞 I/O 的场景
2. **GraalVM 限制**：动态代理、反射、序列化等场景在原生镜像中仍有限制，需通过 `@RegisterReflectionForBinding` 或 `reachability-metadata.json` 配置
3. **兼容性检查**：升级前检查所有第三方依赖是否支持 Spring Boot 3.4 和 JDK 21
4. **Feign 变更**：Spring Cloud 2025 移除了 Feign 的 Hystrix 支持，全面转向 Resilience4j
5. **测试覆盖**：针对虚拟线程场景增加并发测试，虚拟线程的坑包括 `synchronized` 固定（pinning）和线程池饥饿
