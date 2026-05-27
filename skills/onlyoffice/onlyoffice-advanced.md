---
name: onlyoffice-advanced
description: OnlyOffice Document Server 深度配置、回调处理与文档转换实战
tags: [onlyoffice, office, document, integration]
---

## Document Server 部署与配置

### Docker 部署

```yaml
# docker-compose.yml
version: "3.8"

services:
  onlyoffice-document-server:
    image: onlyoffice/documentserver:8.2.1
    container_name: onlyoffice-ds
    ports:
      - "8088:80"    # HTTP 访问
    environment:
      # JWT 密钥（必配！生产安全关键）
      - JWT_ENABLED=true
      - JWT_SECRET=your-super-secret-key-change-in-production
      - JWT_HEADER=Authorization

      # 日志级别
      - LOG_LEVEL=warn

      # 时区
      - TZ=Asia/Shanghai

      # Redis 配置（用于协同编辑会话缓存）
      - REDIS_SERVER_HOST=redis
      - REDIS_SERVER_PORT=6379
      - REDIS_SERVER_PASS=redispassword

      # RabbitMQ（可选，用于文档转换队列）
      - AMQP_URI=amqp://guest:guest@rabbitmq:5672

      # 存储（可选，使用 MinIO 或 S3）
      - STORAGE_TYPE=minio
      - MINIO_ACCESS_KEY=minioadmin
      - MINIO_SECRET_KEY=minioadmin
      - MINIO_BUCKET_NAME=documents
      - MINIO_ENDPOINT=http://minio:9000
      - MINIO_REGION=us-east-1
      - MINIO_SERVER_SECURE=false

    volumes:
      - onlyoffice-ds-data:/var/www/onlyoffice/Data
      - onlyoffice-ds-logs:/var/log/onlyoffice

    depends_on:
      redis:
        condition: service_started

    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/healthcheck"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass redispassword
    volumes:
      - redis-data:/data
    restart: unless-stopped

volumes:
  onlyoffice-ds-data:
  onlyoffice-ds-logs:
  redis-data:
```

### JWT 安全配置

```javascript
// OnlyOffice 前端集成时 JS 配置
const config = {
  document: {
    fileType: "docx",
    key: docKey,
    title: "文档标题.docx",
    url: `${API_BASE}/api/documents/` + docId + "/content",
    // 前端生成临时 JWT（与服务端 JWT_SECRET 一致）
    token: jwtToken
  },
  documentType: "word",
  editorConfig: {
    callbackUrl: `${API_BASE}/api/documents/` + docId + "/callback",
    lang: "zh-CN",
    user: {
      id: userId,
      name: userName
    },
    // 自定义配置
    customization: {
      forcesave: true,        // 强制保存（每次保存都触发回调）
      autosave: true,         // 启用自动保存
      chat: false,            // 禁用聊天
      comments: true,         // 允许评论
      compactHeader: false,   // 紧凑头部
      compactToolbar: false   // 紧凑工具栏
    }
  },
  events: {
    onError: (error) => console.error("OnlyOffice error:", error),
    onRequestHistory: () => getAllHistory()
  }
};
```

## 回调处理（Callback Handler）

回调是 OnlyOffice 集成的核心，用于处理文档保存、强制保存等事件。

### RestController 实现

```java
@RestController
@RequestMapping("/api/documents/{docId}")
@Slf4j
public class OnlyOfficeCallbackController {

    @Autowired
    private DocumentService documentService;

    @Autowired
    private JwtTokenProvider jwtProvider;

    @PostMapping("/callback")
    public ResponseEntity<Map<String, Object>> callback(
            @PathVariable Long docId,
            @RequestBody OnlyOfficeCallback body,
            @RequestHeader(value = "Authorization", required = false) String token) {

        // 1. JWT 验证
        if (!jwtProvider.validateToken(token)) {
            return ResponseEntity.status(401)
                .body(Map.of("error", 1, "message", "Invalid JWT token"));
        }

        log.debug("OnlyOffice callback: status={}, actions={}",
            body.getStatus(), body.getActions());

        // 2. 根据状态码处理
        switch (body.getStatus()) {
            case 0:  // 文档已不存在或读取错误
                log.error("文档读取失败: docId={}", docId);
                return ok(0);

            case 1:  // 文档已就绪，用户可以编辑
                return ok(0);

            case 2:  // 文档正在保存（保存中）
                return ok(0);

            case 3:  // 文档保存出错
                log.error("文档保存失败: docId={}, error={}", docId, body.getHistory());
                return ok(0);

            case 4:  // 文档已关闭（无强制保存时）
                log.info("文档关闭: docId={}, user={}", docId, body.getUserData());
                documentService.releaseLock(docId);
                return ok(0);

            case 6:  // 数据正在强制保存
                // forcesave=true 时，每次保存触发此回调，需更新文档
                try {
                    byte[] content = downloadDocument(body.getUrl());
                    documentService.saveDocument(docId, content);
                    log.info("强制保存成功: docId={}", docId);
                    return ok(0);
                } catch (Exception e) {
                    log.error("强制保存失败: docId={}", docId, e);
                    return ok(1);  // 告诉 OnlyOffice 需要重试
                }

            case 7:  // 文档强制保存错误
                log.error("强制保存出错: docId={}", docId);
                return ok(0);

            default:
                log.warn("未处理的回调状态: {}", body.getStatus());
                return ok(0);
        }
    }

    private ResponseEntity<Map<String, Object>> ok(int error) {
        return ResponseEntity.ok(Map.of("error", error));
    }

    /**
     * 从 OnlyOffice 下载文档（回调 URL 有时效性，需立即下载）
     */
    private byte[] downloadDocument(String url) {
        // 使用 RestTemplate 或 WebClient 下载文档
        return restTemplate.getForObject(url, byte[].class);
    }
}

@Data
public class OnlyOfficeCallback {
    private List<Action> actions;      // 用户操作记录
    private Object history;            // 变更历史
    private String key;                // 文档 key
    private Integer status;            // 状态码
    private String url;                // 文档下载 URL（仅 status=2/6 时有值）
    private String userData;           // 自定义用户数据
    private Long users;                // 在线用户数

    @Data
    public static class Action {
        private Integer type;          // 0=断开 1=连接
        private String userid;         // 用户 ID
    }
}
```

## 文档转换 API

OnlyOffice Document Server 支持文档格式转换（在线预览、PDF 导出）。

### 文档转 PDF（预览用）

```java
@Service
@Slf4j
public class DocumentConversionService {

    private static final String CONVERT_URL = "http://onlyoffice-ds:8088/ConvertService.ashx";

    /**
     * 文档转 PDF
     * @param fileUri     文档下载地址（需 OnlyOffice 可访问）
     * @param fileType    源文件类型：docx, xlsx, pptx
     * @param outputType  目标类型：pdf, docx, txt, html
     * @return 转换后的文件下载 URL
     */
    public String convert(String fileUri, String fileType, String outputType) {
        Map<String, Object> request = new HashMap<>();
        request.put("async", false);                        // 同步转换
        request.put("filetype", fileType);                  // 源格式
        request.put("outputtype", outputType);              // 目标格式
        request.put("title", "document." + fileType);
        request.put("key", UUID.randomUUID().toString());   // 唯一 key
        request.put("url", fileUri);                        // 文档下载地址
        request.put("token", generateJwtToken(request));    // JWT 签名

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);

        ResponseEntity<Map> response = restTemplate.postForEntity(
            CONVERT_URL, new HttpEntity<>(request, headers), Map.class);

        Map<String, Object> body = response.getBody();
        if (body != null && (Integer) body.get("error") == 0) {
            String resultUrl = (String) body.get("fileUrl");
            log.info("文档转换成功: {} -> {}, resultUrl={}", fileType, outputType, resultUrl);
            return resultUrl;
        }

        log.error("文档转换失败: response={}", body);
        throw new RuntimeException("文档转换失败: " + body);
    }

    private String generateJwtToken(Map<String, Object> payload) {
        return Jwts.builder()
            .setPayload(JSON.toJSONString(payload))
            .signWith(SignatureAlgorithm.HS256, secretKey.getBytes(StandardCharsets.UTF_8))
            .compact();
    }
}
```

## 注意事项

1. **回调 URL 必须公网可达**：OnlyOffice Document Server 必须能访问到你的回调 URL（公网 IP 或内网可达）
2. **文档下载 URL 时效性**：回调中的 `url` 字段有时效（通常 30s），需要立即下载保存
3. **forcesave 与正常保存的区别**：
   - forcesave=true：每次保存都触发 status=6 回调，适合需要实时保存到业务存储的场景
   - forcesave=false：仅在文档关闭时触发保存，中间版本存储在 Document Server 缓存
4. **并发编辑冲突**：多个用户编辑同一文档时，最后保存的版本覆盖中间版本。建议用 Document Server 内置的协同编辑（支持实时协同）
5. **临时存储卷清理**：`/var/www/onlyoffice/Data` 会累积临时缓存文件，建议设置 crontab 定期清理：
   ```bash
   0 3 * * * find /var/www/onlyoffice/Data -name "*.tmp" -mtime +7 -delete
   ```
6. **内存配置**：Document Server 默认 2GB 堆内存，生产环境至少 4GB，大文档场景建议 8GB+
7. **字体安装**：默认字体有限，需挂载中文字体包确保文档显示正确：
   ```bash
   # 在 Document Server 容器内
   apt-get install -y fonts-noto-cjk fonts-wqy-zenhei
   ```
8. **版本兼容性**：前端 `@onlyoffice/document-editor` SDK 版本需与 Document Server 版本一致，否则可能出现接口不兼容
9. **文档 Key 的管理**：`key` 参数用于标识文档版本，每次大版本变更需换 key（如：UUID+版本号），否则 Document Server 使用缓存
