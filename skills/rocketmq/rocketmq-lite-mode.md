---
name: rocketmq-lite-mode
description: RocketMQ 5.5.0 Lite Mode——面向 AI 场景的轻量级订阅机制
tags: [rocketmq, mq, messaging, lite-mode, ai, 5.5]
---

## 概述

RocketMQ 5.5.0（2026-04-10 发布）引入了 **Lite Mode（轻量级模式，RIP-83）**，一种全新的消息模型——Lite Topic，专为 AI 场景设计。相比传统 Topic 模式，Lite Mode 资源消耗更低、生命周期管理更轻量。

> 已有技能 `rocketmq5-new-features.md` 覆盖了 5.x 的 DLedger Controller、gRPC 协议、流式分区等架构特性，本文作为 5.5.0 新特性的补充。

## Lite Mode 设计目标

### 传统 Topic 的问题

- 每个 Topic 需预先创建队列（Queue），资源占用固定
- 海量 Topic 场景下 Broker 内存和文件句柄压力大
- AI 场景中消息主题数量巨大（多模型、多任务），传统 Topic 模型成本高

### Lite Mode 解决方案

```
传统模式: Topic → Queue[] → ConsumerGroup
Lite Mode:  LiteTopic（动态生命周期）→ Consumer（按需订阅）
```

Lite Mode 提供：
- **按需创建**：Topic 无需预先创建，首次使用时自动初始化
- **自动回收**：Topic 无活动时自动销毁，释放资源
- **极低开销**：单个 Lite Topic 的资源占用仅为传统 Topic 的 10-20%

## 核心架构

### 新增组件

| 组件 | 作用 |
|------|------|
| Lite Message Processor | 处理 Lite 模式的消息收发 |
| Long-Polling Service | 支持长轮询消费 |
| Control Handler | 管理 Lite Topic 生命周期 |
| Metrics Monitor | 监控 Lite Topic 运行状态 |

### 工作流程

```
生产者发送 Lite 消息
  → Lite Message Processor 接收
  → 动态创建/关联 Lite Topic
  → Long-Polling Service 等待消费者
  → 消费者接收并确认消息
  → 空闲超时后 Lite Topic 自动回收
```

## 配置与使用

### 服务端配置

```properties
# broker.conf - 启用 Lite Mode
liteModeEnabled=true

# Lite Topic 空闲超时（默认 5 分钟）
liteTopicIdleTimeoutInMills=300000

# Lite Mode 最大 Topic 数（默认 10000）
liteTopicMaxCount=10000

# Lite 消息处理器线程数
liteMessageProcessorThreads=16
```

### Java 客户端（gRPC 方式）

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client-java</artifactId>
    <version>5.5.0</version>
</dependency>
```

```java
// 生产者 - 发送 Lite 消息
ClientServiceProvider provider = ClientServiceProvider.loadStatic();

// Lite Topic 无需预先创建，动态生成
Message message = provider.newMessageBuilder()
    .setTopic("ai-inference-tasks")
    .setBody(ByteString.copyFromUtf8("{\"model\":\"gpt-4o\",\"prompt\":\"...\"}"))
    .putProperties("x-lite-topic", "true")  // 标记为 Lite 消息
    .build();

SendReceipt receipt = provider.send(message);
System.out.println("Lite message sent: " + receipt.getMessageId());
```

```java
// 消费者 - 使用 SimpleConsumer 订阅 Lite Topic
SimpleConsumer consumer = provider.newSimpleConsumerBuilder()
    .setClientConfiguration(ClientConfiguration.newBuilder()
        .setEndpoints("localhost:8081")
        .build())
    .setSubscription(Subscription.newBuilder()
        .setTopic("ai-inference-tasks")
        .setExpression(FilterExpression.SUB_ALL)
        .build())
    .build();

// Lite Mode 支持长轮询，减少空轮询开销
while (true) {
    List<MessageView> messages = consumer.receive(10, Duration.ofSeconds(30));
    for (MessageView msg : messages) {
        String body = msg.getBody().toStringUtf8();
        System.out.println("Received Lite: " + body);
        consumer.ack(msg.getMessageId());
    }
}
```

### Spring Boot 集成

```java
@Component
public class LiteMessageProducer {

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    public void sendLiteMessage(String topic, String payload) {
        // Lite Mode 使用普通发送接口，服务端自动识别
        Message<String> msg = MessageBuilder
            .withPayload(payload)
            .setHeader("x-lite-topic", "true")
            .build();
        rocketMQTemplate.send(topic, msg);
    }
}
```

## AI 场景应用

### 1. 模型推理任务分发

```java
// 每个模型推理任务使用独立的 Lite Topic
// 任务完成后 Topic 自动释放
String modelTaskTopic = "inference-" + taskId + "-" + modelName;

// 生产者提交推理任务
producer.send(Message.newBuilder()
    .setTopic(modelTaskTopic)
    .setBody(ByteString.copyFromUtf8(taskPayload))
    .putProperties("x-lite-topic", "true")
    .build());
```

### 2. 批量流式处理

```java
// 流式推理结果使用 Lite Topic 多路分发
for (ModelResult result : batchResults) {
    String topic = "stream-result-" + result.getSessionId();
    producer.send(Message.newBuilder()
        .setTopic(topic)
        .setBody(ByteString.copyFromUtf8(result.toJson()))
        .putProperties("x-lite-topic", "true")
        .build());
}
```

### 3. 动态模型热加载通知

```java
// 模型版本更新时，通过 Lite Topic 通知推理节点
// Topic 按模型名称动态生成，无需提前规划
producer.send(Message.newBuilder()
    .setTopic("model-update-" + modelName)
    .setBody(ByteString.copyFromUtf8(newVersionManifest))
    .putProperties("x-lite-topic", "true")
    .build());
```

## 5.5.0 其他改进

| 特性 | 说明 |
|------|------|
| MessageQueueSelector 重构 | 支持更灵活的队列选择策略 |
| 心跳并发发送 | 支持向 Broker 并发发送心跳，降低延迟 |
| 事务消息修正 | 事务消息不再发送自定义延迟消息 |
| RocksDB 升级 | Broker RocksDB 从 1.0.2 升级到 1.0.6 |
| 分层存储优化 | 减少消费端偏移量时间戳的冗余请求 |
| 服务端去压缩 | 去掉服务端消息体解压缩步骤，减少 CPU 开销 |

## 注意事项

1. **Lite Mode 需要 5.5.0+ Broker**：旧版本 Broker 不支持 Lite Mode，发送时会作为普通消息处理
2. **性能基准**：Lite Mode 适用于 Topic 数量大但单个 Topic 吞吐低的场景（典型 AI 场景），高吞吐核心业务仍建议使用传统 Topic
3. **超时配置**：`liteTopicIdleTimeoutInMills` 设置太短会导致频繁创建/销毁 Topic
4. **客户端兼容**：5.5.0 客户端兼容所有 5.x Broker，但 Lite Mode 功能仅在 5.5.0+ Broker 生效
5. **监控集成**：Lite Topic 的指标通过 Metrics Monitor 组件暴露，需额外配置指标采集
6. **Java 17+ 推荐**：RocketMQ 5.5.0 对 Java 17 做了更好的支持，建议使用 Java 17 运行 Broker
