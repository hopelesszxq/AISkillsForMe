---
name: resilience4j-240-features
description: Resilience4j 2.4.0 新特性：断路器初始状态配置、RateLimiter 名称追踪、TimeLimiter Builder、withFallback 装饰器
tags: [patterns, resilience, circuit-breaker, retry, rate-limiter, spring-cloud]
---

## Resilience4j 2.4.0 新特性

Resilience4j 2.4.0（2026 年发布）在 2.3.x 基础上引入了多个实用的增强特性，对 Spring Boot 3 微服务开发者尤其有用。

> 现有 retry-circuitbreaker-pattern.md 覆盖了 2.3.x 的基础用法，本文聚焦 2.4.0 的新增能力。

## 一、断路器初始状态配置（2.4.0 新增）

### 功能

允许从配置层面指定断路器的**初始状态**（CLOSED / OPEN / HALF_OPEN / DISABLED / FORCED_OPEN），适用于以下场景：

- 服务刚启动时已知下游不可用（如依赖的外部系统维护中），直接置为 OPEN 避免无效调用
- 灰度发布时新节点默认 DISABLED，等确认后再启用熔断保护
- 测试环境下强制断路器处于特定状态

### YAML 配置

```yaml
resilience4j:
  circuitbreaker:
    instances:
      externalPaymentService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        # 2.4.0 新增：启动时直接设为 OPEN 状态
        initialState: OPEN
```

### Java 代码配置

```java
@Bean
public CircuitBreakerConfig customCircuitBreakerConfig() {
    return CircuitBreakerConfig.custom()
        .slidingWindowSize(10)
        .failureRateThreshold(50)
        .waitDurationInOpenState(Duration.ofSeconds(30))
        .initialState(CircuitBreaker.State.OPEN)  // 2.4.0+
        .build();
}
```

### 配合健康检查自动修正

```yaml
# 启动时 OPEN → 对下游健康检查端点周期性探测
# 使用 resilience4j 的 HealthContributorAutoConfiguration
# 当发现下游恢复后，通过 Actuator 接口手动重置：
# POST /actuator/circuitbreakers/externalPaymentService/transitionToClosed
```

## 二、RateLimiter 名称追踪 API（2.4.0 新增）

`getCausingRateLimiterName()` 方法可获取**实际触发限流的 RateLimiter 名称**，在级联限流场景中快速定位瓶颈。

```java
@Bean
public RateLimiter rateLimiterForDb() {
    return RateLimiter.of("dbLimiter", RateLimiterConfig.custom()
        .limitForPeriod(100)
        .limitRefreshPeriod(Duration.ofSeconds(1))
        .build());
}

@Bean
public RateLimiter rateLimiterForApi() {
    return RateLimiter.of("apiLimiter", RateLimiterConfig.custom()
        .limitForPeriod(20)
        .limitRefreshPeriod(Duration.ofSeconds(1))
        .build());
}

// 使用
@Service
public class RateLimitedService {

    public void call() {
        try {
            rateLimiterForApi().acquirePermission(1);
            rateLimiterForDb().acquirePermission(1);
            // 实际业务调用...
        } catch (RequestNotPermitted e) {
            // 2.4.0 新增：精准定位是哪个限流器触发了拒绝
            String causingLimiter = e.getCausingRateLimiterName();
            log.warn("限流触发，来源: {}", causingLimiter);
        }
    }
}
```

## 三、TimeLimiter Registry Builder（2.4.0 新增）

为 TimeLimiter 提供独立的 Registry Builder，支持更灵活的自定义配置。

```java
@Configuration
public class TimeLimiterConfig {

    @Bean
    public TimeLimiterRegistry timeLimiterRegistry() {
        // 使用自定义 Registry 而非默认全局配置
        return TimeLimiterRegistry.of(TimeLimiterConfig.custom()
            .timeoutDuration(Duration.ofSeconds(3))
            .cancelRunningFuture(true)
            .build()
        );
    }

    @Bean
    public TimeLimiter fastTimeout() {
        return timeLimiterRegistry()
            .timeLimiter("fastTimeout", TimeLimiterConfig.custom()
                .timeoutDuration(Duration.ofMillis(500))
                .build());
    }
}
```

## 四、withFallback() 装饰器扩展（2.4.0 新增）

`DecorateFunction` 新增 `withFallback()` 方法，使函数式编程中的降级装配更简洁。

```java
// 2.4.0 之前需要链式组合
Supplier<String> decoratedSupplier = CircuitBreaker
    .decorateSupplier(circuitBreaker, this::callExternalService);
Supplier<String> withRetry = Retry
    .decorateSupplier(retry, decoratedSupplier);
// 降级需要额外拆分

// 2.4.0+ 使用 withFallback
Supplier<String> result = Decorators
    .ofSupplier(this::callExternalService)
    .withCircuitBreaker(circuitBreaker)
    .withRetry(retry)
    .withTimeLimiter(timeLimiter)
    .withFallback(this::getFallbackValue)  // 2.4.0 新增
    .decorate();
```

## 五、THREAD_POOL_BULKHEAD 装饰器支持

`Decorators` 类现已支持 `ThreadPoolBulkhead`，补齐了之前只支持 `SemaphoreBulkhead` 的缺口。

```java
ThreadPoolBulkhead threadPoolBulkhead = ThreadPoolBulkhead.ofDefaults();

CompletableFuture<String> future = Decorators
    .ofSupplier(this::callExternalService)
    .withThreadPoolBulkhead(threadPoolBulkhead)  // 2.4.0 新增
    .withCircuitBreaker(circuitBreaker)
    .get()
    .thenCompose(Function.identity());
```

## 升级注意事项

```xml
<!-- 从 2.3.x 升级 -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.4.0</version>
</dependency>
```

1. 移除对 `kotlin-stdlib-jdk8` 的依赖（不再需要，core 不依赖 Kotlin）
2. `slidingWindow` 恢复默认同步策略（2.3.x 曾改动过默认值）
3. 所有新增 API 向后兼容，2.3.x 配置无需修改

## 参考

- [Resilience4j 2.4.0 Release](https://github.com/resilience4j/resilience4j/releases/tag/v2.4.0)
