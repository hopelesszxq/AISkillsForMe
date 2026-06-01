---
name: rocketmq-batch-filter
description: RocketMQ 批量消息发送与消费、Tag/SQL92 消息过滤、广播集群模式对比
tags: [rocketmq, batch, filter, consumer]
---

## 概述

RocketMQ 支持批量消息、消息过滤（Tag 和 SQL92 表达式）以及多种消费模式。本文深入实际配置和代码示例，覆盖高性能场景下的最佳实践。

## 1. 批量消息发送

批量发送能大幅提升吞吐量，但需注意约束条件。

### 生产者配置

```java
// 依赖 org.apache.rocketmq:rocketmq-client:5.4.0+
DefaultMQProducer producer = new DefaultMQProducer("batch-producer-group");
producer.setNamesrvAddr("127.0.0.1:9876");
producer.start();

// 构建批量消息
List<Message> messages = new ArrayList<>();
for (int i = 0; i < 100; i++) {
    Message msg = new Message(
        "BatchTopic",                     // topic
        "TagA",                           // tag
        ("Hello RocketMQ Batch " + i).getBytes(StandardCharsets.UTF_8)
    );
    messages.add(msg);
}

// 发送批次
SendResult sendResult = producer.send(messages);
System.out.printf("Send Status: %s, MessageQueue: %s%n",
    sendResult.getSendStatus(), sendResult.getMessageQueue());
```

### 批量发送限制

| 约束 | 说明 |
|---|---|
| **总大小** | 不超过 4MB（DefaultMQProducer maxMessageSize，默认 4MB） |
| **拆分处理** | 超限时需手动拆分，批量消息不支持 `send` 自动拆分 |
| **相同 Topic** | 同一批消息必须属于同一个 Topic |
| **等待确认** | 批量只支持同步发送，不支持 oneway 模式 |

### 批量消息拆分工具

```java
// RocketMQ 内置拆分器
ListSplitter splitter = new ListSplitter(messages);
while (splitter.hasNext()) {
    List<Message> subList = splitter.next();
    producer.send(subList);
}

// 拆分器实现
class ListSplitter implements Iterator<List<Message>> {
    private static final int SIZE_LIMIT = 1024 * 1024 * 4; // 4MB
    private final List<Message> messages;
    private int currIndex;

    public ListSplitter(List<Message> messages) {
        this.messages = messages;
    }

    @Override
    public boolean hasNext() {
        return currIndex < messages.size();
    }

    @Override
    public List<Message> next() {
        int nextIndex = currIndex;
        int totalSize = 0;
        for (; nextIndex < messages.size(); nextIndex++) {
            Message msg = messages.get(nextIndex);
            int msgSize = msg.getTopic().length() + msg.getBody().length;
            Map<String, String> properties = msg.getProperties();
            if (properties != null) {
                for (Map.Entry<String, String> entry : properties.entrySet()) {
                    msgSize += entry.getKey().length() + entry.getValue().length();
                }
            }
            msgSize += 20; // 额外开销
            if (totalSize + msgSize > SIZE_LIMIT) break;
            totalSize += msgSize;
        }
        List<Message> subList = messages.subList(currIndex, nextIndex);
        currIndex = nextIndex;
        return subList;
    }
}
```

## 2. 消息过滤

### Tag 过滤（简单高效）

```java
// 生产者 —— 设置 Tag
Message msg1 = new Message("FilterTopic", "TagA", "msg with TagA".getBytes());
Message msg2 = new Message("FilterTopic", "TagB", "msg with TagB".getBytes());
Message msg3 = new Message("FilterTopic", "TagC", "msg with TagC".getBytes());

producer.send(msg1);
producer.send(msg2);
producer.send(msg3);

// 消费者 —— 按 Tag 订阅（支持 || 表达式）
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("filter-consumer-group");
consumer.subscribe("FilterTopic", "TagA || TagC"); // 只消费 TagA 和 TagC
// 或消费全部：consumer.subscribe("FilterTopic", "*");
```

### SQL92 过滤（属性过滤）

生产者通过 `putUserProperty` 设置自定义属性，消费者通过 SQL 表达式过滤。

```java
// 生产者 —— 设置属性
Message msg = new Message("SqlFilterTopic", "TagA", "order msg".getBytes());
msg.putUserProperty("age", "25");
msg.putUserProperty("amount", "199.00");
msg.putUserProperty("isVIP", "true");
producer.send(msg);
```

```java
// 消费者 —— SQL92 过滤（需 Broker 开启 enablePropertyFilter=true）
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("sql-filter-group");
consumer.subscribe("SqlFilterTopic", MessageSelector.bySql(
    "(TAGS is not null and TAGS in ('TagA', 'TagB')) " +
    "and age between 18 and 60 " +
    "and isVIP = true " +
    "and amount > 100"
));
consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {
    for (MessageExt msg : msgs) {
        System.out.printf("Filtered msg: %s %n", new String(msg.getBody()));
    }
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
});
consumer.start();
```

#### SQL92 支持的操作符

| 操作符 | 示例 | 说明 |
|---|---|---|
| `=`, `<>`, `<`, `>`, `<=`, `>=` | `age > 18` | 数值比较 |
| `BETWEEN ... AND ...` | `age BETWEEN 18 AND 60` | 范围匹配 |
| `IN (...)`, `NOT IN (...)` | `city IN ('BJ','SH')` | 集合匹配 |
| `IS NULL`, `IS NOT NULL` | `TAGS IS NOT NULL` | 空值判断 |
| `AND`, `OR`, `NOT` | `age > 18 AND isVIP = true` | 逻辑组合 |

### 过滤方式对比

| 方式 | 性能 | 灵活性 | 配置要求 |
|---|---|---|---|
| **Tag 过滤** | 极高（Broker 端直接比较） | 低（只能按 Tag 值） | 无需额外配置 |
| **SQL92 过滤** | 中等（需要解析表达式） | 高（支持任意属性） | `enablePropertyFilter=true` |

## 3. 消费模式

### 集群消费（CLUSTERING）

```java
// 默认模式，组内各消费者分摊消息
consumer.setMessageModel(MessageModel.CLUSTERING);
// 特点：一条消息只被组内一个消费者处理，适合负载均衡
```

| 特性 | 说明 |
|---|---|
| 消息路由 | 组内轮询分配（默认），也可指定 Queue |
| 重平衡 | 消费者增减时自动 Rebalance |
| 适用场景 | 微服务、任务处理、日志采集 |

### 广播消费（BROADCASTING）

```java
// 每条消息发给组内所有消费者
consumer.setMessageModel(MessageModel.BROADCASTING);

// 注意：广播模式下不再维护消费进度（不持久化 Offset）
// 重启后会重新消费
```

| 特性 | 说明 |
|---|---|
| 消息路由 | 每个消费者都收到完整消息流 |
| Offset | 本地存储（非持久化），重启需指定从何处消费 |
| 适用场景 | 配置下发、缓存刷新、全局通知 |

### 重平衡监听

```java
consumer.setAllocateMessageQueueStrategy(new AllocateMessageQueueAveragelyByCircle());
// 其他策略：AllocateMessageQueueAveragely（默认）、AllocateByConfig、AllocateMachineRoomNearby
```

## 4. 批量消费设置

```java
// 消费者端批量拉取参数
Properties props = new Properties();
props.setProperty(ConsumeParams.MAX_MESSAGE_SIZE, "10");    // 一次消费最大消息数
props.setProperty(ConsumeParams.CONSUME_TIMEOUT, "600000"); // 消费超时时间

// PushConsumer 批量消费
consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {
    System.out.printf("Batch received %d messages%n", msgs.size());
    for (MessageExt msg : msgs) {
        // 批量处理
    }
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
});
```

## 注意事项

1. **批量发送大小**：单批次严格控制在 4MB 以内（含消息属性 overhead），超限报 `MQClientException`
2. **SQL92 过滤性能**：表达式复杂度影响 Broker 过滤性能，避免大量消息使用复杂的嵌套逻辑
3. **Tag 设计规范**：单个 Tag 不宜过多（建议 < 100），Tag 字符串长度建议 <= 128 字符
4. **广播模式重启**：广播模式 Offset 存储于本地文件，K8s 环境中 Pod 重建会导致重复消费
5. **批量消费幂等**：消费者端必须实现幂等处理，因为重平衡会导致消息重复投递
6. **RocketMQ 5.x 建议**：新项目优先使用 Pop 消费模式（`rocketmq-pop-consumption.md`），替代传统 Pull 模式
