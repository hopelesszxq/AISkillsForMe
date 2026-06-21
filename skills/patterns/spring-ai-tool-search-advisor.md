---
name: spring-ai-tool-search-advisor
description: Spring AI 2.0 GA Tool Search Advisor——大工具集的语义搜索与分级内存策略
tags: [patterns, spring-ai, ai, tool-calling, mcp, agent, memory]
---

## 概述

Spring AI 2.0.0 GA（2026-06-12 发布）在 ToolCallAdvisor 基础上新增了 **Tool Search Advisor**（工具搜索顾问）模式，解决大工具集场景下的上下文膨胀问题。同时引入了 **Recursive Advisor** 架构，支持 `OUTSIDE`/`INSIDE` 两种内存作用域，为 Agentic AI 提供更精细的控制。

> 基础 ToolCallAdvisor 和 Multi-Agent 编排请参见 `spring-ai-agent-patterns.md`，本文聚焦 2.0 GA 的新增能力。

## 一、Tool Search Advisor — 大工具集的语义搜索

### 问题背景

当 Agent 注册的工具数量超过 30 个时，将所有工具定义一次性发送给 LLM 会导致：
- **上下文膨胀**：大量 token 消耗在工具描述上
- **精度下降**：LLM 在过多选择中容易选错工具
- **成本增加**：每次请求都携带完整工具定义

### Tool Search Advisor 解决方案

Tool Search Advisor 通过增量检索模式，只在需要时发现并注入相关工具：

```
用户请求 → [Tool Search] → 检索 Top-K 工具 → [LLM 调用] → 工具执行 → 结果返回
                ↑                                              |
                └── 下一轮迭代（如果需要）──────────────────────┘
```

### 启用方式

```yaml
spring:
  ai:
    chat:
      client:
        tool-search-advisor:
          enabled: true                       # 启用工具搜索（默认 false）
          tool-index-type: vector             # 索引类型：regex | lucene | vector
          tool-index-name: my-tool-index      # 索引名称
          max-tools-per-request: 10           # 每轮最多注入的工具数
          min-score: 0.5                      # 相似度最低阈值
```

### 索引类型对比

| 类型 | 原理 | 适用场景 | 依赖 |
|------|------|---------|------|
| `regex` | 关键词正则匹配 | 小型工具集（<50） | 无额外依赖 |
| `lucene` | 全文检索 | 中型工具集（50-200） | 需 lucene 依赖 |
| `vector` | 向量语义搜索 | 大型工具集（>200） | 需 embedding 模型 |

### 会话隔离

每个会话使用独立的工具索引，必须提供 `CONVERSATION_ID`：

```java
String response = ChatClient.create(chatModel)
    .prompt()
    .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, "user-42-session"))
    .tools(new WeatherTools(), new FlightTools(), /* 50+ tools */)
    .user("帮我规划去阿姆斯特丹的旅行")
    .call()
    .content();
```

## 二、Recursive Advisor 与内存作用域

Spring AI 2.0 的 ToolCallAdvisor 是一个 **Recursive Advisor**——可以反复进入下游链直到满足停止条件。

### 内存作用域（Memory Scope）

控制 Advisor 在工具调用循环的**内部**还是**外部**执行：

```yaml
spring:
  ai:
    chat:
      client:
        memory-scope: outside  # outside | inside，默认 outside
```

#### OUTSIDE（外部模式，默认）

```
┌─────────────────────────┐
│ Memory Advisor (外圈)    │ ← 加载历史 → 完整会话 → 持久化
│                         │
│  ┌───────────────────┐  │
│  │ ToolCallAdvisor    │  │ ← 工具调用循环在内圈
│  │  LLM → 工具 → LLM  │  │    内存不可见工具消息
│  │  ...直到无工具调用   │  │
│  └───────────────────┘  │
└─────────────────────────┘
```

特点：
- 内存只保存最终的 User/Assistant 消息
- 工具请求/响应不写入持久化存储
- 兼容所有 `ChatMemory` 实现
- 与 Spring AI 1.x 行为一致

#### INSIDE（内部模式）

```
┌─────────────────────────┐
│ ToolCallAdvisor          │
│                          │
│  ┌───────────────────┐  │
│  │ Memory Advisor     │  │ ← 每轮迭代写入完整工具上下文
│  │ (内圈)             │  │
│  └───────────────────┘  │
│                          │
│  每轮: 工具请求/响应 → 存储 │
└─────────────────────────┘
```

特点：
- 每一轮工具调用的完整上下文被持久化
- LLM 可以回溯之前尝试过的工具和结果
- 必须使用支持 ToolMessage 的 `ChatMemory` 实现
- 需要禁用 `ChatClient` 的内部历史（Builder 自动处理）

```java
// 指定 INSIDE 模式
ChatClient.create(chatModel)
    .defaultAdvisors(
        new MessageChatMemoryAdvisor(memory, "session-1", INSIDE)
    )
    .build();
```

### 内置支持的消息类型

截至 2.0.0 GA，以下 `ChatMemory` 实现支持完整工具消息持久化：

| 实现 | 支持 ToolMessage | 备注 |
|------|-----------------|------|
| `InMemoryChatMemory` | ✅ | 测试/开发用 |
| `CassandraChatMemoryRepository` | ✅ (2.0.0 已修复) | 需过滤不支持的 ToolMessage |
| `MongoChatMemoryRepository` | ✅ (2.0.0 已修复) | 需过滤不支持的 ToolMessage |
| `JdbcChatMemoryRepository` | ✅ (2.0.0 已修复) | 需过滤不支持的 ToolMessage |
| 自定义实现 | 取决于序列化支持 | 需处理 ToolCallRequest/Response |

## 三、完整配置示例

```yaml
spring:
  ai:
    chat:
      client:
        enabled: true
        tool-search-advisor:
          enabled: true
          tool-index-type: vector
          max-tools-per-request: 8
          min-score: 0.4
        memory-scope: inside  # 启用内部模式记录完整工具调用历史

  ai:
    tool:
      call-advisor:
        enabled: true          # 默认 true
```

## 注意事项

1. **Tool Search Advisor 默认关闭**：需要通过 `spring.ai.chat.client.tool-search-advisor.enabled=true` 显式启用
2. **Vector 索引需要 Embedding 模型**：使用 `vector` 类型需要配置 EmbeddingModel Bean
3. **INSIDE 内存模式可能增加存储成本**：每次工具调用都会持久化完整的请求/响应消息
4. **工具消息过滤**：2.0.0 GA 修复了 Cassandra/Mongo/JDBC 仓库中工具消息过滤问题
5. **会话 ID 是必需的**：Tool Search Advisor 和 INSIDE 内存都依赖 `CONVERSATION_ID` 实现会话隔离
6. **与 Spring AI 1.x 不兼容**：Tool Search Advisor 和 Recursive Advisor 是 2.0 专属特性
