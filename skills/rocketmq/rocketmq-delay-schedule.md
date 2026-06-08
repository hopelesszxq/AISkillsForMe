---
name: rocketmq-delay-schedule
description: RocketMQ 延时消息与定时投递——延迟级别、自定义延时与定时任务模式
tags: [rocketmq, mq, messaging, delay, schedule, timer]
---

## 概述

RocketMQ 支持**延时消息**（延迟一定时间后被消费）和**定时消息**（指定精确时间点投递）。本文覆盖延迟级别机制、自定义延迟时间、结合业务场景的最佳实践与常见陷阱。

## 一、延时消息（基础模式）

### 1.1 默认延迟级别

RocketMQ 内置 18 个固定延迟级别（messageDelayLevel），不支持任意秒级延迟：

```properties
# broker.conf 默认值
messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
```

| Level | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|-------|---|---|---|---|---|---|---|---|---|
| Time  | 1s | 5s | 10s | 30s | 1m | 2m | 3m | 4m | 5m |
| Level | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 |
| Time  | 6m | 7m | 8m | 9m | 10m | 20m | 30m | 1h | 2h |

### 1.2 发送延时消息

```java
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.common.message.Message;

DefaultMQProducer producer = new DefaultMQProducer("delay-producer-group");
producer.setNamesrvAddr("127.0.0.1:9876");
producer.start();

Message msg = new Message("OrderTopic", "Cancel", "订单30分钟未支付自动取消".getBytes());
// 设置延迟级别 5 → 1 分钟
msg.setDelayTimeLevel(5);

// 或使用 RocketMQ 5.x 客户端 API（更直观）
// msg.setDelayTimeSeconds(60);

SendResult result = producer.send(msg);
System.out.printf("msgId=%s%n", result.getMsgId());
producer.shutdown();
```

### 1.3 Spring Boot 集成

```java
@Service
@RequiredArgsConstructor
public class OrderTimeoutService {

    private final RocketMQTemplate rocketMQTemplate;

    /**
     * 创建订单后发送延时消息，到期检查支付状态
     */
    public void scheduleOrderTimeout(String orderId) {
        OrderTimeoutMessage msg = new OrderTimeoutMessage(orderId, System.currentTimeMillis());
        // 使用 MessageBuilder 设置延迟级别（1分钟 = level 5）
        rocketMQTemplate.syncSend(
            "OrderTimeoutTopic",
            MessageBuilder.withPayload(msg).build(),
            3000,
            5  // delayLevel
        );
    }
}

// 消费者
@Component
@RocketMQMessageListener(
    topic = "OrderTimeoutTopic",
    consumerGroup = "order-timeout-group"
)
public class OrderTimeoutConsumer implements RocketMQListener<OrderTimeoutMessage> {
    @Override
    public void onMessage(OrderTimeoutMessage message) {
        // 检查订单状态，如果仍为 UNPAID 则自动取消
        log.info("订单超时检查：{}", message.getOrderId());
        orderService.cancelIfUnpaid(message.getOrderId());
    }
}
```

## 二、定时消息（精确时间点投递，RocketMQ 5.x+）

RocketMQ 5.x 支持指定精确时间戳的定时消息（基于 Timer 模块），不再受固定延迟级别限制。

### 2.1 启用 Timer 模块

```properties
# broker.conf
timerEnabled=true
timerPrecisionMs=1000           # 定时精度（毫秒，默认1000）
timerFlushIntervalMs=1000       # 刷盘间隔
timerSkipUnknown=true           # 跳过未知定时消息
timerMetricsEnabled=true        # 开启指标
```

### 2.2 发送精确时间消息（5.x Java 客户端）

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client-java</artifactId>
    <version>5.0.7</version>
</dependency>
```

```java
import org.apache.rocketmq.client.java.producer.ProducerBuilder;
import org.apache.rocketmq.client.java.message.MessageBuilder;

ClientServiceProvider provider = ClientServiceProvider.loadService();
MessageBuilder msgBuilder = provider.newMessageBuilder()
    .setTopic("ScheduledTaskTopic")
    .setKeys("task-001")
    .setBody("定时任务触发".getBytes());

// 设置 30 分钟后投递
long deliverTimestamp = System.currentTimeMillis() + 30 * 60 * 1000;
Message message = msgBuilder.setDeliveryTimestamp(deliverTimestamp).build();

// 发送
Producer producer = ProducerBuilder.build();
SendReceipt receipt = producer.send(message);
System.out.printf("delay msgId=%s, deliverAt=%d%n",
    receipt.getMessageId(), deliverTimestamp);
producer.close();
```

### 2.3 Spring Boot 定时消息

```java
// RocketMQ 5.x Spring Boot Starter 发送定时消息
@Autowired
private RocketMQTemplate rocketMQTemplate;

public void scheduleTask(String taskId, long executeAt) {
    // RocketMQ 5.x Spring Boot 通过 Header 设置投递时间
    Message<byte[]> message = MessageBuilder.withPayload(taskId.getBytes())
        .setHeader("DELIVERY_TIMESTAMP", String.valueOf(executeAt))
        .build();

    rocketMQTemplate.syncSend("ScheduledTaskTopic", message);
}
```

## 三、自定义延迟级别

如果默认 18 级不够用，可修改 Broker 配置：

```properties
# broker.conf
messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h 3h 6h 12h 24h
```

修改后重启 Broker 生效。**注意**：不能减少原有级别数量（最多新增），否则已存储的消息延迟级别可能错乱。

## 四、应用场景与模式

### 4.1 订单超时自动取消

```java
// 下单时发送延时消息
public void createOrder(Order order) {
    // 1. 创建订单
    orderMapper.insert(order);

    // 2. 发送延时 30 分钟的消息
    Message msg = new Message("OrderDelay", order.getId().getBytes());
    msg.setDelayTimeLevel(15);  // 30 分钟
    producer.send(msg);
}

// 消费者处理超时
public void onTimeout(String orderId) {
    Order order = orderMapper.selectById(orderId);
    if (order != null && "UNPAID".equals(order.getStatus())) {
        order.setStatus("CANCELLED");
        orderMapper.updateById(order);
        // 释放库存
        inventoryService.releaseStock(order.getSku(), order.getQuantity());
    }
}
```

### 4.2 定时任务分发（替代 Quartz）

```java
// 每天早上 8:00 发送日报生成任务
public void dispatchDailyReport() {
    long next8am = nextTimeOfDay(8, 0, 0);
    Message msg = new Message("DailyReportTopic", "生成日报".getBytes());
    msg.setDeliveryTimestamp(next8am);
    producer.send(msg);
}

// 延时重试（退避策略）
public void retryWithBackoff(String bizKey, int retryCount) {
    // 第1次重试：30s；第2次：1min；第3次：2min...
    int[] delays = {0, 30, 60, 120, 300, 600};
    if (retryCount < delays.length) {
        int delaySeconds = delays[retryCount];
        scheduleWithDelay(bizKey, delaySeconds);
    }
}
```

### 4.3 消息延时重试（与死信搭配）

```java
@Component
@RocketMQMessageListener(topic = "RetryTopic", consumerGroup = "retry-group")
public class RetryConsumer implements RocketMQListener<RetryMessage> {
    @Override
    public void onMessage(RetryMessage msg) {
        try {
            process(msg.getPayload());
        } catch (RetryableException e) {
            if (msg.getRetryCount() < 3) {
                // 写入延时重试队列，10 分钟后重试
                msg.setRetryCount(msg.getRetryCount() + 1);
                rocketMQTemplate.syncSend(
                    "RetryTopic",
                    MessageBuilder.withPayload(msg).build(),
                    3000,
                    7  // 延时级别 7 = 4分钟
                );
            } else {
                // 超过重试次数 → 死信
                rocketMQTemplate.syncSend("RetryDLQ", msg);
            }
        }
    }
}
```

## 五、注意事项

1. **延迟级别限制**：默认只有 18 个固定级别，不支持任意秒级延迟。5.x Timer 模块可解此限制。
2. **写入量放大**：延时消息在 Broker 内部先写入 SCHEDULE_TOPIC_XXXX，到期后再写入真实 Topic，写入量翻倍。
3. **积压风险**：大量延时消息写入但未到期时，SCHEDULE_TOPIC 会持续膨胀，需监控延时队列积压量。
4. **可观测性缺失**：延时消息的到期时间不会写入消息轨迹，排查问题时需通过 `deliverTimestamp` 字段自行计算。
5. **Timer 模块需显式开启**：5.x 的 Timer 功能默认关闭（`timerEnabled=false`），需手动配置并重启 Broker。
6. **精度取舍**：默认 Timer 精度为 1000ms，如需亚秒级精度可调低 `timerPrecisionMs`，但会增加 CPU 和磁盘 I/O。
7. **消息大小**：延时消息不支持超大消息（默认 4MB），超大会在 SCHEDULE_TOPIC 写入时被拦截。
