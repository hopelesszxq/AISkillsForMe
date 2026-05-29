---
name: feign-virtual-threads
description: OpenFeign 在 Spring Boot 3.x + Java 21 虚拟线程环境中的配置、踩坑与最佳实践
tags: [spring-cloud, feign, virtual-threads, loom, java21, performance]
---

## 概述

Java 21 虚拟线程（Virtual Threads / Project Loom）和 Spring Boot 3.x 的组合让 Feign 的线程模型发生了根本变化。本文介绍 Feign 在虚拟线程环境下的配置、坑点和调优方案。

## 一、虚拟线程对 Feign 的影响

### 默认行为变化

Spring Boot 3.2+ 启用虚拟线程后，Feign 的默认 `ThreadPoolExecutor` 不再适用：

```yaml
# application.yml — 启用虚拟线程
spring:
  threads:
    virtual:
      enabled: true
```

```yaml
# 启用后 Feign 的同步调用行为：
# 1. 每个请求在虚拟线程中执行，而非平台线程
# 2. 连接池的 thread-local 行为发生变化（虚拟线程不共享 ThreadLocal）
# 3. 传统的 Hystrix/Sentinel 线程池隔离模式需要调整
```

### 核心差异

| 方面 | 平台线程（传统） | 虚拟线程（Java 21+） |
|------|-----------------|-------------------|
| Feign 线程模型 | Tomcat 线程池 → Feign 连接池 | 虚拟线程池（自动管理） |
| 连接数 | worker-threads + max-connections | 可创建数百万虚拟线程 |
| ThreadLocal | 稳定且可预期 | 虚拟线程各自独立，可能内存泄漏 |
| 线程池隔离 | Hystrix/Sentinel 线程池方式 | 需要改用信号量隔离 |

## 二、Feign 虚拟线程配置

### 基础配置

```java
@Configuration
public class FeignConfig {

    @Bean
    public Feign.Builder feignBuilder() {
        return Feign.builder()
            // 虚拟线程环境建议使用信号量隔离
            .decode404()
            .build();
    }
}
```

### 自定义异步执行器

使用虚拟线程执行 Feign 异步调用：

```java
@Configuration
public class FeignAsyncConfig {

    @Bean
    public AsyncRestTemplate asyncRestTemplate() {
        // 使用虚拟线程的 SimpleAsyncTaskExecutor
        SimpleAsyncTaskExecutor executor = new SimpleAsyncTaskExecutor();
        executor.setVirtualThreads(true);
        return new AsyncRestTemplate(executor);
    }
}
```

### 连接池配置

```yaml
# feign + 虚拟线程的连接池关键配置
feign:
  client:
    config:
      default:
        connect-timeout: 5000
        read-timeout: 10000
  httpclient:
    enabled: true
    max-connections: 200           # 总连接数（虚拟线程可大量复用）
    max-connections-per-route: 50  # 每路由最大连接
    time-to-live: 900
    time-to-live-unit: seconds
    connection-timer-repeat: 3000
```

```java
// Java 配置：虚拟线程友好的 Feign 连接管理
@Bean
public CloseableHttpClient feignHttpClient() {
    return HttpClients.custom()
        .setMaxConnTotal(500)           // 虚拟线程可创建更多连接
        .setMaxConnPerRoute(100)
        .setConnectionTimeToLive(30, TimeUnit.SECONDS)
        // 启用连接回收
        .evictExpiredConnections()
        .evictIdleConnections(30, TimeUnit.SECONDS)
        .build();
}
```

## 三、虚拟线程下的 ThreadLocal 陷阱

### 问题描述

虚拟线程可以挂起/恢复，ThreadLocal 可能在线程挂起期间积累数据，导致内存泄漏。

```java
// ❌ 问题代码：RequestInterceptor 中使用 ThreadLocal 传递上下文
@Component
public class TokenInterceptor implements RequestInterceptor {
    private static final ThreadLocal<String> tokenHolder = new ThreadLocal<>();

    @Override
    public void apply(RequestTemplate template) {
        String token = tokenHolder.get();  // 虚拟线程环境可能为 null
        if (token != null) {
            template.header("Authorization", "Bearer " + token);
        }
    }
}
```

### 解决方案

```java
// ✅ 方案 1：使用 RequestAttribute 替代 ThreadLocal
@Component
public class SafeTokenInterceptor implements RequestInterceptor {
    
    @Autowired
    private HttpServletRequest request;

    @Override
    public void apply(RequestTemplate template) {
        // 从请求属性中获取，而非 ThreadLocal
        String token = (String) request.getAttribute("token");
        // 或从请求头中传递
        String authHeader = request.getHeader("Authorization");
        if (authHeader != null) {
            template.header("Authorization", authHeader);
        }
    }
}
```

```java
// ✅ 方案 2：显式清除 ThreadLocal
// 利用虚拟线程的 CompletableFuture 手动管理
CompletableFuture.supplyAsync(() -> {
    try {
        tokenHolder.set(token);
        return feignClient.callApi();
    } finally {
        tokenHolder.remove(); // 必须清理！
    }
});
```

## 四、服务隔离策略调整

### Sentinel + 虚拟线程

```java
// ❌ 虚拟线程下不要使用线程池隔离
@SentinelResource(
    value = "user-service",
    fallback = "fallback",
    blockHandlerClass = ExceptionUtil.class
)
// 使用信号量隔离替代线程池隔离
```

```yaml
# 虚拟线程环境推荐配置
spring:
  cloud:
    sentinel:
      scg:
        fallback:
          mode: response
          response-status: 429
      filter:
        enabled: true
      # 禁用线程池隔离，改用信号量
      datasource:
        flow:
          nacos:
            data-id: sentinel-flow-rules
            rule-type: flow
            data-type: json
```

### Resilience4j + 虚拟线程

```java
// Resilience4j 配置 — 使用信号量隔离
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> circuitBreakerCustomizer() {
    return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
        .circuitBreakerConfig(CircuitBreakerConfig.custom()
            .slidingWindowSize(10)
            .failureRateThreshold(50)
            .minimumNumberOfCalls(5)
            .build())
        .timeLimiterConfig(TimeLimiterConfig.custom()
            .timeoutDuration(Duration.ofSeconds(3))
            .cancelRunningFuture(true)
            .build())
        .build());
}
```

## 五、性能调优建议

### 1. 调整最大连接数

```yaml
# 传统平台线程（200 Tomcat threads）：
#   Tomcat threads × 每个线程持有的连接 ≈ 连接池大小
#   一般 max-connections = Tomcat threads × 2
#
# 虚拟线程（无限制）：
#   不存在线程池大小限制，但系统 FD 仍有限制
#   建议 max-connections = 200~500（根据 ulimit -n 调整）
feign:
  httpclient:
    max-connections: 500
```

### 2. 禁用不必要的线程切换

```java
// ✅ 虚拟线程环境下直接同步调用，无需包装为异步
@Service
public class OrderService {
    
    private final UserFeignClient userClient;

    // 直接调用即可，虚拟线程自动挂起
    public Order createOrder(OrderRequest req) {
        User user = userClient.getUser(req.getUserId());
        // 虚拟线程在此处 suspend 而不阻塞平台线程
        return orderRepository.save(new Order(user, req));
    }
}
```

### 3. 调优连接超时

```yaml
# 虚拟线程更密集使用连接，超时需更严格
feign:
  client:
    config:
      default:
        connect-timeout: 2000      # 从 5000 减小到 2000
        read-timeout: 5000         # 从 10000 减小到 5000
```

## 六、常见坑点

| 问题 | 现象 | 解决方案 |
|------|------|---------|
| **ThreadLocal 泄漏** | 虚拟线程池膨胀，内存不断增长 | 改用 `RequestContextHolder` 或手动 `remove()` |
| **死锁** | 虚拟线程持有锁后挂起，导致 pinning | 避免在同步块/锁内调用 Feign；使用 `ReentrantLock` 替代 `synchronized` |
| **Thread pool isolation 失效** | Sentinel/Hystrix 线程池隔离在虚拟线程下不生效 | 使用信号量隔离（`StrategySemaphore`） |
| **连接耗尽** | 虚拟线程快速创建，短时间占满 FD | 调大 `ulimit -n` 和 `max-connections` |
| **MDC 日志丢失** | 虚拟线程切换时 MDC 上下文丢失 | 手动传递 MDC（`ThreadPoolTaskExecutor` 的 `TaskDecorator`） |
| **pinning 性能下降** | 虚拟线程 pinned 到平台线程，失去优势 | 避免在 `synchronized` 块内调用 Feign；改用显式锁 |

```java
// MDC 传递示例：TaskDecorator
@Bean
public Executor virtualThreadExecutor() {
    SimpleAsyncTaskExecutor executor = new SimpleAsyncTaskExecutor();
    executor.setVirtualThreads(true);
    executor.setTaskDecorator(runnable -> {
        Map<String, String> contextMap = MDC.getCopyOfContextMap();
        return () -> {
            try {
                MDC.setContextMap(contextMap);
                runnable.run();
            } finally {
                MDC.clear();
            }
        };
    });
    return executor;
}
```

## 注意事项

| 要点 | 说明 |
|------|------|
| **Spring Boot 版本** | 虚拟线程 + Feign 推荐 Spring Boot 3.2+、Java 21+ |
| **Spring Cloud 版本** | 推荐 2023.0.x（对应 Spring Boot 3.2）或 2024.0.x（对应 Spring Boot 3.3+） |
| **Pinning 检测** | 启动时加 `-Djdk.tracePinnedThreads=short` 检测虚拟线程 pinning |
| **线程池隔离不可用** | Hystrix/Sentinel 的线程池隔离方式与虚拟线程不兼容，必须切换为信号量 |
| **连接池大小** | 虚拟线程每请求不再独占平台线程，但网络连接仍是有限资源，不可无限放大 |
