---
name: minio-batch-operations
description: MinIO Batch 批处理框架：批量复制、过期、密钥轮转与脚本处理
tags: [minio, batch, replication, object-storage, automation, data-management]
---

## 概述

MinIO Batch 框架通过 `mc batch` 命令提供声明式批量操作能力，支持将 YAML 作业定义提交给 MinIO 服务端执行，无需外部调度工具。

## 作业类型

| 类型 | 命令 | 用途 |
|------|------|------|
| 批量复制 (Replicate) | `mc batch start replicate` | 跨桶/站点批量同步对象 |
| 批量过期 (Expire) | `mc batch start expire` | 按规则批量删除过期对象 |
| 批量密钥轮转 (Keyrotate) | `mc batch start keyrotate` | 批量轮转 SSE-KMS 加密密钥 |
| 批量脚本 (Script) | `mc batch start script` | 对桶内对象执行自定义脚本 |

## 通用命令

```bash
# 生成作业模板
mc batch generate <bucket/prefix> --type <replicate|expire|keyrotate|script>

# 启动批处理作业
mc batch start <bucket/prefix> <job.yaml>

# 列出运行中的批量作业
mc batch list <alias>

# 查看作业状态
mc batch status <alias> <job-id>

# 查看作业日志
mc admin trace <alias> --type batch
```

## 1. 批量复制 (Batch Replicate)

将源桶中的对象批量复制到目标桶，支持按前缀/标签过滤。

```yaml
# batch-replicate.yaml
replicate:
  apiVersion: v1
  source:
    bucket: source-bucket       # 源桶
    prefix: logs/               # 可选：前缀过滤
  target:
    bucket: backup-bucket       # 目标桶
    prefix: archive/            # 可选：目标前缀
    endpoint: https://backup.minio.example.com  # 可选：跨集群复制
    credentials:                # 目标集群凭据
      accessKey: TARGET_ACCESS_KEY
      secretKey: TARGET_SECRET_KEY
  notify:
    endpoint: https://hooks.example.com/batch-complete  # 完成通知
    token: webhook-secret-token
```

```bash
# 启动批量复制
mc batch start myminio batch-replicate.yaml

# 示例输出
# Job 'f8a3b2c1-...' started successfully
# Batch replication in progress... use 'mc batch status myminio <job-id>' to check
```

### 跨集群复制场景

```yaml
# batch-cross-cluster.yaml
replicate:
  apiVersion: v1
  source:
    bucket: prod-data
    prefix: critical/
    # 按标签过滤
    tags:
      - key: tier
        value: gold
  target:
    bucket: dr-data
    endpoint: https://dr-cluster.example.com
    credentials:
      accessKey: DR_ACCESS_KEY
      secretKey: DR_SECRET_KEY
  # 失败重试
  retry:
    attempts: 3
    delay: 5s
```

## 2. 批量过期 (Batch Expire)

按规则批量删除对象，适用于生命周期管理中的一次性清理。

```yaml
# batch-expire.yaml
expire:
  apiVersion: v1
  source:
    bucket: logs-bucket
    prefix: debug/
  # 过期规则（多选一）
  rules:
    - olderThan: 90d             # 90 天前的对象
    - createdBefore: "2026-01-01" # 指定日期前创建的对象
    # - type: null                # 删除删除标记（DeleteMarker）
  notify:
    endpoint: https://hooks.example.com/cleanup
    token: clean-token
```

```bash
# 生成过期模板
mc batch generate myminio/logs-bucket --type expire > cleanup.yaml

# 修改后执行
mc batch start myminio cleanup.yaml
```

### 多桶清理

```yaml
# batch-cleanup.yaml
expire:
  apiVersion: v1
  source:
    bucket: temp-files
  rules:
    - olderThan: 7d               # 删除 7 天前的临时文件
    - olderThan: 1d
      prefix: tmp/                # tmp/ 下的文件 1 天即删除
```

## 3. 批量密钥轮转 (Batch Keyrotate)

对 SSE-KMS 加密的对象批量轮转加密密钥。

```yaml
# batch-keyrotate.yaml
keyrotate:
  apiVersion: v1
  source:
    bucket: encrypted-bucket
    prefix: sensitive/
  # 新密钥
  newKey: arn:aws:kms:us-east-1:123456789:key/new-key-id
  # 或使用 MinIO KMS
  # newKey: my-new-encryption-key
```

```bash
# 启动密钥轮转
mc batch start myminio batch-keyrotate.yaml

# 查看进度
mc batch status myminio <job-id>
```

## 4. 批量脚本 (Batch Script)

对桶内对象执行自定义脚本，适合复杂的数据处理逻辑。

```yaml
# batch-script.yaml
script:
  apiVersion: v1
  source:
    bucket: process-bucket
    prefix: incoming/
    # 并发处理数
    concurrency: 5
  # 对每个对象执行的脚本
  script: |-
    #!/bin/bash
    echo "Processing: ${MINIO_SRC_FILE}"
    # ${MINIO_SRC_FILE} 是自动注入的环境变量
    # ${MINIO_SRC_BUCKET} 源桶名
    # ${MINIO_SRC_PATH} 对象路径

    # 示例：检查文件大小并记录
    SIZE=$(stat -c%s "${MINIO_SRC_FILE}")
    if [ $SIZE -gt $((100*1024*1024)) ]; then
      echo "WARNING: Large file detected: ${MINIO_SRC_FILE} (${SIZE} bytes)"
    fi
```

```bash
# 运行批量脚本
mc batch start myminio batch-script.yaml
```

### 脚本环境变量

| 变量 | 说明 |
|------|------|
| `MINIO_SRC_FILE` | 源对象文件路径（以桶为根） |
| `MINIO_SRC_BUCKET` | 源桶名 |
| `MINIO_SRC_PATH` | 对象在桶中的路径 |
| `MINIO_SRC_ETAG` | 对象的 ETag |
| `MINIO_SRC_VERSION_ID` | 对象的版本 ID（如启用了版本控制） |

## 注意事项

1. **权限要求**：执行批量操作需要 `s3:ReplicateObject` / `s3:PutLifecycleConfiguration` / `kms:Decrypt+Encrypt` 等对应 IAM 权限
2. **作业持久化**：批量作业记录存储在 MinIO 内部，重启后不会丢失
3. **并发控制**：批量复制和脚本支持 `concurrency` 参数控制并行度，建议根据集群资源调整
4. **监控**：使用 `mc admin trace --type batch` 实时追踪作业执行
5. **重试策略**：`retry.attempts` 和 `retry.delay` 控制失败重试（仅批量复制支持）
6. **通知**：支持通过 Webhook 通知作业完成，适合集成 CI/CD 流程
7. **版本控制**：如果桶启用了版本控制，批量过期默认只删除当前版本，历史版本需额外规则
8. **跨集群复制需配置 DNS 可达**：目标集群的 endpoint 必须从源集群可访问
