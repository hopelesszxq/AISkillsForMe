---
name: onlyoffice-plugins-development
description: OnlyOffice 插件开发完整指南：manifest 配置、原生 API、事件模型、UI 组件、Vue.js 集成与部署
tags: [onlyoffice, plugin, extension, javascript, api, custom-toolbar, deployment]
---

## 概述

OnlyOffice 插件系统允许在文档编辑器中扩展自定义功能，如自定义工具栏按钮、侧边面板、右键菜单、自动完成任务等。插件本质上是运行在编辑器内的 JavaScript 应用，可以操作文档内容、响应事件、调用外部 API。

### 架构示意图

```
┌─────────────────────────────────────────────┐
│              OnlyOffice Editor               │
│  ┌──────────────────────────────────────┐   │
│  │        Plugin Runtime (JS)           │   │
│  │  ┌─────────┐  ┌──────────────────┐  │   │
│  │  │ Toolbar │  │  Side Panel      │  │   │
│  │  │  Button │  │  (HTML/CSS/JS)   │  │   │
│  │  └─────────┘  └──────────────────┘  │   │
│  │  ┌─────────┐  ┌──────────────────┐  │   │
│  │  │  Macro  │  │  Context Menu    │  │   │
│  │  └─────────┘  └──────────────────┘  │   │
│  └──────────────────────────────────────┘   │
│                                              │
│  Plugin API: window.Asc.plugin.*             │
└─────────────────────────────────────────────┘
```

## 一、插件目录结构

```
my-plugin/
├── pluginConfig.json      # 插件清单（必需）
├── index.html             # 插件主入口（必需）
├── pluginCode.js          # 插件逻辑
├── style.css              # 样式
├── assets/                # 资源文件
│   ├── icon.png
│   └── logo.svg
└── translations/          # 国际化
    ├── en.json
    └── zh.json
```

## 二、pluginConfig.json 配置

```json
{
  "name": "Translation Helper",
  "guid": "com.example.translation-helper",
  "baseUrl": "https://plugins.example.com/translation-helper",
  "version": "1.0.0",
  "minVersion": "7.5.0",
  "description": "选中文本后翻译为指定语言",
  "descriptionLocale": {
    "zh": "选中文本后翻译为目标语言"
  },
  "icon": "assets/icon.png",
  "size": [400, 300],
  "windowType": "panel",
  "variant": "all",
  "variations": [
    {
      "name": "Main",
      "url": "index.html",
      "buttons": [
        {
          "text": "翻译",
          "primary": true,
          "isDefault": true
        },
        {
          "text": "取消",
          "primary": false,
          "isDefault": false
        }
      ],
      "icons": {
        "menu": "assets/icon.svg"
      },
      "description": "翻译助手",
      "EditorsSupport": ["word", "cell", "slide"],
      "initData": {
        "targetLang": "zh-CN"
      },
      "store": {
        "background": {
          "light": "#f0f0f0",
          "dark": "#333333"
        }
      },
      "isViewer": false,
      "EditorsStartMode": ["edit"]
    }
  ]
}
```

### 关键字段说明

| 字段 | 说明 | 示例值 |
|------|------|--------|
| `guid` | 全局唯一标识符 | `com.company.plugin-name` |
| `baseUrl` | 插件资源加载的基础 URL | `https://plugins.example.com/my-plugin` |
| `windowType` | 窗口类型 | `panel`（侧面板）/ `modal`（模态框） |
| `variant` | 插件变体 | `all` / `desktop` / `mobile` |
| `EditorsSupport` | 支持的编辑器 | `word`（文档）`cell`（表格）`slide`（演示） |
| `size` | 面板尺寸 `[width, height]` | `[400, 300]` |
| `buttons` | 底部按钮配置 | `[{text: "确定", primary: true}]` |

## 三、插件 API 核心用法

### 1. 初始化与回调

```javascript
// pluginCode.js
(function () {
    "use strict";

    window.Asc.plugin.init = function (text) {
        // 插件加载完成时调用
        // text: 选中文本（如果编辑器支持）
        console.log("Plugin initialized", text);
        document.getElementById("input-text").value = text || "";
    };

    // 按钮点击回调（对应 pluginConfig.json 中的 buttons）
    window.Asc.plugin.button = function (id) {
        if (id === 0) {  // "翻译" 按钮被点击
            var text = document.getElementById("input-text").value;
            performTranslation(text);
        } else {          // "取消"
            window.Asc.plugin.executeCommand("close", "");
        }
    };

    // 窗口关闭时清理
    window.Asc.plugin.onExternalMouseUp = function () {
        // 点击外部区域时触发
    };
})();
```

### 2. 文档操作 API

```javascript
// 获取当前选中的文本
window.Asc.plugin.executeMethod("GetSelectedText", [], function(result) {
    console.log("Selected text:", result);
});

// 插入文本（在光标位置）
window.Asc.plugin.executeMethod("PasteText", ["Hello from plugin!"]);

// 获取文档内容
window.Asc.plugin.executeMethod("GetAllContent", [], function(content) {
    console.log("Document content:", content);
});

// 设置单元格值（表格专用）
window.Asc.plugin.executeMethod("SetCellValue", ["A1", "Hello"]);

// 插入图片
window.Asc.plugin.executeMethod("PutImage", [
    "data:image/png;base64,iVBORw0KGgo...",
    80,  // width in mm
    60   // height in mm
]);
```

### 3. 命令执行与通信

```javascript
// 向编辑器发命令
window.Asc.plugin.executeCommand("close", "");     // 关闭面板
window.Asc.plugin.executeCommand("showButton", 0); // 显示底部按钮

// 自定义事件监听
window.Asc.plugin.attachEvent("onChangeSelection",
    function(isEmpty) {
        document.getElementById("translate-btn").disabled = isEmpty;
    }
);
```

### 4. 信息获取

```javascript
// 获取文档信息
window.Asc.plugin.info = {
    documentType: "word" | "cell" | "slide",
    documentUrl: "https://...",
    userName: "John",
    userGuid: "user-uuid",
    coauthoringType: "coauthoring" | "viewing",
    filesTypeUrl: "https://..."
};

// 运行时配置获取
var config = window.Asc.plugin.getConfig();
console.log("Plugin config:", config);
```

## 四、插件 UI 最佳实践

### 1. 侧面板 HTML 模板

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <link rel="stylesheet" href="style.css">
    <script src="https://onlyofficeninja.com/plugin-dev/scripts/jquery.js"></script>
</head>
<body>
    <div class="plugin-panel">
        <div class="header">
            <h3>翻译助手</h3>
        </div>
        <div class="content">
            <label>原文</label>
            <textarea id="source-text" rows="5"></textarea>
            <label>目标语言</label>
            <select id="target-lang">
                <option value="zh-CN">中文</option>
                <option value="en">English</option>
                <option value="ja">日本語</option>
            </select>
            <button id="translate-btn" disabled>翻译并插入</button>
            <div id="result" class="result-box"></div>
        </div>
    </div>
    <script src="pluginCode.js"></script>
</body>
</html>
```

### 2. CSS 适配 Dark/Light 主题

```css
/* 响应编辑器主题 */
.plugin-panel {
    padding: 12px;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
    font-size: 12px;
    color: #333;
    background: #fff;
}

/* 暗色主题适配 */
body[data-theme="dark"] .plugin-panel {
    color: #e0e0e0;
    background: #1e1e1e;
}

body[data-theme="dark"] textarea {
    background: #2d2d2d;
    color: #e0e0e0;
    border-color: #444;
}

/* 响应式面板 */
.content {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

textarea {
    width: 100%;
    padding: 6px;
    border: 1px solid #ccc;
    border-radius: 4px;
    resize: vertical;
    box-sizing: border-box;
}

button {
    padding: 8px 16px;
    background: #0072c6;
    color: #fff;
    border: none;
    border-radius: 4px;
    cursor: pointer;
}

button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}
```

## 五、部署方式

### 1. 本地部署

```bash
# 复制插件到 Document Server 的 plugins 目录
cp -r ./my-plugin /var/www/onlyoffice/documentserver/sdkjs-plugins/

# 重启 Document Server
supervisorctl restart onlyoffice-documentserver
```

### 2. 远程部署（推荐）

```bash
# 部署插件静态资源到独立的 Web 服务器（Nginx）
# 配置示例
server {
    listen 443 ssl;
    server_name plugins.example.com;

    root /var/www/plugins;
    
    location / {
        add_header Access-Control-Allow-Origin "*";
    }
}
```

### 3. 配置 Editors 加载远程插件

```json
// 在前端初始化参数中指定
{
  "document": { "fileType": "docx", "url": "..." },
  "editorConfig": {
    "plugins": {
      "autostart": ["com.example.translation-helper"],
      "pluginsData": [
        "https://plugins.example.com/translation-helper/pluginConfig.json"
      ]
    }
  }
}
```

## 六、Vue.js 集成示例

```vue
<template>
  <div class="plugin-app" :class="{ dark: isDark }">
    <div class="toolbar">
      <button @click="translate" :disabled="!selectedText">
        🌐 翻译
      </button>
    </div>
    <div v-if="result" class="result">
      <p>{{ result }}</p>
      <button @click="insertResult">插入文档</button>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      selectedText: '',
      result: '',
      isDark: false
    }
  },
  mounted() {
    // 初始化插件
    window.Asc.plugin.init = (text) => {
      this.selectedText = text || ''
    }

    // 监听主题变化
    window.Asc.plugin.attachEvent('onThemeChange', (theme) => {
      this.isDark = theme.name === 'Dark'
    })
  },
  methods: {
    async translate() {
      const response = await fetch('/api/translate', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ text: this.selectedText })
      })
      const data = await response.json()
      this.result = data.translation
    },
    insertResult() {
      window.Asc.plugin.executeMethod('PasteText', [this.result])
    }
  }
}
</script>
```

## 七、调试与日志

```javascript
// 插件中打印日志（在浏览器开发者工具中查看）
window.Asc.plugin.info("Plugin started");
window.Asc.plugin.error("Something went wrong");
window.Asc.plugin.debug("Selected text:", text);

// 浏览器开发者工具
// Chromium 内核编辑器：F12 打开 DevTools
// 远程调试：在启动命令中添加 --remote-debugging-port=9222
```

### 常见问题排查

| 现象 | 可能原因 | 解决方案 |
|------|---------|---------|
| 插件不加载 | pluginConfig.json 格式错误 | 用 JSON 验证器检查 |
| 按钮不显示 | `EditorsSupport` 不匹配 | 检查是否包含 word/cell/slide |
| API 调用失败 | 版本不兼容 | 检查 `minVersion` 配置 |
| CORS 错误 | 远程插件未设置 CORS 头 | Nginx 添加 `Access-Control-Allow-Origin: *` |
| 安全策略拦截 | JWT 配置了 Content-Security-Policy | 在 CSP 中添加 `script-src` |

## 八、注意事项

1. **插件 GUID 唯一性**：所有插件 GUID 必须全局唯一，推荐格式 `com.company.plugin-name`。冲突会导致插件加载失败。
2. **HTTPS 要求**：OnlyOffice 7.0+ 默认只加载 HTTPS 来源的远程插件。开发环境可使用 `http://localhost`。
3. **跨编辑器兼容**：部分 API（如 `SetCellValue`）只在特定编辑器中有效。使用 `window.Asc.plugin.info.documentType` 判断后差异化处理。
4. **性能影响**：插件的 JS 代码运行在编辑器的主线程中，避免执行耗时同步操作（推荐使用 Web Worker 或异步 API）。
5. **内存泄漏**：在 `init` 方法中注册的事件和定时器，需在 `onDestroy` 或推出时清理。
6. **安全加固**：不要在插件中直接拼接用户输入到 `innerHTML`，使用 `textContent` 或模板引擎避免 XSS。
7. **升级兼容**：升级 Document Server 后需要验证所有插件是否兼容——插件 API 可能在大版本中废弃或变更。
