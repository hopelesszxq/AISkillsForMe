---
name: nacos-32-features
description: Nacos 3.2.x 新特性：AI Registry、OIDC/OAuth2 SSO、PostgreSQL 兼容性增强
tags: [spring-cloud, nacos, ai-registry, oidc, sso, postgresql]
---

## Nacos 3.2.x 新特性概览

Nacos 3.2.1（2026-04-23 发布）是 3.2.0 之后的重要补丁版本，引入了大量新功能和安全增强。

### 1. AI Registry（AI 注册中心）

Nacos 3.2.x 引入了完整的 AI Registry 功能，支持 MCP（Model Context Protocol）的 Agent/Skill 注册与发现。

**核心能力：**
- **Prompt 生命周期管理**：支持 Prompt 的创建、发布、下线全流程管理，提供 UI 界面
- **A2A AgentCard v1 协议**：支持基于 Agent-to-Agent 协议的标准化接口
- **Skill bizTag 过滤**：支持按业务标签过滤 Skill 列表
- **MCP Server 资源规范存储**：支持 Resource Spec 存储，用于 MCP Server 资源描述
- **Admin 强制发布**：管理员可强制发布 Skill，覆盖普通用户的发布权限

```java
// 使用 Nacos AI Registry API 注册 Agent
@Bean
public AgentRegistryClient agentRegistryClient(NacosNamingService namingService) {
    return new NacosAgentRegistryClient(namingService);
}

// 注册一个 AI Agent
AgentCard agentCard = AgentCard.builder()
    .name("customer-service-agent")
    .description("智能客服 Agent")
    .skills(List.of(
        SkillSpec.builder()
            .name("order-query")
            .bizTag("ecommerce")
            .protocol("A2A")
            .build()
    ))
    .build();

agentRegistryClient.register(agentCard);
```

### 2. OIDC/OAuth2 SSO 登录

新增 OIDC（OpenID Connect）和 OAuth2 单点登录支持，两个控制台（Legacy + Next）均可使用。

**配置示例：**
```yaml
# application.properties — Nacos Server
nacos.sso.oidc.enabled=true
nacos.sso.oidc.issuer-uri=https://your-idp.example.com
nacos.sso.oidc.client-id=nacos-client
nacos.sso.oidc.client-secret=your-client-secret
nacos.sso.oidc.scope=openid,profile,email
nacos.sso.oidc.claim-username=preferred_username
```

支持的 IdP：
- Keycloak
- Okta
- Azure AD / Entra ID
- Auth0
- 任意标准 OIDC Provider

### 3. 安全增强

- **LDAP 认证绕过漏洞修复**（CVE 相关）
- 配置表单输入参数校验增强
- Token 过期处理优化
- 依赖升级：Spring Boot 3.5.13、MCP SDK 0.17.0、log4j-core 2.25.4

### 4. PostgreSQL/Oracle/MySQL 兼容性增强

**PostgreSQL 修复：**
- 配置分页查询支持确定性排序（ORDER BY 子句），避免分页结果不一致
- Schema 时间戳问题修复

**Oracle 修复：**
- Schema 兼容性问题修复

**MySQL 修复：**
- `findConfigInfoLike4PageFetchRows` 查询结果精度修复

### 5. 并发与可靠性

- 修复 AI 发布管道的竞态条件
- 修复 Naming 模块 `ConcurrentHashMap` 的 check-then-act 竞态条件
- 修复客户端 Failover 中的竞态条件
- 修复配置导出的线程安全问题
- Derby 快照操作 JDBC 资源泄漏修复

## 升级注意事项

```yaml
# 从 3.1.x 升级到 3.2.x
# 1. 数据库兼容性：3.2.x 继续支持 MySQL/PostgreSQL/Oracle/Derby
# 2. 如果使用 LDAP 认证，需关注安全补丁
# 3. AI Registry 功能为可选，不启用不影响原有服务注册发现
# 4. OIDC 配置后，原有用户名密码登录仍可保留作为 fallback
```

## 参考链接

- [Nacos 3.2.1 Release Notes](https://github.com/alibaba/nacos/releases/tag/3.2.1)
- [Spring Cloud Alibaba 2025.1.x](https://github.com/alibaba/spring-cloud-alibaba)
