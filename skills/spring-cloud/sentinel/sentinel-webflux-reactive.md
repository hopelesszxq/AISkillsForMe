---
name: sentinel-webflux-reactive
description: Sentinel 在 Spring WebFlux / WebClient 响应式场景中的集成与配置
tags: [spring-cloud, sentinel, webflux, reactive, circuit-breaker]
---

## 概述

Sentinel 提供对 Spring WebFlux（Spring 5+）和 Spring WebClient 的完整支持。在响应式编程模型中，流控和熔断的实现方式与传统 Servlet 不同，需要理解其工作机制。

## 1. WebFlux 适配器

### 引入依赖

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-spring-webflux-adapter</artifactId>
    <version>1.8.10</version>
</dependency>
```

### 基础配置

```java
@Configuration
public class SentinelWebFluxConfig {

    @PostConstruct
    public void init() {
        // 初始化 WebFlux 回调处理器
        WebFluxCallbackManager.setBlockHandler((exchange, ex) -> {
            exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            return exchange.getResponse()
                .writeWith(Mono.just(
                    exchange.getResponse()
                        .bufferFactory()
                        .wrap("{\"code\":429,\"msg\":\"rate limited\"}".getBytes())
                ));
        });
    }
}
```

### 资源命名规则

```yaml
# 自定义资源名称提取策略
# 默认规则：请求方法 + 路径（如 GET:/api/users/{id})
# 可通过 WebFluxConfigurer 自定义
sentinel:
  webflux:
    resource-prefix: ""        # 资源名前缀（默认空）
    resource-trim-mode: ADD    # ADD: 添加前缀, CUT: 裁剪前缀
```

```java
// 自定义资源名提取
@Bean
public WebFluxConfigurer sentinelWebFluxConfigurer() {
    return new WebFluxConfigurer() {
        @Override
        public void configureResourceExtractor(
                ResourceExtractorRegistry registry) {
            registry.register((exchange, originParser) -> {
                String method = exchange.getRequest().getMethodValue();
                String path = exchange.getRequest().getPath().value();
                // 自定义资源名
                return method + ":" + path;
            });
        }
    };
}
```

## 2. 响应式流控规则配置

```java
@Configuration
public class SentinelRulesConfig {

    @PostConstruct
    public void initFlowRules() {
        // 流控规则（QPS）
        List<FlowRule> flowRules = new ArrayList<>();
        FlowRule rule = new FlowRule();
        rule.setResource("GET:/api/orders");
        rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rule.setCount(100);          // 限制 100 QPS
        rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER);
        rule.setMaxQueueingTimeMs(500);  // 匀速排队
        flowRules.add(rule);
        FlowRuleManager.loadRules(flowRules);
    }
}
```

### Reactive 特有的熔断策略

```java
// 熔断降级规则
List<DegradeRule> degradeRules = new ArrayList<>();
DegradeRule degradeRule = new DegradeRule();
degradeRule.setResource("GET:/api/products");
degradeRule.setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_RATIO);
degradeRule.setCount(0.5);           // 异常比例 > 50%
degradeRule.setTimeWindow(10);       // 熔断 10 秒
degradeRule.setMinRequestAmount(10); // 最少请求数
degradeRules.add(degradeRule);
DegradeRuleManager.loadRules(degradeRules);

// 热点参数限流（对特定参数做流控）
List<ParamFlowRule> paramRules = new ArrayList<>();
ParamFlowRule paramRule = new ParamFlowRule();
paramRule.setResource("GET:/api/search");
paramRule.setParamIdx(0);              // 第一个参数（query）
paramRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
paramRule.setCount(10);                // 每个 query 限制 10 QPS
paramRules.add(paramRule);
ParamFlowRuleManager.loadRules(paramRules);
```

## 3. WebClient 集成

WebClient 是 Spring WebFlux 生态的核心 HTTP 客户端，Sentinel 提供专门的适配拦截器。

```java
import com.alibaba.csp.sentinel.adapter.spring.webflux.SentinelWebClientCustomizer;

@Configuration
public class SentinelWebClientConfig {

    @Bean
    public WebClient webClient() {
        return WebClient.builder()
            .filter(new SentinelWebClientFilter())
            .build();
    }

    // 或使用自定义器
    @Bean
    public SentinelWebClientCustomizer sentinelWebClientCustomizer() {
        return new SentinelWebClientCustomizer();
    }
}
```

```java
// 自定义 WebClient 资源名
@Bean
public WebClient webClient() {
    ExchangeFilterFunction sentinelFilter = ExchangeFilterFunction.ofResponseProcessor(
        clientResponse -> Mono.just(clientResponse)
    );

    return WebClient.builder()
        .filter((request, next) -> {
            String resourceName = request.method().name() + ":" + request.url().getPath();
            try (Entry entry = SphU.entry(resourceName)) {
                return next.exchange(request);
            } catch (BlockException e) {
                return Mono.error(new RuntimeException("Blocked by Sentinel"));
            }
        })
        .build();
}
```

### 响应式熔断使用示例

```java
@Service
public class OrderReactiveService {

    private final WebClient webClient;

    // Sentinel + WebClient 完整示例
    public Mono<Order> getOrder(Long orderId) {
        return webClient.get()
            .uri("/api/orders/{id}", orderId)
            .retrieve()
            .bodyToMono(Order.class)
            .transform(mono -> {
                // 使用 Sentinel 包装响应式链路
                return SentinelResourceTransformer
                    .of(mono, "getOrder:" + orderId)
                    .fallback(ex -> {
                        log.warn("getOrder blocked, fallback to cache");
                        return getOrderFromCache(orderId);
                    });
            })
            .timeout(Duration.ofSeconds(5))
            .retryWhen(Retry.backoff(3, Duration.ofMillis(100)));
    }
}
```

## 4. 响应式 Fallback 处理

### 全局 Fallback

```yaml
# application.yml
sentinel:
  webflux:
    block-page: classpath:/block.html  # 流控提示页面
    block-json: |
      {"code": 429, "message": "请求太频繁，请稍后再试"}
```

```java
// 编程式全局 Fallback
WebFluxCallbackManager.setBlockHandler((exchange, ex) -> {
    String origin = exchange.getRequest().getHeaders()
        .getFirst("X-Origin");
    if ("internal".equals(origin)) {
        // 内部流量降低响应级别
        exchange.getResponse().setStatusCode(HttpStatus.SERVICE_UNAVAILABLE);
        return exchange.getResponse()
            .writeWith(Mono.just(
                exchange.getResponse().bufferFactory()
                    .wrap("{\"code\":503}".getBytes())
            ));
    }
    // 外部流量直接返回 429
    exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
    return exchange.getResponse()
        .writeWith(Mono.just(
            exchange.getResponse().bufferFactory()
                .wrap("{\"code\":429}".getBytes())
        ));
});

// 设置异常处理器
WebFluxCallbackManager.setExceptionHandler((exchange, ex) -> {
    log.error("Sentinel exception: {}", ex.getMessage());
    exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
    return exchange.getResponse()
        .writeWith(Mono.just(
            exchange.getResponse().bufferFactory()
                .wrap("{\"code\":500,\"msg\":\"internal error\"}".getBytes())
        ));
});
```

## 5. 响应式网关集成

Spring Cloud Gateway 也是基于 WebFlux，可与 Sentinel 深度整合。

```java
// 在 RouterFunction 中手动埋点
@Bean
public RouterFunction<ServerResponse> routes() {
    return route(GET("/api/products"), request -> {
        try (Entry entry = SphU.entry("GET:/api/products")) {
            return ok().body(productService.listProducts(), Product.class);
        } catch (BlockException e) {
            return status(HttpStatus.TOO_MANY_REQUESTS)
                .bodyValue("{\"error\":\"rate limit\"}");
        }
    });
}

// 注解方式（需 aop 支持）
@Component
public class ReactiveProductService {

    @SentinelResource(
        value = "getProduct",
        blockHandler = "getProductBlockHandler",
        fallback = "getProductFallback"
    )
    public Mono<Product> getProduct(Long id) {
        return reactiveRepo.findById(id);
    }

    public Mono<Product> getProductBlockHandler(Long id, BlockException ex) {
        return Mono.just(new Product(id, "limited"));
    }

    public Mono<Product> getProductFallback(Long id, Throwable ex) {
        return Mono.just(new Product(id, "fallback"));
    }
}
```

## 6. 响应式链路跟踪

```java
// 自定义 Origin 解析（区分调用来源）
WebFluxCallbackManager.setRequestOriginParser(request -> {
    String origin = request.getHeaders().getFirst("X-Origin");
    return origin != null ? origin : "unknown";
});
```

### Metrics 指标说明

| 指标 | 类型 | 说明 |
|------|------|------|
| `sentinel_webflux_pass_qps` | Counter | 通过的请求 QPS |
| `sentinel_webflux_block_qps` | Counter | 被限流的请求 QPS |
| `sentinel_webflux_rt_max` | Gauge | 最大响应时间 |
| `sentinel_webflux_total_count` | Counter | 总请求数 |

## 注意事项

1. **线程模型差异**：WebFlux 使用事件循环（少量线程），Sentinel 的 `Context` 需要用 `AsyncEntry` 而非 `Entry`
2. **异常传播**：`BlockException` 在响应式中需要通过 `Mono.error()` 传播，不能直接 throw
3. **`SphU.entry()` 在响应式中要配合 `AsyncEntry`**：
   ```java
   // 响应式场景正确用法
   AsyncEntry entry = SphU.asyncEntry("resourceName");
   return mono
       .doFinally(signalType -> {
           if (entry != null) entry.exit();
       });
   ```
4. **规则来源**：WebFlux 适配器同样支持 Nacos/Pomelo 等动态规则源
5. **性能影响**：每个请求都会有 Sentnel 拦截开销，生产环境建议只对关键热点路径配置流控规则
6. **Gateway 路由优先级**：Sentinel 过滤器应放在其他过滤器之前，避免其他过滤器处理后被限流白费资源
