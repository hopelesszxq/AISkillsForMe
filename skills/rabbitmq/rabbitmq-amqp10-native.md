---
name: rabbitmq-amqp10-native
description: RabbitMQ 4.x AMQP 1.0 原生协议深度解析 — 连接模型、链路管理、功能对比与 AMQP 0-9-1 迁移策略
tags: [rabbitmq, amqp-1.0, protocol, migration, interconnect]
---

## 概述

RabbitMQ 4.0+ 将 **AMQP 1.0** 提升为**一等公民协议**，原生支持 AMQP 1.0 连接（不再需要插件适配）。RabbitMQ 4.x 同时兼容 AMQP 0-9-1 和 AMQP 1.0，两者共享相同的虚拟主机（vhost）、权限模型和部分运维工具。

> RabbitMQ 3.x 通过 `rabbitmq_amqp1_0` 插件提供有限 AMQP 1.0 支持；4.0+ 移除了该插件，改由 Broker 原生实现 AMQP 1.0。

## AMQP 1.0 vs AMQP 0-9-1 核心差异

| 特性 | AMQP 0-9-1 | AMQP 1.0 |
|------|-----------|----------|
| **模型** | Broker 中心化（Exchange/Queue/Binding） | 对等模型（Source/Target/Node） |
| **连接** | 单一连接 + 多个 Channel | 单一连接 + 多个 Session + 多个 Link |
| **路由** | Exchange → Binding → Queue | Source/Terminus 直接映射 |
| **事务** | Tx Select/Commit/Rollback | 无内置事务，依赖分布式协调 |
| **流控** | Channel 级 credit 流控 | Link 级 flow control（更精细） |
| **安全** | SASL + TLS | SASL + TLS + TLS SNI 扩展 |
| **多宿主** | 需手动连接 | 支持多宿主连接（一条连接多个 vhost） |
| **消息确认** | Basic.Ack/Nack（单条确认） | Disposition（可批量确认） |
| **协议复杂性** | 较低，约定固定语义 | 较高，更灵活但学习曲线陡 |

## 2. 连接模型对比

### AMQP 0-9-1 连接模型

```
Connection
  └── Channel 1 (Consumer)
  └── Channel 2 (Publisher)
  └── Channel 3 (事务操作)
```

### AMQP 1.0 连接模型

```
Connection (多宿主)
  └── Session 1 (vhost: /)
      ├── Link 1: consumer (source=topic://order.*, target=orders-queue)
      ├── Link 2: publisher (target=/exchange/default/routing-key)
      └── Link 3: receiver (settled=false, auto-ack)
  └── Session 2 (vhost: /analytics)
      └── Link 4: stream-consumer
```

**关键区别**：
- AMQP 1.0 的 **Session** 对应 0-9-1 的 Channel，但更轻量
- AMQP 1.0 的 **Link** 表示单一方向的消息流（sender/receiver）
- 一条 AMQP 1.0 连接可跨越多个 vhost（无需重新建立 TCP 连接）

## 3. 地址映射关系

RabbitMQ 4.x 为 AMQP 1.0 定义了标准的地址格式，用来映射到 0-9-1 的 Exchange/Queue：

| AMQP 1.0 地址模式 | 对应 0-9-1 概念 | 说明 |
|-------------------|----------------|------|
| `/queue/{name}` | Queue | 直接访问队列 |
| `/exchange/{name}` | Exchange | 访问 exchange，需指定路由键 |
| `/exchange/{name}/{routing-key}` | Exchange + Routing Key | 带路由键的 exchange 访问 |
| `/topic/{name}` | Topic Exchange 简写 | 等效 `/exchange/amq.topic/{name}` |
| `/amq/queue/{name}` | 自动创建队列 | 类似匿名 queue |

### 生产者示例

```java
// Qpid JMS AMQP 1.0 客户端 — 发送消息
import org.apache.qpid.jms.JmsConnectionFactory;
import jakarta.jms.*;

ConnectionFactory factory = new JmsConnectionFactory(
    "amqp://rabbitmq-host:5672"
);
try (Connection conn = factory.createConnection()) {
    Session session = conn.createSession(false, Session.AUTO_ACKNOWLEDGE);
    // 通过 exchange 发送
    Destination dest = session.createQueue(
        "/exchange/orders/order.created"   // Exchange + RoutingKey 映射
    );
    MessageProducer producer = session.createProducer(dest);
    producer.send(session.createTextMessage("{\"orderId\": \"12345\"}"));
}
```

### 消费者示例

```java
// 消费队列
Destination dest = session.createQueue("/queue/order.queue");
MessageConsumer consumer = session.createConsumer(dest);
consumer.setMessageListener(message -> {
    System.out.println("收到: " + ((TextMessage) message).getText());
});
```

## 4. Spring Boot AMQP 1.0 配置

```xml
<dependency>
    <groupId>org.springframework.amqp</groupId>
    <artifactId>spring-rabbit-stream</artifactId>  <!-- AMQP 1.0 支持 -->
</dependency>
```

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    amqp10: true              # 启用 AMQP 1.0 协议
    virtual-host: /
    # 0-9-1 和 1.0 共享连接池
    connection-timeout: 5000
```

```java
@Configuration
public class RabbitAmqp10Config {

    @Bean
    public AmqpTemplate amqp10Template(
            ConnectionFactory connectionFactory) {
        return new RabbitTemplate(connectionFactory);
    }

    @Bean
    public MessageConverter messageConverter() {
        // AMQP 1.0 推荐使用 JSON 序列化
        return new Jackson2JsonMessageConverter();
    }
}
```

> 注意：`spring-rabbit-stream` 模块基于 AMQP 1.0 实现 RabbitMQ Stream 的消费，也适用于普通 AMQP 1.0 消息的收发。

## 5. AMQP 1.0 流控机制

AMQP 1.0 使用 **Link Credit** 机制控制消息流量：

```java
// 手动控制 Credit（流控反压）
Receiver receiver = connection.createReceiver(
    session, "/queue/big-queue"
);

// 初始只给 10 个 credit——Broker 不会发送超过 10 条消息
receiver.setCredit(10);

// 处理完消息后增加 credit
receiver.getMessage((message) -> {
    process(message);
    // 处理完一条增加一个 credit
    receiver.addCredit(1);
});
```

**与 0-9-1 QoS 预取值对比**：

| 特性 | AMQP 0-9-1 Basic.QoS | AMQP 1.0 Link Credit |
|------|---------------------|---------------------|
| 作用域 | Channel 级别 | Link 级别 |
| 流控粒度 | 预取值 | 动态 credit 分配 |
| 反压能力 | 被动（Broker 推送） | 主动（Consumer 控制） |
| 批量 | 限制未确认消息数 | 限制未发送消息数 |

## 6. 迁移策略：从 AMQP 0-9-1 到 AMQP 1.0

### 渐进迁移步骤

```
Phase 1: 共存期
  ┌─────────────┐    ┌─────────────┐
  │ 旧服务 (0-9-1) │    │ 新服务 (1.0)  │
  │ 生产/消费    │    │ 生产/消费    │
  └──────┬──────┘    └──────┬──────┘
         │                   │
         └─────── RabbitMQ ──┘
         Exchange/Queue 由两边共享
```

```
Phase 2: 协议转换
  使用 RabbitMQ Shovel 或 Federation 在 0-9-1 ↔ 1.0 之间桥接
```

```bash
# 启用 Shovel 进行协议转换
rabbitmq-plugins enable rabbitmq_shovel rabbitmq_shovel_management
```

```json
// 通过 Management API 创建 Shovel（0-9-1 → 1.0）
{
  "value": {
    "src-uri": "amqp://localhost:5672",
    "src-protocol": "amqp091",
    "src-queue": "source-queue",
    "dest-uri": "amqp://localhost:5672",
    "dest-protocol": "amqp10",
    "dest-address": "/queue/target-queue"
  }
}
```

### 注意事项

1. **功能不对称**：AMQP 1.0 不支持直接操作 Exchange/Binding，需通过 Management API 或 0-9-1 客户端管理
2. **延迟消息**：AMQP 1.0 发送延迟消息需通过 `x-delay` 消息注解，与 `rabbitmq_delayed_message_exchange` 兼容
3. **事务消息**：AMQP 1.0 无事务语义，需用 Publisher Confirm + 幂等消费替代
4. **监控差异**：`rabbitmq-diagnostics` 命令对 AMQP 1.0 连接显示为 `amqp10` 类型，统计项有所不同

## 注意事项

1. **不要混用协议连接同一 Channel/Queue 的写入**：Broker 内部会将 0-9-1 和 1.0 消息统一调度，但客户端行为差异可能导致预取值混乱
2. **AMQP 1.0 客户端推荐**：Apache Qpid JMS（Java）、Rhea（Node.js）、C Proton（C/Python）。Spring Boot 3.x 推荐使用 `spring-rabbit-stream` 的 ConnectionFactory
3. **虚拟主机隔离**：AMQP 1.0 支持跨 vhost 连接，需谨慎配置权限控制
4. **批量确认**：AMQP 1.0 的 Disposition 支持批量确认（`synchronized` 模式），减少网络往返
5. **Stream 消费**：RabbitMQ Stream 底层使用 AMQP 1.0 协议，消费 Stream 时只能用 AMQP 1.0 客户端
