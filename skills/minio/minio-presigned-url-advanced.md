---
name: minio-presigned-url-advanced
description: MinIO 预签名 URL 高级实战：文件上传下载、过期策略、存储桶策略绑定、CDN 融合与安全防护
tags: [minio, s3, presigned-url, java, security, upload, download]
---

## 预签名 URL 概述

预签名 URL（Presigned URL）为 MinIO/S3 对象提供**临时授权访问**，无需暴露 AccessKey/SecretKey。适合：

- 客户端直传文件到 MinIO（避免服务器中转）
- 临时分享文件下载链接
- 受限时间窗口内的资源访问

## 1. Java SDK 生成预签名 URL

### Maven 依赖

```xml
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.15</version>
</dependency>
```

### 上传预签名 URL

```java
import io.minio.*;
import io.minio.http.Method;
import java.util.concurrent.TimeUnit;

@Service
public class PresignedUrlService {
    
    private final MinioClient minioClient;
    
    public PresignedUrlService(MinioClient minioClient) {
        this.minioClient = minioClient;
    }
    
    /**
     * 生成文件上传的预签名 URL
     * @param bucket 存储桶
     * @param objectName 对象名称
     * @param expiryMinutes 过期时间（分钟）
     * @return 上传 URL
     */
    public String generateUploadUrl(String bucket, String objectName, int expiryMinutes) {
        try {
            return minioClient.getPresignedObjectUrl(
                GetPresignedObjectUrlArgs.builder()
                    .method(Method.PUT)           // PUT 表示上传
                    .bucket(bucket)
                    .object(objectName)
                    .expiry(expiryMinutes, TimeUnit.MINUTES)
                    .build()
            );
        } catch (Exception e) {
            throw new RuntimeException("Failed to generate upload URL", e);
        }
    }
}
```

### 下载预签名 URL

```java
/**
 * 生成文件下载的预签名 URL
 * @param bucket 存储桶
 * @param objectName 对象名称
 * @param expiryMinutes 过期时间
 * @param fileName 下载时显示的文件名（可选）
 * @return 下载 URL
 */
public String generateDownloadUrl(String bucket, String objectName, 
                                   int expiryMinutes, String fileName) {
    try {
        Map<String, String> queryParams = new HashMap<>();
        if (fileName != null && !fileName.isEmpty()) {
            // response-content-disposition 控制下载文件名
            queryParams.put("response-content-disposition", 
                "attachment; filename=\"" + fileName + "\"");
        }
        
        return minioClient.getPresignedObjectUrl(
            GetPresignedObjectUrlArgs.builder()
                .method(Method.GET)
                .bucket(bucket)
                .object(objectName)
                .expiry(expiryMinutes, TimeUnit.MINUTES)
                .extraQueryParams(queryParams)
                .build()
        );
    } catch (Exception e) {
        throw new RuntimeException("Failed to generate download URL", e);
    }
}
```

## 2. 客户端使用预签名 URL

### 前端上传（浏览器直传）

```javascript
// 1. 从后端获取预签名 URL
const response = await fetch('/api/files/upload-url', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ fileName: 'photo.jpg', contentType: 'image/jpeg' })
});
const { uploadUrl, objectName } = await response.json();

// 2. 直传到 MinIO
const file = document.getElementById('fileInput').files[0];
await fetch(uploadUrl, {
    method: 'PUT',
    body: file,
    headers: { 'Content-Type': file.type }
});

console.log('Upload complete:', objectName);
```

### 分片上传（大文件）

```javascript
async function uploadLargeFile(file, presignedUrls) {
    const CHUNK_SIZE = 5 * 1024 * 1024; // 5MB
    const totalChunks = Math.ceil(file.size / CHUNK_SIZE);
    const uploadId = await initiateMultipartUpload(file.name);
    
    const uploadedParts = [];
    for (let i = 0; i < totalChunks; i++) {
        const start = i * CHUNK_SIZE;
        const end = Math.min(start + CHUNK_SIZE, file.size);
        const chunk = file.slice(start, end);
        
        // 获取每个分片的预签名 URL
        const url = await getPresignedPartUrl(uploadId, i + 1);
        const response = await fetch(url, { method: 'PUT', body: chunk });
        const etag = response.headers.get('ETag');
        uploadedParts.push({ PartNumber: i + 1, ETag: etag });
    }
    
    await completeMultipartUpload(uploadId, uploadedParts);
}
```

## 3. 高级安全策略

### 限制预签名 URL 的来源

```java
// 在 MinIO 存储桶策略中限制 Referer
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::my-bucket/uploads/*",
            "Condition": {
                "StringLike": {
                    "aws:Referer": "https://yourdomain.com/*"
                }
            }
        }
    ]
}
```

### IP 白名单限制

```java
// 通过条件限制 IP
"Condition": {
    "IpAddress": {
        "aws:SourceIp": ["192.168.1.0/24", "10.0.0.0/8"]
    }
}
```

### 防重放与时效控制

```java
/**
 * 生成短期令牌 + 预签名 URL 双重验证
 */
public String generateSecureUploadUrl(String bucket, String objectName, 
                                       String userToken) {
    // 1. 验证用户令牌
    if (!tokenService.validateToken(userToken)) {
        throw new SecurityException("Invalid token");
    }
    
    // 2. 生成极短有效期的 URL（1 分钟）
    return generateUploadUrl(bucket, objectName, 1);
}
```

## 4. 预签名 URL + CDN 集成

```java
/**
 * 生成 CDN 签名 URL（Nginx + MinIO 配合）
 */
public String generateCdnSignedUrl(String objectName, int expirySeconds) {
    // 生成 MinIO 预签名 URL
    String minioUrl = generateDownloadUrl("public-bucket", objectName, 
        expirySeconds / 60 + 1, null);
    
    // 替换为 CDN 域名
    return minioUrl.replace(
        "https://minio-internal:9000",
        "https://cdn.yourdomain.com"
    );
}
```

## 5. 最佳实践

### 超时设置指南

| 场景 | 推荐过期时间 | 说明 |
|------|-------------|------|
| 浏览器直传小文件 | 5-15分钟 | 用户上传操作窗口 |
| 大文件分片上传 | 1-2小时 | 分片上传流程较长 |
| 下载分享 | 1小时-7天 | 按分享需求设定 |
| 临时查看 | 1-5分钟 | 如缩略图加载 |

### 文件名校验与防覆盖

```java
@PostMapping("/upload-url")
public Map<String, String> generateUploadUrl(@RequestBody UploadRequest request) {
    // 1. 校验文件类型
    String ext = FilenameUtils.getExtension(request.getFileName());
    if (!ALLOWED_EXTENSIONS.contains(ext.toLowerCase())) {
        throw new BadRequestException("File type not allowed: " + ext);
    }
    
    // 2. 校验文件大小
    if (request.getFileSize() > MAX_FILE_SIZE) {
        throw new BadRequestException("File too large: " + request.getFileSize());
    }
    
    // 3. 生成唯一对象名，防止覆盖
    String objectName = UUID.randomUUID() + "." + ext;
    
    // 4. 生成上传 URL
    String url = presignedUrlService.generateUploadUrl(
        config.getBucket(), objectName, 15);
    
    return Map.of("uploadUrl", url, "objectName", objectName);
}
```

## 注意事项

### 1. 安全风险
- 预签名 URL 一旦泄露，任何人在有效期内都可以使用
- **永远不要**生成过于宽泛的预签名 URL（如 `/*`）
- 建议对预签名 URL 的生成操作进行审计日志

### 2. URL 长度限制
- 预签名 URL 包含签名信息，可能很长（>2000字符）
- 某些浏览器/代理/负载均衡器有 URL 长度限制（如 8KB）
- 分片上传 URL 要通过 POST/JSON 返回而不是 URL 参数传递

### 3. MinIO 版本兼容
- 8.x SDK 的 `getPresignedObjectUrl` 签名方式与 7.x 不同
- 升级 SDK 后需重新生成 URL，旧 URL 会立即失效

### 4. 中文文件名处理
```java
// 必须对中文进行 URLEncoding
String encodedName = URLEncoder.encode(fileName, StandardCharsets.UTF_8)
    .replace("+", "%20");
```
