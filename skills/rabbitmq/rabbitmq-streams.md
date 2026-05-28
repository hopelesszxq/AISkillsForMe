---
name: rabbitmq-streams
description: RabbitMQ Stream 插件与 Quorum Queues 实践（高吞吐、持久化与 4.x 新特性）
tags: [rabbitmq, mq, stream, quorum, high-availability]
---

## RabbitMQ Streams 概述

RabbitMQ Streams 是 RabbitMQ 的流式消息处理插件（已内置，无需额外安装），相比传统 AMQP 队列具有以下优势：

| 特性 | 传统队列 | Stream |
|---|---|---|
| 吞吐量 | ~10K msg/s | ~100K+ msg/s |
| 消息存储 | 消费即删除 | 持久化保留（按 TTL 配置） |
| 消费模式 | 竞争消费 | 任意偏移量读取（多次消费 / 回溯） |
| 协议支持 | AMQP 0-9-1 | AMQP 0-9-1 + Binary Stream Protocol |
| 适用场景 | 任务分发、RPC | 日志、事件溯源、大数据管道 |

### 启用 Stream 插件

```bash
# RabbitMQ 3.9+ 已内置，只需启用
rabbitmq-plugins enable rabbitmq_stream

# 如果使用 Stream 协议（非 AMQP），还需启用端口
# 默认端口 5552（二进制流协议），可在配置中修改
```

### Spring Boot 集成 Stream

```xml
<dependency>
    <groupId>com.rabbitmq.stream</groupId>
    <artifactId>rabbitmq-stream-client</artifactId>
    <version>0.19.0</version>
</dependency>
```

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672       # AMQP 端口
    stream:
      host: localhost
      port: 5552     # Stream 协议端口
      username: guest
      password: guest
```

### 生产者：发送 Stream 消息

```java
@Component
public class StreamProducer {

    @Autowired
    private RabbitStreamTemplate streamTemplate;

    // 环境名（Stream 协议中用于标识 producer）
    private static final String STREAM_NAME = "order_events_stream";

    @PostConstruct
    public void init() {
        // 确保 Stream 存在（自动创建）
        streamTemplate.streamBuilder()
            .name(STREAM_NAME)
            .maxLengthBytes(ByteCapacity.GB)     // 最大 1GB（循环覆盖）
            .maxAge(Duration.ofDays(7))          // 保留 7 天
            .create();
    }

    /**
     * 发送事件到 Stream
     * Stream 消息是 append-only，不可删除
     */
    public void sendOrderEvent(OrderEvent event) {
        Message message = MessageBuilder
            .withBody(JSON.toJSONBytes(event))
            .setMessageAnnotation("event_type", event.getType())  // 自定义标签
            .build();

        streamTemplate.send(STREAM_NAME, message);

        // 同步发送确认（阻塞）
        // streamTemplate.sendAndWait(STREAM_NAME, message);
    }
}
```

### 消费者：从 Stream 读取消息

```java
@Component
public class StreamConsumer {

    private static final String STREAM_NAME = "order_events_stream";

    /**
     * 从特定偏移量开始消费
     */
    @StreamListener(value = STREAM_NAME, offset = StreamOffset.FIRST)
    public void consumeFromFirst(org.springframework.messaging.Message<byte[]> message) {
        // offset = FIRST：从 Stream 中最早的消息开始消费
        byte[] body = message.getPayload();
        OrderEvent event = JSON.parseObject(body, OrderEvent.class);
        System.out.println("消费（从头）: " + event);
    }

    /**
     * 只消费新增消息（类似传统队列）
     */
    @StreamListener(value = STREAM_NAME, offset = StreamOffset.LAST)
    public void consumeNewMessages(com.rabbitmq.stream.Message message) {
        // offset = LAST：只消费订阅之后产生的新消息
        byte[] body = message.getBodyAsBinary();
        OrderEvent event = JSON.parseObject(body, OrderEvent.class);
        System.out.println("消费（新增）: " + event);
    }

    /**
     * 指定偏移量消费（回溯消费）
     */
    @StreamListener(value = STREAM_NAME, offset = StreamOffset.OFFSET, offsetSpec = 50000L)
    public void consumeFromOffset(Message message) {
        // 从第 50000 条消息开始消费
    }

    /**
     * 按时间消费
     */
    @StreamListener(value = STREAM_NAME, offset = StreamOffset.TIMESTAMP)
    public void consumeByTime(Message message) {
        // offsetSpec 属性在 Spring AMQP 中可通过 @StreamListener 参数传递时间戳
        // 但更灵活的方式是手动创建消费者
    }
}
```

### 手动 Stream 消费者

```java
@Service
public class ManualStreamService {

    @Autowired
    private Environment environment;

    public void consumeWithTimestamp() {
        // 获取 Environment 实例
        // 从指定时间开始消费：2026-01-15 00:00:00 UTC
        Instant since = LocalDateTime.of(2026, 1, 15, 0, 0)
            .toInstant(ZoneOffset.UTC);

        Consumer consumer = environment.consumerBuilder()
            .stream("order_events_stream")
            .offset(OffsetSpecification.timestamp(since.toEpochMilli()))
            .messageHandler((offset, message) -> {
                byte[] body = message.getBodyAsBinary();
                System.out.println("回溯消费: " + new String(body));

                // 手动确认（credits 机制）
                // 默认 auto-credit 策略，无需手动确认
            })
            .build();
    }
}
```

## Quorum Queues（仲裁队列）

Quorum Queues 是 RabbitMQ 3.8+ 引入的高可用队列类型，基于 Raft 共识协议。

### 为什么使用 Quorum Queues

- **数据安全**：消息自动复制到多个节点，单节点宕机不丢数据
- **强一致性**：所有节点数据一致，不会出现脑裂
- **自动故障转移**：主节点挂掉后自动选举新主节点

### 声明 Quorum Queue

```java
@Configuration
public class QuorumQueueConfig {

    /**
     * 声明仲裁队列（替代经典镜像队列）
     */
    @Bean
    public Queue orderQuorumQueue() {
        return QueueBuilder.durable("order.quorum.queue")
            .quorum()                             // 声明为仲裁队列
            .withArgument("x-queue-type", "quorum")  // 明确类型
            .withArgument("x-quorum-initial-group-size", 3)  // 集群节点数（3-5 推荐）
            .withArgument("x-max-length", 100_000)      // 最大消息数
            .withArgument("x-overflow", "reject-publish")  // 超量后拒绝新消息
            .build();
    }

    @Bean
    public DirectExchange orderExchange() {
        return new DirectExchange("order.exchange");
    }

    @Bean
    public Binding orderBinding() {
        return BindingBuilder.bind(orderQuorumQueue())
            .to(orderExchange()).with("order.routing");
    }
}
```

### Quorum Queue + 死信

```java
@Bean
public Queue orderQuorumQueue() {
    return QueueBuilder.durable("order.quorum.queue")
        .quorum()
        .deadLetterExchange("order.dlx.exchange")
        .deadLetterRoutingKey("order.dlx.routing")
        .build();
}
```

### Quorum Queue 监控

```bash
# 查看队列状态
rabbitmq-diagnostics check_if_nodes_are_quorum_critical

# 查看队列是否是 quorum 类型
rabbitmqctl list_queues name type

# 查看 Raft 状态
rabbitmqctl eval 'rabbit_quorum_queue:status(<<"order.quorum.queue">>).'
```

## Super Streams（分区流）

RabbitMQ 3.11+ 支持 Super Streams，用于 Stream 的分区扩展。

### 创建 Super Stream

```bash
# 命令行创建 Super Stream（3 个分区）
rabbitmq-streams add_super_stream order_super_stream \
  --partitions 3
```

### Java 客户端写入 Super Stream

```java
@Service
public class SuperStreamProducer {

    @Autowired
    private RabbitStreamTemplate streamTemplate;

    private static final String SUPER_STREAM = "order_super_stream";

    public void sendPartitioned(OrderEvent event) {
        // 基于订单 ID 的哈希值决定分区，保证同一订单的消息在同一个分区
        String routingKey = event.getOrderId();

        Message message = MessageBuilder
            .withBody(JSON.toJSONBytes(event))
            .build();

        // 自动按 routingKey 哈希路由到对应分区
        streamTemplate.send(SUPER_STREAM, routingKey, message);
    }
}
```

## RabbitMQ 4.x 新特性

RabbitMQ 4.0（2025 年发布）带来了多项重要更新：

### 移除经典镜像队列

RabbitMQ 4.0 **完全移除了经典镜像队列（Classic Mirroring）**，必须使用 Quorum Queues 实现高可用。

```yaml
# 如从 3.x 升级到 4.x，需先迁移镜像队列到 Quorum
# 迁移工具：
rabbitmq-queues quorum_migrate <queue_name>
```

### 默认使用 Quorum Queues

```yaml
# RabbitMQ 配置文件 rabbitmq.conf
# 4.x 默认队列类型
default_queue_type = quorum
```

### 新增 Stream 特性

- **Stream 压缩**：支持 GZIP 压缩存储，减少磁盘占用
- **Super Stream 分区自动扩容**：支持在线增加分区数
- **Stream 过滤器**：消费端过滤特定类型的消息，减少网络传输

### 性能优化配置

```yaml
# rabbitmq.conf
# Stream 相关配置
stream.max_segment_size_bytes = 536870912        # 512MB 段大小
stream.initial_cluster_size = 3                  # 初始集群大小

# Quorum Queue 配置
quorum_cluster_size = 3                          # 仲裁队列副本数
cluster_partition_handling = autoheol            # 分区自动修复
quorum_commands_soft_limit = 64                  # 命令软限制
```

## 注意事项

1. **Stream 不是 AMQP 队列**：Stream 协议使用独立端口（5552），需要客户端库支持
2. **Stream 消息不会自动删除**：除非设置 `maxAge` 或 `maxLengthBytes`，否则消息一直保留，注意磁盘空间
3. **Quorum Queue 的性能**：由于 Raft 共识，Quorum Queue 的延迟略高于经典队列（约 20-50%），但远低于镜像队列
4. **Quorum Queue 不支持的事务**：仲裁队列不支持 AMQP 事务（`txSelect`），使用 Publisher Confirm 代替
5. **Queue 类型不能修改**：创建后不能将经典队列转为 Quorum Queue，需使用 `quorum_migrate` 工具迁移
6. **Stream 消费者数不限**：同一个 Stream 可以有多个独立消费者，各自维护 offset，互不影响
7. **消息回溯能力是双刃剑**：方便重放和审计，但可能造成重复消费，消费者必须做好幂等
8. **单分区 Stream 的顺序保证**：同一分区内的消息顺序严格保证，Super Stream 跨分区不保证顺序
