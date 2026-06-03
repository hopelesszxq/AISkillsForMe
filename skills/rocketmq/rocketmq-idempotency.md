---
name: rocketmq-idempotency
description: RocketMQ 消息幂等性实战方案：业务去重与防重复消费
tags: [rocketmq, mq, idempotency, retry, dlq, deduplication]
---

## 概述

消息幂等性是消息队列生产中的硬性要求。RocketMQ 的 At-Least-Once 语义保证消息至少被消费一次，但可能因网络抖动、消费者宕机重启等原因导致重复投递。业务系统自身必须实现幂等保障。

## 为什么需要幂等

```
生产者 ──发送──► Broker ──投递──► 消费者
                   │
                   ├── 消费成功 ✓
                   ├── 消费超时 → 重新投递
                   ├── 消费者宕机 → 切换到另一个消费者
                   └── 未提交 Offset → 重启后重新消费
```

RocketMQ 的以下机制会导致重复消息：
- **重试机制**：消费失败后重新投递
- **Consumer 负载均衡**：Rebalance 可能导致消息被重新消费
- **Timeout 超时**：业务处理超时但 Offset 未提交，消息被重新投递
- **生产者重复发送**：发送超时重试导致 Broker 收到多条相同消息

## 1. 幂等方案对比

| 方案 | 存储位置 | 性能 | 可靠性 | 复杂度 |
|------|----------|------|--------|--------|
| **Redis 去重表** | Redis | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ |
| **数据库唯一索引** | 数据库 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ |
| **业务表幂等字段** | 数据库 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **本地缓存去重** | JVM 内存 | ⭐⭐⭐⭐⭐ | ⭐ | ⭐ |
| **RocketMQ 事务消息** | Broker + DB | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

## 2. 推荐方案：Redis 去重表

### 幂等 Key 生成策略

```java
public class IdempotentKeyGenerator {

    // 方案一：使用消息 ID（最简单，但不同重试的 msgId 可能不同）
    public static String byMessageId(MessageExt msg) {
        return "idempotent:msg:" + msg.getMsgId();
    }

    // 方案二：使用业务 Key（推荐）
    public static String byBusinessKey(MessageExt msg, String bizPrefix) {
        // 从消息 body 中提取业务唯一标识
        String body = new String(msg.getBody(), StandardCharsets.UTF_8);
        // 例如：orderId, transactionId 等
        JSONObject json = JSONObject.parseObject(body);
        String bizId = json.getString("orderId");
        return "idempotent:biz:" + bizPrefix + ":" + bizId;
    }

    // 方案三：使用 Message Key（生产者设置）
    public static String byMessageKey(MessageExt msg) {
        String keys = msg.getKeys();
        return "idempotent:key:" + keys;
    }
}
```

### 完整幂等消费者实现

```java
@Component
public class IdempotentOrderConsumer {

    @Autowired
    private StringRedisTemplate redisTemplate;

    private static final String IDEMPOTENT_PREFIX = "idempotent:order:";

    @RocketMQMessageListener(
        topic = "order-topic",
        consumerGroup = "order-consumer-group",
        consumeMode = ConsumeMode.CONCURRENTLY
    )
    public class OrderConsumer implements RocketMQListener<MessageExt> {

        @Override
        public ConsumeConcurrentlyStatus consumeMessage(
                List<MessageExt> msgs,
                ConsumeConcurrentlyContext context) {

            for (MessageExt msg : msgs) {
                String bizId = extractBizId(msg);
                String idempotentKey = IDEMPOTENT_PREFIX + bizId;

                // 尝试获取幂等锁（SET NX EX）
                Boolean acquired = redisTemplate.opsForValue()
                    .setIfAbsent(idempotentKey, "processing", 
                        Duration.ofMinutes(30));

                if (Boolean.FALSE.equals(acquired)) {
                    String status = redisTemplate.opsForValue()
                        .get(idempotentKey);
                    if ("done".equals(status)) {
                        // 已处理完成，直接跳过
                        log.info("Skip duplicate msg: {}", bizId);
                        continue;
                    }
                    // 正在处理中，稍后重试
                    return ConsumeConcurrentlyStatus.RECONSUME_LATER;
                }

                try {
                    // 执行业务逻辑
                    processOrder(bizId, msg.getBody());

                    // 标记幂等 Key 为已完成
                    redisTemplate.opsForValue()
                        .set(idempotentKey, "done", Duration.ofDays(3));
                } catch (Exception e) {
                    log.error("Process order failed: {}", bizId, e);
                    // 删除幂等锁，允许下次重试
                    redisTemplate.delete(idempotentKey);
                    return ConsumeConcurrentlyStatus.RECONSUME_LATER;
                }
            }
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        }
    }
}
```

## 3. 数据库唯一索引方案

### MySQL 去重表设计

```sql
-- 幂等去重表
CREATE TABLE `mq_idempotent` (
    `id` bigint(20) NOT NULL AUTO_INCREMENT,
    `biz_key` varchar(128) NOT NULL COMMENT '业务幂等键',
    `topic` varchar(64) NOT NULL COMMENT '队列主题',
    `consumer_group` varchar(64) NOT NULL COMMENT '消费组',
    `message_id` varchar(64) DEFAULT NULL COMMENT '消息 ID',
    `status` tinyint(4) NOT NULL DEFAULT '0' COMMENT '0=处理中 1=成功 2=失败',
    `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `update_time` datetime DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_biz_key` (`biz_key`, `topic`, `consumer_group`),
    KEY `idx_message_id` (`message_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='消息幂等去重表';
```

### 消费代码

```java
@Service
public class IdempotentOrderService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Transactional(rollbackFor = Exception.class)
    public boolean tryProcess(String bizKey, String topic, 
                              String consumerGroup, Runnable bizLogic) {
        // INSERT ... ON DUPLICATE KEY 实现幂等
        int inserted = jdbcTemplate.update(
            "INSERT INTO mq_idempotent(biz_key, topic, consumer_group, status) " +
            "VALUES(?, ?, ?, 0) " +
            "ON DUPLICATE KEY UPDATE id=id",
            bizKey, topic, consumerGroup
        );

        if (inserted == 0) {
            // 已存在记录，检查状态
            Integer status = jdbcTemplate.queryForObject(
                "SELECT status FROM mq_idempotent WHERE biz_key=? AND topic=? AND consumer_group=?",
                Integer.class, bizKey, topic, consumerGroup
            );
            if (status != null && status == 1) {
                log.info("Duplicate msg, already processed: {}", bizKey);
                return true; // 已处理，跳过
            }
            return false; // 处理中或失败，重试
        }

        try {
            bizLogic.run();
            jdbcTemplate.update(
                "UPDATE mq_idempotent SET status=1 WHERE biz_key=? AND topic=? AND consumer_group=?",
                bizKey, topic, consumerGroup
            );
            return true;
        } catch (Exception e) {
            jdbcTemplate.update(
                "UPDATE mq_idempotent SET status=2 WHERE biz_key=? AND topic=? AND consumer_group=?",
                bizKey, topic, consumerGroup
            );
            throw e;
        }
    }
}
```

### 使用方式

```java
@RocketMQMessageListener(
    topic = "order-topic",
    consumerGroup = "order-group"
)
public class OrderConsumer implements RocketMQListener<MessageExt> {

    @Autowired
    private IdempotentOrderService idempotentService;

    @Override
    public ConsumeConcurrentlyStatus consumeMessage(
            List<MessageExt> list,
            ConsumeConcurrentlyContext context) {

        for (MessageExt msg : list) {
            String bizKey = extractBizId(msg);
            boolean processed = idempotentService.tryProcess(
                bizKey,
                msg.getTopic(),
                msg.getProperty("CONSUMER_GROUP"),
                () -> doBusiness(msg.getBody())
            );
            if (!processed) {
                return ConsumeConcurrentlyStatus.RECONSUME_LATER;
            }
        }
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
}
```

## 4. 幂等 Key 设计原则

### 好的幂等 Key

```java
// ✅ 推荐的幂等 Key
String goodKey1 = "order:" + order.getOrderNo();        // 订单号
String goodKey2 = "payment:" + payment.getTxNo();       // 支付流水号
String goodKey3 = "refund:" + refund.getRefundNo();     // 退款单号
String goodKey4 = "task:" + task.getId() + ":" + task.getVersion(); // 任务版本
String goodKey5 = "user:" + userId + ":action:" + timestamp; // 用户操作去重
```

### 不好的幂等 Key

```java
// ❌ 不推荐的幂等 Key
String badKey1 = msg.getMsgId();   // 重试时 msgId 可能改变
String badKey2 = String.valueOf(System.nanoTime()); // 每次不同
String badKey3 = new String(msg.getBody());  // 太长的 Key
String badKey4 = "" + new Random().nextInt(); // 随机值
```

## 5. 与重试/DLQ 配合的最佳实践

```java
@RocketMQMessageListener(
    topic = "order-topic",
    consumerGroup = "order-group",
    // 设置合理的重试策略
    consumeMaxRetryTimes = 3,
    // 重试间隔（毫秒），根据业务需要
    delayTimeLevels = {"1s", "5s", "10s"}
)
public class OrderConsumer implements RocketMQListener<MessageExt> {

    private static final int MAX_RETRY_COUNT = 5;

    @Override
    public ConsumeConcurrentlyStatus consumeMessage(...) {
        // 从消息属性获取重试次数
        String retryCountStr = msg.getProperty("RETRY_COUNT");
        int retryCount = retryCountStr != null ? 
            Integer.parseInt(retryCountStr) : 0;

        // 超过最大重试次数，记录到 DB 并 ACK（不进入 DLQ）
        if (retryCount >= MAX_RETRY_COUNT) {
            saveToDeadLetterTable(msg, "超过最大重试次数");
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        }

        // 幂等判断 ...
    }
}
```

## 6. 生产者幂等（防止重复发送）

```java
@Service
public class IdempotentOrderProducer {

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    // 生产者设置唯一 Message Key
    public SendResult sendOrder(Order order) {
        Message<String> message = MessageBuilder
            .withPayload(JSON.toJSONString(order))
            .setHeader("orderNo", order.getOrderNo())
            .build();

        // 设置 Message Key 用于幂等追溯
        return rocketMQTemplate.syncSend(
            "order-topic",
            message,
            // 发送超时
            3000,
            // delayLevel: 0 不延迟
            0
        );
    }
}
```

### 发送去重（基于事务消息）

```java
@Service
public class OrderTransactionProducer {

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    @Transactional
    public void createOrderWithTransaction(Order order) {
        // 1. 先保存订单到数据库
        orderMapper.insert(order);

        // 2. 发送事务消息（如果消息发送失败或消费者处理失败，
        //    事务回查机制保证最终一致性）
        Message<String> msg = MessageBuilder
            .withPayload(JSON.toJSONString(order))
            .setHeader("orderNo", order.getOrderNo())
            .build();

        TransactionSendResult result = rocketMQTemplate.sendMessageInTransaction(
            "order-tx-producer-group",
            "order-tx-topic",
            msg,
            order
        );

        if (result.getLocalTransactionState() != LocalTransactionState.COMMIT_MESSAGE) {
            throw new RuntimeException("事务消息发送失败");
        }
    }
}
```

## 注意事项

1. **幂等 Key 必须全局唯一**：跨 Topic、跨消费者组的场景，Key 需要加上 Topic 和 Group 前缀
2. **Redis 方案注意持久化**：Redis 宕机可能丢失幂等记录，建议配合数据库方案做双保险
3. **TTL 设置合理**：幂等 Key 的 TTL 至少为业务重试窗口的 2 倍（推荐 3-7 天）
4. **有序消息的去重**：顺序消费模式下，幂等 Key 可以更短过期时间（处理完即删）
5. **重试消息的 msgId 变化**：RocketMQ 重试投递时 msgId 会改变，不能直接用 msgId 做幂等 Key
6. **性能考量**：数据库唯一索引方案在高并发（>5000 TPS）场景下可能成为瓶颈，建议先查后插或使用 Redis 缓存
7. **幂等不替代业务校验**：即使有幂等机制，消费者也应该做业务状态校验（如订单状态已为"已完成"则跳过）
8. **监控告警**：对幂等命中次数做监控，异常增高说明可能有重复发送问题
