---
name: spring-boot-41-features
description: Spring Boot 4.1 新特性：Redis 注解监听器、gRPC 原生支持、KafkaTemplate 增强、Webflux HTML 转义
tags: [tools, spring-boot, spring-boot-41, redis, kafka, grpc, webflux]
---

## 概述

Spring Boot 4.1.0 GA（2026-06-10 发布）在 4.0 稳定版基础上引入了多项新特性，包括 **Redis 注解驱动监听器**、**KafkaTemplate 配置增强**、**Webflux HTML 转义** 与 **gRPC 原生支持**。

> gRPC 原生集成的详细内容请参见 `spring-boot-41-grpc-support.md`，本文聚焦其他新增特性。

### 安全更新 ⚠️

4.1.0 GA 基于 Spring Framework 7.0.8，**修复了 16 个 CVE 漏洞**，包括 WebSocket Session 可预测、SpEL 任意方法调用、Jackson 反序列化 RCE 等高危问题。详见 [`spring-framework-708-cve.md`](spring-framework-708-cve.md)。

## 一、Redis 注解驱动监听器（`@RedisListener`）

Spring Boot 4.1 引入了 `@RedisListener` 注解，使 Redis 消息订阅不再需要手动管理 `MessageListenerAdapter`。

### 添加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### 使用方式

```java
@Component
public class RedisMessageHandler {

    @RedisListener(channel = "order:notifications")
    public void handleOrderNotification(String message) {
        // 自动反序列化（默认 JSON）
        OrderNotification notification = JsonUtil.parse(message, OrderNotification.class);
        log.info("收到订单通知: {}", notification.getOrderId());
    }

    @RedisListener(channel = "system:alerts", 
                   serializer = "customSerializer")
    public void handleAlert(AlertMessage alert) {
        log.warn("系统告警: {}", alert.getLevel());
    }
}
```

### 配置选项

```yaml
spring:
  data:
    redis:
      listener:
        container:
          concurrency: 5           # 并发消费者数
          max-attempts: 3          # 消费最大重试次数
          backoff-delay: 1000      # 重试间隔（毫秒）
          error-handler: myErrorHandler  # 自定义错误处理器
```

## 二、KafkaTemplate 配置增强

### allowNonTransactional

可配置 `KafkaTemplate` 在事务管理器存在时是否允许非事务操作。

```yaml
spring:
  kafka:
    template:
      allow-non-transactional: true  # 允许非事务发送
      close-timeout: 10s             # 关闭超时时间
```

```java
// 或通过 @Bean 自定义
@Bean
public KafkaTemplate<String, Object> kafkaTemplate(
        ProducerFactory<String, Object> producerFactory) {
    KafkaTemplate<String, Object> template = new KafkaTemplate<>(producerFactory);
    template.setAllowNonTransactional(true);
    template.setCloseTimeout(Duration.ofSeconds(10));
    return template;
}
```

## 三、ReactorHttpClientBuilder 默认值对齐

Spring Boot 4.1 将 `ReactorHttpClientBuilder` 的默认值与 Spring Framework 对齐，提供关闭选项：

```yaml
spring:
  webflux:
    reactor:
      httpclient:
        # 与 Spring Framework 默认值对齐
        # 如需自定义连接池，取消注释以下配置
        # max-connections: 500
        # max-idle-time: 30s
        # align-defaults: false  # 设为 false 可恢复旧行为
```

## 四、Webflux HTML 转义属性

新增 `spring.webflux.default-html-escape` 属性，实现全应用的 HTML 转义配置：

```yaml
spring:
  webflux:
    default-html-escape: true  # 默认对响应中的 HTML 特殊字符进行转义
```

```java
// 针对单个接口覆盖
@GetMapping(value = "/html", produces = MediaType.TEXT_HTML_VALUE)
public String renderHtml() {
    return "<h1>Hello</h1><script>alert('xss')</script>";
    // 4.1 默认会转义 <script> 标签
}
```

## 五、自定义 SessionTimeout Bean

支持在非 Web 场景或需要自定义超时策略时注入独立的 `SessionTimeout`：

```java
@Bean
public SessionTimeout customSessionTimeout() {
    return () -> Duration.ofMinutes(30);
}
```

## 六、Elasticsearch Rest5Client 支持

新增对 `docker.elastic.co/elasticsearch/elasticsearch` 镜像的支持，同时修复了 `Rest5Client` 自动配置中 HTTP 客户端配置错误的问题。

```yaml
spring:
  elasticsearch:
    rest5:
      uris: http://localhost:9200
```

## 七、Maven 依赖坐标

```xml
<!-- Spring Boot 4.1.x BOM -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.1.0-RC1</version>
    <relativePath/>
</parent>
```

## 八、迁移注意事项

| 旧行为 | 4.1 行为 | 影响 |
|--------|---------|------|
| KafkaTemplate 默认允许非事务 | 需显式配置 `allow-non-transactional` | 事务环境下可能行为变化 |
| ReactorHttpClientBuilder 默认值 | 与 Spring Framework 对齐 | 连接池参数变化 |
| Webflux 不自动转义 HTML | 默认转义（`default-html-escape: true`） | 已有 HTML 响应的接口需检查 |
| 无原生 Redis 监听器注解 | `@RedisListener` 可用 | 可选迁移简化代码 |
| gRPC 需第三方 starter | `spring-boot-starter-grpc-*` 内建 | 可移除第三方依赖 |
