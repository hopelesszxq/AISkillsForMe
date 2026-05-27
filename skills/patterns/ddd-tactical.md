---
name: ddd-tactical
description: DDD 战术模式实战（聚合、值对象、领域事件、仓储）
tags: [ddd, architecture, domain-driven-design, pattern]
---

## 聚合（Aggregate）设计原则

### 聚合边界划分

```java
// ❌ 贫血模型 - 反模式
public class Order {
    private Long id;
    private OrderStatus status;
    private BigDecimal totalAmount;
    // getter/setter 暴露全部内部状态
    public void setStatus(OrderStatus status) { this.status = status; }
}

// ✅ 充血模型 - 聚合根封装业务逻辑
public class Order {
    private OrderId id;
    private OrderStatus status;
    private List<OrderLine> lines;
    private PaymentInfo paymentInfo;

    // 业务方法封装变化
    public void addProduct(Product product, int quantity) {
        if (status != OrderStatus.PENDING) {
            throw new OrderAlreadySubmittedException(id);
        }
        OrderLine line = new OrderLine(product, quantity);
        lines.add(line);
    }

    public PaymentRecord submit() {
        if (lines.isEmpty()) throw new EmptyOrderException();
        this.status = OrderStatus.SUBMITTED;
        // 领域事件
        registerEvent(new OrderSubmittedEvent(this.id, this.totalAmount()));
        return new PaymentRecord(this.id, this.totalAmount());
    }
}
```

### 聚合设计四原则

| 原则 | 说明 | 反例 |
|---|---|---|
| **一致性边界** | 聚合内强一致性，聚合间最终一致性 | 跨聚合事务 |
| **小聚合** | 加载根后只加载必要的内部对象 | 加载整个对象树 |
| **通过ID引用** | 聚合间通过ID而非对象引用 | `Order.customer` 直接引用Customer对象 |
| **最终一致性** | 跨聚合变更通过领域事件同步 | 跨表分布式事务 |

## 值对象（Value Object）

### 不可变设计

```java
// 值对象 - 无身份标识，不可变
@Value
public class Money {
    BigDecimal amount;
    Currency currency;

    // 工厂方法 + 校验
    public static Money of(BigDecimal amount, Currency currency) {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("金额不能为负");
        }
        return new Money(amount, currency);
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException();
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    // 必须实现 equals/hashCode（基于所有字段）
}
```

### 值对象 vs DTO 区别

| | 值对象 | DTO |
|---|---|---|
| **身份** | 无身份（由属性值定义相等性） | 有身份（通常有 ID） |
| **可变性** | 不可变 | 可变（通常被 setter） |
| **行为** | 包含业务方法 | 纯数据容器 |
| **生命周期** | 属于聚合 | 独立于聚合 |
| **序列化** | 可以序列化 | 专为序列化设计 |

## 领域事件（Domain Event）

### 事件定义与发布

```java
// 1. 定义事件
@Value
public class OrderShippedEvent {
    OrderId orderId;
    ShippingInfo shippingInfo;
    Instant occurredAt;

    public OrderShippedEvent(OrderId orderId, ShippingInfo info) {
        this.orderId = orderId;
        this.shippingInfo = info;
        this.occurredAt = Instant.now();
    }
}

// 2. 聚合根中注册事件
public abstract class AggregateRoot<ID> {
    @Setter(AccessLevel.PROTECTED)
    private List<Object> domainEvents = new ArrayList<>();

    protected void registerEvent(Object event) {
        domainEvents.add(event);
    }

    public List<Object> clearEvents() {
        List<Object> events = List.copyOf(domainEvents);
        domainEvents.clear();
        return events;
    }
}

// 3. 仓储保存后发布事件（Spring 实现）
@Component
@RequiredArgsConstructor
public class OrderRepository {
    private final JpaRepository jpa;
    private final ApplicationEventPublisher publisher;

    @Transactional
    public void save(Order order) {
        jpa.save(order);
        order.clearEvents().forEach(publisher::publishEvent);
    }
}

// 4. 事件处理器
@Component
public class OrderShippedHandler {
    @EventListener
    public void on(OrderShippedEvent event) {
        // 发送通知、更新库存、触发物流等
        notificationService.sendShippingUpdate(event.getOrderId());
        inventoryService.reserveStock(event.getOrderId());
    }
}
```

### 事件版本管理

```java
// 事件带上 version 字段，方便向后兼容
@Value
public class OrderShippedEvent {
    String eventType = "order.shipped.v2"; // 版本标识
    OrderId orderId;
    ShippingInfo shippingInfo;
    Instant occurredAt;
}
```

## 仓储（Repository）

### 接口定义规范

```java
// 仓储接口 - 面向聚合根，使用集合语义
public interface OrderRepository {
    // 查询返回完整聚合
    Optional<Order> findById(OrderId id);

    // 保存完整聚合
    void save(Order order);

    // 删除聚合
    void delete(OrderId id);
}

// 按业务维度提供的查询接口（非必须放在仓储中）
public interface OrderQueryRepository {
    Page<OrderSummary> findByUserId(Long userId, Pageable pageable);
    Optional<OrderDetail> findDetail(OrderId id);
}
```

## 应用服务 vs 领域服务

```java
// 应用服务 - 编排、事务、权限、DTO转换
@Service
@RequiredArgsConstructor
public class OrderApplicationService {
    private final OrderRepository orderRepo;
    private final ProductService productService;
    private final DomainEventPublisher eventPublisher;

    @Transactional
    public OrderSubmitResult submitOrder(SubmitOrderCommand cmd) {
        // 1. 权限校验
        // 2. 加载领域对象
        Product product = productService.findById(cmd.getProductId());
        // 3. 委托给领域模型
        Order order = new Order(cmd.getUserId());
        order.addProduct(product, cmd.getQuantity());
        OrderSubmitResult result = order.submit();
        // 4. 保存
        orderRepo.save(order);
        return result;
    }
}

// 领域服务 - 跨聚合、无状态的业务逻辑
public class ShippingDomainService {
    public ShippingPlan calculateShipping(Order order, Address address) {
        // 跨聚合的业务逻辑：根据订单商品、地址计算运费
        Money baseCost = Money.of(new BigDecimal("10.00"), Currency.CNY);
        if (order.totalAmount().compareTo(Money.of(new BigDecimal("99"), Currency.CNY)) > 0) {
            return new ShippingPlan(baseCost, true); // 免运费
        }
        return new ShippingPlan(baseCost, false);
    }
}
```

## 六边形架构（端口-适配器）

```
src/main/java/com/example/
├── domain/              # 领域层（核心）
│   ├── model/           # 聚合、实体、值对象
│   ├── service/         # 领域服务
│   ├── event/           # 领域事件
│   └── repository/      # 仓储接口（端口）
├── application/         # 应用层
│   ├── service/         # 应用服务
│   ├── dto/             # DTO
│   └── command/         # 命令对象
├── infrastructure/      # 基础设施层（适配器）
│   ├── persistence/     # JPA/MyBatis 实现
│   ├── messaging/       # 消息队列实现
│   └── client/          # 外部API客户端
└── interfaces/          # 接口层（适配器）
    ├── rest/            # REST 控制器
    ├── grpc/            # gRPC 服务
    └── scheduler/       # 定时任务
```

## CQRS 模式要点

```java
// 命令侧（写）
public interface OrderCommandService {
    OrderId createOrder(CreateOrderCommand cmd);
    void addItem(AddItemCommand cmd);
    void submitOrder(OrderId id);
}

// 查询侧（读） - 独立数据模型
@Value
public class OrderListView {
    OrderId id;
    String customerName;
    int itemCount;
    String status;
    String createdAt;
}

@Repository
public class OrderListViewRepository {
    // 直接从物化视图或读库查询
    public Page<OrderListView> findByCustomer(Long customerId, Pageable pageable) {
        return jdbc.query(
            "SELECT * FROM order_list_view WHERE customer_id = ?", pageable);
    }
}
```

## 注意事项

1. **不要过度设计**：简单的 CRUD 应用不需要 DDD，微服务内部边界不清晰时不要强行拆分聚合
2. **聚合大小权衡**：聚合越大一致性越强但并发越低；聚合越小并发越高但最终一致性复杂度增加
3. **领域事件幂等性**：事件处理必须是幂等的，同个事件可能被重复消费
4. **ORM 阻抗失衡**：JPA 的懒加载、级联操作与聚合边界可能冲突，用好 MyBatis 作为替代
5. **事件存储**：生产环境建议使用事件表持久化领域事件，配合发件箱模式（Outbox）确保可靠发布
6. **避免贫血模型反模式**：实体不应只有 getter/setter，业务逻辑应封装在实体方法中
