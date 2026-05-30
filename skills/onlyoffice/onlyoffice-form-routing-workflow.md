---
name: onlyoffice-form-routing-workflow
description: ONLYOFFICE 9.4 表单路由与工作流：角色分配、填写状态追踪、onStartFilling 事件回调与表单自动化
tags: [onlyoffice, form-routing, workflow, api, document, callback, 94]
---

## ONLYOFFICE 9.4 表单路由

ONLYOFFICE Docs 9.4 引入了**原生表单路由（Form Routing）** 能力，支持角色分配、接收者路由和填写状态追踪。开发者无需手写大量验证逻辑，编辑器原生处理 UI 校验。

## 1. onStartFilling 事件：角色感知表单填充

### 核心 API

`onStartFilling` 事件现在携带 `roles` 参数，包含角色和用户数据。可根据填写者角色路由到不同字段或触发条件逻辑。

```javascript
// Docs API 配置
function onStartFilling(event) {
    const roles = event.data;
    console.log("Roles:", roles);
    
    // 根据角色判断当前用户
    const currentRole = roles.find(r => r.id === currentUser.roleId);
    if (currentRole) {
        // 只显示该角色需要填写的字段
        showFieldsForRole(currentRole);
    }
}

const config = {
    events: {
        onStartFilling: onStartFilling,  // 9.4 新增角色参数
    },
};

const docEditor = new DocsAPI.DocEditor("placeholder", config);
```

### 进阶：角色条件逻辑

```javascript
function onStartFilling(event) {
    const { roles } = event.data;
    
    // 角色定义示例：
    // [ { id: "approver", name: "审批人", fields: ["approve", "comment"] },
    //   { id: "finance", name: "财务", fields: ["amount", "invoice"] },
    //   { id: "hr", name: "人事", fields: ["employee_name", "department"] } ]
    
    // 根据角色禁用/启用字段
    roles.forEach(role => {
        if (role.id === currentUser.role) {
            enableFields(role.fields);
        } else {
            disableFields(role.fields);
        }
    });
    
    // 触发后续操作
    if (isLastRole()) {
        notifyCompletion();
    }
}
```

## 2. 后端集成：表单路由配置

### Java 后端配置

```java
@Configuration
public class OnlyOfficeConfig {
    
    @Bean
    public EditorConfig editorConfig() {
        EditorConfig config = new EditorConfig();
        
        // 启用表单路由
        config.setFormsMode(FormsMode.FILLING);
        
        // 配置回调 URL
        config.setCallbackUrl("https://your-server.com/callback");
        
        return config;
    }
}

// 回调处理
@RestController
@RequestMapping("/callback")
public class DocumentCallbackController {
    
    @PostMapping
    public ResponseEntity<String> handleCallback(@RequestBody CallbackRequest request) {
        switch (request.getStatus()) {
            case 1:  // 正在编辑
                break;
            case 2:  // 准备保存
                handleSave(request);
                break;
            case 3:  // 保存错误
                handleError(request);
                break;
            case 4:  // 填写完成
                handleFillingCompleted(request);
                break;
            case 6:  // 表单已提交
                handleFormSubmitted(request);
                break;
            case 7:  // 表单路由下一步
                handleRoutingNext(request);
                break;
        }
        return ResponseEntity.ok("{\"error\": 0}");
    }
    
    // 表单路由到下一步
    private void handleRoutingNext(CallbackRequest request) {
        String nextRole = request.getForm().getNextRole();
        String documentId = request.getKey();
        
        // 通知下个角色的用户
        notificationService.notifyRole(nextRole, "请填写表单: " + documentId);
        
        // 记录路由历史
        formAuditLog.recordRouting(documentId, request.getUsers(), nextRole);
    }
    
    // 表单提交完成
    private void handleFormSubmitted(CallbackRequest request) {
        // 触发后续业务流程
        workflowService.completeForm(request.getKey());
    }
}
```

## 3. 完整的表单工作流集成

### 前端集成

```html
<!DOCTYPE html>
<html>
<head>
    <script src="https://your-server/web-apps/apps/api/documents/api.js"></script>
</head>
<body>
    <div id="ds-placeholder"></div>
    <script>
        const config = {
            document: {
                fileType: "docxf",
                key: "doc-" + new Date().getTime(),
                title: "Contract Form.docx",
                url: "https://your-server/download/document.docx",
                permissions: {
                    edit: true,
                    fillForms: true,   // 允许填写表单
                    modifyFilter: false
                }
            },
            editorConfig: {
                callbackUrl: "https://your-server/callback",
                lang: "zh-CN",
                customization: {
                    forcesave: true,
                    hideRightMenu: false
                }
            },
            events: {
                onStartFilling: function(event) {
                    // 9.4: roles 参数包含角色分配信息
                    const roles = event.data;
                    updateRoutingUI(roles);
                },
                onFormSubmit: function(event) {
                    console.log("Form submitted:", event.data);
                    // 通知后端工作流前进
                },
                onRequestInsertImage: function(event) {
                    // 处理插入图片
                }
            }
        };
        
        const docEditor = new DocsAPI.DocEditor("ds-placeholder", config);
        
        function updateRoutingUI(roles) {
            // 显示表单填写进度
            const progressBar = document.getElementById('routing-progress');
            const completed = roles.filter(r => r.status === 'filled').length;
            progressBar.value = (completed / roles.length) * 100;
        }
    </script>
</body>
</html>
```

### Docker Compose 部署 9.4

```yaml
version: '3'
services:
  onlyoffice:
    image: onlyoffice/documentserver:9.4
    ports:
      - "8080:80"
    volumes:
      - ./data:/var/www/onlyoffice/Data
      - ./logs:/var/log/onlyoffice
    environment:
      - JWT_ENABLED=true
      - JWT_SECRET=your-secret-key
      - WOPI_ENABLED=true
```

## 4. 高级 WOPI 设置（9.4 新增）

系统管理员可以通过 Admin Panel 配置 WOPI 设置来提升 SharePoint 兼容性：

```yaml
# /etc/onlyoffice/documentserver/default.json 补充配置
{
  "services": {
    "CoAuthoring": {
      "server": {
        "wopi": {
          "zone": "external-https",
          "includeAuthorizationHeader": true
        }
      }
    }
  }
}
```

## 5. API 增强（9.4 新增方法）

### Doc Builder 新增

```java
// 创建带格式表格
CDocBuilder builder = new CDocBuilder();
builder.createFile("docx");
CDocBuilderContext context = builder.getContext();

// 创建表格
CDocBuilderValue api = context.getAll();
CDocBuilderValue doc = api.call("GetDocument");
CDocBuilderValue table = api.call("CreateTable", 3, 4);  // 3行4列

// 设置格式
api.call("SetTableCellPadding", 5);
table.call("SetWidth", "100%");

// 插入到文档
doc.call("GetElement", 0).call("InsertParagraph").call("AddTable", table);
```

## 注意事项

### 1. 表单设计与模板
- 使用 `.docxf` 格式创建表单模板
- 表单字段需要提前在 ONLYOFFICE Desktop 或 Web 编辑器中设计
- 角色分配在模板设计阶段完成

### 2. 回调 URL 安全
- 回调 URL 必须**外网可访问**，否则编辑器无法通知后端
- 推荐使用 JWT 验证回调来源
- 生产环境建议 HTTPS

### 3. 版本兼容
- onStartFilling 的 roles 参数仅 9.4+ 支持
- 旧版本调用端需做兼容判断：
```javascript
function onStartFilling(event) {
    const roles = event.data && event.data.roles ? event.data.roles : [];
    // 空数组兼容旧版
}
```

### 4. 工作流性能
- 复杂表单路由建议后端异步处理
- 填写状态变化可通过 WebSocket 推送给相关角色
- 表单提交后建议锁定文档防止二次修改
