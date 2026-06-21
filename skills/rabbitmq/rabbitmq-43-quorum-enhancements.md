---
name: rabbitmq-43-quorum-enhancements
description: RabbitMQ 4.3.0 Quorum Queue 增强特性：延迟重试、严格优先级、消费者超时、恢复快照与内存优化
tags: [rabbitmq, mq, quorum-queue, delayed-retry, priority, consumer-timeout]
---

## 概述

RabbitMQ 4.3.0（2026-04-23 发布）升级了 Ra 依赖至 3.x，引入**第 8 版 Quorum Queue 状态机**，新增多项关键特性：严格优先级队列、可配置延迟重试、消费者超时监控、恢复快照和内存优化。

> ⚠️ 4.3.0 移除了 Mnesia 和分区策略 — 迁移前请阅读 `rabbitmq-43-migration.md`。

## 一、严格优先级队列（Strict Priority）

Quorum Queue 现在支持**严格的优先级排序**，消息按优先级从高到低消费，不会因为重新入队而打乱优先级顺序。

```properties
# rabbitmq.conf 全局默认
quorum_queue.priority_queue = true  # 默认开启
```

```bash
# 声明带优先级的 Quorum Queue
rabbitmqadmin declare queue name=priority.orders \
  arguments='{"x-queue-type":"quorum","x-max-priority":10}'
```

```java
// Spring Boot 声明
@Bean
public Queue priorityQueue() {
    return QueueBuilder.durable("priority.orders")
        .quorum()            // 声明为 Quorum Queue
        .maxPriority(10)     // 设置最大优先级 0~255
        .build();
}

// 发送高优先级消息
@Autowired
private RabbitTemplate rabbitTemplate;

public void sendUrgent(String orderId) {
    MessageProperties props = new MessageProperties();
    props.setPriority(10);   // 0=最低, 255=最高
    rabbitTemplate.send("priority.orders",
        MessageBuilder.withBody(orderId.getBytes())
            .andProperties(props)
            .build());
}
```

**与 Classic Queue 优先级的主要区别**：

| 特性 | 新 Quorum 优先级 | 旧 Classic 优先级 |
|------|-----------------|-------------------|
| 严格排序 | ✅ 保证优先级严格排序 | ❌ 仅尽力排序 |
| 重新入队优先级保持 | ✅ 正确处理 | ❌ 可能丢失优先级 |
| 优先级感知过期 | ✅ 先过期低优先级消息 | ❌ 按入队顺序过期 |
| 各优先级消息计数 | ✅ Management UI 可见 | ❌ 不可见 |

## 二、延迟重试（Delayed Retry）

Quorum Queue 现在内置支持**指数退避重试**，无需再借助死信队列+延迟重试交换机的组合模式。

```properties
# 全局默认设置
quorum_queue.delayed_retry.enabled = true
quorum_queue.delayed_retry.initial_interval = 1000   # 初始延迟 1s
quorum_queue.delayed_retry.max_interval = 60000       # 最大延迟 60s
quorum_queue.delayed_retry.backoff_multiplier = 2.0   # 指数因子
```

```bash
# 通过 Policy 按队列配置延迟重试
rabbitmqctl set_policy delayed-retry "^retry\." '{
  "delayed-retry-enabled": true,
  "delayed-retry-initial-interval": 1000,
  "delayed-retry-max-interval": 30000,
  "delayed-retry-multiplier": 2.0
}' --apply-to queues --priority 1
```

```java
// Spring Boot 声明 - 结合死信策略
@Bean
public Queue retryQueue() {
    // 注意: delayed-retry 参数目前需要通过 arguments 或 policy 配置
    return QueueBuilder.durable("retry.orders")
        .quorum()
        .deadLetterExchange("dlx.orders")     // 超过最大重试次数后走 DLX
        .deadLetterRoutingKey("failed")
        .build();
}
```

```yaml
# 通过 Policy 管理（推荐，无需改代码）
# rabbitmqctl set_policy ...
```

**行为说明**：
- 消息被 `nack`（basic.nack/reject）且 `requeue=false` 时触发延迟重试
- 每次重试失败后等待时间按 `倍数 × 上次等待时间` 增长，上限 `max_interval`
- 超过 `delivery-limit`（默认 20）后进入死信队列

## 三、消费者超时（Consumer Timeout）

Quorum Queue 支持为未确认消息设置**消费者超时阈值**，超时后自动重新入队或死信。

```properties
# rabbitmq.conf
quorum_queue.consumer_timeout = 300000  # 5 分钟（毫秒）
consumer_timeout.mqtt.enabled = true    # MQTT 协议也启用
```

```bash
# 通过 Policy 按队列配置
rabbitmqctl set_policy consumer-timeout "^orders\." '{
  "consumer-timeout": 120000
}' --apply-to queues --priority 1
```

**协议支持**：
- **AMQP 0-9-1**：✅ 默认启用
- **AMQP 1.0**：✅ 默认启用
- **MQTT**：✅ 需 `consumer_timeout.mqtt.enabled=true`
- Classic Queue / Stream：❌ 不支持

**超时处理流程**：
```
消息投递 → 消费者未 ack → 超时阈值到达
    ├─ 重新入队 → 投递给其他消费者（如果有）
    └─ 超过 delivery-limit → 进入死信队列（如果配置了 DLX）
```

## 四、恢复快照（Recovery Snapshots）

Quorum Queue 引入了 WAL 快照机制，**显著减少节点重启后的恢复时间**。

```properties
# rabbitmq.conf — 快照相关参数
quorum_queue.snapshot_throttling.enabled = true   # 默认启用
raft.wal.max_entries = 500000                     # WAL 最大条目数
```

**工作原理**：
1. 正常运行期间定期生成**一致性快照**（包含队列状态、消息元数据）
2. 节点重启时直接从快照恢复，跳过大量 WAL 回放
3. 快照节流策略通过 WAL 填充率和可回收字节数决定快照时机

**性能收益**：
- 百万级消息队列：恢复时间从 **分钟级** 降至 **秒级**
- 快照节流避免频繁 snapshot 对正常流量的影响

## 五、内存优化

| 优化项 | 说明 | 效果 |
|--------|------|------|
| 紧凑消息引用 | 使用打包整数存储消息引用 | 单消息内存减少约 20% |
| 延迟键优化元组存储 | 优化延迟消息的排序键存储结构 | 大量延迟消息场景内存减少 30%+ |
| 移除 `rabbit_fifo_index` | 删除冗余索引结构 | 每个队列减少固定内存开销 |
| WAL 回收字节追踪 | 更精确跟踪可回收 WAL 字节 | 减少不必要的 WAL 写入 |

## 六、其他增强

| 特性 | 说明 |
|------|------|
| 无限制消息返回 | delivery-limit 不再限制显式的 `basic.return` |
| 副本删除清理 | 删除副本时同时清理关联的 Raft 日志 |
| 清理时死信处理 | Purge 队列时同时清除待投递的至少一次死信消息 |
| 运行时修改 delivery-limit | 通过 Policy 修改后立即生效，无需重新声明队列 |
| AMQP 1.0 SAC 通知 | 单活跃消费者状态变更通知 AMQP 1.0 客户端 |

## 注意事项

1. **优先级队列开启后不可动态切换**：`x-max-priority` 在队列声明后不可修改
2. **延迟重试需要 RabbitMQ 4.3.0+**：不要与 `rabbitmq-delayed-message-exchange` 插件混淆，这是完全不同的实现
3. **Consumer Timeout 与原有 consumer_timeout 区别**：原有 `consumer_timeout` 是 Channel 级别的（检测消费者无响应），Quorum Queue 的 consumer_timeout 是消息级别的（检测 unacked 消息超时）
4. **升级到 4.3.0 后才能使用**：这些特性依赖第 8 版 Quorum Queue 状态机，4.2.x 无法使用
5. **启用延迟重试的队列仍可配置 DLX**：延迟重试超出 delivery-limit 后进入 DLX，作为最终兜底
