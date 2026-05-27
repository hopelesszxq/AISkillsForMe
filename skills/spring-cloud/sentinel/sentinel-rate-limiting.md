---
name: sentinel-rate-limiting
description: Sentinel 限流熔断与系统保护实战
tags: [spring-cloud, sentinel, rate-limit, circuit-breaker]
---

## 核心概念

- **资源（Resource）**: 被保护的方法、接口或代码块
- **规则（Rule）**: 流控、降级、热点、系统、授权规则
- **Slot 链**: 责任链模式处理流量（NodeSelectorSlot → FlowSlot → DegradeSlot → ...）

## 流控规则配置

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
      datasource:
        # Nacos 动态配置源
        ds1:
          nacos:
            server-addr: localhost:8848
            dataId: sentinel-flow-rules
            rule-type: flow
```

### 限流模式

```java
@GetMapping("/order/{id}")
@SentinelResource(value = "getOrder", blockHandler = "handleBlock", 
                  fallback = "handleFallback")
public Result getOrder(@PathVariable Long id) {
    return orderService.getById(id);
}

// 限流处理（执行原方法前触发）
public Result handleBlock(Long id, BlockException e) {
    return Result.error(429, "请求过于频繁：" + e.getClass().getSimpleName());
}

// 熔断降级（执行原方法异常后触发）
public Result handleFallback(Long id, Throwable t) {
    return Result.error(503, "服务繁忙，请稍后重试");
}
```

### 流控效果

| 模式 | 场景 | 说明 |
|---|---|---|
| 快速失败 | 默认 | 直接抛出 FlowException |
| Warm Up | 秒杀、突增流量 | 预热期逐步提升阈值，codeFactor 默认 3 |
| 排队等待 | 削峰填谷 | 匀速排队处理，阈值=排队等待时间 |
| 冷启动+排队 | 大促场景 | 热身后再匀速处理 |

## 熔断降级

```java
// 熔断规则：慢调用比例
DegradeRule rule = new DegradeRule("getOrder")
    .setGrade(CircuitBreakerStrategy.SLOW_REQUEST_RATIO.getType())
    .setCount(0.5)        // 最大 RT 500ms
    .setTimeWindow(10)    // 熔断时长 10s
    .setStatIntervalMs(1000)
    .setMinRequestAmount(5)
    .setSlowRatioThreshold(0.2);  // 慢调用比例阈值 20%
```

## 热点参数限流

```java
// 对 user_id 参数热点限流
@SentinelResource(value = "getUserInfo", 
                  blockHandler = "handleBlockHot")
public User getUserInfo(@RequestParam("userId") Long userId) {
    return userService.get(userId);
}

// 规则：同一个 userId 每秒最多 10 次，超出限流
// 可针对特定参数值设置例外阈值
Map<String, ParamFlowItem> items = new HashMap<>();
items.put("888888", new ParamFlowItem().setCount(100));  // VIP 用户例外

ParamFlowRule rule = new ParamFlowRule("getUserInfo")
    .setParamIdx(0)
    .setCount(10)
    .setParamFlowItemList(items);
```

## 网关限流

```yaml
# gateway + sentinel 搭配
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/order/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                key-resolver: '#{@userKeyResolver}'
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
```

## 系统自适应保护

```yaml
# 系统规则：结合 Load、CPU、RT、吞吐量
# 当系统 Load1 > 5 且 RT 上升时自动降级
```

## 注意事项

1. **@SentinelResource 必须搭配 blockHandler**：否则限流时直接抛异常给前端
2. **规则持久化**：生产环境用 Nacos/Apollo 做数据源，避免重启丢失
3. **SphU.entry() 手动埋点**：非 Spring Bean 方法需要手动 try-finally exit
4. **Web 上下文自动埋点**：Spring Web 适配器自动拦截，注意排除健康检查路径
5. **Sentinel 与 Hystrix 共存**：不要同时引入，优先用 Sentinel
6. **日志配置**：Sentinel 日志独立目录，避免写满磁盘 `csp.sentinel.log.dir`
7. **性能开销**：单机 QPS < 10万时影响可忽略，超过建议调大 `statistic.max.rt`
