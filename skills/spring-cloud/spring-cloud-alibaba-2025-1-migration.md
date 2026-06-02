---
name: spring-cloud-alibaba-2025-1-migration
description: Spring Cloud Alibaba 2025.1.0 新特性：Spring Boot 4.x 支持、Nacos 配置迁移、Sentinel Jackson 3、Seata WebFlux
tags: [spring-cloud, alibaba, nacos, sentinel, seata, migration, spring-boot4]
---

## 概述

Spring Cloud Alibaba 2025.1.0.0（2026-02-06 发布）是一次**大版本升级**，对应 Spring Boot 4.x + Spring Cloud 2025.1.x 生态。主要变化包括：移除 Nacos bootstrap 配置方式、全面支持 Spring Boot 4、Sentinel 适配 Jackson 3、Seata 支持 WebFlux 等。

> 旧版本用户升级到 SCA 2025.1.x 时需要**特别关注** Nacos 配置迁移和依赖版本变化。

## 版本对应关系

| 组件 | 旧版本（2025.0.x） | 新版本（2025.1.0） |
|------|-------------------|-------------------|
| Spring Boot | 3.5.x | **4.0.0** |
| Spring Cloud | 2025.0.0 | **2025.1.0** |
| Spring Cloud Dependencies Parent | 4.3.0 | **5.0.0** |
| Nacos Client | 3.0.3 | **3.1.1** |
| Sentinel | 1.8.x | 1.8.x（适配 Jackson 3） |

## 关键变更

### 1. 移除 Nacos Bootstrap 支持（⚠️ 破坏性变更）

**变更**：Nacos 模块不再支持 `bootstrap.properties` / `bootstrap.yml` 方式加载配置，必须使用 `spring.config.import`。

```properties
# ❌ 旧方式（bootstrap.properties — 不再支持）
spring.cloud.nacos.config.server-addr=127.0.0.1:8848
spring.cloud.nacos.config.file-extension=yaml

# ✅ 新方式（application.properties）
spring.config.import=nacos:127.0.0.1:8848
spring.cloud.nacos.config.file-extension=yaml

# 带命名空间和分组
spring.config.import=nacos:127.0.0.1:8848?namespace=dev&group=MY_GROUP
```

**迁移步骤**：

1. 删除 `bootstrap.properties` / `bootstrap.yml`
2. 删除 `spring-cloud-starter-bootstrap` 依赖
3. 在 `application.properties`/`application.yml` 中添加 `spring.config.import=nacos:...`
4. 原 `bootstrap.properties` 中的 Nacos 配置项需迁移到 `application.properties`

```xml
<!-- ❌ 移除旧依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
```

### 2. 日志敏感信息脱敏

Nacos 配置日志中自动屏蔽敏感字段（密码、密钥等），无需额外配置：

```properties
# 默认已启用，可关闭
spring.cloud.nacos.config.log-mask-enabled=true
```

```text
# Before: password=my-secret-pass-123
# After:  password=***
```

### 3. Sentinel Jackson 3.x 支持

Sentinel 模块适配 Jackson 3.0，解决 `jackson-databind` 升级问题：

```java
// 规则持久化 — Jackson 3 兼容
@Bean
public SentinelDataSource sentinelDataSource() {
    // 使用 Jackson 3 ObjectMapper
    ObjectMapper mapper = JsonMapper.builder()
        .build();
    return new NacosDataSource<>(...);
}
```

### 4. RocketMQ Spring Boot 4 支持

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rocketmq</artifactId>
    <version>2025.1.0.0</version>
</dependency>
```

```java
// Spring Boot 4 + RocketMQ（API 不变）
@Service
public class OrderMessageService {

    @Autowired
    private StreamBridge streamBridge;

    public void sendOrderCreated(OrderCreatedEvent event) {
        streamBridge.send("order-output", event);
    }
}
```

### 5. Seata WebFlux 支持（实验性）

Seata 模块新增对 WebFlux 响应式编程的支持：

```java
// 响应式事务 — 使用 WebFlux
@Configuration
@EnableTransactionManagement
public class SeataWebFluxConfig {

    @Bean
    public GlobalTransactionalReactiveAspect globalTransactionalAspect() {
        return new GlobalTransactionalReactiveAspect();
    }
}

// 响应式 Service
@Service
public class ReactiveOrderService {

    @GlobalTransactional
    public Mono<Void> createOrderReactive(OrderDTO order) {
        return Mono.fromRunnable(() -> {
            // 分布式事务逻辑
            orderDao.insert(order);
            accountService.deduct(order.getUserId(), order.getAmount());
            inventoryService.deduct(order.getProductId(), order.getQuantity());
        });
    }
}
```

### 6. 依赖版本升级

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2025.1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

```properties
# 关键依赖版本变化
# Nacos Client: 3.0.3 → 3.1.1
# SchedulerX Worker: 1.13.1 → 1.13.3
# Seata: 保持兼容（新增 WebFlux 支持）
```

## 注意事项

1. **Nacos bootstrap 迁移是硬性要求**：2025.1.0 不再接受 `bootstrap.properties` 中的 Nacos 配置，项目启动时会直接报错，非渐进式废弃
2. **Spring Boot 4 兼容性**：2025.1.0 要求 Spring Boot 4.0.0+，Spring Boot 3.x 项目请继续使用 2025.0.x 或 2023.0.x 分支
3. **Seata WebFlux 为实验性**：生产环境建议先在小流量验证，核心业务仍使用传统 Servlet 栈的 Seata 集成
4. **Nacos 客户端升级**：从 3.0.3 升级到 3.1.1，注意 `nacos-client` 内部协议和 gRPC 连接的变更（参见 Nacos 3.1.x 升级文档）
5. **Jackson 3 兼容性**：如果项目中使用了自定义 `Jackson2TypeHandler` 或 Jackson 2.x 特定 API，需升级到 Jackson 3 等价方法
