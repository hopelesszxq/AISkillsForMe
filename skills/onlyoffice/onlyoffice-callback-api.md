---
name: onlyoffice-callback-api
description: OnlyOffice 文档编辑回调 API 深度解析与实战
tags: [onlyoffice, callback, document, integration]
---

## 回调机制概述

OnlyOffice Document Server 通过回调（Callback）与业务后端通信，实现文档的保存、权限校验、状态同步。回调由 Document Server 主动发起，POST JSON 到配置的 `callbackUrl`。

### 回调类型

| 类型 | 说明 | 触发时机 |
|---|---|---|
| `track` | 文档状态追踪（核心） | 用户操作文档时持续触发 |
| `status` | 文档状态变更 | 打开、关闭、保存、强制保存 |
| `meta` | 文档元数据 | 打开文档时查询文件名等 |
| `forceSave` | 强制保存 | 管理员发起或定时保存 |

## Callback URL 配置

### 前端初始化

```javascript
// 初始化文档编辑器
const docEditor = new DocsAPI.DocEditor('placeholder', {
  document: {
    url: 'https://your-bucket/minio/example.docx',
    title: 'example.docx',
    key: 'unique-doc-key-12345',     // 文档唯一标识，变更时触发重新加载
    fileType: 'docx',
  },
  editorConfig: {
    callbackUrl: 'https://your-server/api/onlyoffice/callback',
    lang: 'zh-CN',
    user: {
      id: 'uid-001',
      name: '张三',
    },
    customization: {
      forcesave: true,               // 启用强制保存
    },
  },
});
```

## Track 回调处理（核心）

### 请求体结构

```json
{
  "key": "unique-doc-key-12345",
  "status": 1,
  "url": "https://docserver/cache/.../output.docx",
  "token": "jwt-token-here",
  "users": ["uid-001", "uid-002"],
  "actions": [
    {
      "type": 1,
      "userid": "uid-001"
    }
  ],
  "history": {
    "serverVersion": "8.2.1",
    "changes": [...],
    "previous": {
      "key": "old-doc-key",
      "url": "https://docserver/cache/.../previous.docx"
    }
  }
}
```

### 状态码完整解析

| status | 含义 | 处理动作 |
|---|---|---|
| 0 | 文档无变更 | 无操作 |
| 1 | 文档已保存 | 从 `url` 下载最新版并持久化存储 |
| 2 | 文档正在编辑 | 更新在线用户列表 |
| 3 | 强制保存完成 | 从 `url` 下载并存储，通知协作者 |
| 4 | 文档已关闭 | 清理在线用户状态 |
| 6 | 正在编辑但已保存 | 同 status=1，保存最新内容 |
| 7 | 错误 | 记录错误日志 |

### Spring Boot 回调处理

```java
@RestController
@RequestMapping("/api/onlyoffice")
public class OnlyOfficeCallbackController {

    @Autowired
    private DocumentService documentService;

    @PostMapping("/callback")
    public ResponseEntity<Void> handleCallback(@RequestBody CallbackRequest request) {
        // JWT 验证
        // String decodedToken = JwtUtil.verify(request.getToken());

        switch (request.getStatus()) {
            case 1:  // 文档已保存
            case 6:  // 正在编辑已保存
                handleDocumentSaved(request);
                break;
            case 2:  // 正在编辑
                handleDocumentEditing(request);
                break;
            case 3:  // 强制保存
                handleForceSave(request);
                break;
            case 4:  // 文档关闭
                handleDocumentClosed(request);
                break;
            case 7:  // 错误
                log.error("OnlyOffice callback error: key={}", request.getKey());
                break;
        }
        return ResponseEntity.ok().build();
    }

    private void handleDocumentSaved(CallbackRequest request) {
        try {
            // 下载最新文档内容
            byte[] content = downloadFromUrl(request.getUrl());
            // 持久化到 MinIO 或本地存储
            documentService.saveDocument(request.getKey(), content);
            // 保存版本历史
            if (request.getHistory() != null) {
                documentService.saveHistory(request.getKey(), request.getHistory());
            }
        } catch (Exception e) {
            log.error("Failed to save document: key={}", request.getKey(), e);
        }
    }

    private byte[] downloadFromUrl(String url) {
        RestTemplate rest = new RestTemplate();
        return rest.getForObject(url, byte[].class);
    }
}
```

## 协同编辑场景

### 用户在线状态管理

```java
// 使用 Redis 管理在线用户
@Service
public class DocumentSessionManager {

    @Autowired
    private StringRedisTemplate redis;

    private static final String KEY_PREFIX = "doc:session:";

    /**
     * 用户加入编辑
     */
    public void userJoined(String docKey, String userId, String userName) {
        String hashKey = KEY_PREFIX + docKey;
        redis.opsForHash().put(hashKey, userId, userName);
        redis.expire(hashKey, Duration.ofMinutes(30));
    }

    /**
     * 用户离开编辑
     */
    public void userLeft(String docKey, String userId) {
        redis.opsForHash().delete(KEY_PREFIX + docKey, userId);
    }

    /**
     * 获取在线用户列表
     */
    public Map<Object, Object> getOnlineUsers(String docKey) {
        return redis.opsForHash().entries(KEY_PREFIX + docKey);
    }
}
```

## 强制保存 vs 自动保存

| 保存模式 | 触发方式 | 行为 |
|---|---|---|
| 自动保存 | 用户操作后15秒 | 持续写入 Document Server 缓存，不触发回调 |
| 强制保存 | `forcesave: true` + 用户点击保存按钮 | 触发 status=3 回调，返回完整文档 URL |
| 定时保存 | `editorConfig.customization.autosaveTimeout: 300000` | 每5分钟强制触发一次回调 |

### 推荐配置

```javascript
// 初始化配置推荐
editorConfig: {
  customization: {
    forcesave: true,              // 启用强制保存按钮
    autosave: true,               // 启用自动保存
    autosaveTimeout: 600000,       // 600秒无操作后自动强制保存
    compactHeader: false,
    hideRightMenu: false,
    hideRuler: false,
  }
}
```

## 版本历史管理

OnlyOffice 2.0 版本（v7.0+）支持完整的版本历史追踪：

```json
{
  "history": {
    "serverVersion": "8.2.1",
    "changes": [
      {
        "created": "2026-06-02T10:30:00Z",
        "user": {
          "id": "uid-001",
          "name": "张三"
        }
      }
    ],
    "previous": {
      "key": "previous-doc-key",
      "url": "https://docserver/cache/.../prev-version.docx"
    }
  }
}
```

### 版本历史 API 实现

```java
@RestController
@RequestMapping("/api/onlyoffice/history")
public class DocumentHistoryController {

    @GetMapping("/{docKey}")
    public ResponseEntity<HistoryResponse> getHistory(@PathVariable String docKey) {
        // 返回历史版本列表供前端展示
        List<HistoryEntry> history = documentService.getHistory(docKey);

        HistoryResponse response = new HistoryResponse();
        response.setCurrentVersion(history.size());
        response.setHistory(history.stream().map(h -> {
            HistoryItem item = new HistoryItem();
            item.setKey(h.getVersionKey());
            item.setUrl(generateDownloadUrl(h.getStorePath()));
            item.setCreated(h.getCreatedAt());
            item.setUser(new HistoryUser(h.getUserId(), h.getUserName()));
            return item;
        }).collect(Collectors.toList()));

        return ResponseEntity.ok(response);
    }

    // 生成带签名的下载 URL
    private String generateDownloadUrl(String storePath) {
        // 使用 MinIO presigned URL 或临时签名
        return minioService.getPresignedObjectUrl(storePath, Duration.ofMinutes(5));
    }
}
```

## 注意事项

- **Key 不可变**：文档 key 在内容变更后必须更新，否则 Document Server 不会重新加载
- **Callback URL 需公网可达**：Document Server 必须能 POST 到 callbackUrl，内网部署注意网络打通
- **JWT 必配**：所有回调请求携带 token，后端需验证防止伪造回调
- **并发保存**：多人同时编辑时，Document Server 保证最终一致性，后台收到 status=1 需幂等处理
- **超时处理**：callback 应快速返回 200（<5秒），长时间处理阻塞会导致 Document Server 重试
- **存储选型**：回调下载的文档内容建议存 MinIO 而非数据库 Blob，便于版本管理和备份
- **断线恢复**：用户断线重连后 Document Server 自动同步最新内容，无需特殊处理
