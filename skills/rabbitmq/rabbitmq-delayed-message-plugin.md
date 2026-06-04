---
name: rabbitmq-delayed-message-plugin
description: RabbitMQ 延迟消息插件(rabbitmq_delayed_message_exchange)：安装配置、延迟队列模式与死信回落
tags: [rabbitmq, delayed, plugin, exchange, scheduling, retry]
---

## 概述

RabbitMQ 通过 `rabbitmq_delayed_message_exchange` 插件实现延迟消息投递（鸽巢式延迟），无需依赖死信队列（DLX）。消息发送到延迟交换机后，不会立即路由到队列，而是在指定延迟时间过后才投递。

> 对比 DLX 延迟方案：DLX 利用死信 TTL 模拟延迟，但延迟精度差、不支持任意延迟时间。插件方案支持秒级到天级的任意延迟。

## 安装插件

```bash
# 启用插件（RabbitMQ 3.12+ 内置）
rabbitmq-plugins enable rabbitmq_delayed_message_exchange

# 验证
rabbitmq-plugins list | grep delayed
```

Docker 镜像默认不包含此插件，需自定义镜像：

```dockerfile
FROM rabbitmq:3.13-management
RUN rabbitmq-plugins enable --offline rabbitmq_delayed_message_exchange
```

## 交换机类型详解

延迟插件通过 **x-delayed-message** 类型交换机实现，内部使用 **Mnesia 数据库**存储延迟消息元数据：

| 参数 | 说明 | 示例值 |
|------|------|--------|
| x-delayed-type | 基础交换机类型 | direct / topic / fanout / headers |
| x-delayed-message | 发送时设置的延迟头 | 毫秒数 |

## Spring Boot 集成

### 1. 声明延迟交换机与队列

```java
@Configuration
public class DelayedQueueConfig {

    public static final String DELAYED_EXCHANGE = "exchange.delayed";
    public static final String DELAYED_QUEUE = "queue.delayed";
    public static final String DELAYED_ROUTING_KEY = "order.delayed";

    @Bean
    public CustomExchange delayedExchange() {
        Map<String, Object> args = new HashMap<>();
        args.put("x-delayed-type", "direct");
        // x-delayed-message 类型交换机
        return new CustomExchange(
                DELAYED_EXCHANGE,
                "x-delayed-message",
                true,      // durable
                false,     // auto-delete
                args);
    }

    @Bean
    public Queue delayedQueue() {
        return QueueBuilder.durable(DELAYED_QUEUE).build();
    }

    @Bean
    public Binding delayedBinding(Queue delayedQueue, CustomExchange delayedExchange) {
        return BindingBuilder.bind(delayedQueue)
                .to(delayedExchange)
                .with(DELAYED_ROUTING_KEY)
                .noargs();
    }
}
```

### 2. 发送延迟消息

```java
@Component
public class DelayedMessageSender {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void sendDelayed(String message, long delayMillis) {
        MessageProperties props = new MessageProperties();
        // 关键：设置延迟时间（毫秒）
        props.setDelay(Math.toIntExact(delayMillis));
        props.setDeliveryMode(MessageDeliveryMode.PERSISTENT);

        Message msg = MessageBuilder.withBody(message.getBytes(StandardCharsets.UTF_8))
                .andProperties(props)
                .build();

        rabbitTemplate.send(DelayedQueueConfig.DELAYED_EXCHANGE,
                DelayedQueueConfig.DELAYED_ROUTING_KEY, msg);
        log.info("延迟消息已发送，延迟 {}ms: {}", delayMillis, message);
    }
}
```

### 3. 消费延迟消息

```java
@Component
@Slf4j
public class DelayedMessageConsumer {

    @RabbitListener(queues = DelayedQueueConfig.DELAYED_QUEUE)
    public void handleDelayedMessage(Message message, Channel channel) {
        long deliveryTag = message.getMessageProperties().getDeliveryTag();
        try {
            String body = new String(message.getBody(), StandardCharsets.UTF_8);
            // 获取实际延迟的毫秒数（消息实际延迟可能略长于设置值）
            long actualDelay = System.currentTimeMillis() -
                    message.getMessageProperties().getTimestamp().getTime();
            log.info("收到延迟消息: {}, 实际延迟: {}ms", body, actualDelay);
            channel.basicAck(deliveryTag, false);
        } catch (Exception e) {
            channel.basicNack(deliveryTag, false, true);
        }
    }
}
```

## 业务场景模式

### 模式一：订单超时取消

```java
@Service
public class OrderService {

    public Order createOrder(OrderDTO dto) {
        Order order = orderRepository.save(dto.toEntity());
        // 30 分钟后检查订单是否已支付
        delayedSender.sendDelayed(
                new OrderTimeoutEvent(order.getId()).toJson(),
                Duration.ofMinutes(30).toMillis()
        );
        return order;
    }
}
```

### 模式二：指数退避重试（与死信队列配合）

```
[业务队列] ---消费失败---> [延迟交换机] ----延迟后---> [重试队列]
                                                    |
                                             消费成功? 
                                             /       \
                                            Y         N (3次后进DLQ)
```

```java
@RabbitListener(queues = "queue.retry")
public void handleRetry(Message message, Channel channel) {
    long retryCount = getRetryCount(message);
    try {
        processMessage(message);
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    } catch (Exception e) {
        if (retryCount >= 3) {
            // 进入最终死信队列
            channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, false);
        } else {
            // 指数退避重试：10s, 20s, 40s
            long delay = (long) (Math.pow(2, retryCount) * 10_000);
            delayedSender.sendDelayed(new String(message.getBody()), delay);
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        }
    }
}

private long getRetryCount(Message msg) {
    String retryHeader = msg.getMessageProperties().getHeader("x-retry-count");
    return retryHeader == null ? 0 : Long.parseLong(retryHeader) + 1;
}
```

### 模式三：定时任务调度

```java
@Scheduled(fixedRate = 60_000)
public void scheduleDailyReport() {
    // 每天早上 8 点发送报表
    LocalDateTime now = LocalDateTime.now();
    LocalDateTime target = now.withHour(8).withMinute(0).withSecond(0);
    if (now.isAfter(target)) {
        target = target.plusDays(1);
    }
    long delay = Duration.between(now, target).toMillis();
    delayedSender.sendDelayed("每日报表生成", delay);
}
```

## 配置与性能优化

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 10
        retry:
          enabled: false  # 关闭 Spring Retry，由业务代码控制重试
    template:
      retry:
        enabled: true
        max-attempts: 3
```

插件核心参数（rabbitmq.conf）：

```ini
# 持久化延迟消息到 Mnesia，重启后恢复（有一定性能开销）
delayed_message_store_persistent = true

# 延迟消息检查间隔（毫秒），越小精度越高，CPU 开销越大
delayed_message_check_period_ms = 100
```

## 注意事项

- **插件局限性**：
  - 延迟消息存储在 Mnesia 中，大量延迟消息堆积会严重影响 RabbitMQ 性能（建议不超过数万条）
  - 延迟精度受 `delayed_message_check_period_ms` 限制，非精确到毫秒
  - 插件重启后如果有大量未到期消息，恢复时间较长
  - 不支持仲裁队列（Quorum Queue），只能使用经典队列
- **替代方案**：大批量延迟消息（>10万）建议使用 RocketMQ 延迟消息或 Redis ZSet + 定时任务
- **不可持久化的坑**：如果 RabbitMQ 宕机，Mnesia 表中的延迟消息可能丢失，建议结合业务数据库状态做补偿
- **TTL 与延迟共存**：如果同时设置消息 TTL 和延迟时间，TTL 从消息到达队列时开始计时，而非延迟结束后
- **监控告警**：通过 `rabbitmqctl list_exchanges` 查看延迟消息积压情况，建议配置 Prometheus 告警
