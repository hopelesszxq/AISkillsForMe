---
name: onlyoffice-jwt-security
description: OnlyOffice Document Server JWT 身份认证配置、密钥管理、回调安全与生产部署安全加固
tags: [onlyoffice, jwt, security, authentication, callback, production]
---

## 概述

OnlyOffice Document Server 使用 JWT（JSON Web Token）来保护文档编辑过程中的通信安全。JWT 用于验证来自前端应用的请求和 Document Server 回调的合法性。配置不当会导致未授权访问、文档泄露或回调劫持。本文覆盖 JWT 配置、密钥管理、回调验证和生产安全加固。

## JWT 工作原理

```
前端应用                    OnlyOffice Document Server
    |                              |
    |--- 1. 生成 JWT Token ------->|
    |--- 2. 传入 config.token ---->| (打开编辑器)
    |                              |
    |    <--- 3. 回调 + JWT -------| (文档保存/编辑事件)
    |--- 4. 验证回调 JWT --------->|
    |                              |
```

- 前端配置 Document Editor 时传入 JWT token
- Document Server 在 HTTP 回调中携带 JWT
- 后端应用验证回调 JWT 的真实性

## JWT 配置

### Document Server 端 (docker-compose)

```yaml
version: '3.8'
services:
  onlyoffice-document-server:
    image: onlyoffice/documentserver:8.2
    container_name: onlyoffice-ds
    ports:
      - "8080:80"
    environment:
      # JWT 密钥（必须修改默认值）
      - JWT_ENABLED=true
      - JWT_SECRET=your-256-bit-secret-change-in-production
      - JWT_HEADER=Authorization
      - JWT_IN_BODY=true
      # 内网地址（回调地址）
      - WOPI_ENABLED=false
    volumes:
      - ./data:/var/www/onlyoffice/Data
      - ./logs:/var/log/onlyoffice
    restart: always
```

### Spring Boot 后端 JWT 配置

```yaml
onlyoffice:
  jwt:
    secret: your-256-bit-secret-change-in-production
    # 必须与 Document Server 的 JWT_SECRET 一致
    algorithm: HS256
    # 文档编辑配置
    editor:
      url: https://doc.example.com
      callback-url: https://api.example.com/api/onlyoffice/callback
```

## Java 后端 JWT 实现

### 1. 添加依赖

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

### 2. JWT 工具类

```java
@Component
public class OnlyOfficeJwtUtil {

    private final SecretKey secretKey;

    public OnlyOfficeJwtUtil(@Value("${onlyoffice.jwt.secret}") String secret) {
        byte[] keyBytes = secret.getBytes(StandardCharsets.UTF_8);
        // 确保密钥长度 >= 256 bits
        if (keyBytes.length < 32) {
            keyBytes = Arrays.copyOf(keyBytes, 32);
        }
        this.secretKey = Keys.hmacShaKeyFor(keyBytes);
    }

    /**
     * 生成 Document Editor 配置需要的 JWT Token
     */
    public String createEditorToken(EditorConfig config) {
        return Jwts.builder()
            .subject("onlyoffice-editor")
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + 3600_000)) // 1h 过期
            .claim("document", Map.of(
                "fileType", config.getFileType(),
                "key", config.getDocumentKey(),
                "title", config.getTitle(),
                "url", config.getFileUrl(),
                "permissions", Map.of(
                    "edit", config.isEditable(),
                    "download", config.isDownloadable(),
                    "print", config.isPrintable()
                )
            ))
            .claim("editorConfig", Map.of(
                "callbackUrl", config.getCallbackUrl(),
                "lang", config.getLanguage(),
                "mode", config.isEditable() ? "edit" : "view",
                "user", Map.of(
                    "id", config.getUserId(),
                    "name", config.getUserName()
                )
            ))
            .signWith(secretKey, Jwts.SIG.HS256)
            .compact();
    }

    /**
     * 验证回调请求的 JWT Token
     */
    public Claims parseCallbackToken(String token) {
        try {
            return Jwts.parser()
                .verifyWith(secretKey)
                .build()
                .parseSignedClaims(token)
                .getPayload();
        } catch (JwtException e) {
            throw new SecurityException("Invalid callback JWT token", e);
        }
    }

    /**
     * 验证回调 JWT（从请求头或请求体中提取）
     */
    public Claims validateCallback(HttpServletRequest request) {
        // 1. 先从 Authorization 头获取
        String authHeader = request.getHeader("Authorization");
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            return parseCallbackToken(authHeader.substring(7));
        }
        // 2. 从请求体获取（JWT_IN_BODY=true 时）
        try {
            String body = new String(request.getInputStream().readAllBytes(), 
                StandardCharsets.UTF_8);
            @SuppressWarnings("unchecked")
            Map<String, Object> bodyMap = new ObjectMapper().readValue(body, Map.class);
            if (bodyMap.containsKey("token")) {
                return parseCallbackToken((String) bodyMap.get("token"));
            }
        } catch (IOException e) {
            throw new SecurityException("Cannot read callback body");
        }
        throw new SecurityException("No JWT token found in callback request");
    }
}
```

### 3. 回调处理控制器

```java
@RestController
@RequestMapping("/api/onlyoffice")
public class OnlyOfficeCallbackController {

    private final OnlyOfficeJwtUtil jwtUtil;
    private final DocumentService documentService;

    @PostMapping("/callback")
    public ResponseEntity<Map<String, Object>> callback(
            HttpServletRequest request,
            @RequestBody(required = false) String body) {
        try {
            // 验证 JWT
            Claims claims = jwtUtil.validateCallback(request);
            
            // 解析回调数据（实际应从 body 解析）
            // 这里简化处理
            CallbackData callbackData = parseCallback(body);
            
            // 处理不同回调状态
            switch (callbackData.getStatus()) {
                case 1: // 文档已就绪
                    log.info("Document ready: {}", callbackData.getKey());
                    break;
                case 2: // 文档正在编辑
                    log.debug("Document editing: {}", callbackData.getKey());
                    break;
                case 3: // 文档变更需要保存
                    documentService.saveDocument(callbackData);
                    break;
                case 4: // 文档关闭
                    log.info("Document closed: {}", callbackData.getKey());
                    break;
                case 6: // 强制保存
                    documentService.forceSave(callbackData);
                    break;
                case 7: // 错误
                    log.error("Document error: {}", callbackData.getError());
                    break;
            }
            
            return ResponseEntity.ok(Map.of("error", 0));
            
        } catch (SecurityException e) {
            log.warn("Invalid JWT callback: {}", e.getMessage());
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .body(Map.of("error", 1, "message", "Invalid token"));
        }
    }
}
```

## 前端集成示例

```javascript
// 打开编辑器时传入 JWT
const docEditor = new DocsAPI.DocEditor("placeholder", {
    document: {
        fileType: "docx",
        key: documentKey,
        title: "文档标题.docx",
        url: "https://api.example.com/api/onlyoffice/download?key=" + documentKey,
        permissions: {
            edit: true,
            download: true,
            print: true
        }
    },
    documentType: "word",
    editorConfig: {
        callbackUrl: "https://api.example.com/api/onlyoffice/callback",
        lang: "zh-CN",
        user: {
            id: "user-123",
            name: "张三"
        }
    },
    token: "eyJhbGciOiJIUzI1NiIs..."  // 后端生成的 JWT
});
```

## 密钥管理最佳实践

### 1. 使用强密钥

```java
// 生成安全的 HMAC-SHA256 密钥（256位）
KeyGenerator keyGen = KeyGenerator.getInstance("HmacSHA256");
keyGen.init(256);
SecretKey key = keyGen.generateKey();
String base64Key = Base64.getEncoder().encodeToString(key.getEncoded());
System.out.println(base64Key); // 保存此值到环境变量
```

### 2. 环境变量配置

```yaml
# application-prod.yml
onlyoffice:
  jwt:
    secret: ${ONLYOFFICE_JWT_SECRET}  # 从环境变量读取
```

```bash
# Docker 环境变量
ONLYOFFICE_JWT_SECRET=$(cat /run/secrets/onlyoffice_jwt)
```

### 3. 密钥轮换策略

```java
// 支持多密钥验证（旧密钥用于验证，新密钥用于生成）
@Component
public class RotatingJwtUtil {
    
    private final SecretKey signingKey;      // 当前签名密钥
    private final List<SecretKey> verifyingKeys;  // 所有可验证的密钥

    public RotatingJwtUtil(
            @Value("${onlyoffice.jwt.secret}") String currentSecret,
            @Value("${onlyoffice.jwt.previous-secrets:}") String previousSecrets) {
        this.signingKey = createKey(currentSecret);
        this.verifyingKeys = new ArrayList<>();
        this.verifyingKeys.add(signingKey);
        if (!previousSecrets.isEmpty()) {
            for (String secret : previousSecrets.split(",")) {
                verifyingKeys.add(createKey(secret.trim()));
            }
        }
    }

    public String createToken() { /* 使用 signingKey */ }
    
    public Claims verifyToken(String token) {
        for (SecretKey key : verifyingKeys) {
            try {
                return Jwts.parser().verifyWith(key).build()
                    .parseSignedClaims(token).getPayload();
            } catch (JwtException ignored) {}
        }
        throw new SecurityException("Token not valid with any known key");
    }
}
```

## 生产环境安全加固

### 1. HTTPS 强制

```nginx
# Nginx 反向代理
server {
    listen 443 ssl http2;
    server_name doc.example.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    location / {
        proxy_pass http://onlyoffice-ds:80;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name doc.example.com;
    return 301 https://$host$request_uri;
}
```

### 2. 回调 IP 白名单

```java
@Component
public class CallbackIpWhitelistFilter extends OncePerRequestFilter {

    private final List<String> allowedIps = List.of(
        "10.0.0.0/8",     // Document Server 内网 IP
        "172.16.0.0/12",
        "192.168.0.0/16"
    );

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        
        String remoteIp = request.getRemoteAddr();
        if (!isAllowedIp(remoteIp)) {
            response.setStatus(HttpStatus.FORBIDDEN.value());
            return;
        }
        chain.doFilter(request, response);
    }
}
```

### 3. JWT 验证严格模式

```java
public Claims parseCallbackToken(String token) {
    return Jwts.parser()
        .verifyWith(secretKey)
        .requireSubject("onlyoffice-editor")  // 校验 subject
        .requireIssuer("onlyoffice-server")   // 校验签发者
        .clockSkewSeconds(30)                 // 允许 30 秒时间偏差
        .build()
        .parseSignedClaims(token)
        .getPayload();
}
```

## 常见问题排查

| 问题 | 原因 | 解决 |
|---|---|---|
| 编辑器打开显示 "Invalid token" | JWT_SECRET 前后端不一致 | 检查所有配置的密钥是否一致 |
| 文档保存失败 | 回调 JWT 验证失败 | 确认 JWT_IN_BODY=true 且回调正确携带 token |
| Token 过期 | JWT 过期时间太短 | 设置合理的 expiration（建议 1-2 小时） |
| 密钥长度不足 | HS256 需要至少 256 位密钥 | 使用 32 字节以上的密钥 |
| 跨域导致回调失败 | Document Server 无法调用回调 URL | 确认回调 URL 公网可达且无跨域限制 |
