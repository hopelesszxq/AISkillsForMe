---
name: onlyoffice-document-comparison
description: OnlyOffice 文档对比 API：差异比较、变更追踪与集成实践
tags: [onlyoffice, document, comparison, diff, api]
---

## 概述

OnlyOffice 提供**文档对比（Document Comparison）**功能，可以对比两份文档的差异并以"修订模式"展示。支持 docx、odt 格式。适合合同版本对比、文档审阅、法律文件差异追踪等场景。

## 方式一：Document Builder API（服务端对比）

### 基本原理

1. 加载原始文档和目标文档
2. 调用对比方法生成差异结果
3. 保存结果文档，用 OnlyOffice Editor 打开后以修订模式（Track Changes）展示

### Java 调用示例

```java
// 使用 OnlyOffice Document Builder 命令行
// 需要部署 Document Builder（独立包或 Docker）

import java.nio.file.*;

public class DocumentComparator {

    private static final String DOC_BUILDER_PATH = "/opt/onlyoffice/desktopeditors/converter";

    /**
     * 对比两份文档，生成差异结果
     * @param originalPath 原版文档路径
     * @param modifiedPath 修改版文档路径
     * @param outputPath   输出差异文档路径
     * @param format       输出格式（"pdf" 或 "docx"）
     */
    public void compare(String originalPath, String modifiedPath,
                        String outputPath, String format) {
        try {
            // 构建 Document Builder 脚本
            String script = String.format("""
                var oDocument = Api.LoadDocument("%s");
                var oCompared = Api.LoadDocument("%s");
                var oChanges = oDocument.CompareDocuments(oCompared);
                oDocument.SaveComparison(oChanges);
                oDocument.SaveToFile("%s", "%s");
                """,
                originalPath.replace("\\", "/"),
                modifiedPath.replace("\\", "/"),
                outputPath.replace("\\", "/"),
                format);

            Path tempScript = Files.createTempFile("compare-", ".js");
            Files.writeString(tempScript, script);

            ProcessBuilder pb = new ProcessBuilder(
                DOC_BUILDER_PATH,
                "--script", tempScript.toString()
            );
            Process process = pb.start();
            int exitCode = process.waitFor(30, TimeUnit.SECONDS) ? process.exitValue() : -1;

            Files.deleteIfExists(tempScript);

            if (exitCode != 0) {
                String err = new String(process.getErrorStream().readAllBytes());
                throw new RuntimeException("Document comparison failed: " + err);
            }
        } catch (Exception e) {
            throw new RuntimeException("Compare documents error", e);
        }
    }
}
```

### JavaScript Document Builder 脚本（独立用法）

```javascript
// 对比两个文档并生成修订版
(function() {
    var sOriginal = Api.GetUrlToFileName("original.docx");
    var sModified = Api.GetUrlToFileName("modified.docx");

    var oDocument = Api.LoadDocument(sOriginal);
    var oCompared = Api.LoadDocument(sModified);

    // 执行对比，返回修订集合
    var oChanges = oDocument.CompareDocuments(oCompared);

    // 保存对比结果（作为带修订标记的新文档）
    oDocument.SaveComparison(oChanges);

    // 输出
    oDocument.SaveToFile("comparison-result.docx", "docx");
    oDocument.SaveToFile("comparison-result.pdf", "pdf");
})();
```

## 方式二：OnlyOffice 文档编辑器中直接对比（前端集成）

在 OnlyOffice Document Editor 中，用户可以内置开启对比功能：

```javascript
// 初始化编辑器时配置对比按钮
const docEditor = new DocsAPI.DocEditor("placeholder", {
    document: {
        fileType: "docx",
        url: "https://example.com/original.docx",
        title: "Contract_v1.docx",
        // 对比文档配置
        comparison: {
            url: "https://example.com/modified.docx",
            fileType: "docx",
            title: "Contract_v2.docx"
        }
    },
    editorConfig: {
        callbackUrl: "https://your-server/callback",
        // 开启比较模式
        mode: "edit",
        // 允许用户通过 UI 触发对比
        features: {
            comparison: true
        }
    },
    events: {
        onRequestCompareDocument: function() {
            // 前端触发对比时的回调
            console.log("Document comparison requested");
        }
    }
});
```

### 通过 URL 参数开启对比模式

```
https://your-onlyoffice-server/example.docx?comparison=modified.docx
```

## 方式三：REST API 对比端点（Document Server 7.3+）

OnlyOffice Document Server 7.3+ 提供 HTTP API 进行文档对比：

```bash
POST /coauthoring/ComparisonService.ashx HTTP/1.1
Host: your-onlyoffice-server
Content-Type: application/json

{
    "vKey": "your-jwt-token",
    "original": {
        "url": "http://storage/contract_v1.docx",
        "fileType": "docx",
        "key": "unique-doc-key-v1"
    },
    "modified": {
        "url": "http://storage/contract_v2.docx",
        "fileType": "docx",
        "key": "unique-doc-key-v2"
    },
    "outputType": "url",
    "async": false
}
```

响应：

```json
{
    "result": {
        "url": "http://storage/comparison-result.docx",
        "key": "comparison-result-key",
        "hasChanges": true,
        "changesCount": 15,
        "expire": "2026-06-08T12:00:00Z"
    }
}
```

## 对比结果的处理

对比生成的结果文档包含 **修订标记（Track Changes）**：

- **插入内容**：下划线 + 绿色（通常）
- **删除内容**：删除线 + 红色
- **格式变更**：单独标记
- 编辑器的 `Accept All` / `Reject All` 功能可用

### 获取变更统计

```javascript
// 通过回调或编辑器事件获取变更数量
docEditor.on("onDocumentStateChange", function(info) {
    if (info.type === "document.change") {
        const changesCount = docEditor.getChanges().length;
        console.log(`Document has ${changesCount} changes`);
    }
});
```

## Spring Boot 服务端封装

```java
@Slf4j
@Service
public class DocumentComparisonService {

    @Value("${onlyoffice.ds.url}")
    private String documentServerUrl;

    @Value("${onlyoffice.ds.jwt-secret}")
    private String jwtSecret;

    private final RestTemplate restTemplate;

    /**
     * 请求文档对比
     */
    public ComparisonResult compareDocuments(String originalUrl, String modifiedUrl) {
        Map<String, Object> request = new HashMap<>();
        request.put("vKey", generateJwt());

        Map<String, Object> original = new HashMap<>();
        original.put("url", originalUrl);
        original.put("fileType", "docx");
        original.put("key", UUID.randomUUID().toString());
        request.put("original", original);

        Map<String, Object> modified = new HashMap<>();
        modified.put("url", modifiedUrl);
        modified.put("fileType", "docx");
        modified.put("key", UUID.randomUUID().toString());
        request.put("modified", modified);
        request.put("outputType", "url");
        request.put("async", false);

        String url = documentServerUrl + "/coauthoring/ComparisonService.ashx";
        ResponseEntity<Map> response = restTemplate.postForEntity(url, request, Map.class);

        Map<String, Object> result = (Map<String, Object>) response.getBody().get("result");
        return new ComparisonResult(
            (String) result.get("url"),
            (boolean) result.get("hasChanges"),
            ((Number) result.get("changesCount")).intValue()
        );
    }

    private String generateJwt() {
        return Jwts.builder()
            .issuedAt(new Date())
            .expiration(Date.from(Instant.now().plusSeconds(300)))
            .signWith(SignatureAlgorithm.HS256, jwtSecret)
            .compact();
    }

    public record ComparisonResult(String resultUrl, boolean hasChanges, int changesCount) {}
}
```

## 注意事项

- **文件可访问性**：对比 API 要求原始文档和修改版文档的 URL 能被 Document Server 访问（不能是私有内网地址）
- **JWT 安全**：所有 OnlyOffice API 请求必须携带有效的 JWT Token，有效期建议 5 分钟内
- **大文件对比**：超过 50MB 的文档对比建议用异步模式（`async: true`），轮询结果
- **格式限制**：仅支持 `docx` 和 `odt` 格式对比，不支持 `xlsx`/`pptx`
- **内存消耗**：对比操作占用较多服务器内存，建议部署单独的 Document Builder 实例用于批量对比
- **结果缓存**：对比结果文档有效期通常为 24 小时，需要时可再次发起对比请求
