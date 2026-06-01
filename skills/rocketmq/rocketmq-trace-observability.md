---
name: rocketmq-trace-observability
description: RocketMQ 消息轨迹（Trace）与可观测性：Trace 原理、OpenTelemetry 集成、监控指标与生产实践
tags: [rocketmq, mq, trace, observability, opentelemetry, monitoring]
---

## 概述

RocketMQ 消息轨迹（Message Trace）是追踪消息从生产到消费全生命周期流转的关键能力，包括消息发送、存储、消费确认、重试、死信等环节。结合 OpenTelemetry 可实现端到端分布式链路追踪。

## 1. 消息轨迹（Message Trace）原理

### 轨迹流程

```
Producer → Broker(写轨迹) → Consumer A(消费) → Broker(记录消费状态)
                               ↓ (失败重试)
                            Consumer A(重试)
                               ↓ (超过次数)
                            死信队列(DLQ)
```

每条消息在以下时机生成追踪事件：
- **pub**：消息发送到 Broker
- **sub**：消息投递给消费者
- **consume**：消费完成（成功/失败）

### 启用轨迹

```properties
# broker.conf — 开启消息轨迹
traceTopicEnable=true
# 默认轨迹 Topic 名称为 RMQ_SYS_TRACE_TOPIC
# 可自定义
traceTopicName=my_trace_topic
```

```java
// 生产者启用轨迹
DefaultMQProducer producer = new DefaultMQProducer("producer_group");
producer.setNamesrvAddr("localhost:9876");
producer.setEnableMsgTrace(true);              // 开启轨迹
producer.setCustomizedTraceTopic("my_trace_topic"); // 自定义轨迹 Topic
producer.start();

// 消费者启用轨迹
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("consumer_group");
consumer.setNamesrvAddr("localhost:9876");
consumer.setEnableMsgTrace(true);
consumer.setCustomizedTraceTopic("my_trace_topic");
consumer.start();
```

### Spring Boot 集成

```yaml
rocketmq:
  name-server: localhost:9876
  producer:
    group: my-producer-group
    enable-msg-trace: true       # 生产者轨迹
  consumer:
    group: my-consumer-group
    enable-msg-trace: true       # 消费者轨迹
```

## 2. 轨迹数据查询

### 控制台查看

RocketMQ Dashboard（5.x）提供消息轨迹查询界面：

1. 进入 **消息** → **消息查询** → 按 Topic/Message ID/Key 搜索
2. 点击搜索结果中的 **消息轨迹** 按钮
3. 查看完整的生产→存储→消费链路

### 命令行查询

```bash
# 按消息 ID 查询轨迹
mqadmin queryMsgTraceById -n localhost:9876 -i 0A1B2C3D4E5F6G7H8I9J0K

# 按消息 Key 查询
mqadmin queryMsgByKey -n localhost:9876 -t my_topic -k my_key

# 查看轨迹 Topic 的消息（调试用）
mqadmin queryMsgByKey -n localhost:9876 -t RMQ_SYS_TRACE_TOPIC -k <original_msg_id>
```

### 轨迹数据结构

轨迹消息体 JSON 格式：

```json
{
  "traceBeanList": [
    {
      "topic": "order_topic",
      "msgId": "0A1B2C3D4E5F6G7H8I9J0K",
      "tags": "VIP",
      "keys": "order_12345",
      "storeHost": "192.168.1.10:10911",
      "clientHost": "192.168.1.20",
      "costTime": 15,
      "status": 0,
      "time": 1717200000000
    }
  ],
  "subList": [
    {
      "cell": "consumer_group_name",
      "pubTime": "2026-06-01 10:00:00.000",
      "requestId": "ABC123"
    }
  ]
}
```

## 3. OpenTelemetry 集成

### 依赖引入

```xml
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
    <version>1.40.0</version>
</dependency>
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-rocketmq-client-5.x</artifactId>
    <version>2.8.0-alpha</version>
</dependency>
```

### 生产者链路

```java
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Scope;

@Autowired
private Tracer tracer;

public void sendOrderMessage(Order order) {
    Span span = tracer.spanBuilder("rocketmq.send")
        .setAttribute("messaging.system", "rocketmq")
        .setAttribute("messaging.destination", "order_topic")
        .setAttribute("messaging.message_id", order.getOrderId())
        .startSpan();

    try (Scope ignored = span.makeCurrent()) {
        Message message = new Message("order_topic", 
            JSON.toJSONString(order).getBytes(StandardCharsets.UTF_8));
        message.setKeys(order.getOrderId());
        
        SendResult result = producer.send(message);
        
        span.setAttribute("messaging.send_result", result.getSendStatus().name());
        span.setStatus(StatusCode.OK);
    } catch (Exception e) {
        span.recordException(e);
        span.setStatus(StatusCode.ERROR, e.getMessage());
        throw e;
    } finally {
        span.end();
    }
}
```

### 消费者链路

```java
@RocketMQMessageListener(topic = "order_topic", 
                         consumerGroup = "order_consumer_group")
public class OrderConsumer implements RocketMQListener<MessageExt> {

    @Autowired
    private Tracer tracer;

    @Override
    public void onMessage(MessageExt message) {
        // 从消息属性中提取 trace context
        Span span = tracer.spanBuilder("rocketmq.consume")
            .setAttribute("messaging.system", "rocketmq")
            .setAttribute("messaging.destination", message.getTopic())
            .setAttribute("messaging.message_id", message.getMsgId())
            .setAttribute("messaging.broker", message.getStoreHost().toString())
            .startSpan();

        try (Scope ignored = span.makeCurrent()) {
            // 业务处理
            processOrder(message);
            span.setStatus(StatusCode.OK);
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e; // RocketMQ 会重试
        } finally {
            span.end();
        }
    }
}
```

### gRPC 协议 Trace Propagation

RocketMQ 5.x gRPC 客户端自动支持 OpenTelemetry Context Propagation：

```yaml
# 开启 gRPC 链路传播
rocketmq:
  client:
    trace:
      enabled: true
      exporter: otlp           # OpenTelemetry Protocol
      endpoint: http://localhost:4318
```

## 4. 监控指标与 Prometheus

### RocketMQ-Exporter

```yaml
# docker-compose 部署 RocketMQ Exporter
services:
  rocketmq-exporter:
    image: apache/rocketmq-exporter:0.0.2
    ports:
      - 5557:5557
    environment:
      - ROCKETMQ_NAMESRV_ADDR=localhost:9876
      - ROCKETMQ_EXPORTER_PORT=5557
    restart: always
```

### 关键 Prometheus 指标

| 指标名 | 类型 | 说明 |
|--------|------|------|
| `rocketmq_producer_tps` | Gauge | 生产者 TPS |
| `rocketmq_consumer_tps` | Gauge | 消费者 TPS |
| `rocketmq_consumer_diff` | Gauge | 消费堆积量 |
| `rocketmq_msg_accumulation` | Gauge | 消息堆积总数 |
| `rocketmq_broker_commitlog_disk_ratio` | Gauge | CommitLog 磁盘使用率 |
| `rocketmq_send_timer` | Summary | 发送耗时分布 |
| `rocketmq_consume_timer` | Summary | 消费耗时分布 |

### Grafana 告警规则

```yaml
# prometheus-alerts.yml
groups:
  - name: rocketmq
    rules:
      # 消费堆积告警
      - alert: RocketMQMessageAccumulation
        expr: rocketmq_consumer_diff > 100000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "RocketMQ 消费堆积 {{ $value }} 条"
      
      # Broker 磁盘告警
      - alert: RocketMQBrokerDiskFull
        expr: rocketmq_broker_commitlog_disk_ratio > 0.85
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "RocketMQ Broker 磁盘使用率 {{ $value | humanizePercentage }}"
      
      # 消费耗时过长
      - alert: RocketMQConsumeSlow
        expr: rocketmq_consume_timer{quantile="0.99"} > 5000
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "RocketMQ P99 消费耗时超过 5s"
```

## 5. 生产实践

### 5.1 轨迹采样率控制

高吞吐场景下全量轨迹可能影响性能：

```java
// 按比例采样（如 10%）
public class SamplingTraceDispatcher extends TraceDispatcher {

    private final double samplingRate; // 0.0 ~ 1.0
    private final Random random = new Random();

    public SamplingTraceDispatcher(double samplingRate) {
        this.samplingRate = samplingRate;
    }

    @Override
    public void append(TraceContext ctx) {
        if (random.nextDouble() < samplingRate) {
            super.append(ctx);
        }
    }
}
```

### 5.2 轨迹 Topic 隔离

```properties
# 生产环境建议独立的轨迹 Topic
# 避免轨迹数据影响业务 Topic 性能
traceTopicName=RMQ_SYS_TRACE_PROD  # 生产环境
# traceTopicName=RMQ_SYS_TRACE_STAG # 预发环境

# 轨迹 Topic 保留时长可短于业务 Topic
# broker.conf
traceTopicRetentionHours=48  # 轨迹只保留 48 小时
```

### 5.3 轨迹与业务 Key 关联

```java
// 统一消息 Key 生成规范，便于按业务维度检索
Message message = new Message("order_topic", body);
message.setKeys(String.join(" ",
    order.getOrderId(),          // 订单 ID
    String.valueOf(order.getUserId()),  // 用户 ID
    order.getBusinessType()      // 业务类型
));

// 查询轨迹时可用任意 Key 搜索
// mqadmin queryMsgByKey -t order_topic -k "order_12345"
// mqadmin queryMsgByKey -t order_topic -k "user_67890"
```

### 5.4 消费链路日志埋点

```java
@RocketMQMessageListener(topic = "order_topic",
                         consumerGroup = "order_group")
public class TraceableConsumer implements RocketMQListener<MessageExt> {

    private static final Logger log = LoggerFactory.getLogger(TraceableConsumer.class);

    @Override
    public void onMessage(MessageExt msg) {
        long start = System.currentTimeMillis();
        String msgId = msg.getMsgId();
        int reconsumeTimes = msg.getReconsumeTimes();
        
        log.info("MSG_TRACE|开始消费|msgId={}|reconsumeTimes={}|broker={}",
            msgId, reconsumeTimes, msg.getStoreHost());

        try {
            // 业务处理
            process(msg);
            long cost = System.currentTimeMillis() - start;
            log.info("MSG_TRACE|消费成功|msgId={}|cost={}ms", msgId, cost);
        } catch (Exception e) {
            long cost = System.currentTimeMillis() - start;
            log.warn("MSG_TRACE|消费失败|msgId={}|cost={}ms|error={}",
                msgId, cost, e.getMessage());
            throw e;
        }
    }
}
```

## 注意事项

| 要点 | 说明 |
|------|------|
| **轨迹不影响业务一致性** | 轨迹记录是异步写入的，不会影响业务消息的主流程 |
| **轨迹 Topic 需有足够分区** | 高并发场景建议轨迹 Topic 分区数 ≥ 8 |
| **采样率与性能权衡** | 全量轨迹额外增加约 5-10% 写路径开销；建议高吞吐场景设采样率 10-50% |
| **轨迹数据生命周期** | 轨迹 Topic 的保留时间应独立设置，生产环境建议 24-48h |
| **OpenTelemetry 版本兼容** | opentelemetry-rocketmq-client 需要匹配 RocketMQ 5.x 的 gRPC 协议版本 |
| **Broker 集群轨迹一致性** | 多 Broker 部署时确保所有 Broker `traceTopicEnable=true` |
| **安全注意** | 轨迹数据可能包含消息体元信息，注意脱敏处理 |
