---
name: minio-sdk-java
description: MinIO Java SDK 实战：预签名 URL、分片上传、对象管理完整指南
tags: [minio, java, sdk, presigned-url, multipart-upload]
---

## 依赖配置

```xml
<!-- Maven -->
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.17</version>
</dependency>
```

```groovy
// Gradle
implementation 'io.minio:minio:8.5.17'
```

## MinIO Client 配置

```java
@Configuration
public class MinioConfig {

    @Value("${minio.endpoint:http://127.0.0.1:9000}")
    private String endpoint;

    @Value("${minio.access-key:minioadmin}")
    private String accessKey;

    @Value("${minio.secret-key:minioadmin}")
    private String secretKey;

    @Value("${minio.default-bucket:default}")
    private String defaultBucket;

    @Bean
    public MinioClient minioClient() {
        return MinioClient.builder()
            .endpoint(endpoint)
            .credentials(accessKey, secretKey)
            .region("cn-north-1")
            .build();
    }

    @PostConstruct
    public void initDefaultBucket() {
        try {
            boolean found = minioClient().bucketExists(
                BucketExistsArgs.builder().bucket(defaultBucket).build());
            if (!found) {
                minioClient().makeBucket(
                    MakeBucketArgs.builder().bucket(defaultBucket).build());
                log.info("默认 Bucket 已创建: {}", defaultBucket);
            }
        } catch (Exception e) {
            log.error("初始化 MinIO Bucket 失败", e);
        }
    }
}
```

## 预签名 URL（Presigned URL）

### 服务端生成上传 URL（前端直传）

```java
@Service
public class FileUploadService {

    @Autowired
    private MinioClient minioClient;

    @Value("${minio.default-bucket}")
    private String bucket;

    /**
     * 生成预签名上传 URL（前端直接上传到 MinIO，不经过应用服务器）
     */
    public String generateUploadUrl(String objectName, long expirySeconds) {
        try {
            return minioClient.getPresignedObjectUrl(
                GetPresignedObjectUrlArgs.builder()
                    .method(Method.PUT)                    // HTTP PUT 上传
                    .bucket(bucket)
                    .object(objectName)
                    .expiry(expirySeconds, TimeUnit.SECONDS) // URL 有效期
                    .build()
            );
        } catch (Exception e) {
            throw new RuntimeException("生成上传 URL 失败", e);
        }
    }

    /**
     * 生成预签名上传 URL（带 Content-Type 限制）
     */
    public String generateUploadUrlWithType(String objectName,
                                             String contentType,
                                             long expirySeconds) {
        Map<String, String> reqParams = new HashMap<>();
        reqParams.put("response-content-type", contentType);

        try {
            return minioClient.getPresignedObjectUrl(
                GetPresignedObjectUrlArgs.builder()
                    .method(Method.PUT)
                    .bucket(bucket)
                    .object(objectName)
                    .expiry(expirySeconds, TimeUnit.SECONDS)
                    .extraQueryParams(reqParams)
                    .build()
            );
        } catch (Exception e) {
            throw new RuntimeException("生成带类型的上传 URL 失败", e);
        }
    }
}
```

### 前端使用预签名 URL 上传

```javascript
// 1. 获取预签名 URL
const resp = await fetch('/api/files/upload-url?name=image.jpg&expiry=300');
const { url } = await resp.json();

// 2. 直接 PUT 文件到 MinIO
const file = document.getElementById('fileInput').files[0];
await fetch(url, {
  method: 'PUT',
  body: file,
  headers: { 'Content-Type': file.type }
});
```

### 生成下载/预览 URL

```java
/**
 * 生成预签名下载 URL（浏览器直接下载或预览）
 */
public String generateDownloadUrl(String objectName, long expirySeconds) {
    try {
        return minioClient.getPresignedObjectUrl(
            GetPresignedObjectUrlArgs.builder()
                .method(Method.GET)
                .bucket(bucket)
                .object(objectName)
                .expiry(expirySeconds, TimeUnit.SECONDS)
                .build()
        );
    } catch (Exception e) {
        throw new RuntimeException("生成下载 URL 失败", e);
    }
}

/**
 * 生成图片预览 URL（带缩放参数）
 */
public String generateImagePreviewUrl(String objectName, int width, int height) {
    Map<String, String> params = new HashMap<>();
    params.put("response-content-disposition", "inline");  // 浏览器直接预览
    params.put("x-oss-process", String.format("image/resize,m_fixed,w_%d,h_%d", width, height));

    // MinIO 默认不支持图片处理（需搭配 Imager 或 CDN）
    // 这里仅展示如何传递自定义 query 参数
    try {
        return minioClient.getPresignedObjectUrl(
            GetPresignedObjectUrlArgs.builder()
                .method(Method.GET)
                .bucket(bucket)
                .object(objectName)
                .expiry(3600, TimeUnit.SECONDS)
                .extraQueryParams(params)
                .build()
        );
    } catch (Exception e) {
        throw new RuntimeException("生成预览 URL 失败", e);
    }
}
```

## 服务端上传

```java
/**
 * 服务端上传文件（字节数组）
 */
public void uploadBytes(String objectName, byte[] data, String contentType) {
    try {
        ByteArrayInputStream bais = new ByteArrayInputStream(data);
        minioClient.putObject(
            PutObjectArgs.builder()
                .bucket(bucket)
                .object(objectName)
                .stream(bais, data.length, -1)  // -1 表示未知分片大小
                .contentType(contentType)
                .build()
        );
    } catch (Exception e) {
        throw new RuntimeException("上传失败", e);
    }
}

/**
 * 服务端上传文件（InputStream，用于大文件）
 */
public void uploadStream(String objectName,
                          InputStream stream,
                          long objectSize,
                          String contentType) {
    try {
        minioClient.putObject(
            PutObjectArgs.builder()
                .bucket(bucket)
                .object(objectName)
                .stream(stream, objectSize, PutObjectArgs.MIN_MULTIPART_SIZE)
                .contentType(contentType)
                .build()
        );
    } catch (Exception e) {
        throw new RuntimeException("上传失败", e);
    }
}
```

## 分片上传（Multipart Upload）

大文件（> 100MB）建议用分片上传，支持断点续传：

```java
@Service
public class MultipartUploadService {

    @Autowired
    private MinioClient minioClient;

    @Value("${minio.default-bucket}")
    private String bucket;

    /**
     * 初始化分片上传
     */
    public String initiateMultipartUpload(String objectName) {
        try {
            CreateMultipartUploadResponse response = minioClient.createMultipartUpload(
                CreateMultipartUploadArgs.builder()
                    .bucket(bucket)
                    .object(objectName)
                    .build()
            );
            return response.uploadId();
        } catch (Exception e) {
            throw new RuntimeException("初始化分片上传失败", e);
        }
    }

    /**
     * 上传单个分片（返回 ETag）
     */
    public PartUploadResponse uploadPart(String objectName,
                                          String uploadId,
                                          int partNumber,
                                          InputStream data,
                                          long partSize) {
        try {
            return minioClient.uploadPart(
                UploadPartArgs.builder()
                    .bucket(bucket)
                    .object(objectName)
                    .uploadId(uploadId)
                    .partNumber(partNumber)
                    .data(data)
                    .partSize(partSize)
                    .build()
            );
        } catch (Exception e) {
            throw new RuntimeException("分片上传失败", e);
        }
    }

    /**
     * 完成分片上传
     */
    public void completeMultipartUpload(String objectName,
                                         String uploadId,
                                         List<Part> parts) {
        try {
            minioClient.completeMultipartUpload(
                CompleteMultipartUploadArgs.builder()
                    .bucket(bucket)
                    .object(objectName)
                    .uploadId(uploadId)
                    .parts(parts)
                    .build()
            );
        } catch (Exception e) {
            throw new RuntimeException("完成分片上传失败", e);
        }
    }

    /**
     * 取消分片上传（清理残留数据）
     */
    public void abortMultipartUpload(String objectName, String uploadId) {
        try {
            minioClient.abortMultipartUpload(
                AbortMultipartUploadArgs.builder()
                    .bucket(bucket)
                    .object(objectName)
                    .uploadId(uploadId)
                    .build()
            );
        } catch (Exception e) {
            log.error("取消分片上传失败", e);
        }
    }

    /**
     * 列出未完成的分片上传（用于恢复或清理）
     */
    public List<Upload> listIncompleteUploads(String prefix) {
        try {
            Iterable<Result<Upload>> results = minioClient.listIncompleteUploads(
                ListIncompleteUploadsArgs.builder()
                    .bucket(bucket)
                    .prefix(prefix)
                    .build()
            );
            List<Upload> uploads = new ArrayList<>();
            for (Result<Upload> result : results) {
                uploads.add(result.get());
            }
            return uploads;
        } catch (Exception e) {
            throw new RuntimeException("列出未完成上传失败", e);
        }
    }
}
```

## 对象管理

```java
/**
 * 列出对象（分页）
 */
public List<Item> listObjects(String prefix, int maxKeys) {
    try {
        Iterable<Result<Item>> results = minioClient.listObjects(
            ListObjectsArgs.builder()
                .bucket(bucket)
                .prefix(prefix)
                .maxKeys(maxKeys)
                .build()
        );
        List<Item> items = new ArrayList<>();
        for (Result<Item> result : results) {
            items.add(result.get());
        }
        return items;
    } catch (Exception e) {
        throw new RuntimeException("列出对象失败", e);
    }
}

/**
 * 删除对象
 */
public void deleteObject(String objectName) {
    try {
        minioClient.removeObject(
            RemoveObjectArgs.builder()
                .bucket(bucket)
                .object(objectName)
                .build()
        );
    } catch (Exception e) {
        throw new RuntimeException("删除对象失败", e);
    }
}

/**
 * 批量删除对象
 */
public void deleteObjects(List<String> objectNames) {
    try {
        Iterable<Result<DeleteError>> results = minioClient.removeObjects(
            RemoveObjectsArgs.builder()
                .bucket(bucket)
                .objects(objectNames)
                .bypassGovernanceMode(true)
                .build()
        );
        for (Result<DeleteError> result : results) {
            DeleteError error = result.get();
            log.error("删除失败: object={}, code={}, message={}",
                error.objectName(), error.code(), error.message());
        }
    } catch (Exception e) {
        throw new RuntimeException("批量删除失败", e);
    }
}

/**
 * 复制对象（同 bucket 或跨 bucket）
 */
public void copyObject(String sourceBucket, String sourceObject,
                        String destBucket, String destObject) {
    try {
        CopySource source = CopySource.builder()
            .bucket(sourceBucket)
            .object(sourceObject)
            .build();

        minioClient.copyObject(
            CopyObjectArgs.builder()
                .bucket(destBucket)
                .object(destObject)
                .source(source)
                .build()
        );
    } catch (Exception e) {
        throw new RuntimeException("复制对象失败", e);
    }
}

/**
 * 获取对象元信息
 */
public ObjectStat statObject(String objectName) {
    try {
        return minioClient.statObject(
            StatObjectArgs.builder()
                .bucket(bucket)
                .object(objectName)
                .build()
        );
    } catch (Exception e) {
        throw new RuntimeException("获取对象信息失败", e);
    }
}
```

## Bucket 策略管理

```java
/**
 * 设置公开读策略（前端直接预览图片等）
 */
public void setPublicReadPolicy(String bucketName) {
    String policy = """
        {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Effect": "Allow",
              "Principal": {"AWS": "*"},
              "Action": ["s3:GetObject"],
              "Resource": ["arn:aws:s3:::%s/*"]
            }
          ]
        }
        """.formatted(bucketName);

    try {
        minioClient.setBucketPolicy(
            SetBucketPolicyArgs.builder()
                .bucket(bucketName)
                .config(policy)
                .build()
        );
    } catch (Exception e) {
        throw new RuntimeException("设置 Bucket 策略失败", e);
    }
}

/**
 * 设置带 Referer 防盗链的访问策略
 */
public void setRefererRestrictedPolicy(String bucketName, List<String> allowedDomains) {
    String domainsStr = allowedDomains.stream()
        .map(d -> "\"https://" + d + "/*\",\"http://" + d + "/*\"")
        .collect(Collectors.joining(","));

    String policy = """
        {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Effect": "Allow",
              "Principal": {"AWS": "*"},
              "Action": ["s3:GetObject"],
              "Resource": ["arn:aws:s3:::%s/*"],
              "Condition": {
                "StringLike": {"aws:Referer": [%s]}
              }
            }
          ]
        }
        """.formatted(bucketName, domainsStr);

    try {
        minioClient.setBucketPolicy(
            SetBucketPolicyArgs.builder()
                .bucket(bucketName)
                .config(policy)
                .build()
        );
    } catch (Exception e) {
        throw new RuntimeException("设置防盗链策略失败", e);
    }
}
```

## application.yml 配置

```yaml
minio:
  endpoint: http://127.0.0.1:9000
  access-key: minioadmin
  secret-key: minioadmin
  default-bucket: my-bucket
  # 预签名 URL 默认过期时间（秒）
  presigned-url-expiry:
    upload: 300       # 5分钟
    download: 3600    # 1小时
    preview: 86400    # 24小时
```

## 注意事项

1. **预签名 URL 时效性**：生产环境建议有效期 5-10 分钟（上传），过长会增加安全风险
2. **分片大小**：MinIO 最小分片 5MB，最大 5GB。推荐大文件分片大小 10-50MB
3. **未完成分片清理**：分片上传失败后，残留分片会占用存储空间。定期调用 `listIncompleteUploads` 清理超时未完成的 uploadId
4. **端点可达性**：预签名 URL 中的 endpoint 必须是客户端可访问的地址。内外网分离部署时，应用内用内网 endpoint，生成的 URL 用公网 endpoint
5. **MinIO Client 版本兼容性**：8.5.x 支持 S3 Select 等功能。升级大版本时注意 API 变更（如 8.4.x → 8.5.x 的 Builder API 有变化）
6. **对象名规范**：建议使用 `{业务}/{日期}/{UUID}.{ext}` 结构，避免单目录文件过多（MinIO 虽然是扁平结构，但 `mc ls` 和 UI 浏览时会卡）
7. **断点续传**：分片上传的 uploadId 有有效期（MinIO 默认 7 天），过期后需重新发起分片上传
