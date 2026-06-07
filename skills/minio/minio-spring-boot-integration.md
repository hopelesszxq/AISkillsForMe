---
name: minio-spring-boot-integration
description: MinIO 与 Spring Boot 集成最佳实践：配置管理、文件服务层、异常处理与测试
tags: [minio, spring-boot, file-storage, s3, integration]
---

## 概述

Spring Boot 项目中整合 MinIO 作为文件存储后端是最常见的生产需求。本文提供从依赖配置到 Service 封装、异常处理、单元测试的完整实践。

## 1. 依赖配置

```xml
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.15</version>
</dependency>

<!-- 可选：Spring Retry 实现重试 -->
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

## 2. 配置属性类

```java
@ConfigurationProperties(prefix = "minio")
public record MinioProperties(
    String endpoint,
    String accessKey,
    String secretKey,
    String bucket,
    String region,
    @DefaultValue("false") boolean secure,
    @DefaultValue("PT1H") Duration presignedExpiry,
    @DefaultValue("10485760") long maxUploadSize  // 10MB 默认
) {}
```

```yaml
# application.yml
minio:
  endpoint: http://localhost:9000
  access-key: minioadmin
  secret-key: minioadmin
  bucket: myapp-files
  region: cn-east-1
  secure: false
  presigned-expiry: PT1H
  max-upload-size: 104857600  # 100MB
```

## 3. MinIO 客户端配置

```java
@Configuration
@EnableConfigurationProperties(MinioProperties.class)
public class MinioConfig {

    @Bean
    @ConditionalOnMissingBean
    public MinioClient minioClient(MinioProperties props) {
        return MinioClient.builder()
            .endpoint(props.endpoint())
            .credentials(props.accessKey(), props.secretKey())
            .region(props.region())
            .build();
    }

    /**
     * 应用启动时自动初始化默认存储桶
     */
    @Bean
    public CommandLineRunner initBucket(MinioClient client, MinioProperties props) {
        return args -> {
            String bucket = props.bucket();
            if (!client.bucketExists(BucketExistsArgs.builder().bucket(bucket).build())) {
                client.makeBucket(MakeBucketArgs.builder().bucket(bucket).build());
                log.info("MinIO bucket [{}] created", bucket);
            }
        };
    }
}
```

## 4. 文件服务层封装

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class FileService {

    private final MinioClient minioClient;
    private final MinioProperties props;

    // ========== 上传 ==========

    /**
     * 上传文件（InputStream）
     */
    public String upload(String objectName, InputStream in, String contentType, long size) {
        try {
            ObjectWriteResponse resp = minioClient.putObject(
                PutObjectArgs.builder()
                    .bucket(props.bucket())
                    .object(objectName)
                    .stream(in, size, -1)
                    .contentType(contentType)
                    .build()
            );
            log.info("Upload success: {} -> {}", objectName, resp.etag());
            return objectName;
        } catch (MinioException e) {
            throw new FileStorageException("Upload failed: " + objectName, e);
        } catch (IOException | NoSuchAlgorithmException | InvalidKeyException e) {
            throw new FileStorageException("IO error during upload", e);
        }
    }

    /**
     * 上传 MultipartFile
     */
    public String uploadMultipart(String objectName, MultipartFile file) {
        if (file.getSize() > props.maxUploadSize()) {
            throw new FileTooLargeException(props.maxUploadSize(), file.getSize());
        }
        try (InputStream in = file.getInputStream()) {
            return upload(objectName, in, file.getContentType(), file.getSize());
        } catch (IOException e) {
            throw new FileStorageException("Failed to read multipart file", e);
        }
    }

    // ========== 下载 ==========

    /**
     * 下载文件，返回 InputStream（调用方必须关闭）
     */
    public InputStream download(String objectName) {
        try {
            GetObjectResponse resp = minioClient.getObject(
                GetObjectArgs.builder()
                    .bucket(props.bucket())
                    .object(objectName)
                    .build()
            );
            return resp; // GetObjectResponse extends InputStream
        } catch (MinioException | IOException | NoSuchAlgorithmException | InvalidKeyException e) {
            throw new FileNotFoundException("File not found: " + objectName, e);
        }
    }

    /**
     * 下载到本地临时文件
     */
    public Path downloadToTemp(String objectName) {
        try {
            Path tempFile = Files.createTempFile("minio-", getExtension(objectName));
            minioClient.downloadObject(
                DownloadObjectArgs.builder()
                    .bucket(props.bucket())
                    .object(objectName)
                    .filename(tempFile.toString())
                    .build()
            );
            return tempFile;
        } catch (MinioException | IOException | NoSuchAlgorithmException | InvalidKeyException e) {
            throw new FileNotFoundException("Download failed: " + objectName, e);
        }
    }

    // ========== 删除 ==========

    public void delete(String objectName) {
        try {
            minioClient.removeObject(
                RemoveObjectArgs.builder()
                    .bucket(props.bucket())
                    .object(objectName)
                    .build()
            );
        } catch (MinioException | IOException | NoSuchAlgorithmException | InvalidKeyException e) {
            throw new FileStorageException("Delete failed: " + objectName, e);
        }
    }

    public void batchDelete(List<String> objectNames) {
        try {
            List<DeleteObject> objects = objectNames.stream()
                .map(DeleteObject::new)
                .toList();
            Iterable<Result<DeleteError>> results = minioClient.removeObjects(
                RemoveObjectsArgs.builder()
                    .bucket(props.bucket())
                    .objects(objects)
                    .build()
            );
            for (Result<DeleteError> result : results) {
                DeleteError err = result.get();
                log.warn("Batch delete partial failure: {} - {}", err.objectName(), err.message());
            }
        } catch (MinioException | IOException | NoSuchAlgorithmException | InvalidKeyException e) {
            throw new FileStorageException("Batch delete failed", e);
        }
    }

    // ========== 预签名 URL ==========

    /**
     * 生成上传预签名 URL（客户端直传）
     */
    public String presignedUploadUrl(String objectName, Duration expiry) {
        try {
            return minioClient.getPresignedObjectUrl(
                GetPresignedObjectUrlArgs.builder()
                    .method(Method.PUT)
                    .bucket(props.bucket())
                    .object(objectName)
                    .expiry((int) expiry.toSeconds())
                    .build()
            );
        } catch (MinioException | IOException | NoSuchAlgorithmException | InvalidKeyException e) {
            throw new FileStorageException("Generate presigned upload URL failed", e);
        }
    }

    /**
     * 生成下载预签名 URL
     */
    public String presignedDownloadUrl(String objectName) {
        return presignedDownloadUrl(objectName, props.presignedExpiry());
    }

    public String presignedDownloadUrl(String objectName, Duration expiry) {
        try {
            return minioClient.getPresignedObjectUrl(
                GetPresignedObjectUrlArgs.builder()
                    .method(Method.GET)
                    .bucket(props.bucket())
                    .object(objectName)
                    .expiry((int) expiry.toSeconds())
                    .build()
            );
        } catch (MinioException | IOException | NoSuchAlgorithmException | InvalidKeyException e) {
            throw new FileStorageException("Generate presigned download URL failed", e);
        }
    }

    // ========== 查询 ==========

    /**
     * 列出存储桶内所有对象
     */
    public List<String> listObjects() {
        return listObjects(null);
    }

    public List<String> listObjects(String prefix) {
        try {
            Iterable<Result<Item>> results = minioClient.listObjects(
                ListObjectsArgs.builder()
                    .bucket(props.bucket())
                    .prefix(prefix)
                    .recursive(true)
                    .build()
            );
            List<String> names = new ArrayList<>();
            for (Result<Item> result : results) {
                names.add(result.get().objectName());
            }
            return names;
        } catch (MinioException | IOException | NoSuchAlgorithmException | InvalidKeyException e) {
            throw new FileStorageException("List objects failed", e);
        }
    }

    private String getExtension(String filename) {
        int dot = filename.lastIndexOf('.');
        return dot == -1 ? "" : filename.substring(dot);
    }
}
```

## 5. 异常处理体系

```java
// 运行时异常基类
public class FileStorageException extends RuntimeException {
    public FileStorageException(String message, Throwable cause) {
        super(message, cause);
    }
}

// 文件过大
public class FileTooLargeException extends FileStorageException {
    private final long maxSize;
    private final long actualSize;

    public FileTooLargeException(long maxSize, long actualSize) {
        super("File too large: " + actualSize + " > " + maxSize);
        this.maxSize = maxSize;
        this.actualSize = actualSize;
    }
}

// 文件未找到
public class FileNotFoundException extends FileStorageException {
    public FileNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

```java
@RestControllerAdvice
public class FileExceptionHandler {

    @ExceptionHandler(FileTooLargeException.class)
    public ResponseEntity<Map<String, Object>> handleFileTooLarge(FileTooLargeException e) {
        return ResponseEntity.status(HttpStatus.PAYLOAD_TOO_LARGE).body(Map.of(
            "code", 413,
            "message", e.getMessage(),
            "maxSize", e.getMaxSize(),
            "actualSize", e.getActualSize()
        ));
    }

    @ExceptionHandler(FileNotFoundException.class)
    public ResponseEntity<Map<String, Object>> handleNotFound(FileNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(Map.of(
            "code", 404,
            "message", e.getMessage()
        ));
    }

    @ExceptionHandler(FileStorageException.class)
    public ResponseEntity<Map<String, Object>> handleStorageError(FileStorageException e) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(Map.of(
            "code", 500,
            "message", "File storage error: " + e.getMessage()
        ));
    }
}
```

## 6. Spring Retry 重试机制（网络抖动防护）

```java
@Slf4j
@Service
public class RetryableFileService {

    private final MinioClient minioClient;

    @Retryable(
        retryFor = { IOException.class, MinioException.class },
        maxAttempts = 3,
        backoff = @Backoff(delay = 200, multiplier = 2.0),
        notRecoverable = { ErrorResponseException.class }
    )
    public ObjectWriteResponse uploadWithRetry(PutObjectArgs args) throws Exception {
        return minioClient.putObject(args);
    }

    @Recover
    public ObjectWriteResponse recover(Throwable e, PutObjectArgs args) {
        log.error("Upload failed after 3 retries: {}", args.object(), e);
        throw new FileStorageException("Upload failed after retries", e);
    }
}
```

## 7. 测试（Testcontainers）

```java
@SpringBootTest
@Testcontainers
class FileServiceTest {

    @Container
    static GenericContainer<?> minioContainer = new GenericContainer<>("minio/minio:latest")
        .withCommand("server /data")
        .withExposedPorts(9000, 9001)
        .withEnv("MINIO_ROOT_USER", "minioadmin")
        .withEnv("MINIO_ROOT_PASSWORD", "minioadmin")
        .withEnv("MINIO_DOMAIN", "localhost");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("minio.endpoint", () ->
            "http://" + minioContainer.getHost() + ":" + minioContainer.getMappedPort(9000));
        registry.add("minio.access-key", () -> "minioadmin");
        registry.add("minio.secret-key", () -> "minioadmin");
    }

    @Autowired
    private FileService fileService;

    @Test
    void shouldUploadAndDownloadFile() {
        String content = "Hello MinIO!";
        String objectName = "test/hello.txt";

        // 上传
        fileService.upload(objectName,
            new ByteArrayInputStream(content.getBytes(StandardCharsets.UTF_8)),
            "text/plain", content.length());

        // 下载并验证
        try (InputStream in = fileService.download(objectName)) {
            String result = new String(in.readAllBytes(), StandardCharsets.UTF_8);
            assertThat(result).isEqualTo(content);
        } catch (IOException e) {
            fail("Download verification failed", e);
        }

        // 清理
        fileService.delete(objectName);
    }
}
```

## 注意事项

- **连接池**：MinIO Java SDK 默认无连接池，建议添加 Apache HttpClient 或 OkHttp 依赖以获得连接池支持
- **超时设置**：生产环境必须配置 connect/read/write timeout，避免线程长期阻塞
- **大文件上传**：超过 5MB 的文件会自动使用 multipart upload，但仍建议限制单文件大小
- **Bucket 隔离**：不同业务模块使用不同 bucket，便于权限管理和生命周期策略
- **密钥安全**：生产环境不要硬编码 accessKey/secretKey，优先使用 Kubernetes Secret、Vault 或 AWS IAM Role
- **错误分类**：`ErrorResponseException`（MinIO 服务端错误）、`InsufficientDataException`（网络中断）、`InvalidResponseException`（协议异常）需分别处理
