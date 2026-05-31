---
name: nacos-30-features
description: Nacos 3.0 新特性详解（MCP Registry、安全零信任、架构演进）
tags: [nacos, spring-cloud, service-discovery, mcp]
---

## Nacos 3.0 新特性详解

Nacos 3.0 正式发布，这是自 Nacos 2.0 以来的重大版本升级。核心变化包括：**MCP Registry**、**安全零信任架构**、**Spring Boot 3 + JDK 17 升级**。

> ⚠️ Nacos 3.0 是破坏性升级，从 2.x 升级需要重点关注配置和 API 变化。

## 一、MCP Registry

MCP（Model Context Protocol）是 AI 时代服务发现的新协议标准。Nacos 3.0 率先支持 MCP Registry：

### 核心能力

1. **MCP Server 注册与发现**：支持将 MCP Server 注册到 Nacos，实现 MCP 协议的服务发现
2. **存量应用零改动升级**：已有 Spring Cloud / Dubbo 应用可通过 MCP Registry 对接 MCP 生态
3. **MCP 编排**：支持 MCP 服务的动态编排和管理
4. **双协议支持**：同时支持传统 HTTP/gRPC 注册和 MCP 注册

```yaml
# Nacos 3.0 server 配置
# conf/application.properties

# 开启 MCP Registry
nacos.mcp.registry.enabled=true

# MCP 监听端口
nacos.mcp.server.port=18848

# MCP 协议版本
nacos.mcp.protocol.version=1.0
```

### 客户端使用 MCP 注册

```java
// Spring Cloud 应用 - 通过 MCP 注册到 Nacos 3.0
@SpringBootApplication
@EnableDiscoveryClient
public class McpProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(McpProviderApplication.class, args);
    }
}
```

```yaml
# bootstrap.yml - Nacos 3.0 MCP 客户端配置
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        # MCP 注册开关
        mcp-enabled: true
        # 服务元数据，用于 MCP 协议
        metadata:
          mcp.protocol: "http"
          mcp.version: "1.0"
```

## 二、安全零信任架构

Nacos 3.0 引入了零信任安全架构：

### 主要安全增强

| 特性 | 说明 |
|------|------|
| 双向 TLS (mTLS) | 客户端与服务端双向证书认证 |
| 细粒度 RBAC | 命名空间级别/配置级别权限控制 |
| 请求签名 | API 请求必须携带签名 |
| 审计日志 | 所有操作记录审计日志 |
| IP 白名单 | 控制台/API 访问 IP 限制 |

```yaml
# 零信任安全配置
nacos.security.zero-trust.enabled=true

# mTLS 配置
nacos.security.ssl.require-client-auth=true
nacos.security.ssl.trust-store=classpath:truststore.jks
nacos.security.ssl.trust-store-password=changeit

# RBAC 细粒度权限
nacos.security.rbac.resource-scope=namespace,group,data_id
nacos.security.rbac.enable-data-id-pattern=true
```

### 客户端配置

```yaml
# 客户端 - mTLS 配置
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        tls:
          enabled: true
          key-store: classpath:client.jks
          key-store-password: changeit
          trust-store: classpath:truststore.jks
          trust-store-password: changeit
```

## 三、架构演进：Spring Boot 3 + JDK 17

Nacos 3.0 服务端已升级到 Spring Boot 3 + JDK 17：

```bash
# 环境要求
# JDK 17+
# 不再支持 JDK 8/11

# 启动 Nacos 3.0
sh bin/startup.sh -m standalone

# 检查版本
curl http://localhost:8848/nacos/v1/console/version
```

### 从 2.x 升级注意事项

> 🔴 **以下是从 2.x 升级到 3.0 的常见坑点：**

1. **JDK 必须升级**：不再支持 JDK 8 和 JDK 11，必须使用 JDK 17+
2. **配置文件迁移**：旧版 `application.properties` 中的部分配置项已废弃或重命名
3. **OpenAPI 变更**：部分 API 路径发生变化，客户端 SDK 需要升级
4. **内置数据库变更**：Derby 存储格式变化，集群升级需注意数据迁移
5. **插件兼容性**：自定义插件（鉴权、配置加密等）需要重新适配

```bash
# 升级数据迁移
# 1. 先备份 Nacos 2.x 数据
cp -r data/derby-data data/derby-data-backup

# 2. 启动 3.0 时会自动升级数据格式
# 建议先在测试环境验证升级流程
```

## 四、Nacos 3.0 架构全景

```
┌─────────────────────────────────────────────┐
│               Nacos 3.0                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   HTTP   │  │   gRPC   │  │   MCP    │   │
│  │ Registry │  │ Registry │  │ Registry │   │
│  └──────────┘  └──────────┘  └──────────┘   │
│         ┌────────────────────────┐           │
│         │   零信任安全层           │           │
│         │  mTLS + RBAC + 签名    │           │
│         └────────────────────────┘           │
│  ┌────────────────────────────────────┐      │
│  │         存储层 (CP + AP)           │      │
│  │  Distro + Raft + Derby/MySQL      │      │
│  └────────────────────────────────────┘      │
└─────────────────────────────────────────────┘
```

## 五、Spring Cloud Alibaba 集成

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>2025.0.0</version> <!-- 对应 Spring Cloud 2025 -->
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    <version>2025.0.0</version>
</dependency>
```

## 注意事项

1. **升级前务必在测试环境验证**：Nacos 3.0 是破坏性升级，配置和 API 有大量变更
2. **MCP Registry 是可选特性**：不需要 MCP 的团队可以不启用，不影响常规注册发现
3. **零信任安全按需开启**：开启 mTLS 后所有客户端需配置证书，会增加运维复杂度
4. **监控告警**：升级后注意监控 Nacos Server 的 JVM 内存和 GC，JDK 17 的 GC 行为与 JDK 8 不同
5. **回滚方案**：升级前确保有完整的数据备份和回滚流程
