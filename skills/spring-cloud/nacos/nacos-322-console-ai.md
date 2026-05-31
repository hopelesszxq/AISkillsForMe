---
name: nacos-322-console-ai
description: Nacos 3.2.2 新特性：控制台增强、AI Registry 外部导入、Skill 订阅发现、安全加固
tags: [spring-cloud, nacos, ai-registry, console, skill, security]
---

## 概述

Nacos 3.2.2（2026-05-29 发布）是 3.2.x 系列的 Bugfix 和体验改进版本。主要聚焦于控制台体验优化、AI Registry 外部注册表导入、Skill 全生命周期管理增强和安全加固。

## 主要新特性

### 1. 控制台体验增强

- **配置 Diff 确认**：发布配置前显示变更对比确认对话框
- **Naming 实例管理**：Console v3 支持持久化实例注销
- **配置页命名空间切换**：修复切换后页面显示不一致问题
- **主题颜色修复**：支持 RGB 回退值，避免主题色丢失
- **历史/回滚/对比页面**：修复访问被拒绝的错误

### 2. AI Registry 外部注册表导入

Nacos 3.2.2 支持从运维配置的外部注册表导入 AI 资源：

```yaml
# application.properties
nacos.ai.registry.external-import.enabled=true
nacos.ai.registry.external-import.sources=mcp-hub,skill-market
```

支持的外部来源包括：
- MCP 资源
- Skill 知名资源（Well-known）
- Importer API 接口
- SPI 模型与内置预设
- 基于来源的导入 UI

### 3. Skill 订阅与发现

- **Skill 知名协议发现**: 新增最新 Skill Well-known 协议适配器
- **Skill 订阅支持**：AgentSpec 订阅迁移为 HTTP 轮询 + 304 处理
- **Skill 自动发布**：审核后支持可选自动发布
- **拖拽 ZIP 上传**：支持批量 Skill 上传
- **版本提交信息**：上传 AI Skill 时支持提交消息

### 4. 安全加固

```java
// Zip Slip / Zip Bomb 防御 — 自动生效
nacos.ai.skill.zip.max-entry-count=10000
nacos.ai.skill.zip.max-entry-size=100MB
```

- AgentSpec ZIP 解析加入 Zip Slip、Zip Bomb 和条目计数防御
- 依赖升级：Micrometer 1.15.10（修复 Prometheus 端点）
- PostgreSQL 驱动升级至 42.7.11
- OIDC 插件 JAR 内嵌 Caffeine 避免 NoClassDefFoundError
- LDAP 认证拆分为可选插件（可单独打包 LDAP 插件 JAR）
- Console UI npm 最小发布年限策略（供应链安全）

### 5. 配置灰度规则标签匹配

```yaml
# 配置灰度发布时支持基于标签的匹配
spring.cloud.nacos.config.gray-rule.match-type=label
spring.cloud.nacos.config.gray-rule.labels=version=v2,region=cn-east
```

## PostgreSQL 相关修复

- **租户 null 引起配置同步行爆炸**：修复 PostgreSQL 中 Tenant 为 null 时导致配置同步生成大量异常行的问题
- BaseConfigInfoMapper 补充 group_id LIKE 占位符

## 注意事项

1. **Java 版本要求不变**：Server/Console 仍需 Java 17，Client 仍兼容 Java 8
2. **nacos.functionMode 扩展**：新增 `microservice` 和 `ai` 两种模式
3. **ConfigChangeAspect 修复**：修复了当 srcType 缺失时将 Console 配置变更误判为 RPC 的问题
4. **升级建议**：PostgreSQL 用户强烈建议升级，修复了租户 null 导致的严重问题
