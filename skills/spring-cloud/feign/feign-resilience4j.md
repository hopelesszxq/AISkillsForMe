---
name: feign-resilience4j
description: OpenFeign 集成 Resilience4j：断路器、重试、限流器、舱壁隔离实战配置
tags: [spring-cloud, feign, resilience4j, circuit-breaker, retry, rate-limiter]
---

## 概述

Spring Cloud 2020.0.x 之后移除了 Hystrix，Resilience4j 成为官方推荐的容错方案。Feign 通过 `feign-circuitbreaker` 模块集成 Resilience4j，支持断路器（CircuitBreaker）、重试（Retry）、限流器（RateLimiter）、时间限制器（TimeLimiter）和舱壁隔离（Bulkhead）。

## 依赖配置

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<!-- Actuator 用于查看断路器状态 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

## 基础配置

### application.yml

```yaml
# 启用 Feign 断路器
spring.cloud.openfeign:
  circuitbreaker:
    enabled: true
    alphanumeric-ids:
      enabled: true  # 兼容 Spring Cloud 2023+

# 暴露断路器端点
management:
  endpoints:
    web:
      exposure:
        include: circuitbreakers,circuitbreakerevents,retries
```

### 全局 Resilience4j 配置

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-size: 10           # 滑动窗口大小（次数）
        minimum-number-of-calls: 5         # 最少调用数，低于该值不触发熔断
        sliding-window-type: COUNT_BASED   # COUNT_BASED | TIME_BASED
        failure-rate-threshold: 50         # 失败率阈值(%)
        wait-duration-in-open-state: 10s   # 熔断开启后等待时间
        permitted-number-of-calls-in-half-open-state: 3  # 半开状态允许请求数
        automatic-transition-from-open-to-half-open-enabled: true
        record-exceptions:
          - java.net.ConnectException
          - java.net.SocketTimeoutException
          - feign.FeignException.ServiceUnavailable
          - feign.FeignException.InternalServerError
        ignore-exceptions:
          - feign.FeignException.NotFound          # 404 不触发熔断
          - feign.FeignException.BadRequest         # 400 不触发熔断
    instances:
      user-service:
        base-config: default
        failure-rate-threshold: 40
        wait-duration-in-open-state: 30s

  retry:
    configs:
      default:
        max-retry-attempts: 3              # 最大重试次数
        wait-duration: 500ms               # 重试间隔
        exponential-backoff-multiplier: 2  # 指数退避倍数
        retry-exceptions:
          - java.net.ConnectException
          - java.net.SocketTimeoutException
          - feign.FeignException.ServiceUnavailable  # 503
        ignore-exceptions:
          - feign.FeignException.BadRequest
          - feign.FeignException.NotFound
    instances:
      order-service:
        base-config: default
        max-retry-attempts: 2

  timelimiter:
    configs:
      default:
        timeout-duration: 5s               # 超时时间
        cancel-running-future: true        # 超时是否取消任务

  bulkhead:
    configs:
      default:
        max-concurrent-calls: 20           # 最大并发数
        max-wait-duration: 500ms           # 等待队列超时
    instances:
      payment-service:
        max-concurrent-calls: 10

  ratelimiter:
    configs:
      default:
        limit-for-period: 100              # 周期内允许请求数
        limit-refresh-period: 1s           # 刷新周期
        timeout-duration: 0                # 限流时直接拒绝，不等待
    instances:
      sms-service:
        limit-for-period: 20
```

## Feign 客户端配置

```java
@FeignClient(name = "user-service", path = "/api/users")
public interface UserClient {

    @GetMapping("/{id}")
    User getUser(@PathVariable Long id);

    @PostMapping
    User createUser(@RequestBody User user);
}
```

### 自定义 Fallback

```java
@Component
@Slf4j
public class UserClientFallback implements UserClient {

    @Override
    public User getUser(Long id) {
        log.warn("Fallback: getUser({}) - 服务不可用", id);
        return User.builder()
                .id(id)
                .name("未知用户（降级）")
                .status(UserStatus.OFFLINE)
                .build();
    }

    @Override
    public User createUser(User user) {
        log.warn("Fallback: createUser - 服务不可用，消息入队列等待重试");
        // 投递到死信队列或本地存储，等待恢复后补偿
        throw new CircuitBreakerOpenException("user-service 熔断中，请求暂缓");
    }
}
```

```java
// Feign 客户端绑定 Fallback
@FeignClient(name = "user-service", path = "/api/users",
             fallback = UserClientFallback.class)
public interface UserClient {
    // ...
}
```

### FallbackFactory（获取异常原因）

```java
@Component
@Slf4j
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {

    @Override
    public UserClient create(Throwable cause) {
        log.error("Feign 调用失败，原因: {}", cause.getMessage());

        return new UserClient() {
            @Override
            public User getUser(Long id) {
                if (cause instanceof CircuitBreakerOpenException) {
                    return User.builder()
                            .id(id).name("熔断降级")
                            .status(UserStatus.OFFLINE).build();
                }
                if (cause instanceof RetryableException) {
                    // 重试耗尽
                    return User.builder()
                            .id(id).name("重试耗尽")
                            .status(UserStatus.ERROR).build();
                }
                return User.builder()
                        .id(id).name("服务异常")
                        .status(UserStatus.ERROR).build();
            }

            @Override
            public User createUser(User user) {
                throw new RuntimeException("无法创建用户：" + cause.getMessage(), cause);
            }
        };
    }
}
```

```java
@FeignClient(name = "user-service", path = "/api/users",
             fallbackFactory = UserClientFallbackFactory.class)
public interface UserClient {
    // ...
}
```

## 编程式使用 Resilience4j

当需要更细粒度的控制时，可以直接注入 Resilience4j 组件：

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderClient orderClient;
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final RetryRegistry retryRegistry;

    public Order getOrderWithRetry(Long orderId) {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry
                .circuitBreaker("order-service");
        Retry retry = retryRegistry.retry("order-service");

        // 组合使用：先重试，再熔断
        return Decorators.ofSupplier(() -> orderClient.getOrder(orderId))
                .withCircuitBreaker(circuitBreaker)
                .withRetry(retry)
                .withTimeLimiter(TimeLimiter.of(Duration.ofSeconds(3)))
                .decorate()
                .get();
    }
}
```

## 重试与断路器协同

### 关键顺序：重试在断路器内部

Feign + Resilience4j 的执行顺序是：**TimeLimiter → CircuitBreaker → Retry → Feign 调用**

这意味着：
- 重试失败计入断路器失败率
- 断路器开启后，重试不会执行
- TimeLimiter 在最外层兜底

### 配置建议

```yaml
# 让重试消耗在断路器窗口内，避免熔断过度
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-size: 20          # 加大窗口
        failure-rate-threshold: 60       # 提高阈值
  retry:
    configs:
      default:
        max-attempts: 2                  # 少重试，快速失败触发熔断
```

## 监控与运维

### Actuator 端点

```bash
# 查看所有断路器状态
curl http://localhost:8080/actuator/circuitbreakers

# 查看断路器事件
curl http://localhost:8080/actuator/circuitbreakerevents

# 重试事件
curl http://localhost:8080/actuator/retries
```

### 输出示例

```json
{
  "circuitBreakers": {
    "user-service": {
      "failureRate": 60.0,
      "state": "OPEN",
      "bufferedCalls": 10,
      "failedCalls": 6,
      "notPermittedCalls": 15
    }
  }
}
```

### Prometheus 指标

```yaml
management:
  metrics:
    tags:
      application: ${spring.application.name}
  endpoints:
    web:
      exposure:
        include: prometheus
```

通过 Resilience4j Micrometer 集成自动暴露以下指标：
- `resilience4j.circuitbreaker.state` — 断路器状态
- `resilience4j.circuitbreaker.calls` — 调用结果（success/failure/not_permitted）
- `resilience4j.retry.calls` — 重试调用次数

## 注意事项

1. **Feign + Retry 冲突**：禁用 Feign 内置重试（默认已改为永不重试），否则与 Resilience4j Retry 叠加导致重复调用
   ```yaml
   # 确认 Feign 内置重试已关闭
   spring.cloud.openfeign.client.config.default.retryer: feign.Retryer.NEVER_RETRY
   ```

2. **Fallback 异常透明**：Feign 的 Fallback 中抛出的异常会被 Resilience4j 捕获并记录为 `IGNORED_ERROR`，不影响断路器状态。如果需要让 Fallback 异常也计入熔断，使用 `FallbackFactory` 并抛出特定异常

3. **Spring Cloud 2023.x 的 `alphanumeric-ids`**：此版本要求 `alphanumeric-ids: true`，否则 Feign 客户端 ID 中的连字符导致 Resilience4j 实例名解析失败

4. **虚拟线程环境**：Java 21 虚拟线程下，`ThreadPoolBulkhead` 不适用（会创建平台线程），应使用 `SemaphoreBulkhead`：
   ```yaml
   resilience4j:
     bulkhead:
       instances:
         default:
           type: SEMAPHORE
   ```

5. **重试的幂等性**：仅对 GET/HEAD/DELETE 等幂等方法启用 Retry，POST/PUT 需业务端保证幂等或使用 `retry-exceptions` 精确控制
