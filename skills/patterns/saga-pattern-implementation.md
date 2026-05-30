---
name: saga-pattern-implementation
description: 微服务 Saga 分布式事务模式实现：编排(Orchestration)与编舞(Choreography)两种方式的深度实践与源码示例
tags: [architecture, microservice, pattern, saga, distributed-transaction, transaction]
---

## 概述

Saga 模式是微服务架构中处理**分布式事务**的核心模式。它将一个大事务拆分为一系列本地事务，每个本地事务完成后触发下一个事务，若某一步失败则执行**补偿事务**（Compensating Transaction）来回滚。Saga 有两种实现方式：编排（Orchestration）和编舞（Choreography）。

## 两种模式对比

| 特性 | 编排 (Orchestration) | 编舞 (Choreography) |
|---|---|---|
| 控制中心 | 有（Saga Orchestrator） | 无（去中心化） |
| 复杂度 | 中间件集中管理 | 分散在各服务 |
| 循环依赖 | 无 | 可能有 |
| 调试难度 | 较低（单一入口） | 较高（分散） |
| 适合场景 | 复杂事务（3步以上） | 简单线性流程 |
| 消息依赖 | 命令消息 | 事件消息 |

## 编舞模式（Choreography）

### 工作原理

```
订单服务 → 发布 "订单创建" 事件
    ↓
库存服务 → 消费事件 → 扣减库存 → 发布 "库存已扣减" 事件
    ↓
支付服务 → 消费事件 → 扣款 → 发布 "支付成功" 事件
    ↓
通知服务 → 消费事件 → 发送通知
```

### 实现示例（基于 RabbitMQ）

```java
// 订单服务 - 创建订单并发布事件
@Service
public class OrderService {

    private final RabbitTemplate rabbitTemplate;

    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. 创建订单（状态：待确认）
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setAmount(request.getAmount());
        order.setStatus(OrderStatus.PENDING);
        order = orderRepository.save(order);

        // 2. 发布订单创建事件
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(), order.getUserId(), order.getAmount());
        rabbitTemplate.convertAndSend("order.exchange", "order.created", event);
        
        return order;
    }
}
```

```java
// 库存服务 - 处理库存扣减
@Service
@Slf4j
public class InventoryService {

    @RabbitListener(bindings = @QueueBinding(
        value = @Queue("inventory.deduct.queue"),
        exchange = @Exchange("order.exchange"),
        key = "order.created"
    ))
    @Transactional
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            // 扣减库存
            inventoryRepository.deduct(event.getUserId(), event.getAmount());
            log.info("Inventory deducted for order: {}", event.getOrderId());
            
            // 发布库存已扣减事件
            InventoryDeductedEvent deductedEvent = new InventoryDeductedEvent(
                event.getOrderId(), true);
            rabbitTemplate.convertAndSend(
                "inventory.exchange", "inventory.deducted", deductedEvent);
                
        } catch (Exception e) {
            log.error("Inventory deduction failed: {}", e.getMessage());
            // 发布扣减失败事件 → 触发补偿
            InventoryDeductedEvent failedEvent = new InventoryDeductedEvent(
                event.getOrderId(), false);
            rabbitTemplate.convertAndSend(
                "inventory.exchange", "inventory.deducted", failedEvent);
        }
    }
}
```

### 编舞模式补偿实现

```java
// 订单服务 - 监听失败事件执行补偿
@Service
public class OrderSagaCompensation {

    // 监听库存扣减失败
    @RabbitListener(queues = "order.compensation.queue")
    @Transactional
    public void handleInventoryFailed(InventoryDeductedEvent event) {
        if (!event.isSuccess()) {
            log.warn("Compensating order: {}", event.getOrderId());
            // 将订单状态改为"失败"
            orderRepository.updateStatus(event.getOrderId(), OrderStatus.FAILED);
            // 取消订单
            orderRepository.cancelOrder(event.getOrderId());
        }
    }

    // 监听支付失败（需补偿库存）
    @RabbitListener(queues = "order.payment.failed.queue")
    @Transactional
    public void handlePaymentFailed(PaymentResultEvent event) {
        log.warn("Payment failed, compensating inventory: {}", event.getOrderId());
        // 补偿：恢复库存
        inventoryService.restoreInventory(event.getOrderId());
        // 更新订单状态
        orderRepository.updateStatus(event.getOrderId(), OrderStatus.FAILED);
    }
}
```

## 编排模式（Orchestration）

### 架构设计

```
                    Saga Orchestrator
                    ┌──────────────────┐
                    │                  │
                    │  1. 创建订单     │──── Order Service
                    │  2. 扣减库存     │──── Inventory Service
                    │  3. 处理支付     │──── Payment Service
                    │  4. 发送通知     │──── Notification Service
                    │                  │
                    │  补偿逻辑:       │
                    │  3失败 → 恢复库存 │
                    │  4失败 → 退款    │
                    │                  │
                    └──────────────────┘
```

### 编排器实现

```java
// Saga 编排器 - 集中控制事务流程
@Component
public class CreateOrderSagaOrchestrator {

    private final SagaStateRepository stateRepository;
    private final RestClient restClient;

    public OrderResult execute(CreateOrderRequest request) {
        // 1. 初始化 Saga 状态
        SagaState state = SagaState.create(request);
        stateRepository.save(state);

        try {
            // 2. 创建订单
            OrderResponse order = createOrder(request);
            state.setOrderId(order.orderId());
            state.addStep(SagaStep.ORDER_CREATED);
            stateRepository.save(state);

            // 3. 扣减库存
            InventoryResponse inventory = deductInventory(order);
            state.addStep(SagaStep.INVENTORY_DEDUCTED);
            stateRepository.save(state);

            // 4. 处理支付
            PaymentResponse payment = processPayment(order);
            state.addStep(SagaStep.PAYMENT_COMPLETED);
            stateRepository.save(state);

            // 5. 标记完成
            state.complete();
            stateRepository.save(state);
            
            return new OrderResult(state.getOrderId(), OrderStatus.SUCCESS);

        } catch (Exception e) {
            log.error("Saga failed at step: {}, reason: {}", 
                state.getCurrentStep(), e.getMessage());
            // 执行补偿
            compensate(state);
            return new OrderResult(state.getOrderId(), OrderStatus.COMPENSATED);
        }
    }

    private void compensate(SagaState state) {
        // 从后往前执行补偿
        List<SagaStep> completedSteps = state.getCompletedSteps();
        Collections.reverse(completedSteps);

        for (SagaStep step : completedSteps) {
            try {
                switch (step) {
                    case PAYMENT_COMPLETED -> refundPayment(state);
                    case INVENTORY_DEDUCTED -> restoreInventory(state);
                    case ORDER_CREATED -> cancelOrder(state);
                }
                log.info("Compensated step: {}", step);
            } catch (Exception e) {
                log.error("Compensation failed for step: {}", step, e);
                // 记录补偿失败，需要人工介入
                state.addFailedCompensation(step);
            }
        }
        state.compensate();
        stateRepository.save(state);
    }
}
```

### 使用状态机管理 Saga

复杂 Saga 推荐使用状态机（如 Spring Statemachine）：

```java
@Configuration
@EnableStateMachine
public class SagaStateMachineConfig 
        extends StateMachineConfigurerAdapter<SagaStates, SagaEvents> {

    @Override
    public void configure(StateMachineStateConfigurer<SagaStates, SagaEvents> states) 
            throws Exception {
        states
            .withStates()
                .initial(SagaStates.CREATED)
                .state(SagaStates.ORDER_CREATED)
                .state(SagaStates.INVENTORY_DEDUCTED)
                .state(SagaStates.PAYMENT_PROCESSING)
                .state(SagaStates.COMPLETED)
                .end(SagaStates.COMPENSATED)
                .end(SagaStates.FAILED);
    }

    @Override
    public void configure(
            StateMachineTransitionConfigurer<SagaStates, SagaEvents> transitions) 
            throws Exception {
        transitions
            // 正向流程
            .withExternal()
                .source(SagaStates.CREATED)
                .target(SagaStates.ORDER_CREATED)
                .event(SagaEvents.CREATE_ORDER)
                .action(createOrderAction())
            .and()
            .withExternal()
                .source(SagaStates.ORDER_CREATED)
                .target(SagaStates.INVENTORY_DEDUCTED)
                .event(SagaEvents.DEDUCT_INVENTORY)
                .action(deductInventoryAction())
            .and()
            .withExternal()
                .source(SagaStates.INVENTORY_DEDUCTED)
                .target(SagaStates.PAYMENT_PROCESSING)
                .event(SagaEvents.PROCESS_PAYMENT)
                .action(processPaymentAction())
            .and()
            .withExternal()
                .source(SagaStates.PAYMENT_PROCESSING)
                .target(SagaStates.COMPLETED)
                .event(SagaEvents.COMPLETE)
            // 补偿流程
            .and()
            .withExternal()
                .source(SagaStates.PAYMENT_PROCESSING)
                .target(SagaStates.COMPENSATED)
                .event(SagaEvents.COMPENSATE)
                .action(refundPaymentAction())
            .and()
            .withExternal()
                .source(SagaStates.INVENTORY_DEDUCTED)
                .target(SagaStates.COMPENSATED)
                .event(SagaEvents.COMPENSATE)
                .action(restoreInventoryAction())
                .action(cancelOrderAction());
    }
}
```

## 幂等性处理

无论是正向操作还是补偿操作，都必须保证**幂等性**：

```java
@Service
public class IdempotentInventoryService {

    @Transactional
    public boolean deductInventory(String orderId, Long productId, Integer quantity) {
        // 使用订单 ID 作为幂等键
        String idempotentKey = "inventory:deduct:" + orderId;
        
        // 检查是否已执行
        if (idempotentService.isProcessed(idempotentKey)) {
            return true; // 已执行，直接返回成功
        }
        
        // 执行扣减
        int updated = inventoryRepository.deduct(productId, quantity);
        if (updated == 0) {
            throw new InsufficientInventoryException("库存不足");
        }
        
        // 标记幂等
        idempotentService.markProcessed(idempotentKey);
        return true;
    }

    @Transactional
    public boolean restoreInventory(String orderId, Long productId, Integer quantity) {
        String idempotentKey = "inventory:restore:" + orderId;
        
        if (idempotentService.isProcessed(idempotentKey)) {
            return true;
        }
        
        inventoryRepository.restore(productId, quantity);
        idempotentService.markProcessed(idempotentKey);
        return true;
    }
}
```

## 事务日志表设计

```sql
-- Saga 状态表
CREATE TABLE saga_state (
    id              BIGSERIAL PRIMARY KEY,
    saga_id         VARCHAR(64) NOT NULL UNIQUE,   -- Saga 唯一 ID
    saga_type       VARCHAR(64) NOT NULL,           -- Saga 类型（如：CREATE_ORDER）
    current_state   VARCHAR(32) NOT NULL,           -- 当前状态
    completed_steps TEXT[] DEFAULT '{}',            -- 已完成步骤列表
    payload         JSONB,                          -- 请求参数
    result          JSONB,                          -- 执行结果
    failed_compensations TEXT[] DEFAULT '{}',       -- 补偿失败的步骤
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    version         INT NOT NULL DEFAULT 0          -- 乐观锁
);

-- 幂等表
CREATE TABLE idempotent_record (
    id              BIGSERIAL PRIMARY KEY,
    idempotent_key  VARCHAR(128) NOT NULL UNIQUE,   -- 幂等键
    status          VARCHAR(16) NOT NULL,           -- PROCESSED / COMPENSATED
    response        JSONB,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    expired_at      TIMESTAMP NOT NULL              -- TTL 过期时间
);

CREATE INDEX idx_idempotent_key ON idempotent_record(idempotent_key);
CREATE INDEX idx_saga_id ON saga_state(saga_id);
```

## 注意事项与坑点

### 1. 补偿必须是幂等的

订单已取消的情况下重复取消订单不能报错，日志记录即可。

### 2. 超时处理

```java
// 使用调度任务检测超时的 Saga
@Scheduled(fixedRate = 60000)
public void detectTimeoutSagas() {
    List<SagaState> timeoutSagas = sagaStateRepository
        .findByCurrentStateNotInAndUpdatedAtBefore(
            List.of("COMPLETED", "COMPENSATED", "FAILED"),
            LocalDateTime.now().minusMinutes(30));
    
    for (SagaState saga : timeoutSagas) {
        log.warn("Saga timeout detected: {}", saga.getSagaId());
        sagaOrchestrator.compensate(saga);
    }
}
```

### 3. 避免循环补偿

```java
// 使用补偿计数器限制递归深度
public void compensate(SagaState state) {
    if (state.getCompensationCount() >= MAX_COMPENSATION_RETRIES) {
        log.error("Max compensation retries reached for saga: {}", state.getSagaId());
        // 标记需要人工介入
        state.markManualIntervention();
        return;
    }
    state.incrementCompensationCount();
    // ... 执行补偿
}
```

### 4. 选择建议

- **2-3 步简单流程** → 编舞模式（代码量少，去中心化）
- **4 步以上复杂流程** → 编排模式（易于管理和调试）
- **需要严格事务保证** → 考虑 Seata AT/TCC 模式
- **高吞吐场景** → 编舞 + 事件驱动
- **频繁变更的业务流程** → 编排模式（修改集中）