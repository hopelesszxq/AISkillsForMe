---
name: minio-encryption
description: MinIO 服务端加密 S3-SSE 完整方案：SSE-S3、SSE-KMS、SSE-C 配置与实战
tags: [minio, encryption, sse, kms, security]
---

## 加密方案对比

| 方案 | 密钥管理 | 客户端改动 | 性能 | 适用场景 |
|------|----------|-----------|------|----------|
| **SSE-S3** | MinIO 自动管理 | 无 | 高 | 默认加密、合规需求 |
| **SSE-KMS** | 外部 KMS（KES） | 无 | 中 | 企业审计、密钥轮换 |
| **SSE-C** | 客户端提供密钥 | 需传密钥头 | 高 | 自有密钥、细粒度控制 |

> MinIO 3.x 起已弃用 SSE-S3 风格的自动服务端加密，推荐通过 **KES + KMS** 或 **MinIO Operator** 配置默认加密。

## 一、SSE-KMS（推荐生产用）

### 1. 部署 KES（KMS Engine）

```bash
# docker-compose 部署 KES
version: '3.8'
services:
  kes:
    image: minio/kes:latest
    ports:
      - "7373:7373"
    environment:
      - KES_SERVER=https://0.0.0.0:7373
      - KES_IDENTITY=minio-admin
    volumes:
      - ./kes-config.yaml:/etc/kes/config.yaml
      - ./certs:/etc/kes/certs
    command: server --config /etc/kes/config.yaml --auth off
```

### 2. MinIO 关联 KES

```bash
# docker-compose 配置
environment:
  - MINIO_KMS_KES_ENDPOINT=https://kes:7373
  - MINIO_KMS_KES_KEY_FILE=/etc/minio/certs/kes-key.pem
  - MINIO_KMS_KES_CERT_FILE=/etc/minio/certs/kes-cert.pem
  - MINIO_KMS_KES_KEY_NAME=minio-default-key
  - MINIO_ROOT=off  # 关闭自动根加密
```

### 3. 启用自动桶加密

```bash
# 通过 mc 设置默认加密
mc encrypt set sse-kms myminio/my-bucket

# 验证
mc encrypt info myminio/my-bucket
```

## 二、SSE-C（客户端管理密钥）

适合每个文件使用不同密钥的场景。

### Java SDK 示例

```java
import io.minio.*;
import io.minio.http.*;

// 32 字节 AES-256 密钥
SecretKeySpec secretKey = new SecretKeySpec(
    "0123456789abcdef0123456789abcdef".getBytes(StandardCharsets.UTF_8),
    "AES"
);

// 上传时指定 SSE-C
PutObjectArgs putArgs = PutObjectArgs.builder()
    .bucket("secure-bucket")
    .object("confidential.docx")
    .stream(inputStream, size, -1)
    .ssec(new ServerSideEncryptionWithCustomerKey(secretKey))
    .build();
minioClient.putObject(putArgs);

// 下载时必须提供相同密钥
GetObjectArgs getArgs = GetObjectArgs.builder()
    .bucket("secure-bucket")
    .object("confidential.docx")
    .ssec(new ServerSideEncryptionWithCustomerKey(secretKey))
    .build();
InputStream stream = minioClient.getObject(getArgs);
```

### 注意事项

- **密钥丢失即数据丢失**：SSE-C 密钥由客户端保管，MinIO 不存储
- **每次请求都需要传密钥**：GetObject、DeleteObject、StatObject 都需带 ssec
- **预签名 URL + SSE-C 不支持**：预签名 URL 无法携带加密标头，需用自定义认证
- **建议密钥推导**：用业务主密钥 + 对象 ID 派生子密钥，避免硬编码

```java
// 密钥推导模式
public static SecretKeySpec deriveKey(String masterKey, String objectId) {
    Mac mac = Mac.getInstance("HmacSHA256");
    SecretKeySpec master = new SecretKeySpec(
        masterKey.getBytes(StandardCharsets.UTF_8), "HmacSHA256");
    mac.init(master);
    byte[] derived = mac.doFinal(objectId.getBytes(StandardCharsets.UTF_8));
    return new SecretKeySpec(derived, "AES");
}
```

## 三、传输加密（TLS/HTTPS）

生产环境必须启用 TLS：

```bash
# mc 配置 TLS
mc alias set myminio https://minio.example.com:9000 \
  --api "s3v4" \
  --path "auto"

# MinIO 配置证书
# 将 cert.pem, key.pem 放入 ${MINIO_ROOT_CERTS}/CAs/
# 或通过环境变量
MINIO_ROOT_CERTS=/etc/minio/certs
```

## 四、加密 + 版本控制 + 对象锁定组合

```java
// 创建带默认加密的版本控制桶
CreateBucketArgs bucketArgs = CreateBucketArgs.builder()
    .bucket("compliant-bucket")
    .objectLock(true)  // 启用对象锁定（WORM）
    .build();
minioClient.createBucket(bucketArgs);

// 启用版本控制
SetVersioningArgs versionArgs = SetVersioningArgs.builder()
    .bucket("compliant-bucket")
    .config(VersioningConfiguration.ENABLED)
    .build();
minioClient.setVersioning(versionArgs);

// 启用默认加密（SSE-KMS）
String policy = "{\"Rules\":[{\"ApplyServerSideEncryptionByDefault\":{\"SSEAlgorithm\":\"AES256\"}}]}";
SetBucketEncryptionArgs encArgs = SetBucketEncryptionArgs.builder()
    .bucket("compliant-bucket")
    .encryptionConfig(EncryptionConfig.fromXml(policy))
    .build();
minioClient.setBucketEncryption(encArgs);
```

## 注意事项

1. **性能影响**：SSE-KMS 每次操作需调用 KES，QPS 高的场景考虑本地缓存策略
2. **KES 高可用**：KES 本身是无状态的，可前置负载均衡。推荐部署 2+ 节点
3. **密钥轮换**：SSE-KMS 支持密钥版本管理，旧数据仍用旧密钥解密（不解密重加密）
4. **审计日志**：KES 支持所有操作审计，对接 ELK/Splunk
5. **成本考量**：SSE-S3/AES256 零额外成本，SSE-KMS 需额外部署 KES 资源
