---
name: minio-events
description: MinIO 事件通知、存储桶复制与运维最佳实践
tags: [minio, events, replication, iam, notification]
---

## 事件通知 (Bucket Notification)

支持将事件推送到多种目标：

```bash
# 配置 Webhook 事件通知
mc admin config set myminio notify_webhook:orders \
  endpoint="https://webhook.example.com/minio-events" \
  auth_token="your-token" \
  queue_limit="1000"

mc admin service restart myminio

# 设置存储桶事件监听
mc event add myminio/orders arn:minio:sqs::orders:webhook \
  --event put,delete \
  --suffix ".jpg"

# 查看事件配置
mc event list myminio/orders
```

### 支持的事件目标

| 目标    | 协议/方式                     | 典型场景               |
|---------|-------------------------------|------------------------|
| Webhook | HTTP POST JSON payload        | 自定义处理逻辑         |
| Kafka   | Kafka Producer                | 大数据流处理           |
| MySQL   | JDBC 写入                     | 数据库记录同步         |
| Redis   | Pub/Sub                       | 实时通知               |
| Elasticsearch | REST API 写入            | 日志/元数据搜索        |
| PostgreSQL | JDBC 写入                  | 结构化数据同步         |

### 事件类型

| 事件                | 触发时机               |
|---------------------|------------------------|
| `s3:ObjectCreated:*` | 上传/复制/Put/Post    |
| `s3:ObjectRemoved:*` | 删除对象              |
| `s3:ObjectAccessed:*`| GetObject/HeadObject  |

## 存储桶复制 (Bucket Replication)

实现异地备份或跨可用区数据同步：

```bash
# 前提：源和目标配置各自 MinIO 别名
mc alias set source https://source.minio.io:9000 accessKey secretKey
mc alias set dest https://dest.minio.io:9000 accessKey secretKey

# 目标桶创建
mc mb dest/backup-bucket

# 配置复制规则（JSON）
cat > replication.json << 'EOF'
{
  "Role": "",
  "Rules": [
    {
      "Status": "Enabled",
      "Priority": 1,
      "DeleteMarkerReplication": { "Status": "Disabled" },
      "Filter": { "Prefix": "" },
      "Destination": { 
        "Bucket": "arn:aws:s3:::backup-bucket",
        "StorageClass": "STANDARD"
      },
      "ID": "all-objects"
    }
  ]
}
EOF

# 应用复制规则
mc replicate add source/orders --remote-bucket dest/backup-bucket \
  --arn "arn:minio:replication::dest:backup-bucket"

# 查看复制状态
mc replicate status source/orders
```

## IAM 策略管理

### 基于路径的细粒度权限

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::user-uploads/${aws:username}/*"
    },
    {
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::user-uploads/*"
    }
  ]
}
```

```bash
# 创建用户并分配策略
mc admin user add myminio uploader uploaderPass123
mc admin policy create myminio uploader-policy uploader-policy.json
mc admin policy attach myminio uploader-policy --user uploader
```

## 客户端操作进阶

```bash
# 磁盘统计
mc admin info myminio

# 设置存储桶生命周期（30天自动清理）
mc ilm rule add myminio/logs --expire-days 30

# 设置存储桶配额
mc admin bucket quota myminio/logs --hard 500GB

# 匿名下载（生成预签名 URL，7天下载链接）
mc share download myminio/backups/db-2025.sql

# 查看存储桶使用量
mc du myminio/orders
```

## Java SDK 集成示例

```java
// Maven
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.12</version>
</dependency>

// 监听 Bucket 通知（使用 MinIO Client 事件轮询）
MinioClient client = MinioClient.builder()
    .endpoint("https://play.min.io")
    .credentials("accessKey", "secretKey")
    .build();

// 监听通知事件
client.listenBucketNotification(
    ListenBucketNotificationArgs.builder()
        .bucket("orders")
        .prefix("images/")
        .events(Event.OBJECT_CREATED_PUT)
        .build()
);
```

## 运维建议

- **纠删码模式**：生产至少 4 个磁盘，使用 Erasure Coding（N/2 冗余容忍）
- **ETCD 集成**：集群模式（>=4 节点）搭配 ETCD 做分布式协调
- **TLS 加密**：使用 `mc admin config set` 配置 `MINIO_ROOT_CERT`，生产强制 HTTPS
- **监控**：Prometheus 集成（`/minio/v2/metrics/cluster`），Grafana 模板 ID: 14950
- **备份**：使用 `mc mirror` 定期同步到异地存储，或启用 Bucket Replication
- **版本控制**：启用 Bucket 版本管理，防止误删
