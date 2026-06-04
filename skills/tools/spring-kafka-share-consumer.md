---
name: spring-kafka-share-consumer
description: Spring Kafka 4.1 共享消费者模式：ShareAckMode、KIP-932/KIP-1206 实现、Async Commit、Kafka Streams DLQ
tags: [tools, kafka, spring-kafka, share-consumer, messaging, streams]
---

## 概述

Spring Kafka 4.1.0-RC1（2026-04-20 发布）引入了 **共享消费者（Share Consumer，KIP-932）** 的完整支持，以及 Kafka Streams 原生 DLQ（KIP-1034）和服务端重平衡（KIP-1071）等多项重要特性。

> 共享消费者（KIP-932）是 Kafka 3.7+ 引入的新消费模型——多个消费者可以同时消费同一分区，不再需要消费组内的"独占分区"模式，适合高吞吐、低延迟场景。

## 一、Share Consumer 基础配置

### 依赖

```xml
<!-- Spring Kafka 4.1.x（配合 Spring Boot 4.1） -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <version>4.1.0-RC1</version>
</dependency>
```

### 配置属性

```yaml
spring:
  kafka:
    consumer:
      # Share Consumer 模式下使用共享分组 ID
      group-id: share-group-1
    # Share Consumer 容器工厂配置
    listener:
      type: SHARE # 声明为共享消费者
```

### 基本用法

```java
@Component
public class ShareConsumerExample {

    @KafkaListener(topics = "share-topic", groupId = "share-group-1")
    public void listenShare(ConsumerRecord<String, String> record) {
        // 多个实例可以同时消费同一分区
        System.out.printf("Share consumer received: key=%s, value=%s, partition=%d%n",
            record.key(), record.value(), record.partition());
    }
}
```

## 二、ShareAckMode 确认模式

Spring Kafka 4.1 引入 `ShareAckMode` 枚举，控制共享消费者的消息确认行为：

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| `AUTO` | 自动确认（默认），监听方法返回即确认 | 大部分场景 |
| `MANUAL` | 手动确认，需调用 `Acknowledgment.acknowledge()` | 需要精细控制 |
| `MANUAL_IMMEDIATE` | 立即手动确认，不等批量 | 低延迟要求 |

```java
@KafkaListener(topics = "share-topic", groupId = "share-group-2")
public void listenWithAck(ConsumerRecord<String, String> record,
                          Acknowledgment ack) {
    try {
        // 处理消息
        processMessage(record);
        // 手动确认
        ack.acknowledge();
    } catch (Exception e) {
        // 不调用 acknowledge() 则消息会被重新投递
        log.error("处理失败，消息将重新投递", e);
    }
}
```

### 异步确认（Async Commit）

```java
@Bean
public ShareConsumerContainerCustomizer containerCustomizer() {
    return container -> container.setAsyncCommit(true);
}
```

## 三、Share Acquire Mode（KIP-1206）

KIP-1206 引入了 `share.acquire.mode` 配置，控制共享消费者的消息获取策略：

```yaml
spring:
  kafka:
    properties:
      share:
        acquire:
          mode: default # 默认模式
          # 可选值：default, subscribe-only
```

```java
// 通过 Properties 配置
@Bean
public Map<String, Object> consumerConfigs() {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.GROUP_ID_CONFIG, "share-group-3");
    props.put("share.acquire.mode", "default");
    return props;
}
```

## 四、Share Consumer 生命周期事件

```java
@Component
public class ShareContainerLifecycleListener {

    private static final Logger log = LoggerFactory.getLogger(ShareContainerLifecycleListener.class);

    @EventListener
    public void onContainerStart(ShareConsumerContainerStartEvent event) {
        log.info("Share consumer container started: {}", event.getSource());
    }

    @EventListener
    public void onContainerStop(ShareConsumerContainerStopEvent event) {
        log.info("Share consumer container stopped: {}", event.getSource());
    }

    @EventListener
    public void onContainerFailure(ShareConsumerContainerFailureEvent event) {
        log.error("Share consumer container failure", event.getSource());
    }
}
```

## 五、Kafka Streams 新特性

### 原生 DLQ 支持（KIP-1034）

Kafka Streams 3.8+ 支持在 Streams DSL 中直接配置死信队列：

```yaml
spring:
  kafka:
    streams:
      properties:
        dlq:
          topic.name: "my-dlq-topic"
          context.headers.record: true
```

```java
@Bean
public KStream<String, String> kStream(StreamsBuilder builder) {
    KStream<String, String> stream = builder
        .stream("input-topic", Consumed.with(Serdes.String(), Serdes.String()));

    stream
        .mapValues(value -> {
            // 如果这里抛出异常，消息自动进入 DLQ
            return processValue(value);
        })
        .to("output-topic");

    return stream;
}
```

### 服务端重平衡（KIP-1071, Kafka 4.0+）

暴露 `group.protocol` 配置，启用服务端协调的重平衡：

```yaml
spring:
  kafka:
    streams:
      properties:
        group.protocol: consumer # classic (default) | consumer (KIP-1071)
```

## 六、使用场景

| 场景 | 传统 Consumer Group | Share Consumer |
|------|-------------------|----------------|
| 分区独占 | ✅ 每个分区只能被一个消费者消费 | ❌ 多消费者共享分区 |
| 消费顺序 | ✅ 分区内严格有序 | ❌ 不保证顺序 |
| 吞吐量扩展 | 需增加分区数 | 可增加消费者实例 |
| 延迟 | 中等 | **更低**（可并行处理同一分区） |
| 适用场景 | 需要严格顺序 | **高吞吐、可乱序处理的场景** |

## 注意事项

1. **Kafka 版本要求**：Share Consumer 需要 Kafka Broker 3.7+，建议使用 Kafka 4.0+
2. **顺序性**：Share Consumer 不保证消息顺序，需要顺序消费的场景仍应使用传统 Consumer Group
3. **幂等消费**：由于共享消费者不保证 exactly-once 语义，消费端需自行实现幂等性
4. **Streams DLQ 版本**：KIP-1034 原生 DLQ 需要 Kafka Streams 3.8+（对应 Kafka 3.8+）
5. **Spring Kafka 版本**：4.1.0-RC1 为里程碑版本，生产环境请等待 GA 发布
