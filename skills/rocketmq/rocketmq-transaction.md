---
name: rocketmq-transaction
description: RocketMQ 事务消息与顺序消息实战
tags: [rocketmq, mq, message, transaction]
---

## 事务消息（Transaction Message）

### 原理与流程

RocketMQ 事务消息通过**半消息（Half Message）+ 回查机制**实现分布式事务最终一致性：

```
1. Producer 发送半消息（对 Consumer 不可见）
2. Broker 持久化半消息，返回 ACK
3. Producer 执行本地事务
4. Producer 提交/回滚半消息
5a. 提交 -> Broker 投递消息给 Consumer
5b. 回滚 -> Broker 删除半消息
6. （异常时）Broker 回查 Producer 本地事务状态
```

### Spring Boot 实现

```java
// 1. 事务消息监听器
@Component
@RocketMQTransactionListener
public class OrderTransactionListener implements RocketMQLocalTransactionListener {

    @Autowired
    private OrderService orderService;

    // 执行本地事务
    @Override
    @Transactional
    public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        try {
            // 解析消息中的业务参数
            Map<String, Object> params = (Map<String, Object>) arg;
            Long orderId = (Long) params.get("orderId");

            // 执行本地事务（创建订单）
            orderService.createOrder(orderId);

            return RocketMQLocalTransactionState.COMMIT;
        } catch (Exception e) {
            log.error("本地事务执行失败", e);
            return RocketMQLocalTransactionState.ROLLBACK;
        }
    }

    // 回查（Broker 未收到 Commit/Rollback 时调用）
    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        // 从消息中提取事务ID
        String transactionId = msg.getHeaders().get(RocketMQHeaders.TRANSACTION_ID, String.class);
        Long orderId = Long.parseLong(transactionId);

        // 查询本地事务是否成功
        boolean orderExists = orderService.exists(orderId);
        if (orderExists) {
            return RocketMQLocalTransactionState.COMMIT;
        }
        // 如果不确定，返回 UNKNOWN 让 Broker 继续回查
        return RocketMQLocalTransactionState.UNKNOWN;
    }
}

// 2. 发送事务消息
@Service
public class OrderProducer {

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    public void sendOrderMessage(Long orderId, OrderDTO order) {
        Message<String> message = MessageBuilder
            .withPayload(JSON.toJSONString(order))
            .setHeader(RocketMQHeaders.TRANSACTION_ID, String.valueOf(orderId))
            .build();

        // 发送事务消息
        TransactionSendResult result = rocketMQTemplate.sendMessageInTransaction(
            "order-tx-topic",
            message,
            Map.of("orderId", orderId)    // 传递给 executeLocalTransaction 的参数
        );

        if (result.getLocalTransactionState() == RocketMQLocalTransactionState.COMMIT) {
            log.info("订单事务消息提交成功: orderId={}", orderId);
        }
    }
}

// 3. 消费端
@Component
@RocketMQMessageListener(
    topic = "order-tx-topic",
    consumerGroup = "order-consumer-group"
)
public class OrderConsumer implements RocketMQListener<String> {

    @Override
    public void onMessage(String message) {
        OrderDTO order = JSON.parseObject(message, OrderDTO.class);
        // 处理下单后的业务（积分、通知等）
        log.info("收到订单消息: {}", order.getOrderId());
    }
}
```

### 事务消息配置

```yaml
rocketmq:
  name-server: 127.0.0.1:9876
  producer:
    group: order-producer-group
    # 事务消息必须开启
    send-message-timeout: 6000
    # 事务回查最大次数（达到后回滚）
    transaction-check-interval: 30000
    transaction-check-max: 15
```

## 顺序消息（Ordered Message）

### 实现原理

RocketMQ 使用 **MessageQueueSelector** 将同一业务 ID 的消息发送到同一个 Queue，Queue 内部保证 FIFO 顺序。

```java
// 1. 顺序消息生产者
@Service
public class OrderEventProducer {

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    public void sendOrderEvent(OrderEvent event) {
        // 使用 orderId 作为选择 key，保证同一订单的消息进入同一 Queue
        rocketMQTemplate.asyncSendOrderly(
            "order-event-topic",
            event,
            String.valueOf(event.getOrderId()),
            new SendCallback() {
                @Override
                public void onSuccess(SendResult result) {
                    log.info("顺序消息发送成功: orderId={}, queueId={}",
                        event.getOrderId(), result.getMessageQueue().getQueueId());
                }

                @Override
                public void onException(Throwable e) {
                    log.error("顺序消息发送失败", e);
                }
            }
        );
    }
}

// 2. 顺序消息消费者（单线程消费）
@Component
@RocketMQMessageListener(
    topic = "order-event-topic",
    consumerGroup = "order-event-group",
    consumeMode = ConsumeMode.ORDERLY  // 顺序消费模式
)
public class OrderEventConsumer implements RocketMQListener<OrderEvent> {

    @Override
    public void onMessage(OrderEvent event) {
        // 同一 orderId 的消息在此顺序处理
        switch (event.getEventType()) {
            case "CREATED"  -> handleCreated(event);
            case "PAID"     -> handlePaid(event);
            case "SHIPPED"  -> handleShipped(event);
            case "CANCELLED" -> handleCancelled(event);
        }
    }
}
```

### 顺序消费注意事项

```yaml
# 顺序消费推荐配置
rocketmq:
  consumer:
    group: order-event-group
    # 顺序消费建议关闭批量拉取，确保单线程处理
    pull-batch-size: 1
    # 消费失败的重试间隔
    suspend-current-queue-time-millis: 3000
```

## 延时消息

```java
// 预设延时级别（1s / 5s / 10s / 30s / 1m / 2m / 3m / 4m / 5m / 6m / 7m / 8m / 9m / 10m / 20m / 30m / 1h / 2h）
// 对应级别 1~18

// 发送延时 30 分钟的消息（级别 16）
Message<String> message = MessageBuilder
    .withPayload(payload)
    .build();
SendResult result = rocketMQTemplate.syncSend(
    "delay-topic",
    message,
    3000,
    16  // 指定延时级别
);

// RocketMQ 5.x 支持自定义延时（时间戳）
// 需要 Broker 配置 enableScheduleMessageEscape=true
Message<String> msg = MessageBuilder
    .withPayload(payload)
    .setHeader("DELAY", System.currentTimeMillis() + 3600000) // 1小时后
    .build();
```

## 批量消息

```java
// 批量发送（相同 topic、相同 waitStoreMsgOK）
List<Message<String>> messages = new ArrayList<>();
for (OrderDTO order : orders) {
    messages.add(MessageBuilder.withPayload(JSON.toJSONString(order)).build());
}

// 总大小不超过 4MB（默认）
// 可通过 producer.maxMessageSize 调整
SendResult result = rocketMQTemplate.syncSend("batch-topic", messages);
```

## 注意事项

1. **事务回查必须幂等**：`checkLocalTransaction()` 可能被多次调用，必须返回一致结果
2. **事务消息不支持批量发送**：每个事务消息需要单独发送
3. **顺序消息影响性能**：队列内单线程消费，不要用来处理高并发非顺序敏感的业务
4. **顺序消息失败处理**：消费失败挂起当前队列，后续消息被阻塞直到重试成功
5. **延时消息级别限制**：5.x 之前的版本只能用预设的 18 个级别；5.x 可开启自定义延时
6. **消息去重**：消费端做好幂等，消息可能重复投递（尤其是事务消息回查场景）
7. **半消息不可见**：事务消息在半消息状态下对 Consumer 完全不可见，不会投递
