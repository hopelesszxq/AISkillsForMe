---
name: cqrs-pattern-java
description: CQRS 命令查询职责分离模式 Java/Spring Boot 实现：命令总线、查询模型、事件溯源整合与代码示例
tags: [patterns, cqrs, architecture, java, spring-boot, axon, event-sourcing]
---

## CQRS 模式概述

CQRS（Command Query Responsibility Segregation）将系统的**写操作（Command）**和**读操作（Query）**分离到不同的模型和服务中，允许两端独立优化。

### 适用场景

| 场景 | 说明 |
|------|------|
| 读写负载差异大 | 读请求远多于写请求，或反之 |
| 复杂业务逻辑 | Command 侧需要严格的领域校验 |
| 查询多样化 | Query 侧需要灵活的投影和聚合 |
| 团队分工 | 读写两端可由不同团队独立演进 |

### 核心原则

- **Command**：修改状态，不返回数据（只返回成功/失败）
- **Query**：读取数据，不修改状态
- **模型分离**：Command 用领域模型（DDD Aggregate），Query 用扁平化投影

## 1. Spring Boot CQRS 基础实现

### 命令总线（Command Bus）

```java
// 命令接口
public interface Command<T extends CommandResult> {
}

// 命令处理器
public interface CommandHandler<C extends Command<R>, R extends CommandResult> {
    R handle(C command);
}

// 命令总线
@Component
public class CommandBus {
    private final Map<Class<?>, CommandHandler<?, ?>> handlers = new ConcurrentHashMap<>();
    
    public <C extends Command<R>, R extends CommandResult> void register(
            Class<C> commandType, CommandHandler<C, R> handler) {
        handlers.put(commandType, handler);
    }
    
    @SuppressWarnings("unchecked")
    public <R extends CommandResult> R dispatch(Command<R> command) {
        CommandHandler<Command<R>, R> handler = 
            (CommandHandler<Command<R>, R>) handlers.get(command.getClass());
        if (handler == null) {
            throw new IllegalArgumentException("No handler for: " + command.getClass());
        }
        return handler.handle(command);
    }
}
```

### 具体命令与处理器

```java
// 创建订单命令
@Data
public class CreateOrderCommand implements Command<OrderResult> {
    private String customerId;
    private List<OrderItem> items;
    private Address shippingAddress;
}

// 命令处理器
@Component
public class CreateOrderHandler implements CommandHandler<CreateOrderCommand, OrderResult> {
    
    @Autowired
    private OrderRepository orderRepository;
    @Autowired
    private EventPublisher eventPublisher;
    
    @Transactional
    @Override
    public OrderResult handle(CreateOrderCommand command) {
        // 1. 领域校验
        if (command.getItems().isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one item");
        }
        
        // 2. 创建聚合
        Order order = Order.create(command.getCustomerId(), command.getItems());
        order.setShippingAddress(command.getShippingAddress());
        
        // 3. 持久化
        orderRepository.save(order);
        
        // 4. 发布事件
        eventPublisher.publish(new OrderCreatedEvent(order.getId(), order.getTotalAmount()));
        
        return new OrderResult(order.getId(), "SUCCESS");
    }
}
```

### 查询模型（查询侧）

```java
// 独立的查询 DTO
@Data
public class OrderSummary {
    private String orderId;
    private String customerName;
    private BigDecimal totalAmount;
    private String status;
    private LocalDateTime createdAt;
}

// 查询服务 — 单独的表/投影
@RestController
@RequestMapping("/api/queries/orders")
public class OrderQueryController {
    
    @Autowired
    private OrderProjectionRepository projectionRepo;
    
    @GetMapping("/{orderId}")
    public OrderSummary getOrderSummary(@PathVariable String orderId) {
        return projectionRepo.findById(orderId)
            .orElseThrow(() -> new NotFoundException("Order not found"));
    }
    
    @GetMapping
    public Page<OrderSummary> listOrders(
            @RequestParam String customerId,
            Pageable pageable) {
        return projectionRepo.findByCustomerId(customerId, pageable);
    }
}
```

## 2. 使用 Axon Framework（推荐生产方案）

Axon Framework 提供开箱即用的 CQRS + Event Sourcing 架构。

### Maven 依赖

```xml
<dependency>
    <groupId>org.axoniq</groupId>
    <artifactId>axon-spring-boot-starter</artifactId>
    <version>4.9.x</version>
</dependency>
```

### 命令模型（写侧）

```java
// 聚合根
@Aggregate
public class OrderAggregate {
    
    @AggregateIdentifier
    private String orderId;
    private OrderStatus status;
    private BigDecimal totalAmount;
    
    @CommandHandler
    public OrderAggregate(CreateOrderCommand command) {
        // 校验
        if (command.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        // 应用事件
        apply(new OrderCreatedEvent(command.getOrderId(), command.getAmount()));
    }
    
    @EventSourcingHandler
    public void on(OrderCreatedEvent event) {
        this.orderId = event.getOrderId();
        this.status = OrderStatus.CREATED;
        this.totalAmount = event.getAmount();
    }
}
```

### 投影（读侧）

```java
// 事件处理器，维护读模型
@Component
public class OrderProjection {
    
    @Autowired
    private OrderSummaryRepository repository;
    
    @EventHandler
    public void on(OrderCreatedEvent event) {
        OrderSummary summary = new OrderSummary();
        summary.setOrderId(event.getOrderId());
        summary.setTotalAmount(event.getAmount());
        summary.setStatus("CREATED");
        summary.setCreatedAt(Instant.now());
        repository.save(summary);
    }
    
    @EventHandler
    public void on(OrderShippedEvent event) {
        repository.findById(event.getOrderId()).ifPresent(summary -> {
            summary.setStatus("SHIPPED");
            repository.save(summary);
        });
    }
}
```

## 3. 事件溯源整合

当使用 Event Sourcing 时，聚合状态由事件流重建：

```java
// 事件存储配置
@Configuration
public class AxonConfig {
    
    @Bean
    public EventStorageEngine eventStorageEngine(EntityManager entityManager) {
        // JPA 事件存储（也可用 MongoDB、JDBC）
        return JpaEventStorageEngine.builder()
            .entityManagerProvider(new SimpleEntityManagerProvider(entityManager))
            .build();
    }
}

// 读模型通过事件流重建
@Component
public class OrderHistoryProjection {
    
    @EventHandler
    public void on(OrderCreatedEvent event, @Timestamp Instant timestamp) {
        // 写入历史表
    }
    
    // Reset 处理器允许完全重建
    @EventHandler
    public void on(OrderShippedEvent event) {
        // 更新状态
    }
}
```

## 4. 分离数据库架构

```
┌──────────────────┐      ┌──────────────────┐
│   Command DB     │      │    Query DB      │
│  (3NF/DDD模型)   │      │  (反范式投影)     │
│                  │      │                  │
│  orders          │      │  order_summary   │
│  order_items     │      │  customer_view   │
│  customers       │      │  report_mv       │
└────────┬─────────┘      └────────┬─────────┘
         │                         │
         └───────── 同步 ──────────┘
              (事件驱动)
```

## 注意事项

### 1. 最终一致性
- 写侧和读侧之间存在延迟，前端需处理"写入后查询不到"的情况
- 使用 WebSocket/SSE 推送实时更新，或轮询重试策略

### 2. 不需要 CQRS 的场景
- 简单 CRUD 系统：CQRS 徒增复杂度
- 读写模型差异不大时：额外维护双倍代码
- 团队不熟悉 DDD 时：应先掌握领域驱动设计基础

### 3. 事务边界
- Command 侧使用数据库事务
- Query 侧不要求强一致性，允许最终一致
- 使用 Outbox 模式保证事件可靠发布

### 4. 版本演进
- 事件定义一旦发布不可修改（Event Schema 需兼容）
- 投影可以随时新增或重建
- 使用 `@ProcessingGroup` 隔离不同投影的消费进度
