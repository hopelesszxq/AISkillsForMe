---
name: rocketmq5-new-features
description: RocketMQ 5.x 新架构特性：DLedger Controller、gRPC 协议、多语言轻量级客户端、流式分区
tags: [rocketmq, mq, messaging, streaming, grpc, ha]
---

## 概述

RocketMQ 5.x 是架构层面的大版本升级（当前最新 5.2.0/5.3.0）。核心变化：引入 **DLedger Controller** 替代原 NameServer 主从切换、原生支持 **gRPC 协议**、提供多语言轻量级客户端、支持 **流式分区**（Streaming）、内置 **MQTT** 协议等。

## 1. DLedger Controller（高可用架构）

### 功能说明

RocketMQ 5.x 引入 **DLedger Controller** 模式（基于 Raft 共识算法），替代 4.x 时代的异步/同步主从复制 + NameServer 的自动主从切换，实现了 Broker 自动选主、故障秒级切换。

```
4.x 架构：NameServer（无状态路由） + Broker（Master-Slave 手动切换）
5.x 架构：DLedger Controller（Raft 选主） + Broker（自动选主、自动切换）
```

### 部署配置

```shell
# 1. 启动 Controller（3 节点组成 Raft 集群）
nohup sh bin/mqcontroller -c conf/controller/cluster-1.conf &

# 2. 启动 Broker（自动注册到 Controller）
nohup sh bin/mqbroker -c conf/broker-controller.conf &

# 3. 验证集群状态
sh bin/mqadmin clusterList -n localhost:9876
```

### broker-controller.conf 关键配置

```properties
# 启用 Controller 模式
enableControllerMode=true
# Controller 地址列表
controllerAddr=192.168.1.1:9878;192.168.1.2:9878;192.168.1.3:9878
# 同步复制模式（确保高可用）
brokerRole=SYNC_MASTER
# 刷盘方式
flushDiskType=ASYNC_FLUSH
```

## 2. gRPC 协议与轻量级客户端

### 功能说明

RocketMQ 5.x 原生支持 **gRPC 协议**，替代 4.x 的私有 TCP 协议。提供多语言轻量级客户端（Go、Python、C++、Node.js、Rust），不再依赖 Java 环境。

### Java 客户端（gRPC 模式）

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client-java</artifactId>
    <version>5.0.7</version>
</dependency>
```

```java
// 生产者 - gRPC 方式
ClientServiceProvider provider = ClientServiceProvider.loadStatic();
Message message = provider.newMessageBuilder()
    .setTopic("order-topic")
    .setBody(ByteString.copyFromUtf8("Hello RocketMQ 5.x"))
    .build();

SendReceipt receipt = provider.send(message);
System.out.println("Send success: " + receipt.getMessageId());
```

```java
// 消费者 - gRPC 方式
SimpleConsumer consumer = provider.newSimpleConsumerBuilder()
    .setClientConfiguration(ClientConfiguration.newBuilder()
        .setEndpoints("localhost:8081")  // gRPC 端口
        .build())
    .setSubscription(Subscription.newBuilder()
        .setTopic("order-topic")
        .setExpression(FilterExpression.SUB_ALL)
        .build())
    .build();

while (true) {
    List<MessageView> messages = consumer.receive(10, Duration.ofSeconds(30));
    for (MessageView msg : messages) {
        System.out.println("Received: " + msg.getBody().toStringUtf8());
        consumer.ack(msg.getMessageId());
    }
}
```

### 多语言客户端示例（Go）

```go
import "github.com/apache/rocketmq-clients/golang"

producer, _ := golang.NewProducer(&golang.Config{
    Endpoint: "localhost:8081",
})
producer.Start()
defer producer.Close()

// 发送消息
msg := &golang.Message{
    Topic: "order-topic",
    Body:  []byte("Hello from Go"),
}
_, _ = producer.Send(context.Background(), msg)
```

## 3. 多协议支持

### 支持协议一览

| 协议 | 用途 | 端口 |
|---|---|---|
| gRPC | 多语言客户端、轻量级 SDK | 8081 |
| Remoting（私有 TCP） | 兼容 4.x 客户端 | 10911 |
| MQTT | IoT 设备消息 | 1883 |
| HTTP | RESTful 接口 | 8080 |

### 启用 MQTT 桥接

```xml
<!-- 引入 MQTT 模块 -->
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-mqtt-broker</artifactId>
    <version>5.2.0</version>
</dependency>
```

```shell
# 启动 MQTT 桥接 Broker
nohup sh bin/mqbroker -c conf/mqtt-broker.conf &
```

## 4. 流式分区（Streaming）

### 功能说明

RocketMQ 5.x 内置**轻量级流处理引擎**，支持在队列上做流式聚合计算，类似 Kafka Streams 但更轻量。

```java
// 流式消费者（支持 exactly-once 语义）
StreamConsumer consumer = StreamConsumerBuilder.newBuilder()
    .setGroup("stream-group")
    .setTopic("stream-topic")
    .setWindow(Window.of(Duration.ofMinutes(5)))
    .setProcessor((key, values) -> {
        double avg = values.stream()
            .mapToDouble(v -> v.getBody().toDouble())
            .average()
            .orElse(0);
        return StreamResult.of("avg", String.valueOf(avg));
    })
    .build();
```

## 5. 事务消息改进

RocketMQ 5.x 对事务消息做了性能优化：

```java
// 5.x 事务消息 API（简化）
producer.sendMessageInTransaction(msg, new TransactionChecker() {
    @Override
    public LocalTransactionState check(MessageView messageView) {
        // 回查业务状态
        boolean success = checkOrderStatus(messageView.getBody());
        return success ? LocalTransactionState.COMMIT : LocalTransactionState.ROLLBACK;
    }
});
```

## 注意事项

1. **gRPC 协议端口不同**：5.x 新增 8081（gRPC），旧 Remoting 协议仍在 10911，迁移时注意客户端配置
2. **Controller 最少 3 节点**：DLedger Controller 基于 Raft，至少部署 3 个实例才能容忍 1 台故障
3. **兼容模式**：5.x Broker 可以同时服务 4.x 和 5.x 客户端（Remoting 端口兼容），但 5.x 新特性（gRPC、Streaming）需新客户端
4. **性能基准**：官方测试 gRPC 客户端相比 4.x Remoting 协议有约 10-20% 的额外延迟，适合多语言场景而非极致性能场景
5. **MQTT 协议** 需额外配置桥接 Broker，不在默认启动范围内
6. **升级路径**：4.x → 5.x 推荐使用滚动升级（先升级 Broker，再升级客户端），NameServer 可逐步替换为 Controller
