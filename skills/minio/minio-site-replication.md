---
name: minio-site-replication
description: MinIO 站点复制（Site Replication）实现多站点 Active-Active 数据同步与容灾
tags: [minio, site-replication, active-active, dr, multi-site, ha]
---

## 概述

MinIO 站点复制（Site Replication）允许在多个地理分布的 MinIO 集群（站点）之间自动同步 **IAM 策略、用户、Bucket 配置** 和 **对象数据**，实现 Active-Active 多活架构。

### 与 Bucket Replication 的区别

| 特性 | Bucket Replication | Site Replication |
|------|-------------------|-----------------|
| 作用范围 | 单个 bucket | 整个站点（所有 bucket + 配置） |
| IAM 同步 | ❌ 不同步 | ✅ 用户/策略/Group 自动同步 |
| Bucket 配置 | ❌ 生命周期/通知等不同步 | ✅ 全部同步 |
| 数量限制 | 无限制 | 推荐 ≤4 个站点 |
| 一致性模型 | 最终一致性 | 最终一致性（带冲突检测） |
| 配置方式 | 规则级别（JSON） | 站点级别（mc admin replicate） |
| 适用场景 | 特定数据备份 | 多站点灾备、全局命名空间 |

## 架构要求

### 前提条件

- 每个站点必须是**独立部署的 MinIO 集群**（单节点单盘也可）
- 各站点网络互通（建议站点间延迟 < 100ms）
- 各站点版本一致（必须相同 MinIO 版本）
- 站点数量：2~4 个
- 各站点必须使用**独立的本地存储**

### 网络拓扑示意

```
  ┌──────────────┐         ┌──────────────┐
  │  站点 A      │         │  站点 B      │
  │  Ohio (us)   │◄───────►│  Paris (eu)  │
  │  10.0.1.10   │         │  10.0.2.10   │
  └──────────────┘         └──────────────┘
         │                        │
         ▼                        ▼
  ┌──────────────┐         ┌──────────────┐
  │  应用 A      │         │  应用 B      │
  │（就近写 A）   │         │（就近写 B）   │
  └──────────────┘         └──────────────┘
```

## 配置步骤

### 1. 准备环境变量

在每个站点设置相同的凭据和 Region：

```bash
# 站点 A
export MINIO_ROOT_USER=admin
export MINIO_ROOT_PASSWORD=minio-secret-key
export MINIO_REGION=us-east-1
# 启动 ...
```

```bash
# 站点 B（相同用户/密码，不同 Region 可选）
export MINIO_ROOT_USER=admin
export MINIO_ROOT_PASSWORD=minio-secret-key
export MINIO_REGION=eu-west-1
# 启动 ...
```

> **注意**：Site Replication 要求所有站点的 root 凭证一致。

### 2. 使用 mc 配置站点复制

```bash
# 创建 alias
mc alias set site-a http://10.0.1.10:9000 admin minio-secret-key
mc alias set site-b http://10.0.2.10:9000 admin minio-secret-key

# 添加站点复制
mc admin replicate add site-a site-b

# 输出示例：
# Replication configuration for site-a and site-b added successfully.
# Deployment ID: d1a2b3c4-d5e6-7890-abcd-ef1234567890

# 验证复制状态
mc admin replicate status site-a

# 添加更多站点（最多 4 个）
mc admin replicate add site-a site-b site-c site-d
```

### 3. 验证复制工作

```bash
# 在站点 A 创建 bucket
mc mb site-a/my-bucket

# 在站点 B 确认自动创建
mc ls site-b
# 应看到 my-bucket

# 上传文件到站点 A
mc cp /tmp/test.txt site-a/my-bucket/

# 在站点 B 验证复制
mc ls site-b/my-bucket/
# 应看到 test.txt

# 在站点 B 创建用户/策略
mc admin user svcacct add site-b myuser
mc admin policy set site-b readwrite user=myuser

# 在站点 A 验证同步
mc admin user list site-a
# 应看到 myuser
```

## 高级配置

### 使用部署 ID 验证

```bash
# 查看每个站点的部署 ID
mc admin info site-a --json | jq '.info.deploymentID'
mc admin info site-b --json | jq '.info.deploymentID'

# 验证站点复制组成员
mc admin replicate info site-a

# 输出包含所有站点 ID 和状态
```

### 单站点故障恢复

当某站点完全故障需要重建时：

```bash
# 1. 从剩余某健康站点导出配置
mc admin replicate export site-a > replicate-config.json

# 2. 在新站点恢复（启动新 MinIO 实例后）
mc admin replicate import site-new --file replicate-config.json

# 3. 重新添加到复制组
mc admin replicate add site-a site-b site-new
```

## 冲突处理

当多个站点同时写同一个对象时：

| 场景 | 处理方式 |
|------|---------|
| 同一对象同时 PUT | 使用**最后写入者获胜（LWW）**策略 |
| 对象版本冲突 | 保留所有版本，通过版本 ID 访问 |
| 删除操作 | 删除标记复制到所有站点 |
| IAM 冲突 | 写入顺序按时间戳裁决 |

```bash
# 查看冲突统计
mc admin replicate status site-a --json | jq '.ReplicationStats'

# 手动解决版本冲突
mc ls --versions site-a/my-bucket/conflict-file.txt
# 选择需要的版本
mc cp site-a/my-bucket/conflict-file.txt --version-id "v1" ./resolved.txt
```

## 监控与诊断

### 复制延迟监控

```bash
# 查看复制队列深度
mc admin replicate status site-a --json | jq '.ReplicationStats'

# 关键指标
# - PendingCount:    待复制的对象数
# - FailedCount:     复制失败的永久错误数
# - CompletedCount:  成功复制数
# - ReplicationLatency: 复制平均延迟

# Prometheus 指标（MinIO 默认暴露）
# minio_site_replication_pending_count
# minio_site_replication_failed_count
# minio_site_replication_errors_total
```

### 日志排查

```bash
# 查看复制相关日志
mc admin trace site-a --type replication

# 或通过控制台
# 访问 http://site-a:9001 → Tools → Audit Logs → 筛选 replication
```

## 性能调优

### 站点间带宽

```bash
# 限制复制带宽（mb/s）
export MINIO_SITE_REPLICATION_BANDWIDTH_LIMIT=100

# 或通过 mc 调整（运行时）
mc admin config set site-a site_replication bandwidth_limit=100
mc admin service restart site-a
```

### 批量复制大小

```bash
# 调整批量复制并发（默认 10）
export MINIO_SITE_REPLICATION_BATCH_SIZE=50
export MINIO_SITE_REPLICATION_WORKERS=20
```

## 使用建议

### 写入策略

```
推荐：应用程序写本地站点，读本地站点
  - 写入延迟最小
  - 避免跨站点写导致的高延迟
  - 数据最终一致（秒级~分钟级）

不推荐：全局负载均衡写（随机选择站点）
  - 导致大量跨站点复制
  - 冲突概率升高
  - 延迟不可控
```

### 部署拓扑

- **2 站点**：基础容灾，建议距离 > 100km
- **3 站点**：高可用，2:1 仲裁（任一站点故障不影响整体）
- **4 站点**：最大扩展，但一致性延迟增加

## 注意事项

1. **非对称网络不可用**：所有站点必须互通（不能 A→B→C 链式转发，必须是 A↔B, A↔C, B↔C 全网状）

2. **存储增长**：每个站点独立存储全量数据，总容量 = 单站容量 × 站点数。建议使用纠删码（EC）而非副本

3. **站点移除**：移除站点前确保数据已同步完毕。移除操作不会自动删除远端数据：
   ```bash
   mc admin replicate remove site-a --deployment-id <id>
   ```

4. **兼容性**：Site Replication 不支持以下功能同步：
   - Bucket 通知配置（需要各自配置）
   - Bucket 生命周期规则（各自配置）
   - 对象锁定（Object Lock）保留设置
   - MinIO Console 的个性化设置

5. **版本要求**：MinIO RELEASE.2022-09-17T00-00-00Z 开始支持 Site Replication。建议使用最新版

6. **SSE-KMS 加密**：如果使用 KMS 加密，所有站点必须共享相同 KMS 配置，否则跨站点无法解密对象
