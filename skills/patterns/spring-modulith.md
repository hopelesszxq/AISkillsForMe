---
name: spring-modulith
description: Spring Modulith 模块化单体架构模式——在单体应用内实现模块化边界、依赖验证、独立测试与可观测性
tags: [patterns, spring-modulith, modular-monolith, architecture, spring-boot, ddd]
---

## 概述

Spring Modulith（当前最新 2.1.0-RC1，2026-04-24）是 Spring 官方提供的**模块化单体（Modular Monolith）** 框架。它在保持单体部署简单性的同时，通过强制模块边界、自动验证依赖方向、支持独立测试和模块级可观测性，解决了传统单体应用"大泥球"问题。

> 模块化单体是一种中间架构——在单体部署的便利性和微服务的清晰边界之间取得平衡。当性能要求提升或团队规模扩大时，模块可以平滑拆分为独立微服务。

## 核心概念

### 应用模块（Application Modules）

Spring Modulith 将应用分为**应用模块**，每个模块是主包下的直接子包：

```
com.myapp
├── order/          # 订单模块
│   ├── Order.java
│   ├── OrderRepository.java
│   └── OrderService.java
├── payment/        # 支付模块
│   ├── Payment.java
│   └── PaymentService.java
├── notification/   # 通知模块
│   └── NotificationService.java
└── Application.java
```

### 模块化验证

在测试中自动验证依赖规则，防止模块间非法依赖：

```java
@Test
void verifyModularStructure() {
    ApplicationModules.of(MyApplication.class).verify();
}
```

这确保模块之间的依赖关系符合预期——例如 `order` 可以依赖 `payment`，但 `payment` 不能反向依赖 `order`。

### 模块依赖声明

```java
// 默认：模块只能访问自己的类型和公开的 API 类型
// 显式声明模块依赖（可选）
@ApplicationModule(
    allowedDependencies = {"payment", "notification"}
)
class OrderModule { }
```

## 模块间通信

### 1. 公开 API 包

模块通过专用的 `api` 子包暴露接口，其他模块只能访问 `api` 中的类型：

```
com.myapp.order
├── api/
│   └── OrderManagement.java   ← 其他模块可见
├── internal/
│   ├── Order.java
│   └── OrderRepository.java
└── OrderService.java
```

```java
// 在 application.properties 中配置
spring.modulith.api-packages=com.myapp.**.api
```

### 2. 事件发布（Event Publication Registry）

模块间通过事件解耦通信，Spring Modulith 内置**事件发布注册表**保证可靠投递：

```java
// 发布事件
@Service
class OrderService {
    private final ApplicationEventPublisher events;

    void placeOrder(Order order) {
        // ...业务逻辑
        events.publishEvent(new OrderPlaced(order.getId(), order.getTotal()));
    }
}

// 监听事件（跨模块）
@Service
class NotificationService {
    @TransactionalEventListener
    void on(OrderPlaced event) {
        // 发送通知
    }
}
```

事件发布注册表确保事件至少投递一次（Exactly-Once Delivery）：

```yaml
spring:
  modulith:
    events:
      jdbc:
        schema-initialization:
          enabled: true
```

### 3. 事件重放（Event Replay）

当新模块加入或修复 Bug 后，可以重放历史事件：

```java
@Test
void replayCompletedOrders() {
    CompletedOrders completed = new CompletedOrders();
    EventPublicationRegistry registry = ...;

    registry.findUncompletedPublications()
        .forEach(pub -> completed.handle((OrderPlaced) pub.getEvent()));
}
```

## 独立模块测试

每个模块可以**独立启动 Spring 上下文**进行集成测试，只加载该模块相关 Bean，大幅提升测试速度：

```java
@ApplicationModuleTest
class OrderModuleTest {

    @Autowired
    OrderService orderService;

    @Test
    void shouldPlaceOrder() {
        // 只加载 order 模块的 Bean
        orderService.placeOrder(new Order(...));
    }
}

// 指定单个模块
@ApplicationModuleTest(module = "payment")
class PaymentModuleTest {
    // 只加载 payment 模块
}
```

### 测试隔离

```yaml
# 默认情况下，@ApplicationModuleTest 会：
# 1. 禁用所有不属于该模块的自动配置
# 2. 不加载其他模块的 Bean
# 3. 可以通过 @ModuleTestExecution 自定义加载策略
```

## 模块级可观测性

Spring Modulith 通过 Micrometer 提供模块级指标，**无需拆分为微服务**即可获得类似微服务的可观测性：

### 模块入口/出口指标

```yaml
management:
  endpoints:
    web:
      exposure:
        include: modules
management.endpoint.modules.enabled: true
```

访问 `/actuator/modules` 返回模块依赖图：

```json
{
  "order": {
    "dependencies": ["payment", "notification"],
    "basePackage": "com.myapp.order"
  },
  "payment": {
    "dependencies": [],
    "basePackage": "com.myapp.payment"
  }
}
```

### 事件追踪

```yaml
management.tracing.enabled: true
# 事件发布和消费自动被追踪为 Span
# 每个事件生命周期包含：publish → completion → consumption
```

## 从模块到微服务的迁移路径

模块化单体最大的优势是**迁移路径清晰**：

```
步骤 1: 识别模块边界   →  步骤 2: 验证依赖方向   →  步骤 3: 事件解耦
                                                         ↓
步骤 6: 部署独立服务   ←  步骤 5: 抽取 API 契约   ←  步骤 4: 独立测试
```

```java
// 步骤 5-6：将 order 模块抽取为独立微服务
// 1. 创建单独的 spring-boot 项目，复制 order 模块代码
// 2. 将事件发布改为 REST/gRPC/消息队列调用
// 3. 注册到 Nacos/Eureka 完成服务发现
```

## Maven 依赖

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.modulith</groupId>
            <artifactId>spring-modulith-bom</artifactId>
            <version>2.1.0-RC1</version>
            <scope>import</scope>
            <type>pom</type>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- 核心依赖 -->
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-starter-core</artifactId>
    </dependency>

    <!-- 验证 + 测试 -->
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## 注意事项

1. **不是微服务替代品**：模块化单体适合中小团队，如果团队规模超过"两个比萨"或需要独立扩缩容，仍应考虑微服务
2. **模块粒度不宜过细**：建议 5-10 个模块为合理范围，过多模块增加管理复杂度
3. **事件风暴前置**：实施前应通过 Event Storming 明确聚合边界和事件类型
4. **循环依赖检测**：`@ApplicationModule(allowedDependencies = {...})` 限制明确，自动防止循环依赖
5. **JDK 17+ 必需**：Spring Modulith 2.x 要求 Spring Boot 3.x+ 和 JDK 17+
6. **事件序列化**：跨模块事件建议使用 JSON/Protobuf 序列化，为后续微服务拆分做准备
7. **性能考量**：事件发布注册表依赖数据库表，高频场景（>1000 TPS）建议配合缓存或改用消息队列
