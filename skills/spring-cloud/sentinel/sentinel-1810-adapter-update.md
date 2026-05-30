---
name: sentinel-1810-adapter-update
description: Sentinel 1.8.10 新适配器与特性：HttpClient 5.x、Spring RestClient、正则匹配优化、熔断降级实践
tags: [spring-cloud, sentinel, circuit-breaker, adapter, rate-limiting]
---

## 概述

Sentinel 1.8.10（2026-05-21 发布）新增了 Apache HttpClient 5.x 和 Spring RestClient 适配器，并优化了资源正则匹配行为。本文介绍新特性及熔断降级的最佳实践。

## 一、新增适配器

### 1. Apache HttpClient 5.x 适配器

适用于使用了 Apache HttpClient 5.x（`org.apache.hc.client5.http`）作为 HTTP 客户端的项目。

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-apache-httpclient5-adapter</artifactId>
    <version>1.8.10</version>
</dependency>
```

```java
// 自动拦截 HttpClient 5.x 发起的 HTTP 调用
// 无需额外配置即可对出站 HTTP 请求进行流量控制和熔断

// 配置 Sentinel 保护的 HttpClient
HttpClient client = HttpClientBuilder.create()
    .addExecInterceptorFirst("sentinel", new SentinelExecChainHandler())
    .build();
```

### 2. Spring RestClient 适配器

适用于 Spring 6.1+ 中新增的 `RestClient`（同步 HTTP 客户端）。

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-spring-restclient-adapter</artifactId>
    <version>1.8.10</version>
</dependency>
```

```java
// 定义 RestClient 时注入 Sentinel 拦截
@Bean
public RestClient.Builder restClientBuilder() {
    return RestClient.builder()
        .requestInterceptor(new SentinelRestClientInterceptor());
}

// 使用
@Autowired
private RestClient.Builder restClientBuilder;

public void callRemote() {
    restClientBuilder.build()
        .get()
        .uri("https://api.example.com/users")
        .retrieve()
        .body(String.class);
}
```

## 二、正则资源匹配优化

新增加可选开关：当存在精确匹配时跳过正则匹配，提升高并发下的匹配性能。

```java
// 启用精确匹配优先（跳过正则匹配）
SentinelConfig.setConfig("csp.sentinel.skip-regex-when-exact-match-exists", "true");
```

**适用场景**：大量使用 `SphU.entry(resourceName)` 且部分资源名使用正则表达式 `SphU.entry("user:*")` 的场景。开启后，精确匹配的资源名直接命中，不会回溯到正则表达式。

## 三、熔断降级最佳实践（新版本增强）

### 错误比率阈值 100% 时正确阻断

1.8.10 修复了 `errRatioThreshold` 达到 100% 时请求未被正确阻断的 Bug。

```java
// 熔断规则 - 错误率 100% 熔断
DegradeRule rule = new DegradeRule("userService")
    .setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_RATIO)
    .setCount(1.0)        // 100% 错误率
    .setTimeWindow(10);   // 熔断时长 10 秒
DegradeRuleManager.loadRules(Collections.singletonList(rule));
```

### 熔断降级策略选择

| 策略 | 适用场景 | 推荐阈值 | 说明 |
|------|---------|---------|------|
| 慢调用比例 (RT) | 外部 API 调用、数据库查询 | count>1000ms, maxRt>5000ms | 响应时间超过阈值的调用比例 |
| 异常比例 | 业务逻辑校验、数据一致性 | count=0.5~0.8 | 异常数占总调用量的比例 |
| 异常数 | 非核心链路 | count=100 | 分钟级异常总数 |

## 四、WebMVC v6.x 适配器修复

当 `http-method-specify` 启用时，HTTP 方法前缀拼接异常已修复。

```yaml
# application.yml
sentinel:
  filter:
    enabled: true
    http-method-specify: true  # 资源名包含 HTTP 方法前缀：GET:/api/users
```

## 五、Dashboard 优化

1.8.10 的 Dashboard 修复了新增熔断规则时统计窗口时长的 tooltip 单位显示错误。

## 注意事项

- HttpClient 5.x 适配器与 4.x 版本的适配器 **artifactId 不同**，请确认依赖版本
- RestClient 适配器需要 Spring 6.1+ / Spring Boot 3.2+
- 正则匹配优化开关默认关闭（`false`），以保持向后兼容
- 升级后务必验证熔断规则的 errRatioThreshold=100% 场景是否按预期工作
- Dashboard 的 1.8.10 版本与客户端版本需保持一致，避免 protobuf 序列化问题
