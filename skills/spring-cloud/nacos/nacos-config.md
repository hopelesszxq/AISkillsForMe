---
name: nacos-config
description: Nacos 配置中心最佳实践
tags: [spring-cloud, nacos, config]
---

## 配置管理
- 命名空间（Namespace）用于多环境隔离：dev/test/prod
- 分组（Group）用于同环境不同业务线隔离
- 配置格式推荐 YAML，支持动态刷新 @RefreshScope

## 最佳实践
- 公共配置抽取到共享配置文件，用 `shared-configs` 引入
- 敏感信息用 `nacos:encrypted-data-key` 加密
- 配置变更走审批流程，避免直接改生产
