---
name: rabbitmq-quorum-vs-classic
description: RabbitMQ 仲裁队列与经典队列选型指南、迁移策略及性能对比
tags: [rabbitmq, quorum-queues, performance, reliability]
---

## 背景：经典队列的局限性

RabbitMQ 3.8+ 引入仲裁队列（Quorum Queues），3.13 起经典镜像队列已被标记为不推荐，4.0 完全移除镜像队列。新项目应优先考虑仲裁队列。

## 选型决策树

```
需要消息有序性？
├─ 是 → 是否需要持久化/高可用？
│  ├─ 是 → 单活消费者 + 仲裁队列（Raft 保证一致性）
│  └─ 否 → 经典队列（性能最好，但丢数据风险高）
├─ 否 → 是否需要 exactly-once 投递？
│  ├─ 是 → 仲裁队列 + 发布者确认 + 消费者确认
│  └─ 否 → 队列长度？
│     ├─ < 10000 msg/s → 仲裁队列（推荐统一使用）
│     └─ > 10000 msg/s → Stream 队列（更高吞吐）
```

## 核心对比

| 特性 | 经典队列 | 仲裁队列 | 流队列 |
|---|---|---|---|
| 一致性模型 | 最终一致（镜像模式下） | 强一致（Raft） | 强一致 |
| 吞吐量（单节点） | ~50K msg/s | ~15K msg/s | ~200K msg/s |
| 延迟 | 微秒级 | 毫秒级 | 毫秒级 |
| 持久化 | 可选（AMQP 1.0 废除持久化开关） | 始终持久化 | 始终持久化（磁盘） |
| 消息顺序 | 单消费者有序 | 单消费者有序 | 同分区有序 |
| 数据安全 | 低（非镜像易丢） | 高（Raft 多数派） | 高 |
| 内存使用 | 低 | 中（每个节点有完整副本） | 低（分段存储） |
| 弹性扩容 | 手动 | 自动（Raft 集群管理） | 自动 |
| FIFO | 支持 | 支持（但所有消息先到 Raft Leader） | 不支持 |

## 仲裁队列配置参数

```java
// Spring Boot 配置仲裁队列
@Bean
public Queue quorumQueue() {
    return QueueBuilder.durable("order.queue")
        .quorum() // 声明为仲裁队列
        .withArgument("x-quorum-initial-group-size", 3) // 初始集群大小
        .withArgument("x-max-length", 1000000)
        .withArgument("x-overflow", "reject-publish")
        .build();
}
```

等效声明方式（在 rabbitmq.conf 或 Policy）：

```
# Policy 配置：将所有 durable 队列转为仲裁队列
rabbitmqctl set_policy quorum "^.*\\.durable\\." \
  '{"quorum-initial-group-size":3,"delivery-limit":5}' \
  --priority 1 --apply-to queues
```

## 从镜像队列迁移到仲裁队列

### 迁移策略（零停机）

```
1. 创建新的仲裁队列（不同名称）
2. 配置 Shovel/Federation 将消息从旧镜像队列转发到新仲裁队列
3. 逐步切换消费者到新队列
4. 确认所有消费者迁移完成后，删除旧队列
```

```bash
# 使用 shovel 迁移（动态配置）
rabbitmqctl set_parameter shovel migrate-orders \
  '{"src-protocol": "amqp091",
    "src-queue": "orders.mirrored",
    "dest-protocol": "amqp091",
    "dest-queue": "orders.quorum",
    "src-delete-after": "queue-length"}'
```

## 仲裁队列的坑点

### 1. 事务不支持

```java
// ❌ 仲裁队列不支持事务
channel.txSelect();  // 抛出异常

// ✅ 使用发布者确认替代
channel.confirmSelect();
channel.waitForConfirmsOrDie(5000);
```

### 2. 排他队列不支持

```java
// ❌ 仲裁队列不能设为排他
QueueBuilder.durable("q").quorum().exclusive(); // 报错
```

### 3. 消息大小限制

```java
// 仲裁队列建议消息 < 64MB（默认限制）
@Bean
public Queue quorumQueue() {
    return QueueBuilder.durable("large-msg.queue")
        .quorum()
        .withArgument("x-max-in-memory-length", 0) // 禁止内存缓存
        .withArgument("x-max-in-memory-bytes", 0)
        .build();
}
```

### 4. 消费者数量限制

仲裁队列每个队列最多约 256 个消费者（与节点数相关）。如果需要更多消费者，考虑使用流队列或增加队列分区。

### 5. delivery-limit 防无限重试

```java
@Bean
public Queue quorumQueue() {
    return QueueBuilder.durable("order.queue")
        .quorum()
        .withArgument("x-delivery-limit", 3) // 最多重试3次
        .build();
}
```

达到 delivery-limit 后，消息被丢弃或路由到 DLQ（需配置死信交换器）。

## 性能调优

```bash
# 提高 Raft 性能（rabbitmq.conf）
raft.segment_max_entries = 10000
raft.wal_max_size_bytes = 1024000000
# 增加 Raft 选举超时防止误选举
cluster_partition_handling = pause_minority
```

## 注意事项

- **新项目默认使用仲裁队列**，除非你能证明性能瓶颈
- **经典队列适合**：临时队列、RPC 回调队列、高吞吐非关键日志
- **流队列适合**：>10K msg/s 的日志流、事件溯源、离线消费者积压
- **仲裁队列的强一致性**意味着网络分区时可能不可用（需要多数派节点）
- **不要在同一队列上混合仲裁和经典**：集群内可以共存，但 Policy 可能冲突
- **RabbitMQ 4.0+** 已移除经典镜像队列，升级前需确认迁移计划
