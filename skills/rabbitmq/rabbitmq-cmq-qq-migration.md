---
name: rabbitmq-cmq-qq-migration
description: RabbitMQ Classic Mirrored Queues 到 Quorum Queues 蓝绿迁移完整方案
tags: [rabbitmq, mq, migration, quorum-queues, cmq, blue-green, federation]
---

## 概述

Classic Mirrored Queues (CMQ) 自 RabbitMQ 3.9 起弃用，在 **RabbitMQ 4.0 中彻底移除**。所有仍在使用 CMQ 的集群必须迁移到 Quorum Queues (QQ)。本文提供经过验证的蓝绿迁移方案，基于 RabbitMQ 官方博客文章整理。

> 本文方案已验证：Blue 集群 3.13.7 → Green 集群 4.1.2，支持 TLS/mTLS/plain TCP。

## 为什么要迁移到 Quorum Queues

| 特性 | CMQ (经典镜像队列) | QQ (法定人数队列) |
|------|-------------------|------------------|
| 复制算法 | 链式复制 (Chain Replication) | Raft 共识算法 (并行复制) |
| 数据安全 | 可能丢失消息 | 通过 Jepsen 严格测试 |
| 网络分区 | 行为不可预测 | 自动 Leader 选举，行为可预测 |
| 性能 | 链式复制较慢 | 并行复制，更高吞吐 |
| 维护状态 | 3.13 仅维护，4.0 已移除 | 活跃开发，持续优化 |

### 不需要使用 QQ 的场景

以下场景**不应使用复制队列**，应保留为普通 Classic Queue：

- 临时队列（如 RPC request-reply 模式的回调队列）
- 短生命周期的一次性队列
- fire-and-forget 且消费者不使用 manual ack 的场景

## 蓝绿迁移前提条件

1. **Queue Federation** 必须在 Blue 和 Green 集群之间启用
2. **Blue 集群**必须对 Green 集群可达（用于 Federation 连接）
3. **Green 集群**必须运行 RabbitMQ **4.x**（建议最新版本）
4. Green 集群上**所有稳定 Feature Flag** 必须已启用

## 迁移步骤

### Step 1: 导出 Blue 集群定义并转换

使用 `rabbitmqadmin v2` 导出定义，同时应用转换规则自动将 CMQ 策略转换为 QQ：

```bash
# 导出并转换：CMQ→QQ，并删除空策略
rabbitmqadmin --host blue definitions export \
  --file blue.json \
  -t prepare_for_quorum_queue_migration,drop_empty_policies
```

此转换会自动：
- 移除已废弃的 CMQ 策略键（`ha-mode`、`ha-params`、`ha-promote-on-shutdown` 等）
- 将有镜像策略的队列转换为 Quorum Queue
- 删除因 CMQ 键移除后变空的策略（空策略无法导入）

### Step 2: 导入定义到 Green 集群

```bash
rabbitmqadmin --host green definitions import --file ./blue.json
```

### Step 3: 配置 Federation

#### 3a. 为每个 VHost 创建 Federation Upstream

```bash
rabbitmqadmin --host green federation declare_upstream_for_queues \
  --name cmq-qq-migration \
  --uri 'amqp://<federation-user>:<password>@blue.rabbit'
```

> 💡 建议创建专用的 Federation 用户。

#### 3b. 创建覆盖策略 (Override Policies)

对已存在的每个策略创建 override 以启用 Federation：

```bash
# 列出已有策略
rabbitmqadmin --host green policies list

# 对每个策略创建 override
rabbitmqadmin --host green policies declare_override \
  --name <policy-name> \
  --definition '{"federation-upstream": "cmq-qq-migration"}'
```

#### 3c. 创建兜底策略 (Blanket/Catch-all Policy)

为未被任何策略匹配的队列创建全匹配策略：

```bash
rabbitmqadmin --host green policies declare_blanket \
  --name cmq-qq-migration_blanket \
  --apply-to queues \
  --definition '{"federation-upstream": "cmq-qq-migration"}'
```

#### 3d. 验证 Federation 链路

```bash
rabbitmqadmin --host green federation list_all_links
```

期望输出：每个队列有一条 `running` 状态的 `federation-link`。

### Step 4: 迁移消费者应用

将消费者应用逐步切换到 Green 集群。Federation 会自动将消息从 Blue 拉取到 Green。

> 可以分批次迁移，无需急于一次性完成。先验证消费者在 Green 上正常工作。

### Step 5: 迁移生产者应用

停止生产者 → 等待消费者排空 Blue 中的队列（通过 Federation） → 在 Green 上启动生产者。

> **长积压场景**：如果队列积压严重，消费者速度慢，可额外配置 Shovel 主动搬运消息。

### Step 6: 清理迁移遗留策略

确认所有应用正常后，清理临时策略：

```bash
# 删除兜底策略
rabbitmqadmin --host green policies delete --name cmq-qq-migration_blanket

# 删除各 override 策略
rabbitmqadmin --host green policies delete --name overrides.<policy-name>

# 删除 Federation upstream
rabbitmqadmin --host green federation delete_upstream --name cmq-qq-migration
```

## TLS/mTLS 配置

如 Blue/Green 集群使用 TLS，rabbitmqadmin v2 需要系统信任 CA 证书：

```bash
rabbitmqadmin --use-tls \
  --tls-ca-cert-file /path/to/chained_ca_cert.pem \
  --tls-cert-file /path/to/client_cert.pem \
  --tls-key-file /path/to/client_key.pem \
  queues list
```

也可通过配置文件简化：

```toml
# rabbitmqadmin.toml
[blue]
hostname = "blue.rabbit"
tls = true
ca_certificate_bundle_path = "/path/to/ca.pem"
client_certificate_file_path = "/path/to/cert.pem"
client_private_key_file_path = "/path/to/key.pem"
port = 32795

[green]
hostname = "green.rabbit"
tls = true
ca_certificate_bundle_path = "/path/to/ca.pem"
client_certificate_file_path = "/path/to/cert.pem"
client_private_key_file_path = "/path/to/key.pem"
port = 32811
```

```bash
rabbitmqadmin --config rabbitmqadmin.toml --node blue queues list
```

## 迁移后检查清单

- [ ] 所有消费者应用已连接到 Green 集群
- [ ] 所有生产者应用已连接到 Green 集群
- [ ] 临时策略（override/blanket）已删除
- [ ] Federation upstream 已删除
- [ ] RabbitMQ 无任何告警
- [ ] 所有指标在正常范围内

## 测试迁移

在生产环境之前，**强烈建议在开发/预发环境完整演练**。可使用 `rabbitmq-perf-test` 模拟生产负载：

```bash
java -jar perf-test.jar \
  -y 0 -x 1 -c 10 -p -qq \
  --rate 30 -e inc \
  --routing-key 'bills' \
  --uri "amqp://user:pass@blue.rabbit.example.com"
```

## 注意事项

1. **无法原地转换**：CMQ 不能原地转换为 QQ，必须通过蓝绿部署迁移
2. **队列类型显式声明**：如果应用明确指定了队列类型（如 `arguments.put("x-queue-type", "classic")`），需要更新代码或通过 `rabbitmq.conf` 放宽类型检查
3. **临时队列无需复制**：迁移前审查哪些队列真正需要复制，减少 QQ 使用量
4. **确认 Federation 正常**：迁移的关键步骤，务必在消费者迁移前验证 Federation 链路状态
5. **长积压场景**：考虑使用 Shovel 配合 Federation 加速消息搬运
6. **蓝绿网络连通**：确保 Federation 使用的端口和 VHost 在 Blue 集群上正确授权
