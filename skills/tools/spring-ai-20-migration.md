---
name: spring-ai-20-migration
description: Spring AI 2.0 迁移指南：从 1.x 升级到 2.0 的 Breaking Changes、新特性与最佳实践
tags: [tools, spring-ai, migration, llm, ai, spring-boot-4]
---

## 概述

Spring AI 2.0.0（M7/M8 里程碑，2026 年 5 月）是面向 **Spring Boot 4.x** 的重大版本升级。相比 Spring AI 1.x（Boot 3.x），2.0 引入了大量 Breaking Changes 和架构改进。

> Spring AI 2.0 主线要求 Spring Boot 4.x，1.1.x 分支继续支持 Boot 3.5.x。

## 一、关键变更汇总

| 变更 | 1.x | 2.0 | 影响 |
|------|-----|-----|------|
| Spring Boot 基线 | 3.5.x | **4.x** | 需先升级 Boot |
| 配置属性风格 | camelCase | **dash-separated** | 所有配置需修改 |
| ChatOptions API | 有 setter 方法 | **移除 setter，仅用 mutate()** | 代码需调整 |
| SSE 传输 | 默认 | **废弃，默认 Streamable HTTP** | 服务端协议变更 |
| Tool Calling 管理 | 多种方式 | **ToolCallAdvisor 为默认** | 简化配置 |
| Tool API | 手动构建 | **ToolSpec 流式 API** | 开发体验提升 |
| Spring Cloud Bindings | 支持 | **删除** | 移除依赖 |
| CosmosDB VectorStore | 支持 | **删除** | 迁移到其他 Vector Store |
| Gemini 默认模型 | 2.0 Flash | **2.5 Flash** | 自动升级 |
| MCP SDK | 1.x | **2.0.0-M3** | 需检查 API 兼容性 |

## 二、配置属性变更（最易踩坑）

### 1.x 风格（camelCase）

```yaml
# Spring AI 1.x - 旧风格
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
```

### 2.0 风格（dash-separated）

```yaml
# Spring AI 2.0 - 新风格
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
```

> 注意 `api-key`→`api-key`（已内置转换），但更复杂的属性名如 `maxTokens`→`max-tokens` 需留意。

## 三、ChatOptions API 变更

### 1.x（setter 模式）

```java
// Spring AI 1.x - 有 setter
OpenAiChatOptions options = OpenAiChatOptions.builder()
    .withModel("gpt-4o")
    .withTemperature(0.7)
    .build();
```

### 2.0（mutate 模式，移除 setter）

```java
// Spring AI 2.0 - 仅 mutate()
ChatOptions baseOptions = ChatOptions.builder()
    .model("gpt-4o")
    .temperature(0.7)
    .build();

// 基于已有 options 创建变体
ChatOptions modified = baseOptions.mutate()
    .withTemperature(0.9)
    .build();
```

## 四、ToolSpec 流式 API

Spring AI 2.0 引入了 `ToolSpec` 流式 API，代替手动构建 `ToolInfo`。

```java
// Spring AI 2.0 ToolSpec API
ToolSpec toolSpec = ToolSpec.builder()
    .name("get_weather")
    .description("获取指定城市的天气")
    .inputSchema(ToolSpec.inputSchema()
        .property("city", ToolSpec.stringType()
            .description("城市名称")
            .required())
        .property("unit", ToolSpec.stringType()
            .description("温度单位：celsius/fahrenheit")
            .enumValues("celsius", "fahrenheit"))
        .build())
    .build();
```

## 五、Streamable HTTP 替代 SSE

```yaml
# Spring AI 2.0 默认使用 Streamable HTTP
# SSE 被废弃但仍可用（需显式配置）
spring:
  ai:
    streaming:
      server-transport: streamable-http  # 默认值
```

```java
// 如需显式使用 SSE（不推荐）
@Bean
public SseClientTransport sseClientTransport() {
    return new WebFluxSseClientTransport(webClient);
}
```

## 六、MCP SDK 升级（2.0.0-M3）

```xml
<!-- MCP Client 依赖 -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-mcp-client</artifactId>
</dependency>
```

MCP 2.0 SDK 有 Breaking API 变更，原有 `McpClient` 构建方式需调整：

```java
// Spring AI 2.0 + MCP 2.0 SDK
McpTransport transport = new StdioMcpTransport(
    new StdioClientTransportOptions("npx", "-y", "@modelcontextprotocol/server-filesystem"));

McpClient client = McpClient.sync(transport)
    .clientInfo(new McpClientInfo("my-app", "1.0.0"))
    .build();
```

## 七、移除的组件与替代方案

| 移除组件 | 替代方案 |
|---------|---------|
| CosmosDB VectorStore | Azure AI Search / PGVector / Redis |
| spring-ai-spring-cloud-bindings | 手动配置绑定或使用 Spring Cloud Config |
| SSE (默认) | Streamable HTTP（默认） |

## 八、迁移步骤

```bash
# 1. 先升级 Spring Boot 3.5.x → 4.x
# 2. 升级 Spring AI 1.x → 2.0.x
# 3. 修改 application.yml 中 dash-separated 属性
# 4. 替换 ChatOptions setter 为 mutate()
# 5. 调整 Tool Calling 代码使用 ToolSpec
# 6. 移除已废弃的依赖（spring-cloud-bindings, cosmosdb）
# 7. 检查 SSE 客户端是否需要适配 Streamable HTTP
```

## 注意事项

- **Maven 版本兼容**：Spring AI 2.0 位于主分支（`main`），1.1.x 分支仍在维护
- **Ollama GraalVM 原生镜像**：2.0.0-M7 修复了 Ollama 在 GraalVM Native Image 下的兼容性问题
- **Gemini 默认模型升级**：从 2.0 Flash 变为 2.5 Flash，确认业务兼容性
- **PGVector 向量维度校验**：2.0 新增向量维度校验，确保写入数据维度与索引一致
- **OpenAI Starter 依赖修复**：`spring-ai-starter-model-google-genai` 曾错误声明 embedding 依赖（M8 修复）
