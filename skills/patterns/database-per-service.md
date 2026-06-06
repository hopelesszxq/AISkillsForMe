---
name: database-per-service
description: 微服务 Database per Service 模式——服务数据隔离、私有持久化策略与跨服务查询方案
tags: [architecture, microservice, pattern, database, data-management, spring-boot]
---

## 概述

Database per Service 是微服务架构的核心数据模式：**每个微服务拥有自己的私有数据库**，其他服务只能通过 API 访问数据，不能直接访问其数据库。这是实现服务松耦合和数据封装的基础。

## 核心原则

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Order Service  │     │   User Service   │     │  Payment Service │
│  ┌───────────┐   │     │  ┌───────────┐   │     │  ┌───────────┐   │
│  │  Order DB  │   │     │  │  User DB   │   │     │  Payment DB │   │
│  └───────────┘   │     │  └───────────┘   │     │  └───────────┘   │
│    只允许 API     │     │    只允许 API     │     │    只允许 API     │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         └─────────────── REST / gRPC ───────────────────┘
```

### 优点

| 维度 | 说明 |
|------|------|
| **松耦合** | 一个服务的数据库变更不影响其他服务 |
| **独立扩展** | 每个服务可选最适合的数据库类型（SQL/NoSQL/时序库） |
| **独立部署** | 数据库变更可独立发布，无需全局协调 |
| **数据安全** | 服务天然实现数据访问边界，减少违规泄露风险 |

### 缺点

| 维度 | 说明 |
|------|------|
| **跨服务查询复杂** | 无法跨数据库 JOIN，需通过 API 聚合或 CQRS |
| **分布式事务** | 跨服务数据一致性需要通过 Saga / Outbox 模式保证 |
| **运维成本** | 数据库数量增长，需要容器化或托管数据库管理 |
| **数据冗余** | 不同服务可能重复存储相同数据 |

## 数据隔离策略

### 1. 独立数据库实例（最严格）

每个服务对应独立的数据库实例，适合安全性要求高的场景。

```yaml
# order-service/application.yml
spring:
  datasource:
    url: jdbc:postgresql://order-db-host:5432/orderdb
    username: order_service
    password: ${ORDER_DB_PASSWORD}
```

```yaml
# user-service/application.yml
spring:
  datasource:
    url: jdbc:postgresql://user-db-host:5432/userdb
    username: user_service
    password: ${USER_DB_PASSWORD}
```

**Kubernetes 部署**：每个服务使用独立的 StatefulSet + PVC

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: order-db
spec:
  serviceName: order-db
  replicas: 1
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:17
        env:
        - name: POSTGRES_DB
          value: orderdb
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

### 2. 同一实例不同 Schema（折中方案）

适合中小规模微服务，节约资源但保持逻辑隔离。

```yaml
# 同一 PostgreSQL 实例，不同 schema
spring:
  datasource:
    url: jdbc:postgresql://shared-db:5432/microservices
    username: ${SERVICE_USER}
    password: ${SERVICE_PASSWORD}
```

```sql
-- 初始化时分区
CREATE SCHEMA IF NOT EXISTS order_service;
CREATE SCHEMA IF NOT EXISTS user_service;

-- 权限隔离
REVOKE ALL ON SCHEMA order_service FROM public;
GRANT USAGE ON SCHEMA order_service TO order_user;
```

### 3. 多租户共享表（最松）

适合 SaaS 场景，通过 `tenant_id` 区分数据。

```java
// MyBatis-Plus 多租户插件
@Bean
public TenantLineInnerInterceptor tenantLineInnerInterceptor() {
    return new TenantLineInnerInterceptor(new TenantLineHandler() {
        @Override
        public String getTenantIdColumn() {
            return "tenant_id";
        }

        @Override
        public Expression getTenantId() {
            return new StringValue(TenantContext.getCurrentTenant());
        }
    });
}
```

## 跨服务查询方案

### 1. API 聚合查询（最常用）

```java
// OrderService 调用 UserService 获取用户信息
@Service
public class OrderAggregator {
    private final OrderRepository orderRepository;
    private final UserServiceClient userServiceClient;

    public OrderDetailVO getOrderDetail(Long orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        // 调用 UserService 获取用户详情
        UserDTO user = userServiceClient.getUser(order.getUserId());

        // 调用 PaymentService 获取支付状态
        PaymentDTO payment = paymentServiceClient.getPayment(order.getPaymentId());

        return OrderDetailVO.builder()
            .order(order)
            .user(user)
            .payment(payment)
            .build();
    }
}
```

### 2. CQRS + 物化视图

创建只读查询服务，使用事件驱动的数据同步。

```java
// OrderEventHandler 监听订单事件，更新查询模型
@Component
public class OrderEventHandler {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        // 同步到查询数据库
        queryOrderRepository.save(QueryOrder.builder()
            .orderId(event.getOrderId())
            .userId(event.getUserId())
            .userName(event.getUserName())  // 冗余用户信息
            .status(event.getStatus())
            .createdAt(event.getCreatedAt())
            .build());
    }
}
```

### 3. GraphQL 联邦网关

```graphql
# Federation schema
type Order @key(fields: "id") {
  id: ID!
  userId: ID!
  total: Float!
  user: User    # 由 UserService 解析
}

type User @key(fields: "id") {
  id: ID!
  name: String!
}
```

## 分布式事务处理

### Saga + Outbox 模式

```java
// 订单创建 Saga
@Saga(name = "orderCreation")
public class CreateOrderSaga implements SagaDefinition {

    @SagaStep(compensation = "cancelOrder")
    public void createOrder(CreateOrderRequest request) {
        orderRepository.save(Order.builder()
            .id(request.getOrderId())
            .userId(request.getUserId())
            .amount(request.getAmount())
            .status("PENDING")
            .build());
    }

    @SagaStep(compensation = "refundPayment")
    public boolean reservePayment(PaymentRequest request) {
        return paymentClient.reserve(request);
    }

    @SagaStep
    public void completeOrder(Long orderId) {
        orderRepository.updateStatus(orderId, "CONFIRMED");
    }

    @SagaStep(compensation = "reverseReserve")
    public boolean deductInventory(InventoryRequest request) {
        return inventoryClient.deduct(request);
    }

    // 补偿操作
    public void cancelOrder(Long orderId) {
        orderRepository.updateStatus(orderId, "CANCELLED");
    }
}
```

## 数据库类型选择参考

| 服务类型 | 推荐数据库 | 原因 |
|---------|-----------|------|
| 订单/交易 | PostgreSQL / MySQL | ACID 保证，强一致性 |
| 用户/权限 | PostgreSQL | 关系模型，复杂查询 |
| 产品目录 | MongoDB | 文档模型，灵活 Schema |
| 日志/监控 | Elasticsearch / ClickHouse | 全文搜索，时间序列分析 |
| 缓存/会话 | Redis | 高吞吐，低延迟 |
| 文件/图片 | MinIO | 对象存储，S3 兼容 |
| 时序数据 | InfluxDB / TimescaleDB | 时间序列优化 |

## 注意事项

1. **不要共享数据库**：两个服务读写同一数据库表会打破服务边界，应通过 API 或事件驱动通信
2. **不要跨服务 JOIN**：SQL JOIN 跨数据库实例不可能，应用层聚合或 CQRS 解决
3. **数据冗余是可接受的**：不同服务可以冗余存储同一数据的子集，保持值对象（Value Object）而非实体
4. **关注数据迁移**：每个服务的数据库独立演进，使用 Flyway / Liquibase 管理 Schema 版本
5. **连接池隔离**：每个服务使用独立的连接池，避免互相影响
6. **考虑数据同步延迟**：事件驱动的数据同步有最终一致性延迟，需在前端或 API 层处理
7. **小型项目不适用**：如果只有 2-3 个服务，Shared Database 模式可能更经济，后期再逐步拆分
