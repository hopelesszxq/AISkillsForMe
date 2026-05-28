---
name: microservice-patterns
description: 微服务架构通用模式：Saga、CQRS、事件溯源、绞杀者模式等
tags: [architecture, microservice, pattern, ddd]
---

## 服务注册与发现

### 三种模式对比

| 模式 | 工具 | 特点 | 适用场景 |
|------|------|------|----------|
| **客户端发现** | Nacos / Eureka | 客户端直连服务实例，负载均衡集成在客户端 | 小型微服务、Spring Cloud 体系 |
| **服务端发现** | Kong / Envoy / Nginx+Consul | 客户端请求网关，网关路由到服务实例 | 多语言异构、Kubernetes 环境 |
| **K8s DNS** | CoreDNS | 基于 Service 名称 DNS 解析，K8s 原生 | 云原生部署 |

### 健康检查最佳实践

```yaml
# Nacos 健康检查配置
spring:
  cloud:
    nacos:
      discovery:
        heart-beat-interval: 5000      # 心跳间隔
        heart-beat-timeout: 15000      # 心跳超时
        ip-delete-timeout: 30000       # 实例删除超时
        # 元数据，用于灰度发布
        metadata:
          version: 1.0.0
          gray-version: ${GRAY_VERSION:}
```

## 配置中心模式

### 多环境管理

```yaml
# Nacos Data ID 命名规范
# {prefix}-{profile}.{file-extension}
# 例如：
nacos-config:
  - nacos-config-dev.yaml      # 开发环境
  - nacos-config-test.yaml     # 测试环境
  - nacos-config-prod.yaml     # 生产环境
```

### 配置刷新监听

```java
@Component
public class DynamicConfigListener {

    @Autowired
    private NacosConfigManager configManager;

    @PostConstruct
    public void init() {
        configManager.getConfigService().addListener(
            "dynamic-config.yaml", "DEFAULT_GROUP", new Listener() {
                @Override
                public Executor getExecutor() {
                    return Executors.newSingleThreadExecutor();
                }

                @Override
                public void receiveConfigInfo(String configInfo) {
                    // 配置变更时重新加载
                    System.out.println("配置已更新: " + configInfo);
                }
            }
        );
    }
}
```

## 服务容错模式（Resiliency Patterns）

### 1. 熔断器（Circuit Breaker）

```yaml
# Resilience4j 配置
resilience4j:
  circuitbreaker:
    instances:
      user-service:
        register-health-indicator: true
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10000
        permitted-number-of-calls-in-half-open-state: 3
        automatic-transition-from-open-to-half-open-enabled: true
```

### 2. 隔板模式（Bulkhead）

```yaml
resilience4j:
  bulkhead:
    instances:
      user-service:
        max-concurrent-calls: 10
        max-wait-duration: 500ms
  # 信号量隔板（轻量，适合非阻塞调用）
  # 线程池隔板（重量，适合阻塞调用）
  thread-pool-bulkhead:
    instances:
      order-service:
        max-thread-pool-size: 4
        core-thread-pool-size: 2
        queue-capacity: 10
```

### 3. 重试模式（Retry）

```yaml
resilience4j:
  retry:
    instances:
      user-service:
        max-attempts: 3
        wait-duration: 1s
        retry-exceptions:
          - org.springframework.web.client.HttpServerErrorException
          - java.net.SocketTimeoutException
        ignore-exceptions:
          - org.springframework.web.client.HttpClientErrorException
```

## Saga 分布事务模式

### 编排型 Saga（Choreography）

```java
// 事件驱动，每个服务发布事件，其他服务监听
// 订单服务 → 发布 OrderCreatedEvent
// 库存服务 → 监听事件 → 扣库存 → 发布 InventoryDeductedEvent
// 账户服务 → 监听事件 → 扣余额 → 发布 BalanceDebitedEvent
// 任何服务失败 → 发布补偿事件

// 事件定义
public class OrderCreatedEvent {
    private String orderId;
    private Long userId;
    private Long productId;
    private Integer count;
    private BigDecimal amount;
}

// 库存服务监听器
@Component
public class InventoryEventListener {
    @EventListener
    @Transactional
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            inventoryService.deduct(event.getProductId(), event.getCount());
            eventPublisher.publish(new InventoryDeductedEvent(event.getOrderId()));
        } catch (Exception e) {
            eventPublisher.publish(new OrderCompensationEvent(event.getOrderId(), "库存不足"));
        }
    }
}
```

### 编排器型 Saga（Orchestration）

```java
// 中央协调器服务管理整个事务流程
@Component
public class OrderSagaOrchestrator {

    @Autowired
    private SagaStateMachine stateMachine;

    @GlobalTransactional
    public void createOrder(OrderDTO order) {
        // 1. 创建订单（待支付状态）
        orderService.create(order);

        // 2. 扣减库存（可重试）
        stateMachine.send(OrderEvent.DEDUCT_INVENTORY);
        inventoryService.deduct(order.getProductId(), order.getCount());

        // 3. 扣减余额
        stateMachine.send(OrderEvent.DEBIT_BALANCE);
        accountService.debit(order.getUserId(), order.getAmount());

        // 4. 完成
        stateMachine.send(OrderEvent.COMPLETE);
    }
}
```

## CQRS（命令查询职责分离）

```java
// 命令端（Command）：处理写操作
@RestController
@RequestMapping("/api/v1/orders")
public class OrderCommandController {

    @PostMapping
    public Result<String> createOrder(@RequestBody CreateOrderCommand command) {
        String orderId = orderCommandService.handle(command);
        return Result.success(orderId);
    }
}

// 查询端（Query）：处理读操作（可走缓存/物化视图）
@RestController
@RequestMapping("/api/v1/orders")
public class OrderQueryController {

    @GetMapping("/stats")
    public Result<OrderStats> getOrderStats(OrderStatsQuery query) {
        // 查缓存或专门的统计表，不查主库
        return Result.success(orderStatsService.query(query));
    }
}

// 事件同步（命令端 → 查询端）
@Component
public class OrderEventProcessor {
    @EventListener
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event) {
        // 更新物化视图或缓存
        orderStatsRepo.updateStats(event.getUserId(), event.getAmount());
    }
}
```

## 绞杀者模式（Strangler Fig）

```java
// 渐进式重构：旧系统和新系统共存，通过路由逐步迁移

// 网关层路由规则
@Component
public class StranglerRoutingFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();

        // 已经迁移到新系统的路由
        if (isMigrated(path)) {
            // 改写路由到新服务
            exchange.getAttributes().put("migrated", true);
        }

        return chain.filter(exchange);
    }

    private boolean isMigrated(String path) {
        // 基于 feature flag 或百分比灰度
        return featureFlag.isEnabled(path) || ThreadLocalRandom.current().nextDouble() < 0.3;
    }
}
```

## 事件源（Event Sourcing）

```java
// 不存当前状态，只存事件流
public class AccountEventStore {

    @Autowired
    private EventRepository eventRepository;

    public void saveEvent(AccountEvent event) {
        // 追加写入，永不修改历史
        eventRepository.save(event);
    }

    public Account aggregate(String accountId) {
        // 回放所有事件重建当前状态
        List<AccountEvent> events = eventRepository.findByAccountIdOrderByVersion(accountId);
        Account account = new Account();
        for (AccountEvent event : events) {
            account.apply(event);
        }
        return account;
    }
}

class Account {
    private BigDecimal balance = BigDecimal.ZERO;

    public void apply(AccountEvent event) {
        switch (event.getType()) {
            case ACCOUNT_CREATED -> this.balance = event.getInitialBalance();
            case MONEY_DEPOSITED -> this.balance = this.balance.add(event.getAmount());
            case MONEY_WITHDRAWN -> this.balance = this.balance.subtract(event.getAmount());
        }
    }
}
```

## 微服务拆分原则

### 按业务能力拆分

```
用户服务 (user-service)         → 注册、登录、权限
订单服务 (order-service)        → 订单 CRUD、状态流转
商品服务 (product-service)      → 商品信息、库存
支付服务 (payment-service)      → 支付渠道对接、账单
通知服务 (notification-service) → 短信、邮件、推送
```

### 拆分策略决策树

```
该模块是否经常独立变更？
  ├── 是 → 拆分为独立服务
  └── 否 → 是否对性能和扩展性有特殊要求？
       ├── 是 → 拆分为独立服务
       └── 否 → 是否被多个团队维护？
            ├── 是 → 拆分为独立服务
            └── 否 → 保留在现有服务中
```

### 数据库拆分原则

| 策略 | 说明 | 挑战 |
|------|------|------|
| **Database per Service** | 每个服务私有数据库 | 跨服务查询困难，分布式事务 |
| **Shared Database** | 多个服务共享数据库（反模式） | 耦合高，不推荐 |
| **Saga** | 事件驱动的最终一致性 | 最终一致性容忍度 |
| **API Composition** | 聚合层对多个服务查询做组合 | 性能开销 |

## 注意事项

1. **不要过早拆分**：单体优先，拆分理由是「独立部署需求」而非「技术时髦」
2. **避免分布式事务**：能用最终一致性解决的不要用强一致性
3. **API 版本管理**：`/api/v1/orders` 不要轻易修改接口，向后兼容优先
4. **服务间通讯超时必设**：所有远程调用必须设置 connectTimeout 和 readTimeout
5. **幂等设计必须做**：至少一次 + 幂等 = 恰好一次
6. **可观测性三件套**：日志（结构化）+ 指标（Micrometer）+ 链路追踪（SkyWalking/Zipkin）
7. **配置环境分离**：配置文件中不要硬编码环境信息，使用占位符
8. **独立数据存储**：不要跨服务访问数据库，只能通过 API 调用
