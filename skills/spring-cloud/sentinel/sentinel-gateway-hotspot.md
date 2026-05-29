---
name: sentinel-gateway-hotspot
description: Sentinel 参数热点限流与 Spring Cloud Gateway 集成实战
tags: [spring-cloud, sentinel, gateway, hotspot, param-flow]
---

## Sentinel 参数热点限流与 Gateway 集成

Sentinel 参数热点限流（Hotspot Param Flow Control）针对**特定参数值**精细控制流量，区别于传统 QPS 限流。结合 Spring Cloud Gateway 可实现对 API 路径、Header、IP 等维度的精准管控。

## 一、参数热点限流

### 核心概念

参数热点限流对资源的**特定参数**进行流量控制，例如：
- 对同一个商品 ID 的查询请求限流（防止热点商品冲垮系统）
- 对同一个用户的请求限流（防刷）
- 对同一个 IP 的访问限流

### 规则配置

```java
// 代码定义热点规则
public void initHotParamRule() {
    ParamFlowRule rule = new ParamFlowRule("getOrder")
        // 对第 1 个参数（商品 ID）进行限流
        .setParamIdx(0)
        // QPS 阈值 10
        .setCount(10)
        // 统计窗口时长（秒）
        .setDurationInSec(1)
        // 参数例外项：特定商品 ID 单独限流
        .setParamFlowItemList(Collections.singletonList(
            new ParamFlowItem("hot-item-123", 2, 2.0)  // 热点 item 限流 QPS=2
        ));

    ParamFlowRuleManager.loadRules(Collections.singletonList(rule));
}
```

### Nacos 动态配置

```json
// Nacos dataId: sentinel-hotspot-rules.json
[
    {
        "resource": "getOrder",
        "paramIdx": 1,
        "count": 10,
        "grade": 1,
        "durationInSec": 1,
        "controlBehavior": 0,
        "maxQueueingTimeMs": 500,
        "paramFlowItemList": [
            {
                "itemType": 0,
                "itemValue": "hot-item-456",
                "threshold": 2
            }
        ]
    }
]
```

```yaml
# 配置 Nacos 数据源加载热点规则
spring:
  cloud:
    sentinel:
      datasource:
        hotspot:
          nacos:
            server-addr: ${nacos.addr}
            data-id: sentinel-hotspot-rules
            group: SENTINEL_GROUP
            rule-type: param-flow
            data-type: json
```

### 参数索引说明

```java
@GetMapping("/order/{id}")
@SentinelResource(value = "getOrder")
public Result getOrder(
    @PathVariable Long id,          // paramIdx=0
    @RequestParam String userId,    // paramIdx=1
    @RequestHeader("X-Region") String region  // paramIdx=2
) {
    // 热点规则指定 paramIdx=0 → 对 id 参数限流
    // paramIdx=1 → 对 userId 限流
}
```

### 多个参数组合

```java
// 对参数组合 (id, userId) 限流
ParamFlowRule rule = new ParamFlowRule("searchProduct")
    .setParamIdx(0)        // 入口参数索引
    .setCount(5)
    .setParamFlowItemList(Arrays.asList(
        new ParamFlowItem("1001", 1, 2.0),    // id=1001+任何userId → QPS=2
        new ParamFlowItem("2002", 1, 1.0)     // id=2002+任何userId → QPS=1
    ));
```

## 二、Spring Cloud Gateway + Sentinel 集成

### 依赖配置

```xml
<!-- Spring Cloud Alibaba Sentinel + Gateway -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-spring-cloud-gateway-adapter</artifactId>
</dependency>
```

### 配置

```yaml
spring:
  cloud:
    sentinel:
      enabled: true
      transport:
        dashboard: localhost:8858      # Sentinel Dashboard
        port: 8719                      # 客户端上报端口
      # Gateway 路由限流
      scg:
        fallback:
          mode: response                # 限流响应模式
          response-status: 429
          response-body: '{"code":429,"msg":"Too Many Requests"}'
        order:
          -1000                         # 过滤器优先级（默认 -1000）
```

### Gateway API 分组定义

```java
@Configuration
public class GatewayApiDefineConfig {

    @PostConstruct
    public void initApiGroups() {
        // 将 /order/** 路径定义为 API 组
        Set<ApiDefinition> apis = new HashSet<>();
        apis.add(new ApiDefinition("order_api")
            .setPredicateItems(new HashSet<>(Arrays.asList(
                new ApiPathPredicateItem()
                    .setPattern("/order/**")     // 路径匹配
                    .setMatchStrategy(SentinelGatewayConstants.URL_MATCH_STRATEGY_PREFIX),
                new ApiPathPredicateItem()
                    .setPattern("/api/order/**")
                    .setMatchStrategy(SentinelGatewayConstants.URL_MATCH_STRATEGY_PREFIX)
            )))
        );
        apis.add(new ApiDefinition("user_api")
            .setPredicateItems(new HashSet<>(Arrays.asList(
                new ApiPathPredicateItem()
                    .setPattern("/user/**")
                    .setMatchStrategy(SentinelGatewayConstants.URL_MATCH_STRATEGY_PREFIX)
            )))
        );
        GatewayApiDefinitionManager.loadApiDefinitions(apis);
    }
}
```

### Gateway 路由限流规则

```java
@PostConstruct
public void initGatewayRules() {
    Set<GatewayFlowRule> rules = new HashSet<>();

    // 对 API 组限流
    rules.add(new GatewayFlowRule("order_api")
        .setResourceMode(SentinelGatewayConstants.RESOURCE_MODE_CUSTOM_API_NAME)
        .setGrade(RuleConstant.FLOW_GRADE_QPS)
        .setCount(100)                       // QPS 100
        .setIntervalSec(1)
        .setBurst(20)                        // 允许 20 的突发
        .setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER)
        .setMaxQueueingTimeMs(500));         // 排队等待 500ms

    // 按路由 ID 限流
    rules.add(new GatewayFlowRule("book-service")
        .setResourceMode(SentinelGatewayConstants.RESOURCE_MODE_ROUTE_ID)
        .setGrade(RuleConstant.FLOW_GRADE_QPS)
        .setCount(50));

    // 按请求参数限流（对 userId 参数做热点限流）
    rules.add(new GatewayFlowRule("order_api")
        .setResourceMode(SentinelGatewayConstants.RESOURCE_MODE_CUSTOM_API_NAME)
        .setGrade(RuleConstant.FLOW_GRADE_PARAM_FLOW)
        .setCount(5)
        .setParamItem(new GatewayParamFlowItem()
            .setParseStrategy(SentinelGatewayConstants.PARAM_PARSE_STRATEGY_URL_PARAM)
            .setFieldName("userId")));        // 从 URL 参数获取

    GatewayRuleManager.loadRules(rules);
}
```

### 参数解析策略

```java
// 从不同位置提取限流参数
new GatewayParamFlowItem()
    .setParseStrategy(SentinelGatewayConstants.PARAM_PARSE_STRATEGY_CLIENT_IP)
    .setFieldName(null);                     // 按客户端 IP 限流

new GatewayParamFlowItem()
    .setParseStrategy(SentinelGatewayConstants.PARAM_PARSE_STRATEGY_HEADER)
    .setFieldName("X-User-Id");              // 从请求头提取

new GatewayParamFlowItem()
    .setParseStrategy(SentinelGatewayConstants.PARAM_PARSE_STRATEGY_URL_PARAM)
    .setFieldName("itemId");                 // 从 URL 参数提取

new GatewayParamFlowItem()
    .setParseStrategy(SentinelGatewayConstants.PARAM_PARSE_STRATEGY_HOST)
    .setFieldName(null);                     // 按请求 Host 限流

// 自定义参数解析
new GatewayParamFlowItem()
    .setParseStrategy(SentinelGatewayConstants.PARAM_PARSE_STRATEGY_COOKIE)
    .setFieldName("session_id");             // 从 Cookie 提取
```

## 三、Gateway 限流降级

### 全局异常处理

```java
@Configuration
public class SentinelGatewayConfig {

    @PostConstruct
    public void initBlockHandlers() {
        // 设置限流降级处理器
        BlockRequestHandler handler = (exchange, t) -> {
            // 记录限流日志
            log.warn("Gateway blocked: uri={}, client={}",
                exchange.getRequest().getURI(), exchange.getRequest().getRemoteAddress());

            // 根据异常类型返回不同响应
            String msg;
            if (t instanceof ParamFlowException) {
                msg = "热点参数限流";
            } else if (t instanceof FlowException) {
                msg = "请求过于频繁";
            } else if (t instanceof DegradeException) {
                msg = "服务降级";
            } else {
                msg = "系统繁忙";
            }

            ServerHttpResponse response = exchange.getResponse();
            response.setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            response.getHeaders().setContentType(MediaType.APPLICATION_JSON);
            return response.writeWith(Mono.just(response.bufferFactory()
                .wrapString("{\"code\":429,\"msg\":\"" + msg + "\"}",
                    StandardCharsets.UTF_8)));
        };

        GatewayCallbackManager.setBlockHandler(handler);
    }
}
```

### 自定义异常页

```yaml
spring:
  cloud:
    sentinel:
      scg:
        fallback:
          mode: redirect              # 重定向到错误页面
          redirect-url: /error/429.html
```

## 四、生产实践

### 4.1 热点商品限流方案

```java
// 针对高频查询商品接口做热点限流
@RestController
public class ProductController {

    @GetMapping("/product/{id}")
    @SentinelResource(
        value = "queryProduct",
        blockHandler = "hotBlockHandler",
        fallback = "hotFallback"
    )
    public Result<ProductVO> queryProduct(@PathVariable Long id) {
        return productService.getById(id);
    }

    // 热点限流（参数维度）
    public Result<ProductVO> hotBlockHandler(Long id, BlockException e) {
        // 返回缓存数据 + 提示
        ProductVO cached = cacheService.get("hot:product:" + id);
        if (cached != null) {
            return Result.success(cached).msg("当前商品访问量过高，返回缓存数据");
        }
        return Result.error(429, "当前商品访问量过高，请稍后重试");
    }

    // 降级（后端异常）
    public Result<ProductVO> hotFallback(Long id, Throwable t) {
        return Result.error(503, "商品查询暂不可用");
    }
}
```

### 4.2 防刷（同一用户限流）

```java
// Gateway 层按 userId 限流
// 每个 userId 每秒最多 10 次请求
@PostConstruct
public void initAntiSpamRules() {
    GatewayFlowRule rule = new GatewayFlowRule("order_api")
        .setResourceMode(RESOURCE_MODE_CUSTOM_API_NAME)
        .setGrade(FLOW_GRADE_PARAM_FLOW)
        .setCount(10)
        .setIntervalSec(1)
        .setParamItem(new GatewayParamFlowItem()
            .setParseStrategy(PARAM_PARSE_STRATEGY_URL_PARAM)
            .setFieldName("userId"));

    // 用户等级例外：VIP 用户不限流
    rule.setParamFlowItemList(Collections.singletonList(
        new ParamFlowItem("vip-001", 1, Integer.MAX_VALUE)
    ));

    GatewayRuleManager.loadRules(Collections.singleton(rule));
}
```

### 4.3 灰度场景下的限流

```yaml
# 灰度版本接口限流策略不同
spring:
  cloud:
    sentinel:
      scg:
        fallback:
          mode: response
      datasource:
        flow:
          nacos:
            data-id: sentinel-gateway-flow-rules
            rule-type: gw-flow
        # 灰度规则（带版本号标签）
        flow-gray:
          nacos:
            data-id: sentinel-gateway-flow-gray-rules
            rule-type: gw-flow
```

## 五、注意事项

### 1. 参数索引与配置一致性

```java
// ❌ 容易出错：修改方法签名后忘记更新规则
public Result query(Long id, String type) { }   // paramIdx 0=id, 1=type

// 改成：
public Result query(Long id, String type, String version) { }
// 此时 paramIdx 1 变成了 version 而非 type → 规则失效
// ✅ 建议：变更方法签名后同步检查 param-flow 规则
```

### 2. 热点参数数据类型

Sentinel 热点参数类型必须实现 `equals()` 和 `hashCode()`：
- 基础类型（Integer, Long, String）自动支持
- 自定义类型需要确保正确重写 equals/hashCode

### 3. 与普通流控规则共存

```yaml
# 推荐策略：普通 QPS 限流 + 热点参数限流结合
# 1. 普通流控：接口总 QPS ≤ 1000（网关层）
# 2. 热点限流：单个商品 ID 的 QPS ≤ 10（应用层）
# 3. 系统保护：全局 Load/CPU 超过 80% 触发系统保护
```

### 4. Gateway 适配器版本

| Sentinel 版本 | 支持 Gateway 版本 | Spring Cloud 版本 |
|---------------|------------------|------------------|
| 1.8.6+ | Spring Cloud Gateway 4.x | 2023.0.x+ |
| 1.8.10 | Spring Cloud Gateway 4.x | 2023.0.x / 2024.0.x |
