---
name: onlyoffice-editor-callback
description: OnlyOffice 在线文档编辑集成——编辑器初始化、Callback 回调、强制保存与权限控制最佳实践
tags: [onlyoffice, editor, callback, integration, spring-boot, jwt]
---

## 概述

OnlyOffice Document Server 提供完整的在线文档编辑能力。核心集成涉及前端编辑器初始化、服务端回调处理和 JWT 鉴权。本文聚焦在线编辑工作流中的关键环节。

## 一、架构总览

```
浏览器 (DocsAPI)  →  OnlyOffice Document Server
                      ↕ (callback)
Spring Boot 服务端（存储、权限、回调处理）
```

- **前端**：使用 DocsAPI 初始化编辑器，传入文档 URL、配置和回调端点
- **Document Server**：渲染编辑器，用户编辑后通过回调通知服务端
- **服务端**：接收回调（保存、强制保存、错误等），处理文档存储

## 二、配置类

```yaml
# application.yml
onlyoffice:
  ds:
    url: http://doc-server.example.com  # Document Server 地址
    jwt-secret: your-secret-key-123456   # JWT 密钥（必须与 DS 配置一致）
    jwt-expire: 300                      # Token 有效期（秒）
  storage:
    path: /data/onlyoffice/documents     # 文档本地存储路径
```

```java
@ConfigurationProperties(prefix = "onlyoffice.ds")
public record OnlyOfficeProperties(
    String url,
    @DefaultValue("secret") String jwtSecret,
    @DefaultValue("300") int jwtExpire
) {}

@ConfigurationProperties(prefix = "onlyoffice.storage")
public record StorageProperties(
    String path
) {}
```

## 三、前端编辑器初始化（Thymeleaf/Vue）

```javascript
const docEditor = new DocsAPI.DocEditor("placeholderId", {
    document: {
        fileType: "docx",
        url: "https://your-server/api/documents/123/content?token=xxx",
        title: "合同文档-v1.docx",
        key: "doc-key-123",                  // 唯一标识，内容变更必须更新
        permissions: {
            edit: true,                       // 是否可编辑
            download: false,                  // 禁止下载
            print: false,                     // 禁止打印
            copy: true,
            comment: true,
            review: true,                     // 允许审阅模式
            changeHistory: true               // 显示版本历史
        }
    },
    documentType: "word",   // "word" | "cell" | "slide" | "pdf"
    editorConfig: {
        callbackUrl: "https://your-server/api/callback",  // 回调端点
        mode: "edit",        // "edit" | "view" | "comment" | "review"
        lang: "zh-CN",
        user: {
            id: "user-001",
            name: "张三"
        },
        customization: {
            hideRightMenu: false,
            hideRulers: false,
            hideTitle: true,
            autosaveTimeout: 5000,            // 自动保存间隔（毫秒）
            forcesave: true,                  // 允许强制保存
            zoom: 100
        }
    },
    events: {
        onDocumentReady: function() {
            console.log("文档已就绪");
        },
        onDocumentStateChange: function(info) {
            // info = { type: "document.change", data: true/false }
            if (info.data) {
                console.log("文档有未保存的更改");
            }
        },
        onError: function(error) {
            console.error("编辑器错误:", error.code, error.description);
        },
        onRequestSaveAs: function() {
            // 用户点击「另存为」
            docEditor.downloadAs("pdf");
        },
        onRequestHistory: function() {
            // 用户查看版本历史
            fetchHistory(docEditor);
        },
        onRequestHistoryClose: function() {
            docEditor.refreshHistory({ serverVersion: null, history: [] });
        }
    }
});
```

## 四、Callback 回调处理（核心）

OnlyOffice Document Server 通过 `editorConfig.callbackUrl` 回调服务端，需要正确处理各类状态：

### 4.1 回调请求格式

```json
POST /api/callback
Content-Type: application/json

{
  "actions": [{"type": 1, "userid": "user-001"}],
  "changesurl": "https://doc-server/changes/...",
  "history": {"serverVersion": "0.0.1", "changes": [...]},
  "key": "doc-key-123",
  "status": 1,
  "url": "https://doc-server/download/...",
  "users": ["user-001"]
}
```

| status | 含义 | 处理方式 |
|--------|------|---------|
| 0 | 无编辑状态 | 忽略 |
| 1 | 已保存 | 从 `url` 下载更新后的文档并存储 |
| 2 | 强制保存（forcesave） | 必须立即保存，即使有冲突 |
| 3 | 文档编辑错误 | 记录日志，通知用户 |
| 4 | 文档已关闭 | 可选清理会话 |
| 6 | 正在编辑 | 忽略 |
| 7 | 强制保存请求 | 类似 status=2 |

### 4.2 Spring Boot 回调端点

```java
@Slf4j
@RestController
@RequiredArgsConstructor
public class DocumentCallbackController {

    private final DocumentService documentService;
    private final JwtService jwtService;

    @PostMapping("/api/callback")
    public ResponseEntity<Map<String, Object>> handleCallback(
            @RequestBody CallbackRequest request,
            @RequestHeader(value = "Authorization", required = false) String auth) {

        // 1. JWT 验证（如果 DS 开启了 JWT）
        if (auth != null && auth.startsWith("Bearer ")) {
            String token = auth.substring(7);
            if (!jwtService.validateToken(token)) {
                return ResponseEntity.status(403).body(Map.of("error", 1, "message", "Invalid JWT"));
            }
        }

        int status = request.getStatus();
        String key = request.getKey();
        String downloadUrl = request.getUrl();
        List<Map<String, Object>> actions = request.getActions();

        log.info("Callback received: status={}, key={}, actions={}", status, key, actions);

        try {
            switch (status) {
                case 1 -> { // 文档已保存
                    if (downloadUrl != null) {
                        documentService.saveDocument(key, downloadUrl);
                    }
                    return ResponseEntity.ok(Map.of("error", 0));
                }
                case 2, 7 -> { // 强制保存
                    if (downloadUrl != null) {
                        documentService.saveDocument(key, downloadUrl);
                        // 强制保存后需要返回新版本的 key
                        String newKey = documentService.refreshKey(key);
                        return ResponseEntity.ok(Map.of("error", 0, "key", newKey));
                    }
                    return ResponseEntity.ok(Map.of("error", 0));
                }
                case 3 -> { // 编辑错误
                    log.error("Document editing error: key={}, users={}",
                        key, request.getUsers());
                    return ResponseEntity.ok(Map.of("error", 0));
                }
                case 4 -> { // 文档关闭
                    documentService.closeSession(key);
                    return ResponseEntity.ok(Map.of("error", 0));
                }
                default -> {
                    return ResponseEntity.ok(Map.of("error", 0));
                }
            }
        } catch (Exception e) {
            log.error("Callback processing failed", e);
            return ResponseEntity.ok(Map.of("error", 1, "message", e.getMessage()));
        }
    }
}
```

### 4.3 文档保存服务

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class DocumentService {

    private final OnlyOfficeProperties props;
    private final RestTemplate restTemplate;

    /**
     * 从 Document Server 下载文档保存到本地/MinIO
     */
    public void saveDocument(String key, String downloadUrl) {
        try {
            // 1. 下载文档内容（带 JWT 鉴权）
            HttpHeaders headers = new HttpHeaders();
            headers.setBearerAuth(generateJwtToken());
            ResponseEntity<byte[]> response = restTemplate.exchange(
                downloadUrl,
                HttpMethod.GET,
                new HttpEntity<>(headers),
                byte[].class
            );

            byte[] content = response.getBody();
            if (content == null || content.length == 0) {
                log.warn("Empty document content for key: {}", key);
                return;
            }

            // 2. 存储到本地 / MinIO / 数据库
            String docId = resolveDocId(key);
            saveToStorage(docId, content);

            // 3. 记录版本历史
            saveDocumentVersion(docId, content, key);

            log.info("Document saved: key={}, size={}", key, content.length);

        } catch (Exception e) {
            log.error("Failed to save document: key={}", key, e);
            throw new DocumentSaveException("保存文档失败", e);
        }
    }

    private String generateJwtToken() {
        return Jwts.builder()
            .claim("payload", Map.of(
                "url", "download",
                "key", "callback"
            ))
            .issuedAt(new Date())
            .expiration(Date.from(Instant.now().plusSeconds(60)))
            .signWith(SignatureAlgorithm.HS256, props.jwtSecret())
            .compact();
    }
}
```

## 五、文档内容访问接口

```java
@RestController
@RequiredArgsConstructor
public class DocumentContentController {

    private final DocumentService documentService;
    private final JwtService jwtService;

    /**
     * 提供文档内容给 Document Server 加载
     */
    @GetMapping("/api/documents/{id}/content")
    public ResponseEntity<byte[]> getDocumentContent(
            @PathVariable String id,
            @RequestParam String token) {

        if (!jwtService.validateDocumentToken(token, id)) {
            return ResponseEntity.status(403).build();
        }

        byte[] content = documentService.loadDocument(id);
        if (content == null) {
            return ResponseEntity.notFound().build();
        }

        return ResponseEntity.ok()
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .body(content);
    }

    /**
     * 生成文档加载令牌（JWT Token 中包含文档 URL）
     */
    @GetMapping("/api/documents/{id}/token")
    public ResponseEntity<Map<String, String>> getDocumentToken(
            @PathVariable String id) {

        String downloadUrl = "https://your-server/api/documents/" + id + "/content";
        String token = jwtService.createDocumentToken(downloadUrl, id);

        // 同时返回编辑器初始化所需的信息
        return ResponseEntity.ok(Map.of(
            "token", token,
            "url", downloadUrl,
            "key", documentService.getDocumentKey(id),
            "title", documentService.getDocumentTitle(id),
            "fileType", documentService.getDocumentFileType(id)
        ));
    }
}
```

## 六、JWT 鉴权组件

```java
@Component
public class JwtService {

    @Value("${onlyoffice.ds.jwt-secret}")
    private String secret;

    @Value("${onlyoffice.ds.jwt-expire:300}")
    private int expireSeconds;

    /**
     * 生成用于 Document Server 通信的 Token
     */
    public String generateToken(Map<String, Object> payload) {
        return Jwts.builder()
            .claims(payload)
            .issuedAt(new Date())
            .expiration(Date.from(Instant.now().plusSeconds(expireSeconds)))
            .signWith(SignatureAlgorithm.HS256, secret)
            .compact();
    }

    /**
     * 验证 Token 并返回 payload
     */
    public Claims validateToken(String token) {
        try {
            return Jwts.parser()
                .setSigningKey(secret)
                .build()
                .parseClaimsJws(token)
                .getBody();
        } catch (JwtException e) {
            log.warn("Invalid JWT: {}", e.getMessage());
            return null;
        }
    }

    public boolean validateDocumentToken(String token, String documentId) {
        Claims claims = validateToken(token);
        return claims != null && documentId.equals(claims.get("documentId"));
    }
}
```

## 七、版本历史管理

```java
@RestController
@RequiredArgsConstructor
public class DocumentHistoryController {

    private final DocumentVersionService versionService;

    /**
     * 查看文档版本历史
     */
    @GetMapping("/api/documents/{id}/history")
    public ResponseEntity<Map<String, Object>> getHistory(@PathVariable String id) {
        List<VersionInfo> versions = versionService.getVersions(id);
        return ResponseEntity.ok(Map.of(
            "currentVersion", versions.size(),
            "history", versions.stream().map(v -> Map.of(
                "version", v.version(),
                "key", v.key(),
                "created", v.createdAt().toString(),
                "user", Map.of("id", v.userId(), "name", v.userName())
            )).toList()
        ));
    }

    /**
     * 查看指定版本的文档
     */
    @GetMapping("/api/documents/{id}/history/{version}/content")
    public ResponseEntity<byte[]> getHistoryContent(
            @PathVariable String id,
            @PathVariable int version) {
        byte[] content = versionService.loadVersion(id, version);
        if (content == null) {
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.ok(content);
    }
}
```

## 八、注意事项

- **JWT 必须一致**：OnlyOffice Document Server 的 `services.CoAuthoring.token.secret` 配置必须与 Spring Boot 服务的 `jwtSecret` 完全一致
- **key 必须更新**：每次文档内容变更后，初始化时的 `document.key` 必须生成新值，否则 DS 会返回旧缓存
- **HTTPS 要求**：Document Server 与浏览器之间、Document Server 与服务端之间必须使用 HTTPS（除非在同一内网）
- **回调超时**：Document Server 默认等待回调响应 30 秒，如果服务端处理慢需要缩短超时或异步处理
- **forcesave 与 conflicts**：status=2 的 forcesave 即使文档有冲突也必须保存，通常以 "版本冲突" 方式保留两份
- **大文件分片**：Document Server 会通过多个分片上传方式回调大文件，回调处理必须支持流式下载
- **只读关闭 callback**：如果文档为 view-only 模式，callbackUrl 不会收到 status=4 的关闭事件
- **用户头像**：`editorConfig.user.image` 可选，但 URL 必须可被 Document Server 直接访问
