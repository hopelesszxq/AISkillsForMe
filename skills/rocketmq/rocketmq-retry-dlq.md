---
name: rocketmq-retry-dlq
description: RocketMQ 消息重试机制与死信队列（DLQ）深度实践：重试策略、重试队列、死信处理、监控告警
tags: [rocketmq, mq, messaging, retry, dlq, error-handling, monitoring]
---

## 概述

RocketMQ 的 **消息重试** 和 **死信队列（DLQ，Dead Letter Queue）** 是保证消息可靠投递的核心机制。当消费者消费消息失败时，RocketMQ 会自动重试；超过最大重试次数后，消息进入死信队列。本文覆盖重试原理、重试策略配置、死信处理方案、监控告警等最佳实践。

## 1. 重试机制原理

### 1.1 消费失败判定

```java
// 消费者监听器返回以下状态表示消费失败
import org.apache.rocketmq.spring.annotation.RocketMQMessageListener;
import org.apache.rocketmq.spring.core.RocketMQListener;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;

@Component
@RocketMQMessageListener(topic = "order-topic", consumerGroup = "order-group")
public class OrderConsumer implements RocketMQListener<OrderMessage> {

    @Override
    public void onMessage(OrderMessage message) {
        try {
            // 处理业务逻辑
            processOrder(message);
            // 默认返回 SUCCESS（无需显式返回）
        } catch (Exception e) {
            // 抛出异常或返回 RECONSUME_LATER 触发重试
            throw new RuntimeException("处理失败，触发重试", e);
            // 或：return ConsumeConcurrentlyStatus.RECONSUME_LATER;
        }
    }
}
```

**重试触发条件**：
- 消费者抛出任何异常
- 返回 `ConsumeConcurrentlyStatus.RECONSUME_LATER`
- 消费超时（默认 15 分钟）

### 1.2 重试间隔（阶梯式退避）

RocketMQ 的重试间隔是 **递增的阶梯式延迟**，而不是固定间隔。默认重试时间表（共 16 级）：

| 重试次数 | 时间间隔  | 重试次数 | 时间间隔 |
|----------|-----------|----------|----------|
| 1        | 10 秒     | 9        | 7 分钟   |
| 2        | 30 秒     | 10       | 8 分钟   |
| 3        | 1 分钟    | 11       | 9 分钟   |
| 4        | 2 分钟    | 12       | 10 分钟  |
| 5        | 3 分钟    | 13       | 20 分钟  |
| 6        | 4 分钟    | 14       | 30 分钟  |
| 7        | 5 分钟    | 15       | 1 小时   |
| 8        | 6 分钟    | 16       | 2 小时   |

**最大重试次数**：默认 16 次（从 0 开始计数），即总共尝试 16 次后进入死信队列。

### 1.3 重试队列结构

```markdown
%RETRY%{consumerGroup}  →  重试队列（每个消费者组一个）
%DLQ%{consumerGroup}    →  死信队列（每个消费者组一个）
```

例如，消费者组 `order-group` 的重试队列为 `%RETY%order-group`，死信队列为 `%DLQ%order-group`。

## 2. 重试策略配置

### 2.1 修改最大重试次数

```java
@Component
@RocketMQMessageListener(
    topic = "order-topic",
    consumerGroup = "order-group",
    maxReconsumeTimes = 5  // 最多重试 5 次（第 6 次进 DLQ）
)
public class OrderConsumer implements RocketMQListener<OrderMessage> {
    // ...
}
```

**推荐值**：
- 临时性故障（网络抖动）：3-5 次即可
- 业务依赖外部系统：8-16 次
- 关键业务：16 次（默认）

### 2.2 自定义延迟级别

```java
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyContext;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.common.message.MessageExt;

// 使用 MessageListenerConcurrently 自定义延迟
@Component
public class CustomDelayConsumer implements MessageListenerConcurrently {

    @Override
    public ConsumeConcurrentlyStatus consumeMessage(
            List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        
        MessageExt msg = msgs.get(0);
        int reconsumeTimes = msg.getReconsumeTimes();
        
        if (reconsumeTimes <= 3) {
            // 前 3 次重试使用标准延迟
            return ConsumeConcurrentlyStatus.RECONSUME_LATER;
        }
        
        // 第 4 次起，等待 30 分钟后重试
        context.setDelayLevelWhenNextConsume(
            broker.GetDelayTimeLevel(30 * 60 * 1000) // 自定义延迟
        );
        return ConsumeConcurrentlyStatus.RECONSUME_LATER;
    }
}
```

### 2.3 服务端配置

```properties
# broker.conf — 全局重试配置
# 单个消息最大重试次数（默认 16）
maxReconsumeTimes=16

# 是否异步落盘重试队列
enableAsyncReput=false

# 重试消息延迟级别
messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
```

## 3. 死信队列处理

### 3.1 死信队列特点

- 死信队列名：`%DLQ%{consumerGroup}`
- 一旦消息进入 DLQ，**消费者组不再自动消费**，需手动处理
- DLQ 消息保留时间与普通消息相同（默认 48 小时）
- 每个消费者组独立拥有自己的 DLQ

### 3.2 监控 DLQ 消息

```java
// 使用 Admin API 监控 DLQ 深度
import org.apache.rocketmq.tools.admin.DefaultMQAdminExt;

public class DlqMonitor {

    public long getDlqCount(String nameSrvAddr, String consumerGroup) throws Exception {
        DefaultMQAdminExt admin = new DefaultMQAdminExt();
        admin.setNamesrvAddr(nameSrvAddr);
        admin.start();

        try {
            String dlqTopic = "%DLQ%" + consumerGroup;
            // 获取队列统计
            TopicStatsTable stats = admin.examineTopicStats(dlqTopic);
            long total = 0;
            for (QueueStats qs : stats.getOffsetTable().values()) {
                total += qs.getMaxOffset() - qs.getMinOffset();
            }
            return total;
        } finally {
            admin.shutdown();
        }
    }
}
```

### 3.3 DLQ 消息查询（RocketMQ Dashboard）

```bash
# 通过 MQAdmin 工具查询 DLQ
sh bin/mqadmin queryMsgByKey -n localhost:9876 -t %DLQ%order-group -k orderId123
```

通过 RocketMQ Dashboard（控制台）可直观查看：
1. 进入 DLQ 页面：`运维 → DLQ 管理`
2. 查看队列深度和消息内容
3. 支持按 Key 和时间范围搜索

### 3.4 手动重新投递 DLQ 消息

```java
// 将 DLQ 消息重新投递到原始 Topic 或某个修复 Topic
import org.apache.rocketmq.common.message.Message;

public void redeliverDlqMessage(
        DefaultMQProducer producer, 
        MessageExt dlqMsg, 
        String originalTopic) throws Exception {
    
    Message newMsg = new Message(
        originalTopic,
        dlqMsg.getTags(),
        dlqMsg.getKeys(),
        dlqMsg.getBody()
    );
    newMsg.setDelayTimeLevel(0); // 不延迟
    SendResult result = producer.send(newMsg);
    
    if (result.getSendStatus() == SendStatus.SEND_OK) {
        System.out.println("DLQ 消息重新投递成功，原始 MsgId: " + dlqMsg.getMsgId());
    }
}
```

### 3.5 DLQ 自动化处理（重试死信 + 告警）

```java
@Component
@RocketMQMessageListener(
    topic = "%DLQ%order-group",
    consumerGroup = "dlq-order-group"  // 专用消费者组处理 DLQ
)
public class DlqHandler implements RocketMQListener<MessageExt> {

    @Autowired
    private OrderService orderService;

    @Override
    public void onMessage(MessageExt msg) {
        String body = new String(msg.getBody(), StandardCharsets.UTF_8);
        String msgId = msg.getMsgId();
        int reconsumeTimes = msg.getReconsumeTimes();

        log.warn("收到死信消息: msgId={}, reconsumeTimes={}, body={}", 
            msgId, reconsumeTimes, body);

        // 1. 尝试最后一次处理
        try {
            orderService.processOrder(JSON.parseObject(body, OrderMessage.class));
            log.info("DLQ 消息最终处理成功: msgId={}", msgId);
            return; // 消费成功，从 DLQ 移除
        } catch (Exception e) {
            log.error("DLQ 消息处理仍失败: msgId={}", msgId, e);
        }

        // 2. 记录到数据库进行人工干预
        saveToErrorLog(msgId, body, reconsumeTimes);

        // 3. 发送告警通知
        alertService.sendAlert(
            AlertLevel.WARN,
            "订单消息进入死信队列",
            String.format("msgId=%s, group=order-group, body=%s", msgId, body)
        );
    }

    private void saveToErrorLog(String msgId, String body, int retries) {
        // 持久化到 error_message 表，供运维人员后台查看和重新投递
    }
}
```

## 4. 最佳实践

### 4.1 区分可重试和不可重试异常

```java
@Override
public void onMessage(OrderMessage message) {
    try {
        processOrder(message);
    } catch (ValidationException e) {
        // ❌ 参数校验失败：即使重试也无法成功，直接记录日志，不重试
        log.error("订单参数校验失败，不进行重试: {}", e.getMessage());
        // 直接返回 SUCCESS 消费掉这条消息，或记录到错误表
    } catch (NetworkTimeoutException e) {
        // ✅ 网络超时：可能是临时故障，应该重试
        throw new RuntimeException("网络超时，触发重试", e);
    } catch (BusinessException e) {
        // ❌ 业务逻辑异常（如余额不足）：根据具体错误码决定是否重试
        if (e.isRetryable()) {
            throw new RuntimeException("可重试业务异常", e);
        }
        log.error("不可重试业务异常: {}", e.getMessage());
    }
}
```

### 4.2 幂等消费

消息重试可能导致**重复消费**，消费端必须幂等：

```java
@Override
public void onMessage(OrderMessage message) {
    // 1. 先检查是否已处理（去重表）
    if (idempotentService.isProcessed(message.getOrderId())) {
        log.info("消息已处理，跳过: orderId={}", message.getOrderId());
        return;
    }

    // 2. 处理业务
    processOrder(message);

    // 3. 记录处理记录
    idempotentService.markProcessed(message.getOrderId());
}
```

### 4.3 设置合理的消费超时

```yaml
# application.yml
rocketmq:
  consumer:
    # 消费超时时间（毫秒），超过此时间未返回结果视为消费失败
    consume-timeout: 300000  # 5 分钟，默认 15 分钟
```

### 4.4 重试次数监控告警

```bash
# Prometheus + Grafana 监控重试队列和 DLQ
# RocketMQ Exporter 会暴露以下指标：
rocketmq_broker_retry_queue_size{topic="%RETRY%order-group"}
rocketmq_broker_dlq_queue_size{topic="%DLQ%order-group"}

# 告警规则
# - 重试队列深度 > 1000 持续 5 分钟 → Warning
# - DLQ 深度 > 0 → Critical（死信消息需要处理）
```

## 5. 注意事项

### 5.1 顺序消息没有重试

**顺序消息**在消费失败后**不会自动重试**，而是继续消费下一条消息。顺序消息重试需自行实现：

```java
// 顺序消息的消费失败处理
@Component
@RocketMQMessageListener(
    topic = "order-topic",
    consumerGroup = "order-group",
    consumeMode = ConsumeMode.ORDERLY
)
public class OrderlyConsumer implements RocketMQListener<OrderMessage> {

    @Override
    public void onMessage(OrderMessage message) {
        try {
            processOrder(message);
        } catch (Exception e) {
            // 顺序消息不会重试！需要自行处理
            // 方案1：记录到错误表，人工介入
            // 方案2：发送到单独的重试 Topic 异步处理
            log.error("顺序消息处理失败: {}", message.getOrderId(), e);
            saveToErrorLog(message);
        }
    }
}
```

### 5.2 重试消息的存储

- 重试消息存在 Broker 的 `ConsumeQueue` 中，不会删除原始消息
- 每重试一次，消息的 `ReconsumeTimes` 加 1
- 可通过消息属性 `RECONSUME_TIME` 查看已重试次数

### 5.3 预防 DLQ 堆积

- 设置合理的告警阈值：DLQ 深度 > 0 即告警
- 定期巡检 DLQ，对长期堆积的消息进行人工干预
- 考虑 DLQ 消息自动归档（超过 7 天自动清理）
