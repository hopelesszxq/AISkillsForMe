---
name: rabbitmq-shovel-federation
description: RabbitMQ Shovel 和 Federation 实现跨集群消息同步与异地多活部署
tags: [rabbitmq, shovel, federation, cross-dc, multi-cluster, replication]
---

## 概述

RabbitMQ 提供两种跨集群消息同步机制：

| 特性 | Shovel（铲子） | Federation（联邦） |
|------|---------------|-------------------|
| **粒度** | 队列维度 | Exchange 维度 |
| **方向** | 单向（可配多对实现双向） | 单向（上行/下行） |
| **拓扑** | 1:1 队列映射 | N:M Exchange 绑定 |
| **可靠性** | 内置确认机制，保证至少一次投递 | AMQP 事务级确认 |
| **延迟** | 低（推模式） | 略高（拉模式 + 轮询） |
| **动态增减** | 需手动配置 | 自动发现 binding 变更 |
| **适用场景** | 特定队列数据同步、迁移 | 多站点 Exchange 拓扑共享 |

## Shovel 配置

### 插件启用

```bash
# 启用 Shovel 插件（两端都需要）
rabbitmq-plugins enable rabbitmq_shovel rabbitmq_shovel_management

# 确认插件已加载
rabbitmq-plugins list | grep shovel
```

### 静态 Shovel（通过 rabbitmq.conf）

```ini
# /etc/rabbitmq/rabbitmq.conf — 源集群

# 定义 shovel
shovels.my-shovel.sources.queue = source-queue
shovels.my-shovel.sources.delete-after = never

# 目标配置
shovels.my-shovel.destination.uris.1 = amqps://dest-user:dest-pass@dest-cluster:5671
shovels.my-shovel.destination.queue = dest-queue
shovels.my-shovel.destination.protocol = amqp091
shovels.my-shovel.destination.add-forward-headers = true

# ACK 模式 — on-confirm 确保消息到达目标后才 ACK 源
shovels.my-shovel.ack-mode = on-confirm

# 重连策略
shovels.my-shovel.reconnect-delay = 5
shovels.my-shovel.reconnect-delay-max-after-backoff = 30
```

### 动态 Shovel（通过 rabbitmqctl）

```bash
# 参数结构：shovel_name 后接 JSON 配置
rabbitmqctl set_parameter shovel my-shovel \
  '{"src-protocol": "amqp091",
    "src-queue": "source-queue",
    "dest-protocol": "amqp091",
    "dest-queue": "dest-queue",
    "dest-uri": ["amqps://dest-user:dest-pass@dest-cluster:5671"],
    "src-uri": ["amqp://"],
    "ack-mode": "on-confirm",
    "reconnect-delay": 5}'
```

### 跨协议 Shovel（AMQP 0-9-1 → AMQP 1.0）

```bash
rabbitmqctl set_parameter shovel amqp10-bridge \
  '{"src-protocol": "amqp091",
    "src-queue": "legacy-queue",
    "dest-protocol": "amqp10",
    "dest-address": "amqp://target:5672/topic://events",
    "dest-uri": ["amqps://user:pass@target:5671"],
    "ack-mode": "on-confirm"}'
```

### 监控 Shovel 状态

```bash
# 列出现有 shovel
rabbitmqctl list_parameters

# 查看 shovel 运行状态（Management UI → Admin → Shovel）
# 或通过 HTTP API
curl -u guest:guest http://localhost:15672/api/shovels

# 日志中查看 shovel 错误
grep shovel /var/log/rabbitmq/rabbit@*.log
```

## Federation 配置

### 插件启用

```bash
rabbitmq-plugins enable rabbitmq_federation rabbitmq_federation_management
```

### 上行 Federation（Upstream）：从远程 Exchange 拉取消息到本地

```bash
# 1. 定义 upstream（在消费端集群执行）
rabbitmqctl set_parameter federation-upstream my-upstream \
  '{"uri": "amqps://source-user:source-pass@source-cluster:5671",
    "exchange": "source-exchange",
    "max-hops": 1,
    "prefetch-count": 100,
    "reconnect-delay": 5,
    "ack-mode": "on-confirm",
    "trust-user-id": false}'

# 2. 将本地 Exchange 绑定到 upstream
rabbitmqctl set_parameter federation-upstream-set my-set \
  '[{"upstream": "my-upstream"}]'

# 3. 配置 Exchange 使用 federation
rabbitmqctl set_policy federation-policy \
  "^federated-" \
  '{"federation-upstream-set":"my-set"}' \
  --priority 10 \
  --apply-to exchanges
```

### Federation 配置示例（rabbitmq.conf）

```ini
# /etc/rabbitmq/rabbitmq.conf

# 定义 upstream 集群
federation.upstreams.source.uri = amqps://src-user:src-pass@src-cluster:5671
federation.upstreams.source.exchange = src-exchange

# 自动重连
federation.upstreams.source.reconnect-delay = 5
federation.upstreams.source.ack-mode = on-confirm

# 定义 upstream set
federation.upstream-sets.default.1 = source

# 策略：自动联邦所有以 federated- 开头的 Exchange
federation.policy.federate-exchanges.pattern = ^federated-
federation.policy.federate-exchanges.definition.federation-upstream-set = default
federation.policy.federate-exchanges.apply-to = exchanges
```

### 基于 Federation 的异地多活架构

```
          ┌──────────────┐          ┌──────────────┐
          │  数据中心A   │          │  数据中心B   │
          │  (主写)      │          │  (主读)      │
          │              │          │              │
          │  Exchange_A  │◄─────────│  Exchange_B  │ ← Federation 拉取
          │      │       │  上行     │      │       │
          │      ▼       │          │      ▼       │
          │  Queue_A     │          │  Queue_B     │
          └──────────────┘          └──────────────┘
```

## 异地多活架构设计

### 方案一：双活 + 本地消费

```
每个数据中心独立部署 RabbitMQ 集群，通过 Federation 或 Shovel 同步关键队列。

写入策略：
  - 生产者写入本地集群（就近写入）
  - Federation 将消息同步到远端集群
  - 消费者优先消费本地队列

故障切换：
  - 本地集群故障 → DNS/负载均衡切换到远端
  - 恢复后 Shovel 回同步积压消息
```

### 方案二：主备 + Shovel

```
主集群写入，Shovel 单向同步到备集群。

优点：简单可靠，配置少
缺点：备集群仅用于容灾，不承担读写
```

## 重要配置参数

### Shovel 参数

| 参数 | 默认值 | 推荐值 | 说明 |
|------|--------|--------|------|
| `ack-mode` | `on-confirm` | `on-confirm` | 确认后再确认源端，防止丢失 |
| `reconnect-delay` | 1s | 5s | 断连后重试间隔 |
| `reconnect-delay-max-after-backoff` | 无 | 30s | 退避后最大重连间隔 |
| `delete-after` | `never` | `never` | 任务执行完后是否自删 |

### Federation 参数

| 参数 | 默认值 | 推荐值 | 说明 |
|------|--------|--------|------|
| `prefetch-count` | 1000 | 100~500 | 批量拉取消息数，越大吞吐越高但延迟越大 |
| `max-hops` | 1 | 1 | 防止环形转发（环路检测） |
| `ack-mode` | `on-confirm` | `on-confirm` | 确认模式 |
| `trust-user-id` | false | false | 是否信任上游用户 ID |

## 注意事项

1. **环路防止**：Federation 的 `max-hops=1` 防止消息在两个集群间无限转发。如果 A→B→A 形成环，消息会被丢弃

2. **消息优先级**：Shovel 和 Federation 不保留消息的优先级属性（priority）。如果业务依赖消息优先级，需在传输层额外处理

3. **安全传输**：跨集群务必使用 AMQPS（TLS）：
   ```ini
   src-uri = amqps://user:pass@src:5671?verify=verify_peer&fail_if_no_peer_cert=true
   ```

4. **性能监控**：Shovel 和 Federation 都在 Erlang 虚拟机内运行，大量 Shovel 实例会消耗内存。建议每个队列一个 Shovel，而非用 Shovel 批量处理

5. **备集群写入**：Federation 默认上层（upstream）Exchange 绑定关系变化不会自动同步到下游。如果需要在备集群也接受写入，需配置双向 Federation 或使用 Shovel 反向同步

6. **版本兼容**：
   - Shovel 在 RabbitMQ 3.x+ 全部支持
   - Federation 的 AMQP 1.0 支持需要 RabbitMQ 4.0+
   - 跨协议 Shovel（AMQP 0-9-1 ↔ AMQP 1.0）需要 3.13+

7. **迁移场景**：Shovel 特别适合 RabbitMQ 集群迁移场景——配置源队列 Shovel 到目标集群，消费者切换完成后确认积压清空，移除 Shovel。全程零停机
