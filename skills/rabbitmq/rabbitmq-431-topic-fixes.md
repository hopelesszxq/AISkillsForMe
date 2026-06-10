---
name: rabbitmq-431-topic-fixes
description: RabbitMQ 4.3.1 重要 Bug 修复：Topic Exchange 空绑定键路由 Bug、Passive 声明权限放宽
tags: [rabbitmq, topic-exchange, bugfix, permission, binding-key]
---

## 概述

RabbitMQ `4.3.1`（2026-06-XX 发布）是 4.3.x 系列的首个维护版本，修复了生产环境中可能造成**消息错误路由**和**权限拒绝异常**的关键问题。

## 重大 Bug 修复

### 1. Topic Exchange 空绑定键路由错误（🔥 关键）

**问题**：当队列使用**空字符串** (`""`) 作为 Binding Key 绑定到 Topic Exchange 时，发布到**任何** Topic Exchange 且 Routing Key 为空的（`""`）消息都会被错误路由到该队列。

**影响**：
- 如果某个队列不小心使用了空 Binding Key 绑定 Topic Exchange，它会收到**所有其他 Topic Exchange** 的空 Routing Key 消息
- 跨 Exchange、跨 VHost 的消息泄漏风险
- 难以排查，因为 Topic Exchange 通常预期空 Routing Key 不会被路由

**复现场景**：

```bash
# 创建一个 Topic Exchange
rabbitmqadmin declare exchange name=topic-a type=topic

# 创建一个队列，用空键绑定 — 这是问题所在
rabbitmqadmin declare queue name=leaked-queue
rabbitmqadmin declare binding source=topic-a destination=leaked-queue routing_key=""

# 向另一个 Topic Exchange 发空 Routing Key 消息
rabbitmqadmin publish exchange=topic-b routing_key="" payload="secret-data"
# ❌ leaked-queue 会意外收到此消息
```

**修复方式**：启用 `topic_binding_projection_v5` 功能标记。

```bash
# 所有集群节点升级到 4.3.1 后，启用修复
rabbitmqctl enable_feature_flag topic_binding_projection_v5
```

**升级步骤**：

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | 升级所有节点到 4.3.1 | 滚动升级，确保所有节点运行新版本 |
| 2 | 确认集群状态 | `rabbitmqctl cluster_status` 确保所有节点在线 |
| 3 | 启用 Feature Flag | `rabbitmqctl enable_feature_flag topic_binding_projection_v5` |
| 4 | 验证修复 | 测试空 Routing Key 消息路由行为 |

> ⚠️ 启用 Feature Flag 是不可逆操作，请先在测试环境验证。

### 2. Passive 声明权限放宽

**问题**：`4.3.0` 中引入 Khepri 元数据存储后，Passive Queue / Exchange 声明（`declare --passive`）仅允许拥有 `configure` 权限的用户执行。这导致很多监控工具（如 Prometheus RabbitMQ Exporter、Kubernetes liveness probe）出现权限拒绝错误。

**修复**：`4.3.1` 放宽了检查逻辑，用户只需要在 VHost 上拥有**任意权限**（`configure` / `write` / `read` 之一）即可执行 Passive 声明。

```bash
# 旧行为（4.3.0）— 权限拒绝
# rabbitmqadmin declare queue name=test-queue passive=true
# ❌ 如果用户只有 read 权限，会报错

# 新行为（4.3.1）— 正常工作
rabbitmqadmin declare queue name=test-queue passive=true
# ✅ 有 read / write / configure 任意权限即可
```

### 3. Classic Queue 共享消息存储 GC 滞后

**问题**：在高负载下，Classic Queue 的共享消息存储（Shared Message Store）GC 可能落后于队列活动，导致内存占用持续增长。

**影响**：使用 Classic Queue 的节点在**高吞吐场景**下可能出现内存泄漏倾向。

**缓解/修复**：
- 对于新项目，推荐使用 **Quorum Queue**（性能接近，运维更友好）
- 已在 4.3.1 中优化 GC 触发频率
- 监控指标：`rabbitmq_queue_messages_ready` + `node_mem_used` 对比

## 升级建议

```bash
# 1. 下载 4.3.1
# Docker
docker pull rabbitmq:4.3.1-management

# 2. 滚动升级
rabbitmq-upgrade drain   # 逐节点排空
# 升级后重启
rabbitmqctl start_app

# 3. 验证版本
rabbitmqctl status | grep "RabbitMQ"
# 应输出 4.3.1

# 4. 启用关键 Feature Flag
rabbitmqctl enable_feature_flag topic_binding_projection_v5
```

## 注意事项

1. **Topic Exchange 空键绑定**是相对少见的配置陷阱，但一旦触发影响面广——建议检查所有 Topic Exchange 的 Binding Key 是否使用了空字符串
2. `4.3.0` 已移除 Mnesia（仅支持 Khepri），`4.3.1` 继承此架构，无法原地降级到 4.2.x
3. Passive 声明权限放宽是兼容性修复，监控系统可以安全升级

## 参考

- [RabbitMQ 4.3.1 Release Notes](https://github.com/rabbitmq/rabbitmq-server/releases/tag/v4.3.1)
- [Topic Exchange 文档](https://www.rabbitmq.com/tutorials/tutorial-five-python.html)
- [Feature Flags 文档](https://www.rabbitmq.com/docs/feature-flags)
