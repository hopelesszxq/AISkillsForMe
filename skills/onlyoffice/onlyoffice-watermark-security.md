---
name: onlyoffice-watermark-security
description: OnlyOffice 文档水印与文档级安全权限控制：文字水印、图片水印、权限矩阵、访问控制与防泄露策略
tags: [onlyoffice, watermark, security, document-permissions, access-control, drm]
---

## 概述

OnlyOffice Document Server 提供了丰富的文档安全功能，包括**编辑态水印**（文字/图片）、**文档级权限控制**（禁止下载/打印/复制）、**访问令牌校验**以及**回调签名验证**。本文聚焦文档层面的安全防护，区别于 JWT 服务间认证。

## 一、编辑时水印（文字水印）

OnlyOffice 支持在文档编辑界面上叠加文字水印，用于防止截屏泄露。

### 前端配置

```javascript
// 集成配置
const config = {
    document: {
        fileType: "docx",
        key: "doc_key_001",
        title: "保密合同.docx",
        url: "https://your-server/api/document/download"
    },
    editorConfig: {
        callbackUrl: "https://your-server/api/callback",
        user: {
            id: "user-001",
            name: "张三",
            group: "财务部"
        }
    },
    // === 水印配置 ===
    watermark: {
        type: "text",              // text | image | all
        text: "机密文件 - ${user.name} - ${date}",
        fontName: "Microsoft YaHei",
        fontSize: 16,
        fontColor: "FF0000",       // 红色
        transparency: 0.3,
        rotation: -30,
        // 页面范围
        pages: "1-3,5",
        // 放置位置
        align: "center",           // left | center | right
        margins: {
            left: 20,
            top: 20,
            right: 20,
            bottom: 20
        },
        // 水印参数
        parameters: {
            // 支持的自定义变量（在 text 中用 ${xxx} 引用）
            userName: "${user.name}",
            date: new Date().toLocaleDateString("zh-CN")
        }
    }
};

// 初始化
const docEditor = new DocsAPI.DocEditor("placeholder", config);
```

### 效果说明

- `text` 中的 `${user.name}` 自动替换为当前用户姓名
- `${date}` 替换为访问日期
- `transparency: 0.3` 表示 30% 透明度
- `rotation: -30` 水印旋转角度（负值逆时针）
- `pages: "1-3,5"` 只在第 1、2、3、5 页显示水印

## 二、图片水印

支持将公司 Logo 或自定义图片作为水印叠加在文档上。

```javascript
watermark: {
    type: "image",
    imageUrl: "https://your-server/watermark/logo.png",
    // 图片缩放
    width: 200,
    height: 100,
    // 重复平铺
    tile: true,
    spacing: {
        x: 50,
        y: 50
    },
    transparency: 0.2,
    rotation: 0,
    pages: "all",
    align: "center"
}
```

### 文字+图片混合水印

```javascript
watermark: {
    type: "all",  // 同时显示文字和图片
    text: "CONFIDENTIAL",
    imageUrl: "https://your-server/watermark/shield.png",
    imageWidth: 80,
    imageHeight: 80,
    transparency: 0.25,
    // 文字图片共享透明度
    parameters: {
        // ...
    }
}
```

## 三、文档级权限控制（权限矩阵）

OnlyOffice v8.x 支持细粒度的文档操作权限控制，通过 `editorConfig.customization.goback` 和 `permissions` 字段实现。

### 权限配置完整示例

```javascript
const config = {
    document: {
        // ...
        permissions: {
            // === 编辑权限 ===
            edit: true,                // 允许编辑（false=只读）
            review: true,              // 允许修订
            comment: true,             // 允许批注
            fillForms: true,           // 允许填写表单
            modifyContentControl: true,// 允许修改内容控件

            // === 导出权限 ===
            download: false,           // 禁止下载原始文件
            print: false,              // 禁止打印
            copy: false,               // 禁止复制内容到剪贴板
            export: false,             // 禁止导出为其他格式

            // === 操作限制 ===
            chat: true,                // 允许聊天
            reviewGroups: ["财务部"],   // 仅特定组可接受/拒绝修订
            commentGroups: ["财务部"],  // 仅特定组可添加批注
            userInfoGroups: ["管理员"], // 仅特定组可查看用户信息
            protect: true,             // 启用文档保护（禁止修改格式）

            // === 自动保存 ===
            forcesave: true,           // 强制启用主动保存
        }
    }
};
```

### 按角色动态配置权限

```java
// Java 服务端根据用户角色动态生成权限
public Map<String, Object> buildPermissions(User user, Document document) {
    Map<String, Object> permissions = new HashMap<>();

    switch (user.getRole()) {
        case "ADMIN":
            permissions.put("edit", true);
            permissions.put("download", true);
            permissions.put("print", true);
            permissions.put("copy", true);
            break;
        case "REVIEWER":
            permissions.put("edit", false);
            permissions.put("review", true);
            permissions.put("comment", true);
            permissions.put("download", false);
            permissions.put("print", false);
            permissions.put("copy", false);
            break;
        case "VIEWER":
            permissions.put("edit", false);
            permissions.put("review", false);
            permissions.put("comment", false);
            permissions.put("download", false);
            permissions.put("print", false);
            permissions.put("copy", false);
            break;
        case "FORM_FILLER":
            permissions.put("edit", false);
            permissions.put("fillForms", true);
            permissions.put("download", false);
            permissions.put("print", true);
            break;
    }

    // 其他人查看但不可编辑
    return permissions;
}
```

## 四、基于 JWT 的文档访问令牌

除了服务间认证的 JWT，还可以对每次文档查看请求下发短期 Token。

### 服务端生成 Token

```java
// 使用 io.jsonwebtoken:jjwt
public String generateDocumentToken(Document document, User user, long expiryMinutes) {
    Map<String, Object> claims = new HashMap<>();
    claims.put("documentId", document.getId());
    claims.put("userId", user.getId());
    claims.put("role", user.getRole());
    claims.put("permissions", buildPermissions(user, document));

    // 水印个性化参数
    claims.put("watermark", Map.of(
        "text", "机密 - " + user.getName(),
        "transparency", 0.25
    ));

    return Jwts.builder()
        .setClaims(claims)
        .setIssuedAt(new Date())
        .setExpiration(Date.from(Instant.now().plus(expiryMinutes, ChronoUnit.MINUTES)))
        .signWith(SignatureAlgorithm.HS256, secretKey.getBytes(StandardCharsets.UTF_8))
        .compact();
}
```

### 前端集成

```javascript
// 从服务端获取 Token
const response = await fetch("/api/document/token", {
    method: "POST",
    body: JSON.stringify({ docId: "doc_001" }),
    headers: { "Content-Type": "application/json" }
});
const { token } = await response.json();

const config = {
    document: {
        fileType: "docx",
        key: "doc_key_001",
        title: "保密合同.docx",
        url: "/api/document/download?token=" + token
    },
    editorConfig: {
        callbackUrl: "/api/callback?token=" + token,
        // ...
    },
    // Token 放在请求 Header 中
    requestToken: token
};

const docEditor = new DocsAPI.DocEditor("placeholder", config);
```

## 五、文档水印动态下发的服务端实现

```java
@RestController
@RequestMapping("/api/document")
public class DocumentController {

    @PostMapping("/config")
    public ResponseEntity<Map<String, Object>> getDocumentConfig(
            @RequestBody DocumentRequest request,
            @AuthenticationPrincipal User user) {

        Document document = documentService.findById(request.getDocId());
        Map<String, Object> permissions = buildPermissions(user, document);

        Map<String, Object> watermark = null;
        if (document.isConfidential()) {
            watermark = Map.of(
                "type", "text",
                "text", "机密 - " + user.getName() + " - " + LocalDate.now(),
                "fontSize", 14,
                "transparency", 0.25,
                "rotation", -30,
                "fontColor", "FF0000"
            );
        }

        Map<String, Object> config = new HashMap<>();
        config.put("document", Map.of(
            "fileType", document.getFileType(),
            "key", document.getKey(),
            "title", document.getTitle(),
            "url", generateDownloadUrl(document, user),
            "permissions", permissions
        ));
        config.put("editorConfig", Map.of(
            "callbackUrl", generateCallbackUrl(document, user),
            "mode", user.canEdit() ? "edit" : "view",
            "user", Map.of("id", user.getId(), "name", user.getName())
        ));
        if (watermark != null) {
            config.put("watermark", watermark);
        }

        return ResponseEntity.ok(config);
    }
}
```

## 六、防泄露策略最佳实践

### 1. 屏幕水印 + 追踪

```
结合 OnlyOffice 水印 + 回调中的 user 信息，截屏泄露时可追溯责任人。
建议水印包含：用户名、时间、IP 后四位。
```

### 2. 文档访问时效性

```
Token 有效期建议控制在 30 分钟以内，超时需重新鉴权。
强制保存后关闭编辑器，释放 Token。
```

### 3. 防止恶意下载

```
- 设置 permissions.download = false（适用于 OnlyOffice 8.x+）
- 服务端文档 URL 使用临时 Token（无法直接访问）
- callback 中 forceSave 后，将原始文件存储到安全存储（MinIO 私有桶）
```

### 4. 文档追踪回调

```java
// 回调中记录用户操作日志
public void onDocumentCallback(DocumentCallback callback) {
    DocumentCallbackLog log = new DocumentCallbackLog();
    log.setDocKey(callback.getKey());
    log.setUserId(callback.getUser().getId());
    log.setActions(callback.getActions());  // edit, review, comment, download etc.
    log.setTimestamp(Instant.now());

    // 如果检测到异常行为（如批量下载），触发告警
    if (callback.getActions().contains("download") && !canDownload(callback.getUser())) {
        alertService.sendAlert("UNAUTHORIZED_DOWNLOAD", log);
    }

    callbackLogRepository.save(log);
}
```

## 注意事项

1. **水印仅在编辑器中显示**：导出为 PDF/打印时水印可能不包含，需要服务端叠加或设置 `print: false`
2. **权限与界面一致性**：设置 `edit: false` 时工具栏会自动隐藏编辑按钮，但防君子不防小人（禁止下载+服务端权限校验才能防泄露）
3. **性能影响**：水印叠加是前端渲染，图片水印比文字水印更耗资源，大量并发编辑时注意服务器负载
4. **OnlyOffice 版本兼容**：水印功能从 v7.4+ 开始支持，`permissions` 细粒度控制从 v8.0+ 开始
5. **权限在回调中验证**：前端配置的 `permissions` 只是 UI 展示控制，真正授权应在回调 URL 的服务端做二次校验
6. **自定义变量转义**：水印文字中的 `${user.name}` 应做 HTML 转义，防止 XSS 注入
