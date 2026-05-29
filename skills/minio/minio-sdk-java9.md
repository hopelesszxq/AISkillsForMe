---
name: minio-sdk-java9
description: MinIO Java SDK 9.0 重大重构：API 变更、S3 协议对齐、迁移指南
tags: [minio, java, sdk, migration, s3-compatible]
---

## 概述

MinIO Java SDK 9.0（2026-03）是一次**重大重构**，核心变化：

- 完全重构代码架构，与 S3 规范对齐
- API 命名和参数大量变更（不再兼容 8.x）
- 增加对 S3 规范中缺失字段的支持
- 9.0.1（2026-05）修复 versionId 预签名 URL 等问题

> 当前最新版本：**9.0.1** | Maven：`io.minio:minio:9.0.1`

## 依赖配置

```xml
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>9.0.1</version>
</dependency>
```

```groovy
implementation 'io.minio:minio:9.0.1'
```

## 9.0 主要变更

### 1. API 重构

| 8.x API | 9.0 API | 说明 |
|---------|---------|------|
| `MinioClient.builder().endpoint(url).credentials(key, secret).build()` | 相同但内部重构 | 兼容 |
| `putObject(PutObjectArgs)` | 参数类变更 | 需重新编译 |
| `getObject(GetObjectArgs)` | 参数类变更 | 需重新编译 |
| `getPresignedObjectUrl(GetPresignedObjectUrlArgs)` | 修复 versionId null 检查 | 9.0.1 修复 |
| 部分 `MinioException` 子类 | 移除 checked Exception | 更少的 try-catch |

### 2. 最大分片大小变更

```java
// 8.x: 最大分片上传总大小为 5 TiB
// 9.0.1: 更新为 48.828125 TiB（与 S3 规范对齐）
```

### 3. 异常处理简化（9.0.1+）

```java
// 8.x: 需要捕获 MinioException 等多层异常
try {
    minioClient.uploadObject(args);
} catch (MinioException | IOException | InvalidKeyException | NoSuchAlgorithmException e) {
    // 多个异常类型
}

// 9.0.1+: BaseS3Client 移除了不必要的 checked Exception
// 但仍需捕获 MinioException（运行时异常？需验证具体版本行为）
// 建议统一使用 Exception 或 MinioException
```

### 4. 重试支持（9.0.1+）

```java
// 9.0.1 增加了 HTTP 执行失败的重试支持
// 配置方式（通过 MinioClient.builder() 的额外配置）
MinioClient minioClient =
    MinioClient.builder()
        .endpoint("https://play.min.io")
        .credentials("accessKey", "secretKey")
        .build();
// 重试行为由底层 HTTP 客户端自动处理
```

## 基本使用（9.0 风格）

```java
import io.minio.*;

MinioClient minioClient = MinioClient.builder()
    .endpoint("https://play.min.io")
    .credentials("Q3AM3UQ867SPQQA43P2F", "zuf+tfteSlswRu7BJ86wekitnifILbZam1KYY3TG")
    .build();

// 检查桶是否存在
boolean found = minioClient.bucketExists(
    BucketExistsArgs.builder().bucket("my-bucket").build()
);

// 创建桶
if (!found) {
    minioClient.makeBucket(
        MakeBucketArgs.builder().bucket("my-bucket").build()
    );
}

// 上传文件
minioClient.uploadObject(
    UploadObjectArgs.builder()
        .bucket("my-bucket")
        .object("photo.jpg")
        .filename("/path/to/photo.jpg")
        .build()
);
```

## 预签名 URL（9.0.x 注意事项）

```java
// 9.0.1 修复了 versionId=null 时预签名 URL 生成失败的问题
// 升级到 9.0.1 即可修复
String url = minioClient.getPresignedObjectUrl(
    GetPresignedObjectUrlArgs.builder()
        .method(Method.GET)
        .bucket("my-bucket")
        .object("photo.jpg")
        .versionId("")       // 可以传空字符串
        .expiry(24, TimeUnit.HOURS)
        .build()
);
```

## 从 8.x 迁移到 9.x

### 步骤 1：升级依赖

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>9.0.1</version>
</dependency>
```

### 步骤 2：编译检查

```bash
mvn compile
```

由于参数类重构，编译时会发现不兼容的 API 调用。主要需要修改的地方：

- `PutObjectArgs` / `GetObjectArgs` / `RemoveObjectArgs` 等参数类
- 异常处理代码（移除不可达的 catch 块）

### 步骤 3：验证功能

```bash
# 运行集成测试，重点验证：
# 1. 预签名 URL 生成
# 2. 分片上传
# 3. 版本管理操作
# 4. 异常处理路径
mvn test
```

### 常见迁移问题

| 问题 | 解决 |
|------|------|
| `removeObjects()` 删除失败 | 9.0.1 修复了删除清理中的瞬时失败重试 |
| SSL 证书目录 | 9.0.1 支持 `SSL_CERT_DIR` 环境变量中的多个目录 |
| 方法签名不匹配 | 检查 Args Builder 的链式调用顺序和参数名 |
| 检查异常过多 | 8.x 中的部分 checked Exception 在 9.x 中已移除 |

## 注意事项

1. **8.x → 9.0 不兼容**：直接升级需要重新编译和测试，没有二进制兼容
2. **9.0.0 → 9.0.1 推荐升级**：9.0.1 修复了多个关键 Bug（预签名 URL、重试、删除）
3. **SSL_CERT_DIR 多目录**：9.0.1 支持用冒号分隔多个证书目录路径
4. **重试行为**：9.0.1 新增 HTTP 执行失败重试，需评估对超时场景的影响
5. **仍在 8.x 的建议**：如果未遇到 Bug，可继续使用 8.6.0，但推荐规划迁移到 9.x
