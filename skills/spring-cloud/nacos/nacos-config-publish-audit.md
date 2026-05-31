---
name: nacos-config-publish-audit
description: Nacos 3.2.2 配置发布确认与审计：Diff 确认、发布审批、变更追踪最佳实践
tags: [spring-cloud, nacos, config, publish, audit, safety]
---

## Nacos 3.2.2 配置发布安全增强

Nacos 3.2.2（2026-05-29）在配置管理和操作安全方面引入了多项改进，其中 **Config Diff 发布确认** 和 **nacos.functionMode** 分离是最实用的变化。

## 一、Config 发布 Diff 确认（3.2.2 新特性）

### 功能描述

在 Nacos 新 Console（控制台）中，**发布或修改配置时** 会弹出 Diff 对比窗口，清晰显示当前配置与即将发布配置的差异，避免误操作。

### 效果

```
┌─────────────── Config Publish ───────────────┐
│                                               │
│  dataId: application.yaml                     │
│  group: DEFAULT_GROUP                         │
│                                               │
│  ┌──── Current ────┬──── New ───────────────┐ │
│  │ db.url: old-db  │ db.url: new-db         │ │
│  │ feature: false  │ feature: true          │ │
│  │                 │ cache.ttl: 300         │ │
│  └─────────────────┴────────────────────────┘ │
│                                               │
│  🔴 -1 line  🟢 +2 lines                     │
│                                               │
│  [Cancel]                    [Confirm Publish] │
└───────────────────────────────────────────────┘
```

### API 层面调用（跳过确认）

```bash
# 直接发布（跳过 Console 确认，适合 CI/CD）
curl -X POST "http://nacos:8848/nacos/v1/cs/configs" \
  -d "dataId=application.yaml&group=DEFAULT_GROUP&content=$(cat app.yaml | base64)"

# 带备注发布（审计日志中使用）
curl -X POST "http://nacos:8848/nacos/v1/cs/configs" \
  -d "dataId=application.yaml&group=DEFAULT_GROUP&content=...&desc=修改数据库连接串 v2.3"
```

## 二、nacos.functionMode 模式分离（3.2.2）

### 背景

Nacos 3.2.x 引入了 AI Registry 功能，但多数微服务团队并不需要 AI 能力。3.2.2 新增 `nacos.functionMode` 配置，允许按需启用功能集。

```yaml
# application.properties — Nacos Server
# 可选值: microservice | ai | all（默认）
nacos.functionMode=microservice
```

| 模式 | 启用功能 | 禁用功能 | 推荐场景 |
|------|---------|---------|----------|
| `microservice` | 配置管理 + 服务注册发现 | AI Registry、MCP、Skill | **纯微服务场景（推荐）** |
| `ai` | AI Registry、MCP、Skill | 传统配置和服务管理 | AI 注册中心 |
| `all`（默认） | 全部功能 | 无 | 调试 / 全功能 |

### 迁移建议

```bash
# 1. 检查当前 Nacos 配置
grep -r "nacos.functionMode" /opt/nacos/conf/

# 2. 如果是纯 Spring Cloud 微服务，建议设置为 microservice
echo "nacos.functionMode=microservice" >> /opt/nacos/conf/application.properties

# 3. 重启 Nacos 生效
docker compose restart nacos
```

## 三、配置变更审计最佳实践

### 1. 结合描述信息追踪变更

```bash
# 发布时携带描述，便于后续审计
curl -X POST "http://nacos:8848/nacos/v1/cs/configs" \
  -d "dataId=application.yaml&group=DEFAULT_GROUP&content=...&desc=release-v2.3-db-update"

# 通过 OpenAPI 获取变更记录
curl "http://nacos:8848/nacos/v1/cs/history?dataId=application.yaml&group=DEFAULT_GROUP" \
  | jq '.data[] | {id, dataId, lastModifiedTime, srcDesc}'
```

### 2. 灰度发布 + Diff 确认双保险

推荐发布流程：

```
编写配置 → [可选]标签灰度发布验证 → 控制台 Diff 确认 → 正式发布 → 删除灰度规则
```

```yaml
# 流程编号对应工具
# 1. 灰度发布: Nacos Console → 配置管理 → 灰度
# 2. Diff 确认: 3.2.2 Console 自动弹出
# 3. 正式发布: "发布为正式配置" 按钮
# 4. 清理: 灰度结束后删除灰度配置
```

### 3. Nacos 配置变更 Webhook（自定义插件）

如果需要将配置变更通知到外部审计系统（如 Grafana、ELK），可以通过 Nacos Event 插件实现：

```java
// 自定义 Nacos Config 变更监听
@Component
public class NacosAuditListener {

    @EventListener
    public void onConfigPublish(ConfigPublishEvent event) {
        AuditLog log = AuditLog.builder()
            .dataId(event.getDataId())
            .group(event.getGroup())
            .operator(event.getOperator())  // 操作人（需要开启认证）
            .operation("PUBLISH")
            .timestamp(Instant.now())
            .md5Before(event.getMd5Before())
            .md5After(event.getMd5After())
            .build();
        // 发送到审计系统
        auditClient.send(log);
    }
}
```

### 4. 配置变更回滚流程

```bash
# 查看历史版本
curl "http://nacos:8848/nacos/v1/cs/history?dataId=app.yaml&group=DEFAULT_GROUP&pageNo=1&pageSize=10"

# 回滚到指定版本（保留最近 30 天历史）
curl -X POST "http://nacos:8848/nacos/v1/cs/configs/rollback" \
  -d "dataId=app.yaml&group=DEFAULT_GROUP&nid=12345"
```

## 四、升级建议

```yaml
# Nacos Server 3.2.2 application.properties 推荐配置
nacos.functionMode=microservice          # 纯微服务场景
nacos.config.gray.label-match-enabled=true  # 启用标签灰度
nacos.core.auth.enabled=true              # 开启认证
nacos.core.auth.server.identity.key=server-identity
nacos.core.auth.server.identity.value=change-me
```

## 参考

- [Nacos 3.2.2 Release Notes](https://github.com/alibaba/nacos/releases/tag/3.2.2)
- [Nacos OpenAPI 文档](https://nacos.io/docs/latest/guide/user/open-api/)
