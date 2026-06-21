---
name: rocketmq-55-enhancements
description: RocketMQ 5.5.0 增强功能：LiteTopic 通配符订阅/消费者暂停、gRPC 并发控制、Proxy TLS 加密私钥、定时消息类型校验、并发心跳、CQ 迭代优化
tags: [rocketmq, mq, lite-mode, grpc, tls, performance]
---

## 概述

RocketMQ 5.5.0（2026-04-10）在 5.4.0 的 Lite Mode（RIP-83）、优先级消息（RIP-80）、RocksDB 存储（RIP-82）等主要特性基础上，进一步增强了 Lite Mode 功能、Proxy 安全性、性能优化和生产可用性。

> 注意：Lite Mode 基础知识已在 `rocketmq-54-features.md` 中详细说明，本文重点介绍 5.5.0 新增的增强功能。

## 1. LiteTopic 增强

### 1.1 通配符订阅

LiteTopic 支持通配符表达式订阅，适用于 AI Agent 的多 Topic 监听场景：

```java
// LiteTopic 通配符订阅
LiteConsumer consumer = provider.newLiteConsumerBuilder()
    .setTopic("ai-agent-*")     // 匹配 ai-agent- 开头的所有 LiteTopic
    .setConsumerGroup("agent-group")
    .setSubscription(Subscription.newBuilder()
        .setTopic("ai-agent-*")
        .setExpression(FilterExpression.SUB_ALL)
        .build())
    .build();
```

### 1.2 消费者暂停与恢复

```shell
# 暂停指定 LiteTopic 的消费者
mqadmin suspendLiteConsumer -b <broker-addr> -g <group> -t <topic>

# 恢复消费者
mqadmin resumeLiteConsumer -b <broker-addr> -g <group> -t <topic>
```

适用于 AI Agent 的按需启停、灰度发布等场景。

### 1.3 PopLite 自动清理

Lite Mode 的长轮询服务（PopLiteLongPollingService）新增自动清理未使用资源的能力：

```properties
# broker.conf
# 自动清理空闲的 Lite 订阅者（默认 600 秒）
liteModeAutoCleanInterval=600
```

## 2. Proxy 增强

### 2.1 gRPC 并发连接数控制

```properties
# proxy.conf — 控制 gRPC 每连接最大并发调用数
grpcMaxConcurrentCallsPerConnection=100
```

防止单个 gRPC 连接耗尽 Proxy 资源，适用于大规模客户端场景。

### 2.2 Proxy TLS 加密私钥支持

```properties
# proxy.conf — 使用密码加密的私钥
tls.key.password=your-key-password
tls.key.chains.path=/path/to/encrypted-key.pem
tls.key.cert.chain.path=/path/to/cert-chain.pem
```

支持密码加密的私钥，提升 Proxy TLS 配置安全性。

### 2.3 Proxy Metrics 导出器

```properties
# proxy.conf — 批量导出 Metrics 避免 OTLP gRPC 导出失败
metricsExporterType=BATCH_OTLP
```

新增 `BatchSplittingMetricExporter` 防止大规模集群下 Metrics 导出失败。

## 3. 定时消息增强

### 3.1 延迟消息类型标记

5.5.0 为自定义延迟消息标记延迟类型：

```java
// 发送延迟消息——自动标记延迟类型
Message message = provider.newMessageBuilder()
    .setTopic("delay-topic")
    .setBody(ByteString.copyFromUtf8("delayed message"))
    .setDelayTimeLevel(3)        // 预设延迟等级
    .build();

// 或使用自定义延迟时间
Message message = provider.newMessageBuilder()
    .setTopic("delay-topic")
    .setBody(ByteString.copyFromUtf8("custom delayed"))
    .putProperties("DELAY", "60000")   // 自定义延迟 60s
    .build();
```

### 3.2 批量发送延迟消息校验

```java
// 批量发送时也会校验延迟消息类型
List<Message> messages = Arrays.asList(msg1, msg2, msg3);
SendReceipt receipt = provider.sendBatch(MessageBatchBuilder.newBuilder()
    .addMessages(messages)
    .build());
// ❌ 如果消息类型与延迟设置冲突，会在批量发送时拒绝
```

### 3.3 事务消息禁止发送延迟消息

```java
// ❌ 5.5.0 修复：事务消息不应发送自定义延迟消息
TransactionMessage transactionMsg = provider.newTransactionMessageBuilder()
    .setTopic("tx-topic")
    .setBody(ByteString.copyFromUtf8("tx message"))
    .setDelayTimeLevel(3)   // ❌ 事务消息不能设置延迟
    .build();
// ⚠️ 5.5.0 会拒绝此类消息，旧版本可能无声通过
```

## 4. 性能优化

### 4.1 并发心跳

```properties
# broker.conf — 大规模集群下并发发送心跳
enableConcurrentHeartbeat=true
```

新 Broker 支持并发发送心跳，适用于数千节点的集群场景，避免心跳串行导致的超时问题。

### 4.2 CQ 迭代器优化

```shell
# 计算消费延迟更高效
mqadmin consumerProgress -g <group-name>
# 5.5.0 优化了 CQ（ConsumeQueue）迭代器和延迟计算逻辑
```

### 4.3 写优化：页面对齐

```properties
# broker.conf — 避免 read-modify-write
writeWithoutMmapPageAlignment=true
```

消息写入时进行页面对齐，避免不必要的读取-修改-写入操作，提升写入性能。

### 4.4 定时消息引擎线程池配置

```properties
# broker.conf — 定时消息恢复服务线程池
timerMessageReputThreads=4
timerMessageReputQueueCapacity=10000
```

定时消息恢复服务线程池可配置，支持优雅关闭。

### 4.5 接收但不消费时不影响重试次数

```java
// 修改不可见时间，但不增加消息重新消费次数
consumer.changeInvisibleTime(receipt, Duration.ofSeconds(30));
// 5.5.0 新行为：不递增 reconsumeTimes
```

## 5. 运维与监控

### 5.1 ThreadPool 拒绝处理器

```properties
# broker.conf — 线程池拒绝策略
threadPoolRejectedHandler=abort  # abort | discard | callerRuns
```

为 `ThreadPoolMonitor` 增加 `RejectedExecutionHandler` 支持，当线程池满载时可控制拒绝行为。

### 5.2 通知请求支持表达式

```shell
# 通知请求支持订阅表达式，实现按需唤醒
mqadmin notificationRequest -b <broker-addr> -t <topic> -e "TAG_A || TAG_B"
```

### 5.3 RocksDB 兼容性改进

```properties
# broker.conf — 主从同步时 RocksDB 兼容性
enableRocksDBCompatibility=true
```

RocksDB 存储的定时/事务消息在主从同步时的兼容性修复。

### 5.4 定时引擎配置正确持久化

```shell
# 切换定时引擎配置现在能正确持久化
mqadmin switchTimerEngine -b <broker-addr> -c RocksDB
# 修复了之前切换后配置键可能不正确的问题
```

## 6. 依赖升级

| 组件 | 5.4.0 → 5.5.0 |
|------|---------------|
| Netty | 4.1.130.Final（CVE 修复） |
| commons-validator | 1.10.0 |
| LZ4 | 迁移到新 groupId |
| Bouncy Castle | 升级修复 CVE |
| Commons Lang3 | 升级修复 CVE |
| RocksDB | 1.0.6 |

## 注意事项

1. **LiteTopic 通配符**：通配符仅适用于 LiteTopic，传统 Topic 不支持
2. **事务+延迟消息**：5.5.0 明确禁止事务消息携带延迟属性，升级后需检查代码
3. **Proxy gRPC 配置**：`grpcMaxConcurrentCallsPerConnection` 默认值可能较小，大规模场景建议调整
4. **TLS 私钥密码**：如果使用加密私钥，需确保 Proxy 能访问密码（环境变量或文件）
5. **并发心跳**：仅对大规模集群（>500 节点）有明显收益，小集群无需开启
6. **页面对齐**：`writeWithoutMmapPageAlignment=true` 会略微增加磁盘占用，但提升写入性能
7. **RocksDB 版本**：升级 RocksDB 到 1.0.6 后，建议验证定时消息恢复流程
