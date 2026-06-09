---
name: rocketmq-transaction-message
description: RocketMQ 事务消息——半消息机制、本地事务表、消息幂等性与分布式事务最终一致性实战
tags: [rocketmq, mq, messaging, transaction, distributed-transaction, tcc]
---

## 概述

RocketMQ 的事务消息通过**半消息（Half Message）** + **本地事务回查**机制，实现了分布式场景下消息投递与本地事务的最终一致性。适合订单创建、账户扣款、积分发放等场景。

## 一、事务消息原理

```
Producer                    Broker                       Consumer
   │                          │                            │
   │── 1. 发送半消息 ────────→│ (半消息对消费者不可见)     │
   │                          │                            │
   │←── 2. 半消息写入成功 ────│                            │
   │                          │                            │
   │── 3. 执行本地事务 ───────│                            │
   │                          │                            │
   │── 4. COMMIT/ROLLBACK ───→│                            │
   │                          │ (COMMIT → 消息可见)        │
   │                          │ (ROLLBACK → 删除消息)      │
   │                          │── 5. 消息投递 ────────────→│
   │                          │                            │
   │   (若步骤4超时或丢失)     │                            │
   │←── 6. 回查事务状态 ──────│                            │
   │── 7. 回查结果(COMMIT/ROLLBACK) →│                     │
```

## 二、事务消息生产者

### 2.1 实现 `TransactionListener`

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class OrderTransactionListener implements TransactionListener {

    private final OrderService orderService;

    /**
     * 执行本地事务
     * @param msg 半消息
     * @param arg 发送时传入的业务参数
     * @return 本地事务执行结果
     */
    @Override
    @Transactional
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        try {
            // 1. 从消息体中解析业务数据
            OrderCreateMessage orderMsg = JsonUtil.parse(
                new String(msg.getBody(), StandardCharsets.UTF_8),
                OrderCreateMessage.class
            );

            // 2. 执行本地事务（创建订单 + 扣减库存）
            orderService.createOrder(orderMsg);

            // 3. 返回 COMMIT，通知 Broker 让消息可见
            return LocalTransactionState.COMMIT_MESSAGE;

        } catch (DuplicateOrderException e) {
            // 幂等处理：重复消息当作成功
            log.warn("Duplicate order message, commit anyway: {}", e.getMessage());
            return LocalTransactionState.COMMIT_MESSAGE;

        } catch (Exception e) {
            log.error("Local transaction failed, will rollback message", e);
            // 返回 ROLLBACK，Broker 会删除半消息
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }
    }

    /**
     * 回查本地事务状态
     * 当 Broker 未收到 COMMIT/ROLLBACK 时（服务宕机或超时），会主动回查
     */
    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        String transactionId = msg.getTransactionId();
        String orderId = msg.getKeys(); // 业务主键

        // 检查本地事务记录表中的状态
        TransactionStatus status = orderService.getTransactionStatus(transactionId);

        return switch (status) {
            case COMMIT -> LocalTransactionState.COMMIT_MESSAGE;
            case ROLLBACK -> LocalTransactionState.ROLLBACK_MESSAGE;
            case UNKNOWN -> {
                log.warn("Transaction status unknown, will retry later: txId={}", transactionId);
                yield LocalTransactionState.UNKNOW; // 下次继续回查
            }
        };
    }
}
```

### 2.2 发送事务消息

```java
@Service
@RequiredArgsConstructor
public class OrderProducer {

    private final RocketMQTemplate rocketMQTemplate;
    private final TransactionStatusService txStatusService;

    /**
     * 发送事务消息创建订单
     */
    public void createOrderTransaction(OrderCreateRequest request) {
        String orderId = request.getOrderId();
        String transactionId = generateTransactionId();

        // 1. 生成事务消息
        Message<byte[]> message = MessageBuilder.withPayload(
            JsonUtil.toJsonBytes(request)
        ).build();

        // 2. 发送事务消息（半消息）
        TransactionSendResult result = rocketMQTemplate.sendMessageInTransaction(
            "OrderCreateTopic",       // Topic
            message,                  // 消息体
            request.getOrderId()      // 业务参数（传给 TransactionListener 的 arg）
        );

        // 3. 根据发送结果处理
        if (result.getLocalTransactionState() == LocalTransactionState.COMMIT_MESSAGE) {
            log.info("Order transaction committed: {}", orderId);
        } else {
            log.warn("Order transaction failed: {} state={}",
                orderId, result.getLocalTransactionState());
            throw new TransactionException("订单事务失败");
        }
    }

    private String generateTransactionId() {
        return "TX" + System.currentTimeMillis() + ThreadLocalRandom.current().nextInt(1000);
    }
}
```

### 2.3 低版本 API（原生 Producer）

```java
// 使用 DefaultMQProducer 原生 API 发送事务消息
DefaultMQProducer producer = new DefaultMQProducer("tx-producer-group");
producer.setNamesrvAddr("127.0.0.1:9876");
producer.start();

// 设置事务监听器
producer.setTransactionListener(new OrderTransactionListener());

// 发送事务消息
Message msg = new Message("OrderCreateTopic", "order-123", orderJson.getBytes());
TransactionSendResult result = producer.sendMessageInTransaction(msg, null);
producer.shutdown();
```

## 三、事务状态表（回查关键）

Broker 回查时需要一个可查询的持久化记录表，推荐在业务数据库中创建：

```sql
CREATE TABLE transaction_record (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    transaction_id  VARCHAR(64)   NOT NULL COMMENT '事务ID',
    business_key    VARCHAR(128)  NOT NULL COMMENT '业务主键',
    business_type   VARCHAR(32)   NOT NULL COMMENT '业务类型',
    status          TINYINT       NOT NULL DEFAULT 0 COMMENT '0=PENDING 1=COMMIT 2=ROLLBACK',
    create_time     DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
    update_time     DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_tx_id (transaction_id),
    INDEX idx_biz_key (business_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

```java
@Service
@Transactional
public class TransactionStatusService {

    @Autowired
    private TransactionRecordMapper mapper;

    /**
     * 记录事务状态（在本地事务中执行）
     */
    public void saveTransaction(String transactionId, String businessKey,
                                 String businessType) {
        TransactionRecord record = new TransactionRecord();
        record.setTransactionId(transactionId);
        record.setBusinessKey(businessKey);
        record.setBusinessType(businessType);
        record.setStatus(TransactionStatus.PENDING);
        mapper.insert(record);
    }

    /**
     * 提交事务（修改状态为 COMMIT）
     */
    public void markCommitted(String transactionId) {
        mapper.updateStatus(transactionId, TransactionStatus.COMMIT);
    }

    /**
     * 回滚事务
     */
    public void markRollback(String transactionId) {
        mapper.updateStatus(transactionId, TransactionStatus.ROLLBACK);
    }

    /**
     * 查询事务状态（供回查使用）
     */
    public TransactionStatus getTransactionStatus(String transactionId) {
        TransactionRecord record = mapper.selectByTransactionId(transactionId);
        if (record == null) {
            return TransactionStatus.UNKNOWN;
        }
        return record.getStatus();
    }
}
```

## 四、消费者幂等处理

事务消息投递到消费者时，可能因网络波动导致消息重复，消费者侧必须幂等：

```java
@Component
@RocketMQMessageListener(
    topic = "OrderCreateTopic",
    consumerGroup = "order-create-consumer-group"
)
@Slf4j
@RequiredArgsConstructor
public class OrderCreateConsumer implements RocketMQListener<MessageExt> {

    private final OrderService orderService;
    private final IdempotentService idempotentService;

    @Override
    public void onMessage(MessageExt msg) {
        String msgId = msg.getMsgId();
        String keys = msg.getKeys(); // 业务主键

        // 1. 幂等校验（使用业务主键去重）
        if (idempotentService.isProcessed(keys)) {
            log.info("Duplicate message skipped: keys={}, msgId={}", keys, msgId);
            return;
        }

        try {
            // 2. 反序列化消息
            OrderCreateMessage orderMsg = JsonUtil.parse(
                new String(msg.getBody(), StandardCharsets.UTF_8),
                OrderCreateMessage.class
            );

            // 3. 执行业务逻辑
            orderService.processOrder(orderMsg);

            // 4. 记录已处理
            idempotentService.markProcessed(keys, msgId);

        } catch (Exception e) {
            log.error("Failed to process order message: keys={}", keys, e);
            // 不抛异常 = 消费成功，防止无限重试
            // 如有需要，可投递到死信队列
            if (msg.getReconsumeTimes() >= 3) {
                log.warn("Message retry exhausted, send to DLQ: keys={}", keys);
                throw new RuntimeException("重试耗尽,keys=" + keys);
            }
            throw e; // RocketMQ 会重试
        }
    }
}
```

### 幂等表设计

```sql
CREATE TABLE idempotent_record (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    business_key    VARCHAR(128) NOT NULL COMMENT '业务幂等键',
    msg_id          VARCHAR(64)  NOT NULL COMMENT 'RocketMQ 消息ID',
    status          TINYINT      NOT NULL DEFAULT 1 COMMENT '1=已处理',
    create_time     DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_biz_key (business_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 使用 INSERT IGNORE 实现去重
-- INSERT IGNORE INTO idempotent_record(business_key, msg_id) VALUES(?, ?)
-- 影响行数 = 0 表示已存在，跳过
```

## 五、Spring Boot 配置

```yaml
rocketmq:
  name-server: 127.0.0.1:9876
  producer:
    group: tx-producer-group
    # 事务消息必须开启，保证回查正常
    send-message-timeout: 5000
    # 事务回查线程数
    check-transaction-thread-max-nums: 10
    # 回查初始间隔（毫秒）
    check-transaction-initial-interval: 60000
    # 回查间隔（毫秒）
    check-transaction-interval: 60000
```

## 六、事务消息回查优化

Broker 端回查配置：

```properties
# broker.conf
# 事务回查并发数
transactionCheckMax=15
# 回查间隔（毫秒）
transactionCheckInterval=60000
# 超时时间（毫秒），超过此时间未收到 COMMIT/ROLLBACK 则触发回查
transactionTimeout=6000
```


## 七、常见场景

### 7.1 订单创建 + 积分发放

```java
@Transactional
public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
    OrderCreateMessage orderMsg = parseMessage(msg);

    // 步骤1：创建订单
    orderMapper.insert(orderMsg.toOrder());

    // 步骤2：扣减库存
    inventoryMapper.deduct(orderMsg.getSkuId(), orderMsg.getQuantity());

    // 步骤3：记录事务（供回查）
    transactionStatusService.saveTransaction(
        msg.getTransactionId(), orderMsg.getOrderId(), "ORDER"
    );

    // 如果到这里没抛异常，消息将变成 COMMIT
    // 对应消费者收到消息后发放积分
    return LocalTransactionState.COMMIT_MESSAGE;
}
```

### 7.2 多 Topic 事务依赖

一个本地事务成功后需要发出多个独立消息（不同 Topic）：

```java
@Transactional
public void complexBusinessProcess(BizRequest request) {
    // 同一本地事务中写多个半消息
    TransactionSendResult result1 = rocketMQTemplate.sendMessageInTransaction(
        "TopicA", MessageBuilder.withPayload(data1).build(), null);
    TransactionSendResult result2 = rocketMQTemplate.sendMessageInTransaction(
        "TopicB", MessageBuilder.withPayload(data2).build(), null);

    // 两个消息的事务状态需独立回查
}
```

## 八、注意事项

- **回查有延迟**：默认 Broker 每 60 秒回查一次，事务消息不能保证亚秒级可见
- **回查次数限制**：默认回查 15 次，超过后 Broker 会打印告警但仍会继续重试
- **不能批量发送**：事务消息不支持批量发送（`send()` 的集合参数）
- **不能延时/定时**：事务消息不能同时设置延迟级别，否则事务特性失效
- **临时队列**：半消息存储在 `RMQ_SYS_TRANS_HALF_TOPIC`，回查通过 `RMQ_SYS_TRANS_OP_HALF_TOPIC` 追踪
- **生产者组固定**：`producerGroup` 一旦被用于事务消息，不能混用非事务消息
- **消费者失败不回滚**：消费者处理失败不会反悔已提交的事务——需要设计补偿机制（重试、死信、人工介入）
- **Broker 宕机恢复**：Broker 重启后不会丢失事务消息状态，半消息和回查记录均持久化存储
- **建议配合 TCC**：对于强一致性要求更高的场景，建议在事务消息上层叠加 TCC 模式（Seata TCC + RocketMQ）
