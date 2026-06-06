---
name: feign-oauth2-integration
description: Spring Cloud OpenFeign 集成 OAuth2 client_credentials 认证的自动配置方法
tags: [spring-cloud, feign, oauth2, security, spring-security]
---

## 概述

Spring Cloud OpenFeign 可以无缝集成 Spring Security 的 OAuth2 自动配置，通过 `spring.cloud.openfeign.oauth2.enabled=true` 开启，无需手动编写 `RequestInterceptor`。Feign 客户端会自动获取 Token 并在过期时刷新。

依赖：`spring-cloud-starter-openfeign` + `spring-boot-starter-oauth2-client`

## 配置步骤

### 1. Spring Security OAuth2 客户端配置

```properties
# application.properties
spring.security.oauth2.client.registration.myapp.client-id=${client_id}
spring.security.oauth2.client.registration.myapp.client-secret=${client_secret}
spring.security.oauth2.client.registration.myapp.authorization-grant-type=client_credentials
spring.security.oauth2.client.registration.myapp.scope=${scope_list}
spring.security.oauth2.client.provider.myapp.token-uri=${token_uri}
```

`myapp` 是 `clientRegistrationId`，可自定义。支持配置多个不同 OAuth2 认证（不同 `clientRegistrationId`）。

### 2. 启用 OpenFeign OAuth2 集成

```properties
spring.cloud.openfeign.oauth2.enabled=true
spring.cloud.openfeign.oauth2.clientRegistrationId=myapp
```

`clientRegistrationId` 必须与 Spring Security 配置中的注册 ID 一致。

### 3. 多客户端场景

如果需要对不同 FeignClient 使用不同的 OAuth2 认证：

```properties
spring.cloud.openfeign.oauth2.enabled=true
spring.cloud.openfeign.oauth2.clientRegistrationId=myapp
# 默认使用 myapp，特定 Client 可通过 @FeignClient configuration 覆盖
```

## 高级配置

### 自定义 Feign 配置类指定 OAuth2

```java
@FeignClient(name = "payment-service", configuration = PaymentOAuth2Config.class)
public interface PaymentClient {
    @GetMapping("/api/payments")
    List<Payment> getPayments();
}
```

```java
public class PaymentOAuth2Config {
    @Bean
    public RequestInterceptor oauth2FeignRequestInterceptor(
            OAuth2AuthorizedClientManager manager,
            OAuth2ClientProperties properties) {
        return new OAuth2FeignRequestInterceptor(manager, properties, "payment-service");
    }
}
```

### 手动控制 Token 获取

如果自动配置不满足需求，可以实现自己的 `RequestInterceptor`：

```java
@Component
public class CustomOAuth2Interceptor implements RequestInterceptor {
    private final OAuth2AuthorizedClientManager authorizedClientManager;
    private final String registrationId = "myapp";

    @Override
    public void apply(RequestTemplate template) {
        OAuth2AuthorizeRequest authorizeRequest = OAuth2AuthorizeRequest
            .withClientRegistrationId(registrationId)
            .principal("feign-client")
            .build();
        OAuth2AuthorizedClient client = authorizedClientManager.authorize(authorizeRequest);
        if (client != null) {
            template.header("Authorization", "Bearer " + client.getAccessToken().getTokenValue());
        }
    }
}
```

## 注意事项

- `spring.cloud.openfeign.oauth2.enabled=true` 仅对 Spring Cloud OpenFeign 有效，不兼容 OpenFeign 原生（`io.github.openfeign`）
- 确保 Feign 请求的服务端配置了正确的 JWT 验证公钥
- Token 自动刷新由 OAuth2AuthorizedClientManager 内置处理，过期前自动获取新 Token
- 生产环境应使用 Vault / Kubernetes Secret 等管理 client-secret，避免硬编码
- 不同微服务间的 Feign 调用链中，建议在网关层统一 Token 透传，避免每个服务各自获取 Token

## 常见坑点

### clientRegistrationId 属性名格式（Feign 4.3.2+ 修复）

Feign 4.3.2 之前，`spring.cloud.openfeign.oauth2.clientRegistrationId` 的值**只接受 CamelCase**，如果使用 dash-case（短横线）则配置不生效：

```properties
# ❌ Feign 4.3.2 前不生效
spring.cloud.openfeign.oauth2.clientRegistrationId=my-oauth-client

# ✅ 必须使用 CamelCase
spring.cloud.openfeign.oauth2.clientRegistrationId=myOauthClient
```

**解决方案**：
- 升级到 Feign 4.3.2+（两种格式均可）
- 或统一使用 CamelCase 作为 OAuth2 clientRegistrationId

> 参考 Spring Cloud OpenFeign issue [#1270]
