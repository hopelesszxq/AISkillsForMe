---
name: rocketmq-pop-consumption
description: RocketMQ 5.x POP 消费模式：原理、配置、对比 Push/Pull 与实战场景
tags: [rocketmq, mq, messaging, pop, consumption, 5x]
---

## 概述

POP 消费模式是 RocketMQ 5.0 引入的全新消费模型。客户端主动发起 POP 请求拉取消息，Broker 维护消息的不可见期（Invisible Duration）、ACK 确认、续租和超时恢复机制。相比传统 Push 模式，POP 模式更轻量、无状态，适合大规模客户端和云原生场景。

## 1. 三种消费模式对比

| 特性 | PushConsumer（传统） | PullConsumer（传统） | SimpleConsumer（POP 模式） |
|------|---------------------|---------------------|--------------------------|
| **引入版本** | 4.x | 4.x | 5.0+ |
| **消费模型** | Broker 推送 | 客户端主动拉取 | 客户端 POP 请求 |
| **重平衡** | 客户端自动 Rebalance | 客户端自行管理 | Broker 端无状态（无需 Rebalance） |
| **队列锁定** | 客户端锁定队列 | 无 | 无（Broker 管理可见性） |
| **消费进度管理** | 客户端维护 Offset | 客户端自行维护 | Broker 端管理 |
| **重试/死信** | 内置集群重试队列 | 需自行实现 | 内置，支持不可见期超时重试 |
| **无状态消费** | ❌ 有状态 | ✅ | ✅ |
| **适用场景** | 传统 Java 应用 | 自定义消费逻辑 | 云原生、大规模客户端、多语言 |

## 2. SimpleConsumer（POP 模式）实战

### Java 客户端示例

```java
// Maven 依赖
// <dependency>
//     <groupId>org.apache.rocketmq</groupId>
//     <artifactId>rocketmq-client-java</artifactId>
//     <version>5.0.7</version>
// </dependency>

import org.apache.rocketmq.client.consumer.SimpleConsumer;
import org.apache.rocketmq.client.consumer.MessageSelector;
import org.apache.rocketmq.common.message.MessageExt;
import java.time.Duration;
import java.util.List;

public class PopConsumerExample {

    public static void main(String[] args) throws Exception {
        // 构建 SimpleConsumer（POP 模式）
        SimpleConsumer consumer = SimpleConsumer.newBuilder()
            .withClientConfiguration(ClientConfiguration.newBuilder()
                .withEndpoints("192.168.1.10:9876")
                .build()
            )
            .withConsumerGroup("pop-consumer-group")
            .withNamespace("namespace")
            .build();

        // 订阅主题
        consumer.subscribe("OrderTopic", "*", MessageSelector.byTag("*"));

        try {
            while (true) {
                // POP 拉取消息：最多 16 条，不可见期 30 秒
                List<MessageExt> messages = consumer.receive(
                    16,                          // maxMessageNum
                    Duration.ofSeconds(30)       // invisibleDuration
                );

                for (MessageExt msg : messages) {
                    try {
                        // 处理消息
                        System.out.printf("收到消息: %s%n", new String(msg.getBody()));
                        
                        // ACK 确认（告诉 Broker 可以删除）
                        consumer.ack(msg);
                    } catch (Exception e) {
                        // 处理失败：变更不可见期，让消息再次可见
                        consumer.changeInvisibleDuration(msg, Duration.ofSeconds(10));
                    }
                }
            }
        } finally {
            consumer.close();
        }
    }
}
```

### 不可见期（Invisible Duration）机制

```
时间线：
1. T0:    Client POP 请求 → Broker 返回消息，设置不可见期 30s
2. T0~T30: 消息对当前 Client 可见，对其他 Client 不可见
3. T5:    Client 处理完 → 调用 ack()，Broker 删除消息 ✅
   或 T30: 不可见期超时 → 消息对其他 Client 再次可见（重试）
   或 T15: Client 调用 changeInvisibleDuration(60s) → 续租
```

### 续租场景

```java
// 处理耗时较长时，在不可见期结束前续租
for (MessageExt msg : messages) {
    // 预估处理需要 60 秒，续租为 90 秒
    consumer.changeInvisibleDuration(msg, Duration.ofSeconds(90));
    try {
        processLongRunningTask(msg);
        consumer.ack(msg);
    } catch (Exception e) {
        // 续租 + 重试
        consumer.changeInvisibleDuration(msg, Duration.ofSeconds(30));
    }
}
```

## 3. 批量 POP 消费

```java
// 每轮最多 POP 32 条，减少网络开销
List<MessageExt> messages = consumer.receive(32, Duration.ofSeconds(60));

// 批量 ACK
for (MessageExt msg : messages) {
    consumer.ack(msg);
}
```

## 4. 消息过滤（SQL92 过滤）

```java
// Tag 过滤
consumer.subscribe("OrderTopic", "vip||normal", MessageSelector.byTag("vip||normal"));

// SQL92 属性过滤
consumer.subscribe("OrderTopic", "*",
    MessageSelector.bySql("(TAGS is not null AND TAGS = 'vip') AND amount > 100")
);
```

## 5. 死信队列

当消息 POP 后一直未被 ACK（超过最大重试次数），会自动进入死信队列：

```java
// 监听死信队列（DLQ）
consumer.subscribe("%DLQ%pop-consumer-group", "*");
```

## 6. Spring Boot 集成 POP 模式

```java
@Component
public class PopMessageListener {

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    @PostConstruct
    public void startPopConsumer() throws Exception {
        SimpleConsumer consumer = SimpleConsumer.newBuilder()
            .withClientConfiguration(ClientConfiguration.newBuilder()
                .withEndpoints("${rocketmq.name-server}")
                .build()
            )
            .withConsumerGroup("order-pop-group")
            .build();

        consumer.subscribe("OrderTopic", "*");

        Executors.newSingleThreadExecutor().submit(() -> {
            try {
                while (true) {
                    List<MessageExt> messages = consumer.receive(16, Duration.ofSeconds(30));
                    for (MessageExt msg : messages) {
                        try {
                            handleMessage(msg);
                            consumer.ack(msg);
                        } catch (Exception e) {
                            consumer.changeInvisibleDuration(msg, Duration.ofSeconds(10));
                        }
                    }
                }
            } catch (Exception e) {
                log.error("POP 消费异常", e);
            }
        });
    }

    private void handleMessage(MessageExt msg) {
        // 业务处理
    }
}
```

## 注意事项

- **POP 模式无需再平衡**：客户端上下线不会触发 Rebalance，适合弹性伸缩场景
- **不可见期设置**：根据业务处理耗时合理设置，过短会导致频繁重试，过长会影响消息及时性
- **续租及时性**：在不可见期结束前至少 5 秒调用 changeInvisibleDuration
- **幂等消费**：POP 模式允许多次投递（不可见期超时后），消费端必须做好幂等
- **批量大小建议**：单次 POP 条数建议 8~32 条，过大增加网络延迟
- **不兼容旧版**：POP 模式需要 Broker 5.x+ 支持，4.x Broker 无法使用
- **与 Push 模式共存**：同一消费组不能混用 PushConsumer 和 SimpleConsumer
