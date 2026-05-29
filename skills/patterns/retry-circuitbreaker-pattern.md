---
name: retry-circuitbreaker-pattern
description: 微服务弹性模式：重试（Retry）与断路器（Circuit Breaker）实战指南
tags: [architecture, microservice, pattern, resilience, retry, circuit-breaker, spring-cloud]
---

## 概述

在微服务架构中，服务间调用不可避免会出现网络抖动、服务暂时不可用等问题。**重试（Retry）** 和 **断路器（Circuit Breaker）** 是保证系统弹性的核心模式。重试处理瞬态故障，断路器防止雪崩效应。

## 选型对比

| 组件 | 重试 | 断路器 | 限流 | 超时 | 舱壁隔离 | 适用场景 |
|------|------|--------|------|------|---------|----------|
| **Resilience4j** | ✅ | ✅ | ✅ | ✅ | ✅ | Spring Boot 3.x / 非 Spring Cloud 项目 |
| **Spring Retry** | ✅ | ❌ | ❌ | ❌ | ❌ | 简单重试场景 |
| **Sentinel** | ❌ | ✅（熔断降级） | ✅ | ✅ | ✅ | 阿里系、流量治理 |
| **Hystrix（已停维）** | ❌ | ✅ | ✅ | ✅ | ✅ | 遗留项目（不建议新项目使用） |

## 1. Resilience4j 实战（推荐方案）

### 依赖配置

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### 配置（application.yml）

```yaml
resilience4j:
  # === 重试配置 ===
  retry:
    configs:
      default:
        maxRetryAttempts: 3                   # 最大重试次数
        waitDuration: 500ms                   # 重试间隔（固定）
        exponentialBackoffMultiplier: 2       # 指数退避倍数（500ms → 1s → 2s）
        enableExponentialBackoff: true
        retryExceptions:
          - org.springframework.web.client.HttpServerErrorException  # 5xx 错误重试
          - java.net.ConnectException
          - java.net.SocketTimeoutException
        ignoreExceptions:
          - org.springframework.web.client.HttpClientErrorException  # 4xx 错误不重试
        failAfterMaxRetries: true
    instances:
      userServiceRetry:
        baseConfig: default
        maxRetryAttempts: 3

  # === 断路器配置 ===
  circuitbreaker:
    configs:
      default:
        slidingWindowType: COUNT_BASED        # 滑动窗口类型（COUNT_BASED / TIME_BASED）
        slidingWindowSize: 10                  # 滑动窗口大小（10 次请求）
        minimumNumberOfCalls: 5                # 最少调用数（低于此值不触发断路）
        failureRateThreshold: 50               # 失败率阈值（50%）
        waitDurationInOpenState: 30s           # 断路器打开持续时间
        permittedNumberOfCallsInHalfOpenState: 3  # 半开状态允许的请求数
        recordExceptions:
          - org.springframework.web.client.HttpServerErrorException
          - java.util.concurrent.TimeoutException
          - java.net.ConnectException
        ignoreExceptions:
          - org.springframework.web.client.HttpClientErrorException
    instances:
      userServiceCB:
        baseConfig: default
        slidingWindowSize: 10
        failureRateThreshold: 40               # 40% 失败率即打开断路器

  # === 超时配置 ===
  timelimiter:
    configs:
      default:
        timeoutDuration: 5s                    # 超时时间
        cancelRunningFuture: true              # 超时后取消正在执行的 Future
    instances:
      userServiceTimeout:
        baseConfig: default
        timeoutDuration: 3s

  # === 舱壁隔离 ===
  bulkhead:
    configs:
      default:
        maxConcurrentCalls: 10                 # 最大并发数
        maxWaitDuration: 500ms                 # 等待队列超时
    instances:
      userServiceBulkhead:
        baseConfig: default
        maxConcurrentCalls: 20

# Feign 客户端集成
spring.cloud.openfeign.circuitbreaker:
  enabled: true
```

### Java 注解使用

```java
@Service
@Slf4j
public class UserServiceClient {

    @Autowired
    private RestTemplate restTemplate;

    /**
     * 重试：网络抖动自动重试
     */
    @Retry(name = "userServiceRetry", fallbackMethod = "getUserFallback")
    @TimeLimiter(name = "userServiceTimeout")
    @CircuitBreaker(name = "userServiceCB", fallbackMethod = "getUserFallback")
    @Bulkhead(name = "userServiceBulkhead", type = Bulkhead.Type.THREADPOOL)
    public CompletableFuture<User> getUser(Long userId) {
        return CompletableFuture.completedFuture(
            restTemplate.getForObject(
                "http://user-service/api/users/{id}",
                User.class, userId
            )
        );
    }

    /**
     * 降级方法（参数需与原始方法一致 + Throwable 参数）
     */
    public CompletableFuture<User> getUserFallback(Long userId, Throwable t) {
        log.warn("获取用户降级: userId={}, error={}", userId, t.getMessage());
        return CompletableFuture.completedFuture(
            User.builder()
                .id(userId)
                .name("未知用户")
                .available(false)
                .build()
        );
    }
}
```

### 编程式使用（更灵活）

```java
@Service
public class OrderService {

    private final Retry retry;
    private final CircuitBreaker circuitBreaker;

    public OrderService(RetryRegistry retryRegistry,
                        CircuitBreakerRegistry cbRegistry) {
        this.retry = retryRegistry.retry("orderServiceRetry");
        this.circuitBreaker = cbRegistry.circuitBreaker("orderServiceCB");
    }

    public Order createOrder(OrderRequest request) {
        // 组合使用：先断路器再重试
        Supplier<Order> decorated = Decorators.ofSupplier(
            () -> callPaymentService(request)
        )
        .withCircuitBreaker(circuitBreaker)
        .withRetry(retry)
        .withFallback(
            Arrays.asList(TimeoutException.class, IllegalArgumentException.class),
            e -> Order.builder()
                .status(OrderStatus.PAYMENT_PENDING)
                .remark("支付服务暂时不可用")
                .build()
        )
        .decorate();

        return decorated.get();
    }

    private Order callPaymentService(OrderRequest request) {
        // 调用支付服务
        return paymentClient.pay(request);
    }
}
```

## 2. Spring Retry（轻量级重试）

### 依赖

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### 配置与使用

```java
@EnableRetry
@SpringBootApplication
public class Application {}

@Service
@Slf4j
public class DatabaseService {

    @Retryable(
        retryFor = {DataAccessException.class, DeadlockLoserDataAccessException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 5000)
    )
    public void saveOrder(Order order) {
        // 保存订单（可能因死锁失败）
        orderRepository.save(order);
    }

    @Recover
    public void recover(DataAccessException e, Order order) {
        log.error("保存订单失败，已重试耗尽: orderId={}, error={}", order.getId(), e.getMessage());
        // 发送到死信队列或人工处理
        deadLetterQueue.send(order, e);
    }
}
```

## 3. Sentinel 熔断降级（阿里系方案）

### 配置文件

```yaml
spring:
  cloud:
    sentinel:
      enabled: true
      transport:
        dashboard: localhost:8080
      datasource:
        ds1:
          nacos:
            server-addr: localhost:8848
            data-id: sentinel-rules.json
            rule-type: flow
```

### 熔断规则（代码配置）

```java
@Configuration
public class SentinelConfig {

    @PostConstruct
    public void initDegradeRules() {
        List<DegradeRule> rules = new ArrayList<>();

        DegradeRule rule = new DegradeRule("getUser")
            .setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_RATIO)  // 异常比例
            .setCount(0.5)              // 50% 异常率触发熔断
            .setTimeWindow(30);         // 熔断时长 30s
        // 其他熔断策略：
        // DEGRADE_GRADE_EXCEPTION_COUNT  — 异常数（分钟级）
        // DEGRADE_GRADE_RT               — 平均响应时间（ms）

        DegradeRule slowRule = new DegradeRule("getUser")
            .setGrade(RuleConstant.DEGRADE_GRADE_RT)
            .setCount(2000)              // RT > 2000ms 熔断
            .setTimeWindow(30)
            .setMinRequestAmount(5);     // 最少请求数（5次后才触发）

        rules.add(rule);
        rules.add(slowRule);
        DegradeRuleManager.loadRules(rules);
    }
}
```

### 注解使用

```java
@Service
public class ProductService {

    @SentinelResource(
        value = "getProduct",
        fallback = "getProductFallback",
        fallbackClass = ProductServiceFallback.class,   // 全局降级类
        blockHandler = "getProductBlockHandler",          // 限流处理
        exceptionsToIgnore = {IllegalArgumentException.class}
    )
    public Product getProduct(Long id) {
        return productClient.getProduct(id);
    }
}
```

## 4. 模式组合实战

### 超时 + 重试 + 断路器 + 降级（完整链路）

```java
@Service
@Slf4j
public class ResilientOrderService {

    @Autowired
    private InventoryClient inventoryClient;

    /**
     * 完整弹性策略：
     * 1. TimeLimiter（超时控制）         → 3s 超时
     * 2. Retry（重试）                   → 指数退避重试 3 次
     * 3. CircuitBreaker（断路器）        → 50% 失败率熔断 30s
     * 4. Bulkhead（舱壁）                → 最大 10 并发
     * 5. Fallback（降级）                → 返回降级响应
     */
    @Bulkhead(name = "inventoryBulkhead", type = Bulkhead.Type.SEMAPHORE)
    @CircuitBreaker(name = "inventoryCB", fallbackMethod = "checkStockFallback")
    @TimeLimiter(name = "inventoryTimeout")
    @Retry(name = "inventoryRetry")
    public CompletableFuture<StockResult> checkStock(Long productId, Integer quantity) {
        return CompletableFuture.completedFuture(
            inventoryClient.checkStock(productId, quantity)
        );
    }

    public CompletableFuture<StockResult> checkStockFallback(
            Long productId, Integer quantity, Throwable t) {

        log.warn("库存查询降级: productId={}, error={}", productId, t.getMessage());

        // 判断降级原因
        if (t instanceof CircuitBreakerOpenException) {
            // 断路器已打开，快速失败
            return CompletableFuture.completedFuture(
                StockResult.unknown("库存服务异常，稍后重试")
            );
        } else if (t instanceof TimeoutException) {
            // 超时，返回保守判断（假设有货）
            return CompletableFuture.completedFuture(
                StockResult.available(quantity)
            );
        }

        return CompletableFuture.completedFuture(
            StockResult.unavailable("库存服务不可用")
        );
    }
}
```

## 5. 断路器状态变化监听

```java
@Component
@Slf4j
public class CircuitBreakerEventListener {

    public CircuitBreakerEventListener(CircuitBreakerRegistry registry) {
        registry.getAllCircuitBreakers().forEach(cb -> {
            cb.getEventPublisher()
                .onStateTransition(event -> {
                    log.warn("断路器状态变化: name={}, from={}, to={}",
                        event.getCircuitBreakerName(),
                        event.getStateTransition().getFromState(),
                        event.getStateTransition().getToState());

                    // 状态变化告警
                    if (event.getStateTransition().getToState() == CircuitBreaker.State.OPEN) {
                        alertService.sendAlert("断路器打开: " + event.getCircuitBreakerName());
                    }
                    if (event.getStateTransition().getToState() == CircuitBreaker.State.HALF_OPEN) {
                        log.info("断路器半开，即将尝试恢复: {}", event.getCircuitBreakerName());
                    }
                })
                .onError(event -> {
                    log.error("断路器记录错误: name={}, count={}",
                        event.getCircuitBreakerName(),
                        event.getNumberOfBufferedCalls());
                })
                .onSuccess(event -> {
                    if (event.getCircuitBreakerName().contains("degraded")) {
                        log.info("降级服务恢复成功: {}", event.getCircuitBreakerName());
                    }
                });
        });
    }
}
```

## 注意事项

1. **重试幂等性**：只有幂等接口才能自动重试（GET 安全、PUT 幂等），POST 创建操作需谨慎
2. **重试风暴**：多个服务级联重试可能导致请求放大多倍，建议在最外层（Gateway 或 Feign 层）统一配置重试
3. **断路器参数调优**：
   - 滑动窗口太小 → 误判频繁（抖动触发熔断）
   - 滑动窗口太大 → 反应迟钝（故障蔓延）
   - 建议初始值：10（COUNT_BASED）或 60s（TIME_BASED），根据业务压测调整
4. **超时时间**：建议设置比接口 P99 响应时间略大（如 P99=2s，超时设 3s），同时考虑重试累积时间
5. **降级返回值**：降级方法应返回业务可接受的值，如空列表、默认值、缓存数据，避免抛异常
6. **Feign 自动集成**：`spring.cloud.openfeign.circuitbreaker.enabled=true` 后，Feign 客户端自动包装 CircuitBreaker
7. **日志链路追踪**：降级/重试时保留 TraceId，方便排查问题链路
8. **监控告警**：断路器打开时必须告警⚠️，半开恢复成功或失败也应有监控
9. **测试方法**：使用 Resilience4j 的 `CircuitBreakerStateMachine` 手动切换状态进行测试
10. **Sentinel vs Resilience4j**：Spring Cloud Alibaba 生态选 Sentinel，标准 Spring Cloud 选 Resilience4j
