---
name: onlyoffice-conversion-api
description: OnlyOffice Document Converter REST API 实战：文档格式转换、批量转换、转换回调与高级参数调优
tags: [onlyoffice, conversion, document, api, format, pdf, docx]
---

## 概述

OnlyOffice Document Server 内置了独立的 **文档转换 API**，支持 50+ 格式之间的互相转换（DOCX → PDF、XLSX → CSV、PPTX → PDF、HTML → DOCX 等）。与 Document Builder 不同，转换 API 是轻量级的 REST 接口，无需模板引擎，适合文档格式转换场景。

## 1. 转换 API 基础

### 端点

```
POST /converter
```

### 请求格式

```json
{
  "async": false,
  "filetype": "docx",
  "outputtype": "pdf",
  "title": "report.docx",
  "url": "http://your-app/files/report.docx",
  "key": "unique-conversion-key-123",
  "token": "JWT-token"
}
```

### 请求参数说明

| 参数 | 必填 | 类型 | 说明 |
|------|------|------|------|
| `async` | 否 | bool | 是否异步转换（默认 false） |
| `filetype` | 是 | string | 源文件扩展名（不含点） |
| `outputtype` | 是 | string | 目标文件扩展名 |
| `title` | 是 | string | 文件名（含扩展名） |
| `url` | 是 | string | 可公开访问的文件下载地址 |
| `key` | 是 | string | 唯一转换标识（避免重复转换） |
| `token` | 否 | string | JWT 令牌（JWT 开启时必填） |

### 同步转换响应

```json
{
  "endConvert": true,
  "fileUrl": "http://docserver/cache/convert_abc123.pdf",
  "fileType": "pdf",
  "percent": 100,
  "fileSize": 245760,
  "key": "unique-conversion-key-123"
}
```

### Java 实现

```java
@Component
public class OnlyOfficeConverter {

    private final RestTemplate restTemplate;
    private final String converterUrl;
    private final String jwtSecret;

    public ConversionResult convert(String fileUrl, String fileType, String outputType, String title) {
        // 构建请求体
        Map<String, Object> body = new HashMap<>();
        body.put("async", false);
        body.put("filetype", fileType);
        body.put("outputtype", outputType);
        body.put("title", title);
        body.put("url", fileUrl);
        body.put("key", UUID.randomUUID().toString());

        // JWT 签名
        if (StringUtils.isNotBlank(jwtSecret)) {
            String token = Jwts.builder()
                .setPayload(JsonUtils.toJson(body))
                .signWith(SignatureAlgorithm.HS256, jwtSecret.getBytes(StandardCharsets.UTF_8))
                .compact();
            body.put("token", token);
        }

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        HttpEntity<Map<String, Object>> entity = new HttpEntity<>(body, headers);

        ResponseEntity<Map> response = restTemplate.postForEntity(
            converterUrl, entity, Map.class);

        Map<String, Object> result = response.getBody();
        return ConversionResult.builder()
            .fileUrl((String) result.get("fileUrl"))
            .fileType((String) result.get("fileType"))
            .fileSize((Integer) result.get("fileSize"))
            .success((Boolean) result.get("endConvert"))
            .build();
    }
}
```

## 2. 异步转换

### 适用场景

- 大文件转换（>10MB）
- 批量转换（几十个文件同时）
- 耗时格式转换（如 PDF/A 转 Word）

### 请求

```json
{
  "async": true,
  "filetype": "pptx",
  "outputtype": "pdf",
  "title": "presentation.pptx",
  "url": "http://your-app/files/pptx/presentation.pptx",
  "key": "async-conv-456"
}
```

### 查询转换进度

```
GET /converter?key=async-conv-456
```

```json
{
  "endConvert": false,
  "percent": 62,
  "key": "async-conv-456"
}
```

### 轮询实现

```java
public ConversionResult convertAsync(String fileUrl, String fileType, String outputType, String title) {
    String key = UUID.randomUUID().toString();

    // 发起异步转换
    Map<String, Object> body = new HashMap<>();
    body.put("async", true);
    body.put("filetype", fileType);
    body.put("outputtype", outputType);
    body.put("title", title);
    body.put("url", fileUrl);
    body.put("key", key);
    // ... 发送请求

    // 轮询进度（最多 60 秒）
    for (int i = 0; i < 60; i++) {
        Thread.sleep(1000);
        Map<String, Object> status = queryConversionStatus(key);
        boolean end = (Boolean) status.getOrDefault("endConvert", false);
        if (end) {
            return buildResult(status);
        }
    }
    throw new TimeoutException("Conversion timeout for: " + key);
}
```

## 3. 支持格式速查

### 文档类

| 源格式 | 可转换目标 |
|--------|------------|
| DOCX, DOC, ODT, RTF, TXT | PDF, PDF/A, DOCX, ODT, RTF, TXT, HTML, EPUB, FB2 |
| HTML, MHT | DOCX, ODT, PDF, TXT |

### 表格类

| 源格式 | 可转换目标 |
|--------|------------|
| XLSX, XLS, ODS, CSV | PDF, PDF/A, XLSX, ODS, CSV, HTML |
| CSV | XLSX, ODS, PDF |

### 演示类

| 源格式 | 可转换目标 |
|--------|------------|
| PPTX, PPT, ODP | PDF, PDF/A, PPTX, ODP, PNG, JPG |
| PPTX | GIF (逐帧导出为动画) |

## 4. 安全配置

### 内网转换（推荐）

```
┌──────────┐     HTTP (内网)     ┌────────────────┐
│  App     │ ──────────────────→ │  OnlyOffice     │
│  Server  │ ←────────────────── │  Document       │
└──────────┘                     │  Server         │
     │                           └────────────────┘
     │ HTTP (文件提供)
     ▼
┌──────────┐
│  OSS     │  对象存储/MinIO
└──────────┘
```

### 文件访问限制

```json
// App Server 提供文件的临时签名 URL
{
  "url": "http://app-server/api/files/preview?fileId=123&token=signed-token",
  "filetype": "docx",
  "outputtype": "pdf"
}
```

```java
@GetMapping("/api/files/preview")
public ResponseEntity<Resource> getFileForConversion(
        @RequestParam String fileId,
        @RequestParam String token) {
    // 验证 token（限制只有 OnlyOffice 服务器的 IP 可调用）
    String onlyOfficeIp = request.getRemoteAddr();
    if (!allowedIps.contains(onlyOfficeIp)) {
        return ResponseEntity.status(403).build();
    }
    // 验证 token 时效
    if (!jwtService.validateToken(token)) {
        return ResponseEntity.status(403).build();
    }
    // 返回文件流
    File file = fileService.getFileById(fileId);
    return ResponseEntity.ok()
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .body(new FileSystemResource(file));
}
```

### 转换 API JWT 配置

```properties
# onlyoffice-documentserver 配置文件 (/etc/onlyoffice/documentserver/local.json)
{
  "services": {
    "CoAuthoring": {
      "token": {
        "enable": {
          "request": {
            "inbox": true,
            "outbox": true
          }
        },
        "secret": "your-jwt-secret",
        "algorithm": "HS256"
      }
    }
  }
}
```

## 5. 批量转换实战

```java
@Service
public class BatchConversionService {

    public List<ConversionResult> batchConvertToPdf(List<String> fileUrls) {
        ExecutorService executor = Executors.newFixedThreadPool(5); // 限制并发
        List<Future<ConversionResult>> futures = new ArrayList<>();

        for (String url : fileUrls) {
            futures.add(executor.submit(() -> convertSingle(url)));
        }

        List<ConversionResult> results = new ArrayList<>();
        for (Future<ConversionResult> future : futures) {
            try {
                results.add(future.get(120, TimeUnit.SECONDS));
            } catch (Exception e) {
                results.add(ConversionResult.failed(e.getMessage()));
            }
        }
        executor.shutdown();
        return results;
    }
}
```

## 6. 常见问题排查

| 问题 | 原因 | 解决 |
|------|------|------|
| 返回 `{"error": -5}` | URL 不可访问 | 确保 Document Server 可访问文件 URL |
| 返回 `{"error": -4}` | 格式不支持 | 检查 input/output 格式是否在支持列表中 |
| 返回 `{"error": -20}` | JWT 校验失败 | 检查 token 签名算法和 secret 是否一致 |
| 转换中文乱码 | 缺少中文字体 | 挂载字体目录到容器：`-v /usr/share/fonts:/usr/share/fonts` |
| 异步转换一直 0% | 大文件+同步写 | 检查磁盘 IO，使用暂存目录 |
| `percent: 100` 但无文件 | 存储空间不足 | 检查 `/var/www/onlyoffice/Data/cache` 磁盘空间 |

## 注意事项

- 转换 API 要求 Document Server 能 **直接访问** 源文件 URL，不能用 localhost（容器内 localhost 指向自己）
- 高并发下建议限制请求速率（每秒最多 5-10 个转换请求，视服务器配置而定）
- 转换后的文件会缓存一定时间（默认生周期与编辑文档缓存一致），建议转换后立即下载到自己的存储
- PDF/A 转换比普通 PDF 慢约 2-3 倍，且需要额外字体嵌入
- 对于 Office 2003 格式（.doc/.xls/.ppt），转换前需要安装对应兼容包
