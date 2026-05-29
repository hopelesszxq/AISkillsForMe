---
name: onlyoffice-docbuilder
description: OnlyOffice Document Builder 服务端文档生成：模板引擎、数据填充、批量导出与格式转换
tags: [onlyoffice, docbuilder, document, template, generation, automation]
---

## 概述

OnlyOffice Document Builder 是服务端文档生成引擎，支持从模板动态生成 DOCX/XLSX/PPTX 文档，支持数据填充、格式转换、批量处理和 PDF 导出。适用于合同生成、报表导出、批量制证等场景。

## 1. Document Builder 部署

### Docker 部署

```yaml
# docker-compose.yml
services:
  onlyoffice-document-builder:
    image: onlyoffice/documentbuilder:latest
    container_name: onlyoffice-docbuilder
    ports:
      - "9090:9090"          # HTTP API
    environment:
      - JWT_ENABLED=true
      - JWT_SECRET=your-secret-key
    volumes:
      - ./data:/home/data    # 文档输入输出目录
      - ./fonts:/usr/share/fonts  # 自定义字体
    restart: unless-stopped
```

### 检查是否可用

```bash
# 进入容器
docker exec -it onlyoffice-docbuilder bash

# 测试 Document Builder 命令行
documentbuilder --help

# 查看版本
cat /opt/onlyoffice/documentbuilder/VERSION
```

## 2. 使用 Java 调用 Document Builder

### Maven 依赖

```xml
<!-- OnlyOffice DocBuilder Java API -->
<dependency>
    <groupId>com.onlyoffice</groupId>
    <artifactId>docbuilder-java</artifactId>
    <version>3.0.1</version>
</dependency>
```

### 从模板生成文档

```java
import com.onlyoffice.docbuilder.DocBuilder;
import com.onlyoffice.docbuilder.DocBuilderContext;
import com.onlyoffice.docbuilder.filetypes.FileType;

public class DocumentGenerator {

    private static final String BUILD_PATH = "/home/data";

    public byte[] generateFromTemplate(String templatePath, Map<String, Object> data) throws Exception {
        // 初始化 Document Builder
        DocBuilder builder = new DocBuilder();
        builder.setProperty("document", "{\"filePath\": \"" + templatePath + "\"}");

        // 打开模板文件
        builder.openFile(templatePath);

        // 替换模板中的占位符
        builder.replaceText("{{company_name}}", (String) data.get("companyName"));
        builder.replaceText("{{contract_date}}", (String) data.get("contractDate"));
        builder.replaceText("{{contract_no}}", (String) data.get("contractNo"));

        // 填充表格数据
        String[][] tableData = (String[][]) data.get("tableData");
        int row = 0;
        for (String[] rowData : tableData) {
            int col = 0;
            for (String cellValue : rowData) {
                builder.replaceText("{{table:" + row + ":" + col + "}}", cellValue);
                col++;
            }
            row++;
        }

        // 设置条件段落（仅当金额大于 10000 时显示）
        double amount = (double) data.get("amount");
        if (amount > 10000) {
            builder.removeText("{{hide_if_small}}");
        } else {
            builder.replaceText("{{hide_if_small}}", "小额合同，自动审批");
        }

        // 保存为临时文件
        String outputPath = BUILD_PATH + "/output_" + System.currentTimeMillis() + ".docx";
        builder.saveFile(FileType.DOCX, outputPath);
        builder.closeFile();

        // 读取生成的文档
        return Files.readAllBytes(Paths.get(outputPath));
    }
}
```

### 替换图片占位符

```java
// 模板中插入图片占位符 {{image:logo}}
builder.replaceText("{{image:logo}}", "");
builder.insertImage("{{image:logo}}", "/home/data/company_logo.png", 120, 40);
```

## 3. 格式转换

Document Builder 支持在不同格式之间转换。

```java
// DOCX → PDF
FileConverter converter = new FileConverter();
converter.convert(
    new File("/home/data/input.docx"),
    new File("/home/data/output.pdf"),
    FileType.PDF
);

// DOCX → HTML（用于在线预览）
converter.convert(
    new File("/home/data/report.docx"),
    new File("/home/data/report.html"),
    FileType.HTML
);

// XLSX → CSV
converter.convert(
    new File("/home/data/data.xlsx"),
    new File("/home/data/export.csv"),
    FileType.CSV
);
```

## 4. Python 方式调用

```python
import subprocess
import json

class DocBuilderAPI:
    """通过 subprocess 调用 documentbuilder CLI"""

    def __init__(self):
        self.builder_path = "/opt/onlyoffice/documentbuilder/documentbuilder"

    def generate(self, template_path, output_path, data):
        """生成文档"""
        # 创建 builder 脚本
        script = f'''
builder.OpenFile("{template_path}")
builder.ReplaceText("{{name}}", "{data['name']}")
builder.ReplaceText("{{date}}", "{data['date']}")
builder.SaveFile("output", "{output_path}")
builder.CloseFile()
'''
        script_path = "/tmp/build_script.js"
        with open(script_path, "w") as f:
            f.write(script)

        # 执行
        result = subprocess.run(
            [self.builder_path, script_path],
            capture_output=True, text=True, timeout=60
        )
        return result.returncode == 0
```

## 5. 模板制作要点

### Word 模板（docx）

```
合同编号：{{contract_no}}

甲方：{{party_a}}
乙方：{{party_b}}

签约日期：{{sign_date}}

第一条 合同标的
{{description}}

第二条 付款条款
合同金额：¥{{amount}}
付款方式：{{payment_terms}}

第三条 违约责任
{{penalty_clause}}

{% if need_witness %}
见证方：{{witness}}
{% endif %}
```

### Excel 模板（xlsx）

| 序号 | 姓名 | 部门 | 金额 | 备注 |
|------|------|------|------|------|
| {{row:0:0}} | {{row:0:1}} | {{row:0:2}} | {{row:0:3}} | {{row:0:4}} |
| {{row:1:0}} | {{row:1:1}} | {{row:1:2}} | {{row:1:3}} | {{row:1:4}} |

### 模板设计最佳实践

1. **占位符命名规范**：`{{prefix:name}}` 方式，便于分类替换
2. **表格占位符**：`{{row:索引:列索引}}` 或 `{{col:列名:行索引}}`
3. **避免 Word 自动格式化**：占位符内容不要使用自动编号列表
4. **保持模板布局**：占位符占位长度应相近，避免替换后布局错乱
5. **测试覆盖**：对每个占位符都进行 null/empty 边界测试

## 6. 批量生成优化

```java
public class BatchGenerator {

    private final ExecutorService executor = Executors.newFixedThreadPool(4);

    public List<byte[]> batchGenerate(List<DocRequest> requests) {
        List<Future<byte[]>> futures = requests.stream()
            .map(req -> executor.submit(() -> generateOne(req)))
            .toList();

        return futures.stream().map(f -> {
            try { return f.get(120, TimeUnit.SECONDS); }
            catch (Exception e) { throw new RuntimeException(e); }
        }).toList();
    }

    private byte[] generateOne(DocRequest req) {
        // 每个线程使用独立的 builder 实例
        DocBuilder builder = new DocBuilder();
        // ... 生成逻辑
        return result;
    }
}
```

## 7. API 模式（RESTful）

Document Builder 7.0+ 支持 HTTP API 调用。

```bash
# 生成文档
curl -X POST http://localhost:9090/api/v1/generate \
  -H "Authorization: Bearer YOUR_JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "template": "/home/data/contract_template.docx",
    "output": {
      "format": "pdf",
      "path": "/home/data/output/contract_123.pdf"
    },
    "replace": {
      "{{company_name}}": "ABC科技有限公司",
      "{{contract_date}}": "2026-05-29",
      "{{amount}}": "150000.00"
    }
  }'
```

```bash
# 格式转换 API
curl -X POST http://localhost:9090/api/v1/convert \
  -H "Authorization: Bearer YOUR_JWT" \
  -F "file=@report.docx" \
  -F "outputFormat=pdf"
```

## 注意事项

- **字体问题**：生成的 PDF 中文字符需安装中文字体（SimSun、Microsoft YaHei 等）
- **JWT 安全**：生产环境必须启用 JWT 认证，防止 API 被滥用
- **资源限制**：批量生成时注意内存占用，建议单线程 + 队列方式处理
- **模板版本管理**：模板文件应纳入 Git 管理，适配不同版的 Document Builder
- **复杂模板**：过于复杂的格式化（嵌套表格、合并单元格）可能需要额外调试
- **Document Builder 与 Document Server 的区别**：Builder 是服务端生成引擎，不提供在线编辑功能。需在线编辑则使用 Document Server
- **内存泄漏**：每个 DocBuilder 实例用完必须 `closeFile()` 或 `dispose()`，避免内存泄漏
