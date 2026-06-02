---
name: rabbitmq-431-fixes
description: RabbitMQ 4.3.1 关键修复与 operation 指南：Topic 绑定 Bug、被动声明权限修正、Quorum 队列增强
tags: [rabbitmq, mq, 4.3.1, bugfix, quorum, topic]
---

## 概述

RabbitMQ 4.3.1（2026-05-20 发布）是 4.3.x 系列的维护版本，修复了多个**影响生产环境的 Bug**，特别是 Topic Exchange 空路由键的错误路由问题需要手动启用功能标记。

> 如果正在使用 RabbitMQ 4.3.x，**强烈建议升级到 4.3.1**。

## 关键修复

### 1. Topic Exchange 空绑定键错误路由（⚠️ 需手动启用）

**问题**：当队列使用空字符串绑定到 Topic Exchange 时，发布到**任何** Topic Exchange 的空路由键消息都会被错误路由到该队列。

**修复**：升级到 4.3.1 后，需手动启用 `topic_binding_projection_v5` 功能标记：

```bash
# 在所有节点升级后，任选一个节点执行
rabbitmqctl enable_feature_flag topic_binding_projection_v5
```

```bash
# 确认功能标记状态
rabbitmqctl list_feature_flags | grep topic_binding_projection
```

**影响范围**：任何使用了 `binding_key=""`（空字符串）绑定到 Topic Exchange 的场景。

**建议操作**：
- 升级后立即检查代码中是否有 `bindingKey("")` 或 `withRoutingKey("")` 绑定到 Topic Exchange
- 先在生产集群的测试环境启用标记验证，再推到生产

### 2. 被动声明权限变更

**变更**：`queue.declare`（passive=true）和 `exchange.declare`（passive=true）现在允许对 VHost 拥有**任意权限**（configure/write/read）的用户执行，不再仅限 `configure` 权限。

- **之前**：只有 `configure` 权限用户可执行被动声明
- **现在**：`write` 或 `read` 权限用户也可执行

```java
// Spring Boot — 使用被动声明确认队列存在
@Bean
public Queue queue() {
    // 使用 passive() 声明，仅检查存在性
    return QueueBuilder.durable("orders.queue")
        .passive()  // 4.3.1 起: write/read 权限也能通过
        .build();
}
```

### 3. Quorum Queue 修复

| 修复 | 说明 | 影响 |
|------|------|------|
| 负优先级处理 | Quorum Queue 现在正确处理负优先级值 | 使用负优先级的消费者 |
| 延迟重试策略键 | `delayed-retry` 相关策略键现在被正确接受 | 使用 delayed-retry 策略 |
| 至少一次死信 | 修复死信命令可能发到错误副本的问题 | 高可用死信场景 |
| Raft WAL 默认条目 | 恢复 500K 的 Raft WAL 最大条目默认值 | 避免 WAL 无限增长 |
| 恢复崩溃修复 | 修复非正常关闭后恢复时可能崩溃的问题 | 所有 Quorum Queue 用户 |

**建议配置**：

```properties
# rabbitmq.conf — 4.3.1 推荐参数
quorum_commands_soft_limit = 64
raft.wal.max_entries = 500000  # 显式设置（4.3.1 已恢复默认值）
```

### 4. 其他修复

- **连接 API**：`GET /api/connections` 在 STOMP 连接存在时不再返回 500
- **MQTT keepalive**：已关闭连接上的 socket 错误不再导致异常日志
- **Federation**：修复滚动重启期间 Federation 链接无法启动的问题
- **TLS**：插件 TLS 配置中的 `fail_if_no_peer_cert` 现在被正确使用

## 升级建议

```bash
# 1. 所有节点升级到 4.3.1
# 滚动升级（先升级非磁盘节点，再升级磁盘节点）
# 2. 确认所有节点升级完成
rabbitmqctl cluster_status
# 3. 启用 Topic Binding v5 功能标记
rabbitmqctl enable_feature_flag topic_binding_projection_v5
# 4. 验证功能标记就绪
rabbitmqctl list_feature_flags
```

## 注意事项

1. **功能标记不可逆**：`topic_binding_projection_v5` 一旦启用不能回退，确保在升级窗口一次性完成
2. **升级路径**：只有 4.2.x 及以上版本才能[原地升级](https://www.rabbitmq.com/docs/upgrade)到 4.3.x
3. **Stream Queue**：4.3.1 改进了 stream queue 参数验证，升级后如有 steam 参数错误会在声明时提前暴露
4. **Cache 修复**：4.3.1 修复了重连客户端缓存失效的问题，使用连接池/短连接的重连场景会明显改善
