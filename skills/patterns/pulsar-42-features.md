---
name: pulsar-42-features
description: Apache Pulsar 4.2 新特性：OpenTelemetry 原生追踪、精细化消息投递延迟策略、Functions 事务支持、可自定义 Prometheus 标签
tags: [patterns, pulsar, messaging, event-streaming, opentelemetry, distributed-systems]
---

## 概述

Apache Pulsar 4.2.0（2026-04-01 发布，4.2.2 于 2026-06-08 发布）是 4.x 系列的重要功能版本。核心改进：**OpenTelemetry 原生追踪**、**精细化消息投递延迟策略**、**Pulsar Functions 事务封装**，以及**可自定义 Prometheus 标签**。

## 一、OpenTelemetry 原生追踪（PIP-446）

Pulsar Java Client 现在原生支持 OpenTelemetry Tracing，无需额外 agent 或手动埋点。

### 配置方式

```yaml
# broker.conf
pulsar.opentelemetry.enabled=true
```

```java
// Java Client — 自动集成 OTel
import org.apache.pulsar.client.api.PulsarClient;
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.sdk.OpenTelemetrySdk;

// 方式一：自动发现（推荐）
PulsarClient client = PulsarClient.builder()
    .serviceUrl("pulsar://localhost:6650")
    .build();  // 自动从全局 OTel SDK 读取配置

// 方式二：手动传入
OpenTelemetry otel = OpenTelemetrySdk.builder().build();
PulsarClient client = PulsarClient.builder()
    .serviceUrl("pulsar://localhost:6650")
    .openTelemetry(otel)
    .build();
```

### 追踪的 Span 覆盖范围

| 操作 | Span 名称 | 属性 |
|------|-----------|------|
| 消息发送 | `pulsar.producer.send` | topic, partition, messageId |
| 消息接收 | `pulsar.consumer.receive` | topic, subscription, messageId |
| 消息确认 | `pulsar.consumer.ack` | topic, subscription, msgId |
| 批量发送 | `pulsar.producer.batch` | batchSize, topic |
| 延迟投递 | `pulsar.producer.delayed` | delayDuration, topic |

与此同时，Pulsar 4.2 升级了 OTel 版本至 1.56.0，OTel Instrumentation 至 2.21.0，语义约定至 1.37.0，并新增了发布延迟直方图、ML 写入延迟直方图等 OTel 指标。

## 二、精细化消息投递延迟策略（PIP-437）

Pulsar 4.2 支持更灵活的消息延迟投递策略，不再局限于固定的延迟级别。

### 配置示例

```java
// Producer — 设置消息延迟投递
producer.newMessage()
    .value("delayed payload".getBytes())
    .deliverAfter(30, TimeUnit.SECONDS)   // 相对延迟
    .deliverAt(System.currentTimeMillis() + 60000)  // 绝对时间戳
    .send();

// Consumer — 无需特殊配置，自动接收到期消息
consumer.receiveAsync().thenAccept(msg -> {
    System.out.println("Received: " + new String(msg.getData()));
    consumer.acknowledge(msg);
});
```

### 应用场景

- **订单超时取消**：消息延迟 30 分钟投递进行超时检测
- **定时任务调度**：延迟到指定时间点执行
- **重试队列**：指数退避延迟重试

## 三、Pulsar Functions 事务支持（PIP-439）

Pulsar Functions 现在支持通过**托管事务包装**实现 Exactly-Once 语义。

```java
import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;
import org.apache.pulsar.client.api.transaction.Transaction;

public class TransactionalFunction implements Function<String, String> {

    @Override
    public String process(String input, Context ctx) throws Exception {
        // Functions 框架自动管理事务生命周期
        Transaction txn = ctx.getTransaction();  // 获取当前事务
        
        // 处理逻辑
        String output = transform(input);
        
        // 写入多个 topic — 全部成功或全部失败
        ctx.newOutputMessage("output-topic", Schema.STRING)
            .value(output)
            .send();  // 自动参与事务
        
        // 函数返回后事务自动提交
        return output;
    }
}
```

### 配置

```yaml
# functions worker 配置
functionTransactionEnabled: true
# 事务超时（默认 10 分钟）
transactionTimeoutMs: 600000
```

## 四、可自定义 Prometheus 标签（PIP-447）

允许为 Topic 级别指标配置自定义 Prometheus 标签，提升监控可观测性。

```yaml
# broker.conf
# 格式：label=source
pulsar.customTopicMetricsLabels=cluster=\${cluster},env=\${env},dc=\${dc}
```

```yaml
# 支持的指标源
# static: 静态字符串值
# env: 环境变量
# property: 命名空间属性
pulsar.customTopicMetricsLabels=region=static:cn-east,env=env:DEPLOY_ENV,team=property:team
```

### 效果

Prometheus 指标将附加这些自定义标签：

```
pulsar_rate_in{...,cluster="prod",env="staging",dc="shanghai"} 100.0
pulsar_storage_size{...,cluster="prod",env="staging",dc="shanghai"} 2048
```

## 五、元数据存储迁移框架（PIP-454）

Pulsar 4.2 引入了元数据存储迁移框架，支持在运行时将元数据从一种存储后端迁移到另一种。

### 支持场景

| 迁移路径 | 说明 |
|----------|------|
| ZooKeeper → Etcd | 替换 ZK 为 Etcd |
| ZooKeeper → Raft | 使用内置 Raft 替代外部 ZK |
| 单副本 → 多副本 | 扩展元数据高可用 |

```yaml
# broker.conf
metadataStoreMigration:
  enabled: true
  source: zk:localhost:2181
  target: raft:localhost:6666
  batchSize: 100
  continueOnError: false
```

## 六、Netty 背压控制（PIP-434）

暴露 Netty 通道的 WRITE_BUFFER_WATER_MARK 配置，防止背压时 OOM。

```yaml
# broker.conf
nettyWriteBufferLowWaterMark: 32768    # 32KB
nettyWriteBufferHighWaterMark: 65536   # 64KB
# 超过 highWaterMark 时暂停接收请求
channelUnwritableAction: PAUSE_RECEIVE
```

## 七、升级注意事项

1. **OTel 依赖冲突**：如果项目中已有 OTel SDK，确保版本 ≥ 1.56.0
2. **Pulsar Functions 事务**：需要开启 `functionTransactionEnabled=true`，会增加性能开销
3. **自定义 Prometheus 标签**：标签数量过多会增加 Prometheus 存储成本，建议 ≤ 5 个
4. **元数据迁移**：生产环境建议先在小规模集群验证
5. **Pulsar 4.2.x** 最新补丁：4.2.2 包含安全漏洞修复（async-http-client、Netty 升级）

```xml
<!-- Maven 依赖 -->
<dependency>
    <groupId>org.apache.pulsar</groupId>
    <artifactId>pulsar-client</artifactId>
    <version>4.2.2</version>
</dependency>
```
