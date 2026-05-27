---
name: rocketmq-spring-boot
description: RocketMQ Spring Boot 集成与生产配置
tags: [rocketmq, mq, spring-boot, integration]
---

## 依赖配置

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.3.0</version>
</dependency>
```

```groovy
// Gradle
implementation 'org.apache.rocketmq:rocketmq-spring-boot-starter:2.3.0'
```

## 基础配置

```yaml
rocketmq:
  name-server: 192.168.1.10:9876;192.168.1.11:9876
  producer:
    group: ${spring.application.name}-producer
    send-message-timeout: 3000          # 发送超时 3s
    compress-message-body-threshold: 4096  # 4KB 以上自动压缩
    max-message-size: 4194304           # 最大消息 4MB
    retry-times-when-send-failed: 2     # 同步发送重试次数
    retry-times-when-send-async-failed: 2
  consumer:
    # 全局消费者配置（各消费者可覆盖）
    pull-batch-size: 32                 # 批量拉取消息数
```

## 消息生产者

### 同步发送（最常用）

```java
@Component
public class OrderMessageProducer {

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    /**
     * 发送同步消息
     * @param topic   主题
     * @param tag     标签（用于分类过滤）
     * @param body    消息体
     */
    public SendResult sendSync(String topic, String tag, Object body) {
        Message<Object> message = MessageBuilder
            .withPayload(body)
            .setHeader("KEYS", UUID.randomUUID().toString())  // 业务 key，用于去重
            .build();

        // 格式: topic:tag
        SendResult result = rocketMQTemplate.syncSend(
            topic + ":" + tag,
            message,
            3000     // 超时时间
        );

        log.info("消息发送完成 topic={} tag={} status={} msgId={}",
            topic, tag, result.getSendStatus(), result.getMsgId());

        return result;
    }
}
```

### 异步发送（高吞吐场景）

```java
public void sendAsync(String topic, String tag, Object body) {
    Message<Object> message = MessageBuilder.withPayload(body).build();

    rocketMQTemplate.asyncSend(
        topic + ":" + tag,
        message,
        new SendCallback() {
            @Override
            public void onSuccess(SendResult result) {
                log.info("异步发送成功 msgId={}", result.getMsgId());
            }

            @Override
            public void onException(Throwable e) {
                log.error("异步发送失败 topic={}", topic, e);
                // 补偿处理：写入本地重试表
                saveToRetryTable(topic, tag, body);
            }
        }
    );
}
```

### 延时消息

```java
// RocketMQ 支持 18 个预设延迟级别（messageDelayLevel）
// 1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
// level=3 → 10s

public SendResult sendDelay(String topic, Object body, int delayLevel) {
    Message<Object> message = MessageBuilder.withPayload(body).build();
    return rocketMQTemplate.syncSend(
        topic,
        message,
        3000,
        delayLevel    // 延迟级别
    );
}

// 使用示例：订单 30 分钟后超时取消
sendDelay("order-topic", order, 4);  // level 4 = 30s，生产环境用 10m/30m
```

### 顺序消息

```java
/**
 * 顺序消息保证同一 queue 内的消息有序
 * 使用 message queue selector 将相同业务 ID 的消息路由到同一 queue
 */
public SendResult sendOrderly(String topic, Object body, String businessKey) {
    return rocketMQTemplate.syncSendOrderly(
        topic,
        MessageBuilder.withPayload(body).build(),
        businessKey,        // 队列选择依据（如 orderId）
        3000
    );
}

// 使用场景：订单状态变更必须按顺序执行
// create → pay → ship → complete
// 相同 orderId 的消息始终进入同一 queue
```

## 消息消费者

### 广播模式

```yaml
rocketmq:
  consumer:
    group: ${spring.application.name}-consumer
    topic: order-topic
    message-model: BROADCASTING     # 每个消费者都收到全量消息
```

### 集群模式（默认）

```yaml
rocketmq:
  consumer:
    group: order-consumer-group
    topic: order-topic
    message-model: CLUSTERING       # 一条消息只被一个消费者处理
```

### 基础监听器

```java
@Component
@RocketMQMessageListener(
    topic = "order-topic",
    selectorExpression = "create",     // 只消费 tag=create 的消息
    consumerGroup = "order-create-consumer",
    consumeMode = ConsumeMode.CONCURRENTLY,   // 并发消费
    messageModel = MessageModel.CLUSTERING
)
public class OrderCreateConsumer implements RocketMQListener<OrderMessage> {

    @Override
    public void onMessage(OrderMessage message) {
        log.info("收到订单创建消息 orderId={}", message.getOrderId());
        // 业务处理
        orderService.handleCreate(message);
        // 默认返回 CONSUME_SUCCESS，自动 ACK
    }
}
```

### 顺序消费

```java
@Component
@RocketMQMessageListener(
    topic = "order-status-topic",
    consumerGroup = "order-status-consumer",
    consumeMode = ConsumeMode.ORDERLY,        // 顺序消费（同一 queue 串行）
    messageModel = MessageModel.CLUSTERING
)
public class OrderStatusConsumer implements RocketMQListener<OrderStatusMessage> {

    @Override
    public void onMessage(OrderStatusMessage message) {
        // 同一订单的消息串行处理，不会出现并发问题
        orderService.updateStatus(message.getOrderId(), message.getStatus());
    }
}
```

### 批量消费

```java
@Component
@RocketMQMessageListener(
    topic = "log-topic",
    consumerGroup = "log-consumer",
    consumeMode = ConsumeMode.CONCURRENTLY,
    pullBatchSize = 100           // 每次拉取 100 条
)
public class LogBatchConsumer implements RocketMQListener<List<LogMessage>> {

    @Override
    public void onMessage(List<LogMessage> messages) {
        // 批量写入 ES/ClickHouse
        logService.batchInsert(messages.stream()
            .map(LogMessage::getPayload)
            .collect(Collectors.toList()));
    }
}
```

## 自定义线程池

```java
@Bean
public ThreadPoolExecutor messageThreadPool() {
    return new ThreadPoolExecutor(
        10,                    // corePoolSize
        20,                    // maxPoolSize
        60, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(2000),
        new ThreadFactoryBuilder()
            .setNameFormat("rocketmq-consumer-%d")
            .build(),
        new ThreadPoolExecutor.CallerRunsPolicy()
    );
}

// 在消费者中注入
@Component
@RocketMQMessageListener(
    topic = "order-topic",
    consumerGroup = "order-consumer",
    consumeMode = ConsumeMode.CONCURRENTLY,
    consumeThreadNumber = 10
)
public class CustomThreadConsumer implements RocketMQListener<OrderMessage> {

    @Autowired
    private ThreadPoolExecutor executor;

    @Override
    public void onMessage(OrderMessage message) {
        executor.submit(() -> {
            try {
                // 异步处理
                processOrder(message);
            } catch (Exception e) {
                log.error("处理异常", e);
            }
        });
    }
}
```

## 事务消息

```java
@Component
@RocketMQTransactionListener
public class OrderTransactionListener implements RocketMQLocalTransactionListener {

    @Autowired
    private OrderMapper orderMapper;

    /**
     * 执行本地事务（半消息发送成功后回调）
     */
    @Override
    @Transactional
    public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        try {
            Order order = (Order) arg;
            orderMapper.insert(order);               // 本地事务
            return RocketMQLocalTransactionState.COMMIT;  // 提交半消息
        } catch (Exception e) {
            log.error("本地事务失败，回滚消息", e);
            return RocketMQLocalTransactionState.ROLLBACK;
        }
    }

    /**
     * 回查本地事务状态（Broker 未收到提交/回滚时触发）
     */
    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        String orderId = (String) msg.getHeaders().get("order_id");
        Order order = orderMapper.selectById(orderId);
        if (order != null) {
            return RocketMQLocalTransactionState.COMMIT;
        }
        return RocketMQLocalTransactionState.UNKNOWN;  // 继续回查
    }
}

// 发送事务消息
public void sendTxMessage(Order order) {
    Message<Order> message = MessageBuilder
        .withPayload(order)
        .setHeader("order_id", order.getId().toString())
        .build();

    TransactionSendResult result = rocketMQTemplate.sendMessageInTransaction(
        "order-tx-topic",
        message,
        order                 // 传递给 executeLocalTransaction 的 arg
    );
    log.info("事务消息发送结果: {}", result.getSendStatus());
}
```

## 消息轨迹

```yaml
rocketmq:
  producer:
    enable-msg-trace: true
    customized-trace-topic: RMQ_TRACE_TOPIC
  consumer:
    enable-msg-trace: true
```

## 生产配置优化

```yaml
rocketmq:
  producer:
    # 生产发送优化
    send-message-timeout: 5000
    compress-message-body-threshold: 2048   # 2KB 开始压缩
    retry-times-when-send-failed: 3
    retry-times-when-send-async-failed: 3
  consumer:
    # 消费优化
    pull-interval: 20                        # 拉取间隔（默认 0，建议 20ms）
    pull-batch-size: 32
    consume-thread-min: 20
    consume-thread-max: 64
```

## 注意事项

1. **消息幂等性**：消息可能重复投递，消费端必须做幂等（业务 key 去重 + 状态机校验）
2. **Tag 不要过多**：一个 Topic 的 Tag 建议控制在 10 个以内，过多的 Tag 影响性能
3. **消息体大小**：建议不超过 1MB，大消息用 OSS 存储，消息中只传文件链接
4. **Consumer Group 唯一性**：同一 Topic 的不同消费逻辑必须使用不同 Group 名
5. **重试队列**：消费失败的消息会进入 `%RETRY%{consumerGroup}` 重试队列，默认重试 16 次后进入死信队列
6. **死信队列**：`%DLQ%{consumerGroup}` 中人工兜底处理，不要自动消费
7. **NameServer 高可用**：至少部署 2 个 NameServer（无状态，可前置 Nginx 或直连）
8. **Broker 配置**：`messageDelayLevel` 自定义延迟级别需重启 Broker
9. **Spring Boot 版本兼容**：rocketmq-spring-boot-starter 2.3.0 兼容 Spring Boot 3.x，2.2.x 兼容 2.x
10. **不要在生产打开 autoCreateTopicEnable**：Topic 提前创建，避免动态创建带来的性能开销
