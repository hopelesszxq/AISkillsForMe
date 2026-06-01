---
name: redis-streams-mq
description: Redis Streams 消息队列实践：生产者、消费组、ACK 机制与可靠性保障
tags: [redis, streams, message-queue, pub-sub]
---

## 概述

Redis Streams 5.0+ 引入的原生消息队列数据结构，支持消费者组、消息持久化、ACK 确认和 Pending 列表，适合轻量级消息场景。相比 Pub/Sub 的即发即弃，Streams 保证消息不丢失。

## 1. 核心概念

| 概念 | 说明 |
|---|---|
| **Stream** | 消息链表，每条消息有唯一 ID（时间戳+序号） |
| **Consumer Group** | 消费组，组内消费者分摊消息（类似 Kafka） |
| **Consumer** | 组内消费者，自主维护消费进度 |
| **Pending Entries List (PEL)** | 已投递但未 ACK 的消息列表 |
| **Message ID** | 格式 `时间戳-序号`（如 `1748779200000-0`） |

## 2. 生产者端

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# 添加消息（自动生成 ID）
msg_id = r.xadd('order:stream', {
    'order_id': 'ORD-20260601-001',
    'user_id': 'u10086',
    'amount': '299.00',
    'status': 'CREATED'
})
print(f"Message ID: {msg_id}")

# 指定 ID（MAXLEN 限制长度）
r.xadd('order:stream', {'event': 'timeout'}, id='*', maxlen=1000)

# 指定最大长度裁剪（~ 近似裁剪，更高效）
r.xadd('order:stream', {'event': 'pay'}, maxlen=1000, approximate=True)
```

### Java (Lettuce) 生产者

```java
// 依赖 io.lettuce:lettuce-core
RedisCommands<String, String> sync = connection.sync();
Map<String, String> body = Map.of(
    "orderId", "ORD-20260601-001",
    "userId", "u10086"
);
String msgId = sync.xadd("order:stream", body);
```

## 3. 消费者组

```python
# 创建消费组（从已存在 Stream 创建）
try:
    r.xgroup_create('order:stream', 'order-processors', id='0')
except redis.exceptions.ResponseError as e:
    if 'BUSYGROUP' not in str(e):
        raise  # 组已存在则忽略

# 读取消息 —— 组内分发 (使用 > 获取未分发的消息)
results = r.xreadgroup(
    groupname='order-processors',
    consumername='worker-1',
    streams={'order:stream': '>'},
    count=10,
    block=2000  # 阻塞等待 2 秒
)

for stream_name, messages in results:
    for msg_id, msg_data in messages:
        print(f"Processing {msg_id}: {msg_data}")
        # 处理完成后 ACK
        r.xack('order:stream', 'order-processors', msg_id)
```

### Java (Lettuce) 消费者

```java
// 消费组读取
List<StreamMessage<String, String>> messages = sync.xreadgroup(
    Consumer.from("order-processors", "worker-1"),
    XReadArgs.Builder.block(Duration.ofSeconds(2)).count(10),
    XReadArgs.StreamOffset.lastConsumed("order:stream")
);

for (StreamMessage<String, String> msg : messages) {
    System.out.println(msg.getId() + " -> " + msg.getBody());
    // ACK
    sync.xack("order:stream", "order-processors", msg.getId());
}
```

## 4. 死信处理与重试

```python
# 查看 Pending（未 ACK）消息
pending = r.xpending('order:stream', 'order-processors')
print(f"Total pending: {pending['pending']}")

# 查看指定消费者的 Pending 消息详情
pending_detail = r.xpending_range(
    'order:stream', 'order-processors',
    min='-', max='+', count=10, consumername='worker-1'
)

# 重新认领超时消息（其他消费者处理）
CLAIM_TIMEOUT_MS = 3600000  # 1小时未 ACK 的重新分配
claimed = r.xclaim(
    'order:stream', 'order-processors', 'worker-2',
    CLAIM_TIMEOUT_MS,
    ['1748779200000-0', '1748779300000-1']
)
```

## 5. 消息可靠性保障策略

| 场景 | 解决方案 |
|---|---|
| 消费者宕机未 ACK | PEL 保留消息，`XCLAIM` 给其他消费者 |
| 消息处理失败 | 记录重试次数，超过阈值转入死信 Stream |
| 防止重复消费 | 消息 ID + 幂等性判断（Redis SETNX） |
| 消息顺序保障 | 同一个 Consumer 消费同一个 Stream 分区 |
| 监控积压 | `XLEN` 查看 Stream 长度 + `XPENDING` 查看未 ACK |

```python
# 死信转移实现
MAX_RETRIES = 3

def process_with_retry(stream, group, consumer, msg_id, msg_data):
    # 检查重试次数（从 Pending 信息获取投递次数）
    pending_info = r.xpending_range(
        stream, group, min=msg_id, max=msg_id, count=1, consumername=consumer
    )
    if pending_info and pending_info[0]['times_since_delivered'] > MAX_RETRIES:
        # 转入死信 Stream
        r.xadd(f'{stream}:dlq', msg_data)
        r.xack(stream, group, msg_id)
        return False
    return True
```

## 6. Redis 7+ 新特性

- **Auto-claim** (`XAUTOCLAIM`)：自动扫描超时 Pending 消息并重新分配，替代手动的 `XPENDING` + `XCLAIM` 两步操作
- **Stream 多消费者组**：一个 Stream 支持多个独立消费组，各组进度独立

```python
# Redis 7+ 自动认领
claimed = r.xautoclaim(
    'order:stream', 'order-processors', 'worker-2',
    min_idle_time=3600000,
    start_id='0-0',
    count=10
)
# 返回 (next_start_id, [messages])
```

## 注意事项

1. **Stream 不是 Kafka**：消息不分区（只有单节点），吞吐量有限，仅适合中小规模场景
2. **内存限制**：务必设置 `MAXLEN` 控制 Stream 长度，避免 OOM
3. **消费组 ID 选择**：`0` 表示从头消费，`$` 表示只消费新消息
4. **阻塞 vs 轮询**：`block` 参数避免空轮询浪费 CPU，建议设置 2-5 秒超时
5. **ACK 必须手动调用**：`XACK` 不自动触发，忘记调用会导致 PEL 无限增长
6. **生产环境**：Streams + Sentinel/Cluster 配合使用时需注意节点故障转移可能导致消息丢失
