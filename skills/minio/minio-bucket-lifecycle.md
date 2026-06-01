---
name: minio-bucket-lifecycle
description: MinIO 存储桶生命周期管理：ILM 策略实现自动过期、分层转换与合规清理
tags: [minio, lifecycle, ilm, object-management, retention, s3]
---

## 概述

MinIO 的 ILM（Lifecycle Management）策略支持对存储桶中的对象**自动执行过期删除**和**分层转换**操作。适用于日志过期清理、冷热数据分层、合规自动删除等场景。

> ILM 策略与 Object Lock 可同时使用，但受 Object Lock 保护的对象不会被 ILM 删除。

## 1. ILM 策略规则组成

每条规则包含：

| 要素 | 说明 |
|------|------|
| **ID** | 规则唯一标识 |
| **Status** | `Enabled` / `Disabled` |
| **Filter** | 按前缀、标签过滤对象 |
| **Expiration** | 过期删除配置 |
| **Transition** | 分层转换配置（MinIO 无存储类转换，但兼容 S3 API） |

## 2. 过期删除规则

### 按天数过期

```xml
<!-- Lifecycle XML 配置 -->
<LifecycleConfiguration>
  <Rule>
    <ID>expire-old-logs</ID>
    <Status>Enabled</Status>
    <Filter>
      <Prefix>logs/</Prefix>
    </Filter>
    <Expiration>
      <Days>30</Days>
    </Expiration>
  </Rule>
</LifecycleConfiguration>
```

### 按日期删除未完成的分段上传

```xml
<Rule>
  <ID>abort-incomplete-upload</ID>
  <Status>Enabled</Status>
  <Filter>
    <Prefix>uploads/</Prefix>
  </Filter>
  <AbortIncompleteMultipartUpload>
    <DaysAfterInitiation>7</DaysAfterInitiation>
  </AbortIncompleteMultipartUpload>
</Rule>
```

## 3. 使用 mc 命令管理生命周期

```bash
# 查看当前 lifecycle 配置
mc ilm list myminio/my-bucket

# 导入 lifecycle 配置（XML 或 JSON）
mc ilm import myminio/my-bucket < lifecycle.xml

# 导出 lifecycle 配置
mc ilm export myminio/my-bucket

# 添加过期规则（一键语法）
mc ilm add --expire-days "90" --prefix "temp/" myminio/my-bucket

# 添加分段上传清理规则
mc ilm add --expire-days "1" --filter-upload myminio/my-bucket

# 删除规则
mc ilm remove --id "expire-old-logs" myminio/my-bucket

# 设置 RF（Replication Factor）相关规则
mc ilm add --expire-days "0" --prefix "trash/" myminio/my-bucket
```

## 4. 使用 Java SDK 设置生命周期

```java
import io.minio.*;
import io.minio.messages.lifecycle.*;

// 创建过期规则
LifecycleRule rule = LifecycleRule.builder()
    .id("expire-logs")
    .status(LifecycleRule.Status.ENABLED)
    .filter(LifecycleFilter.builder()
        .prefix("logs/")
        .build())
    .expiration(LifecycleExpiration.builder()
        .days(30)
        .build())
    .build();

// 应用到存储桶
LifecycleConfiguration config = LifecycleConfiguration.builder()
    .rule(rule)
    .build();

minioClient.setBucketLifecycle(
    SetBucketLifecycleArgs.builder()
        .bucket("my-bucket")
        .config(config)
        .build()
);

// 读取生命周期配置
LifecycleConfiguration existing = minioClient.getBucketLifecycle(
    GetBucketLifecycleArgs.builder()
        .bucket("my-bucket")
        .build()
);

for (LifecycleRule r : existing.rules()) {
    System.out.println(r.id() + " -> " + r.status());
}
```

## 5. 典型场景配置

### 场景一：日志桶自动清理

保留 90 天，同时清理未完成的上传碎片：

```json
{
  "Rules": [
    {
      "ID": "clean-app-logs",
      "Status": "Enabled",
      "Filter": {"Prefix": "app/"},
      "Expiration": {"Days": 90}
    },
    {
      "ID": "clean-nginx-logs",
      "Status": "Enabled",
      "Filter": {"Prefix": "nginx/"},
      "Expiration": {"Days": 30}
    },
    {
      "ID": "abort-old-uploads",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 1}
    }
  ]
}
```

### 场景二：数据湖冷热分层（兼容 S3 Transition）

MinIO 自身不支持存储类自动转换，但可通过组合事件通知实现：

```bash
# 方案：事件通知 + MinIO Client 复制
mc event add myminio/data-lake arn:minio:sqs::mywebhook:webhook \
  --event put --suffix ".data"

# 由 webhook 服务根据对象修改时间判断是否移动到"冷存储"前缀
mc mv myminio/data-lake/hot/2025/ data-lake/cold/2025/
```

### 场景三：合规保留期后的自动删除

```json
{
  "Rules": [{
    "ID": "legal-hold-release",
    "Status": "Enabled",
    "Filter": {"Prefix": "legal/"},
    "Expiration": {"Days": 2555}
  }]
}
```

> 搭配 Object Lock 使用：7 年保留期内对象不可删除，到期后 ILM 自动清理

## 6. MinIO 控制台操作

1. 登录 MinIO Console → 选择存储桶
2. 进入 **Lifecycle** 标签页
3. 点击 **Add Lifecycle Rule**
4. 配置过滤条件（前缀、标签）和过期天数
5. 保存后立即生效（分钟级延迟）

## 注意事项

### ⚠️ 规则生效延迟
- ILM 规则设置后，MinIO 每隔 **24 小时**执行一次扫描清理
- 可通过 `mc ilm ls --watch` 查看执行状态
- 强制立即清理：`mc ilm add --expire-days "0" <bucket/>`

### ⚠️ 版本控制下的行为
- 桶启用版本控制后，`Expiration` 删除的是**当前版本**
- 若要删除所有历史版本，需设置 `Expiration.DeleteMarker` 相关规则
- 非当前版本（删除标记）到达天数后自动清理

### ⚠️ 性能影响
- 大量小文件的 ILM 扫描会消耗 I/O，建议分批设置规则
- 每秒处理的过期对象数受限于磁盘 I/O
- 针对超大桶（百万级对象），ILM 扫描时间可达数小时

### ⚠️ 资源开销
- 定期检查 `mc admin info myminio` 中的磁盘使用趋势
- 配合监控告警：设置桶容量超过 80% 时触发告警
