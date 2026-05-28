---
name: onlyoffice-94-upgrade
description: OnlyOffice 9.4 重大架构变更：去 RabbitMQ/数据库依赖、单进程合并、无 20 文档并发限制
tags: [onlyoffice, docs, architecture, upgrade, deployment]
---

## 概述

OnlyOffice DocumentServer 9.4.0 是一个**重大架构升级**，移除了对 RabbitMQ 和外部数据库的依赖，将多组件合并为单进程，大幅简化部署和运维。从低版本升级时需关注架构变化。

## 架构变化对比

| 方面 | 9.3.x 及之前 | 9.4.0+ |
|------|-------------|--------|
| 进程模型 | 多服务多进程 | 单进程（组件合并） |
| 消息队列 | 依赖 RabbitMQ | **不再需要 RabbitMQ** |
| 外部数据库 | 依赖 PostgreSQL/MySQL | **不再需要外部数据库** |
| 并发文档数 | 最多 20 个同时编辑 | **无限制** |
| 代码混淆 | 代码经过 minify 混淆 | **可读源码** |
| 资源消耗 | 多进程占用较多 | 单进程优化，资源更省 |

### 架构迁移示意图

```yaml
# 9.3.x 架构（已废弃）
services:
  onlyoffice-document-server:
    image: onlyoffice/documentserver:8.9
    depends_on:
      - rabbitmq      # ❌ 不再需要
      - postgres      # ❌ 不再需要
    environment:
      - AMQP_URI=amqp://guest:guest@rabbitmq
      - DB_HOST=postgres
      - DB_NAME=onlyoffice
      - DB_USER=onlyoffice
      - DB_PWD=onlyoffice

# 9.4.0 架构（简化）
services:
  onlyoffice-document-server:
    image: onlyoffice/documentserver:9.4
    # ✅ 不再需要 RabbitMQ 和 PostgreSQL
    # ✅ 直接运行，配置极大简化
    environment:
      - JWT_SECRET=my-secret-key    # JWT 认证（必配）
      - WOPI_ENABLED=true           # WOPI 协议支持
```

## 迁移到 9.4.0

### 1. 使用新版 Docker Compose

```yaml
# docker-compose.yml — 9.4.0 简化版
version: "3.8"

services:
  onlyoffice:
    image: onlyoffice/documentserver:9.4
    container_name: onlyoffice-server
    ports:
      - "8088:80"       # HTTP
      - "443:443"       # HTTPS（可选，建议用反向代理）
    environment:
      # JWT 认证（生产环境必须设置）
      JWT_SECRET: ${ONLYOFFICE_JWT_SECRET:?请设置 JWT_SECRET}
      JWT_HEADER: Authorization
      JWT_IN_BODY: false

      # 日志级别
      LOG_LEVEL: warn

      # WOPI 协议（配合 Nextcloud/ownCloud）
      WOPI_ENABLED: true

    volumes:
      # 持久化文档数据
      - onlyoffice-data:/var/www/onlyoffice/Data
      # 日志
      - onlyoffice-logs:/var/log/onlyoffice
    restart: unless-stopped
    # 资源限制（推荐）
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: "4"
        reservations:
          memory: 2G
          cpus: "2"

volumes:
  onlyoffice-data:
  onlyoffice-logs:
```

### 2. 配置更改清单

```yaml
# 旧配置 → 新配置映射
# ❌ 不再支持的配置项：
#   AMQP_URI            → 已移除
#   DB_HOST/DB_NAME/... → 已移除
#   RABBITMQ_*          → 已移除
#   REDIS_SERVER         → 已移除（9.4 内部处理）

# ✅ 保留的关键配置：
#   JWT_SECRET           → 必须设置
#   JWT_HEADER           → 默认 Authorization
#   WOPI_ENABLED         → WOPI 协议支持
#   LOG_LEVEL            → warn / info / debug

# ⚠️ 新增配置：
#   JWT_IN_BODY          → 是否在请求体中传递 JWT（默认 false）
#   METRICS_ENABLED      → 暴露 /metrics 端点
#   METRICS_PORT         → metrics 端口（默认 9090）
```

### 3. 集成示例（Java Spring Boot）

```java
// OnlyOffice 回调处理 — 9.4.0 兼容
@RestController
@RequestMapping("/onlyoffice")
public class OnlyOfficeCallbackController {

    @Autowired
    private OnlyOfficeConfig config;

    @PostMapping("/callback")
    public ResponseEntity<Map<String, Object>> callback(
            @RequestBody Map<String, Object> body,
            @RequestHeader("Authorization") String authHeader) {

        // 1. JWT 验证
        String token = extractToken(authHeader);
        if (!verifyJwt(token, config.getJwtSecret())) {
            return ResponseEntity.status(401)
                .body(Map.of("error", 1, "message", "JWT 验证失败"));
        }

        // 2. 处理回调
        Integer status = (Integer) body.get("status");
        String key = (String) body.get("key");
        String url = (String) body.get("url");

        switch (status) {
            case 1 -> handleEditing(key);           // 正在编辑
            case 2 -> handleSaving(key, url);       // 保存文档
            case 3 -> handleSavingError(key);       // 保存失败
            case 4 -> handleClosed(key);            // 关闭无变更
            case 6 -> handleForceSaving(key, url);  // 强制保存
            case 7 -> handleCorrupted(key);         // 文档损坏
        }

        return ResponseEntity.ok(Map.of("error", 0));
    }

    // 保存文档回调
    private void handleSaving(String key, String url) {
        byte[] document = downloadDocument(url);
        saveToStorage(key, document);
    }

    private String extractToken(String authHeader) {
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            return authHeader.substring(7);
        }
        return authHeader;
    }

    private boolean verifyJwt(String token, String secret) {
        try {
            Algorithm algorithm = Algorithm.HMAC256(secret);
            JWT.require(algorithm).build().verify(token);
            return true;
        } catch (Exception e) {
            log.warn("JWT 验证失败: {}", e.getMessage());
            return false;
        }
    }
}

// 配置
@Data
@Component
@ConfigurationProperties(prefix = "onlyoffice")
public class OnlyOfficeConfig {
    private String docsUrl = "http://localhost:8088";   // 文档服务地址
    private String jwtSecret = "my-secret-key";          // JWT 密钥
    private Integer jwtExpiration = 3600;                // JWT 过期时间（秒）
}
```

```yaml
# application.yml
onlyoffice:
  docs-url: http://onlyoffice:8088
  jwt-secret: ${ONLYOFFICE_JWT_SECRET}
  jwt-expiration: 3600
```

### 4. 前端集成（生成文档 URL）

```java
@Service
public class DocumentUrlService {

    @Autowired
    private OnlyOfficeConfig config;

    public String generateEditorUrl(String fileId, String fileName, 
                                     String user, boolean isEdit) {
        // 构建 config JSON
        Map<String, Object> document = Map.of(
            "fileType", FilenameUtils.getExtension(fileName),
            "key", generateKey(fileId),
            "title", fileName,
            "url", buildDownloadUrl(fileId),
            "permissions", Map.of(
                "edit", isEdit,
                "download", true,
                "print", true
            )
        );

        Map<String, Object> editorConfig = Map.of(
            "callbackUrl", buildCallbackUrl(),
            "user", Map.of(
                "id", user,
                "name", user
            ),
            "lang", "zh-CN",
            "mode", isEdit ? "edit" : "view",
            "customization", Map.of(
                "autosaveFrequency", 600000,        // 10分钟自动保存
                "forcesave", true,                   // 强制保存
                "compactHeader", true,               // 紧凑头部
                "feedback", false,                   // 禁用反馈
                "help", false,                       // 禁用帮助
                "hideRightMenu", false
            )
        );

        Map<String, Object> configMap = Map.of(
            "document", document,
            "editorConfig", editorConfig,
            "token", generateJwt(document, editorConfig)  // JWT 签名
        );

        // 返回前端模板中的配置
        return JSON.toJSONString(configMap);
    }

    private String generateJwt(Object... payloads) {
        Algorithm algorithm = Algorithm.HMAC256(config.getJwtSecret());
        return JWT.create()
            .withClaim("document", JSON.toJSONString(payloads[0]))
            .withClaim("editorConfig", JSON.toJSONString(payloads[1]))
            .withExpiresAt(new Date(System.currentTimeMillis() 
                + config.getJwtExpiration() * 1000L))
            .sign(algorithm);
    }

    private String generateKey(String fileId) {
        // 文档 key — 内容变更时需变化，OnlyOffice 用 key 判断是否需要刷新缓存
        return DigestUtils.md5DigestAsHex(
            (fileId + ":" + System.currentTimeMillis() / 600_000).getBytes());
    }
}
```

## 9.4.0 新功能亮点

### 1. 表单签名增强

```javascript
// 自定义插件：控制签名
window.Asc.plugin.init = function() {
    // 9.4 新增：移动光标到/离开字段
    this.callCommand(function() {
        Api.MoveCursorToField("signature_1");  // 移动到签名域
    });
};
```

### 2. Dark Mode 支持（表格）

```javascript
// 通过 API 设置 Dark Mode（仅 Spreadsheet）
editor.setProperty({
    "darkMode": true,
    "darkModeOptions": {
        "theme": "dark-contrast"  // 或 "dark-default"
    }
});
```

### 3. 禁用插件

```json
// docs 配置中禁用不安全的插件
{
    "plugins": {
        "disable": ["plugin-id-1", "plugin-id-2"],
        "pluginsData": ["http://..."],
        "autostart": true
    }
}
```

## 迁移注意事项

1. **无法回滚**：9.4.0 使用的内部存储格式与之前不兼容，升级后不可降级
2. **原有数据迁移**：升级前需要导出旧版数据库中的文档元数据（连接记录等）
3. **JWT 必配**：9.4 默认开启 JWT 认证，不配置将无法使用
4. **Nginx 反向代理**：如果使用 HTTPS，需要配置 WebSocket 支持
   ```nginx
   location /onlyoffice/ {
       proxy_pass http://localhost:8088/;
       proxy_http_version 1.1;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
       proxy_set_header Host $host;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
   }
   ```
5. **资源评估**：单进程模型下，单个容器建议至少 4G 内存（比之前少，但仍需充裕）
6. **保留旧版本的 docker-compose**：升级前保存一份，确保能回退
7. **监控指标**：9.4 新增 `/metrics` 端点（Prometheus 格式），可接入监控系统
