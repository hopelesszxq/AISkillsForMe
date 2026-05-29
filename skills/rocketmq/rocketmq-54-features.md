---
name: rocketmq-54-features
description: RocketMQ 5.4+/5.5+ 新特性：优先级消息、RocksDB 事务/定时消息、Lite 订阅模式
tags: [rocketmq, mq, priority, lite-mode, rocksdb, timer]
---

## 概述

RocketMQ 在 5.4.0（2025-12）和 5.5.0（2026-04）中引入了三个重要新特性：

- **5.4.0**：优先级消息（RIP-80）、RocksDB 存储定时/事务消息（RIP-82）
- **5.5.0**：Lite Mode 轻量级订阅模式（RIP-83），专为 AI 场景优化

## 1. 优先级消息（RIP-80, 5.4.0+）

### 功能说明

优先级消息允许消息在队列中按优先级排序，高优先级的消息会被优先消费。适用于：VIP 用户订单处理、告警消息分级、付费用户优先服务等场景。

### 使用方式

```java
// 发送优先级消息（优先级范围：0-10，数值越大优先级越高）
Message message = provider.newMessageBuilder()
    .setTopic("priority-topic")
    .setBody(ByteString.copyFromUtf8("High priority message"))
    .setPriority(10)  // 设置优先级
    .build();

// 消费者无需特殊配置，自动按优先级消费
SimpleConsumer consumer = provider.newSimpleConsumerBuilder()
    .setClientConfiguration(ClientConfiguration.newBuilder()
        .setEndpoints("localhost:8081")
        .build())
    .setSubscription(Subscription.newBuilder()
        .setTopic("priority-topic")
        .setExpression(FilterExpression.SUB_ALL)
        .build())
    .build();
```

### Broker 配置

```properties
# broker.conf
# 开启优先级消息支持
enablePriorityMessage=true
# 优先级队列数量（默认 10）
priorityQueueNum=10
```

### 优先级存储原理

- 每个 Topic 内部维护多个优先级队列
- 消息根据优先级写入对应级别的队列
- 消费时 Broker 按高 → 低优先级遍历队列投递
- 同优先级内仍保证 FIFO 顺序

## 2. RocksDB 存储（RIP-82, 5.4.0+）

### 功能说明

将 RocketMQ 的**定时消息**、**事务消息**和**消费索引**从内存/文件存储迁移到 RocksDB 存储后端，带来以下好处：

- 定时消息：支持海量定时任务，不再受内存限制
- 事务消息：回查状态持久化，Broker 重启不丢失
- 消费索引：Pop 消费模式下索引可靠性提升

### 启用配置

```properties
# broker.conf — 切换定时消息引擎到 RocksDB
timerEngine=RocksDB

# broker.conf — 事务消息存储到 RocksDB
transactionStore=RocksDB

# broker.conf — Pop 消费索引使用 RocksDB
enableRocksDBPopIndex=true

# RocksDB 存储路径
rocksDBStorePath=/data/rocketmq/rocksdb
```

### 运行时切换

```shell
# 动态切换定时引擎（无需重启）
mqadmin switchTimerEngine -b <broker-addr> -c RocksDB
mqadmin switchTimerEngine -b <broker-addr> -c TimerWheel  # 切回原引擎
```

### 迁移注意事项

```shell
# 1. 先备份原有定时消息
mqadmin queryTimerMessage -b <broker-addr> -t <topic> -f backup.dat

# 2. 切换引擎
mqadmin switchTimerEngine -b <broker-addr> -c RocksDB

# 3. 验证切换状态
mqadmin getTimerEngineStatus -b <broker-addr>
```

## 3. Lite Mode（RIP-83, 5.5.0+）

### 功能说明

Lite Mode（轻量级订阅模式）是 RocketMQ 5.5.0 引入的全新消息模型，专为 **AI 场景下的高效消息传递**设计。相比传统模式，资源消耗更低、生命周期管理更完善。

### 核心特性

- **轻量级消息处理器**：减少内存和 CPU 开销
- **新事件分发机制**：基于事件驱动的长轮询
- **订阅注册管理**：更灵活的订阅方式
- **精简协议支持**：适用于 IoT / AI Agent 等轻客户端

### 适用场景

| 场景 | 传统模式 | Lite Mode |
|------|----------|-----------|
| AI Agent 通信 | 太重 | 轻量高效 |
| IoT 设备消息 | 协议复杂 | 精简协议 |
| 边缘节点 | 资源消耗大 | 资源友好 |
| 短生命周期 Topic | 管理成本高 | 动态注册 |

### 配置启用

```properties
# broker.conf
# 启用 Lite Mode
enableLiteMode=true
# Lite Mode 长轮询超时（毫秒）
liteModeLongPollingTimeout=30000
# Lite Mode 控制处理器线程数
liteModeHandlerThreads=8
```

### 客户端使用

```java
// Lite Mode 订阅（使用轻量级 API）
LiteConsumer consumer = provider.newLiteConsumerBuilder()
    .setTopic("ai-agent-topic")
    .setConsumerGroup("ai-agent-group")
    .setSubscription(Subscription.newBuilder()
        .setTopic("ai-agent-topic")
        .setExpression(FilterExpression.SUB_ALL)
        .build())
    .build();

// 消息处理
consumer.setMessageListener(message -> {
    String payload = message.getBody().toStringUtf8();
    // AI Agent 处理逻辑
    return ConsumeResult.SUCCESS;
});
```

## 4. 其他改进（5.4.0-5.5.0）

### 并发心跳

```properties
# broker.conf — 支持并发发送心跳，适合大规模集群
enableConcurrentHeartbeat=true
```

### 写优化

```properties
# broker.conf — 页面对齐避免 read-modify-write
writeWithoutMmapPageAlignment=true
```

### ACL 2.0

5.3.3 起不再支持 ACL 1.0，必须迁移到 ACL 2.0：

```properties
# broker.conf
aclEnable=true
aclVersion=2.0
```

## 注意事项

1. **优先级消息依赖 Broker 配置**：需显式开启 `enablePriorityMessage=true`
2. **RocksDB 切换不可逆**：从 TimerWheel 切换到 RocksDB 后，已存储在 RocksDB 的定时消息无法自动切回
3. **Lite Mode 新客户端**：需使用 5.5.0+ 的客户端 SDK，旧客户端不兼容
4. **ACL 1.0 已废弃**：5.3.3+ 需使用 ACL 2.0，升级前需迁移配置
5. **RocksDB 存储路径**：建议使用 SSD，且预留 50% 以上空间
6. **优先级消息与顺序消息冲突**：优先级消息不保证同优先级外的顺序性，不可与严格顺序消息混用
