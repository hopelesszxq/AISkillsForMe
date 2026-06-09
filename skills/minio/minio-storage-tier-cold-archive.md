---
name: minio-storage-tier-cold-archive
description: MinIO 对象存储冷热分层与归档——ILM Transition 配置跨集群 Tier、低成本归档存储实战
tags: [minio, storage-tier, cold-archive, ilm, transition, cost-optimization]
---

## 概述

海量对象存储中，数据通常有**冷热分层**需求：
- **热数据**：频繁访问，存放在高性能 SSD 集群
- **温数据**：偶尔访问，存放在普通 HDD 集群
- **冷数据**：几乎不访问，存放在低成本归档集群

MinIO 从 v2023 起支持 **ILM Transition** 规则，自动将对象从源集群迁移到目标 Tier（另一个 MinIO 集群或 S3 兼容存储），实现生命周期自动化管理。

## 架构设计

```
┌─────────────────────────────────────────────────────┐
│                  热集群（SSD/NVMe）                    │
│      ILM Transition: 30天后 → 温集群                    │
├─────────────────────────────────────────────────────┤
│                  温集群（HDD/SATA）                    │
│      ILM Transition: 90天后 → 冷集群                    │
├─────────────────────────────────────────────────────┤
│                  冷集群（归档 HDD / 低频存储）            │
│      ILM Expiration: 365天后删除                       │
└─────────────────────────────────────────────────────┘
```

## 1. 配置远程 Tier

### 1.1 使用 mc 命令行配置 Tier 目标

```bash
# 1. 查看当前 Tier
mc ilm tier list myminio

# 2. 添加 S3 兼容 Tier（目标可以是另一个 MinIO、AWS S3、Backblaze B2 等）
mc ilm tier add myminio \
    --endpoint https://cold-storage.example.com \
    --access-key COLD_ACCESS_KEY \
    --secret-key COLD_SECRET_KEY \
    --name cold-tier \
    --bucket cold-archive

# 3. 验证 Tier 连通性
mc ilm tier check myminio cold-tier
```

### 1.2 支持的 Tier 后端类型

| 类型 | 配置示例 | 适用场景 |
|------|---------|---------|
| MinIO (S3) | `--endpoint https://minio-cold:9000` | 自建冷热分层 |
| AWS S3 | `--endpoint https://s3.amazonaws.com` | 混合云归档 |
| AWS S3 Glacier | 通过 S3 Lifecycle 进一步转 Glacier | 超低成本归档 |
| Backblaze B2 | `--endpoint https://s3.us-west-001.backblazeb2.com` | 低成本云归档 |
| 任何 S3 兼容 | 标准 S3 API | 私有云/IDC 部署 |

### 1.3 通过 Java SDK 配置 Tier

```java
import io.minio.*;
import io.minio.admin.TierConfig;
import io.minio.admin.TierType;

public class TierSetup {
    
    public static void addColdTier(MinioClient adminClient) throws Exception {
        TierConfig tier = TierConfig.builder()
            .name("cold-tier")
            .type(TierType.MINIO)  // S3 / GCS / AZURE / MINIO
            .endpoint("https://cold-storage.example.com")
            .accessKey("COLD_ACCESS_KEY")
            .secretKey("COLD_SECRET_KEY")
            .bucket("cold-archive")
            .region("cn-east-1")
            .build();
        
        adminClient.setBucketTier(tier);
        System.out.println("Tier 'cold-tier' configured");
    }
}
```

## 2. 配置 ILM Transition 规则

### 2.1 通过 mc 配置

```bash
# 创建 ILM 规则：30天后转温集群，90天后转冷集群
cat <<EOF > /tmp/ilm-tier.json
{
    "Rules": [
        {
            "ID": "tier-to-warm",
            "Status": "Enabled",
            "Filter": {
                "Prefix": "uploads/"
            },
            "Transitions": [
                {
                    "Days": 30,
                    "StorageClass": "warm-tier"
                },
                {
                    "Days": 90,
                    "StorageClass": "cold-tier"
                }
            ],
            "Expiration": {
                "Days": 365
            }
        },
        {
            "ID": "logs-expire",
            "Status": "Enabled",
            "Filter": {
                "Prefix": "logs/"
            },
            "Transitions": [
                {
                    "Days": 7,
                    "StorageClass": "cold-tier"
                }
            ],
            "Expiration": {
                "Days": 90
            }
        }
    ]
}
EOF

# 导入规则
mc ilm import myminio/data-bucket < /tmp/ilm-tier.json

# 查看现有规则
mc ilm export myminio/data-bucket
```

### 2.2 通过 Java SDK 配置 Transition 规则

```java
import io.minio.*;
import io.minio.metadata.lifecycle.*;

public class TransitionRuleSetup {
    
    public static void setupTransitionRules(MinioClient client) throws Exception {
        
        // 规则 1：uploads/ 前缀的对象 30天→温，90天→冷，365天删除
        LifecycleRule rule1 = LifecycleRule.builder()
            .id("uploads-tiering")
            .status(LifecycleRule.Status.ENABLED)
            .filter(LifecycleFilter.builder()
                .prefix("uploads/")
                .build())
            .transitions(List.of(
                Transition.builder()
                    .days(30)
                    .storageClass("warm-tier")
                    .build(),
                Transition.builder()
                    .days(90)
                    .storageClass("cold-tier")
                    .build()
            ))
            .expiration(Expiration.builder()
                .days(365)
                .build())
            .build();
        
        // 规则 2：logs/ 前缀的对象 7天直接→冷，90天删除
        LifecycleRule rule2 = LifecycleRule.builder()
            .id("logs-tiering")
            .status(LifecycleRule.Status.ENABLED)
            .filter(LifecycleFilter.builder()
                .prefix("logs/")
                .build())
            .transitions(List.of(
                Transition.builder()
                    .days(7)
                    .storageClass("cold-tier")
                    .build()
            ))
            .expiration(Expiration.builder()
                .days(90)
                .build())
            .build();
        
        // 应用到 bucket
        client.setBucketLifecycle(
            SetBucketLifecycleArgs.builder()
                .bucket("data-bucket")
                .rules(List.of(rule1, rule2))
                .build()
        );
    }
}
```

## 3. 冷集群配置（归档集群）

### 3.1 部署归档 MinIO

```yaml
# docker-compose.yml — 冷存储集群
services:
  minio-cold:
    image: minio/minio:latest
    command: server --console-address ":9001" /data
    ports:
      - "9002:9000"
      - "9003:9001"
    environment:
      MINIO_ROOT_USER: cold-admin
      MINIO_ROOT_PASSWORD: ${COLD_MINIO_PASSWORD}
      MINIO_STORAGE_CLASS_STANDARD: EC:2      # 纠删码 2 副本
    volumes:
      - /mnt/hdd-archive:/data   # 低成本 HDD 存储
    deploy:
      resources:
        limits:
          memory: 8G
```

### 3.2 冷集群生命周期

```bash
# 冷集群自身也可以设置 ILM，进一步管理归档数据
mc ilm rule add cold-clone/cold-archive \
    --expire-days 365 \
    --filter-prefix "uploads/"

# 禁止冷集群写入（只读归档）
mc anonymous set readonly cold-clone/cold-archive
```

## 4. Transition 数据验证与监控

### 4.1 检查对象存储层

```bash
# 查看单个对象的存储层
mc stat myminio/data-bucket/uploads/2026/01/doc.pdf

# 查看对象的 Transition 状态
mc stat --json myminio/data-bucket/uploads/2026/01/doc.pdf | jq .metadata

# 检查 Tier 统计
mc admin tier info myminio cold-tier
```

### 4.2 监控指标

```java
// 通过 Prometheus 指标监控 Tier 使用情况
// MinIO 暴露以下 Tier 相关指标：
//   minio_bucket_ilm_transition_pending_objects  — 待迁移对象数
//   minio_bucket_ilm_transition_failed_objects   — 迁移失败对象数
//   minio_ilm_transition_bytes                    — 已迁移总字节数
//   minio_tier_usage_bytes                        — 各 Tier 使用量
```

```yaml
# Prometheus 告警规则
groups:
  - name: minio-tier
    rules:
      - alert: MinioTransitionFailed
        expr: minio_bucket_ilm_transition_failed_objects > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MinIO ILM Transition 失败"
          description: "{{ $value }} 个对象迁移失败"

      - alert: MinioTierDiskFull
        expr: minio_tier_usage_bytes / minio_tier_capacity_bytes > 0.85
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "冷存储集群即将写满"
```

## 5. 分块上传对象的 Transition 处理

```java
/**
 * 已完成的分块上传对象同样受 ILM Transition 管理
 * 但未完成的分块上传不会触发 Transition
 */
public class MultipartTransition {
    
    // 建议设置 AbortIncompleteMultipartUpload 规则
    public static LifecycleRule abortMultipartRule() {
        return LifecycleRule.builder()
            .id("abort-incomplete-multipart")
            .status(LifecycleRule.Status.ENABLED)
            .filter(LifecycleFilter.builder()
                .prefix("")
                .build())
            .abortIncompleteMultipartUpload(
                AbortIncompleteMultipartUpload.builder()
                    .daysAfterInitiation(7)
                    .build())
            .build();
    }
}
```

## 6. 场景示例：用户文件存储分层

```yaml
# 用户文件存储分层策略
bucket: user-files
rules:
  - id: user-files-tiering
    status: enabled
    filter:
      prefix: ""
    transitions:
      - days: 0         # 立即迁移旧文件
        storage_class: warm-tier
      - days: 90        # 90 天以上 → 冷
        storage_class: cold-tier
    expiration:
      days: 730         # 2 年后删除（合规要求）
```

```bash
# 对已有对象批量设置 Transition（immutable 版本化数据）
mc ilm rule add myminio/user-files \
    --transition-days 90 \
    --transition-tier cold-tier \
    --expire-days 730 \
    --noncurrent-transition-days 30 \      # 非当前版本 30 天后转冷
    --noncurrent-expire-days 180           # 非当前版本 180 天后删除
```

## 注意事项

1. **Transition 不可逆**：对象迁移到 Tier 后，无法自动回迁到原集群。读取时 MinIO 会自动从 Tier 拉取，但延迟显著增加
2. **读取性能**：冷数据读取时 MinIO 需要从远程 Tier 拉取，首次访问延迟从毫秒级变为秒级。建议结合 `presigned-url` 设置更长的过期时间
3. **带宽成本**：跨集群 Transition 会占用网络带宽，建议在业务低峰期（凌晨）集中执行
4. **配额规划**：Tier 目标集群需要预留足够的存储空间，迁移失败会导致对象停留在原集群
5. **版本控制**：开启版本控制的 bucket 中，Transition 规则默认作用于当前版本，非当前版本需额外配置 `NoncurrentVersionTransition`
6. **对象锁定**：受 Object Lock 保护的对象仍可 Transition，但受保留期限约束
7. **Tier 网络**：源集群与 Tier 集群之间的网络必须稳定可靠，建议使用专线或内网
8. **mc ilm tier check**：配置 Tier 后务必执行连通性检查，避免 Transition 失败
9. **删除行为**：Transition 到 Tier 的对象在原集群会保留标记（约 500 字节），原空间几乎完全释放
