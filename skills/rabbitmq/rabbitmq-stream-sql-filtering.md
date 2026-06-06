---
name: rabbitmq-stream-sql-filtering
description: RabbitMQ 4.2+ 流式 SQL 过滤 — Broker 端按 SQL 条件过滤消息，Bloom + SQL 两级加速
tags: [rabbitmq, streams, amqp-1.0, filtering, sql, performance]
---

## 要点

RabbitMQ 4.2 引入**流（Stream）的 SQL 过滤表达式**，在 Broker 端完成消息过滤，大幅减少网络传输和客户端处理开销。

### 两级过滤架构

| 阶段 | 机制 | 作用 |
|------|------|------|
| Stage 1 | **Bloom Filter**（`x-stream-filter-value` 注解） | 跳过不包含目标值的完整 Chunk，减少磁盘 I/O |
| Stage 2 | **SQL Filter Expression** | 对剩余消息执行精确逐条过滤 |

**性能基准**：高度选择性场景下（10M 条消息仅匹配 10 条），纯 SQL 过滤可达 40 万 msg/s，Bloom + SQL 组合可达 **480 万 msg/s**，比客户端过滤快数个数量级。

## 代码示例

### 1. 生产者：设置 Bloom filter 值 + 业务属性

```java
// 每条消息设置 Bloom filter 值和业务属性
publisher
    .message(body.getBytes(StandardCharsets.UTF_8))
    .priority(priority)
    // Stage 1: Bloom filter 值
    .annotation("x-stream-filter-value", eventType)
    .subject(eventType)
    .creationTime(creationTime)
    // 业务属性，用于 SQL 过滤
    .property("region", region);
```

### 2. 消费者：定义 SQL 过滤条件

```java
// 复杂过滤条件示例：高价值订单
String SQL =
    "properties.subject = 'order.created' AND " +
    "properties.creation_time > UTC() - 3600000 AND " +
    "region IN ('AMER', 'EMEA', 'APJ') AND " +
    "(header.priority > 4 OR price >= 99.99 OR premium_customer = TRUE)";

Consumer consumer = connection
    .consumerBuilder()
    .queue(STREAM_NAME)
    .stream()
    .offset(Offset.FIRST)
    // Stage 1: Bloom filter（可选）
    .filterValues("order.created")
    // Stage 2: SQL 精确过滤
    .filter().sql(SQL).stream()
    .builder()
    .messageHandler((ctx, msg) -> {
        System.out.printf("[Consumer] Received: %s%n", new String(msg.body()));
        ctx.accept();
    })
    .build();
```

### 3. 启动 Docker 环境

```bash
# 启动 RabbitMQ（单调度线程便于基准测试）
docker run -it --rm --name rabbitmq -p 5672:5672 \
  -e ERL_AFLAGS="+S 1" rabbitmq:4.2.0-beta.3
```

### 4. SQL 过滤器支持的函数

```sql
-- 时间函数
UTC()                          -- 当前 UTC 时间戳（毫秒）

-- 属性访问
properties.subject             -- AMQP subject
properties.creation_time       -- 消息创建时间
header.priority                -- 消息优先级
<custom_property>              -- 自定义 application property

-- 逻辑运算符
AND, OR, NOT, IN, BETWEEN, =, <>, >, <, >=, <=
```

## 适用场景

| 场景 | 传统方案问题 | SQL 过滤方案 |
|------|-------------|-------------|
| 多租户事件流共享一个 Stream | 每个租户拉取全量数据过滤，网络开销大 | Broker 端按 tenant_id 过滤 |
| 订单事件流只关心特定类型 | 客户端需要消费所有事件类型 | Bloom filter 跳过无关 Chunk |
| 复杂业务规则过滤 | 客户端需下载全部数据再过滤 | SQL 表达式下推至 Broker |
| 海量 IoT 设备数据上报 | 每个消费者负载高、延迟大 | 480 万 msg/s 过滤效率 |

## 注意事项

1. **AMQP 1.0 专属**：SQL filter expressions 是 AMQP 1.0 协议特性，需要使用 AMQP 1.0 客户端（Java、.NET 等），不支持 AMQP 0-9-1。
2. **Bloom filter 选择性依赖**：实际效能取决于每条消息的 Chunk 内的消息密度，高写入速率下效果更明显。
3. **性能基准参考**：480 万 msg/s 是高度选择性场景（0.001% 匹配率）的数据，实际场景请根据自身消息大小、匹配率测试。
4. **RabbitMQ 版本要求**：4.2+（4.0 和 4.1 不支持此特性）。
5. **单调度线程建议**：基准测试使用 `ERL_AFLAGS="+S 1"` 限制调度线程以获取稳定结果，生产环境无需此设置。
