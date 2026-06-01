---
name: onlyoffice-macros
description: OnlyOffice 宏（Macros）开发指南：从基础到高级，自动化文档处理与脚本编写
tags: [onlyoffice, macros, automation, javascript, document]
---

## 概述

OnlyOffice 宏（Macros）是运行在文档编辑器中的 JavaScript 脚本，可以**自动操作文档内容、格式、结构**。与插件（Plugins）不同，宏无需外部服务器，直接在浏览器端执行，适合轻量级自动化任务。

## 宏的基础结构

```javascript
(function() {
    // 获取当前文档的 API 对象
    const doc = Api.GetDocument();
    const para = Api.CreateParagraph();
    para.AddText("Hello, OnlyOffice Macros!");
    doc.InsertContent([para]);
})();
```

### 宏的三种触发方式

| 方式 | 说明 | 适用场景 |
|------|------|---------|
| 手动运行 | 在「插件 → 宏」菜单中手动执行 | 一次性操作 |
| 自动运行 | 文档打开时通过 `onDocumentOpen` 事件触发 | 初始化、校验 |
| 定时运行 | `setInterval` 在宏内启动轮询 | 实时协作辅助 |

## 文档内容操作

### 1. 文本操作

```javascript
(function() {
    const doc = Api.GetDocument();

    // 插入段落
    const para = Api.CreateParagraph();
    para.AddText("这是通过宏插入的文本");
    para.SetBold(true);
    para.SetFontSize(14);
    doc.InsertContent([para]);

    // 替换全部文本
    const search = doc.Search("旧文本");
    search.forEach(item => {
        item.ReplaceElementText("新文本");
    });

    // 获取选中内容
    const sel = doc.GetSelection();
    if (sel.GetText()) {
        sel.AddText(" - 已选中");
    }
})();
```

### 2. 表格操作

```javascript
(function() {
    const doc = Api.GetDocument();

    // 创建 3x4 表格
    const table = Api.CreateTable(3, 4);
    table.SetWidth("percent", 100);

    // 遍历单元格填充数据
    for (let r = 0; r < 3; r++) {
        for (let c = 0; c < 4; c++) {
            const cell = table.GetCell(r, c);
            cell.GetContent().GetElement(0).AddText(`R${r}C${c}`);

            // 表头加粗
            if (r === 0) {
                cell.SetBold(true);
                cell.SetShdColor("4472C4");
                cell.GetContent().GetElement(0).SetColor("FFFFFF");
            }
        }
    }
    doc.InsertContent([table]);
})();
```

### 3. 图片与形状

```javascript
(function() {
    const doc = Api.GetDocument();
    const shape = Api.CreateShape("rect", 100, 50);
    shape.SetPosition(100, 200);
    shape.SetSize(200, 100);
    shape.GetDocContent().GetElement(0).AddText("形状中的文本");
    doc.InsertContent([shape]);
})();
```

## 文档处理自动化

### 批量替换 - 模板填充

```javascript
(function() {
    const doc = Api.GetDocument();

    // 模板变量替换
    const variables = {
        "{{姓名}}": "张三",
        "{{职位}}": "高级工程师",
        "{{日期}}": new Date().toLocaleDateString("zh-CN")
    };

    Object.entries(variables).forEach(([key, value]) => {
        const results = doc.Search(key);
        results.forEach(item => {
            item.ReplaceElementText(value);
        });
    });
})();
```

### 批量导出为 PDF

```javascript
(function() {
    const doc = Api.GetDocument();

    // 检查文档是否包含必填字段
    const requiredFields = ["签名", "日期"];
    const fullText = doc.GetText();

    requiredFields.forEach(field => {
        if (fullText.indexOf(field) === -1) {
            throw new Error(`缺少必填区域: ${field}`);
        }
    });

    // 创建 PDF 下载链接
    doc.Save("pdf", function(result) {
        const link = document.createElement("a");
        link.href = "data:application/pdf;base64," + result;
        link.download = "output.pdf";
        link.click();
    });
})();
```

## 高级宏技巧

### 1. 与插件通信

```javascript
(function() {
    // 宏内调用插件 API
    window.Asc.plugin.executeMethod("GetDocumentRenderUrl", [], function(url) {
        // 处理回调结果
        console.log("Render URL:", url);
    });

    // 向插件发送消息
    window.Asc.plugin.sendToPlugin("customAction", {
        action: "macroCompleted",
        data: { message: "宏执行完成" }
    });
})();
```

### 2. 条件格式与校验

```javascript
(function() {
    const doc = Api.GetDocument();
    const tables = doc.GetTables();

    tables.forEach(table => {
        // 校验数值列
        for (let r = 1; r < table.GetRowsCount(); r++) {
            const cell = table.GetCell(r, 1); // 第2列
            const text = cell.GetContent().GetElement(0).GetText();

            const value = parseFloat(text);
            if (isNaN(value) || value < 0) {
                // 标记异常单元格
                cell.SetShdColor("FF0000");
                cell.GetContent().GetElement(0).SetColor("FFFFFF");
            } else if (value > 10000) {
                cell.SetShdColor("FFFF00");
            }
        }
    });
})();
```

### 3. 批量文档处理

```javascript
(function() {
    // 在当前文档中收集数据
    const doc = Api.GetDocument();
    const data = {};

    // 读取表单字段
    const forms = doc.GetForms();
    forms.forEach(form => {
        data[form.GetFormKey()] = form.GetFormValue();
    });

    // 发送到后端处理
    fetch("/api/documents/process", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(data)
    }).then(res => res.json())
      .then(result => {
          // 根据后端返回更新文档
          if (result.status === "approved") {
              const para = Api.CreateParagraph();
              para.AddText("审批通过: " + new Date().toLocaleString());
              para.SetColor("008000");
              doc.InsertContent([para]);
          }
      });
})();
```

## 宏调试

### 1. 日志输出

```javascript
(function() {
    // 控制台日志（仅在开发者工具中可见）
    console.log("宏开始执行");
    console.warn("潜在问题：字段为空");
    console.error("错误：无法获取文档");

    // 在文档中输出调试信息
    const doc = Api.GetDocument();
    const debugPara = Api.CreateParagraph();
    debugPara.AddText("[DEBUG] " + new Date().toISOString());
    debugPara.SetColor("999999");
    doc.InsertContent([debugPara]);
})();
```

### 2. 错误处理

```javascript
(function() {
    try {
        const doc = Api.GetDocument();
        if (!doc) throw new Error("文档未加载");

        // 业务逻辑
        const table = doc.GetTable(0);
        if (!table) throw new Error("未找到表格");

        // ...
        Api.asc_calc.Execute();
    } catch (e) {
        // 用户友好的错误提示
        const errorPara = Api.CreateParagraph();
        errorPara.AddText("❌ 宏执行失败: " + e.message);
        errorPara.SetColor("FF0000");
        Api.GetDocument().InsertContent([errorPara]);
    }
})();
```

## 宏的存储与分发

### 项目结构

```
macros/
  generate-report.js     # 生成报告
  validate-forms.js      # 表单校验
  batch-export.js        # 批量导出
  template-fill.js       # 模板填充
  README.md
```

### 部署到 OnlyOffice

```bash
# 1. 将宏文件放到文档服务器目录
sudo mkdir -p /var/www/onlyoffice/documentserver/sdkjs-plugins/macros
sudo cp generate-report.js /var/www/onlyoffice/documentserver/sdkjs-plugins/macros/

# 2. 或在 Document Server 配置中注册（仅适用于插件形式的宏管理）
```

### 宏的版本控制

```javascript
// 宏头部添加版本信息
/*
 * @name: 合同模板填充
 * @version: 1.2.0
 * @author: 开发团队
 * @desc: 读取 JSON 数据填充合同模板
 * @date: 2026-06-01
 * @changelog:
 *   - 1.2.0: 支持多语言字段
 *   - 1.1.0: 增加表单校验
 *   - 1.0.0: 初始版本
 */
```

## 注意事项

1. **运行环境限制**：宏运行在浏览器沙箱中，不能直接访问文件系统
2. **异步处理**：`Api` 的大多数操作是同步的，但 `fetch` 调用是异步的，注意处理流程
3. **跨文档协作**：多个用户同时编辑时，宏操作可能产生冲突，建议在操作前检查文档锁定状态
4. **性能开销**：大量表格遍历和格式设置会消耗性能，复杂操作建议分批执行
5. **安全性**：不要执行用户传入的不可信宏代码，避免 XSS 攻击
6. **API 兼容性**：不同版本 OnlyOffice 的宏 API 可能有差异，建议针对目标版本测试
7. **回滚机制**：在修改文档前可先调用 `doc.GetSavedState()` 记录状态，出错时恢复
