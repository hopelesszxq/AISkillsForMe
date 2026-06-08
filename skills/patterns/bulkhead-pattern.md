---
name: bulkhead-pattern
description: Bulkhead（隔舱）隔离模式——线程池隔离与信号量隔离的微服务容错最佳实践
tags: [patterns, resilience, bulkhead, thread-pool, isolation, fault-tolerance]
---

## 概述

Bulkhead（隔舱）模式借鉴自船舶设计——将船舱分隔成独立隔间，单个隔舱进水不会导致整船沉没。在微服务中，Bulkhead 模式通过隔离资源（线程池、连接池、队列）防止一个下游故障耗尽所有系统资源。

## 一、两种隔离策略

| 特性 | 线程池隔离 | 信号量隔离 |
|------|-----------|-----------|
| **资源模型** | 独立线程池 | 计数器（Semaphore） |
| **线程切换** | 有（提交到独立线程执行） | 无（调用线程执行） |
| **上下文传播** | 需要显式传递（MDC、Trace） | 自动继承 |
| **超时控制** | 线程池自带 | 需配合 TimeLimiter |
| **内存开销** | 较高（每个线程栈约 1MB） | 极低 |
| **适用场景** | 耗时操作、第三方 API 调用 | 内存操作、本地计算、低延迟 |

## 二、Resilience4j 实现

### 2.1 线程池隔离

```yaml
# application.yml
resilience4j:
  bulkhead:
    configs:
      default:
        max-concurrent-calls: 10         # 最大并发请求数
        max-wait-duration: 500ms         # 请求等待超时
    instances:
      paymentService:
        base-config: default
        max-concurrent-calls: 5
  thread-pool-bulkhead:
    configs:
      default:
        core-thread-pool-size: 5         # 核心线程数
        max-thread-pool-size: 10         # 最大线程数
        queue-capacity: 20               # 队列容量
        keep-alive-duration: 30s         # 空闲线程存活时间
    instances:
      paymentService:
        base-config: default
        core-thread-pool-size: 5
        max-thread-pool-size: 10
        queue-capacity: 20
```

```java
// 线程池隔舱——适用于支付等耗时远程调用
@Bulkhead(name = "paymentService", type = Bulkhead.Type.THREADPOOL)
@CircuitBreaker(name = "paymentService")
@TimeLimiter(name = "paymentService")
public CompletableFuture<PaymentResult> processPayment(PaymentRequest request) {
    return CompletableFuture.supplyAsync(() ->
        paymentClient.charge(request)
    );
}

// 信号量隔舱——适用于本地计算/缓存查询
@Bulkhead(name = "cacheService", type = Bulkhead.Type.SEMAPHORE)
public CacheEntry getFromCache(String key) {
    return cacheClient.get(key);
}
```

### 2.2 Spring Boot 自动配置

```java
@Configuration
public class BulkheadConfig {

    @Bean
    public Customizer<ThreadPoolBulkheadConfig> paymentBulkheadCustomizer() {
        return config -> {
            config.setCoreThreadPoolSize(5);
            config.setMaxThreadPoolSize(10);
            config.setQueueCapacity(20);
            config.setKeepAliveDuration(Duration.ofSeconds(30));
            // 上下文传播
            config.setContextPropagators(ContextPropagator.of(MDCPropagator.getInstance()));
        };
    }

    @Bean
    public BulkheadCustomizer semaphoreBulkheadCustomizer() {
        return config -> {
            config.setMaxConcurrentCalls(20);
            config.setMaxWaitDuration(Duration.ofMillis(500));
        };
    }
}
```

### 2.3 事件监控

```java
@EventListener
public void onBulkheadEvent(BulkheadEvent event) {
    switch (event.getEventType()) {
        case CALL_PERMITTED -> metrics.counter("bulkhead.call.permitted").increment();
        case CALL_REJECTED  -> {
            metrics.counter("bulkhead.call.rejected").increment();
            log.warn("Bulkhead rejected: {} (active={}/{})",
                event.getBulkheadName(),
                event.getNumberOfAllowedCalls(),
                event.getNumberOfPermittedCalls());
        }
        case CALL_FINISHED  -> metrics.counter("bulkhead.call.finished").increment();
    }
}
```

## 三、Hystrix 遗留模式对比

Spring Cloud Hystrix 默认使用**线程池隔离**，每个依赖对应一个独立线程池：

```yaml
# Hystrix（已停维，仅作参考）
hystrix:
  command:
    userService:
      execution:
        isolation:
          strategy: THREAD       # THREAD（线程池）/ SEMAPHORE（信号量）
          thread:
            timeoutInMilliseconds: 3000
  threadpool:
    userService:
      coreSize: 10
      maxQueueSize: 20
```

> Resilience4j 是 Hystrix 的现代替代品，线程池隔离默认通过 CompletableFuture 实现，信号量隔离无线程切换开销。

## 四、连接池隔离模式

### 4.1 HTTP 连接池隔离

```java
@Configuration
public class HttpPoolIsolationConfig {

    // 支付服务专用连接池
    @Bean("paymentRestClient")
    public RestClient paymentRestClient() {
        return RestClient.builder()
            .baseUrl("https://payment.example.com")
            .requestFactory(new JdkClientHttpRequestFactory(
                HttpClient.newBuilder()
                    .executor(Executors.newFixedThreadPool(5))  // 隔离线程
                    .build()
            ))
            .build();
    }

    // 库存服务专用连接池
    @Bean("inventoryRestClient")
    public RestClient inventoryRestClient() {
        return RestClient.builder()
            .baseUrl("https://inventory.example.com")
            .requestFactory(new JdkClientHttpRequestFactory(
                HttpClient.newBuilder()
                    .executor(Executors.newFixedThreadPool(3))
                    .build()
            ))
            .build();
    }
}
```

### 4.2 数据库连接池隔离

```java
// 写库连接池（有界）
@Bean("writeDataSource")
public DataSource writeDataSource() {
    HikariConfig config = new HikariConfig();
    config.setMaximumPoolSize(10);
    config.setPoolName("write-pool");
    return new HikariDataSource(config);
}

// 报表查询连接池（大查询不阻塞写）
@Bean("reportDataSource")
public DataSource reportDataSource() {
    HikariConfig config = new HikariConfig();
    config.setMaximumPoolSize(5);
    config.setPoolName("report-pool");
    config.setConnectionTimeout(30000);  // 报表查询允许更长的等待
    return new HikariDataSource(config);
}
```

## 五、Virtual Thread 场景的特殊考量

```java
// JVM 21+ 虚拟线程下，线程池隔离意义减弱
// 因为虚拟线程本身极轻量，但仍需限制并发数

@Bean("virtualPaymentExecutor")
public Executor virtualPaymentExecutor() {
    return Executors.newVirtualThreadPerTaskExecutor();
}

// 用信号量限制并发
private final Semaphore paymentSemaphore = new Semaphore(20);

public void processPaymentVirtual(PaymentRequest request) {
    if (!paymentSemaphore.tryAcquire(500, TimeUnit.MILLISECONDS)) {
        throw new BulkheadFullException("支付服务繁忙，请稍后重试");
    }
    try {
        Thread.startVirtualThread(() -> paymentClient.charge(request));
    } finally {
        paymentSemaphore.release();
    }
}
```

## 六、Spring Cloud Gateway 中的 Bulkhead

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: payment-route
          uri: lb://payment-service
          predicates:
            - Path=/api/payment/**
          filters:
            - name: RequestRateLimiter
              args:
                key-resolver: "#{@userKeyResolver}"
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
```

配合 Resilience4j 在 Gateway 使用：

```yaml
resilience4j:
  bulkhead:
    instances:
      payment-route:
        max-concurrent-calls: 20
        max-wait-duration: 50ms
```

## 七、最佳实践

### 7.1 池大小计算公式

```
线程数 = (QPS × P99延迟(秒)) / 单线程可处理请求数

# 示例：预计 QPS=200, P99=500ms, 单线程处理 50 req/s
线程数 = (200 × 0.5) / 50 = 2 → 实际取 5~8（含缓冲）
```

### 7.2 监控指标

| 指标 | 含义 | 告警阈值 |
|------|------|---------|
| `bulkhead.active_count` | 当前活跃请求数 | > max_calls × 0.8 |
| `bulkhead.queue_depth` | 排队等待数 | > 0 即关注 |
| `bulkhead.rejected_count` | 被拒绝请求数 | > 0 需优化 |
| `bulkhead.running_duration` | 请求执行耗时 | > P99 阈值 |

### 7.3 上下文传播

```java
// MDC、TraceId 跨线程传播
ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
    .contextPropagators(ContextPropagator.of(
        new MDCContextPropagator(),
        new TraceContextPropagator()
    ))
    .build();

// 或在代码中手动传播
ThreadPoolBulkhead bulkhead = ThreadPoolBulkhead.of("payment", config);
```

## 注意事项

1. **线程池隔离不是银弹**：线程池切换有上下文开销，大量线程池可能导致线程碎片化。
2. **虚拟线程时代的隔离**：JDK 21+ 虚拟线程场景下，信号量隔离更轻量，线程池隔离应替换为信号量+并发限制。
3. **连接池与线程池数量关系**：每个线程池需要独立的连接池，否则连接池依然共享。线程池数 × 连接池大小需 < 系统文件描述符限制。
4. **回退处理**：被拒绝的请求必须提供 fallback，默认抛出 BulkheadFullException，建议包装为友好的降级响应。
5. **死锁风险**：线程池 A 调用线程池 B，而 B 又回调用 A——形成双向依赖死锁。通过依赖调用图避免循环依赖。
6. **上下文丢失**：线程池切换会丢失 MDC、SecurityContext、TraceId，需显式配置 ContextPropagator 或手动传播。
