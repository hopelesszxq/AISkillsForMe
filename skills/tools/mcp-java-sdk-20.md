---
name: mcp-java-sdk-20
description: MCP Java SDK 2.0 集成模式——客户端/服务端架构、工具定义、资源访问、Spring AI 整合与传输层选择
tags: [tools, mcp, spring-ai, ai, model-context-protocol, java, agent]
---

## 概述

**Model Context Protocol (MCP)** 是 Anthropic 主导的开放协议，定义 AI 应用与外部工具/数据源之间的标准化通信方式。**MCP Java SDK 2.0.0-RC1**（2026-06-04 发布）提供了完整的客户端/服务端实现，Spring AI 2.0 已深度集成 MCP 作为 Agent 工具调用协议。

> 当前最新：**v2.0.0-RC1**（Java SDK），Spring AI 2.0.0-RC1 默认集成 MCP 2.0 SDK。

## 核心架构

```
┌─────────────────┐     MCP 协议      ┌──────────────────┐
│  AI Application  │ ◄────────────► │   MCP Server(s)   │
│  (MCP Client)    │   Stdio / SSE    │ ┌─ Tool Provider │
│                  │                  │ ├─ Resource      │
│  Spring AI       │                  │ ├─ Prompt        │
│  ChatClient      │                  │ └─ Roots         │
└─────────────────┘                  └──────────────────┘
```

**三个核心概念**：

| 概念 | 说明 | 类比 |
|------|------|------|
| **Tools** | AI 可调用的函数（读文件、查数据库、发请求） | Function Calling 的标准化 |
| **Resources** | AI 可读取的数据（文件内容、DB 记录） | REST GET 的 AI 版本 |
| **Prompts** | 预定义的提示模板 | 可复用的 Prompt 片段 |

## 一、MCP Server 实现

### 1.1 基础 Server（Stdio 传输）

Stdio 传输通过标准输入输出通信，适合本地进程调用：

```java
// 依赖：io.modelcontextprotocol:mcp:2.0.0-RC1
import io.modelcontextprotocol.server.McpServer;
import io.modelcontextprotocol.server.McpServerFeatures;
import io.modelcontextprotocol.spec.McpSchema;

// 创建 Server
var server = McpServer.sync(transport)
    .serverInfo("my-tools-server", "1.0.0")
    .build();

// 注册工具
server.addTool(new McpServerFeatures.SyncToolSpecification(
    new McpSchema.Tool("read-file", "读取本地文件内容",
        new McpSchema.ObjectSchema()
            .addProperty("path", new McpSchema.StringSchema("文件路径"))
            .build()),
    (exchange, arguments) -> {
        String path = (String) arguments.get("path");
        String content = Files.readString(Path.of(path));
        return new McpSchema.CallToolResult(
            List.of(new McpSchema.TextContent(content)), false);
    }
));
```

### 1.2 使用 StdioServerTransport

```java
import io.modelcontextprotocol.server.transport.StdioServerTransport;

var transport = new StdioServerTransport();
var server = McpServer.sync(transport)
    .serverInfo("code-analyzer", "0.1.0")
    .build();
```

### 1.3 使用 SSEServerTransport（远程调用）

SSE（Server-Sent Events）传输支持远程 MCP Server：

```java
import io.modelcontextprotocol.server.transport.SseServerTransport;

// Spring Boot 中注册
@Bean
public McpServer mcpServer() {
    var transport = new SseServerTransport("/mcp/message");
    return McpServer.sync(transport)
        .serverInfo("remote-tools", "1.0.0")
        .build();
}

// 在 Controller 中处理 SSE 连接
@GetMapping("/mcp")
public SseEmitter handleMcpConnection(
        @RequestParam("sessionId") String sessionId) {
    var transport = new SseServerTransport("/mcp/message");
    return transport.handleConnection(sessionId);
}
```

## 二、注册资源（Resources）

资源允许 AI 读取结构化数据，类似 REST GET：

```java
// 注册静态资源
server.addResource(new McpServerFeatures.SyncResourceSpecification(
    new McpSchema.Resource(
        "docs://api-guide",       // URI
        "API 接口文档",            // 名称
        "项目 API 文档内容",       // 描述
        "application/markdown",   // MIME 类型
        null                      // 元数据
    ),
    (exchange, request) -> {
        // 返回资源内容
        return new McpSchema.ReadResourceResult(
            List.of(new McpSchema.TextResourceContents(
                "docs://api-guide",
                "application/markdown",
                Files.readString(Path.of("API.md"))
            ))
        );
    }
));
```

## 三、MCP Client 配置

### 3.1 Stdio Client（本地进程）

```java
import io.modelcontextprotocol.client.McpClient;
import io.modelcontextprotocol.client.transport.StdioClientTransport;

var transport = new StdioClientTransport(
    new ProcessBuilder("java", "-jar", "my-mcp-server.jar"));
var client = McpClient.sync(transport)
    .clientInfo(new McpSchema.ClientInfo("my-agent", "1.0.0"))
    .build();

// 获取可用工具列表
var tools = client.listTools();
tools.forEach(t -> System.out.println(t.getName() + ": " + t.getDescription()));

// 调用工具
var result = client.callTool(new McpSchema.CallingToolRequest(
    "read-file", Map.of("path", "/tmp/data.txt")));
```

### 3.2 SSE Client（远程服务）

```java
import io.modelcontextprotocol.client.transport.SseClientTransport;

var transport = new SseClientTransport(
    URI.create("https://mcp-server.example.com/mcp"));
var client = McpClient.sync(transport)
    .clientInfo(new McpSchema.ClientInfo("remote-agent", "1.0.0"))
    .build();
```

### 3.3 异步 Client

```java
// 使用 reactive API
McpClient.async(transport)
    .clientInfo(new McpSchema.ClientInfo("async-agent", "1.0.0"))
    .build()
    .flatMap(client -> client.listTools())
    .flatMap(tools -> {
        System.out.println("可用工具: " + tools.size());
        return client.callTool(new McpSchema.CallingToolRequest(
            "search", Map.of("q", "MCP protocol")));
    })
    .subscribe();
```

## 四、Spring AI 2.0 + MCP 集成

### 4.1 自动配置

Spring AI 2.0 提供了 `spring-ai-mcp-client` 起步依赖，自动发现和注册 MCP 工具：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-mcp-client</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  ai:
    mcp:
      client:
        enabled: true
        # Stdio 方式：指向 MCP Server 进程
        stdio:
          - command: java
            args: ["-jar", "my-mcp-server.jar"]
        # SSE 方式：远程 MCP Server
        sse:
          - url: "https://mcp-server.example.com/mcp"
```

### 4.2 直接在 ChatClient 中使用

```java
@Autowired
private McpClient mcpClient;

// 获取所有 MCP 工具作为 ToolCallback
List<ToolCallback> mcpTools = mcpClient.listTools().stream()
    .map(tool -> new McpToolCallback(mcpClient, tool))
    .toList();

// 注入到 ChatClient
String response = ChatClient.create(chatModel)
    .prompt("读取 /tmp/config.json 的内容并总结")
    .tools(mcpTools)
    .call()
    .content();
```

### 4.3 手动注册多个 MCP Client

```java
@Configuration
public class McpConfig {

    @Bean
    public McpClient fileSystemClient() {
        return McpClient.sync(new StdioClientTransport(
                new ProcessBuilder("npx", "@modelcontextprotocol/server-filesystem", "/tmp")))
            .clientInfo(new McpSchema.ClientInfo("fs-agent", "1.0.0"))
            .build();
    }

    @Bean
    public McpClient sqlClient() {
        return McpClient.sync(new StdioClientTransport(
                new ProcessBuilder("npx", "@modelcontextprotocol/server-sqlite", "data.db")))
            .clientInfo(new McpSchema.ClientInfo("sql-agent", "1.0.0"))
            .build();
    }

    @Bean
    public List<ToolCallback> allMcpTools(
            List<McpClient> clients) {
        return clients.stream()
            .flatMap(client -> client.listTools().stream()
                .map(tool -> new McpToolCallback(client, tool)))
            .toList();
    }
}
```

## 五、MCP 2.0 RC1 新特性

### 5.1 URL Elicitation（SEP-1036）

MCP 2.0 新增 URL 诱导（URL Elicitation）能力，客户端可以通过 URL 向 AI 提供上下文：

```java
// Server 端支持 URL 资源
server.addResource(new McpServerFeatures.SyncResourceSpecification(
    new McpSchema.Resource(
        "https://api.example.com/docs",  // URL 作为资源 URI
        "外部 API 文档",
        "application/json"
    ),
    (exchange, request) -> {
        var content = HttpClient.newHttpClient()
            .send(HttpRequest.newBuilder(URI.create(request.getUri())).build(),
                  HttpResponse.BodyHandlers.ofString())
            .body();
        return new McpSchema.ReadResourceResult(
            List.of(new McpSchema.TextResourceContents(
                request.getUri(), "application/json", content)));
    }
));
```

### 5.2 增强的授权错误处理

`McpHttpClientAuthorizationErrorHandler` 现在暴露请求 URI，方便自定义身份验证逻辑：

```java
// 自定义授权处理
client = McpClient.sync(transport)
    .clientInfo(...)
    .authorizationHandler((requestUri, response) -> {
        if (response.statusCode() == 401) {
            // 刷新 Token 并重试
            String token = refreshToken();
            return HttpRequest.newBuilder(requestUri)
                .header("Authorization", "Bearer " + token)
                .build();
        }
        return null; // 不处理
    })
    .build();
```

### 5.3 引导 Schema 默认值（SEP-1034）

MCP 2.0 支持客户端侧应用引导 schema 默认值，减少重复参数：

```java
// Server 定义工具参数及默认值
new McpSchema.Tool("search", "搜索", 
    new McpSchema.ObjectSchema()
        .addProperty("query", new McpSchema.StringSchema("搜索关键词"))
        .addProperty("limit", new McpSchema.NumberSchema("返回条数")
            .defaultValue(10))  // ← 客户端侧应用默认值
        .addProperty("language", new McpSchema.StringSchema("语言")
            .defaultValue("zh-CN"))
        .build());
```

## 六、最佳实践

### 6.1 工具设计原则

- **单一职责**：每个工具只做一件事，名称清晰（`read-file`、`search-db`）
- **参数验证**：在 Server 端做参数校验，返回明确错误信息
- **幂等性**：读操作保持幂等，写操作提供确认机制

### 6.2 安全注意事项

```java
// 限制文件系统工具的访问范围
server.addTool(new McpServerFeatures.SyncToolSpecification(
    new McpSchema.Tool("read-file", "读取文件",
        new McpSchema.ObjectSchema()
            .addProperty("path", new McpSchema.StringSchema("文件路径"))
            .build()),
    (exchange, args) -> {
        String path = (String) args.get("path");
        // ⚠️ 路径安全校验：禁止路径穿越
        Path resolved = Path.of("/allowed/dir").resolve(path).normalize();
        if (!resolved.startsWith("/allowed/dir")) {
            return new McpSchema.CallToolResult(
                List.of(new McpSchema.TextContent("Access denied")), true);
        }
        // ... 读取文件
    }
));
```

### 6.3 传输层选择

| 传输方式 | 适用场景 | 优点 | 缺点 |
|---------|---------|------|------|
| **Stdio** | 本地进程调用 | 低延迟、无网络开销 | 进程管理复杂 |
| **SSE** | 远程服务 | 跨网络、可水平扩展 | 需处理认证和网络问题 |

## 注意事项

- **MCP 2.0 breaking changes**：从 1.x 升级时，`McpClient` 构建方式已变为 Builder 模式，`toolNames()` API 已移除，需使用显式的 `ToolCallback` bean
- **Stdio 进程生命周期**：确保 MCP server 进程在应用关闭时正确终止，可以使用 `Runtime.getRuntime().addShutdownHook()`
- **工具命名冲突**：多个 MCP Client 注册相同名称的工具时，Spring AI 默认去重（保留最后注册的），可通过 `toolCallbackNameResolver` 自定义命名策略
- **SSE 会话管理**：SSE 传输需要管理会话 ID，确保每个客户端连接有唯一标识
- **测试**：MCP SDK 提供了 `McpServer.sync(transport).build()` 的测试辅助方法，可以通过模拟 transport 进行单元测试
