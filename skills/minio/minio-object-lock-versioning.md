---
name: minio-object-lock-versioning
description: MinIO 对象锁定（WORM / Object Lock）与版本控制实战
tags: [minio, object-lock, versioning, retention, legal-hold]
---

## MinIO 对象锁定（Object Lock）

对象锁定实现 **WORM（Write Once Read Many）** 模型，防止数据被修改或删除。适用于合规归档、日志审计、医疗影像等场景。

### 启用条件

> ⚠️ 对象锁定**只能在创建存储桶时启用**，创建后无法更改

```bash
# mc 命令创建带锁定的存储桶
mc mb --with-lock myminio/compliance-bucket

# 或通过 Java SDK 创建
```

```java
import io.minio.*;
import io.minio.messages.ObjectLockConfiguration;
import io.minio.messages.RetentionMode;
import io.minio.messages.RetentionDurationDays;

MakeBucketArgs args = MakeBucketArgs.builder()
    .bucket("compliance-bucket")
    .objectLock(true)  // 必须在创建时开启
    .build();
minioClient.makeBucket(args);
```

### 保留模式

| 模式 | 说明 | 使用场景 |
|------|------|----------|
| **GOVERNANCE**（治理模式） | 有特殊权限的用户可以覆盖保留设置 | 日常合规管理，允许管理员修正 |
| **COMPLIANCE**（合规模式） | 在保留期内绝对不可修改/删除 | 法律合规、监管审计 |

### 设置默认保留策略

```java
// 存储桶级默认保留策略（对所有新上传对象生效）
SetObjectLockConfigArgs config = SetObjectLockConfigArgs.builder()
    .bucket("compliance-bucket")
    .config(new ObjectLockConfiguration(
        RetentionMode.GOVERNANCE,
        new RetentionDurationDays(365)
    ))
    .build();
minioClient.setObjectLockConfig(config);
```

### 上传时设置对象级保留

```java
// 上传文件时单独设置保留策略
PutObjectArgs putArgs = PutObjectArgs.builder()
    .bucket("compliance-bucket")
    .object("audit/2025-06-01.json")
    .stream(inputStream, fileSize, -1)
    .contentType("application/json")
    .retentionMode(RetentionMode.COMPLIANCE)
    .retentionDuration(365)  // 保留 365 天
    .build();
minioClient.putObject(putArgs);
```

## 版本控制

### 启用版本控制

```bash
# mc 命令
mc version enable myminio/my-bucket

# 或通过 SDK
SetBucketVersioningArgs args = SetBucketVersioningArgs.builder()
    .bucket("my-bucket")
    .config(new VersioningConfig(VersioningConfig.Status.ENABLED))
    .build();
minioClient.setBucketVersioning(args);
```

### 版本管理操作

```java
// 列出对象的版本历史
ListObjectsArgs listArgs = ListObjectsArgs.builder()
    .bucket("my-bucket")
    .prefix("documents/")
    .includeVersions(true)
    .build();
Iterable<Result<Item>> results = minioClient.listObjects(listArgs);
for (Result<Item> result : results) {
    Item item = result.get();
    System.out.printf("key=%s, versionId=%s, isLatest=%s%n",
        item.objectName(), item.versionId(), item.isLatest());
}

// 删除指定版本（软删除 = 只删除标记）
RemoveObjectArgs removeArgs = RemoveObjectArgs.builder()
    .bucket("my-bucket")
    .object("documents/report.pdf")
    .versionId("version-id-xxx")
    .build();
minioClient.removeObject(removeArgs);

// 恢复旧版本 = 复制旧版本覆盖当前
CopyObjectArgs copyArgs = CopyObjectArgs.builder()
    .bucket("my-bucket")
    .object("documents/report.pdf")
    .source(CopySource.builder()
        .bucket("my-bucket")
        .object("documents/report.pdf")
        .versionId("version-id-xxx")
        .build())
    .build();
minioClient.copyObject(copyArgs);
```

### 删除标记与垃圾回收

```
版本控制启用后，DELETE 操作规则：

┌─────────────────────────────────────────────────────────┐
│  操作                           │  结果                  │
├─────────────────────────────────────────────────────────┤
│  删除当前版本（无版本ID）       │  插入删除标记          │
│  删除指定版本（有版本ID）       │  永久删除该版本        │
│  删除删除标记                  │  恢复对象              │
│  上传同名对象                  │  创建新版本，旧版本保留  │
└─────────────────────────────────────────────────────────┘
```

## Legal Hold（法定保留）

与保留策略独立，用于法律调查场景下锁定特定版本。

```java
// 设置 Legal Hold（锁定文件作为法律证据）
SetObjectLegalHoldArgs legalArgs = SetObjectLegalHoldArgs.builder()
    .bucket("compliance-bucket")
    .object("audit/2025-Q1-report.pdf")
    .versionId(versionId)
    .legalHold(true)
    .build();
minioClient.setObjectLegalHold(legalArgs);

// 检查 Legal Hold 状态
GetObjectLegalHoldArgs getLegalArgs = GetObjectLegalHoldArgs.builder()
    .bucket("compliance-bucket")
    .object("audit/2025-Q1-report.pdf")
    .versionId(versionId)
    .build();
LegalHold hold = minioClient.getObjectLegalHold(getLegalArgs);
System.out.println("Legal hold: " + hold.status());
```

## 生命周期管理 + 版本控制

### 规则配置

```json
{
  "Rules": [
    {
      "ID": "auto-cleanup",
      "Status": "Enabled",
      "Filter": {"Prefix": "temp/"},
      "Expiration": {
        "Days": 7,
        "ExpiredObjectDeleteMarker": true
      }
    },
    {
      "ID": "archive-old-versions",
      "Status": "Enabled",
      "Filter": {"Prefix": "documents/"},
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90
      }
    }
  ]
}
```

### Java SDK 配置生命周期

```java
String bucketPolicy = """
{
  "Rules": [
    {
      "ID": "delete-after-30d",
      "Status": "Enabled",
      "Filter": {"Prefix": "logs/"},
      "Expiration": {"Days": 30}
    }
  ]
}
""";

SetBucketLifecycleArgs lifecycleArgs = SetBucketLifecycleArgs.builder()
    .bucket("my-bucket")
    .lifecycleConfig(LifecycleConfig.fromXml(bucketPolicy))
    .build();
minioClient.setBucketLifecycle(lifecycleArgs);
```

## 对象锁定 + 版本控制 + 加密组合使用

```java
import io.minio.*;
import io.minio.messages.*;

// 创建带锁定的加密存储桶
MakeBucketArgs bucketArgs = MakeBucketArgs.builder()
    .bucket("secure-compliance")
    .objectLock(true)
    .build();
minioClient.makeBucket(bucketArgs);

// 启用版本控制
minioClient.setBucketVersioning(SetBucketVersioningArgs.builder()
    .bucket("secure-compliance")
    .config(new VersioningConfig(VersioningConfig.Status.ENABLED))
    .build());

// 上传对象：加密 + 锁定 + 版本控制
PutObjectArgs putArgs = PutObjectArgs.builder()
    .bucket("secure-compliance")
    .object("finance/2025-tax-report.xlsx")
    .stream(inputStream, fileSize, -1)
    .ssec(new ServerSideEncryptionWithCustomerKey(encryptionKey))
    .retentionMode(RetentionMode.COMPLIANCE)
    .retentionDuration(2555) // 7 年
    .build();
minioClient.putObject(putArgs);
```

## 注意事项

1. **锁定不可逆**：COMPLIANCE 模式下，保留期内无法删除任何版本，包括过期日志清理
2. **版本爆炸**：启用版本控制后频繁上传同名对象会积累大量版本，需配合生命周期规则清理非当前版本
3. **Legal Hold 优先级**：Legal Hold 优先级高于保留模式——Legal Hold 开启时，即使保留期已过，对象仍不可删除
4. **存储成本**：版本控制和对象锁定会增加存储用量，监控 bucket 大小设置配额告警
5. **mc 管理命令**：
   ```bash
   mc retention set --mode compliance --duration 365d myminio/bucket/file
   mc legal-hold set myminio/bucket/file
   mc version info myminio/bucket
   ```
