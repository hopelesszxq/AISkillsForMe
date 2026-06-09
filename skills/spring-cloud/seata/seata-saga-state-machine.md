---
name: seata-saga-state-machine
description: Seata Saga 状态机模式深度实战 —— JSON 状态机定义、补偿策略、并行节点与事件触发
tags: [spring-cloud, seata, saga, distributed-transaction, state-machine, compensation]
---

## 概述

Seata Saga 模式基于**状态机引擎（State Machine）** 编排分布式事务，通过 JSON 定义事务流程，支持**正向补偿 + 逆向补偿**机制。相比 AT 和 TCC，Saga 更适合**长事务、跨系统编排**场景（如订单创建 → 支付 → 发货）。

核心特点：
- **无全局锁**：事务隔离由业务层保证
- **状态持久化**：引擎记录每一步执行状态，支持失败重试
- **JSON 编排**：状态机以 JSON 描述，无需写额外代码

> Seata 2.6.0 优化了 Saga 模块的方法命名和代码结构（[#7894](https://github.com/apache/incubator-seata/pull/7894)），并使用 `@SagaTransactional` 替代旧 `@LocalTCC` 注解（[#7443](https://github.com/apache/incubator-seata/pull/7443)）。

---

## 快速集成

```xml
<dependency>
    <groupId>org.apache.seata</groupId>
    <artifactId>seata-saga-spring</artifactId>
    <version>${seata.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.seata</groupId>
    <artifactId>seata-saga-statelang</artifactId>
    <version>${seata.version}</version>
</dependency>
```

---

## 状态机 JSON 定义

### 基础示例：下单流程

```json
{
  "Name": "createOrderSaga",
  "Comment": "创建订单 Saga",
  "StartState": "ReserveStock",
  "Version": "0.0.1",
  "States": {
    "ReserveStock": {
      "Type": "ServiceTask",
      "ServiceName": "stockService",
      "ServiceMethod": "reserve",
      "CompensateState": "CancelReserve",
      "Input": ["$request.itemId", "$request.quantity"],
      "Output": {
        "stockResult": "$response"
      },
      "Next": "CreateOrder"
    },
    "CreateOrder": {
      "Type": "ServiceTask",
      "ServiceName": "orderService",
      "ServiceMethod": "create",
      "CompensateState": "CancelOrder",
      "Input": ["$request.userId", "$request.itemId", "$stockResult"],
      "Output": {
        "orderId": "$response.id"
      },
      "Next": "PayOrder"
    },
    "PayOrder": {
      "Type": "ServiceTask",
      "ServiceName": "paymentService",
      "ServiceMethod": "pay",
      "CompensateState": "RefundPayment",
      "Input": ["$orderId", "$request.amount"],
      "Next": "SendNotify"
    },
    "SendNotify": {
      "Type": "ServiceTask",
      "ServiceName": "notificationService",
      "ServiceMethod": "send",
      "Input": ["$orderId"],
      "End": true
    },
    "CancelReserve": {
      "Type": "ServiceTask",
      "ServiceName": "stockService",
      "ServiceMethod": "cancelReserve",
      "Input": ["$request.itemId", "$request.quantity"]
    },
    "CancelOrder": {
      "Type": "ServiceTask",
      "ServiceName": "orderService",
      "ServiceMethod": "cancel",
      "Input": ["$orderId"]
    },
    "RefundPayment": {
      "Type": "ServiceTask",
      "ServiceName": "paymentService",
      "ServiceMethod": "refund",
      "Input": ["$orderId", "$request.amount"]
    }
  }
}
```

### 状态机类型

| 类型 | 说明 | 用途 |
|---|---|---|
| `ServiceTask` | 调用服务方法 | 核心业务逻辑 |
| `Choice` | 条件分支 | 根据返回值决策走向 |
| `Fail` | 标记失败 | 终止当前分支 |
| `Success` | 标记成功 | 终止当前分支 |
| `Loop` | 循环执行 | 重试/轮询场景 |
| `SubStateMachine` | 子状态机 | 复用已有编排 |
| `Join` | 并行合并 | 多路并行后聚合 |

---

## 并行节点

### 并行处理多个任务

```json
{
  "StartState": "ParallelTasks",
  "States": {
    "ParallelTasks": {
      "Type": "ServiceTask",
      "ServiceName": "parallelGateway",
      "ServiceMethod": "split",
      "IsForUpdate": false,
      "Next": {
        "Default": "CheckInventory",
        "Choices": [
          {
            "Expression": "$response.splitResult == true",
            "Next": "AllTasks"
          }
        ]
      },
      "Output": {
        "splitResult": "$response"
      }
    },
    "AllTasks": {
      "Type": "Parallel",
      "Branches": [
        {
          "State": "ChargePayment",
          "CompensateState": "RefundPayment"
        },
        {
          "State": "SendEmail",
          "CompensateState": "VoidEmail"
        },
        {
          "State": "UpdateCoupon",
          "CompensateState": "RestoreCoupon"
        }
      ]
    },
    "ChargePayment": {
      "Type": "ServiceTask",
      "ServiceName": "paymentService",
      "ServiceMethod": "charge",
      "CompensateState": "RefundPayment",
      "Input": ["$request.orderId", "$request.amount"]
    },
    "SendEmail": {
      "Type": "ServiceTask",
      "ServiceName": "notificationService",
      "ServiceMethod": "sendEmail",
      "CompensateState": "VoidEmail",
      "Input": ["$request.orderId"]
    },
    "UpdateCoupon": {
      "Type": "ServiceTask",
      "ServiceName": "couponService",
      "ServiceMethod": "use",
      "CompensateState": "RestoreCoupon",
      "Input": ["$request.userId", "$request.couponId"]
    },
    "RefundPayment": { "Type": "ServiceTask", "ServiceName": "paymentService", "ServiceMethod": "refund", "Input": ["$request.orderId"] },
    "VoidEmail": { "Type": "ServiceTask", "ServiceName": "notificationService", "ServiceMethod": "voidEmail", "Input": ["$request.orderId"] },
    "RestoreCoupon": { "Type": "ServiceTask", "ServiceName": "couponService", "ServiceMethod": "restore", "Input": ["$request.userId", "$request.couponId"] }
  }
}
```

---

## Spring Boot 集成

### 1. 服务接口定义

每个 `ServiceTask` 调用的 `ServiceName` 需要对应 Spring Bean 名称：

```java
@Component("stockService")
public class StockService {

    // 正向操作
    public StockResult reserve(Long itemId, Integer quantity) {
        // 扣减库存
        return new StockResult(true, "成功");
    }

    // 补偿操作
    public boolean cancelReserve(Long itemId, Integer quantity) {
        // 回退库存
        return true;
    }
}
```

### 2. 状态机加载

```java
@Component
public class SagaConfig {

    @PostConstruct
    public void init() {
        // 从 classpath 加载 JSON
        StateMachineEngine engine = SagaContext.getEngine();
        engine.registerStateMachine(
            "classpath:saga/create-order-saga.json"
        );
    }
}
```

### 3. 启动 Saga

```java
@Service
public class OrderSagaService {

    @SagaTransactional(
        sagaName = "createOrderSaga",
        timeout = 60000,
        async = false
    )
    public OrderResult createOrder(CreateOrderRequest request) {
        // 状态机引擎通过 JSON 自动编排
        // 返回值作为 $response 传入状态机
        return orderFacade.prepare(request);
    }
}
```

> ⚠️ 注意：`@SagaTransactional` 替代了旧版 `@LocalTCC`（Seata 2.6.0+），不要混用。

---

## 条件分支 Choice

支持 JSONPath 表达式的条件路由：

```json
{
  "CheckResult": {
    "Type": "Choice",
    "Choices": [
      {
        "Expression": "$response.success == true",
        "Next": "ConfirmOrder"
      },
      {
        "Expression": "$response.retryable == true",
        "Next": "RetryReserve"
      }
    ],
    "Default": "NotifyFailure"
  }
}
```

---

## 关键注意事项

### 1. 空补偿
补偿方法被调用但正向操作未执行（因超时等原因）。必须支持**空补偿**：

```java
// 补偿方法要做幂等判断
public boolean cancelReserve(Long itemId, Integer quantity) {
    // 先检查正向操作是否已执行
    if (!stockRepository.existsReserveRecord(itemId, quantity)) {
        // 未执行，直接返回成功
        return true;
    }
    // 执行真正的补偿
    stockRepository.restore(itemId, quantity);
    return true;
}
```

### 2. 悬挂补偿
补偿方法比正向方法先执行完。需要使用事务日志/幂等表：

```sql
-- 在正向和补偿方法中都记录操作状态
CREATE TABLE saga_log (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    saga_id     VARCHAR(64)  NOT NULL,
    state_name  VARCHAR(64)  NOT NULL,
    action      VARCHAR(16)  NOT NULL,  -- 'forward' 或 'compensate'
    status      VARCHAR(16)  NOT NULL,  -- 'pending', 'success', 'failed'
    created_at  TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_saga_state (saga_id, state_name, action)
);
```

### 3. 无隔离性陷阱
Saga 不会加锁，中间状态对外可见。需在业务层处理：

- **脏读**：其他服务可能读到未完成 Saga 的中间数据
- **解决**：使用 `status` 字段标记数据是否最终有效，或使用 `final` 表

### 4. 补偿方法的幂等性
TC 会重试补偿方法直到成功，**所有补偿方法必须幂等**：

```java
@Retryable  // 本身就是幂等的
public boolean refund(Long orderId, BigDecimal amount) {
    RefundRecord record = refundRepository.findByOrderId(orderId);
    if (record != null) {
        return true;  // 已退款，幂等返回
    }
    // 执行退款
    refundRepository.save(new RefundRecord(orderId, amount));
    return true;
}
```

### 5. 超时配置

```yaml
seata:
  saga:
    json-translator:
      # 每次 ServiceTask 调用超时（毫秒）
      service-task-timeout: 5000
    # 状态机重试配置
    retry:
      max-retries: 3
      retry-interval: 1000
```

---

## 与其他模式对比

| 维度 | Saga | AT | TCC |
|---|---|---|---|
| 隔离性 | 无（业务自行保证） | 有（全局锁） | 有（资源预留） |
| 性能 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| 编码量 | 中（JSON + 补偿方法） | 少（自动回滚 SQL） | 多（三段式） |
| 长事务 | ✅ 天然适合 | ❌ 锁竞争严重 | ❌ 资源锁释放慢 |
| 跨服务编排 | ✅ 状态机原生支持 | ❌ 只做单服务包裹 | ❌ 需自行编排 |
| 状态持久化 | ✅ 内置 | ❌ | ❌ |

---

## 参考链接

- [Apache Seata Saga 官方文档](https://seata.apache.org/docs/user/saga/)
- [Seata 2.6.0 Saga 注解改进 PR#7443](https://github.com/apache/seata/pull/7443)
- [Seata 状态机 JSON 规范](https://github.com/apache/incubator-seata/blob/2.x/saga/seata-saga-statelang/README.md)
