---
name: sentinel-1810-features
description: Sentinel 1.8.10 新特性：HttpClient 5.x 适配、Spring RestClient 支持、高级配置
tags: [spring-cloud, sentinel, httpclient, restclient, circuit-breaker]
---

## 概述

Sentinel 1.8.10（2026-05-21 发布）在现有功能基础上新增了多个重要集成能力，进一步扩展了 Sentinel 在 Spring Boot 3.x 和 Java 21 生态中的适用场景。

## 主要新特性

### 1. Apache HttpClient 5.x 适配器

支持对 Apache HttpClient 5 的调用进行 Sentinel 流控/熔断保护，覆盖 `CloseableHttpClient` 和 `HttpAsyncClient`。

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-apache-httpclient5-adapter</artifactId>
    <version>1.8.10</version>
</dependency>
```

```java
// 构建被保护的 HttpClient
CloseableHttpClient client = HttpClients.custom()
    .addRequestInterceptorLast(
        new SentinelApacheHttpClient5Interceptor())
    .build();

// 自动按 URL 资源名进行流控统计
HttpGet request = new HttpGet("http://user-service/api/users");
try (CloseableHttpResponse response = client.execute(request)) {
    // 业务处理
}
```

**资源命名规则**：默认使用请求的 HTTP method + URI 路径作为资源名（如 `GET:/api/users`），可通过自定义 `SentinelApacheHttpClient5Builder` 修改。

### 2. Spring RestClient 支持

Spring Boot 3.2+ 引入了新的 `RestClient` 替代 `RestTemplate`，Sentinel 1.8.10 新增对应适配。

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-spring-webclient-adapter</artifactId>
    <version>1.8.10</version>
</dependency>
```

```java
@Configuration
public class SentinelRestClientConfig {

    @Bean
    public RestClient.Builder restClientBuilder() {
        return RestClient.builder()
            .requestInterceptor(new SentinelRestClientInterceptor());
    }
}

// 使用时自动拦截
@Autowired
private RestClient restClient;

public User getUser(Long id) {
    return restClient.get()
        .uri("http://user-service/users/{id}", id)
        .retrieve()
        .body(User.class);
    // Sentinel 自动以 "GET:http://user-service/users/{id}" 为资源名
}
```

### 3. 正则匹配资源优化开关

新增可选的开关，当已有精确匹配时跳过正则资源匹配，减少匹配开销。

```properties
# 开启后，若某资源已有精确匹配（如 GET:/api/users），
# 则不再进行正则表达式匹配（如 GET:/api/{path}）
# 默认关闭（false），开启后可能影响兼容性
csp.sentinel.skip.regex.when.exact.match=true
```

```java
// 适合高 QPS 场景，减少正则匹配 CPU 消耗
FlowRule rule = new FlowRule();
rule.setResource("GET:/api/users");    // 精确匹配
rule.setCount(1000);
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
```

### 4. 断路器错误比率阈值修复

修复了当 `errRatioThreshold` 设置为 100% 时请求无法正确阻断的问题（#1857）。

```properties
# 旧版本中设置 100% 会导致熔断永不触发
# 1.8.10 已修复
errRatioThreshold=1.0  # 即 100%
```

## 配置示例

### 完整 Spring Boot 3 集成

```yaml
spring:
  cloud:
    sentinel:
      enabled: true
      transport:
        dashboard: localhost:8080
      datasource:
        # Nacos 动态规则源
        flow:
          nacos:
            server-addr: ${nacos.addr}
            data-id: sentinel-flow-rules
            group-id: SENTINEL
            rule-type: flow
```

```java
// 使用新适配器保护外部 HTTP 调用
@Service
public class ExternalApiService {

    private final CloseableHttpClient httpClient;

    public ExternalApiService() {
        this.httpClient = HttpClients.custom()
            .addRequestInterceptorLast(
                new SentinelApacheHttpClient5Interceptor(
                    // 自定义资源名策略
                    request -> "EXT_API:" + request.getMethod() 
                               + ":" + request.getUri().getPath()
                ))
            .build();
    }

    @SentinelResource(value = "external:payment", 
                      blockHandler = "fallback")
    public String callPaymentApi(String orderId) {
        HttpGet request = new HttpGet(
            "http://payment-service/api/pay/" + orderId);
        try (CloseableHttpResponse resp = httpClient.execute(request)) {
            return EntityUtils.toString(resp.getEntity());
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    public String fallback(String orderId, BlockException ex) {
        return "{\"status\":\"degraded\",\"orderId\":\"" + orderId + "\"}";
    }
}
```

## 注意事项

| 要点 | 说明 |
|------|------|
| **HttpClient 5 vs 4** | 旧版 `sentinel-apache-httpclient-adapter` 仅支持 HttpClient 4.x，1.8.10 新增的 `sentinel-apache-httpclient5-adapter` 是独立模块 |
| **RestClient vs RestTemplate** | RestClient 是 Spring Boot 3.2+ 的非阻塞方式，Sentinel RestClient 适配器走的是 `ClientHttpRequestInterceptor` 同步拦截 |
| **正则匹配开关** | 精确匹配优先策略仅适用于资源名完全一致的场景。如果规则用正则表达式定义且同一资源路径有多个不同参数，不要开启此开关 |
| **Sentinel Dashboard** | 1.8.10 Dashboard 修复了断路器规则统计窗口时间单位 tooltip 显示问题，建议同步升级 |
