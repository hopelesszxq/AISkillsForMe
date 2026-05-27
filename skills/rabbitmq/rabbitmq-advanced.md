---
name: rabbitmq-advanced
description: RabbitMQ 死信队列、延迟消息与可靠性保障实战
tags: [rabbitmq, mq, message, dlq, delay]
---

## 死信队列（DLQ）完整配置

死信队列用于处理消费失败的消息，是消息可靠性保障的核心机制。

### 声明配置

```java
@Configuration
public class RabbitMQDLQConfig {

    // === 业务队列 ===
    @Bean
    public Queue orderQueue() {
        return QueueBuilder.durable("order.queue")
            .withArgument("x-dead-letter-exchange", "order.dlx.exchange")   // 死信转发到的交换机
            .withArgument("x-dead-letter-routing-key", "order.dlx.routing") // 死信路由键
            .withArgument("x-message-ttl", 60000)    // 消息 TTL 60s（可选）
            .withArgument("x-max-length", 10000)     // 队列最大长度（可选）
            .build();
    }

    @Bean
    public DirectExchange orderExchange() {
        return new DirectExchange("order.exchange");
    }

    @Bean
    public Binding orderBinding() {
        return BindingBuilder.bind(orderQueue())
            .to(orderExchange()).with("order.routing");
    }

    // === 死信队列 ===
    @Bean
    public Queue orderDLQ() {
        return QueueBuilder.durable("order.dlq")
            .withArgument("x-dead-letter-exchange", "")      // 防止死信再次进入死信
            .build();
    }

    @Bean
    public DirectExchange orderDLXExchange() {
        return new DirectExchange("order.dlx.exchange");
    }

    @Bean
    public Binding orderDLQBinding() {
        return BindingBuilder.bind(orderDLQ())
            .to(orderDLXExchange()).with("order.dlx.routing");
    }
}
```

### 死信产生的三种情况

| 场景 | 触发条件 | 应对策略 |
|---|---|---|
| 消息被拒绝 | `basic.reject()` / `basic.nack()` 且 requeue=false | 记录死信原因到数据库，人工补偿或自动重试 |
| 消息超期 | 队列设置了 `x-message-ttl` 或消息设置了 `expiration` | 延长 TTL 或拆分不同优先级的队列 |
| 队列达到上限 | 队列设置了 `x-max-length` 或 `x-max-length-bytes` | 扩容队列容量或拆分业务 |

### 消费重试 + 死信转发

```java
@Component
@Slf4j
public class OrderConsumer {

    @RabbitListener(queues = "order.queue")
    public void handleOrder(Message message, Channel channel) {
        long deliveryTag = message.getMessageProperties().getDeliveryTag();
        try {
            String body = new String(message.getBody());
            Order order = JSON.parseObject(body, Order.class);
            processOrder(order);
            // 处理成功 → 手动确认
            channel.basicAck(deliveryTag, false);
        } catch (Exception e) {
            log.error("订单处理失败: {}", e.getMessage(), e);
            try {
                // requeue=false → 进入死信队列
                channel.basicNack(deliveryTag, false, false);
            } catch (IOException ex) {
                log.error("NACK 失败", ex);
            }
        }
    }

    // 死信队列消费者：记录失败原因 + 人工介入
    @RabbitListener(queues = "order.dlq")
    public void handleDeadLetter(Message message) {
        String body = new String(message.getBody());
        MessageProperties props = message.getMessageProperties();

        log.warn("死信消息: body={}, reason={}, exchange={}, routingKey={}",
            body,
            props.getHeader("x-death"),  // 死信原因
            props.getHeader("x-original-exchange"),
            props.getHeader("x-original-routing-key"));

        // 存入数据库，等待人工处理或定时补偿
        deadLetterService.save(new DeadLetterRecord(body, props.getHeaders()));
    }
}
```

## 延迟消息实现

RabbitMQ 原生不支持任意延迟，但通过 **死信队列 + TTL** 或 **延迟消息插件** 实现。

### 方式一：死信队列 + TTL（推荐，不依赖插件）

```java
@Configuration
public class DelayQueueConfig {

    // 延迟队列 —— 消息过期后转入业务队列
    @Bean
    public Queue delayQueue() {
        return QueueBuilder.durable("delay.queue")
            .withArgument("x-dead-letter-exchange", "order.exchange")      // 到期转到的交换机
            .withArgument("x-dead-letter-routing-key", "order.routing")    // 到期后的路由键
            .build();
    }

    @Bean
    public DirectExchange delayExchange() {
        return new DirectExchange("delay.exchange");
    }

    @Bean
    public Binding delayBinding() {
        return BindingBuilder.bind(delayQueue())
            .to(delayExchange()).with("delay.routing");
    }

    // 发送延迟消息：根据不同的延迟时间设置不同的 TTL
    public void sendDelay(Order order, long delayMillis) {
        MessageProperties props = new MessageProperties();
        props.setExpiration(String.valueOf(delayMillis));  // 消息级别 TTL
        Message message = MessageBuilder.withBody(JSON.toJSONBytes(order))
            .andProperties(props).build();

        rabbitTemplate.send("delay.exchange", "delay.routing", message);
    }
}
```

> **注意**：队列级别的 TTL（`x-message-ttl`）优先级低于消息级别的 TTL。多个不同 TTL 的消息在同一队列时，**只有队首消息过期后才检查下一个**，这会导致长 TTL 消息阻塞短 TTL 消息。解决方案是按延迟时间分队列。

### 方式二：延迟消息插件

```bash
# 1. 安装插件（RabbitMQ 3.13+ 内置）
rabbitmq-plugins enable rabbitmq_delayed_message_exchange

# 2. 声明延迟交换机
```

```java
@Bean
public CustomExchange delayExchange() {
    Map<String, Object> args = new HashMap<>();
    args.put("x-delayed-type", "direct");  // 底层用的交换机类型
    return new CustomExchange("delay.exchange",
        "x-delayed-message", true, false, args);
}

// 发送延迟
public void sendWithDelay(Order order, long delayMillis) {
    MessageProperties props = new MessageProperties();
    props.setDelay(Math.toIntExact(delayMillis));  // 关键！设置延迟毫秒数
    Message message = MessageBuilder.withBody(JSON.toJSONBytes(order))
        .andProperties(props).build();

    rabbitTemplate.send("delay.exchange", "order.routing", message);
}
```

## 发布者确认（Publisher Confirm）

保证消息成功到达 RabbitMQ Broker。

### 配置

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated   # 开启确认回调（确认消息是否到达交换机）
    publisher-returns: true              # 开启消息回退（消息无法路由到队列时回调）
    template:
      mandatory: true                    # 强制：没有路由到队列的消息必须回调
```

### 代码实现

```java
@Component
@Slf4j
public class RabbitMQPublisherConfig implements RabbitTemplate.ConfirmCallback,
                                               RabbitTemplate.ReturnsCallback {

    @PostConstruct
    public void init() {
        rabbitTemplate.setConfirmCallback(this);
        rabbitTemplate.setReturnsCallback(this);
    }

    /**
     * 消息确认回调：确认消息是否到达交换机
     * @param correlationData 消息关联数据（含消息ID）
     * @param ack true=交换机成功接收 false=交换机未接收
     * @param cause 失败原因
     */
    @Override
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {
        if (ack) {
            log.debug("消息确认成功: {}", correlationData.getId());
        } else {
            log.error("消息确认失败: id={}, cause={}", correlationData.getId(), cause);
            // 写入失败表，定时补偿
            failedMessageService.save(correlationData.getId(), cause);
        }
    }

    /**
     * 消息回退回调：消息到达交换机但无法路由到队列
     */
    @Override
    public void returnedMessage(ReturnedMessage returned) {
        log.warn("消息路由失败: exchange={}, routingKey={}, replyCode={}, replyText={}",
            returned.getExchange(), returned.getRoutingKey(),
            returned.getReplyCode(), returned.getReplyText());
        // 存储失败消息，后续处理
        failedMessageService.save(returned.getMessage(), returned.getReplyText());
    }

    // 发送消息时携带 CorrelationData
    public void sendWithConfirm(Order order) {
        CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());
        Message message = MessageBuilder.withBody(JSON.toJSONBytes(order))
            .setCorrelationId(correlationData.getId())
            .build();

        rabbitTemplate.send("order.exchange", "order.routing", message, correlationData);
    }
}
```

## 注意事项

1. **死信队列一定要监控**：死信堆积意味着业务处理持续失败，需要告警和补偿机制
2. **延迟消息精度**：基于 TTL 的延迟精度在毫秒级，基于插件的延迟精度更高但增加交换机开销
3. **幂等消费**：即使开启 Publisher Confirm，也可能出现重复消息，消费者必须幂等（唯一键去重）
4. **批量确认**：`basicAck` 的 `multiple=true` 参数可一次确认多条消息，提升吞吐量
5. **Prefetch 控制**：`spring.rabbitmq.listener.simple.prefetch=10` 控制消费者预取数量，避免消息倾斜
6. **连接监听**：实现 `ConnectionListener` 监控连接断连，触发重连后的队列声明补偿
7. **消息体序列化**：生产环境用 JSON（通用）或者 Protobuf（高性能），避免 JDK 序列化
