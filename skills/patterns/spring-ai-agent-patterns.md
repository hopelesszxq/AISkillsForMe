---
name: spring-ai-agent-patterns
description: Spring AI 2.0 Agentic AI 模式：ToolCallAdvisor、MCP Agent、Multi-Agent 编排架构
tags: [patterns, spring-ai, agent, ai, mcp, tool-calling, llm]
---

## 概述

Spring AI 2.0 在 Agentic AI 方面引入了重要改进：**ToolCallAdvisor 默认化**、**ToolSpec 流式 API** 和 **MCP SDK 2.0** 支持。这些特性使 Java 开发者能够以更低的复杂度构建 AI Agent。

## 一、ToolCallAdvisor 模式（2.0 默认）

Spring AI 2.0 将 `ToolCallAdvisor` 设为默认的 Tool Calling 管理方案，取代了 1.x 中需要手动配置的多种方式。

### 自动装配

```yaml
# 默认已启用，无需额外配置
spring:
  ai:
    tool:
      call-advisor:
        enabled: true  # 默认 true
```

### 自定义 Advisor

```java
@Configuration
public class AgentConfig {

    @Bean
    public ToolCallingManager toolCallingManager() {
        return ToolCallingManager.builder()
            .advisor(new ToolCallAdvisor())
            .build();
    }
}
```

## 二、ToolSpec 流式 API 与 @Tool 注解对比

| 方式 | 适用场景 | 复杂度 | 灵活性 |
|------|---------|--------|--------|
| `@Tool` 注解 | 简单工具，方法级声明 | 低 | 中 |
| `ToolSpec` 流式 API | 动态工具注册、复杂 Schema | 中 | 高 |

### @Tool 注解（推荐简单场景）

```java
@Component
public class WeatherTools {

    @Tool(name = "get_weather", description = "获取天气信息")
    public String getWeather(@ToolParam("city") String city) {
        return weatherService.fetch(city);
    }
}
```

### ToolSpec API（动态/复杂场景）

```java
@Service
public class DynamicToolRegistry {

    private final ToolCallingManager toolManager;

    public void registerDynamicTools(List<ApiEndpoint> endpoints) {
        for (ApiEndpoint ep : endpoints) {
            ToolSpec spec = ToolSpec.builder()
                .name(ep.getName())
                .description(ep.getDescription())
                .inputSchema(buildSchema(ep))
                .build();

            toolManager.registerTool(spec, request -> {
                return callExternalApi(ep, request);
            });
        }
    }
}
```

## 三、MCP Agent 模式

Spring AI 2.0 + MCP SDK 2.0 实现了标准的 Agent 与外部工具交互协议。

### 架构

```
┌──────────────────┐
│   AI Agent       │
│  (Spring AI)     │
├──────────────────┤
│ ToolCallAdvisor  │ ← 管理工具调用协调
├──────────────────┤
│ MCP Client       │ ← 通过 MCP 协议连接外部工具
└──────┬───────────┘
       │
┌──────┴───────────┐
│ MCP Server(s)    │ ← 插件化工具（文件系统、数据库、Web）
└──────────────────┘
```

### MCP Client 配置

```java
@Bean
public McpClient mcpFileSystemClient() {
    return McpClient.sync(
        new StdioMcpTransport(
            new StdioClientTransportOptions(
                "npx", "-y", "@modelcontextprotocol/server-filesystem")))
        .clientInfo(new McpClientInfo("my-agent", "1.0.0"))
        .build();
}

@Bean
public McpClient mcpSqlClient() {
    return McpClient.sync(
        new StdioMcpTransport(
            new StdioClientTransportOptions(
                "npx", "-y", "@modelcontextprotocol/server-sqlite", "--db", "data.db")))
        .build();
}
```

## 四、Multi-Agent 编排模式

复杂的 Agent 场景可通过 `ToolCallAdvisor` + `ChatClient` 组合实现多 Agent 编排。

### 路由 Agent 模式

```java
@Service
public class RouterAgent {

    private final ChatClient routerClient;
    private final Map<String, ChatClient> specializedAgents;

    public RouterAgent(ChatClient.Builder builder) {
        // 路由 Agent：判断用户意图
        this.routerClient = builder
            .defaultSystem("""
                你是一个路由助手。分析用户请求，判断应由哪个专业 Agent 处理：
                - order: 订单相关问题
                - product: 商品相关问题
                - customer: 客户服务问题
                只需返回 Agent 名称。
                """)
            .build();
    }

    public String handleRequest(String userMessage) {
        // 1. 路由决策
        String target = routerClient.prompt()
            .user(userMessage)
            .call()
            .content();

        // 2. 转发给专业 Agent
        ChatClient agent = specializedAgents.get(target.trim());
        return agent.prompt()
            .user(userMessage)
            .call()
            .content();
    }
}
```

## 五、GraalVM Native Image 支持

Spring AI 2.0.0-M7 修复了 Ollama 在 GraalVM Native Image 下的兼容性问题。

```xml
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
</plugin>
```

```java
// 确保在 runtime 初始化 AI 相关类
// src/main/resources/META-INF/native-image/reflect-config.json
[
  {
    "name": "org.springframework.ai.ollama.OllamaModel",
    "allDeclaredMethods": true
  }
]
```

## 注意事项

- **单 ToolAdvisor 不变性**：`ChatClient` 中只能有一个 `ToolAdvisor`，多 Advisor 会抛出异常（2.0.0-M7 强制校验）
- **MCP SDK 2.0 API 不兼容**：升级后需适配新的 `McpClient` 构建方式
- **ToolSpec 与 @Tool 不能混用**：同一 `ToolCallingManager` 实例中保持一致
- **Native Image 内存预热**：AI 模型加载在首次调用时可能较慢，建议启动后预热
