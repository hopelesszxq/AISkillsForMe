---
name: contract-testing-pact
description: 微服务消费者驱动契约测试实践——Pact 模式、OpenAPI-first 替代方案及落地策略
tags: [patterns, microservices, contract-testing, pact, api, testing]
---

## 概述

契约测试（Contract Testing）是微服务架构中确保服务间 API 兼容性的关键技术。核心思想：**验证消费者（Consumer）对提供者（Provider）API 的期望是否被满足**，而非运行全链路集成测试。

当前主流有两种路线：
- **Pact**（消费者驱动契约测试，CDC）——高保真但高门槛
- **OpenAPI-first**（基于 OpenAPI 子集对比）——低保真但低门槛

> 数据显示，实际采用契约测试的微服务团队不到 5%，主要阻力来自实施成本而非认知不足。

## Pact 消费者驱动契约模式

### 核心流程

```
消费者编写测试 → 生成契约 → 发布到 Broker → 提供者在 CI 验证 → 阻塞不兼容的部署
```

### 1. 消费者端（Pact DSL）

```java
// Java 消费者测试示例
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "orders-service", port = "8080")
class OrderClientTest {

    @Pact(consumer = "web-frontend")
    public V4Pact createOrderPact(PactDslWithProvider builder) {
        return builder
            .given("order 123 exists")
            .uponReceiving("a request for order 123")
                .path("/orders/123")
                .method("GET")
            .willRespondWith()
                .status(200)
                .headers(Map.of("Content-Type", "application/json"))
                .body(new PactDslJsonBody()
                    .integerType("id", 123)
                    .stringType("status", "shipped")
                    .eachLike("items", 1)
                        .stringType("sku")
                        .numberType("price")
                    .closeArray()
                )
            .toPact();
    }

    @Test
    void testGetOrder() {
        Order order = client.getOrder(123);
        assertThat(order.getStatus()).isEqualTo("shipped");
    }
}
```

### 2. 提供者端（验证）

```java
@Provider("orders-service")
@PactBroker(url = "https://pact-broker.example.com")
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderServicePactTest {

    @LocalServerPort
    int port;

    @BeforeEach
    void setup() {
        // 设置测试数据状态
        orderRepository.save(new Order(123L, "shipped"));
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact(PactVerificationContext context) {
        context.verifyInteraction();
    }
}
```

## 实施契约测试的 5 大成本

| 成本维度 | 说明 | 缓解方案 |
|---------|------|---------|
| DSL 学习 | 每个消费者团队需要学习 Pact 的 Matcher DSL | 提供模板/脚手架 |
| Broker 运维 | 需要自建 Pact Broker 或购买 PactFlow | 高可用部署 + CI 集成 |
| 不重用现有资产 | OpenAPI/集成测试不能直接复用 | 考虑 OpenAPI-first 方案 |
| Provider CI 改造 | 需要 Provider 启动完整服务环境验证 | 使用 Docker Compose / Testcontainers |
| 网络效应门槛 | 需要所有消费者+提供者同时接入 | 渐进式推进，优先核心链路 |

## OpenAPI-first 契约测试

### 核心理念

提供者维护完整的 OpenAPI 规范，消费者声明使用的 API 子集（subset），由引擎判断兼容性：

```
提供者 OpenAPI 规范（完整）
        ↕ 子集对比
消费者 API 使用声明（自动/手动生成）
```

### 常用工具

| 工具 | 类型 | 特点 |
|------|------|------|
| **Specmatic** | OSS | Contract-Driven Development，支持 Mock + Stub |
| **Microcks** | CNCF 孵化 | 多协议支持（OpenAPI, AsyncAPI, gRPC, GraphQL） |
| **oasdiff** | OSS CLI | OpenAPI 差异对比，10 分钟可接入 CI |
| **Bump.sh** | SaaS | OpenAPI Diff + API 文档 |

### oasdiff 快速接入示例

```yaml
# .github/workflows/api-check.yml
name: API Compatibility Check
on: [pull_request]
jobs:
  diff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check API diff
        uses: oasdiff/oasdiff-action@main
        with:
          base: main/openapi.yaml
          revision: pr/openapi.yaml
          fail-on: changed
```

### Specmatic 示例

```java
// 基于 OpenAPI 的契约测试（Java）
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class ContractVerificationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void verifyOrderContract() {
        Specmatic specmatic = new Specmatic("openapi.yaml");
        specmatic.verify(restTemplate);
    }
}
```

## 实践建议

### 团队决策树

```
当前是否做过任何契约测试？
├── 否 → 从 oasdiff 开始（~10 分钟配置）
│        ├── 够用 → 保持
│        └── 不够 → 升级到 Specmatic/Microcks
│
├── 已用 Pact 且运转良好 → 保持 Pact
│
└── 试过 Pact 但放弃了 → OpenAPI-first 方案
```

### 渐进式推进策略

1. **第一阶段**：oasdiff 检查 OpenAPI 变更（阻挡显式破坏性变更）
2. **第二阶段**：核心服务使用 Specmatic/Microcks（验证字段级兼容性）
3. **第三阶段**：关键消费者链路采用 Pact（高保真验证）

### CI 集成要点

```yaml
# 契约检查应在合并前阻塞，而非合并后告警
# 失败策略：block（默认）vs warn（仅告警）
fail-on: changed  # 合理起点
```

## 注意事项

- **Pact 不是银弹**：对于中小规模团队（< 10 个服务），OpenAPI diff 可能已经足够
- **契约测试 ≠ 端到端测试**：契约测试验证 API 合约，不替代业务功能的 E2E 测试
- **Broker 是单点**：自建 Pact Broker 需要额外的运维投入，考虑开源自托管方案
- **Polyglot 环境**：Pact 支持 12 种语言，但跨语言团队的一致性维护成本不可忽视
- **OpenAPI-first 的局限**：无法验证行为语义（如正则约束、状态依赖），只验证结构兼容性
- **避免过度工程**：2-3 个服务的项目不需要契约测试，API 变更直接协调即可
