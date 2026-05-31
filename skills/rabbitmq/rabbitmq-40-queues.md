---
name: rabbitmq-40-queues
description: RabbitMQ 4.0 新增及改进队列类型（AMQP 1.0、Stream Queues、Quorum Queues 改进）
tags: [rabbitmq, mq, amqp, steam-queue]
---

## RabbitMQ 4.0 队列类型深度解析

RabbitMQ 4.0 是一个重要的版本更新，主要包括 **AMQP 1.0 原生支持**、**流队列（Stream Queues）**、**仲裁队列改进**，以及去除经典队列 v1 版本。

## 四种队列类型对比

| 特性 | 经典队列 v2 | 仲裁队列 | 流队列 | AMQP 1.0 |
|------|------------|---------|--------|----------|
| 协议 | AMQP 0-9-1 | AMQP 0-9-1 | 自有协议 + AMQP 0-9-1 | AMQP 1.0 |
| 持久化 | 可选 | 默认 | 默认 | 可选 |
| 复制 | 无 | Raft 多副本 | Raft 多副本 | N/A |
| 消费模式 | 破坏性 | 破坏性 | 非破坏性 | 标准 AMQP |
| 适用场景 | 通用消息 | 高可用 | 日志/事件溯源/大数据 | 跨平台互通 |

## AMQP 1.0 原生支持（重大更新）

RabbitMQ 4.0 将 AMQP 1.0 从插件升级为**原生支持的协议**，与 AMQP 0-9-1 平级。

```yaml
# 配置文件启用 AMQP 1.0
# rabbitmq.conf
amqp_1_0.protocol = true

# 默认监听 5672 端口同时支持两种协议
# 也可以指定独立端口
listeners.amqp_1_0 = 5673
```

**代码示例（Java 客户端）：**

```java
import org.apache.qpid.jms.JmsConnectionFactory;
import javax.jms.*;

// AMQP 1.0 连接
ConnectionFactory factory = new JmsConnectionFactory(
    "amqp://guest:guest@localhost:5673");
try (Connection conn = factory.createConnection();
     Session session = conn.createSession(false, Session.AUTO_ACKNOWLEDGE)) {
    
    Queue queue = session.createQueue("my-queue");
    MessageProducer producer = session.createProducer(queue);
    producer.send(session.createTextMessage("Hello AMQP 1.0"));
}
```

**注意：** AMQP 1.0 与 AMQP 0-9-1 的队列可**互操作**，但概念不同（如 AMQP 1.0 的 `link` 对应 AMQP 0-9-1 的 `consumer`）。

## 流队列（Stream Queue）

流队列是 RabbitMQ 4.0 的核心新功能，支持**追加写、非破坏性消费、消息回溯**。

### 特性
- 消息不会被消费后删除（非破坏性）
- 支持从任意偏移量读取
- 适合日志收集、事件溯源、审计追踪
- 高吞吐写入，但延迟略高于经典队列

### 创建流队列

```bash
# CLI 创建
rabbitmqadmin declare queue name=my-stream queue_type=stream

# 或通过 Management UI 创建时选择 Stream 类型
```

```java
// Java 客户端（AMQP 0-9-1）
 ConnectionFactory factory = new ConnectionFactory();
 factory.setHost("localhost");
 try (Connection conn = factory.newConnection();
      Channel ch = conn.createChannel()) {
     
     Map<String, Object> args = new HashMap<>();
     // 设置流队列
     args.put("x-queue-type", "stream");
     // 最大保留大小（字节）
     args.put("x-max-length-bytes", 10 * 1024 * 1024 * 1024L); // 10GB
     // 消息最大存活时间
     args.put("x-max-age", "7d"); // 7天
     
     ch.queueDeclare("my-stream", true, false, false, args);
 }
```

### 消费流队列

```java
// 流队列需要指定 offset
Map<String, Object> consumerArgs = new HashMap<>();
// "first" - 从头读取, "last" - 从最新读取
// "next" - 仅新消息, 数字 - 指定偏移量
consumerArgs.put("x-stream-offset", "first");

ch.basicConsume("my-stream", true, "my-consumer", 
    consumerArgs, (consumerTag, message) -> {
        System.out.println("Received: " + new String(message.getBody()));
    }, consumerTag -> {});
```

## 仲裁队列改进

仲裁队列在 4.0 中进行了以下优化：

- **动态目标集群大小**：不再固定为节点数，可指定副本数
- **更快的领导选举**：改进 Raft 实现，故障切换更快
- **降低内存占用**：优化 WAL 写入，减少内存开销
- **批量确认优化**：大吞吐场景性能提升约 30%

```java
// 创建仲裁队列
Map<String, Object> args = new HashMap<>();
args.put("x-queue-type", "quorum");
args.put("x-quorum-initial-group-size", 3); // 3 副本
args.put("x-max-in-memory-length", 10000); // 内存中最大消息数

channel.queueDeclare("quorum-queue", true, false, false, args);
```

## 经典队列 v2（Classic Queue v2）

经典队列 v2 在 4.0 中成为默认版本，采用新存储引擎：

```bash
# 显式指定 v2 版本
rabbitmqadmin declare queue name=cq-v2 queue_type=classic arguments='{"x-queue-version": 2}'
```

**v2 改进：**
- 新的索引存储格式，减少磁盘 I/O
- 惰性队列（Lazy Queue）成为默认行为（消息直接写入磁盘）
- 更快重启恢复

## 注意事项

1. **版本兼容**：从 3.x 升级到 4.0 前确保所有插件兼容，移除 `rabbitmq_amqp1_0` 插件（已内置）
2. **经典队列 v1 将移除**：RabbitMQ 4.x 已移除经典队列 v1 版本，升级前需将所有经典队列迁移到 v2
3. **流队列的消费者模型**：流队列是**非破坏性消费**，多个消费者可独立消费同一消息，适合广播模式
4. **流队列延迟较高**：不适合需要低延迟的即时消息场景，适合高吞吐、大容量场景
5. **AMQP 1.0 与 AMQP 0-9-1**：虽然互相可见，但 routing key、exchange 等概念的使用方式不同，需要适配层
6. **性能基准**：流队列吞吐 ≈ 经典队列的 2-3 倍，但延迟增加约 30-50ms
