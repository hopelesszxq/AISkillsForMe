---
name: spring-ai-patterns
description: Spring AI 1.0 集成模式：MCP、Tool Calling、RAG、Vector Store 实战指南
tags: [patterns, spring, ai, mcp, rag, llm, java]
---

## 概述

Spring AI 1.0（2025 Q1 GA）为 Java 生态提供了统一的 AI/LLM 集成框架。核心能力包括：Chat Client、Tool Calling、RAG（检索增强生成）、Vector Store、MCP（Model Context Protocol）等。阿里巴巴的 spring-ai-alibaba 扩展了 Agentic AI 框架能力。

## 一、核心架构

```
┌─────────────────────┐
│   Chat Client API    │
├─────────────────────┤
│ ChatModel → Adapter │ ← OpenAI / Ollama / DashScope / DeepSeek
├─────────────────────┤
│   Tool Calling      │ ← @Tool 注解方法注册
├─────────────────────┤
│   Vector Store      │ ← PGvector / Redis / Milvus
├─────────────────────┤
│   MCP Client        │ ← 标准协议接入外部工具
└─────────────────────┘
```

## 二、基础 Chat 集成

```java
// pom.xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

```yaml
# application.yml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
```

```java
@RestController
public class ChatController {
    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @GetMapping("/chat")
    public String chat(@RequestParam String message) {
        return chatClient.prompt()
            .user(message)
            .call()
            .content();
    }
}
```

## 三、Tool Calling（函数调用）

Spring AI 支持通过 `@Tool` 注解将 Java 方法暴露为 LLM 可调用的工具。

```java
@Component
public class OrderTools {

    @Tool(name = "get_order_status", description = "查询订单状态")
    public String getOrderStatus(@ToolParam("orderId") String orderId) {
        // 实际业务查询
        return "订单 " + orderId + " 状态: 已发货";
    }

    @Tool(name = "create_refund", description = "创建退款申请")
    public String createRefund(
            @ToolParam("orderId") String orderId,
            @ToolParam("reason") String reason) {
        return "退款申请已提交: " + orderId;
    }
}
```

```java
// 注册工具
@Bean
public ToolCallback toolCallbacks(List<OrderTools> tools) {
    return new MethodToolCallback.Builder()
        .toolDefinition(ToolDefinition.fromMethod(
            tools.get(0).getClass().getMethod("getOrderStatus", String.class)))
        .toolObject(tools.get(0))
        .build();
}
```

## 四、RAG（检索增强生成）

### 依赖配置

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
</dependency>
```

### 文档嵌入与检索

```java
@Service
public class RagService {

    private final VectorStore vectorStore;
    private final ChatClient chatClient;

    public RagService(VectorStore vectorStore, ChatClient.Builder builder) {
        this.vectorStore = vectorStore;
        this.chatClient = builder.build();
    }

    // 文档入库
    public void indexDocument(String id, String content) {
        Document doc = new Document(id, content);
        List<Document> chunks = new TokenTextSplitter()
            .withChunkSize(500)
            .withOverlap(50)
            .split(List.of(doc));
        vectorStore.add(chunks);
    }

    // 检索回答
    public String ask(String question) {
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.query(question).withTopK(3));

        String context = docs.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n---\n"));

        return chatClient.prompt()
            .user(u -> u.text("基于以下信息回答问题：\n{context}\n\n问题：{question}")
                .param("context", context)
                .param("question", question))
            .call()
            .content();
    }
}
```

## 五、MCP (Model Context Protocol) 集成

MCP 是 AI 模型与外部工具之间的标准协议。Spring AI 支持作为 MCP Client。

```java
// MCP 工具定义 (服务端)
@SpringBootApplication
@EnableMcpServer
public class McpServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(McpServerApplication.class, args);
    }
}

// 暴露为 MCP Tool
@Component
public class DatabaseMcpTool {
    @McpTool(name = "query_database")
    public String queryDatabase(
            @McpToolParam("sql") String sql) {
        // 执行 SQL 查询
        return queryResult;
    }
}
```

```java
// MCP Client 端（Spring AI 应用）
@Bean
public McpClient mcpClient() {
    return McpClient.builder()
        .url("http://localhost:8081/mcp")
        .build();
}
```

## 六、阿里云 DashScope / 通义千问 集成

```xml
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      chat:
        options:
          model: qwen-max
```

```java
@Bean
public ChatClient chatClient(ChatClient.Builder builder) {
    return builder
        .defaultSystem("你是一个Java开发助手，用中文回答")
        .build();
}
```

## 七、Vector Store 选型对比

| 存储 | 适用场景 | 维度上限 | 部署方式 |
|------|---------|---------|---------|
| PGvector | 与业务数据共存 | 2000 | PostgreSQL 插件 |
| Redis Stack | 低延迟缓存场景 | 10000 | Redis Stack |
| Milvus | 大规模向量检索（百万级以上）| 32768 | 独立集群 |
| Elasticsearch | 全文搜索+向量混合 | 2048 | ES 插件 |

## 八、最佳实践

### 1. 请求限流与重试

```java
@Bean
public ChatClient chatClient(ChatClient.Builder builder) {
    return builder
        .defaultAdvisors(
            new SimpleRetryAdvisor(3),       // 失败重试3次
            new SimpleLoggerAdvisor())       // 日志记录
        .build();
}
```

### 2. 输出结构化（Bean Output）

```java
public record OrderSummary(String orderId, String status, BigDecimal amount) {}

@GetMapping("/analyze")
public OrderSummary analyze(@RequestParam String text) {
    return chatClient.prompt()
        .user(text)
        .call()
        .entity(OrderSummary.class);  // 自动映射为 Java Bean
}
```

### 3. 多轮对话

```java
ChatMemory memory = new InMemoryChatMemory();

// 每次请求携带历史
chatClient.prompt()
    .user("我的订单号是什么？")
    .advisors(a -> a.param("chat_memory", memory))
    .call();
```

## 注意事项

| 要点 | 说明 |
|------|------|
| **API Key 保护** | 永远不要硬编码 API Key，使用环境变量或 Vault |
| **Token 成本** | RAG 检索时控制 TopK 和 chunk 大小，避免超出上下文窗口 |
| **Tool 安全** | Tool Calling 的 Java 方法必须做权限校验，防止 SQL 注入等 |
| **异常处理** | LLM 调用可能超时/失败，必须做好降级和重试策略 |
| **MCP 认证** | MCP 服务端需要 Token 认证，勿暴露到公网 |
| **版本兼容** | spring-ai-alibaba 和 spring-ai 官方版本号不同步，注意版本矩阵 |
| **Embedding 模型** | 生产环境选用专用 Embedding 模型（如 text-embedding-3-small）而非对话模型 |
