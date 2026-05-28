---
name: onlyoffice-collaboration
description: OnlyOffice 协同编辑进阶：版本历史、差异对比、插件开发与 8.x 新特性
tags: [onlyoffice, office, document, collaboration, plugin]
---

## 版本历史与差异对比

OnlyOffice Document Server 支持完整的版本历史管理和文档差异对比。

### 请求版本历史

```javascript
// 前端配置：启用历史记录功能
const config = {
  document: {
    key: docKey,
    url: docUrl,
  },
  editorConfig: {
    customization: {
      showReviewChanges: true,     // 显示修订
      review: {
        reviewDisplay: 'original', // original / current / markup
        showReviewChanges: true,
      },
    },
  },
  events: {
    onRequestHistory: () => {
      // 请求历史记录数据（触发后端返回历史数据）
      fetch(`/api/documents/${docId}/history`)
        .then(res => res.json())
        .then(data => docEditor.setHistory(data));
    },
    onRequestHistoryClose: () => {
      docEditor.history.close();
    },
    onRequestHistoryData: (event) => {
      // 请求特定版本的数据
      const version = event.data;
      fetch(`/api/documents/${docId}/history/${version}/data`)
        .then(res => res.json())
        .then(data => docEditor.setHistoryData(data));
    },
  },
};
```

### 后端版本管理接口

```java
@RestController
@RequestMapping("/api/documents/{docId}")
@Slf4j
public class DocumentHistoryController {

    @Autowired
    private DocumentService documentService;

    /**
     * 获取历史版本列表
     */
    @GetMapping("/history")
    public ResponseEntity<Map<String, Object>> getHistory(@PathVariable Long docId) {
        List<VersionInfo> versions = documentService.getVersionHistory(docId);

        Map<String, Object> response = new HashMap<>();
        response.put("currentVersion", versions.size());
        response.put("history", versions.stream().map(v -> {
            Map<String, Object> item = new HashMap<>();
            item.put("key", v.getVersionKey());
            item.put("version", v.getVersionNumber());
            item.put("created", v.getCreatedAt().toString());
            item.put("user", Map.of(
                "id", v.getUserId(),
                "name", v.getUserName()
            ));
            return item;
        }).collect(Collectors.toList()));

        return ResponseEntity.ok(response);
    }

    /**
     * 获取特定版本的文档 URL
     */
    @GetMapping("/history/{version}/data")
    public ResponseEntity<Map<String, Object>> getHistoryData(
            @PathVariable Long docId,
            @PathVariable int version) {

        String versionUrl = documentService.getVersionDownloadUrl(docId, version);

        // OnlyOffice 需要能够访问此 URL
        Map<String, Object> response = new HashMap<>();
        response.put("fileType", "docx");
        response.put("url", versionUrl);
        response.put("key", docId + "_v" + version);
        response.put("version", version);

        // 如果启用了 JWT，需要对返回的 URL 生成 token
        String token = generateToken(response);
        response.put("token", token);

        return ResponseEntity.ok(response);
    }

    /**
     * 文档差异对比（查看两个版本之间的变更）
     */
    @GetMapping("/diff")
    public ResponseEntity<Map<String, Object>> getDiff(
            @PathVariable Long docId,
            @RequestParam int fromVersion,
            @RequestParam int toVersion) {

        String diffUrl = documentService.getDiffUrl(docId, fromVersion, toVersion);

        Map<String, Object> response = new HashMap<>();
        response.put("fileType", "docx");
        response.put("url", diffUrl);
        response.put("key", docId + "_diff_" + fromVersion + "_" + toVersion);

        return ResponseEntity.ok(response);
    }
}
```

### 文档保存时生成新版本

```java
@Service
public class DocumentService {

    /**
     * 保存文档时创建版本快照
     */
    @Transactional
    public void saveDocument(Long docId, byte[] content, String userId, String userName) {
        // 1. 保存当前文档内容
        Document doc = documentMapper.selectById(docId);
        String storagePath = storeFile(docId, content);
        doc.setStoragePath(storagePath);
        doc.setUpdatedAt(LocalDateTime.now());
        documentMapper.updateById(doc);

        // 2. 创建版本记录
        DocumentVersion version = new DocumentVersion();
        version.setDocId(docId);
        version.setVersionNumber(doc.getCurrentVersion() + 1);
        version.setStoragePath(storagePath);
        version.setFileSize(content.length);
        version.setUserId(userId);
        version.setUserName(userName);
        version.setCreatedAt(LocalDateTime.now());
        documentVersionMapper.insert(version);

        // 3. 更新文档版本号
        doc.setCurrentVersion(version.getVersionNumber());
        documentMapper.updateById(doc);
    }
}
```

## OnlyOffice 8.x 新特性

OnlyOffice Document Server 8.x 带来了多项重要更新。

### 共享编辑性能优化（8.0+）

```yaml
# docker-compose 配置优化
services:
  onlyoffice-document-server:
    image: onlyoffice/documentserver:8.2
    environment:
      # 协同编辑性能调优
      - COEDITING_MODE=fast       # fast（快速模式，推荐）/ strict（严格模式）
      - COEDITING_TIMEOUT=60      # 协同会话超时秒数

      # 新：文档预览缩放性能
      - USE_DOCUMENT_PREVIEW_CACHE=true

      # 新：内存优化
      - JX_BROWSER_MEMORY_LIMIT=512m
```

### 新增表单填写功能

OnlyOffice 8.1+ 支持创建和填写表单（PDF 表单、OFORM 格式）：

```javascript
// 前端配置中启用表单
const config = {
  document: {
    fileType: "docxf",  // 表单模板格式
  },
  editorConfig: {
    mode: "edit",        // edit / fill（仅填写）/ review（审阅）
    customization: {
      features: {
        roles: true,     // 启用表单角色分配
      },
    },
  },
};
```

### 对比模式（Changes 增强）

```javascript
// 以对比模式打开文档，查看两个文档的差异
const docEditor = new DocsAPI.DocEditor("placeholder", {
  document: {
    fileType: "docx",
    key: "diff_" + Date.now(),
    url: originalDocUrl,
    // 对比文档
    info: {
      comparing: {
        type: "previous",  // previous / external
        url: changedDocUrl,
      },
    },
  },
});
```

## 插件开发

OnlyOffice 支持开发自定义插件扩展功能。

### 插件目录结构

```
plugins/
  example-plugin/
    index.html          # 插件入口（HTML + JS）
    pluginCode.js       # 插件逻辑
    pluginConfig.json   # 配置
    icon.png            # 图标
```

```json
// pluginConfig.json
{
  "name": "AI 辅助写作",
  "guid": "com.example.ai-writer",
  "baseUrl": "https://your-plugin-server.com",
  "variations": [
    {
      "description": "AI 辅助写作插件",
      "url": "index.html",
      "icons": ["icon.png"],
      "isViewer": false,
      "EditorsSupport": ["word", "cell", "slide"],
      "isVisual": true,
      "buttons": [
        {
          "name": "AI 写作",
          "action": "onClick",
          "primary": true
        }
      ],
      "size": [600, 400]
    }
  ]
}
```

```javascript
// index.html — 插件界面
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <script src="/plugin-dev/scripts/asc.offline.pako.js"></script>
</head>
<body>
  <div>
    <textarea id="prompt" placeholder="输入写作提示..."></textarea>
    <button onclick="generateText()">生成</button>
    <div id="result"></div>
  </div>

  <script>
    async function generateText() {
      const prompt = document.getElementById('prompt').value;
      const response = await fetch('https://api.example.com/generate', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ prompt })
      });
      const data = await response.json();

      // 将生成的内容插入文档
      if (docEditor && data.text) {
        docEditor.service.InsertText(data.text);
      }
    }

    // 插件初始化
    window.Asc.plugin.init = function () {
      // 插件加载完成后的回调
    };
  </script>
</body>
</html>
```

### 插件部署

```bash
# 方式一：挂载到 Document Server
docker cp ./plugins/ onlyoffice-ds:/var/www/onlyoffice/documentserver/sdkjs-plugins/

# 方式二：在 default.json 中配置
```

```json
// /etc/onlyoffice/documentserver/default.json
{
  "services": {
    "CoAuthoring": {
      "plugins": {
        "plugins": [
          "https://your-plugin-server.com/example-plugin/pluginConfig.json"
        ]
      }
    }
  }
}
```

## 注意事项

1. **版本历史存储**：每次保存都生成一份完整文档副本，建议定期归档旧版本（保留最近 N 个版本 + 每月快照）
2. **差异对比性能**：大文档（>50MB）的差异对比消耗较大，建议限制版本差异的跨度
3. **协同编辑模式**：
   - fast 模式：高实时性，适合团队协作（推荐）
   - strict 模式：严格一致性，适合需要精确控制版本场景
4. **插件安全**：插件在 Document Server 沙箱中运行，但对外部 API 请求需注意鉴权和 CORS
5. **版本 key 的使用**：每次保存后更新文档 key，否则 OnlyOffice 可能使用缓存版本
6. **字体兼容性**：插件中插入特殊字符（如数学符号）时，确保 Document Server 已安装对应字体
7. **并发保存冲突**：forcesave=true 时可能短时间内多次触发保存，建议业务层做去重（按文档 key + 时间戳）
