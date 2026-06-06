---
name: rabbitmq-43-migration
description: RabbitMQ 4.3.0 重大变更迁移指南：Mnesia 移除、Khepri 独占、CQv1 删除、分区策略废弃与队列兼容性
tags: [rabbitmq, migration, breaking-change, khepri, mnesia, quorum-queues, mq]
---

## 概述

RabbitMQ 4.3.0（2026-05 发布）是一次**重大版本升级**，移除了多个遗留组件和功能，包含多项**破坏性变更**。从 4.2.x 升级到 4.3.x 需要仔细评估兼容性。

> ⚠️ **仅支持从 4.2.x 直接升级到 4.3.x**，旧版本必须先升级到 4.2.x。

## 一、Khepri 成为唯一的元数据存储（Mnesia 移除 🚫）

**最大变更**：Mnesia 被彻底移除，Khepri 成为唯一的元数据存储。

### 对运维的影响

```text
Mnesia 模式（4.2.x 之前）               Khepri 模式（4.3.0 起）
─────────────────────────              ────────────────────────
支持少数节点挂起继续服务                 集群多数节点在线才能可用
多种分区处理策略                        统一 Raft 恢复语义
独立元数据副本                          与 Quorum/Stream 共享 Raft 恢复
```

### 必须执行的操作

```ini
# rabbitmq.conf — 移除以下已无意义的配置
# cluster_partition_handling = pause_minority     ❌ 已无效
# cluster_partition_handling.pause_if_all_down...  ❌ 已无效
```

### 集群可用性变化

- **集群必须保持多数节点在线**（≥ N/2 + 1），否则元数据不可用
- 网络分区恢复统一走 Raft 语义，不再有 `pause_minority`、`autoheal` 等策略
- 建议部署奇数节点（3/5/7），避免偶数节点网络裂脑

## 二、Classic Queues v1 (CQv1) 存储格式移除

4.3.0 彻底移除了 CQv1 存储格式（CQv2 自 4.2.0 起为默认）。

### 无法创建的队列

以下队列声明将在 4.3.0 上**直接报错**：

| 参数 | 值 | 说明 |
|------|-----|------|
| `x-queue-mode` | `lazy` 等 | CQv1 的延迟模式 |
| `x-queue-version` | `1` | 显式指定 v1 版本 |

```bash
# ❌ 以下声明在 4.3.0 失败
rabbitmqadmin declare queue name=q1 arguments='{"x-queue-version": 1}'
```

### 迁移检查

```bash
# 检查是否还有 CQv1 队列
rabbitmqctl list_queues name type version | grep "classic.*1$"

# 如果有，升级到 4.2.x 时已自动迁移到 CQv2
# 4.3.0 中这些队列继续正常工作，只是无法新建 CQv1
```

## 三、临时非排他队列默认禁用

4.3.0 默认**拒绝声明临时非排他队列**（non-durable + non-exclusive）。

```bash
# ❌ 4.3.0 默认拒绝
channel.queueDeclare("temp-queue", false, false, false, null);
# → AMQP access refused: queue 'temp-queue'...
```

### 解决方案

```bash
# ✅ 选项1：使用持久队列
channel.queueDeclare("queue", true, false, false, null);

# ✅ 选项2：使用排他临时队列（连接断开自动删除）
channel.queueDeclare("", false, true, true, null);

# ✅ 选项3：持久队列 + TTL
Map<String, Object> args = new HashMap<>();
args.put("x-expires", 60000);  // 60秒后自动删除
channel.queueDeclare("auto-delete-queue", true, false, false, args);
```

### 如确需恢复旧行为

```ini
# rabbitmq.conf（所有节点必须一致配置并重启）
deprecated_features.permit.transient_nonexcl_queues = true
```

## 四、消费者超时行为变更

消费者超时处理逻辑从 Broker 核心移至各个队列类型自身：

| 队列类型 | 4.2.x | 4.3.0 |
|----------|-------|-------|
| Classic Queue | 评估消费者超时 | **不评估**（消费者不会因超时被断开） |
| Stream | 评估消费者超时 | **不评估** |
| Quorum Queue | 评估消费者超时 | 仍评估（协议层支持） |

**影响**：Classic Queue 和 Stream 的消费者将不再因处理时间过长而被断开。

## 五、Quorum Queue 增强

### Ra 升级到 3.x + 8th 版状态机

```text
新增能力：
├── 严格优先级队列（per-priority 计数的消息 + 优先级感知重投递）
├── Raft 日志压缩效率提升
└── 更好的内存管理
```

```java
// 声明严格优先级 Quorum Queue
Map<String, Object> args = new HashMap<>();
args.put("x-queue-type", "quorum");
args.put("x-max-priority", 10);       // 支持优先级
args.put("x-quorum-initial-group-size", 3);  // 初始 Raft 组大小
channel.queueDeclare("priority-q", true, false, false, args);
```

## 六、升级步骤

```bash
# 1. 检查当前版本
rabbitmqctl version

# 2. 确认当前为 4.2.x，不是则先升级到 4.2.x
# 3. 检查 CQv1 队列
rabbitmqctl list_queues name type version | grep "classic.*1$"

# 4. 检查是否有项目使用临时非排他队列
# 5. 移除 rabbitmq.conf 中已废弃的配置
#    cluster_partition_handling *

# 6. 滚动升级（逐个节点）
#    - 停节点 → 更新包 → 启动
rabbitmqctl stop_app
# 更新 RabbitMQ 包（APT/YUM）
sudo apt-get install rabbitmq-server=4.3.1-1
rabbitmqctl start_app

# 7. 验证集群状态
rabbitmqctl cluster_status
rabbitmqctl list_queues name type
```

## 注意事项

1. **仅支持从 4.2.x 升级**：4.1.x 及以下必须先升级到 4.2.x，再升级到 4.3.x
2. **Erlang 版本要求**：RabbitMQ 4.3.x 需要 Erlang/OTP 26+，节点将拒绝在旧 Erlang 上启动
3. **集群多数在线**：4.3.0 要求集群多数节点在线才能提供服务，建议至少 3 节点
4. **Deprecated 配置主动移除**：`cluster_partition_handling` 相关配置不会被报错，但**完全无效**，应移除避免运维误解
5. **客户端兼容性**：AMQP 0-9-1 和 AMQP 1.0 协议层无破坏性变化，客户端无需修改
6. **CQv2 性能**：如果升级自旧版本（<4.2），关注 CQv2 的性能差异（通常优于 CQv1，但少数批量写入场景可能有变化）
