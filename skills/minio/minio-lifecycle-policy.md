---
name: minio-lifecycle-policy
description: MinIO 存储桶生命周期管理与访问策略——对象过期、自动分层、Bucket Policy 配置实战
tags: [minio, lifecycle, bucket-policy, object-storage, s3]
---

## 概述

MinIO 支持通过**生命周期规则**自动管理对象过期与分层，以及通过 **Bucket Policy** 精细控制访问权限。本文覆盖 Java SDK 操作与配置实践。

## 一、生命周期管理（对象过期与自动清理）

### 1.1 生命周期规则核心概念

| 规则类型 | 说明 | 适用场景 |
|---------|------|---------|
| Expiration | 对象过期后自动删除 | 临时文件、日志、备份 |
| Transition | 对象迁移到不同存储层 | 冷热数据分层（MinIO 暂不支持直接分层，需自定义实现） |
| NoncurrentVersion Expiration | 非当前版本过期删除 | 开启版本控制时的历史版本清理 |
| AbortIncompleteMultipartUpload | 未完成的分片上传超时中止 | 防止残留分片占用存储 |

### 1.2 通过 MinIO Console 配置生命周期规则

```
1. 登录 MinIO Console → Buckets → 选择 bucket
2. 切换到 "Lifecycle" 选项卡
3. 添加规则：
   - 规则名称：expire-temp-files
   - 前缀：temp/  （只作用于 temp/ 目录下的对象）
   - 过期天数：7 天
   - 标签：可选的过滤标签
4. 保存
```

### 1.3 使用 Java SDK 配置生命周期

```java
import io.minio.*;
import io.minio.messages.*;
import java.util.List;

@Service
@RequiredArgsConstructor
public class BucketLifecycleService {

    private final MinioClient minioClient;

    /**
     * 为存储桶设置生命周期规则
     */
    public void setLifecycleRules(String bucket) {
        // 规则1：temp/ 前缀的文件 3 天后过期
        LifecycleRule rule1 = LifecycleRule.builder()
            .id("expire-temp-files")
            .status(Status.ENABLED)
            .filter(Filter.builder()
                .prefix("temp/")
                .build())
            .expiration(Expiration.builder()
                .days(3)
                .build())
            .build();

        // 规则2：logs/ 前缀的文件 30 天后过期
        LifecycleRule rule2 = LifecycleRule.builder()
            .id("expire-old-logs")
            .status(Status.ENABLED)
            .filter(Filter.builder()
                .prefix("logs/")
                .build())
            .expiration(Expiration.builder()
                .days(30)
                .build())
            .build();

        // 规则3：删除超过 7 天的非当前版本（版本控制启用时）
        LifecycleRule rule3 = LifecycleRule.builder()
            .id("expire-old-versions")
            .status(Status.ENABLED)
            .filter(Filter.builder()
                .prefix("")
                .build())
            .noncurrentVersionExpiration(NoncurrentVersionExpiration.builder()
                .noncurrentDays(7)
                .build())
            .build();

        // 规则4：中止超过 1 天的未完成分片上传
        LifecycleRule rule4 = LifecycleRule.builder()
            .id("abort-incomplete-uploads")
            .status(Status.ENABLED)
            .filter(Filter.builder()
                .prefix("")
                .build())
            .abortIncompleteMultipartUpload(AbortIncompleteMultipartUpload.builder()
                .daysAfterInitiation(1)
                .build())
            .build();

        List<LifecycleRule> rules = List.of(rule1, rule2, rule3, rule4);

        try {
            minioClient.setBucketLifecycle(
                SetBucketLifecycleArgs.builder()
                    .bucket(bucket)
                    .lifecycleConfiguration(
                        LifecycleConfiguration.builder()
                            .rules(rules)
                            .build())
                    .build()
            );
            log.info("Lifecycle rules set for bucket: {}", bucket);
        } catch (Exception e) {
            throw new BucketOperationException("设置生命周期规则失败", e);
        }
    }

    /**
     * 查看当前生命周期规则
     */
    public List<LifecycleRule> getLifecycleRules(String bucket) {
        try {
            LifecycleConfiguration config = minioClient.getBucketLifecycle(
                GetBucketLifecycleArgs.builder()
                    .bucket(bucket)
                    .build()
            );
            return config.rules();
        } catch (ErrorResponseException e) {
            if ("NoSuchLifecycleConfiguration".equals(e.errorResponse().code())) {
                return List.of(); // 没有配置规则
            }
            throw new BucketOperationException("获取生命周期规则失败", e);
        }
    }

    /**
     * 删除所有生命周期规则
     */
    public void deleteLifecycleRules(String bucket) {
        try {
            minioClient.deleteBucketLifecycle(
                DeleteBucketLifecycleArgs.builder()
                    .bucket(bucket)
                    .build()
            );
            log.info("Lifecycle rules deleted for bucket: {}", bucket);
        } catch (Exception e) {
            throw new BucketOperationException("删除生命周期规则失败", e);
        }
    }
}
```

### 1.4 使用 XML 格式配置

```java
/**
 * 直接使用 XML 字符串设置复杂生命周期规则
 */
public void setLifecycleXml(String bucket) {
    String xml = """
        <LifecycleConfiguration>
            <Rule>
                <ID>expire-old-backups</ID>
                <Status>Enabled</Status>
                <Filter>
                    <Prefix>backup/</Prefix>
                </Filter>
                <Expiration>
                    <Days>90</Days>
                </Expiration>
            </Rule>
            <Rule>
                <ID>abort-stale-uploads</ID>
                <Status>Enabled</Status>
                <Filter>
                    <Prefix></Prefix>
                </Filter>
                <AbortIncompleteMultipartUpload>
                    <DaysAfterInitiation>1</DaysAfterInitiation>
                </AbortIncompleteMultipartUpload>
            </Rule>
        </LifecycleConfiguration>
        """;

    try {
        byte[] configBytes = xml.getBytes(StandardCharsets.UTF_8);
        // 使用 AWS S3 SDK 方式发送
        // ... 通过 AmazonS3 client.setBucketLifecycleConfiguration(...)
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

## 二、Bucket Policy（桶级访问策略）

### 2.1 常见策略场景

| 策略类型 | 效果 | 适用场景 |
|---------|------|---------|
| 公开读 | 允许匿名 GET 请求 | 静态资源托管（图片、CSS） |
| 公开读写 | 允许匿名上传和下载 | 临时文件共享 |
| IP 限制 | 只允许特定 IP 访问 | 内部系统文件 |
| 条件读写 | 要求特定 Header 或来源 | 防盗链 |

### 2.2 Java SDK 配置 Bucket Policy

```java
@Service
@RequiredArgsConstructor
public class BucketPolicyService {

    private final MinioClient minioClient;

    /**
     * 设置只读公开策略（允许匿名读）
     */
    public void setPublicReadPolicy(String bucket) {
        String policy = """
            {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {"AWS": ["*"]},
                        "Action": ["s3:GetObject"],
                        "Resource": ["arn:aws:s3:::%s/*"]
                    }
                ]
            }
            """.formatted(bucket);

        applyPolicy(bucket, policy);
    }

    /**
     * 设置仅限特定 IP 访问的策略
     */
    public void setIpRestrictedPolicy(String bucket, List<String> allowedIps) {
        String ipConditions = allowedIps.stream()
            .map(ip -> "\"" + ip + "\"")
            .collect(Collectors.joining(", "));

        String policy = """
            {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Deny",
                        "Principal": {"AWS": ["*"]},
                        "Action": ["s3:*"],
                        "Resource": ["arn:aws:s3:::%s/*", "arn:aws:s3:::%s"],
                        "Condition": {
                            "NotIpAddress": {
                                "aws:SourceIp": [%s]
                            }
                        }
                    },
                    {
                        "Effect": "Allow",
                        "Principal": {"AWS": ["*"]},
                        "Action": ["s3:GetObject"],
                        "Resource": ["arn:aws:s3:::%s/*"]
                    }
                ]
            }
            """.formatted(bucket, bucket, ipConditions, bucket);

        applyPolicy(bucket, policy);
    }

    /**
     * 设置资源上传政策（只允许 PUT，且限制 Content-Type）
     */
    public void setUploadOnlyPolicy(String bucket) {
        String policy = """
            {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {"AWS": ["*"]},
                        "Action": ["s3:PutObject"],
                        "Resource": ["arn:aws:s3:::%s/uploads/*"],
                        "Condition": {
                            "StringEquals": {
                                "s3:x-amz-acl": ["public-read"]
                            }
                        }
                    }
                ]
            }
            """.formatted(bucket);

        applyPolicy(bucket, policy);
    }

    /**
     * 防盗链（只允许指定域名引用）
     */
    public void setRefererRestrictedPolicy(String bucket, String allowedDomain) {
        String policy = """
            {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Deny",
                        "Principal": {"AWS": ["*"]},
                        "Action": ["s3:GetObject"],
                        "Resource": ["arn:aws:s3:::%s/*"],
                        "Condition": {
                            "StringNotLike": {
                                "aws:Referer": ["https://%s/*", "http://%s/*"]
                            }
                        }
                    },
                    {
                        "Effect": "Allow",
                        "Principal": {"AWS": ["*"]},
                        "Action": ["s3:GetObject"],
                        "Resource": ["arn:aws:s3:::%s/*"]
                    }
                ]
            }
            """.formatted(bucket, allowedDomain, allowedDomain, bucket);

        applyPolicy(bucket, policy);
    }

    private void applyPolicy(String bucket, String policyJson) {
        try {
            minioClient.setBucketPolicy(
                SetBucketPolicyArgs.builder()
                    .bucket(bucket)
                    .config(policyJson)
                    .build()
            );
            log.info("Policy applied to bucket: {}", bucket);
        } catch (Exception e) {
            throw new BucketOperationException("设置Bucket策略失败", e);
        }
    }

    /**
     * 获取当前 Bucket Policy
     */
    public String getBucketPolicy(String bucket) {
        try {
            return minioClient.getBucketPolicy(
                GetBucketPolicyArgs.builder()
                    .bucket(bucket)
                    .build()
            );
        } catch (ErrorResponseException e) {
            if ("NoSuchBucketPolicy".equals(e.errorResponse().code())) {
                return null; // 无策略
            }
            throw new BucketOperationException("获取Bucket策略失败", e);
        }
    }

    /**
     * 删除 Bucket Policy
     */
    public void deleteBucketPolicy(String bucket) {
        try {
            minioClient.deleteBucketPolicy(
                DeleteBucketPolicyArgs.builder()
                    .bucket(bucket)
                    .build()
            );
        } catch (Exception e) {
            throw new BucketOperationException("删除Bucket策略失败", e);
        }
    }
}
```

## 三、版本控制

```java
@Service
public class BucketVersioningService {

    private final MinioClient minioClient;

    /**
     * 启用版本控制
     */
    public void enableVersioning(String bucket) {
        try {
            minioClient.setBucketVersioning(
                SetBucketVersioningArgs.builder()
                    .bucket(bucket)
                    .versioningConfig(
                        new VersioningConfig(VersioningConfig.Status.ENABLED)
                    )
                    .build()
            );
        } catch (Exception e) {
            throw new RuntimeException("启用版本控制失败", e);
        }
    }

    /**
     * 获取对象的所有版本
     */
    public List<VersionInfo> listVersions(String bucket, String objectName) {
        try {
            Iterable<Result<DeleteMarker>> versions = minioClient.listObjects(
                ListObjectsArgs.builder()
                    .bucket(bucket)
                    .prefix(objectName)
                    .includeVersions(true)
                    .build()
            );
            // 返回版本信息列表
            List<VersionInfo> result = new ArrayList<>();
            for (Result<DeleteMarker> v : versions) {
                result.add(new VersionInfo(v.get().versionId(), v.get().lastModified()));
            }
            return result;
        } catch (Exception e) {
            throw new RuntimeException("获取版本列表失败", e);
        }
    }
}
```

## 四、对象标签（Tagging）

利用标签与生命周期规则结合，实现更精细的对象管理：

```java
// 上传时打标签
PutObjectArgs args = PutObjectArgs.builder()
    .bucket("my-bucket")
    .object("logs/2026-06.log")
    .stream(inputStream, size, -1)
    .tags(Map.of(
        "env", "production",
        "retention", "30days",
        "project", "order-service"
    ))
    .build();
minioClient.putObject(args);
```

生命周期规则中按标签过滤：

```java
// 只清除 30 天前标记为 "retention=30days" 的对象
LifecycleRule rule = LifecycleRule.builder()
    .id("tag-based-expiration")
    .status(Status.ENABLED)
    .filter(Filter.builder()
        .and(And.builder()
            .prefix("logs/")
            .tags(List.of(new Tag("retention", "30days")))
            .build())
        .build())
    .expiration(Expiration.builder().days(30).build())
    .build();
```

## 五、存储桶事件通知

```java
/**
 * 配置存储桶事件通知（需要先部署 MinIO 事件目标，如 Webhook/Redis/AMQP）
 */
public void configureBucketNotification(String bucket) {
    // 配置存储桶事件通知需要通过 MinIO Client (mc) 或 AWS S3 SDK
    // MinIO Java SDK 暂不直接支持 setBucketNotification
    
    // 等价 mc 命令：
    // mc event add myminio/my-bucket arn:minio:sqs::webhook:event-url \
    //   --event put,delete --prefix uploads/
}
```

```bash
# mc 命令配置方式
mc admin config set myminio notify_webhook:1 \
  endpoint="http://your-server/webhook" \
  queue_limit="10"
mc event add myminio/my-bucket arn:minio:sqs::webhook:1 \
  --event put,delete --suffix .jpg,.png
```

## 六、生产推荐配置

### 6.1 多存储桶规划

| Bucket 名称 | 用途 | 生命周期 | 访问策略 | 版本控制 |
|------------|------|---------|---------|---------|
| app-uploads | 用户上传文件 | 保留 30 天 | 仅应用服务器写入 | 启用 |
| app-static | 静态资源（图片、CSS） | 永久保留 | 公开只读 | 禁用 |
| app-logs | 应用日志 | 保留 7 天 | 仅管理员访问 | 禁用 |
| app-backups | 数据库备份 | 保留 90 天 | 仅内部 IP 访问 | 启用 |
| app-temp | 临时处理文件 | 24 小时过期 | 私有 | 禁用 |

### 6.2 生命周期建议

```java
@PostConstruct
public void setupProductionBuckets() {
    // 创建生产环境的存储桶
    for (String bucket : List.of("app-uploads", "app-static", "app-logs", "app-backups", "app-temp")) {
        createBucketIfNotExists(bucket);
    }
    
    // 不同存储桶应用不同生命周期策略
    setLifecycleRules("app-temp", List.of(
        expireRule("expire-temp", "", 1)  // 24 小时过期
    ));
    
    setLifecycleRules("app-logs", List.of(
        expireRule("expire-logs-7d", "", 7),
        abortUploadRule("abort-stale", 1)
    ));
    
    setLifecycleRules("app-backups", List.of(
        expireRule("expire-backups-90d", "", 90),
        expireNoncurrentRule("keep-only-latest", 7) // 保留非当前版本 7 天
    ));
}
```

## 注意事项

- **生命周期生效时间**：规则设置后最长可能需要 24 小时才会完全生效（MinIO 每天凌晨检查执行）
- **不向后兼容**：删除生命周期规则后，已设置的过期时间仍然生效，只是不再继续新增
- **策略语法**：MinIO 使用 AWS IAM 策略语法（`2012-10-17`），字段大小写敏感
- **公开策略风险**：设置公开读取时务必确认不会泄漏敏感数据，建议配合 `--prefix` 限制目录
- **版本控制不可逆**：Bucket 启用版本控制后无法关闭，只能暂停（Suspended）
- **分片残留清理**：启用 `AbortIncompleteMultipartUpload` 规则可避免 uploads 目录残留 `.minio` 分片文件
- **IAM Policy 优先**：Bucket Policy + IAM User Policy 同时匹配时，Deny 始终优先于 Allow
