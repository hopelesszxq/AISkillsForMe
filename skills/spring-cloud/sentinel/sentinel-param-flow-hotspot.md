---
name: sentinel-param-flow-hotspot
description: Sentinel 热点参数限流进阶实战 — 参数例外项、集群热点、自定义兜底策略、流量效果控制与商品秒杀等真实场景
tags: [spring-cloud, sentinel, hotspot, param-flow, rate-limiting, traffic-control]
---

## 概述

Sentinel 热点参数限流（Param Flow Control）针对**同一资源的不同参数值**执行差异化限流。与普通流控规则按资源总 QPS 限流不同，热点参数限流可以精确到参数的维度。

## 1. 热点规则核心配置

### 参数索引（`paramIdx`）

热点规则通过参数索引（从 0 开始）定位受保护的参数：

```java
// 被保护的接口
@SentinelResource("create_order")
public Order createOrder(Long userId, Long productId, Integer quantity) {
    // userId → paramIdx=0, productId → paramIdx=1, quantity → paramIdx=2
}
```

> **限制**：参数索引只有 `0`、`1`、`2`、`3` 四个有效值，超过 3 个参数时需设计参数对象并解析。

### 规则定义

```java
// 热点参数规则
ParamFlowRule rule = new ParamFlowRule("create_order")
    .setParamIdx(0)                  // 按 userId 限流
    .setCount(100)                   // 每个用户允许 100 QPS
    .setDurationInSec(1);            // 统计窗口（秒）

// 参数例外项（白名单/黑名单效果）
ParamFlowItem vipItem = new ParamFlowItem()
    .setObject("VIP_USER_001")       // 特定用户 ID
    .setClassType(String.class.getName())
    .setCount(500);                  // 该用户允许 500 QPS

ParamFlowItem throttledItem = new ParamFlowItem()
    .setObject("BLACKLIST_IP")
    .setClassType(String.class.getName())
    .setCount(0);                    // 该用户完全禁止调用

rule.setParamFlowItemList(Arrays.asList(vipItem, throttledItem));

ParamFlowRuleManager.loadRules(Collections.singletonList(rule));
```

### YAML 配置（通过 Nacos 动态推）

```yaml
# sentinel-hotspot-rules.json — Nacos dataId
[
  {
    "resource": "create_order",
    "paramIdx": 0,
    "count": 100,
    "durationInSec": 1,
    "grade": 1,
    "paramFlowItemList": [
      {"object": "VIP_USER_001", "classType": "java.lang.String", "count": 500},
      {"object": "BLACKLIST_IP", "classType": "java.lang.String", "count": 0}
    ]
  },
  {
    "resource": "search_product",
    "paramIdx": 1,
    "count": 50,
    "durationInSec": 1,
    "grade": 1,
    "burstCount": 10
  }
]
```

| 字段 | 说明 | 默认值 |
|------|------|--------|
| `paramIdx` | 参数索引，支持 0~3 | 必填 |
| `count` | 阈值，单机或集群 | 必填 |
| `durationInSec` | 统计窗口（秒） | 1 |
| `grade` | 阈值类型：1-QPS，0-并发线程数 | 1 |
| `burstCount` | 突发容忍 | 0 |
| `paramFlowItemList` | 参数例外项（按具体值设置不同阈值） | 空 |
| `clusterMode` | 是否集群模式 | false |
| `controlBehavior` | 流量效果控制（见下文） | 0（直接拒绝） |
| `maxQueueingTimeMs` | 排队模式下最大排队时间 | 500 |

## 2. 流量效果控制

### 直接拒绝（默认）

```java
// controlBehavior = 0
// 超过阈值直接抛出 BlockException
```

### 匀速排队（Throttling）

```java
// controlBehavior = 1 — 适用于削峰填谷
ParamFlowRule rule = new ParamFlowRule("create_order")
    .setParamIdx(0)
    .setCount(50)                // 50 QPS
    .setDurationInSec(1)
    .setControlBehavior(1)       // 匀速排队模式
    .setMaxQueueingTimeMs(500);  // 最大排队等待 500ms
```

**适用场景**：
- **秒杀抢购**：瞬时流量极高，通过排队平摊请求
- **优惠券领取**：控制每秒发放数量，避免数据库写压力
- **消息推送**：控制下游系统能承受的最大推送速率

### 预热（Warm Up）

```java
// controlBehavior = 2 — 适用于冷启动慢慢提升阈值
ParamFlowRule rule = new ParamFlowRule("search_product")
    .setParamIdx(1)
    .setCount(1000)
    .setControlBehavior(2)       // 预热模式
    .setWarmUpPeriodSec(10);     // 预热 10 秒
```

**适用场景**：
- **缓存冷启动**：系统刚重启后缓存为空，低温运行防止缓存穿透
- **数据库连接池预热**：连接池初始化完成前限制请求
- **JIT 预热**：给 JVM 编译优化留出时间

## 3. 集群热点参数限流

热点参数限流同样支持集群模式，解决单机流控不精确的问题：

```java
// Token Client 端配置
ParamFlowRule rule = new ParamFlowRule("create_order")
    .setParamIdx(0)
    .setCount(1000)
    .setClusterMode(true)                        // 开启集群模式
    .setClusterConfig(new ParamFlowClusterConfig()
        .setFlowId(100L)                         // 集群规则 ID（全局唯一）
        .setThresholdType(0)                     // 0-全局阈值, 1-单机均摊
        .setGlobalThreshold(5000)                // 集群全局阈值
        .setSampleCount(10)                      // 采样窗口数
        .setWindowIntervalMs(1000)               // 窗口总时长(ms)
    );
```

**集群热点提示**：
- Token Server 的性能会成为瓶颈，建议独立部署 Token Server
- 全局模式（`thresholdType=0`）慎用——单个热点 key 的突发流量可能打满 Token Server
- 推荐使用单机均摊模式（`thresholdType=1`）+ 全局兜底

## 4. 实战场景：商品秒杀

```java
@RestController
@RequestMapping("/seckill")
public class SeckillController {

    @SentinelResource(
        value = "seckill_order",
        blockHandler = "seckillBlockHandler",
        fallback = "seckillFallback"
    )
    @PostMapping("/create")
    public Result createSeckillOrder(
            @RequestParam Long userId,
            @RequestParam Long skuId,
            @RequestParam Integer quantity) {

        // 业务逻辑
        return seckillService.createOrder(userId, skuId, quantity);
    }

    // 限流降级（BlockException）
    public Result seckillBlockHandler(
            Long userId, Long skuId, Integer quantity,
            BlockException ex) {
        if (ex instanceof ParamFlowException) {
            // 热点参数限流
            Object hotParam = ((ParamFlowException) ex).getParams();
            log.warn("热点限流: skuId={}, userId={}", hotParam, userId);
            return Result.fail(429, "当前商品太火爆，请稍后再试");
        }
        return Result.fail(429, "系统繁忙，请稍后再试");
    }

    // 降级兜底（业务异常）
    public Result seckillFallback(
            Long userId, Long skuId, Integer quantity,
            Throwable ex) {
        log.error("秒杀异常", ex);
        return Result.fail(500, "系统异常，请联系客服");
    }
}
```

### 秒杀场景热点规则配置

```java
// 规则 1：每个 skuId 每秒最多 100 QPS（防止单个商品被打爆）
ParamFlowRule skuRule = new ParamFlowRule("seckill_order")
    .setParamIdx(1)                // skuId 在第二个参数
    .setCount(100)
    .setControlBehavior(1)         // 匀速排队
    .setMaxQueueingTimeMs(200);    // 排队最多 200ms

// 规则 2：特价商品例外（热门 SKU 更严格）
ParamFlowItem hotSku = new ParamFlowItem()
    .setObject("HOT_SKU_001")
    .setClassType("java.lang.Long")
    .setCount(10);                 // 这个 SKU 只允许 10 QPS

// 规则 3：黑名单用户完全拦截
ParamFlowItem blackUser = new ParamFlowItem()
    .setObject("MALICIOUS_USER")
    .setClassType("java.lang.Long")
    .setCount(0);

// 规则 4：UID 维度限制（每个用户限制速率）
ParamFlowRule userRule = new ParamFlowRule("seckill_order")
    .setParamIdx(0)
    .setCount(3)                   // 每个用户每秒最多 3 次
    .setDurationInSec(1)
    .setControlBehavior(2)         // 预热
    .setWarmUpPeriodSec(5);
```

## 5. 与普通流控规则配合

```
流量入口 → Sentinel 总体 QPS 流控（普通规则）
            ↓
        热点参数限流（参数维度）
            ↓
        业务处理
```

```java
// 两层保护策略
// 第一层：整体流控（保护系统整体负载）
FlowRule totalFlow = new FlowRule("seckill_order")
    .setCount(5000)                // 整体 5000 QPS
    .setGrade(RuleConstant.FLOW_GRADE_QPS);

// 第二层：热点流控（保护单个热点）
ParamFlowRule hotParamRule = new ParamFlowRule("seckill_order")
    .setParamIdx(1)
    .setCount(100);                // 单个 sku 100 QPS

// 加载规则
FlowRuleManager.loadRules(Collections.singletonList(totalFlow));
ParamFlowRuleManager.loadRules(Collections.singletonList(hotParamRule));
```

## 6. 自定义热点参数解析

默认按参数位置匹配，但可以自定义解析逻辑实现更灵活的匹配：

```java
// 实现 ParamFlowArgumentResolver 接口来解析自定义参数
public class OrderParamResolver implements RequestArgumentResolver {

    @Override
    public Object resolveArgument(
            Method method, Object[] args, int paramIdx) {
        if (paramIdx == 0 && args[0] instanceof OrderRequest) {
            // 从请求对象中提取 userId 作为限流维度
            return ((OrderRequest) args[0]).getUserId();
        }
        return args[paramIdx];
    }
}

// 注册到 Sentinel
WebCallbackManager.setRequestOriginParser(request -> {
    // 从请求头中提取来源
    return request.getHeader("X-Origin");
});
```

## 注意事项

1. **参数类型限制**：热点参数支持基本类型、String 和 Long。自定义对象需要实现 `ParamFlowArgumentResolver` 或拆解为基本参数
2. **参数例外项过多影响性能**：`paramFlowItemList` 长度达到 1 万以上时，查找性能显著下降。建议例外项数量控制在 1000 以内，大量例外用 Nacos 配置动态管理
3. **集群热点性能**：集群热点模式下，Token Server 需要处理每个参数的计数器，热点 key 数量过多可能导致 Server 端 OOM。建议 `maxHotParamCount` 控制在 5000 以内
4. **热点参数与熔断的配合**：热点参数限流只控制 QPS/并发，不处理慢调用熔断。建议与熔断降级规则配合使用：
   ```java
   DegradeRule degradeRule = new DegradeRule("seckill_order")
       .setCount(50)                // RT > 50ms
       .setGrade(RuleConstant.DEGRADE_GRADE_RT)
       .setTimeWindow(10);          // 熔断 10 秒
   ```
5. **Spring Cloud Gateway 中的热点参数限流**：Gateway 场景参数索引对应位置在请求体或请求头中，需配合 `SentinelGatewayFilter` 使用，不支持 URL 参数的自动解析（需要手动提取 ServerWebExchange 中的参数）
6. **`SphU.entry()` 手动埋点**：如果使用 `@SentinelResource` 无法满足需求（如需动态参数索引），可手动调用 `SphU.entry("resource", EntryType.IN, 1, args)` 传递热点参数
